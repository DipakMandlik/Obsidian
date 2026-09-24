---
type: cortex-code-concepts
title: "Cortex Code Architecture and Core Concepts"
category: core-concepts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - architecture
  - concepts
aliases:
  - "CoCo Architecture"
  - "Core Concepts"
---

# Cortex Code Architecture and Core Concepts

**Type:** Architecture & Concepts Reference
**Source:** Snowflake-Labs/coco-skills
**Last Reviewed:** 2026-09-24

---

## Architectural Principles

1. **Separation of Discovery, Generation, and Audit:**
   - Unsupervised agents fail when the same model explores, generates, and self-approves.
   - Cortex Code enforces role boundaries: Cold explorers gather facts without edit rights; warm generators implement against approved specifications; cold critics verify claims with verifiable evidence.

2. **Durable, Git-Backed Memory:**
   - LLM sessions lose context during summarization and compaction.
   - The XO harness captures decisions in local markdown files (`diary/`, `tasks/`, `notes/`) and commits them to a versioned repository.

3. **Deterministic Evidence Over Inference:**
   - Every operational claim must trace to a line-numbered file read or SQL execution output. Gaps are explicitly stated rather than filled with speculative inferences.

4. **The OODA Loop (Observe -> Orient -> Decide -> Act):**
   - Work is decomposed into modular stages, preventing premature execution before scope is bounded.

---

## Key Component Definitions

- **Skill:** An executable operational recipe declared via `SKILL.md` containing triggers, instructions, and workflows.
- **Plugin:** A bundled package containing a manifest (`plugin.json`), hooks, skills, and specialized agents.
- **Agent:** An independent model role configuration with explicit tool locks and instructions (e.g. [[Agent - XO Bounded Writer]]).
- **Hook:** Deterministic scripts executed on session lifecycle events (`SessionStart`, `UserPromptSubmit`, `PostToolUse`).

---

## Related Notes

- [[How to Choose a Cortex Code Skill]]
- [[Plugin - XO]]
- [[Workflow - XO Operator OODA Loop]]
