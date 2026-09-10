---
title: IDP Platform Architecture
type: project-doc
created: 2026-09-07
updated: 2026-09-07
tags:
  - prototype
  - idp
  - snowflake
  - document-processing
  - ai-platform
  - pibythree
  - quickstart
  - limitations
---

# IDP Platform Architecture

## What It Does

The platform ingests mixed business documents (invoices, purchase orders, contracts, KYC packs), parses them, classifies them, extracts structured fields, validates the extraction, resolves entities across documents, and serves the results to both BI consumers and a conversational AI agent — all running entirely inside Snowflake.

## Quick Start

```bash
# 1. Run preflight
snow sql -f sql/00_setup/000_preflight.sql

# 2. Deploy (--dry-run to preview)
./deploy/deploy.sh --env DEV

# 3. Seed sample data
snow sql -f sample_data/seed_sample_documents.sql

# 4. Run smoke test
snow sql -f tests/810_smoke_test.sql

# 5. Resume task DAG
snow sql -f sql/50_orchestration/520_task_control.sql
```

Deploys the schemas in [[#Data Layer Architecture]], starts the [[#Pipeline Task DAG]], and runs the 7/7 checks reported in [[#Current Deployment Numbers]].

## Core Design Principle

> [!important] Cost is determined by how many documents reach an LLM.
> The entire architecture is optimized around routing. A regex/template-based deterministic lane handles the majority of documents (currently 75%). Only unmatched or failed documents touch a model. This is why **routing** is the primary engineering concern, not prompting.

---

## Data Layer Architecture

The platform uses a strict layered separation so that a change in one layer never forces reprocessing in another.

```mermaid
flowchart LR
    subgraph ACCT["SNOWFLAKE ACCOUNT (IDP_DEV)"]
        direction LR
        L["**LANDING**\nRegistry\n(state machine)"]
        B["**BRONZE**\nParsed text\n(raw)"]
        S["**SILVER**\nFields + Queue\n(interpreted)"]
        G["**GOLD**\nEntities + Insights\n(resolved)"]
        V["**SERVE**\nViews, Semantic,\nSearch, Agent"]
        L --> B --> S --> G --> V

        C["**CONFIG**\nTemplates, Prompts,\nSettings"]
        O["**OPS**\nRun Log, Cost Ledger,\nErrors"]
    end

    classDef layer fill:#e8f0fe,stroke:#4285f4,color:#1a1a1a;
    classDef support fill:#fef7e0,stroke:#f9ab00,color:#1a1a1a;
    class L,B,S,G,V layer;
    class C,O support;
```

| Layer | Schema | Contract | Why Separate |
|---|---|---|---|
| Landing | LANDING | Immutable source copy + document registry | Registry is the state machine and the idempotency boundary |
| Bronze | BRONZE | Faithful parsed text, zero interpretation | A prompt change never forces a re-parse of the corpus |
| Silver | SILVER | Extracted fields with confidence and provenance | Reprocessing is expensive here, so it's isolated |
| Gold | GOLD | Resolved entities and cross-document aggregates | Entity clusters and insights span multiple documents |
| Serve | SERVE | Read projections, semantic view, search, agent | No business logic — only consumption interfaces |
| Config | CONFIG | Templates, prompts, settings | All config is data, not code. Adding a new vendor layout is an INSERT, not a deploy |
| Ops | OPS | Run log, errors, cost ledger, watermarks | Built day one — the platform can't fly blind |

---

## Document Lifecycle (State Machine)

Every document's lifecycle is tracked in `LANDING.DOC_REGISTRY`:

```mermaid
stateDiagram-v2
    [*] --> REGISTERED: Stage file discovered,\nMD5 dedup check passed
    REGISTERED --> PARSED: T20_PARSE\n(CPU only — pypdf2,\npython-docx, text reader)
    PARSED --> ROUTED: T30_ROUTE\n(fingerprint → regex →\nsource hint → model fallback)
    ROUTED --> EXTRACTED_DET: DETERMINISTIC lane
    ROUTED --> EXTRACTED_MODEL: MODEL lane
    EXTRACTED_DET --> VALIDATED: T40A/T40B run concurrently
    EXTRACTED_MODEL --> VALIDATED
    VALIDATED --> RESOLVED: PASS
    VALIDATED --> VALIDATE_FAILED: FAIL
    VALIDATE_FAILED --> VALIDATED: escalate to model\n(one retry)
    VALIDATE_FAILED --> MANUAL: max attempts reached
    RESOLVED --> COMPLETE: Entity resolution —\nembeddings → cosine\nblocking → LLM adjudication
    COMPLETE --> [*]
    MANUAL --> [*]: Human review

    note right of REGISTERED
        Dead-letter at MAX_ATTEMPTS (default 3).
        Every transition logged in OPS.RUN_LOG.
    end note
```

Dead-letter at `MAX_ATTEMPTS` (default 3). Every state transition is logged in `OPS.RUN_LOG`.

---

## Pipeline Task DAG

All 11 tasks run in the OPS schema, orchestrated as a Snowflake task DAG:

```mermaid
flowchart LR
    T10["T10_REGISTER\n(15 min)"] --> T20["T20_PARSE\n(WH_PARSE)"]
    T20 --> T30["T30_ROUTE\n(WH_TRANS)"]
    T30 --> T40A["T40A_EXTRACT_DET\n(WH_TRANS)"]
    T30 --> T40B["T40B_EXTRACT_MODEL\n(WH_AI)"]
    T40A --> T50["T50_VALIDATE"]
    T40B --> T50
    T50 -->|on fail| T40BR["T40B_RETRY\n(escalation)"]
    T40BR --> T50R["T50_RETRY"]
    T50 --> T60["T60_RESOLVE\n(WH_AI)"]
    T50R --> T60
    T60 --> T70["T70_AGGREGATE\n(WH_AI)"]

    T90["T90_METRICS\n(independent, daily 06:00 UTC)"]

    classDef ai fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    classDef indep fill:#f1f3f4,stroke:#5f6368,color:#1a1a1a,stroke-dasharray: 4 3;
    class T40B,T40BR,T60,T70 ai;
    class T90 indep;
```

**Key design decisions:**

- Streams + Tasks, not Dynamic Tables where models are called (non-deterministic functions force full refresh, which re-bills the entire corpus)
- Bounded batches everywhere (`BATCH_SIZE_PARSE=500`, `BATCH_SIZE_EXTRACT=200`) to prevent a bad batch from burning the monthly budget
- Concurrent extraction lanes (T40A and T40B run in parallel after routing)

---

## Routing Economics

This is the most important part of the architecture. The router decides which lane each document takes, and that decision determines cost.

```mermaid
flowchart TD
    DOC["INCOMING DOCUMENT"]
    DOC --> SF["Structural Fingerprint\n(weighted) — free"]
    DOC --> RA["Regex Anchors\n(pattern) — free"]
    DOC --> SH["Source Hint\n(path) — free"]

    SF --> SCORE["Score = 50% structural\n+ 40% regex + 10% hint"]
    RA --> SCORE
    SH --> SCORE

    SCORE -->|score >= 0.6| DET["DETERMINISTIC LANE\n~75% docs · $0 LLM cost"]
    SCORE -->|score < 0.6| MODEL["MODEL LANE (LLM call)\n~25% docs · $0.0002/doc"]

    classDef free fill:#e6f4ea,stroke:#34a853,color:#1a1a1a;
    classDef det fill:#e8f0fe,stroke:#4285f4,color:#1a1a1a;
    classDef paid fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    class SF,RA,SH free;
    class DET det;
    class MODEL paid;
```

Signals are evaluated in cost order: structural first (free), regex second (free), source hint third (free), and only when all fail does the document go to the model lane ($).

---

## Warehouse Isolation

Each pipeline stage has its own warehouse for cost attribution without query-tag forensics:

```mermaid
flowchart TB
    subgraph WH1["WH_IDP_INGEST (XS)"]
        T1["T10: Register"]
    end
    subgraph WH2["WH_IDP_PARSE (M, 1-4 nodes)"]
        T2["T20: Parse (CPU only)"]
    end
    subgraph WH3["WH_IDP_TRANSFORM (S)"]
        T3["T30: Route"]
        T3b["T40A: Det. extract"]
        T3c["T50: Validate"]
        T3d["T90: Metrics"]
    end
    subgraph WH4["WH_IDP_AI (M)"]
        T4["T40B: Model extract"]
        T4b["T60: Resolve"]
        T4c["T70: Aggregate"]
    end

    M1["Monitor: 50cr (shared)"] -.-> WH1
    M1 -.-> WH2
    M1 -.-> WH3
    M2["Monitor: 25cr AI (tighter)"] -.-> WH4

    classDef ai fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    classDef mon fill:#f1f3f4,stroke:#5f6368,color:#1a1a1a,stroke-dasharray: 3 3;
    class WH4,T4,T4b,T4c ai;
    class M1,M2 mon;
```

The AI warehouse gets its own tighter resource monitor (25 credits, suspends at 90%) because it's the only one that can run away. See [[#Resource Monitors]] for budget details and [[#Pipeline Task DAG]] for which task runs on which warehouse.

---

## Extraction & Validation with Escalation

The extraction system uses a two-lane architecture with escalation:

```mermaid
flowchart TD
    ROUTED["ROUTED DOCUMENT"]
    ROUTED --> DET["DETERMINISTIC\nTemplate regex, UDTF per field\nConfidence = 1/distinct_vals"]
    ROUTED --> MOD["MODEL LANE\nAI_COMPLETE, JSON parse\nConfidence = 0.7 (fixed)"]

    DET --> VAL["VALIDATE\nRules from CONFIG.TEMPLATES:\nREQUIRED · ARITHMETIC · RANGE · FORMAT"]
    MOD --> VAL

    VAL -->|PASS| OK["VALIDATED"]
    VAL -->|FAIL| CHECK{"Was it\nDET lane?"}
    CHECK -->|YES| ESC["ESCALATE to MODEL\n(one retry)"]
    CHECK -->|NO| MQ["MANUAL_QUEUE entry\n(human review)"]
    ESC --> OK

    classDef det fill:#e8f0fe,stroke:#4285f4,color:#1a1a1a;
    classDef model fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    classDef pass fill:#e6f4ea,stroke:#34a853,color:#1a1a1a;
    classDef fail fill:#fef7e0,stroke:#f9ab00,color:#1a1a1a;
    class DET det;
    class MOD,ESC model;
    class OK pass;
    class MQ fail;
```

This is escalation, not one-shot routing. Deterministic first; validation failure retries on the model lane once; second failure goes to a human. This permits an aggressive deterministic threshold without an accuracy cost.

---

## Entity Resolution

Resolves "Zephyr Solutions LLC", "ZEPHYR SOLUTIONS INC", and "Zephyr Soln." into a single canonical entity:

```mermaid
flowchart TD
    NAMES["All entity names from\nvendor_name, party_a, party_b"]
    NAMES --> EMBED["EMBED_TEXT_768\n(vectorize all names)\nsnowflake-arctic-embed-m-v1.5"]
    EMBED --> BLOCK["COSINE BLOCKING\nsimilarity >= 0.85\n(upper triangle only, avoids N² model calls)"]

    BLOCK -->|sim >= 0.97| AUTO["AUTO_MERGE\n(skip LLM)"]
    BLOCK -->|0.85 <= sim < 0.97| ASK["LLM ASK\nAI_COMPLETE: 'SAME or DIFFERENT?'"]

    AUTO --> CLOSURE["TRANSITIVE CLOSURE\nUnion-find, depth-capped at 4\n→ ENTITY_CLUSTERS (canonical = shortest name)"]
    ASK --> CLOSURE

    classDef free fill:#e6f4ea,stroke:#34a853,color:#1a1a1a;
    classDef paid fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    class AUTO free;
    class ASK paid;
```

---

## Serve Layer & AI Agent

The serve layer provides three interfaces to the processed data:

```mermaid
flowchart TB
    subgraph SERVE["SERVE SCHEMA"]
        VIEWS["SQL Views\nV_DOCUMENTS · V_INVOICE_FIELDS\nV_CONTRACT_FIELDS · V_OPS_DASHBOARD\nV_ENTITIES"]
        SEM["IDP_SEMANTIC_VIEW\n(Cortex Analyst)\n6 tables, 16 dims, 9 facts,\n7 metrics, with synonyms"]
        SEARCH["IDP_DOC_SEARCH\n(Cortex Search)\nFull-text over DOC_PARSED\n1-hour lag"]
        AGENT["IDP_AGENT — 'IDP Assistant'\n(on CoWork)\nRoutes: Numeric→Analyst,\nContent→Search, Both→Reconcile\nGrounded: cites doc_ids, flags low\nconfidence, refuses when data not held"]

        VIEWS --- SEM
        SEM --> AGENT
        SEARCH --> AGENT
    end

    AGENT --> COWORK["Snowflake CoWork (published)\nBusiness users interact here"]

    classDef data fill:#e8f0fe,stroke:#4285f4,color:#1a1a1a;
    classDef agent fill:#fce8e6,stroke:#d93025,color:#1a1a1a;
    classDef ui fill:#e6f4ea,stroke:#34a853,color:#1a1a1a;
    class VIEWS,SEM,SEARCH data;
    class AGENT agent;
    class COWORK ui;
```

`SERVE.IDP_AGENT` routes structured/numeric/aggregate questions to **Cortex Analyst** (via `IDP_SEMANTIC_VIEW`) and clause-level/free-text questions to **Cortex Search** (via `IDP_DOC_SEARCH`). Access it in Snowsight under **AI & ML > Agents**.

---

## RBAC Model

```mermaid
flowchart TD
    ACC["ACCOUNTADMIN"] --> ADM["IDP_ADMIN\nPlatform administrator"]
    ADM --> ENG["IDP_ENGINEER"]
    ADM --> SVC["IDP_SERVICE\nOwns tasks — survives\npeople leaving"]
    ENG --> OPR["IDP_OPERATOR\nDay-to-day operations"]
    OPR --> ANL["IDP_ANALYST\nRead-only BI consumer"]

    classDef svc fill:#fef7e0,stroke:#f9ab00,color:#1a1a1a;
    class SVC svc;
```

Tasks are owned by `IDP_SERVICE`, never a named user, so the pipeline survives someone leaving the team.

| Role | Purpose |
|---|---|
| `IDP_ADMIN` | Platform administrator |
| `IDP_ENGINEER` | Developer and template author — writes rows into [[#Configuration as Data]] |
| `IDP_SERVICE` | Owns tasks and pipeline objects — see [[#Pipeline Task DAG]] |
| `IDP_OPERATOR` | Day-to-day operations — works the `MANUAL` queue from [[#Document Lifecycle (State Machine)]] |
| `IDP_ANALYST` | Read-only analytics consumer — reads [[#Serve Layer & AI Agent]] views |

---

## Configuration as Data

Adding a new document type is an INSERT, not a code deployment:

| Table | What It Controls |
|---|---|
| `CONFIG.TEMPLATES` | Routing anchors, field extraction regexes, validation rules per doc class |
| `CONFIG.PROMPTS` | Versioned LLM prompts (only `SP_EXTRACT_MODEL` calls a model) |
| `CONFIG.SETTINGS` | Thresholds, batch sizes, model selection, feature flags |

The model lane is swappable behind one setting: `MODEL_LANE_IMPL` takes `CORTEX`, `EXTERNAL`, or `DISABLED`. Only one procedure (`SP_EXTRACT_MODEL`) calls a model. Everything downstream depends solely on the `DOC_FIELDS` contract.

---

## Known Limitations

1. **No OCR path enabled** — PDF pages with <50 chars/page are flagged `needs_ocr=TRUE` but not processed
2. **Entity clustering is depth-capped** at 4 levels of transitive closure (see [[#Entity Resolution]])
3. **Regex templates are locale-specific** — date patterns assume MM/DD/YYYY format
4. **No reviewer UI** — the `MANUAL` queue from [[#Document Lifecycle (State Machine)]] requires direct SQL
5. **Validation rule JSON serialization** — `SP_VALIDATE` has edge cases with ARRAY→STRING conversion for validation rules; some documents may need manual advancement
6. **ACCOUNT_USAGE lags** the cost ledger by hours
7. **Model extraction JSON parsing** — LLM responses from the model lane (see [[#Routing Economics]]) occasionally fail JSON parsing, causing extraction errors

---

## Resource Monitors

- `IDP_DEV_MONITOR`: 50 credits/month, covers all warehouses except AI (see [[#Warehouse Isolation]])
- `IDP_DEV_AI_MONITOR`: 25 credits/month, `WH_IDP_AI` only, tighter limits — suspends at 90%

---

## Current Deployment Numbers

| Metric | Value |
|---|---|
| Documents processed | 8 (synthetic corpus) |
| Deterministic lane share | 75% |
| Cost per document (LLM) | ~$0.00007 |
| Projected 1K docs/month | ~$0.07 LLM + ~3 warehouse credits |
| Smoke test assertions | 7/7 PASS |
| Agent acceptance tests | 6/6 PASS |
| Task DAG | 11 tasks, all STARTED |
| Resource monitors | 50cr total / 25cr AI |
