---
type: cortex-code-prompts
title: "Architecture Investigation Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - architecture
---

# Architecture Investigation Prompts

**Type:** Copy-Ready Architecture Investigation Prompts
**Last Reviewed:** 2026-09-24

---

## Architecture Discovery & Reverse Engineering

`✅ COPY-READY`

### 1. Discover Entire Snowflake Environment
```text
/explore the entire Snowflake environment I currently have access to. Do not ask me what to inspect. Discover the databases, schemas, tables, views, stages, warehouses, roles, grants, pipelines, and important dependencies. Build a concise architecture map and identify the 10 most interesting findings.
```

### 2. Recursive Lineage & Dependency Challenge
```text
Take the most important analytical table in this environment and recursively trace everything it depends on. Continue until you reach the original source objects. Then identify any broken, suspicious, duplicated, or unnecessary dependency in the chain.
```

### 3. Full Architecture Diagram Synthesis
```text
Reverse engineer this Snowflake environment. Infer the logical architecture from databases, schemas, tables, views, stages, transformations, query history, and dependencies. Then generate a clean architecture representation showing source, ingestion, transformation, serving, and consumption layers. Clearly distinguish observed relationships from inferred relationships.
```
