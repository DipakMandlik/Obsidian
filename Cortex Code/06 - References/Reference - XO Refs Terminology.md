---
type: cortex-code-reference
title: "Reference - XO Refs Terminology"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/skills/xo/references/refs-terminology.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/skills/xo/references/refs-terminology.md
category: xo-reference
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - reference
  - xo
aliases:
  - "Reference - XO Refs Terminology"
  - "refs-terminology"
---

# Reference - XO Refs Terminology

**Type:** Reference
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo/skills/xo/references/refs-terminology.md`
**Source URL:** [refs-terminology.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/skills/xo/references/refs-terminology.md)
**Category:** XO OODA Reference Guidance
**Last Reviewed:** 2026-09-24

---

## Purpose

Provides official operational guidance and procedural requirements for the **Refs Terminology** stage of the XO operator harness in Cortex Code.

---

## Canonical Operational Guidance

`📘 REFERENCE` `✅ COPY-READY`

---
name: refs-terminology
description: "Standard terminology across xo-* skills. Defines the lexicon for artifacts, actions, and concepts."
---

# XO Cortex Terminology

Standard terms used across xo-* skills. Consistent terminology helps agents understand what artifacts to produce and helps operators know what to expect.

## Artifacts

| Term | Nature | Audience | Purpose |
|------|--------|----------|---------|
| **Brief** | Summary for decision-making | Operator | Present findings, recommendations, or status for approval/awareness |
| **Spec** | Intent/requirements document | Agent (current or future) | Define what should be built and why |
| **Findings** | Raw discoveries, not yet synthesised | Internal/working | Capture what was learned during investigation |
| **Notes** | Persistent working memory | Agent recovery | Accumulated understanding of a work item |
| **Diary** | Session activity log | Agent recovery | What happened, when, organised by session |
| **Task** | Work item state file | Agent recovery | Current state, next action, status of a work item |

### Artifact Hierarchy

```
Findings → Brief → Operator decision
   ↓
Notes (persisted for recovery)
```

- **Findings** are raw — what you discovered
- **Brief** synthesises findings for the operator — what it means and what you recommend
- **Notes** persist understanding across context resets — what the agent needs to remember

## Actions as Verbs

These terms describe processes, not outputs:

| Verb | Meaning | Output |
|------|---------|--------|
| **Assess** | Evaluate against criteria | Brief |
| **Analyse** | Investigate in depth | Findings (then Brief) |
| **Review** | Examine for correctness/quality | Brief |
| **Investigate** | Explore to understand | Findings |
| **Validate** | Confirm against reference | Brief |

Example: "Assess the PR" → process is assessment → output is a brief.

## Roles

| Term | Meaning |
|------|---------|
| **Operator** | The human directing the work (senior in the relationship) |
| **Agent** | The AI performing the work (reports to operator) |
| **Orchestrator** | The top-level agent coordinating subagents |
| **Subagent** | A specialist agent dispatched for a specific task |

## Recording Terms

| Term | Meaning |
|------|---------|
| **Savepoint** | The act of recording state across all three layers (diary + task + notes) |
| **Context state** | The stored context fullness percentage (triggers savepoint reminders) |
| **Checkpoint** | VS Code's session rollback feature (not an xo-cortex term) |

## Workflow Terms

| Term | Meaning |
|------|---------|
| **Fast-path** | Direct action without full deliberative process |
| **Deliberative** | Multi-agent process (proposer/critic/validator/etc.) for quality assurance |
| **Empirical** | Code-first approach — build then document |
| **Verified** | Tested through execution, not just static analysis |

---

## Related Notes

- [[Plugin - XO]]
- [[Skill - XO Operator Workflow]]
- [[Workflow - XO Operator OODA Loop]]
- [[Cortex Code Skills Master Index]]

---

## Official Repository Knowledge

Directly extracted from `plugins/xo/skills/xo/references/refs-terminology.md`. This forms part of the authoritative stage reference library for Cortex Code operator workflows.
