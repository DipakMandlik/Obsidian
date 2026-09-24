# Cortex Code (CoCo) Skills Library

This directory contains the official Snowflake Cortex Code skills from `Snowflake-Labs/coco-skills`.

You can copy any skill folder or individual `.md` file directly into your Cortex Code workspace or global profile to activate it.

## How to Add Skills into Cortex Code (CoCo)

### Option 1: Copy to Project Workspace (Recommended)
In your project directory, copy the skill folder into `.snowflake/cortex/skills/`:
```bash
# Create skills directory if not present
mkdir -p .snowflake/cortex/skills

# Example: Add RBAC skill
cp -r "skills/COCO/rbac" .snowflake/cortex/skills/

# Example: Add Well-Architected Framework Assessment
cp -r "skills/COCO/well-architected-framework-assessment" .snowflake/cortex/skills/

# Example: Add Spark Migration
cp -r "skills/COCO/spark-migration" .snowflake/cortex/skills/
```

### Option 2: Copy Single `.md` File
You can also copy the standalone `.md` file directly into your Cortex Code skills folder:
```bash
mkdir -p .snowflake/cortex/skills/rbac
cp "skills/COCO/rbac.md" .snowflake/cortex/skills/rbac/SKILL.md
```

### Option 3: Global Install (All Projects)
```bash
mkdir -p ~/.snowflake/cortex/skills
cp -r "skills/COCO/<skill-folder>" ~/.snowflake/cortex/skills/
```

---

## Complete Skills Catalog

| Skill | Standalone File | Full Folder | Description |
|---|---|---|---|
| **Audit Cortex Agent Access** | `check-agent-access.md` | `check-agent-access/` | Audit every Cortex Agent for a given role, identify privilege gaps across all dependencies, and generate remediation grants. |
| **DCR V1 to V2 Migration** | `dcr-v1-to-v2.md` | `dcr-v1-to-v2/` | Migrate a Snowflake Data Clean Room from V1 SAMOOHA Provider/Consumer API to V2 Collaboration API. |
| **SAP BDC Zero-Copy Connector** | `manage-zerocopy-sapbdc.md` | `manage-zerocopy-sapbdc/` | Manage the lifecycle of the SAP BDC zero-copy connector with minimal CSN generation. |
| **Plan and Run MLOps** | `mlops.md` | `mlops/` | Router for MLOps work on Snowflake: assess maturity, pick promotion patterns, and implement CI/CD, monitoring, and governance. |
| **Build Ontology Stack on Snowflake** | `ontology-stack-builder.md` | `ontology-stack-builder/` | Generates the full Ontology-on-Snowflake stack from any relational schema through a 7-phase gated workflow. |
| **OpenFlow PrivateLink Setup** | `openflow-spcs-privatelink.md` | `openflow-spcs-privatelink/` | Set up AWS PrivateLink between OpenFlow on SPCS and private data sources like RDS or on-prem databases. |
| **Learn Snowflake Quickstarts** | `quickstart-guide.md` | `quickstart-guide/` | Paste a Snowflake Quickstart URL and get a guided, interactive learning experience. |
| **Snowflake RBAC Patterns** | `rbac.md` | `rbac/` | Router skill for designing Snowflake Role-Based Access Control hierarchies and access role patterns. |
| **Review Skill for SF Labs** | `review-skill-sflabs.md` | `review-skill-sflabs/` | Pre-PR self-check that audits a local skill directory for Snowflake Labs catalog readiness. |
| **Apply Semantic View Patterns** | `semantic-view-patterns.md` | `semantic-view-patterns/` | Tutorials and apply-mode for 25 Snowflake Semantic View modeling patterns spanning joins, metrics, and dimensions. |
| **Snowflake Docs** | `snowflake-docs.md` | `snowflake-docs/` | Official documentation lookup via Cortex Knowledge Extension (CKE) / Cortex Search. |
| **Snowpipe BCDR on Azure** | `snowpipe-bcdr.md` | `snowpipe-bcdr/` | Snowpipe disaster recovery patterns for Azure ADLS Gen2 with failover, failback, and catchup procedures. |
| **SnowPro Certification Study** | `snowpro-study.md` | `snowpro-study/` | Comprehensive skill for generating study plans, domain deep-dives, flashcards, and exam questions for SnowPro certifications. |
| **Install Snowflake Industry Solutions** | `solutions-installer.md` | `solutions-installer/` | Install pre-built industry solutions into your Snowflake account from the sf-solutions repository. |
| **Spark Migration Coordinator** | `spark-migration.md` | `spark-migration/` | Master router for Spark scripts and notebooks to Snowflake conversion. Routes to SCOS or SMA. |
| **Snowpark Connect (SCOS)** | `snowpark-connect.md` | `snowpark-connect/` | Default recommended Spark migration path preserving PySpark DataFrame API surface. |
| **Snowpark API (SMA CLI)** | `snowpark-api.md` | `snowpark-api/` | Migrate PySpark to native Snowpark DataFrame API with SMA CLI, EWI fixes, and DVP validation. |
| **Databricks Notebook Migration** | `snowflake-notebook-migration.md` | `snowflake-notebook-migration/` | Convert Databricks and Jupyter notebooks to native Snowflake Notebooks. |
| **Spec-Driven Development** | `spec-driven.md` | `spec-driven/` | SDLC workflow enforcing EARS-notation specifications and approval gates before any code is written. |
| **Swap Snowflake Databases** | `swap-databases.md` | `swap-databases/` | Swap two Snowflake databases safely with object sync, grant mirroring, and zerocopy support. |
| **Well-Architected Framework Assessment** | `well-architected-framework-assessment.md` | `well-architected-framework-assessment/` | Assess a Snowflake account against all five Well-Architected Framework pillars. |
| **XO Operator Workflow System** | `xo.md` | `xo/` | OODA loop workflow system, durable memory vault, task index, and specialized subagents. |

---

## Live Killer Demo Commands
See [Commands.md](Commands.md) for 15+ verified, field-tested demonstration prompts including autonomous environment discovery, multi-agent swarms, recursive dependency checks, and the autonomous remediation boundary.