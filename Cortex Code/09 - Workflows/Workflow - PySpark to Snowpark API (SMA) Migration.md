---
type: cortex-code-workflow
title: "Workflow - PySpark to Snowpark API (SMA) Migration"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/spark-migration/snowpark-api/
category: migration
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - sma
  - snowpark-api
aliases:
  - "PySpark to Snowpark API Workflow"
---

# Workflow - PySpark to Snowpark API (SMA) Migration

**Type:** End-to-End Migration Workflow
**Source:** Snowflake-Labs/coco-skills
**Primary Skill:** [[Skill - Snowpark API Migration]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

Used when rewriting PySpark scripts into native Snowflake Snowpark Python DataFrame API (`snowflake.snowpark`). This path utilizes the Snowpark Migration Accelerator (SMA CLI) and the Deterministic Validation Pipeline (DVP).

---

## Phases

1. **Conversion:** [[Skill - Migrate PySpark to Snowpark API]] runs SMA CLI to convert code.
2. **EWI Resolution:** [[Skill - DVP EWI Fixer]] automatically resolves Errors, Warnings, and Info markers.
3. **Stage Conversion:** [[Skill - Stage Conversion Embedded File Paths]] converts local/cloud paths to Snowflake Stages (`@stage/...`).
4. **Validation Pipeline:** [[Skill - DVP Orchestrator]] synthesizes test schemas, generates synthetic test datasets, and runs parity tests.
5. **Reporting:** [[Skill - SMA Dashboard Generator]] compiles the interactive HTML readiness dashboard.

---

## Related Notes

- [[Skill - Snowpark API Migration]]
- [[Skill - DVP Orchestrator]]
- [[Asset - SMA Executive Dashboard Template]]
