---
type: cortex-code-prompts
title: "Migration Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - migration
---

# Migration Prompts

**Type:** Copy-Ready Migration Prompts
**Last Reviewed:** 2026-09-24

---

## Spark & Legacy Workload Migration

`✅ COPY-READY`

### 1. Rapid Spark Assessment
```text
Scan all PySpark scripts in ./spark_pipelines. Produce a compatibility score for Snowpark Connect (SCOS), identify unsupported Spark constructs, estimate conversion effort, and generate a migration readiness report.
```

### 2. End-to-End PySpark to SCOS Migration
```text
Migrate the PySpark workload in ./etl to Snowflake Snowpark Connect. Preserve the native PySpark DataFrame API calls. Run automated compatibility fixing, generate synthetic test datasets, execute two-phase validation against Snowflake, and report dataset parity.
```

### 3. Databricks Notebook Conversion
```text
Convert the Databricks notebook ./notebooks/customer_analytics.ipynb into a native Snowflake Notebook. Convert Databricks widgets to Snowflake session parameters and configure stage-backed code execution.
```
