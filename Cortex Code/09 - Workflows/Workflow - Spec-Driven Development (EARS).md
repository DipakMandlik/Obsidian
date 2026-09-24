---
type: cortex-code-workflow
title: "Workflow - Spec-Driven Development (EARS)"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/spec-driven/
category: sdlc
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - spec-driven
  - ears
aliases:
  - "Spec-Driven SDLC Workflow"
---

# Workflow - Spec-Driven Development (EARS)

**Type:** Software Engineering Workflow
**Source:** Snowflake-Labs/coco-skills
**Primary Skill:** [[Skill - Spec-Driven Development]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

Spec-Driven Development enforces requirements authoring using EARS notation (Easy Approach to Requirements Syntax) and human operator approval gates **before** any code or SQL is generated.

---

## Lifecycle Steps

1. **Requirement Elicitation:** Classify change as Feature, Bugfix, Evolve, or Refactor.
2. **Specification Draft:** [[Skill - Spec-Driven Feature]] or relevant sub-skill drafts EARS specification in `specs/{spec-name}.md`.
3. **Operator Approval Gate:** Human operator reviews and approves the bounding contract.
4. **Implementation:** [[Skill - Spec-Driven Implement]] generates code matching requirements.
5. **Verification & Proof:** Automated tests prove each EARS requirement without scope creep.

---

## Related Notes

- [[Skill - Spec-Driven Development]]
- [[Asset - EARS Spec Template]]
