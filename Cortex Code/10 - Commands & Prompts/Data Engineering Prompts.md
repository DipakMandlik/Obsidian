---
type: cortex-code-prompts
title: "Data Engineering Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - data-engineering
---

# Data Engineering Prompts

**Type:** Copy-Ready Data Engineering Prompts
**Last Reviewed:** 2026-09-24

---

## Performance & Optimization

`✅ COPY-READY`

### 1. Expensive Query Root-Cause Analysis
```text
Analyze recent Snowflake workload and identify the queries that are consuming disproportionate compute. Don't simply rank queries by credits. Determine WHY they are expensive, inspect their SQL, identify inefficient patterns (e.g., cartesian joins, lack of pruning, disk spilling), and propose the smallest code or architecture changes that would reduce cost without degrading correctness.
```

### 2. Unused & Underutilized Resource Discovery
```text
Find Snowflake objects and warehouses that appear to be unused or underutilized. Do not simply use creation dates. Correlate object metadata with query/access history and warehouse workload. Distinguish genuinely unused resources from resources that are infrequently but legitimately used.
```

### 3. Mysterious Data-Quality Issue
```text
Assume the business reports that today's customer numbers are incorrect. You have no information about where the problem originates. Investigate the complete data path, trace dependencies backward from the final analytical objects, identify where the discrepancy originates, and prove your conclusion with SQL evidence.
```
