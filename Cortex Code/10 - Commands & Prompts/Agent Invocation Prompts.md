---
type: cortex-code-prompts
title: "Agent Invocation Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - agents
---

# Agent Invocation Prompts

**Type:** Copy-Ready Agent Prompts
**Last Reviewed:** 2026-09-24

---

## Single Specialist Agent Invocations

`✅ COPY-READY`

### 1. Cold Fast Surveyor (Discovery)
```text
Spawn subagent xo-cold-fast with:
- PERSONA: Surveyor: Existing Patterns
- INPUTS: ./models/**/*.sql
- OUTPUT_PATH: ./tmp/survey_results.md
Survey all existing SQL transformations, locate common CTE patterns, record what you checked, and note any gaps.
```

### 2. Cold Smart Critic (Adversarial Quality Gate)
```text
Spawn subagent xo-cold-smart with:
- PERSONA: Critic
- REFERENCE: ./specs/auth-rbac.md
- INPUTS: ./sql/grants.sql
- OUTPUT_PATH: ./tmp/rbac_critique.md
Verify that the generated SQL grants strictly satisfy the approved RBAC spec. Ground every finding in line-numbered evidence. Issue a clear verdict: APPROVE, REVISE, or BLOCK.
```

### 3. Bounded Writer (Zero-Hallucination Documentation)
```text
Spawn subagent xo-bounded-writer with:
- PERSONA: Drafter
- INPUT_PATH: ./tmp/survey_results.md
- OUTPUT_PATH: ./docs/architecture_summary.md
Draft the architectural summary using strictly the verified facts in the survey brief. Do not invent unmentioned details.
```

### 4. Warm Generator (Bounded Implementation)
```text
Spawn subagent xo-generator with:
- PERSONA: Proposer
- SPECIFICATION: ./specs/auth-rbac.md
- WORKTREE: ./
Implement the required database access roles and schema access roles. Modify only what the specification requires.
```
