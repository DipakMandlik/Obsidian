---
type: cortex-code-workflow
title: "Workflow - XO Operator OODA Loop"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/
category: workflow-harness
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - xo
  - ooda
aliases:
  - "XO OODA Workflow"
---

# Workflow - XO Operator OODA Loop

**Type:** Operator Workflow Framework
**Source:** Snowflake-Labs/coco-skills
**Primary Plugin:** [[Plugin - XO]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

XO organizes multi-step AI engineering work around a structured OODA loop (Observe -> Orient -> Decide -> Act), ensuring high auditability, persistent memory, and strict quality gates.

---

## The Four Stages

```text
OBSERVE               ORIENT              DECIDE               ACT
+-------------+      +-------------+      +-------------+      +-------------+
| Survey      | ---> | Distil      | ---> | Spec        | ---> | Provision   |
| Analyse     |      | Workshop    |      | Plan        |      | Build       |
| Recall      |      |             |      | Review      |      | Prove & Ship|
+-------------+      +-------------+      +-------------+      +-------------+
```

- **Observe:** Locate facts, survey existing code ([[Reference - XO Observe Survey]]), deep investigation ([[Reference - XO Observe Analyse]]), memory recall ([[Reference - XO Observe Recall]]).
- **Orient:** Synthesize evidence into a brief ([[Reference - XO Orient Distil]]), explore trade-offs ([[Reference - XO Orient Workshop]]).
- **Decide:** Author specification ([[Reference - XO Decide Spec]]), construct plan ([[Reference - XO Decide Plan]]), conduct cold critique ([[Reference - XO Decide Review]]).
- **Act:** Set up workspace ([[Reference - XO Act Provision]]), bounded implementation ([[Reference - XO Act Build]]), rigorous verification ([[Reference - XO Act Prove]]), deliver changes ([[Reference - XO Act Ship]]).

---

## Related Notes

- [[Plugin - XO]]
- [[Skill - XO Operator Workflow]]
- [[Agent - XO Cold Smart]]
- [[Agent - XO Generator]]
