---
type: cortex-code-guide
title: "How to Choose a Cortex Code Skill"
category: core-concepts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - decision-matrix
  - skill-selection
aliases:
  - "Skill Selection Guide"
  - "Which Skill Should I Use"
---

# How to Choose a Cortex Code Skill

> *"I have this problem. Which skill should I use?"*

This guide provides a deterministic decision matrix and natural-language triggers to select the exact Cortex Code skill, plugin, or specialist agent for any task.

---

## Skill Decision Matrix

| Problem / Objective | Recommended Skill / Plugin | Why This Choice | Note Link |
|---|---|---|---|
| **Audit agent permissions & fix missing grants** | `check-agent-access` | Introspects Cortex Agent specifications and checks USAGE on all dependencies. | [[Skill - Audit Cortex Agent Access]] |
| **Migrate Data Clean Room from V1 to V2** | `dcr-v1-to-v2` | Maps SAMOOHA API to Collaboration API and converts JinjaSQL templates. | [[Skill - DCR V1 to V2 Migration]] |
| **Share Snowflake tables into SAP BDC** | `manage-zerocopy-sapbdc` | Generates minimal, SDK-compatible Core Schema Notation (CSN) zero-copy connectors. | [[Skill - SAP BDC Zero-Copy Connector]] |
| **Plan MLOps promotion, CI/CD, or governance** | `mlops` | Evaluates MLOps maturity and selects promotion patterns (Code/Model/Hybrid). | [[Skill - Plan and Run MLOps]] |
| **Generate knowledge graph / ontology from schema** | `ontology-stack-builder` | Implements 7-phase ontology layer, creating KG_NODE and KG_EDGE views. | [[Skill - Build Ontology Stack on Snowflake]] |
| **Private connectivity from SPCS to AWS RDS** | `openflow-spcs-privatelink` | Sets up NLB, Endpoint Service, and EAI for secure private ingestion. | [[Skill - OpenFlow PrivateLink Setup]] |
| **Walk through a Snowflake Quickstart tutorial** | `quickstart-guide` | Ingests Quickstart markdown from GitHub and guides interactive building. | [[Skill - Learn Snowflake Quickstarts]] |
| **Design or refactor Snowflake RBAC** | `rbac` | Creates tiered functional and access roles (DBAR, SCAR, WAR). | [[Skill - Snowflake RBAC Patterns]] |
| **Audit local skill before PR to Snowflake Labs** | `review-skill-sflabs` | Runs pre-PR catalog readiness checks on local skill directories. | [[Skill - Review Skill for SF Labs]] |
| **Model complex metrics in Semantic Views** | `semantic-view-patterns` | Implements 25 semantic modeling patterns (time intelligence, windowing, ASOF). | [[Skill - Apply Semantic View Patterns]] |
| **Official Snowflake documentation lookup** | `snowflake-docs` | Searches documentation via Cortex Knowledge Extension / Cortex Search. | [[Skill - Snowflake Docs]] |
| **Azure ADLS Snowpipe Disaster Recovery** | `snowpipe-bcdr` | Manages failover, catchup ingestion, and failback across Azure regions. | [[Skill - Snowpipe BCDR on Azure]] |
| **Prepare for SnowPro certification exam** | `snowpro-study` | Generates domain deep-dives, flashcards, and exam practice questions. | [[Skill - SnowPro Certification Study]] |
| **Deploy pre-built industry solution accelerators** | `solutions-installer` | Installs Snowflake Industry Solutions manifests into account. | [[Skill - Install Snowflake Solutions]] |
| **Migrate Spark/PySpark to Snowflake (Default)** | `spark-migration/snowpark-connect` | Preserves PySpark API surface while executing on Snowflake compute. | [[Skill - Snowpark Connect]] |
| **Rewrite PySpark to native Snowpark DataFrame API** | `spark-migration/snowpark-api` | Uses SMA CLI to rewrite code and DVP to validate parity. | [[Skill - Snowpark API Migration]] |
| **Convert Databricks Notebook to Snowflake** | `spark-migration/snowflake-notebook-migration` | Converts notebooks, widgets, and multi-language cells. | [[Skill - Databricks Notebook Migration]] |
| **Structured SDLC with EARS requirements** | `spec-driven` | Enforces specification approval before any code or SQL is written. | [[Skill - Spec-Driven Development]] |
| **Safely swap two Snowflake databases** | `swap-databases` | Bidirectional rename with object sync, grant mirroring, and zerocopy swap. | [[Skill - Swap Snowflake Databases]] |
| **Audit account against 5 WAF pillars** | `well-architected-framework-assessment` | Evaluates Security, Reliability, Cost, Ops, and Performance. | [[Skill - Well-Architected Framework Assessment]] |
| **Multi-session workflow harness with durable memory** | `xo` (Plugin) | OODA loop stages, session diary, task index, and subagents. | [[Plugin - XO]] |

---

## Natural Language Triggers

- **"When I need to migrate Spark workloads"** -> [[Skill - Snowpark Connect]] (or [[Skill - Snowpark API Migration]] if rewriting)
- **"When I need to perform a WAF assessment"** -> [[Skill - Well-Architected Framework Assessment]]
- **"When I need spec-driven implementation"** -> [[Skill - Spec-Driven Development]]
- **"When I need XO workflows and persistent memory"** -> [[Plugin - XO]]
- **"When I need to check role access to Cortex Agents"** -> [[Skill - Audit Cortex Agent Access]]
- **"When I need to swap database names without downtime"** -> [[Skill - Swap Snowflake Databases]]
- **"When I need to set up disaster recovery for Snowpipe"** -> [[Skill - Snowpipe BCDR on Azure]]
- **"When I need to connect Snowflake with SAP BDC"** -> [[Skill - SAP BDC Zero-Copy Connector]]

---

## Related Notes

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[CoCo Command Cheat Sheet]]
