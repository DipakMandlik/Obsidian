---
type: cortex-code-prompts
title: "Skill Activation Commands"
category: commands
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skills
aliases:
  - "Skill Activation Prompts"
---

# Skill Activation Commands

**Type:** Copy-Ready Skill Triggers
**Last Reviewed:** 2026-09-24

---

## Verified Natural Language Activation Commands

`✅ COPY-READY`

### Security, Governance & Compliance
```text
# RBAC
Help me design an RBAC hierarchy for my Snowflake account.
Create functional roles and schema access roles for analytics.

# Agent Access Audit
Audit every Cortex Agent for role ANALYST_ROLE and generate remediation grants.
Who can use my Cortex Agents in this account?

# Well-Architected Framework
Run a Well-Architected Framework assessment for this account.
Assess the Security and Governance pillar for our production account.

# Database Swap
Swap databases PROD_V1 and PROD_V2 with zero-copy grant mirroring.
```

### Data Migration & Engineering
```text
# Spark Migration (Default SCOS Path)
Assess my PySpark workload in ./pipelines for Snowflake readiness.
Migrate PySpark to SCOS and validate results against source Spark.

# Spark Migration (Snowpark API Path)
Run SMA conversion on ./spark_etl to generate Snowpark DataFrame code.
Resolve all EWIs and convert embedded file paths to stages.

# Snowpipe BCDR
Design a Snowpipe disaster recovery pattern on Azure ADLS Gen2 with failover.

# SAP BDC Integration
Create zero-copy connector to share Snowflake schema to SAP BDC using minimal CSN.
```

### AI, Semantic Layer & Development
```text
# Spec-Driven SDLC
Create a feature specification for real-time order processing using EARS notation.
Implement the approved spec in ./specs/order-processing.md.

# Semantic Views
Walk me through time intelligence and window metrics in Snowflake Semantic Views.

# Ontology Stack
Build an ontology stack on Snowflake from relational tables in schema CRM.

# MLOps
Design an MLOps model promotion and CI/CD strategy on Snowflake.
```
