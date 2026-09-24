---
type: cortex-code-asset
title: "Asset - EARS Spec Template"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/spec-driven/references/ears-notation.md
category: template
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - asset
  - template
  - ears
  - spec-driven
---

# Asset - EARS Spec Template

**Type:** Specification Template
**Category:** SDLC Spec-Driven Development
**Notation:** Easy Approach to Requirements Syntax (EARS)
**Last Reviewed:** 2026-09-24

---

## Purpose

Standardized specification template using EARS (Easy Approach to Requirements Syntax). Guarantees unambiguous requirements before implementing features, bugfixes, or refactoring in Cortex Code.

---

## EARS Grammar Rules

| Pattern | Template | When to use |
|---|---|---|
| **Ubiquitous** | The <system> shall <system response> | Requirements active at all times |
| **Event-driven** | WHEN <trigger>, the <system> shall <system response> | Actions triggered by external event |
| **State-driven** | WHILE <state>, the <system> shall <system response> | Actions active during specific state |
| **Unwanted Behavior** | IF <condition>, THEN the <system> shall <system response> | Error handling and edge cases |
| **Optional Feature** | WHERE <feature>, the <system> shall <system response> | Behavior conditional on feature enablement |

---

## Copy-Ready Template

`⚙️ CONFIG` `✅ COPY-READY`

```markdown
# Specification: [System or Feature Name]

## 1. Context and Objective
[Short explanation of business goal and engineering context]

## 2. Scope
### In Scope
- [Deliverable 1]
- [Deliverable 2]

### Out of Scope
- [Explicit boundary item 1]
- [Explicit boundary item 2]

## 3. EARS Requirements
- REQ-01 (Ubiquitous): The system shall [response].
- REQ-02 (Event-driven): WHEN [trigger event], the system shall [response].
- REQ-03 (State-driven): WHILE [in state X], the system shall [response].
- REQ-04 (Unwanted Behavior): IF [invalid input or error], THEN the system shall [graceful error response].

## 4. Verification and Acceptance Criteria
- [ ] AC-01: [Verifiable check linked to REQ-01]
- [ ] AC-02: [Automated test proving REQ-02]

## 5. Approval Gate
- Author: [Agent/Human]
- Operator Approved: [YES/NO]
- Date: [YYYY-MM-DD]
```

---

## Related Notes

- [[Skill - Spec-Driven Development]]
- [[Workflow - Spec-Driven Development (EARS)]]
