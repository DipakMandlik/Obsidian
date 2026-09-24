---
type: cortex-code-reference
title: "Reference - XO Refs Verified Review"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/skills/xo/references/refs-verified-review.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/skills/xo/references/refs-verified-review.md
category: xo-reference
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - reference
  - xo
aliases:
  - "Reference - XO Refs Verified Review"
  - "refs-verified-review"
---

# Reference - XO Refs Verified Review

**Type:** Reference
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo/skills/xo/references/refs-verified-review.md`
**Source URL:** [refs-verified-review.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/skills/xo/references/refs-verified-review.md)
**Category:** XO OODA Reference Guidance
**Last Reviewed:** 2026-09-24

---

## Purpose

Provides official operational guidance and procedural requirements for the **Refs Verified Review** stage of the XO operator harness in Cortex Code.

---

## Canonical Operational Guidance

`📘 REFERENCE` `✅ COPY-READY`

---
name: refs-verified-review
description: "Build-and-test protocol for verified PR review. Risk factor assessment, empirical evidence gathering, before/after comparison."
---

# Verified Review Process

Full build-and-test protocol for PR review when static analysis is insufficient.

## When to Use Verified Review

| Risk Factor | Present? | Implication |
|-------------|----------|-------------|
| API or spec changes | Yes → verified | Downstream consumers may break |
| Breaking change potential | Yes → verified | Need empirical confirmation |
| Downstream consumers exist | Yes → verified | Must check generated artifacts |
| Annotation/metadata changes | Yes → verified | Static analysis can be misleading |
| Doc/typo/cosmetic only | Yes → static | No runtime impact |
| Time-constrained | Yes → static | Note the caveat in assessment |

**Key lesson**: Static analysis of annotation changes may predict HIGH risk, but empirical build can reveal actual impact is LOW (e.g., spec unchanged). However, builds also catch *other* breaking changes missed by static analysis. Always prefer verified when risk factors are present.

## Procedure

### Step 1: Static Baseline

Checkout and inspect the PR normally. Read the diff, understand the intent. This gives you a hypothesis to test empirically.

### Step 2: Build the Project

Follow the project's build instructions:
- Maven: `mvn clean install -DskipTests` (build first, test separately)
- npm/yarn: `npm ci && npm run build`
- Other: check CONTRIBUTING.md, Makefile, or CI config

Record: does it build cleanly? Any new warnings?

### Step 3: Generate Downstream Artifacts

If the project produces artifacts consumed by others:
- API specs (Swagger/OpenAPI): generate and compare before/after
- Client SDKs: regenerate and diff
- Docker images: build and compare layers
- Package manifests: check dependency changes

### Step 4: Before/After Comparison

Diff the generated artifacts against the main branch versions:
- Spec files: structural diff (not just text diff)
- Client code: check for signature changes, new/removed endpoints
- Config files: check for semantic changes vs cosmetic

### Step 5: Run Test Suite

Run the project's test suite:
- Full suite if practical
- Targeted suite if full suite is too slow (document what was skipped)
- Record: pass/fail counts, new failures, flaky tests

### Step 6: Revise Risk Assessment

Update the initial static risk assessment with empirical findings:

| Aspect | Static Assessment | Empirical Finding | Revised |
|--------|------------------|-------------------|---------|
| [area] | [initial call] | [what build/test showed] | [updated] |

Feed the empirical findings table to the review Briefer as additional context when producing the review brief.

## Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|-------------|---------------|
| Skipping build because "it's just annotations" | Annotations can affect codegen, specs, clients |
| Trusting CI alone | CI may not run all downstream checks |
| Verified review without before/after diff | Build success doesn't mean no breaking changes |
| Static-only when downstream consumers exist | Risk of silent breakage |

---

## Related Notes

- [[Plugin - XO]]
- [[Skill - XO Operator Workflow]]
- [[Workflow - XO Operator OODA Loop]]
- [[Cortex Code Skills Master Index]]

---

## Official Repository Knowledge

Directly extracted from `plugins/xo/skills/xo/references/refs-verified-review.md`. This forms part of the authoritative stage reference library for Cortex Code operator workflows.
