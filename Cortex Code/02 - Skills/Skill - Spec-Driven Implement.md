---
type: cortex-code-skill
name: implement
title: "Spec-Driven Implement"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/spec-driven/implement/SKILL.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spec-driven/implement/SKILL.md
category: sdlc
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skill
  - spec-driven
aliases:
  - "Spec-Driven Implement"
  - "implement"
---

# Spec-Driven Implement

**Type:** Skill
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/spec-driven/implement/SKILL.md`
**Source URL:** [skills/spec-driven/implement/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spec-driven/implement/SKILL.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

Provides specialized operational guidance and instructions for Snowflake Cortex Code.

---

## When To Use

Use when requiring this specific Cortex Code operational capability.



---

## When Not To Use

Do not use when outside the designated Snowflake or Cortex Code domain boundary.

---

## What It Enables

This skill provides Cortex Code with deterministic, domain-specific execution patterns for **Spec-Driven Implement**, enforcing safety, verification standards, and optimal Snowflake resource utilization without hallucinating commands or grants.

---

## Dependencies

- **Platform:** Snowflake Cortex Code CLI / Desktop
- **Category:** `sdlc`
- **Related Notes:** [[Cortex Code Skills Master Index]], [[CoCo Quick Access]], [[How to Choose a Cortex Code Skill]], [[Skill - Spec-Driven Development]], [[Workflow - Spec-Driven Development (EARS)]]

---

## Workflow

1. **Invocation:** Activate the skill explicitly via prompt or natural-language trigger.
2. **Context Inspection:** Inspect the environment, objects, and local files according to the skill instructions.
3. **Execution:** Execute deterministic actions or code generation following the skill protocol.
4. **Verification & Proof:** Validate outputs, grants, or generated code before concluding.

---

## Activation

```text
Help me with Spec-Driven Implement
```

---

## Copy-Ready Skill

`✅ COPY-READY`

```markdown
# Implement Sub-Skill

Execute an existing specification that has already been created and approved.

## When to Use

- User has an existing spec file they want implemented
- Resuming implementation after a break
- Re-implementing a spec that was previously attempted

## Workflow

### PHASE 1: LOCATE SPEC

**Actions**:
1. Ask user for spec location, or
2. Search `specs/` folder for recent specs
3. Present found specs for selection

**Search Pattern**:
```
specs/
├── features/*/requirements.md
├── bugfixes/*/bugfix-spec.md
└── refactors/*/spec.md
```

**⚠️ MANDATORY STOPPING POINT**: Confirm correct spec:
```
Found specification: `{spec path}`

**Type**: {feature/bugfix/refactor}
**Status**: {draft/approved/in-progress}
**Summary**: {brief summary}

Is this the spec you want to implement?
```

---

### PHASE 2: VALIDATE SPEC READINESS

**Check List**:
- [ ] Spec has `status: approved` or user confirms ready
- [ ] All requirements are clear and unambiguous
- [ ] No open questions remain
- [ ] Dependencies are available

**If Issues Found**:
```
This spec has issues that need resolution:

- [ ] {Issue 1}
- [ ] {Issue 2}

Should we resolve these before implementing, or proceed anyway?
```

---

### PHASE 3: CREATE IMPLEMENTATION PLAN

**For Features**:
Extract tasks from `requirements.md`:
1. Parse each REQ-XXX requirement
2. Create implementation task for each
3. Order by dependencies

**For Bugfixes**:
Extract from `bugfix-spec.md`:
1. Root cause location
2. Fix approach
3. Unchanged behavior verifications

**For Refactors**:
Extract from `spec.md`:
1. Refactoring steps in order
2. Preserved behavior checkpoints

**Output Format**:
```markdown
## Implementation Plan for: {spec name}

### Tasks (in order)

1. **{Task title}**
   - Source: REQ-001 / EB-001 / Step 1
   - Files: `{files to modify}`
   - Acceptance: {criteria}

2. **{Task title}**
   - Source: {requirement reference}
   - Files: `{files}`
   - Acceptance: {criteria}

### Verification Points
- After Task 2: Verify {preserved behavior}
- After Task 4: Run {tests}
```

**⚠️ MANDATORY STOPPING POINT**: Present plan for approval.

---

### PHASE 4: EXECUTE TASKS

**Process for Each Task**:

1. **Announce**: State which task you're starting
2. **Implement**: Make the code changes
3. **Verify**: Check acceptance criteria
4. **Report**: Mark complete and summarize

**Progress Updates**:
```
## Task Progress

### Task 1: {title}
- Status: ✅ COMPLETE
- Changes: `file.ts:42-56` - {description}
- Verification: {criteria met}

### Task 2: {title}  
- Status: 🔄 IN PROGRESS
```

**If Task Fails**:
```
⚠️ Task blocked: {task title}

**Issue**: {what went wrong}
**Options**:
1. {Alternative approach 1}
2. {Alternative approach 2}
3. Skip and continue

How should I proceed?
```

---

### PHASE 5: FINAL VALIDATION

**Run Validation Based on Spec Type**:

**Feature Validation**:
- [ ] All REQ-XXX requirements implemented
- [ ] All acceptance criteria met
- [ ] No regressions in related functionality

**Bugfix Validation**:
- [ ] Bug no longer reproducible
- [ ] Expected behaviors (EB-*) achieved
- [ ] Unchanged behaviors (UB-*) verified

**Refactor Validation**:
- [ ] All preserved behaviors (PB-*) verified
- [ ] Test suite passes
- [ ] No functional changes

**⚠️ MANDATORY STOPPING POINT**: Present final results:
```
Implementation complete for: {spec name}

**Summary**:
- Tasks completed: {X}/{Y}
- Files modified: {list}
- Files created: {list}

**Validation**:
- Requirements met: {X}/{Y}
- Tests passing: {YES/NO}
- Regressions: {NONE/list}

Ready to finalize? (commit, update spec status, etc.)
```

---

### PHASE 6: FINALIZE

**Actions**:
1. Update spec status to `complete`
2. Add completion date
3. Document any deviations from spec
4. Suggest commit message

**Spec Status Update**:
```yaml
---
status: complete
completed: {date}
implemented_by: Cortex Code
deviations: {any differences from original spec}
---
```

**Commit Message Template**:
```
{type}: {brief description}

Implements specification: {spec path}

Changes:
- {change 1}
- {change 2}

Refs: {any ticket/issue numbers}
```

---

## Resuming Incomplete Implementation

If implementation was interrupted:

1. **Find Progress**: Check `tasks.md` or spec for completion markers
2. **Assess State**: Verify which tasks are truly complete
3. **Resume**: Continue from first incomplete task

**Resume Message**:
```
Resuming implementation of: {spec name}

**Progress Found**:
- Completed: Tasks 1-3
- Next: Task 4

Should I continue from Task 4?
```

---

## Quick Reference

**Phases**:
LOCATE → VALIDATE → PLAN → EXECUTE → VALIDATE → FINALIZE

**Key Files**:
- Features: `specs/features/{name}/requirements.md`
- Bugfixes: `specs/bugfixes/{id}/bugfix-spec.md`
- Refactors: `specs/refactors/{name}/spec.md`
```

---

## Example Prompts

- `Run Spec-Driven Implement workflow for my Snowflake account`

---

## Related

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[How to Choose a Cortex Code Skill]]
- [[Skill - Spec-Driven Development]]
- [[Workflow - Spec-Driven Development (EARS)]]

---

## Official Source

- **Repository Path:** `skills/spec-driven/implement/SKILL.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/spec-driven/implement/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spec-driven/implement/SKILL.md)

---

## Version Tracking

- **Repository Commit:** `28b549f48da9994307081a9d4d2f16379f5a9c16`
- **Branch:** `main`
- **Sync Date:** 2026-09-24

---

## Official Repository Knowledge

This skill is directly extracted from the official Snowflake Labs Cortex Code skills repository (`Snowflake-Labs/coco-skills`). It reflects official Snowflake best practices and validated agent behaviors.

---

## My Operational Notes

- Store local artifacts generated by this skill in designated project subfolders.
- When running in production Snowflake accounts, ensure warehouse size and role privileges align with the operation.
- For multi-step workflows, verify intermediate outputs before proceeding to downstream phases.

---

## Client Demonstration Notes

- Highlight that Cortex Code executes verified, repository-backed skills rather than guessing.
- Demonstrate evidence capture and deterministic output validation live for clients.
