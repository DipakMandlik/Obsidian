---
type: cortex-code-cheat-sheet
title: "CoCo Command Cheat Sheet"
category: reference
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - cheat-sheet
  - commands
aliases:
  - "Cortex Code Cheat Sheet"
  - "CoCo Cheat Sheet"
---

# CoCo Command Cheat Sheet

**Type:** Verified Command Reference
**Source:** Snowflake-Labs/coco-skills & Official Cortex Code Runtime
**Last Reviewed:** 2026-09-24

---

## 1. CLI Commands & Built-In Controls

| Command | Purpose | Example |
|---|---|---|
| `/skills` | List all loaded and available skills | `/skills` |
| `/help` | Display Cortex Code built-in help and commands | `/help` |
| `/clear` | Reset current conversational context | `/clear` |
| `/status` | View active plugin and connection status | `/status` |

---

## 2. Plugin Setup & Management

```bash
# Install XO plugin from local clone
coco-skills/plugins/xo/setup.sh install

# Initialize git-backed memory vault and register XO
coco-skills/plugins/xo/setup.sh init

# Check XO plugin health and path resolution
coco-skills/plugins/xo/setup.sh status

# Migrate XO between user-profile and workspace
coco-skills/plugins/xo/setup.sh migrate --target workspace
```

---

## 3. Skill Activation Patterns

`✅ COPY-READY`

```text
# Architecture & Governance
"Run a Well-Architected Framework assessment for this account"
"Help me design an RBAC hierarchy for my Snowflake account"
"Audit which Cortex Agents a role can access and fix any gaps"

# Migration & Workloads
"Assess my PySpark workload in ./jobs for Snowpark Connect compatibility"
"Migrate PySpark to Snowpark Connect in ./src/pipeline"
"Convert Spark notebook ./notebooks/etl.ipynb to Snowflake Notebook"
"Swap databases PROD_DB_OLD and PROD_DB_NEW"

# Development & SDLC
"Start a feature spec for customer churn prediction in EARS notation"
"Fix bug in revenue aggregation using spec-driven bugfix"
"Search official Snowflake documentation for Dynamic Tables syntax"
```

---

## 4. Agent Invocation Syntax

```text
# Delegate to specialized XO Subagents
"Spawn xo-cold-fast to survey existing schema patterns in ./models"
"Spawn xo-cold-smart as a Critic to evaluate implementation against spec"
"Spawn xo-generator to implement the approved specification in ./src"
"Spawn xo-bounded-writer to compile the findings brief into documentation"

# Snowpark Connect Specialists
"Dispatch scos-pyspark-analyzer on conversion_root"
"Dispatch scos-pyspark-fixer for chunk 1"
"Dispatch scos-pyspark-scos-runner to execute Phase B validation"
```

---

## 5. Multi-Agent & Orchestration Commands

```text
Investigate the current Snowflake environment using specialist subagents.
Create separate investigators for:
1. Security
2. FinOps
3. Performance
4. Data Quality
5. Governance
6. Architecture
Allow them to investigate independently, reconcile conflicting evidence, and return verified findings.
```

---

## Related Notes

- [[Skill Activation Commands]]
- [[Agent Invocation Prompts]]
- [[Multi-Agent Prompts]]
- [[Cortex Code Skills Master Index]]
