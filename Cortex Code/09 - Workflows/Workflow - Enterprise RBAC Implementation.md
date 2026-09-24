---
type: cortex-code-workflow
title: "Workflow - Enterprise RBAC Implementation"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/rbac/
category: security-governance
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - rbac
  - governance
aliases:
  - "Enterprise RBAC Workflow"
---

# Workflow - Enterprise RBAC Implementation

**Type:** Governance Implementation Workflow
**Source:** Snowflake-Labs/coco-skills
**Primary Skill:** [[Skill - Snowflake RBAC Patterns]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

Guides the architecture, generation, and verification of enterprise Role-Based Access Control (RBAC) in Snowflake, cleanly decoupling Access Roles from Functional Roles.

---

## Implementation Stages

1. **Access Role Layering:** Generate Database Access Roles (DBAR), Schema Access Roles (SCAR), and Warehouse Access Roles (WAR).
2. **Privilege Assignment:** Grant `USAGE`, `SELECT`, `INSERT` to SCARs with managed access schema enforcement.
3. **Functional Role Mapping:** Create job-based Functional Roles (e.g. `DATA_ENGINEER`, `ANALYST`) and grant appropriate access roles.
4. **User Assignment:** Assign functional roles to end-user identities.
5. **Privilege Audit & Verification:** Execute [[Skill - Audit Cortex Agent Access]] and hierarchy traversal checks to verify zero privilege escalation.

---

## Related Notes

- [[Skill - Snowflake RBAC Patterns]]
- [[Reference - Snowflake RBAC Roles and Hierarchies]]
