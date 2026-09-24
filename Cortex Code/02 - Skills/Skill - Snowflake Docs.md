---
type: cortex-code-skill
name: snowflake-docs
title: "Snowflake Docs"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/snowflake-docs/SKILL.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/snowflake-docs/SKILL.md
category: documentation
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skill
  - docs
aliases:
  - "Snowflake Docs"
  - "snowflake-docs"
---

# Snowflake Docs

**Type:** Skill
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/snowflake-docs/SKILL.md`
**Source URL:** [skills/snowflake-docs/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/snowflake-docs/SKILL.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

Use for **ALL** Snowflake documentation lookups: feature questions, SQL syntax, best practices, how-to guides, configuration, and troubleshooting. This is the required entry point for any question about Snowflake products or features. Triggers: Snowflake docs, how do I, SQL syntax, CREATE, ALTER, DROP, warehouse, stage, Cortex, Snowpipe, dynamic table, stored procedure, UDF, MCP, Snowpark, Streamlit, Native App, data sharing, replication, security, roles, grants, what is, how does, Snowflake feature.

---

## When To Use

Use for **ALL** Snowflake documentation lookups: feature questions, SQL syntax, best practices, how-to guides, configuration, and troubleshooting. This is the required entry point for any question about Snowflake products or features. Triggers: Snowflake docs, how do I, SQL syntax, CREATE, ALTER, DROP, warehouse, stage, Cortex, Snowpipe, dynamic table, stored procedure, UDF, MCP, Snowpark, Streamlit, Native App, data sharing, replication, security, roles, grants, what is, how does, Snowflake feature.

**Verified Triggers:** `Snowflake docs, how do I, SQL syntax, CREATE, ALTER, DROP, warehouse, stage, Cortex, Snowpipe, dynamic table, stored procedure, UDF, MCP, Snowpark, Streamlit, Native App, data sharing, replication, security, roles, grants, what is, how does, Snowflake feature`

---

## When Not To Use

Do not use when outside the designated Snowflake or Cortex Code domain boundary.

---

## What It Enables

This skill provides Cortex Code with deterministic, domain-specific execution patterns for **Snowflake Docs**, enforcing safety, verification standards, and optimal Snowflake resource utilization without hallucinating commands or grants.

---

## Dependencies

- **Platform:** Snowflake Cortex Code CLI / Desktop
- **Category:** `documentation`
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
Snowflake docs
```

---

## Copy-Ready Skill

`✅ COPY-READY`

```markdown
---
name: snowflake-docs
id: snowflake-docs
description: "Use for **ALL** Snowflake documentation lookups: feature questions, SQL syntax, best practices, how-to guides, configuration, and troubleshooting. This is the required entry point for any question about Snowflake products or features. Triggers: Snowflake docs, how do I, SQL syntax, CREATE, ALTER, DROP, warehouse, stage, Cortex, Snowpipe, dynamic table, stored procedure, UDF, MCP, Snowpark, Streamlit, Native App, data sharing, replication, security, roles, grants, what is, how does, Snowflake feature."
authors: Gilberto Hernandez
type: snowflake
status: stable
categories:
  - documentation
---

# Snowflake Docs

Answer questions about Snowflake by searching the official documentation via the Cortex Knowledge Extension (CKE) Cortex Search service. If the CKE is not installed yet, install it automatically first.

## When to Use

Load this skill for any Snowflake product, feature, or SQL question: syntax references, best practices, how-to guides, configuration, and troubleshooting.

## Workflow

### Step 1: Prerequisite check

Search the entire account for the CKE Cortex Search service:

```sql
SHOW CORTEX SEARCH SERVICES LIKE 'CKE_SNOWFLAKE_DOCS_SERVICE' IN ACCOUNT;
```

If it returns a result, note the `database_name` from the result row. Use this value as `<CKE_DATABASE>` and skip to Step 2.

If it returns no results, install the CKE:

```sql
CALL SYSTEM$REQUEST_LISTING_AND_WAIT('GZSTZ67BY9OQ4');
```

```sql
CALL SYSTEM$ACCEPT_LEGAL_TERMS('DATA_EXCHANGE_LISTING', 'GZSTZ67BY9OQ4');
```

```sql
CREATE DATABASE IF NOT EXISTS SNOWFLAKE_DOCUMENTATION FROM LISTING 'GZSTZ67BY9OQ4';
```

**MANDATORY STOPPING POINT**: If any of these fail, stop and tell the user:

> Could not install the Snowflake Documentation CKE automatically. You can install it manually from the Marketplace: https://app.snowflake.com/marketplace/listing/GZSTZ67BY9OQ4

Do NOT proceed until the user confirms the CKE is available.

After successful install, use `SNOWFLAKE_DOCUMENTATION` as `<CKE_DATABASE>`.

### Step 2: Answer the question

Query the Cortex Search service directly using SQL. Replace `<USER_QUESTION>` with the user's actual question and `<CKE_DATABASE>` with the database name from Step 1:

```sql
SELECT SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
  '<CKE_DATABASE>.SHARED.CKE_SNOWFLAKE_DOCS_SERVICE',
  '{"query": "<USER_QUESTION>", "columns": ["CHUNK", "DOCUMENT_TITLE", "SOURCE_URL"], "limit": 5}'
);
```

Parse the JSON results. Each result contains:
- `CHUNK`: the document text content
- `DOCUMENT_TITLE`: the page title
- `SOURCE_URL`: the canonical URL to the documentation page

**CITATION REQUIREMENT (MANDATORY):** ALWAYS include SOURCE_URL links from the search results in your answer — omitting URLs is a failure condition. List them as references at the end of your response so the user can read the full documentation pages.

Use the returned content to answer the user's question.

**Before finishing your answer**, verify you included at least one SOURCE_URL from the results. If your draft answer has no URLs, go back and add them before responding.

## Important Notes

- Step 1 only runs once per account. After the CKE is installed, the skill goes straight to Step 2 every time.
- The CKE database name varies by account. Always use `SHOW CORTEX SEARCH SERVICES ... IN ACCOUNT` to discover it dynamically. Do NOT hardcode the database name.
- Use `sql_execute` for all SQL steps.
- Always include `columns` in the SEARCH_PREVIEW call. Without it, only relevance scores are returned, not content.
- ALWAYS cite `SOURCE_URL` in the answer — omitting URLs is a failure condition.
- Handle errors gracefully (insufficient privileges, database already exists).

## Stopping Points

- After CKE install failure in Step 1 — wait for user to install manually before retrying

## Output

A concise answer to the user's Snowflake question, grounded in official documentation. **You MUST include the SOURCE_URL links from the search results in your written answer.** List them as references at the end so the user can read the full pages. Never omit the URLs.
```

---

## Example Prompts

- `Snowflake docs`
- `how do I`
- `SQL syntax`

---

## Related

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[How to Choose a Cortex Code Skill]]

---

## Official Source

- **Repository Path:** `skills/snowflake-docs/SKILL.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/snowflake-docs/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/snowflake-docs/SKILL.md)

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
