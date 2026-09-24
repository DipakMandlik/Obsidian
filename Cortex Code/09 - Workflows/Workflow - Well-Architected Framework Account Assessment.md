---
type: cortex-code-workflow
title: "Workflow - Well-Architected Framework Account Assessment"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/well-architected-framework-assessment/
category: architecture-assessment
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - waf
  - architecture
aliases:
  - "WAF Assessment Workflow"
---

# Workflow - Well-Architected Framework Account Assessment

**Type:** Architectural Audit Workflow
**Source:** Snowflake-Labs/coco-skills
**Primary Skill:** [[Skill - Well-Architected Framework Assessment]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

Executes an automated account-wide health check across all 5 Snowflake Well-Architected Framework pillars, producing an evidence-backed score and exportable executive report.

---

## Assessment Stages

1. **Scope Selection:** Select specific pillars or execute comprehensive 5-pillar assessment.
2. **Metadata Introspection:** Executes SQL inspection queries against `ACCOUNT_USAGE` and `INFORMATION_SCHEMA`.
3. **Pillar Scoring:** Grades checks as On Track (Green), Needs Improvement (Yellow), or Needs Attention (Red).
4. **Interactive Review:** Inspects findings inline in Cortex Code.
5. **Report Export:** Generates standalone client HTML report using [[Asset - WAF Assessment HTML Report Template]].

---

## Related Notes

- [[Skill - Well-Architected Framework Assessment]]
- [[Reference - Well-Architected Framework Pillars and Assessment Criteria]]
- [[Asset - WAF Assessment HTML Report Template]]
