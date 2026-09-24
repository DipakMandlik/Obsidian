---
type: cortex-code-skill
name: check-agent-access
title: "Audit Cortex Agent Access"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/check-agent-access/SKILL.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/check-agent-access/SKILL.md
category: security-governance
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skill
  - governance
aliases:
  - "Audit Cortex Agent Access"
  - "check-agent-access"
---

# Audit Cortex Agent Access

**Type:** Skill
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/check-agent-access/SKILL.md`
**Source URL:** [skills/check-agent-access/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/check-agent-access/SKILL.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

Audit every Cortex Agent for a given role, identify privilege gaps across all dependencies, and generate remediation grants.

Audits all (or selected) Cortex Agents in a Snowflake account. For each agent the skill reads the live specification, extracts every dependency (semantic views, Cortex Search services, UDFs, warehouses, tables), then checks whether a user-supplied role holds the required privilege on each object — including inherited grants via role hierarchy. Results are presented as a gap table grouped by agent, followed by an optional one-click remediation script. Triggers: check agent access, audit agent privileges, who can use my agents, role access to agents, missing agent grants, grant agent usage, agent USAGE check, PUBLIC can't call agent, fix agent permissions, check role permissions for Cortex Agent. Do NOT use for: general RBAC design (use `rbac`), warehouse credit audits, or auditing non-agent Snowflake objects.

---

## When To Use

Audits all (or selected) Cortex Agents in a Snowflake account. For each agent the skill reads the live specification, extracts every dependency (semantic views, Cortex Search services, UDFs, warehouses, tables), then checks whether a user-supplied role holds the required privilege on each object — including inherited grants via role hierarchy. Results are presented as a gap table grouped by agent, followed by an optional one-click remediation script. Triggers: check agent access, audit agent privileges, who can use my agents, role access to agents, missing agent grants, grant agent usage, agent USAGE check, PUBLIC can't call agent, fix agent permissions, check role permissions for Cortex Agent.

**Verified Triggers:** `check agent access, audit agent privileges, who can use my agents,`

---

## When Not To Use

general RBAC design (use `rbac`), warehouse credit audits, or auditing non-agent Snowflake objects.

---

## What It Enables

This skill provides Cortex Code with deterministic, domain-specific execution patterns for **Audit Cortex Agent Access**, enforcing safety, verification standards, and optimal Snowflake resource utilization without hallucinating commands or grants.

---

## Dependencies

- **Platform:** Snowflake Cortex Code CLI / Desktop
- **Category:** `security-governance`
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
Audit which Cortex Agents a role can access and fix any gaps.
```

---

## Copy-Ready Skill

`✅ COPY-READY`

```markdown
---
name: check-agent-access
title: Audit Cortex Agent Access
summary: Audit every Cortex Agent for a given role, identify privilege gaps across all dependencies, and generate remediation grants.
description: |
  Audits all (or selected) Cortex Agents in a Snowflake account. For each agent the
  skill reads the live specification, extracts every dependency (semantic views, Cortex
  Search services, UDFs, warehouses, tables), then checks whether a user-supplied role
  holds the required privilege on each object — including inherited grants via role
  hierarchy. Results are presented as a gap table grouped by agent, followed by an
  optional one-click remediation script.

  Triggers: check agent access, audit agent privileges, who can use my agents,
  role access to agents, missing agent grants, grant agent usage, agent USAGE check,
  PUBLIC can't call agent, fix agent permissions, check role permissions for Cortex Agent.

  Do NOT use for: general RBAC design (use `rbac`), warehouse credit audits,
  or auditing non-agent Snowflake objects.
prompt: Audit which Cortex Agents a role can access and fix any gaps.
language: en
status: beta
author: Martin Seifert
type: community
tools:
  - snowflake_sql_execute
  - semantic_studio
  - ask_user_question
---

## Overview

This skill performs an end-to-end privilege audit for Cortex Agents. It discovers every
agent in the account, reads each agent's live spec to build a dependency tree, then runs
parallel `SHOW GRANTS` checks to identify privilege gaps for a target role. Results are
presented per-agent with ✅ / ❌ status and a one-click remediation script is generated
for any gaps found.

## When to Use

Use this skill when you need to:

- Verify that a role (e.g. `PUBLIC`, `READER`, a custom app role) can call one or more Cortex Agents end-to-end.
- Identify missing `USAGE` grants on agents, semantic views, Cortex Search services, UDFs, or warehouses.
- Generate a ready-to-run remediation script for access gaps.
- Audit a newly created agent before rolling it out to users.

## When NOT to Use

| Topic | Delegate to |
|---|---|
| Designing or refactoring a role hierarchy | `rbac` |
| Writing masking / row access policies | `data-governance` |
| Warehouse credit or cost analysis | `cost-intelligence` |

## Workflow

1. **Discover agents** — Run `SHOW AGENTS IN ACCOUNT;`. Fall back to
   `SELECT agent_catalog, agent_schema, agent_name FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AGENTS`
   if that errors. Present the list and confirm scope with the user (all agents or a subset).

2. **Ask for the role** — Ask: *"Which role should I audit? (e.g. PUBLIC, READER, MY_CUSTOM_ROLE)"*
   Wait for the answer before proceeding.

3. **Read agent specs in parallel** — For every agent in scope call `semantic_studio` with
   `action: cortex_agent_read, source: snowflake, fqn: <DB.SCHEMA.AGENT>`. Extract all
   dependencies: semantic views, Cortex Search services, UDFs, warehouses, and any table
   FQNs found in `instructions`.

   ⚠️ If a spec result is trimmed or unavailable, fall back to fetching that agent's spec
   individually in a separate call rather than skipping it. Never assume an empty spec means
   no dependencies.

4. **Check grants in parallel** — For each dependency run the matching
   `SHOW GRANTS ON <object_type> <fqn>`:

   | Object type | Required privilege | SQL |
   |---|---|---|
   | Agent | `USAGE` | `SHOW GRANTS ON AGENT <fqn>` |
   | Semantic view | `SELECT` | `SHOW GRANTS ON SEMANTIC VIEW <fqn>` |
   | Cortex Search service | `USAGE` | `SHOW GRANTS ON CORTEX SEARCH SERVICE <fqn>` |
   | Function / UDF | `USAGE` | `SHOW GRANTS ON FUNCTION <fqn>(<arg_types>)` |
   | Warehouse | `USAGE` | `SHOW GRANTS ON WAREHOUSE <name>` |
   | Table | `SELECT` | `SHOW GRANTS ON TABLE <fqn>` |

   Also run `SHOW GRANTS TO ROLE <target_role>` and recurse up parent roles via
   `SHOW GRANTS TO ROLE <parent_role>`. Inherited grants are sufficient — only flag a gap
   when neither the role itself nor any ancestor holds the required privilege.

5. **Present the gap report** — Produce a Markdown table grouped by agent with columns
   **Object | Type | Required Privilege | Status**. Summarise: agents checked, dependencies
   checked, gaps found. Stop here if there are no gaps.

6. **Offer remediation** — Generate one `GRANT` statement per gap. Ask:
   *"Found N gap(s) — execute the GRANTs now (Yes) or show for copy-paste (No)?"*
   On **Yes**, run each statement and confirm success or report the error. On failure,
   suggest switching to `SYSADMIN` or `ACCOUNTADMIN`.

## Common Mistakes

- **Missing inherited grants** — Always resolve the full role hierarchy before flagging a
  gap; a privilege on any parent role is sufficient.
- **Inventing FQNs** — Only audit objects confirmed in the agent spec or via catalog search.
  Never guess object names.
- **Wrong role for GRANTs** — `GRANT` statements require `SYSADMIN` or `ACCOUNTADMIN`.
  Remind the user to switch roles if a GRANT fails with an insufficient-privileges error.
- **SELECT vs REFERENCES on semantic views** — Both privileges appear in grants output;
  either counts as sufficient read access.

## Examples

**Audit all agents for PUBLIC:**
> "Check which Cortex Agents the PUBLIC role can access."

Expected output: a gap table grouped by agent showing ✅ / ❌ for every dependency
(agent USAGE, semantic view SELECT, search service USAGE, function USAGE, warehouse USAGE),
followed by a remediation script for any gaps.

**Audit a single agent for a custom role:**
> "Can the ANALYST role call the FUNDRAISING_ANALYST agent?"

Expected output: single-agent gap table; if all grants are present, reports no gaps and stops.

**Execute remediation:**
> "Fix the missing grants for PUBLIC on CONSUME.ELTERNBRIEF.ELTERNBRIEF_ANALYST."

Expected output: targeted GRANT statements executed one by one, each confirmed or the error reported.

---

## Stopping Points

⚠️ **STOPPING POINT** — After presenting the agent list (Step 1), confirm scope before proceeding.

⚠️ **STOPPING POINT** — After the role is confirmed (Step 2), wait for explicit approval before reading any agent specs.

⚠️ **STOPPING POINT** — After presenting the gap report (Step 5), wait for explicit user
confirmation before executing any `GRANT` statements. RBAC changes are easy to apply but
can inadvertently expose sensitive data if the wrong role is targeted.
```

---

## Example Prompts

- `Audit which Cortex Agents a role can access and fix any gaps.`
- `check agent access`
- `audit agent privileges`
- `who can use my agents`

---

## Related

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[How to Choose a Cortex Code Skill]]

---

## Official Source

- **Repository Path:** `skills/check-agent-access/SKILL.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/check-agent-access/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/check-agent-access/SKILL.md)

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
