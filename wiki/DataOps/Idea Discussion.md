# Pi-Monitor — Databricks Cost Intelligence Platform

> **Pi-Monitor is a Databricks-native AI FinOps platform that transforms Databricks billing and resource telemetry into explanations, optimization recommendations, and measurable cost savings.**

---

## Core Problem

Databricks provides extensive billing, compute, job, query, and resource telemetry. However, understanding a cost increase often requires engineers to manually investigate multiple sources.

**Example:** *"Why did our Databricks cost increase yesterday?"*

A platform engineer may need to investigate: Billing Usage → Jobs → Clusters → SQL Warehouses → Runtime Changes → Owners → Historical Usage.

Pi-Monitor brings this information together and converts raw telemetry into **actionable cost intelligence**.

---

## Product Philosophy

| Principle | Description |
|-----------|-------------|
| **Minimal** | Only important information is surfaced |
| **AI-first** | Users investigate their environment conversationally |
| **Context-aware** | The system understands the resource/workload being investigated |
| **Action-oriented** | Every insight leads toward a decision or optimization |
| **Databricks-native** | Data, analytics, AI, deployment stay within the Databricks ecosystem |
| **Enterprise-ready** | Respects Databricks auth, Unity Catalog governance, permissions, auditability |

---

## Core Product Flow

**Observe → Detect → Explain → Recommend → Optimize → Measure**

- **Observe**: Continuously understand usage and cost via System Tables
- **Detect**: Unusual behavior — cost spikes, expensive jobs, idle compute, excessive retries
- **Explain**: Determine *why* and *which resource/owner/workload* caused it
- **Recommend**: Actionable optimizations with estimated financial impact
- **Optimize**: Help platform teams decide *what action* to take
- **Measure**: Track whether the optimization actually reduced cost

---

## Architecture

**Databricks is both the data platform and the application backend.**

No external databases (no PostgreSQL, Supabase, Firebase, MongoDB).

```
User → Pi-Monitor UI → Backend → Databricks → Unity Catalog / System Tables / Delta Tables / Genie
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js / React, TypeScript, Tailwind CSS, shadcn/ui |
| Backend | Python, FastAPI, Databricks SDK, Databricks SQL |
| Data | Databricks System Tables, Unity Catalog, Delta Tables, Databricks SQL |
| AI | Databricks Genie (and other Databricks AI capabilities) |
| Deployment | Databricks Apps |

---

## Product Experiences

Pi-Monitor deliberately avoids 10–15 modules. The initial product has **four** primary experiences:

### 1. Overview

Understand the environment in ~30 seconds.

- Total Spend
- DBU Consumption
- Cost Efficiency
- Potential Savings
- Cost Trend
- Top Cost Drivers
- Important Insights

### 2. Resources

A unified explorer for all Databricks resources: Jobs, Clusters, SQL Warehouses, Pipelines.

Opening a resource shows: **Cost → DBUs → Utilization → History → Issues → Recommendations**

Users can ask AI questions about the selected resource.

### 3. Insights

An intelligent FinOps inbox — Pi-Monitor proactively surfaces important events.

**Example:** *Cost Anomaly — ETL_Pipeline_Prod increased cost by 32% yesterday. Likely cause: repeated executions after an upstream failure. Estimated additional spend: $320.*

**Example:** *Idle Compute — Three clusters accumulated 11.4 idle hours. Estimated savings: $680/month.*

Insights are prioritized by financial impact, severity, and confidence.

### 4. AI Assistant

Integrates with **Databricks Genie** for natural-language investigation.

**Examples:**

- *Why did our cost increase yesterday?*
- *Which jobs consumed the most DBUs this week?*
- *Find idle compute.*
- *What changed compared with last week?*
- *Where can we save money?*

Workflow: **Question → Investigation → Databricks Query → Evidence → Explanation → Recommendation**

The AI is context-aware — when viewing a resource like `ETL_Pipeline_Prod`, the user can ask *"Why is this expensive?"* and Pi-Monitor investigates billing, execution history, DBU consumption, failures, retries, and historical behavior automatically.

---

## Data Strategy

```
Databricks System Tables → Pi-Monitor Analytics Layer → Backend Services → UI / AI
```

### Delta Tables in Unity Catalog

**Analytics:**
- `pi_monitor.analytics.daily_cost`
- `pi_monitor.analytics.resource_cost`
- `pi_monitor.analytics.job_cost`
- `pi_monitor.analytics.compute_utilization`
- `pi_monitor.analytics.cost_anomalies`
- `pi_monitor.analytics.optimization_opportunities`

**Application:**
- `pi_monitor.application.settings`
- `pi_monitor.application.insight_status`
- `pi_monitor.application.preferences`

**AI:**
- `pi_monitor.ai.conversations`
- `pi_monitor.ai.messages`
- `pi_monitor.ai.feedback`

---

## Key Differentiator

| Traditional Tools | Pi-Monitor |
|-------------------|-----------|
| How much did we spend? | What changed? Why? |
| — | Which resource caused it? |
| — | Is the behavior expected? |
| — | What should we do about it? |
| — | How much could we save? |

Pi-Monitor transforms from a **cost dashboard** into a **Databricks Cost Intelligence and Decision Platform**.

---

## Product Scenario

> **Yesterday: Databricks spend increased by 18.6%.**

Pi-Monitor automatically investigates:

**Primary Driver:** ETL_Pipeline_Prod — DBU consumption +42%

**Cause:** The pipeline executed 14 times instead of 8 due to repeated retries.

**Additional Spend:** $420

**Recommendation:** Review retry policy and pipeline scheduling.

**Potential Savings:** $1,480/month.

> **Don't make users analyze dashboards to find problems. Pi-Monitor finds the problem, explains why it happened, and tells them what to do next.**