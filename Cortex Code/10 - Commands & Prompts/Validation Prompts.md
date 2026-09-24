---
type: cortex-code-prompts
title: "Validation Prompts"
category: prompts
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - validation
---

# Validation Prompts

**Type:** Copy-Ready Validation Prompts
**Last Reviewed:** 2026-09-24

---

## Adversarial Self-Review & Proof

`✅ COPY-READY`

### 1. Adversarial Review of Own Findings
```text
Analyze this environment and give me your 5 most important findings.

Now assume your first answer may be wrong.

Create an adversarial review of your own findings. Attempt to disprove every finding using additional Snowflake queries and metadata. Remove anything that cannot be independently verified.
```

### 2. Two-Phase Dataset Parity Validation
```text
Execute Phase A (source Spark) and Phase B (Snowpark Connect) against synthetic data schemas in ./schemas. Compare row counts, column types, null distributions, and aggregation checksums. Flag any discrepancies with line-numbered code traces.
```
