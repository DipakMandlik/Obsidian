---
type: cortex-code-prompts
title: "Multi-Agent Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - multi-agent
---

# Multi-Agent Prompts

**Type:** Copy-Ready Multi-Agent Orchestration Prompts
**Last Reviewed:** 2026-09-24

---

## Killer Multi-Agent Investigation Prompts

`✅ COPY-READY`

### 1. Multi-Specialist Account Audit
```text
Investigate the current Snowflake environment using specialist subagents.

Create separate investigators for:
1. Security (privileges, public objects, role hierarchies)
2. FinOps (expensive queries, warehouse sizing, idle clusters)
3. Performance (spillover to remote storage, partition pruning)
4. Data Quality (freshness, null spikes, schema drift)
5. Governance (tags, classifications, masking policies)
6. Architecture (unnecessary dependencies, circular views)

Allow them to investigate independently. Then act as the lead investigator and reconcile their findings. If two investigators disagree, do not choose one arbitrarily; run additional SQL investigation to determine which conclusion is supported by evidence. Return only findings supported by verifiable SQL evidence.
```

### 2. Proposer / Critic / Verifier Triad
```text
Execute this implementation using an isolated 3-agent triad:
1. Proposer: Drafts the DDL and pipeline changes in a local scratch file.
2. Critic: Evaluates the draft against security standards and our RBAC specification.
3. Verifier: Executes dry-run validation against Snowflake test schemas and verifies output parity.
Do not apply changes to production until the Verifier produces a PASS verdict.
```
