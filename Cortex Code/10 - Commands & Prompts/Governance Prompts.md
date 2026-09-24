---
type: cortex-code-prompts
title: "Governance Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - governance
---

# Governance Prompts

**Type:** Copy-Ready Governance Prompts
**Last Reviewed:** 2026-09-24

---

## Security & Access Control Audits

`✅ COPY-READY`

### 1. Account Governance Inventory
```text
Build a governance inventory of this Snowflake account.
For every important database, schema, and object, determine:
- owner
- object type
- row count where available
- approximate size
- last activity
- grants
- dependent objects
- sensitivity indicators
- tags/classification where available
Highlight missing governance metadata rather than inventing values.
```

### 2. Forensic Access Control Investigation
```text
Perform a Snowflake access-control investigation. Identify users and roles with potentially excessive privileges, trace inherited privileges through role hierarchies, identify sensitive objects that are broadly accessible, and explain each finding with the exact grants that support it. Do not modify anything.
```

### 3. Privilege Path Forensic Analysis
```text
Pick the most privileged non-admin role available to me. Reconstruct exactly what this role can access, including inherited privileges. Identify the most sensitive objects it can reach and explain the privilege path that gives it access. Do not make any changes.
```
