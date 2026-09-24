---
type: cortex-code-skill
name: mlops
title: "Plan and run MLOps"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/mlops/SKILL.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/mlops/SKILL.md
category: ai-mlops
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skill
  - mlops
aliases:
  - "Plan and run MLOps"
  - "mlops"
---

# Plan and run MLOps

**Type:** Skill
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/mlops/SKILL.md`
**Source URL:** [skills/mlops/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/mlops/SKILL.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

Router for MLOps work on Snowflake — assess maturity, pick promotion patterns, and implement CI/CD, monitoring, and governance.

Use when a developer or data engineer wants to assess MLOps maturity, design a promotion strategy (Code/Model/Hybrid), or implement MLOps capabilities (CI/CD, monitoring, retraining, governance) on Snowflake for traditional ML or LLM/GenAI workloads. Triggers: mlops, mlops maturity, mlops assessment, mlops strategy, mlops pattern, mlops framework, model promotion, ml ci/cd, ml monitoring, llmops, rag pipeline ops, fine-tuning ops.

---

## When To Use

Router for MLOps work on Snowflake — assess maturity, pick promotion patterns, and implement CI/CD, monitoring, and governance.

**Verified Triggers:** `mlops, mlops maturity, mlops assessment, mlops strategy, mlops pattern, mlops framework, model promotion, ml ci/cd, ml monitoring, llmops, rag pipeline ops, fine-tuning ops`

---

## When Not To Use

Do not use when outside the designated Snowflake or Cortex Code domain boundary.

---

## What It Enables

This skill provides Cortex Code with deterministic, domain-specific execution patterns for **Plan and run MLOps**, enforcing safety, verification standards, and optimal Snowflake resource utilization without hallucinating commands or grants.

---

## Dependencies

- **Platform:** Snowflake Cortex Code CLI / Desktop
- **Category:** `ai-mlops`
- **Related Notes:** [[Cortex Code Skills Master Index]], [[CoCo Quick Access]], [[How to Choose a Cortex Code Skill]]

---

## Workflow

1. **Invocation:** Activate the skill explicitly via prompt or natural-language trigger.
2. **Context Inspection:** Inspect the environment, objects, and local files according to the skill instructions.
3. **Execution:** Execute deterministic actions or code generation following the skill protocol.
4. **Verification & Proof:** Validate outputs, grants, or generated code before concluding.

---

## Activation

```text
Help me set up MLOps on Snowflake for my ML project.
```

---

## Copy-Ready Skill

`✅ COPY-READY`

```markdown
---
name: mlops
title: Plan and run MLOps
summary: Router for MLOps work on Snowflake — assess maturity, pick promotion patterns, and implement CI/CD, monitoring, and governance.
description: "Use when a developer or data engineer wants to assess MLOps maturity, design a promotion strategy (Code/Model/Hybrid), or implement MLOps capabilities (CI/CD, monitoring, retraining, governance) on Snowflake for traditional ML or LLM/GenAI workloads. Triggers: mlops, mlops maturity, mlops assessment, mlops strategy, mlops pattern, mlops framework, model promotion, ml ci/cd, ml monitoring, llmops, rag pipeline ops, fine-tuning ops."
prompt: Help me set up MLOps on Snowflake for my ML project.
language: en
status: Published
author: Snowflake Solutions Team
type: snowflake
tools:
  - snowflake_sql_execute
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Plan and run MLOps

## Overview

Router skill for operationalizing ML and LLM/GenAI workloads on Snowflake. It covers the *process and governance layer* — when to promote, what gates to enforce, what to monitor, how to roll back. It does **not** cover SDK-level code (model registration, feature store APIs, training loops) — that belongs to the `machine-learning` skill.

This skill applies to traditional ML *and* GenAI (prompt management, RAG, fine-tuning, agentic apps). There is no separate "LLMOps" — LLM operationalization is part of MLOps with workload-specific adaptations.

**Scope split**

| Question | Owner |
|---|---|
| When should I promote a model? What gates must it pass? | mlops |
| How do I register a model or deploy an endpoint? (code) | machine-learning |
| What should I monitor after deployment? When to roll back? | mlops |
| How do I set up Feature Store / Cortex Search? (code) | machine-learning |
| How should I govern Feature Store / Registry across environments? | mlops |
| How do I train / fine-tune / build RAG? (code) | machine-learning |
| How should I operationalize training across environments? | mlops |

**Platform constraint:** All recommendations assume Snowflake as the platform (Model Registry, Feature Store, Cortex AI, Snowpark, Tasks/Streams). Do not propose third-party platforms unless the user explicitly asks.

**Explain before asking:** Always introduce concepts (maturity levels L0–L3, promotion patterns, capability dimensions) before asking the user to make decisions about them. Do not assume prior knowledge.

## Sub-flows

- `implement-patterns/INSTRUCTIONS.md` — implementation playbooks for promotion, CI/CD, monitoring, governance (includes maturity assessment as part of the pattern selection workflow)

## Workflow

### Step 1: Detect intent

Ask the user which path they need:

1. **Assessment & strategy** — evaluate current maturity, pick patterns, build a roadmap
2. **Implementation patterns** — guidance for a specific capability (CI/CD, monitoring, etc.)
3. **Full setup** — end-to-end MLOps design from scratch

### Step 2: Route

| Intent | Route |
|---|---|
| ASSESS — "assess maturity", "gap analysis", "roadmap", "where are we" | Load `implement-patterns/INSTRUCTIONS.md` — start with promotion pattern determination |
| PATTERNS — "promotion pattern", "ci/cd", "monitoring", "retraining", "feature store governance", "RAG pipeline ops", "LLM monitoring" | Load `implement-patterns/INSTRUCTIONS.md` |
| FULL SETUP — "setup mlops from scratch", "end to end" | Load `implement-patterns/INSTRUCTIONS.md` — start with promotion pattern determination, then work through capabilities per priority |

⚠️ STOPPING POINT: Before loading `implement-patterns/INSTRUCTIONS.md`, the user MUST have an explicit promotion pattern (Code / Model / Hybrid). If unknown, run the decision tree (ask about team structure, artifact type, deployment frequency). Do not generate implementation guidance without it.

### Step 3: Per-message intent re-evaluation

On every user message — not just the first — re-check intent. If the user shifts to implementation ("start with X", "let's build", "show me the code", "what SQL do I need"):

1. STOP generating from general knowledge.
2. Load `implement-patterns/INSTRUCTIONS.md` immediately, passing known context (pattern, maturity, environments).
3. If promotion pattern is unknown, determine it briefly before loading.

## Common Mistakes

- Generating implementation code from general knowledge instead of loading `implement-patterns/INSTRUCTIONS.md`.
- Skipping promotion-pattern selection and producing pattern-agnostic recommendations (they will be wrong).
- Treating LLM/GenAI as a separate "LLMOps" track instead of a workload variant.
- Recommending non-Snowflake tools (SageMaker, Vertex, Databricks, MLflow) when the user did not ask.
- Answering "how do I register a model" inside this skill — that's `machine-learning`.
- Asking the user to choose between L1 and L2 without first explaining what the levels mean.

## Red Flags

Refuse these rationalizations:

- "The user seems to know what they want, I'll skip the promotion-pattern question." — No. Pattern is a hard prerequisite.
- "I'll generate the CI/CD pipeline from memory, faster than loading the sub-flow." — No. Sub-flow content is curated and tested; general-knowledge output drifts.
- "They asked about MLflow, I'll just answer." — Only if they explicitly asked. Default is Snowflake-native.
- "The roadmap is obvious, I'll skip the assessment." — No. Maturity baseline drives sequencing.
- "They want to start implementing, I don't need to re-check intent each turn." — Re-evaluate every message.

## Stopping Points

- Step 2 — wait for explicit promotion pattern (Code / Model / Hybrid) before loading `implement-patterns/INSTRUCTIONS.md`. If unknown, run decision tree or full assessment first.

## Output

- Assessment route: maturity scorecard + prioritized roadmap.
- Patterns route: implementation playbook for the selected capability.
- Full setup: complete architecture with sequenced implementation plan.
```

---

## Example Prompts

- `Help me set up MLOps on Snowflake for my ML project.`
- `mlops`
- `mlops maturity`
- `mlops assessment`

---

## Related

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[How to Choose a Cortex Code Skill]]

---

## Official Source

- **Repository Path:** `skills/mlops/SKILL.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/mlops/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/mlops/SKILL.md)

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
