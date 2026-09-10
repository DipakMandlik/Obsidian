---
tags: [dmart, snowflake, ml-forecast, demand-planning]
---

# Snowflake ML Forecast — DMart Demand Planning

> Core principle: there is **no universally "highest-accuracy" model**. The right model is the one that wins on DMart's own SKU × Store data via proper time-series backtesting — not the one that "sounds more advanced."

## 1. Recommendation

**Evaluate `SNOWFLAKE.ML.FORECAST` first**, before introducing an external forecasting platform.

```text
DMART SALES DATA → SNOWFLAKE.ML.FORECAST → FORECAST → FACT_SALES_PRODUCT_FORECAST → Cortex Analyst / Cortex Agent
```

Native capabilities: multiple time series, exogenous/additional features, prediction intervals, out-of-sample evaluation, feature importance, training logs, SQL-based inference. Fits DMart's existing Snowflake-centric architecture directly.

## 2. `method='best'` vs `method='fast'`

| Method     | Approach                                                                                 | When to use                                                                      |
| ---------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **`best`** | Ensemble across Prophet, ARIMA, Exponential Smoothing, GBM  framework selects per series | **Start here for DMart POC**  prioritizes forecast quality                       |
| `fast`     | GBM-based only                                                                           | Recommended by Snowflake only at 10,000+ individual series (speed over accuracy) |

## 3. Why this fits DMart specifically

Not just `Date → Sales`. Real shape of the problem:

```text
Store + SKU + Historical Sales + Calendar + Seasonality + Promotions
+ Holiday/Event + Inventory + External factors → Demand Forecast
```

Snowflake Forecast supports exogenous features (Snowflake's own examples: weather, holidays, ad campaigns, event schedules), so the target model is:

```text
SKU × STORE × DATE
  ├── SALES_QTY
  ├── PRICE
  ├── PROMOTION
  ├── HOLIDAY
  ├── WEEK_OF_YEAR / MONTH
  ├── INVENTORY
  └── EVENT
        ↓
  SNOWFLAKE.ML.FORECAST → DEMAND FORECAST
```

## 4. Implementation architecture

**Do not train directly off `FACT_SALES_PRODUCT_FORECAST`** — that's the output layer. Build a training view upstream:

```text
FACT_SALES_TRANS + DIM_PRODUCT + DIM_LOCATION + Calendar + Promotion/Event data
        ↓
FORECAST_TRAINING_VIEW → SNOWFLAKE.ML.FORECAST → FORECAST_OUTPUT → FACT_SALES_PRODUCT_FORECAST
```

```sql
CREATE OR REPLACE VIEW VW_DMART_FORECAST_TRAINING AS
SELECT
    SALES_DATE,
    LOC_SID,
    SKU_SID,
    SUM(SALES_QTY) AS SALES_QTY
FROM DMART_INDIA_DB.GOLD.FACT_SALES_TRANS
GROUP BY SALES_DATE, LOC_SID, SKU_SID;

CREATE OR REPLACE SNOWFLAKE.ML.FORECAST
    DMART_DEMAND_FORECAST(
        INPUT_DATA => TABLE(DMART_INDIA_DB.GOLD.VW_DMART_FORECAST_TRAINING),
        TIMESTAMP_COLNAME => 'SALES_DATE',
        TARGET_COLNAME => 'SALES_QTY',
        CONFIG_OBJECT => {'method': 'best', 'evaluate': TRUE}
    );
```

Note: `CREATE SNOWFLAKE.ML.FORECAST` requires that privilege on the schema. Adapt columns/grain to actual DMart data.

## 5. Never claim "highest accuracy" without testing

Build a real evaluation loop:

```text
Historical Data → Training Window → Forecast → Actual Future Data → Compare
→ MAE / MAPE / SMAPE / MSE / MDA → Model Selection
```

`SHOW_EVALUATION_METRICS()` exposes: MAE, MAPE, SMAPE, MSE, MDA, prediction-interval coverage, Winkler score.

**For DMart, prioritize MAE + WAPE/SMAPE + interval coverage over MAPE** — retail demand contains zeros, which breaks MAPE.

## 6. Backtest example (illustrative only, not real DMart numbers)

Train `Jan 2024 → Dec 2025`, test on `Jan 2026 → Mar 2026`:

| Model | MAE | SMAPE | MDA | Rank |
|---|--:|--:|--:|--:|
| Seasonal baseline | 18.4 | 31.2% | 55% | 4 |
| ARIMA | 15.8 | 27.5% | 61% | 3 |
| GBM | 13.9 | 23.1% | 68% | 2 |
| **Snowflake `best`** | **12.7** | **21.4%** | **72%** | **1** |

This is the difference between "we evaluated forecast error out-of-sample and picked a champion" vs. "Snowflake is accurate because it's Snowflake."

## 7. Snowflake-native vs external model

```text
Option A: DMart Data → Snowflake → SNOWFLAKE.ML.FORECAST → Forecast → Cortex Agent
Option B: Snowflake → Export → Databricks/SageMaker/Vertex → XGBoost/LightGBM/TFT → Prediction → back to Snowflake → Cortex Agent
```

External models aren't necessarily a full exit from Snowflake either  the Model Registry / ML platform can host externally-trained models too.

| Factor | Snowflake ML Forecast | External ML |
|---|---|---|
| Data movement | **Minimal** | Higher |
| Separate infrastructure | **No** | Usually yes |
| Training | Warehouse compute | External compute |
| Inference | Snowflake | External endpoint/compute |
| Governance | **Centralized** | Multiple systems |
| Cortex integration | **Native** | Integration required |
| Experimentation flexibility | Good | **Excellent** |
| Algorithm control | Moderate | **Very high** |
| MLOps complexity | **Low** | Higher |
| Custom models (intermittent demand, cold-start, hierarchical) | Limited | **Excellent** |
| Initial POC cost | **Lower** | Usually higher |
| Potential ceiling accuracy | Good | **Potentially higher** |
| Operational simplicity | **Excellent** | Moderate/low |

**Cost shape:**
- Snowflake-native: training compute + inference compute + storage, reusing existing warehouse spend. Snowflake's own benchmark: ~400s on XS warehouse for 100K rows, ~850s for 1M rows (evaluation disabled; more splits = more time).
- External: Snowflake compute + data export + external compute + training + model registry + inference endpoint + network transfer + monitoring + MLOps — materially more moving parts.

## 8. When external could actually win

If DMart's data has:
- highly intermittent demand
- thousands/millions of SKU-store combinations
- complex promotions, price elasticity, substitution, cannibalization
- new-product cold starts
- hierarchical forecasting
- strongly nonlinear relationships

...then a tuned external model (e.g., XGBoost/LightGBM + lag features, rolling stats, price, promotion, holiday, inventory, store/product attributes) could beat Snowflake `best` — **but only proven on DMart's own data**, not assumed.

**Skip deep learning (TFT, DeepAR, N-BEATS, N-HiTS, LSTM) for the POC** — not automatically more accurate for retail, and adds training complexity, GPU needs, weaker explainability, more MLOps. Establish a strong classical/ML baseline first.

## 9. Recommended architecture — benchmark, not single-model commitment

```text
                 DMART SALES
                     │
              FEATURE LAYER
       ┌─────────────┼──────────────┐
       ▼             ▼              ▼
 Snowflake       External        Baseline
 Forecast(best)  XGBoost/LGBM    Seasonal/Naive
       └─────────────┼──────────────┘
                     ▼
              BACKTEST ENGINE (MAE/SMAPE/MDA)
                     ▼
             MODEL CHAMPION
                     ▼
          FACT_SALES_PRODUCT_FORECAST → CORTEX AGENT
```

## 10. Phased plan for the DMart POC

| Phase | Action |
|---|---|
| **1 — Native Snowflake** | Build `SNOWFLAKE.ML.FORECAST` with `method='best'`, `evaluate=true` at SKU × Store × Day (or actual supported grain). Add calendar, holidays, promotions, price, inventory where future values are genuinely known. |
| **2 — Benchmark** | Compare naive/seasonal baseline vs Snowflake `best` vs GBM/custom model. Don't assume a winner — measure MAE, SMAPE, MDA, prediction-interval coverage. |
| **3 — Production decision** | If Snowflake `best` ≥ external model → ship on Snowflake (cleanest architecture: Data + Forecast + Cortex Analyst/Search/Agent + MCP + Procurement Action, all in one platform). If external wins by a **material, repeatable** margin → use it for forecasting, keep Snowflake as system of record + agent/action layer. |

## 11. The one rule

Don't pick a model because it has the "highest advertised accuracy." Ask instead:

> Which model minimizes forecast error on DMart's unseen historical demand while remaining operationally and economically acceptable?

That model is the **champion model**. Snowflake `best` (Prophet + ARIMA + Exponential Smoothing + GBM ensemble) is the strongest first candidate to benchmark against.

## 12. Next action

Build a **DMart Forecast Model Benchmark**: `SNOWFLAKE.ML.FORECAST` vs. a strong baseline, producing a client-ready accuracy + cost comparison table — before touching the existing `FACT_SALES_PRODUCT_FORECAST`.

---

# Model Registry, Cortex Models & Cortex Agent — Don't Mix These Up

Three genuinely different things in Snowflake, easy to conflate:

| # | Concept | What it is | Relevant to |
|---|---|---|---|
| 1 | **Model Registry** | Where *your trained ML models* (e.g. custom XGBoost) live, version, and get served | DMart forecasting model |
| 2 | **Cortex models** | Foundation/LLM models (Claude, OpenAI, Gemini, Llama, etc.) available inside Cortex | Cortex Agent's reasoning |
| 3 | **Cortex Agent** | Orchestration layer: an LLM + Analyst/Search/MCP/custom tools working together | Turning forecast → recommendation → action |

For DMart: **#1 drives the forecast number. #2 and #3 drive the agent that explains it and acts on it.**

## 13. Snowflake Model Registry — what it actually is

Think: **GitHub + MLflow Model Registry + model serving, integrated into Snowflake.**

```text
DMart Historical Data → Feature Engineering → Train ML Model → Model
        → Snowflake Model Registry → [V1, V2, V3, ...] → Production
```

Provides a unified interface for models that run either through the **warehouse/SQL engine** or **Snowpark Container Services (SPCS)**.

### Concrete DMart example

Train `DMART_DEMAND_XGB` on features `SKU, STORE, DATE, DAY_OF_WEEK, MONTH, WEEK_OF_YEAR, LAG_7, LAG_14, LAG_28, ROLLING_7, ROLLING_28, PRICE, PROMOTION, HOLIDAY, INVENTORY` → target `SALES_QTY`. Register each training run as a version:

| Version | Accuracy | Status |
|---|---|---|
| V1 | 71% | Archived |
| V2 | 76% | Archived |
| V3 | 81% | **Champion (Production)** |
| V4 | 78% | Challenger (worse — don't promote) |

Versioning means a worse retrain (V4) never forces you to destroy the working production model (V3) — keep V3 as champion, V4 as challenger, until V4 actually wins on backtest.

### What's stored with a registered model

```text
DMART_DEMAND_XGB
├── Version: V3
├── Algorithm: XGBoost
├── Metrics: MAE 12.7, SMAPE 21.4%, RMSE 18.3
├── Features: sales_lag_7, sales_lag_28, price, promotion, holiday
├── Parameters, dependencies, artifacts, signature, metadata
└── Status: CHAMPION
```

A registered model is a full ML lifecycle record — not an anonymous `.pkl` sitting in someone's Python environment.

## 14. Where does a registered model actually run?

| Inference environment | Flow | Best for |
|---|---|---|
| **Warehouse / SQL inference** | Model Registry → SQL → Virtual Warehouse → Prediction | DMart's case — data already in Snowflake; native batch inference inside SQL/data pipelines |
| **Snowpark Container Services (SPCS)** | Model Registry → SPCS → Inference Service → REST/Python/app | Real-time inference, complex/custom-environment models, high-throughput, anything that doesn't fit warehouse inference |

## 15. Model Registry vs `SNOWFLAKE.ML.FORECAST`

|  | Snowflake ML Forecast | Model Registry |
|---|---|---|
| Primary purpose | Time-series forecasting | General ML model lifecycle |
| Algorithms | Snowflake-managed (ensemble/GBM) | Your trained models |
| XGBoost / scikit-learn | Not the same workflow | **Yes** |
| Custom model | Limited | **Yes** |
| Versioning | Managed forecast object | **Strong model versioning** |
| SQL inference | Yes | Yes (depends on model/runtime) |
| Custom dependencies | Limited | **Much more flexible** |
| Best for | Fast forecasting implementation | Enterprise ML lifecycle |
| Operational complexity | **Low** | Medium |
| Customization | Medium | **High** |

**For DMart:** don't treat these as either/or. Run both as champion candidates against a baseline :

```text
              DMART DATA
                  │
          ┌───────────────┐
          │  BACKTESTING  │
          └───────┬───────┘
        ┌─────────┼──────────┐
        ▼         ▼          ▼
   Seasonal    Snowflake   XGBoost
    Naive       Forecast    (Model Registry)
        └─────────┼──────────┘
                  ▼
            MODEL SCORECARD → CHAMPION
```

## 16. Clarifying "GENERIC" - a naming collision, not a model concept

The **`GENERIC` MCP tool** used in DMart's procurement MCP has **nothing to do with ML models**. It just means the MCP server exposes a general-purpose tool backed by a Snowflake procedure:

```text
DMART_PROCUREMENT_MCP → create_purchase_order → SP_CREATE_PURCHASE_ORDER
```

This is fully separate from `Snowflake ML → Forecast Model` and from `Cortex Agent → Orchestration LLM`. Three unrelated things that happen to share vocabulary.

## 17. Separating responsibilities: model vs agent vs action

| Layer | Produces |
|---|---|
| **ML model** (Forecast or Registry) | `Demand = 1,245 units` |
| **Cortex Agent** | "Demand expected +14%. Inventory covers 8 days. Recommended purchase = 600." (reasoning, not the number itself) |
| **MCP** | Executes `CREATE PO` |

```text
ML MODEL → Demand Forecast → Cortex Agent → (Reasoning, MCP → PO)
```

The orchestration LLM never generates the numeric forecast — it reasons over it.

## 18. Cortex Agent cost structure

An agent request can stack multiple cost sources:

```text
User question → Cortex Agent
  ├── Orchestration LLM  → AI credits (token-based)
  ├── Cortex Analyst     → AI credits (token-based)
  ├── Cortex Search      → index/serving + embedding cost
  └── MCP / custom tool  → warehouse compute
```

Snowflake explicitly states these costs stack depending on which tools the agent invokes — **agent cost ≠ "number of questions × LLM token price."**

Cortex AI Credit pricing (before contract/discount): **$2.00/credit (global routing)**, **$2.20/credit (regional routing)**. This is only the AI-credit layer - normal Snowflake platform/warehouse credits are priced separately per your account/edition/contract. Don't multiply every Snowflake operation by $2.

## 19. Cost monitoring queries for the DMart POC

| What | Query |
|---|---|
| Cortex AI function usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY;` |
| Model Registry inference usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.MODEL_SERVING_USAGE_HISTORY;` |
| Account-wide AI services | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY WHERE SERVICE_TYPE = 'AI_SERVICES';` |
| Warehouse compute | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY;` |

Roll these up into a single DMart cost breakdown: Forecast Training, Forecast Inference, Cortex Agent, Cortex Analyst, Cortex Search, MCP/PO execution, PDF/Email.

## 20. Final consolidated DMart architecture

```text
                         DMART DATA
                             │
                    FEATURE ENGINEERING
                             │
              ┌──────────────────────────┐
              │      ML MODEL LAYER      │
              │  Snowflake Forecast OR   │
              │  Registered XGBoost      │
              └────────────┬─────────────┘
                           ▼
                 DEMAND FORECAST OUTPUT
                           ▼
                 FACT_SALES_PRODUCT_FORECAST
                           ▼
                  ┌──────────────────┐
                  │  CORTEX AGENT    │
                  │ Orchestration LLM│
                  └────────┬─────────┘
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Analyst         Search          MCP
             ▼             ▼             ▼
          SQL/Data      Documents   SP_CREATE_PURCHASE_ORDER
                                         ▼
                                   PO + Audit → PDF → Email
```

**The core separation to remember:** Model Registry manages your ML models. Cortex provides foundation models and AI services. Cortex Agent orchestrates reasoning. MCP performs controlled actions. Don't replace the current forecast blind — register a strong candidate, backtest it against the same holdout period as the current forecast, compare MAE/SMAPE + compute cost, then promote the winner.

## References

- [Time-Series Forecasting (Snowflake ML Functions)](https://docs.snowflake.com/en/user-guide/ml-functions/forecasting)
- [CREATE SNOWFLAKE.ML.FORECAST](https://docs.snowflake.com/en/sql-reference/classes/forecast/commands/create-forecast)
- [`<model_name>!SHOW_EVALUATION_METRICS`](https://docs.snowflake.com/en/sql-reference/classes/forecast/methods/show_evaluation_metrics)
- [Snowflake ML: End-to-End Agentic Machine Learning](https://docs.snowflake.com/en/developer-guide/snowflake-ml/overview)
- [Model Inference Overview](https://docs.snowflake.com/en/developer-guide/snowflake-ml/inference/inference-overview)
- [MODEL_SERVING_USAGE_HISTORY](https://docs.snowflake.com/en/sql-reference/account-usage/model_serving_usage_history)
- [Cortex Agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Cortex AI Cost Management & Governance](https://docs.snowflake.com/en/user-guide/snowflake-cortex/governance-and-availability/ai-cost-management-and-governance)
- [Cortex Pricing](https://docs.snowflake.com/en/user-guide/snowflake-cortex/pricing)
- [Understanding Compute Cost](https://docs.snowflake.com/en/user-guide/cost-understanding-compute)

Related: [[wiki/prototypes/Dmart/Agents.md|Dmart Agents]] · [[wiki/prototypes/Dmart/Agent in Action.md|Agent in Action]]
