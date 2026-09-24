---
type: cortex-code-prompts
title: "Debugging Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - debugging
---

# Debugging Prompts

**Type:** Copy-Ready Debugging Prompts
**Last Reviewed:** 2026-09-24

---

## Forensic Incident Investigation

`✅ COPY-READY`

### 1. "I Don't Know What Happened" Forensic Reconstruction
```text
Something changed in this Snowflake environment recently and I don't know what.
Investigate recent changes across objects, permissions, workloads, schemas, and data pipelines. Identify unusual changes, correlate them with query and access history, and reconstruct the most likely sequence of events.
Separate confirmed evidence from inference.
```

### 2. Query Failure & Compiler Error Diagnosis
```text
Inspect the failed query ID in Snowflake query history. Extract the execution profile, determine why the query failed (e.g. timeout, memory limit, schema mismatch), and inspect the underlying table statistics to determine the root cause.
```
