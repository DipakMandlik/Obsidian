---
type: cortex-code-prompts
title: "Planning Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - planning
---

# Planning Prompts

**Type:** Copy-Ready Planning Prompts
**Last Reviewed:** 2026-09-24

---

## Bounded Planning & Autonomous Boundary

`✅ COPY-READY`

### 1. Discover, Plan, and Wait for Approval
```text
Find one real, reproducible engineering problem in this Snowflake environment.
First investigate and prove the problem.
Then create a remediation plan.
Do not change anything yet.
After presenting the plan, wait for approval.
```

### 2. Autonomous vs Human Boundary Definition
```text
You have now investigated the environment.
Take your most significant confirmed finding.
Without asking me to explain the implementation, determine everything required to remediate it safely.
Inspect the relevant code, objects, dependencies, and permissions.
Create a step-by-step remediation plan.
Identify every risk associated with the change.
Then tell me exactly which actions you can safely execute yourself and which actions require human approval.
```
