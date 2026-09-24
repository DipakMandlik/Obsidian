---
type: cortex-code-reference
title: "Reference - Snowflake RBAC Roles and Hierarchies"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/rbac/
source_url: https://github.com/Snowflake-Labs/coco-skills/tree/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/rbac
category: security-governance
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - reference
  - rbac
  - governance
aliases:
  - "Snowflake RBAC Reference"
  - "RBAC Role Hierarchy"
---

# Reference - Snowflake RBAC Roles and Hierarchies

**Type:** Reference
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/rbac/`
**Category:** Security & Governance
**Last Reviewed:** 2026-09-24

---

## Purpose

Comprehensive reference for designing and implementing Snowflake Role-Based Access Control (RBAC). Covers functional vs access roles, database access roles (DBAR), schema access roles (SCAR), warehouse access roles (WAR), domain admin roles, policy roles, and secondary role usage.

---

## Canonical RBAC Reference Material

`📘 REFERENCE` `✅ COPY-READY`

### ARCHITECTURE PATTERNS

---
name: architecture-patterns
description: "Determine your Snowflake RBAC architecture based on account structure, environment strategy, and admin model. Use when: starting RBAC design, deciding how many role layers needed, choosing between centralized and federated admin."
---

# RBAC Architecture Patterns

Before implementing roles, determine which layers of the RBAC hierarchy your organization needs. Not every layer applies to every organization.

## The Full Hierarchy

```
1. OOB Account Roles
   └── 2. Environment Admin Roles
       └── 3. Business Domain Admins
           └── 4. Data Product Admins
               └── 5. Database Roles
                   └── 6. Schema Roles
```

Most organizations will **skip** one or more layers based on their architecture decisions.

---

## Decision 1: Account Strategy

The most fundamental decision. Account boundaries provide the strongest isolation - everything else is logical separation via grants.

### Option A: Single Account

All environments (Dev, Test, Prod) and all business domains in one account.

| Pros | Cons |
|------|------|
| **Simplicity** - one account to manage | **Logical separation only** - environments separated by grants, not hard boundaries |
| **No replication setup** - no cross-account data sharing or replication needed | **Compliance risk** - some orgs won't accept prod/non-prod in same account |
| **Single security perimeter** - one set of network policies, one Private Link config | **Blast radius** - misconfiguration affects everything |
| **Cloning** - clone entire databases for on-demand dev/UAT environments | **Shared limits** - account-level limits shared across all workloads |

**Best for**: Smaller organizations, centralized IT, rapid development cycles where cloning is valuable.

**Requires**: Environment Admin Roles (Level 2) to logically separate Dev/Test/Prod.

### Option B: Account Per Environment

Separate accounts for each environment tier (e.g., one for Dev/Test, one for Prod).

| Pros | Cons |
|------|------|
| **Hard separation** - prod data physically isolated from non-prod | **Replication required** - need cross-account sharing for data promotion |
| **Compliance friendly** - satisfies auditors wanting environment isolation | **Multiple security configs** - network policies, Private Link per account |
| **Independent scaling** - prod limits unaffected by dev activity | **No cross-account cloning** - can't clone prod to dev for testing |
| **Cleaner prod** - no dev/test clutter in production account | **More accounts to manage** - multiplied operational overhead |

**Best for**: Regulated industries, organizations with strict prod/non-prod separation requirements.

**Skip**: Environment Admin Roles (Level 2) - environment isolation handled by account boundaries.

### Option C: Account Per Business Unit

Large or mature business units get their own account(s), potentially with their own environment separation.

| Pros | Cons |
|------|------|
| **Full autonomy** - BUs configure their account as they see fit | **Significant overhead** - many accounts to manage |
| **Cost isolation** - clear billing separation by BU | **Data sharing complexity** - cross-BU data requires explicit sharing |
| **Independent governance** - each BU owns their security posture | **Inconsistent patterns** - BUs may diverge in RBAC approach |
| **Blast radius contained** - BU issues don't affect others | **Central visibility harder** - monitoring across accounts more complex |

**Best for**: Large enterprises, holding companies, organizations with autonomous business units.

**Skip**: Business Domain Admin Roles (Level 3) - domain isolation handled by account boundaries.

### Option D: Account Per Business Unit AND Environment

The most granular option - separate accounts for each combination (e.g., PROD_FINANCE, DEV_FINANCE, PROD_SALES, DEV_SALES).

| Pros | Cons |
|------|------|
| **Maximum isolation** - hard boundaries for both env and BU | **Account sprawl** - many accounts to manage |
| **Full autonomy + compliance** - BUs independent AND prod protected | **Complex data flows** - cross-account sharing for everything |
| **Clearest cost attribution** - billing by BU and environment | **Operational burden** - multiplied by both dimensions |

**Best for**: Large regulated enterprises with autonomous business units.

**Skip**: Both Environment Admin Roles (Level 2) AND Business Domain Admin Roles (Level 3).

### Hybrid Approaches

These options represent maximum configurations. In practice, hybrid approaches are common:

- **Mostly single account, large domains separate** - 80% of domains share accounts, but 1-2 large/sensitive domains (e.g., HR, Finance) get their own
- **Shared non-prod, separate prod** - Dev and Test share an account, Prod is separate
- **Core vs satellite** - Central data platform in shared accounts, autonomous BUs in their own

The key is consistency within each boundary - don't mix patterns arbitrarily.

### Decision Matrix

| Factor | Single Account | Per Environment | Per BU | Per BU + Env |
|--------|---------------|-----------------|--------|--------------|
| Compliance needs hard separation | ❌ | ✓ | ✓ | ✓ |
| Want cloning for dev/test | ✓ | ❌ | ❌ | ❌ |
| Centralized IT team | ✓ | ✓ | ❌ | ❌ |
| Autonomous business units | ❌ | ❌ | ✓ | ✓ |
| Minimize operational overhead | ✓ | Moderate | Low | ❌ |
| Simple data sharing | ✓ | Moderate | Low | ❌ |

---

## Decision 2: Admin Model

**Question**: Who administers databases and access?

| Choice | Implication |
|--------|-------------|
| **Centralized** | One team manages all databases. Skip Levels 3-4. Go directly from OOB roles (or Env Admins) to Database/Schema roles. |
| **Federated** | Domains have their own admins. Need Level 3 (Business Domain Admins), possibly Level 4 (Data Product Admins). |

Federated models range from "hub-and-spoke" (central platform team + domain admins) to "fully autonomous" (domains operate independently). From an RBAC perspective, these are the same - both require Domain Admin roles. The difference is organizational, not technical.

---

## Decision 3: Domain vs Data Product Admin

This decision only applies if you chose **Federated** admin model.

**Question**: How are teams organized within your domains?

| Scenario | Implication |
|----------|-------------|
| **Team owns many data products** | The team IS the domain. Use Domain Admin roles only. Skip Data Product Admin roles (Level 4). |
| **Team owns one data product** | Need Data Product Admin roles (Level 4). Domain Admin may be unnecessary or just an aggregation point. |
| **Multiple teams under one domain** | Need both. Domain Admin for shared access/governance across the domain, Data Product Admins for each team's assets. Example: HR domain with common access requirements but separate teams for Payroll, Recruitment, Benefits. |

---

## Decision 4: Database Scope

**This decision only applies if you chose Single Account (Option A) or Account Per Business Unit (Option C).**

If you have environments in separate accounts (Options B or D), cloning across accounts is not possible. Without cloning as a driver, there's little reason to constrain yourself to a single database - you will almost certainly have multiple databases per environment.

**Single database setups only make sense when environments share an account** where you want to maximize cloning benefits (clone entire environment in one operation).

**Question**: How many databases per environment?

| Choice | Implication |
|--------|-------------|
| **Single database** | Skip Database Access Roles. Schema roles grant directly to functional roles. Maximizes cloning simplicity. |
| **Multiple databases** | Need Database Access Roles (DB_R, DB_RW, DB_C) to aggregate schema access. |

---

## Common Patterns

### Pattern A: Single Account, Centralized IT
```
OOB Roles (SYSADMIN, SECURITYADMIN)
    └── Environment Admins (DEV_ADMIN, TST_ADMIN, PRD_ADMIN)
        └── Database Access Roles
            └── Schema Access Roles (<schema>_R, <schema>_RW, <schema>_C)
```
- Single account with Dev/Test/Prod environments
- Centralized team, no domain delegation
- Cloning available for on-demand environments
- Layers needed: 1, 2, 5, 6

### Pattern B: Single Account, Federated Domains
```
OOB Roles
    └── Environment Admins (DEV_ADMIN, TST_ADMIN, PRD_ADMIN)
        └── Domain Admins (SALES_ADMIN, FINANCE_ADMIN)
            └── Database Access Roles
                └── Schema Access Roles
```
- Single account with Dev/Test/Prod environments
- Federated domain administration
- Cloning available for on-demand environments
- Layers needed: 1, 2, 3, 5, 6

### Pattern C: Multi-Account by Environment, Federated Domains
```
OOB Roles (per account)
    └── Domain Admins (SALES_ADMIN, FINANCE_ADMIN)
        └── Database Access Roles
            └── Schema Access Roles
```
- Separate accounts for Dev, Test, Prod (skip Level 2)
- Multiple business domains per account
- Layers needed: 1, 3, 5, 6

### Pattern D: Large Domain with Data Product Teams
```
OOB Roles
    └── Domain Admin (HR_ADMIN)
        └── Data Product Admins (PAYROLL_ADMIN, RECRUITMENT_ADMIN)
            └── Database Access Roles
                └── Schema Access Roles
```
- Single domain with multiple teams producing different data assets
- Shared access requirements at domain level
- Layers needed: 1, 3, 4, 5, 6

### Pattern E: Maximum Complexity
```
OOB Roles
    └── Environment Admins
        └── Domain Admins
            └── Data Product Admins
                └── Database Access Roles
                    └── Schema Access Roles
```
- Single account, multi-environment
- Federated domains with discrete data product teams
- Layers needed: All

---

## Workflow

1. Answer the four decisions above
2. Identify which pattern (A-D) most closely matches
3. Note which layers you need
4. Implement top-down: start with highest layer, work down

## Next Steps by Layer

| Layer | Skill |
|-------|-------|
| 1. OOB Account Roles | `oob-account-roles` |
| 2. Environment Admin Roles | `environment-admin-roles` |
| 3. Business Domain Admins | `domain-admin-roles` |
| 4. Data Product Admins | `data-product-admin-roles` |
| 5. Database Access Roles | `database-access-roles` |
| 6. Schema Access Roles | `schema-access-roles` |

---

## Key Principles

- **Skip layers that don't apply** - unnecessary hierarchy adds complexity without value
- **Account boundaries are strongest isolation** - use multi-account when hard separation required
- **Centralized control vs federated autonomy** - pick one, don't mix inconsistently
- **Start simple, add layers as needed** - easier to add hierarchy than remove it


### DATABASE ACCESS ROLES

---
name: database-access-roles
description: "Create and manage Database Access Roles - Database Roles aggregating schema-level access across an entire database. Use when: setting up database-wide permissions, aggregating schema access roles."
---

# Database Access Roles

Database Roles that aggregate schema-level access across an entire database, providing a single point of grant for consumers.

## Relationship to Schema Access Roles

Database Access Roles sit **above** Schema Access Roles in the hierarchy:

```
Account Functional Roles (AFR)
    ↓ granted
Database Access Roles (DB_R, DB_RW, DB_C)  ← This skill
    ↓ granted
Schema Access Roles (<schema>_R, <schema>_RW, <schema>_C)  ← schema-access-roles skill
    ↓ privileges on
Schema Objects (tables, views, etc.)
```

Database Access Roles **inherit** permissions from Schema Access Roles - they receive no direct grants themselves.

## Why Database Access Roles?

| Problem | Solution |
|---------|----------|
| Granting access to many schemas individually is tedious | Aggregate all schema roles into one database role |
| Account roles shouldn't hold privileges directly | Database roles wrap privileges; account roles hold database roles |
| Consumers need a single "reader" entrypoint | DB_R provides read access to entire database |
| Clone-friendly access | Database roles clone with the database |

## Access Tiers

| Tier | Role Name | Aggregates | Use Case |
|------|-----------|------------|----------|
| READ | DB_R | All <schema>_R roles in database | Data consumers, analysts |
| READ-WRITE | DB_RW | All <schema>_RW roles in database | ETL processes, applications |
| CREATE | DB_C | All <schema>_C roles in database | Developers, CI/CD |

## Role Hierarchy

```
DB_C  (Create - highest)
  ↓ inherits
DB_RW  (Read-Write)
  ↓ inherits
DB_R  (Read - lowest)
  ↓ aggregates
<schema1>_R, <schema2>_R, ...
```

## Integration with Account Functional Roles

Database Access Roles bridge the gap between schema-level Database Roles and account-level Functional Roles:

| Account Functional Role | Database Role Granted | Purpose |
|------------------------|----------------------|---------|
| `<PREFIX>_READER` | DB_R | Database consumers |
| `<PREFIX>_ETL` | DB_RW | Data engineers |
| `<PREFIX>_SYSADMIN` | DB_C | Deployment/CI-CD |
| `<PREFIX>_ADMIN` | (owns database) | Delegated administration |
| `<PREFIX>_RBAC` | (owns DB roles) | Access governance |

**Naming Convention:**
- `<PREFIX>` = Derived from database name components (e.g., `ENT_SALES` for database `ENT_SALES_RAW`)

## Workflow

Database Access Roles are created **with the database**, before any schemas exist. Schema Access Roles are then granted up to them as schemas are added.

### Step 1: Gather Requirements
Ask user for:
- Database name
- Associated Account Functional Role names (or derive from naming convention)

### Step 2: Generate SQL
Use the template below.

### Step 3: Execute
Run as the Delegated Admin role that owns the database.

### Step 4: As Schemas Are Added
When schemas are created (see `schema-access-roles` skill), their Schema Access Roles get granted up to these Database Access Roles.

---

## SQL Template

### Variables
```sql
SET dbNm = '<DATABASE_NAME>';
SET prefixNm = '<PREFIX>';  -- For deriving Account Functional Role names

SET dbrR = 'DB_R';
SET dbrRW = 'DB_RW';
SET dbrC = 'DB_C';



SET afrAdmin  = $prefixNm || '_SYSADMIN';
SET afrCreate = $prefixNm || '_SYSADMIN';
SET afrETL    = $prefixNm || '_ETL';
SET afrRbac   = $prefixNm || '_RBAC';
SET afrReader = $prefixNm || '_READER';
```

### Review Configuration
```sql
SELECT 
   $dbNm       AS "Database Name"
  ,$afrAdmin   AS "Delegated Admin Role"
  ,$afrCreate  AS "Deploy/Create Role"
  ,$afrETL     AS "ETL Role"
  ,$afrRbac    AS "RBAC Owner Role"
  ,$afrReader  AS "Data Product Reader Role"
;
```

### Create Database Access Roles
```sql
USE ROLE IDENTIFIER($afrAdmin);
USE DATABASE IDENTIFIER($dbNm);

CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($dbrR);
CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($dbrRW);
CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($dbrC);
```

### Transfer Ownership to RBAC Role
```sql
GRANT OWNERSHIP ON DATABASE ROLE IDENTIFIER($dbrR) TO ROLE IDENTIFIER($afrRbac) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON DATABASE ROLE IDENTIFIER($dbrRW) TO ROLE IDENTIFIER($afrRbac) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON DATABASE ROLE IDENTIFIER($dbrC) TO ROLE IDENTIFIER($afrRbac) COPY CURRENT GRANTS;
```

### Grant Database Roles to Account Functional Roles
```sql
GRANT DATABASE ROLE IDENTIFIER($dbrR) TO ROLE IDENTIFIER($afrReader);
GRANT DATABASE ROLE IDENTIFIER($dbrRW) TO ROLE IDENTIFIER($afrETL);
GRANT DATABASE ROLE IDENTIFIER($dbrC) TO ROLE IDENTIFIER($afrCreate);
```

### Aggregate Schema Access Roles (per schema)
For each schema in the database, grant its Schema Access Roles to the corresponding Database Access Role:

```sql
SET scNm = '<SCHEMA_NAME>';
SET sarR = $scNm || '_R';
SET sarRW = $scNm || '_RW';
SET sarC = $scNm || '_C';

GRANT DATABASE ROLE IDENTIFIER($sarR) TO DATABASE ROLE IDENTIFIER($dbrR);
GRANT DATABASE ROLE IDENTIFIER($sarRW) TO DATABASE ROLE IDENTIFIER($dbrRW);
GRANT DATABASE ROLE IDENTIFIER($sarC) TO DATABASE ROLE IDENTIFIER($dbrC);
```

Repeat the above block for each schema.

---

## Ownership Model

| Object | Owner | Rationale |
|--------|-------|-----------|
| Database | `<PREFIX>_SYSADMIN` | Delegated admin controls database settings |
| Database Access Roles | `<PREFIX>_RBAC` | Separates access governance from administration |
| Schema Access Roles | `<PREFIX>_RBAC` | Consistent ownership for all access roles |

**Why separate ADMIN and RBAC roles?**
- ADMIN handles structural changes (DDL, database settings)
- RBAC handles access governance (who can read/write what)
- Separation of duties - the person deploying code shouldn't control who accesses data

## Key Points

- **No Direct Grants**: Database Access Roles receive no privilege grants directly - they only aggregate Schema Access Roles.
- **Single Entrypoint**: Consumers get one database role (DB_R) that provides read access to the entire database.
- **Clone-Friendly**: When you clone a database, all Database Roles and their grants clone with it.
- **RBAC Ownership**: The RBAC role owns the access roles, enabling governance teams to manage access without admin privileges.
- **Account Functional Roles**: These are created separately (by USERADMIN/SECURITYADMIN) and receive the database roles as grants.


### DOMAIN ADMIN ROLES

---
name: domain-admin-roles
description: "Create Domain Admin roles for federated team administration. Use when: teams own multiple data products, federated admin model, need to delegate object ownership within databases managed by a higher level."
---

# Domain Admin Roles

Account-level roles that provide delegated administration for a business domain (team) that owns multiple data products within databases and schemas managed by a higher level in the hierarchy.

## When You Need This

Domain Admin Roles are required when:
- A team owns **multiple data products**
- You have a **federated admin model** (not centralized)
- You want teams to manage their own objects without owning the infrastructure

**Skip this layer if**:
- A team owns only **one data product** - use Data Product Admin roles instead (Level 4)
- You have **centralized administration** - go directly to Database/Schema roles

## Key Distinction from Environment Admins

| Aspect | Environment Admin | Domain Admin |
|--------|-------------------|--------------|
| **Owns** | Databases, schemas, warehouses | Schema-bound objects only (tables, views, tasks, etc.) |
| **Infrastructure** | Creates and controls | Uses what's provisioned for them |
| **Scope** | All objects in environment | Objects within their domain's schemas |

Domain admins work **within** infrastructure owned by a higher level (Account or Environment roles).

## Role Structure

For each domain, create:

| Role | Purpose |
|------|---------|
| `<DOMAIN>_SYSADMIN` | Owns schema-bound objects (tables, views, procedures, tasks, etc.). Does NOT own databases, schemas, or warehouses. |
| `<DOMAIN>_RBAC` | Manages grants and database role hierarchy within the domain |
| `<DOMAIN>_READER` | Read-only access path. Granted DB_R roles. Warehouse access for domain's own users. SYSADMIN inherits this. |

If using environment separation (Options A or C), include environment in the name:

| Role | Example |
|------|---------|
| `<ENV>_<DOMAIN>_SYSADMIN` | `DEV_SALES_SYSADMIN`, `PRD_SALES_SYSADMIN` |
| `<ENV>_<DOMAIN>_RBAC` | `DEV_SALES_RBAC`, `PRD_SALES_RBAC` |
| `<ENV>_<DOMAIN>_READER` | `DEV_SALES_READER`, `PRD_SALES_READER` |

## Role Hierarchy

```
ENV_SYSADMIN (or SYSADMIN if no env layer)
    ↓ owns databases, schemas, warehouses
ENV_DOMAIN_SYSADMIN
    ↓ granted
ENV_DOMAIN_READER
    ↓ granted DB_R roles and warehouse access
    ↓ allows prod read-only usage without write permissions

ENV_RBAC (or SECURITYADMIN if no env layer)
    ↓ grants role management to domain
ENV_DOMAIN_RBAC
    ↓ owns database roles (<schema>_R, <schema>_RW, <schema>_C, DB_R, DB_RW, DB_C)
    ↓ manages grants within domain
```

**Why READER?** In production, domain teams often need read-only access. SYSADMIN inherits READER, so users with write needs use SYSADMIN, while read-only users use READER directly.

---

## SQL Templates

### Pattern A: Domain Under Environment (Single Account, Multi-Env)

Domain roles grant up to environment roles.

```sql
-- Variables
SET env = 'DEV';
SET domain = 'SALES';

SET domainSysadmin = $env || '_' || $domain || '_SYSADMIN';
SET domainRbac = $env || '_' || $domain || '_RBAC';
SET domainReader = $env || '_' || $domain || '_READER';

-- Parent roles (environment level)
SET parentSysadmin = $env || '_SYSADMIN';
SET parentRbac = $env || '_RBAC';
SET parentReader = $env || '_READER';

-- Create domain roles
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($domainSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($domainRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($domainReader);

-- Build role hierarchy
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read access)
GRANT ROLE IDENTIFIER($domainReader) TO ROLE IDENTIFIER($domainSysadmin);

-- Domain roles grant up to environment level
GRANT ROLE IDENTIFIER($domainSysadmin) TO ROLE IDENTIFIER($parentSysadmin);
GRANT ROLE IDENTIFIER($domainRbac) TO ROLE IDENTIFIER($parentRbac);
GRANT ROLE IDENTIFIER($domainReader) TO ROLE IDENTIFIER($parentReader);

-- Grant account-level access roles
GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($domainSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($domainSysadmin);
GRANT ROLE _AR_APPLY_TAG TO ROLE IDENTIFIER($domainSysadmin);
```

### Pattern B: Domain Under Account (Multi-Account by Env, No Env Layer)

Domain roles grant directly to OOB account roles.

```sql
-- Variables (no env prefix needed - environments are separate accounts)
SET domain = 'SALES';

SET domainSysadmin = $domain || '_SYSADMIN';
SET domainRbac = $domain || '_RBAC';
SET domainReader = $domain || '_READER';

-- Parent roles (OOB account roles)
SET parentSysadmin = 'SYSADMIN';
SET parentRbac = 'SECURITYADMIN';

-- Create domain roles
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($domainSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($domainRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($domainReader);

-- Build role hierarchy
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read access)
GRANT ROLE IDENTIFIER($domainReader) TO ROLE IDENTIFIER($domainSysadmin);

-- Grant to account level

GRANT ROLE IDENTIFIER($domainSysadmin) TO ROLE IDENTIFIER($parentSysadmin);
GRANT ROLE IDENTIFIER($domainRbac) TO ROLE IDENTIFIER($parentRbac);
GRANT ROLE IDENTIFIER($domainReader) TO ROLE SYSADMIN;  -- No parent reader at OOB level

-- Grant account-level access roles
GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($domainSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($domainSysadmin);
GRANT ROLE _AR_APPLY_TAG TO ROLE IDENTIFIER($domainSysadmin);
```

---

## Connecting to Database Roles

**Schema ownership always sits at Environment or Account level** - never at Domain or Data Product level.

### Data Product Level (Object Ownership)
Data Product SYSADMIN:
- Receives CREATE privileges on schemas via database role hierarchy (DB_C ← <schema>_C)
- **Owns the objects they create** (tables, views, procedures, etc.)
- Does NOT own the schema itself

Data Product READER:
- Receives READ access via DB_R
- Warehouse access (direct or via access roles) granted here
- SYSADMIN inherits through being granted READER

```
DataProduct_SYSADMIN
    ↓ granted
DataProduct_READER
    ↓ granted
DB_R (read access) + Warehouse access
```

```
DataProduct_SYSADMIN
    ↓ granted
DB_C (Database Role)
    ↓ granted
<schema>_C (Database Role)
    ↓ has CREATE privileges on
Schema (owned by ENV_SYSADMIN or SYSADMIN)
    ↓ creates
Objects (owned by DataProduct_SYSADMIN)
```

### Domain Level (Inherited Access)
Domain SYSADMIN:
- Granted Data Product SYSADMIN roles
- **Inherits object ownership** and **read/warehouse access** through the hierarchy
- Has **no direct schema-level privileges**

```
Domain_SYSADMIN
    ↓ granted
Domain_READER
    ↓ granted
DataProduct_READER (inherits DB_R)
```

See `database-access-roles` and `schema-access-roles` for the SQL templates that connect these layers.

---

## Warehouse Access

Warehouses are owned by a higher level. Warehouse access is granted to the **domain's own READER** role for consumption within that domain. This is critical:

**Consumers bring their own compute.** When users from Domain A query data from Domain B:
- Domain B's READER grants **data access only** (DB_R)
- Domain A's READER provides **warehouse access**
- Secondary roles aggregate both at runtime

Bundling warehouse access with data product access creates a perverse incentive - popular data products would drive costs to the producing domain, discouraging data sharing.

```sql
-- Domain's own warehouse for its own users
GRANT USAGE ON WAREHOUSE DEV_SALES_TRNFRM TO ROLE DEV_SALES_READER;
```

For multiple warehouses or multiple privileges, use an access role granted to READER - see `warehouse-access-roles`.

---

## Data Product Admin Roles (Level 4)

The same pattern applies when you need an additional layer **below** domain for individual data products.

### When to Use Data Product Admins

| Scenario | Use |
|----------|-----|
| Team owns many data products | Domain Admin only - the team IS the domain |
| Team owns one data product, no larger domain | Domain Admin only - call it whatever fits |
| Multiple teams under one domain | Both - Domain Admin for shared concerns, Data Product Admin per team |

### Structure

Data Product Admins are identical to Domain Admins in structure:
- `<ENV>_<DOMAIN>_<PRODUCT>_SYSADMIN` - owns schema-bound objects
- `<ENV>_<DOMAIN>_<PRODUCT>_RBAC` - manages grants

They sit below Domain Admins in the hierarchy:
```
ENV_DOMAIN_SYSADMIN
    ↓ granted (inherits object ownership)
ENV_DOMAIN_PRODUCT_SYSADMIN
    ↓ owns objects in those schemas
```

### Example: HR Domain with Product Teams

```
PRD_HR_SYSADMIN (domain level)
    ├── PRD_HR_PAYROLL_SYSADMIN (product team)
    ├── PRD_HR_RECRUITMENT_SYSADMIN (product team)
    └── PRD_HR_BENEFITS_SYSADMIN (product team)
```

Each product team owns objects in their schemas. The domain admin inherits across all products and can access all product areas when needed.

### Data Product SQL Templates

#### Pattern A: Data Product Under Domain (with Environment Layer)

```sql
-- Variables
SET env = 'DEV';
SET domain = 'HR';
SET product = 'PAYROLL';

SET productSysadmin = $env || '_' || $domain || '_' || $product || '_SYSADMIN';
SET productRbac = $env || '_' || $domain || '_' || $product || '_RBAC';
SET productReader = $env || '_' || $domain || '_' || $product || '_READER';

-- Parent roles (domain level)
SET parentSysadmin = $env || '_' || $domain || '_SYSADMIN';
SET parentRbac = $env || '_' || $domain || '_RBAC';
SET parentReader = $env || '_' || $domain || '_READER';

-- Create data product roles
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($productSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productReader);

-- Build role hierarchy
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read access)
GRANT ROLE IDENTIFIER($productReader) TO ROLE IDENTIFIER($productSysadmin);

-- Grant to domain level
GRANT ROLE IDENTIFIER($productSysadmin) TO ROLE IDENTIFIER($parentSysadmin);
GRANT ROLE IDENTIFIER($productRbac) TO ROLE IDENTIFIER($parentRbac);
GRANT ROLE IDENTIFIER($productReader) TO ROLE IDENTIFIER($parentReader);

-- Grant account-level access roles
GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($productSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($productSysadmin);
```

#### Pattern B: Data Product Under Environment (No Domain Layer)

When a single team owns one data product, skip the domain layer.

```sql
-- Variables
SET env = 'DEV';
SET product = 'PAYROLL';

SET productSysadmin = $env || '_' || $product || '_SYSADMIN';
SET productRbac = $env || '_' || $product || '_RBAC';
SET productReader = $env || '_' || $product || '_READER';

-- Parent roles (environment level)
SET parentSysadmin = $env || '_SYSADMIN';
SET parentRbac = $env || '_RBAC';
SET parentReader = $env || '_READER';

-- Create data product roles
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($productSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productReader);

-- Build role hierarchy
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read access)
GRANT ROLE IDENTIFIER($productReader) TO ROLE IDENTIFIER($productSysadmin);

-- Grant to environment level
GRANT ROLE IDENTIFIER($productSysadmin) TO ROLE IDENTIFIER($parentSysadmin);
GRANT ROLE IDENTIFIER($productRbac) TO ROLE IDENTIFIER($parentRbac);
GRANT ROLE IDENTIFIER($productReader) TO ROLE IDENTIFIER($parentReader);

-- Grant account-level access roles
GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($productSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($productSysadmin);
```

#### Pattern C: Data Product Under Account (No Domain, No Environment Layer)

Multi-account by environment setup with single team per data product.

```sql
-- Variables (no env or domain prefix)
SET product = 'PAYROLL';

SET productSysadmin = $product || '_SYSADMIN';
SET productRbac = $product || '_RBAC';
SET productReader = $product || '_READER';

-- Parent roles (OOB account roles)
SET parentSysadmin = 'SYSADMIN';
SET parentRbac = 'SECURITYADMIN';

-- Create data product roles
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($productSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($productReader);

-- Build role hierarchy
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read access)
GRANT ROLE IDENTIFIER($productReader) TO ROLE IDENTIFIER($productSysadmin);

-- Grant to account level
GRANT ROLE IDENTIFIER($productSysadmin) TO ROLE IDENTIFIER($parentSysadmin);
GRANT ROLE IDENTIFIER($productRbac) TO ROLE IDENTIFIER($parentRbac);
GRANT ROLE IDENTIFIER($productReader) TO ROLE SYSADMIN;  -- No parent reader at OOB level

-- Grant account-level access roles
GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($productSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($productSysadmin);
```

---

## Key Points

- **Domain owns objects, not infrastructure** - tables, views, tasks, procedures, but NOT databases, schemas, or warehouses
- **Infrastructure provisioned by higher level** - Environment or Account admin creates DBs, schemas, WHs
- **CREATE grants enable object ownership** - Domain sysadmin creates and owns objects via granted CREATE privileges
- **RBAC manages access roles** - Domain RBAC owns and manages the <schema>_R, <schema>_RW, <schema>_C, DB_R, DB_RW, DB_C database roles
- **Cross-domain access requires escalation** - To access another domain's objects, escalate to higher level role
- **Data Product Admins follow same pattern** - Just an additional layer below domain when needed for discrete teams


### ENVIRONMENT ADMIN ROLES

---
name: environment-admin-roles
description: "Create Environment Admin roles for logical separation of Dev/Test/Prod within a single account. Use when: single account with multiple environments, account-per-domain setup, need to delegate environment administration."
---

# Environment Admin Roles

Account-level roles that provide logical separation between environments (Dev/Test/Prod) when they share the same Snowflake account.

## When You Need This

Environment Admin Roles are required when:
- **Option A**: Single account with all environments
- **Option C**: Account per business domain (environments share within each BU's account)

Skip this layer if environments are already in separate accounts (Options B or D).

## Purpose

Environment Admin Roles provide:
- **Logical isolation** - Dev admins can't accidentally modify Prod
- **Delegated administration** - Environment teams manage their own space
- **Clear boundaries** - All objects in an environment share a naming prefix

```
SYSADMIN
    ↓ creates DB/WH, transfers ownership
DEV_SYSADMIN / TST_SYSADMIN / PRD_SYSADMIN
    ↓ owns and manages
Environment databases, warehouses, roles
```

## Role Structure

For each environment, create a parallel set of admin roles:

| Role | Purpose |
|------|---------|
| `<ENV>_SYSADMIN` | Owns databases and warehouses for the environment. Does NOT manage roles. |
| `<ENV>_RBAC` | Manages grants and role hierarchy within the environment |
| `<ENV>_READER` | Read-only access path. Granted DB_R roles. Warehouse access for users within this environment. SYSADMIN inherits this. |

Common environment prefixes:
- `DEV` or `D` - Development
- `TST` or `T` - Test
- `UAT` or `U` - User Acceptance Testing
- `PRD` or `P` - Production

## Role Hierarchy

```
SYSADMIN ─────────────────────────────────────────────────────┐
    ↓ granted                                                 │
PRD_SYSADMIN ← TST_SYSADMIN ← DEV_SYSADMIN                   │
    ↓ granted                                                 │
PRD_READER ← TST_READER ← DEV_READER (DB_R + own warehouse)  │
                                                              │
SECURITYADMIN ────────────────────────────────────────────────┘
    ↓ granted
PRD_RBAC ← TST_RBAC ← DEV_RBAC
```

**Note**: Environment roles roll up to their OOB counterparts so central admins retain access when needed. Cross-environment access requires escalating to the OOB role (SYSADMIN, SECURITYADMIN) - environment admin roles should never grant across environment boundaries.

---

## SQL Template

### Variables
```sql
SET env = 'DEV';  -- Change for each environment: DEV, TST, PRD

SET envSysadmin = $env || '_SYSADMIN';
SET envRbac = $env || '_RBAC';
SET envReader = $env || '_READER';
```

### Create Environment Admin Roles
```sql
USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($envSysadmin);
CREATE ROLE IF NOT EXISTS IDENTIFIER($envRbac);
CREATE ROLE IF NOT EXISTS IDENTIFIER($envReader);
```

### Grant to OOB Roles
```sql
USE ROLE SECURITYADMIN;

-- READER granted to SYSADMIN (SYSADMIN inherits read + warehouse access)
GRANT ROLE IDENTIFIER($envReader) TO ROLE IDENTIFIER($envSysadmin);

-- Environment roles grant up to OOB roles
GRANT ROLE IDENTIFIER($envSysadmin) TO ROLE SYSADMIN;
GRANT ROLE IDENTIFIER($envRbac) TO ROLE SECURITYADMIN;
GRANT ROLE IDENTIFIER($envReader) TO ROLE SYSADMIN;  -- Central admin inherits all read access
```

### Grant Account-Level Access Roles
Environment admins typically need certain account-level privileges. Grant via the access roles created in `oob-account-roles`:

```sql
USE ROLE SECURITYADMIN;

GRANT ROLE _AR_EXEC_TASK TO ROLE IDENTIFIER($envSysadmin);
GRANT ROLE _AR_VIEW_AUSG TO ROLE IDENTIFIER($envSysadmin);
```

---

## Creating Environment Objects

Once environment admin roles exist, SYSADMIN creates objects and transfers ownership:

### Database Creation Pattern
```sql
SET env = 'DEV';
SET dbName = $env || '_SALES_RAW';
SET envSysadmin = $env || '_SYSADMIN';

USE ROLE SYSADMIN;

CREATE DATABASE IF NOT EXISTS IDENTIFIER($dbName);

GRANT OWNERSHIP ON DATABASE IDENTIFIER($dbName) TO ROLE IDENTIFIER($envSysadmin) COPY CURRENT GRANTS;
```

### Warehouse Creation Pattern
```sql
SET env = 'DEV';
SET whName = $env || '_SALES_INGEST';
SET envSysadmin = $env || '_SYSADMIN';

USE ROLE SYSADMIN;

CREATE WAREHOUSE IF NOT EXISTS IDENTIFIER($whName)
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

GRANT OWNERSHIP ON WAREHOUSE IDENTIFIER($whName) TO ROLE IDENTIFIER($envSysadmin) COPY CURRENT GRANTS;
```

---

## Naming Convention

All objects within an environment should be prefixed with the environment code:

| Object Type | Pattern | Example |
|-------------|---------|---------|
| Database | `<ENV>_<DOMAIN>_<ZONE>` | `DEV_SALES_RAW` |
| Warehouse | `<ENV>_<DOMAIN>_<WORKLOAD>` | `DEV_SALES_INGEST` |
| Functional Role | `<ENV>_<DOMAIN>_<ROLE>` | `DEV_SALES_CREATE` |

This ensures:
- Objects sort together by environment in UI lists
- Clear visual identification of environment
- Prevents accidental cross-environment operations

---

## Key Points

- **Logical separation only** - Environment admin roles don't provide hard isolation; that requires separate accounts
- **SYSADMIN creates, then transfers** - Central admin creates databases/warehouses, transfers ownership to environment admin
- **Naming prefix is critical** - All objects must include environment in name to maintain clarity
- **Roll up to OOB roles** - Environment admins granted to their OOB counterparts so central team retains access
- **Grant access roles, not privileges** - Use `_AR_` roles from `oob-account-roles`, don't grant ACCOUNTADMIN privileges directly


### OOB ACCOUNT ROLES

---
name: oob-account-roles
description: "Guide to Snowflake's out-of-box account roles and account-level access roles. Use when: setting up a new account, understanding ACCOUNTADMIN/SYSADMIN/SECURITYADMIN/USERADMIN, creating account-level access roles for delegated privileges."
---

# Out-of-Box Account Roles

How to use Snowflake's built-in roles correctly, plus account-level access roles that should exist in every account.

## The OOB Roles

Snowflake provides these roles out of the box. **Do not alter their privileges.**

| Role | Purpose | Use For |
|------|---------|---------|
| **ACCOUNTADMIN** | Superuser | Account-level settings, granting account privileges to access roles. **Never own objects with this role.** |
| **SYSADMIN** | Object administration | Creating databases, warehouses. Transfer ownership to delegated admins after creation. |
| **SECURITYADMIN** | Security administration | Granting privileges to roles, managing role hierarchy. |
| **USERADMIN** | User/role administration | Creating users and custom roles. |
| **PUBLIC** | Default role for all users | Minimal privileges, baseline for all sessions. |

## First Principles

1. **Never alter OOB role privileges** - Don't add or remove grants from ACCOUNTADMIN, SYSADMIN, etc.

2. **ACCOUNTADMIN should never own objects** - Use SYSADMIN or a custom role to create/own account objects.

3. **Don't grant privileges directly to functional roles** - Wrap privileges in access roles first, then grant those to functional roles.

4. **Use USERADMIN to create all custom roles** - Keeps role administration separate from object administration.

5. **Don't wrap OOB roles with custom roles of similar names** - Don't create SCIM_ACCOUNTADMIN or CUSTOM_SYSADMIN. If you need to assign OOB roles via SCIM, grant them directly to users via script.

## Account-Level Access Roles

These access roles wrap account-level privileges that would otherwise require ACCOUNTADMIN. Create these in every account regardless of architecture pattern.

### Why Access Roles?

ACCOUNTADMIN has powerful privileges. Rather than granting ACCOUNTADMIN to delegated admins, wrap specific privileges in access roles and grant those instead.

```
ACCOUNTADMIN privilege
    ↓ granted to
Access Role (_AR_EXEC_TASK)
    ↓ granted to
Functional Role (SALES_CREATE)
```

### Standard Access Roles

| Access Role | Privilege | Use Case |
|-------------|-----------|----------|
| `_AR_EXEC_TASK` | EXECUTE TASK ON ACCOUNT | Roles that own/run tasks |
| `_AR_VIEW_AUSG` | IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE | Roles that query Account Usage |
| `_AR_APPLY_TAG` | APPLY TAG ON ACCOUNT | Roles that apply tags to objects |
| `_AR_APPLY_DDM` | APPLY MASKING POLICY ON ACCOUNT | Roles that apply dynamic data masking |
| `_AR_APPLY_RAP` | APPLY ROW ACCESS POLICY ON ACCOUNT | Roles that apply row access policies |

The `_AR_` prefix and leading underscore:
- `_` prefix sorts these to the bottom of role lists
- `AR` indicates "Access Role"
- Distinguishes from functional roles that are granted to users

---

## SQL Template

### Create Access Roles
```sql
SET arPrefix = '_AR_';
SET viewAusgAr = $arPrefix || 'VIEW_AUSG';
SET execTaskAr = $arPrefix || 'EXEC_TASK';
SET applyTagAr = $arPrefix || 'APPLY_TAG';
SET applyDdmAr = $arPrefix || 'APPLY_DDM';
SET applyRapAr = $arPrefix || 'APPLY_RAP';

USE ROLE USERADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($viewAusgAr);
CREATE ROLE IF NOT EXISTS IDENTIFIER($execTaskAr);
CREATE ROLE IF NOT EXISTS IDENTIFIER($applyTagAr);
CREATE ROLE IF NOT EXISTS IDENTIFIER($applyDdmAr);
CREATE ROLE IF NOT EXISTS IDENTIFIER($applyRapAr);
```

### Grant Privileges to Access Roles
```sql
USE ROLE ACCOUNTADMIN;

GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE IDENTIFIER($viewAusgAr);
GRANT EXECUTE TASK ON ACCOUNT TO ROLE IDENTIFIER($execTaskAr);
GRANT APPLY TAG ON ACCOUNT TO ROLE IDENTIFIER($applyTagAr);
GRANT APPLY MASKING POLICY ON ACCOUNT TO ROLE IDENTIFIER($applyDdmAr);
GRANT APPLY ROW ACCESS POLICY ON ACCOUNT TO ROLE IDENTIFIER($applyRapAr);
```

### Grant Access Roles to Functional Roles
```sql
USE ROLE SECURITYADMIN;

GRANT ROLE IDENTIFIER($execTaskAr) TO ROLE <FUNCTIONAL_ROLE_THAT_RUNS_TASKS>;
GRANT ROLE IDENTIFIER($viewAusgAr) TO ROLE <FUNCTIONAL_ROLE_THAT_MONITORS>;
GRANT ROLE IDENTIFIER($applyTagAr) TO ROLE <FUNCTIONAL_ROLE_THAT_GOVERNS>;
GRANT ROLE IDENTIFIER($applyDdmAr) TO ROLE <FUNCTIONAL_ROLE_THAT_GOVERNS>;
GRANT ROLE IDENTIFIER($applyRapAr) TO ROLE <FUNCTIONAL_ROLE_THAT_GOVERNS>;
```

---

## Granting OOB Roles to Users

If using SCIM, don't create wrapper roles for OOB roles. Instead, use a versioned script:

```sql
USE ROLE ACCOUNTADMIN;
GRANT ROLE ACCOUNTADMIN TO USER <admin_user>;
GRANT ROLE SYSADMIN TO USER <admin_user>;

USE ROLE SECURITYADMIN;
GRANT ROLE SECURITYADMIN TO USER <security_user>;
GRANT ROLE USERADMIN TO USER <security_user>;
```

---

## Ownership Model

| Object | Created By | Owned By |
|--------|------------|----------|
| Access Roles | USERADMIN | USERADMIN |
| Databases | SYSADMIN | Delegated Admin (after transfer) |
| Warehouses | SYSADMIN | Delegated Admin (after transfer) |
| Custom Functional Roles | USERADMIN | USERADMIN (or SCIM provisioner) |

## Key Points

- **OOB roles are immutable** - use them as-is, don't modify their privileges
- **Access roles wrap privileges** - never grant account privileges directly to functional roles
- **SYSADMIN creates, then transfers** - databases and warehouses created by SYSADMIN, ownership transferred to delegated admin
- **USERADMIN owns custom roles** - unless you have a SCIM provisioner role


### PERSONAS

---
name: personas
description: "Persona-based access design framework. Use when: translating business responsibilities to roles, designing functional roles, choosing between persona-aligned vs data-product-aligned approaches, onboarding patterns."
---

# Personas

## What Is a Persona?

A **Persona** is a conceptual "hat" a human or service account wears - NOT a Snowflake object. It describes *how* someone interacts with the platform.

- One human may have **multiple personas** (e.g., Admin + Analyst)
- A service account typically has a **single persona**

Personas translate business responsibilities into access requirements **before** mapping to physical Snowflake roles.

---

## Persona Categories

### Maintainer Personas (Least Privilege)

Focus on **platform and data product operation**. Guiding principle: **minimum rights necessary**, accept granular roles and explicit role switching.

| Persona | Responsibilities |
|---------|-----------------|
| **Admin** | Account configuration, role hierarchy, warehouse provisioning |
| **Engineer** | Pipeline development, schema DDL, data loading/transformation |
| **Support** | Incident triage, query profiling, troubleshooting |
| **Governance** | Policy enforcement, tagging, masking, auditing |

### Analytic Personas (Least Effort)

Focus on **consuming and analyzing data**. Guiding principle: **minimize friction**, provide clear defaults (role, warehouse, namespace).

| Persona | Responsibilities |
|---------|-----------------|
| **Report Viewer** | Run pre-built dashboards; read-only |
| **Data Analyst** | Ad-hoc SQL, BI tool exploration |
| **Power Analyst / Data Scientist** | Advanced analytics, ML, sandbox write-back |
| **Governance Viewer** | Read-only audit logs, lineage, classification |

For analytic personas, use an **aggregate Functional Role** bundling all needed access - users "log in and run queries" without role switching.

---

## Two Design Approaches

### Approach A: Persona-Aligned Functional Roles

Each persona maps 1:1 to a Snowflake functional role containing all access roles the persona needs.

```
FUNCTIONAL_ROLE: FR_DATA_ANALYST
  ├── AR_FINANCE_READ
  ├── AR_MARKETING_READ
  ├── AR_WAREHOUSE_ANALYST_WH
  └── SANDBOX.DB_RW
```

| Pros | Cons |
|------|------|
| Business legibility - names match job functions | Custom bundles require hand-curation |
| Simple SCIM mapping (1 group = 1 role) | Persona splitting causes proliferation |
| Least-effort for consumers | Drift risk - stale/broad grants accumulate |
| Clear audit trail per persona | Tight coupling to org structure |

### Approach B: Data-Product-Aligned Functional Roles

Functional roles organized around **data products/domains**, not personas. Users granted specific domain roles they need.

```
USER: jane.doe
  ├── DR_FINANCE_ANALYST
  ├── DR_MARKETING_ANALYST
  └── DR_SANDBOX_POWERUSER
```

| Pros | Cons |
|------|------|
| Standardized, reusable building blocks | Multiple SCIM groups per user |
| Additive access - one grant per new domain | Requires secondary roles for cross-domain |
| Reduced proliferation (scales with domains) | Higher friction for consumers |
| Clear ownership by domain teams | Harder to audit "what can Jane do?" |

---

## Side-by-Side Comparison

| Dimension | Persona-Aligned (A) | Data-Product-Aligned (B) |
|-----------|---------------------|--------------------------|
| Role granularity | Coarse (per persona) | Fine (per domain) |
| SCIM complexity | Low (1:1) | Higher (N:N) |
| Maintenance burden | Higher (custom bundles) | Lower (templates) |
| Role proliferation | Higher over time | Lower, scales with domains |
| Consumer experience | Frictionless | Requires secondary roles |
| Cross-domain queries | Built-in | Requires aggregation |
| Auditability | Strong (named role) | Spread across roles |
| Exception handling | Low (forces forking) | High (grant/revoke) |

---

## Recommended Hybrid Approach

Most mature deployments combine both:

1. **Define personas conceptually** - for requirements, documentation, SCIM group naming
2. **Build data-product-aligned access roles** as atomic building blocks (Approach B)
3. **Create persona functional roles** for high-volume, well-defined populations (Approach A convenience layer)
4. **Enable secondary roles** for power users crossing domain boundaries
5. **Establish governance cadence** - quarterly reviews, grant diffing, clear escalation paths

This provides persona legibility for common cases while keeping architecture modular for exceptions.

---

## Mapping to RBAC Hierarchy

| Persona Category | Maps To |
|------------------|---------|
| Admin | Environment/Domain SYSADMIN + RBAC roles |
| Engineer | Data Product SYSADMIN roles |
| Analyst | READER roles (aggregated via functional role or secondary roles) |
| Governance | Account-level governance roles |

### Pattern: Analyst Persona with Secondary Roles
```sql
-- Grant domain READER roles
GRANT ROLE SALES_READER TO USER analyst_jane;
GRANT ROLE MARKETING_READER TO USER analyst_jane;
GRANT ROLE FINANCE_READER TO USER analyst_jane;

-- Enable secondary roles for seamless cross-domain access
ALTER USER analyst_jane SET DEFAULT_SECONDARY_ROLES = ('ALL');
```

### Pattern: Analyst Persona with Functional Role
```sql
-- Create aggregate functional role
CREATE ROLE FR_BUSINESS_ANALYST;

-- Grant all required access roles
GRANT ROLE SALES_READER TO ROLE FR_BUSINESS_ANALYST;
GRANT ROLE MARKETING_READER TO ROLE FR_BUSINESS_ANALYST;
GRANT ROLE FINANCE_READER TO ROLE FR_BUSINESS_ANALYST;
GRANT ROLE ANALYST_WH_USER TO ROLE FR_BUSINESS_ANALYST;

-- Grant functional role to user
GRANT ROLE FR_BUSINESS_ANALYST TO USER analyst_jane;

-- Set as default
ALTER USER analyst_jane SET DEFAULT_ROLE = 'FR_BUSINESS_ANALYST';
```

---

## Decision Guide

| If... | Then... |
|-------|---------|
| Well-defined, stable user populations | Approach A (persona-aligned) |
| Frequent exceptions / varied access needs | Approach B (data-product-aligned) + secondary roles |
| Simple SCIM integration priority | Approach A |
| Domain team ownership priority | Approach B (requires secondary roles for cross-domain) |

### On Disabling Secondary Roles

**Disabling secondary roles is an outlier, not a legitimate design choice.**

If your organisation has disabled secondary roles via session policy, this should be treated as a legacy constraint to work around - not a security posture to aspire to. Disabling secondary roles:
- Provides **no security benefit** (see `secondary-roles` skill for myth-busting)
- Blocks personal databases, sandbox schemas, and other self-service patterns
- Forces Approach A or complex hybrid workarounds
- Creates friction that drives users toward shadow IT alternatives

If secondary roles are disabled: Approach A or hybrid is **required**, not recommended.

---

## Data Product Access Provisioning (Approach B in Detail)

The other half of the access story: instead of provisioning by job role, **provision by Data Product**. Users request access to specific Data Products, not personas.

### The Model

```
Data Product Team                    Identity Provider (Okta/Entra)
       │                                        │
       ▼                                        ▼
┌─────────────────┐                   ┌─────────────────┐
│ SALES_READER    │ ◄──── SCIM ────► │ SNOW-SALES-READ │
│ (Snowflake Role)│                   │ (IdP Group)     │
└─────────────────┘                   └─────────────────┘
       │
       ▼
┌─────────────────┐
│ SALES.DB_R      │
│ (Database Role) │
└─────────────────┘
```

Each Data Product publishes:
1. A **Snowflake account role** (e.g., `SALES_READER`) that holds access to its database roles
2. A corresponding **IdP group** (e.g., `SNOW-SALES-READ`) mapped via SCIM

Users request membership in IdP groups. SCIM provisions the role grants. Secondary roles aggregate access at runtime.

### SCIM Integration Pattern

```
IdP Group                    Snowflake Role              Grants
─────────────────────────────────────────────────────────────────
SNOW-SALES-READ         →    SALES_READER           →    SALES.DB_R
SNOW-SALES-ETL          →    SALES_ETL              →    SALES.DB_RW
SNOW-MARKETING-READ     →    MARKETING_READER       →    MARKETING.DB_R
SNOW-FINANCE-READ       →    FINANCE_READER         →    FINANCE.DB_R
```

### Why This Works

| Benefit | Explanation |
|---------|-------------|
| **Self-service** | Users request access via IdP portal, no Snowflake admin involvement |
| **Data Product ownership** | Each team controls who gets access to their product |
| **Standardised** | Every Data Product follows the same pattern |
| **Auditable** | IdP provides access request history; Snowflake shows current grants |
| **Scalable** | Adding a new Data Product = new role + new IdP group |

### Setup: Data Product Side

```sql
-- Data Product team creates their READER role
CREATE ROLE SALES_READER;

-- Grant database role to account role
GRANT DATABASE ROLE SALES.DB_R TO ROLE SALES_READER;

-- NO WAREHOUSE GRANT HERE
-- Consumers bring their own compute via their domain's READER role

-- Transfer ownership to RBAC role for governance
GRANT OWNERSHIP ON ROLE SALES_READER TO ROLE SALES_RBAC COPY CURRENT GRANTS;
```

**Critical**: Data Product READER roles grant **data access only**, never warehouse access. Bundling compute with data access creates a perverse incentive - popular data products would drive consumption costs to the producing domain, discouraging data sharing.

### Setup: IdP Side (Okta Example)

1. Create group `SNOW-SALES-READ` in Okta
2. Configure SCIM provisioning to map group → Snowflake role `SALES_READER`
3. Users request group membership via Okta access request workflow
4. Approval (manual or automated) grants membership
5. SCIM syncs membership to Snowflake role grant

### User Experience

**Jane is a Marketing analyst who needs Sales and Marketing data:**

```sql
-- Jane's domain (Marketing) gives her warehouse access via her domain role:
--   GRANT ROLE MARKETING_ANALYST TO USER jane.doe;  (includes MARKETING_WH access)

-- Jane requests access to Sales data via IdP:
--   SCIM provisions: GRANT ROLE SALES_READER TO USER jane.doe;

-- Jane logs in with DEFAULT_SECONDARY_ROLES = ('ALL')
-- Secondary roles aggregate:
--   - MARKETING_ANALYST: warehouse access + Marketing data
--   - SALES_READER: Sales data access only (no warehouse)

-- She can query both using her Marketing warehouse:
SELECT * FROM SALES.CORE.CUSTOMERS;      -- data via SALES_READER, compute via MARKETING_ANALYST
SELECT * FROM MARKETING.CAMPAIGNS.DATA;  -- both via MARKETING_ANALYST
```

**Key point**: Jane's domain provides compute. Sales provides data. The cost of Jane's queries is borne by Marketing, not Sales - creating the right incentives for data sharing.

### Comparison: Persona vs Data Product Provisioning

| Aspect | Persona Provisioning | Data Product Provisioning |
|--------|---------------------|--------------------------|
| Access request | "I need Analyst access" | "I need Sales data access" |
| Approval | Central team decides what Analyst means | Data Product owner approves |
| IdP groups | Few (one per persona) | Many (one per Data Product) |
| Snowflake roles | Custom bundles | Standardised per product |
| Adding new data | Update persona role | User requests new product |
| Removing access | Complex (which products?) | Remove from specific group |
| Secondary roles | Optional | Required |
| Warehouse access | Bundled in persona role | Consumer's domain provides compute |


### POLICY ROLES

---
name: policy-roles
description: "Roles in data policies (Row Access, Masking, Projection, Aggregation). Use when: writing policy conditions, choosing between role-based and attribute-based policies, understanding IS_ROLE_IN_SESSION vs CURRENT_ROLE."
---

# Roles in Data Policies

How roles interact with Row Access Policies, Masking Policies, and other policy types.

---

## Core Principle: Policy Roles as Attributes, Not Access

**Policy roles should have NO direct access to data.**

The ability to query tables is encapsulated by **schema access roles** (`<schema>_R`, `<schema>_RW`). Policy roles exist solely as **user attributes** that modify runtime behaviour when checked by policies.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  SEPARATION OF CONCERNS                                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Schema Access Role (RBAC)          Policy Role (ABAC)                  │
│  ─────────────────────────          ────────────────────                │
│  • CUSTOMERS_R                      • PII_READER                        │
│  • Grants: SELECT on tables         • Grants: NONE (no data access)    │
│  • Purpose: CAN query               • Purpose: HOW query behaves       │
│                                                                          │
│  User queries table via CUSTOMERS_R                                     │
│  Policy checks PII_READER to determine masking level                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why This Matters

1. **Clear separation**: Access roles grant access; policy roles modify behaviour
2. **SCIM integration**: Policy roles can be provisioned via IdP (Okta/Entra) as user attributes
3. **Audit clarity**: Access audit shows WHO accessed data; policy roles show WHAT they saw
4. **Composability**: Same data access + different policy roles = different views of data

### Anti-Pattern: Policy Roles with Data Access

```sql
-- WRONG: Policy role that also grants table access
CREATE ROLE PII_READER;
GRANT SELECT ON customers TO ROLE PII_READER;  -- Don't do this!

-- CORRECT: Policy role is purely an attribute
CREATE ROLE PII_READER;  -- No grants to data objects
-- Access comes from schema role: CUSTOMERS_R
```

### RBAC + ABAC Combined

Policy roles enable **Attribute Based Access Control (ABAC)** layered on classic RBAC:

| Layer | Role Type | Purpose | Grants Data Access? |
|-------|-----------|---------|---------------------|
| RBAC | Schema access role (`CUSTOMERS_R`) | Permission to query | **Yes** |
| ABAC | Policy role (`PII_READER`) | Attribute checked by policy | **No** |

```sql
-- User: jane.doe
-- Roles: CUSTOMERS_R (schema access), PII_READER (policy attribute)
-- 
-- CUSTOMERS_R → Can SELECT from customers table
-- PII_READER  → Masking policy sees this role, returns unmasked PII
--
-- Without PII_READER: Same access, but PII columns masked
```

---

## Policy Types Overview

| Policy Type | Purpose | Common Role Usage |
|-------------|---------|-------------------|
| **Row Access Policy** | Filter rows based on context | Role determines which rows visible |
| **Masking Policy** | Transform column values | Role determines masking level |
| **Projection Policy** | Control which columns queryable | Role determines column visibility |
| **Aggregation Policy** | Require MIN_GROUP_SIZE | Role may bypass aggregation requirement |

---

## Choosing the Right Role Function

Four functions check roles in policies. Choosing the right one is critical.

| Function | Checks | Use When |
|----------|--------|----------|
| `CURRENT_ROLE()` | Primary role only | Almost never - legacy only |
| `IS_ROLE_IN_SESSION()` | Active account roles (primary + secondary) | Account roles, secondary roles enabled |
| `IS_DATABASE_ROLE_IN_SESSION()` | Active database roles | Schema/database access roles, data sharing |
| `CURRENT_AVAILABLE_ROLES()` | All granted roles | Check grants regardless of session state |

**Note:** `IS_DATABASE_ROLE_IN_SESSION()` is the only function that works across data shares and aligns with the `<schema>_R`, `<schema>_RW`, `DB_R`, `DB_RW` access role pattern.

### CURRENT_ROLE() - Avoid

Returns only the **primary role**. With secondary roles enabled (the default), this misses most effective privileges.

```sql
-- WRONG: Only checks primary role
CREATE OR REPLACE ROW ACCESS POLICY sales_rap
AS (region STRING) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('SALES_ADMIN', 'SALES_MANAGER')
  OR region = 'PUBLIC';

-- User with SALES_MANAGER as secondary role: DENIED (incorrect)
```

### IS_ROLE_IN_SESSION() - Recommended

Checks if a role is **currently active** as either primary OR secondary.

```sql
-- CORRECT: Checks all active roles
CREATE OR REPLACE ROW ACCESS POLICY sales_rap
AS (region STRING) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('SALES_ADMIN')
  OR IS_ROLE_IN_SESSION('SALES_MANAGER')
  OR region = 'PUBLIC';

-- User with SALES_MANAGER as secondary role: ALLOWED (correct)
```

**Use this when**: Secondary roles are enabled (default configuration). This is the recommended approach for most deployments.

### CURRENT_AVAILABLE_ROLES() - Granted Roles

Returns all roles **granted to the user**, regardless of whether they are currently active in the session.

```sql
-- Check if user has been granted the role (active or not)
CREATE OR REPLACE ROW ACCESS POLICY grants_based_rap
AS (region STRING) RETURNS BOOLEAN ->
  ARRAY_CONTAINS('SALES_MANAGER'::VARIANT, CURRENT_AVAILABLE_ROLES())
  OR region = 'PUBLIC';
```

**Use this when**: 
- You want policy to reflect **granted access**, not session state
- Session policies may restrict secondary roles but you want to honour the underlying grant
- Users may not have activated all their roles but should still have access

### Decision Matrix

| Scenario | Function |
|----------|----------|
| Account roles, secondary roles enabled | `IS_ROLE_IN_SESSION()` |
| Schema/database access roles | `IS_DATABASE_ROLE_IN_SESSION()` |
| Policies on shared data | `IS_DATABASE_ROLE_IN_SESSION()` |
| Honour grants regardless of session | `CURRENT_AVAILABLE_ROLES()` |
| Secondary roles disabled by policy | `IS_ROLE_IN_SESSION()` (respects restriction) |
| Legacy - primary role only | `CURRENT_ROLE()` (avoid if possible) |

### Key Difference: Session State vs Grants

```sql
-- User: jane.doe
-- Granted: SALES_MANAGER, FINANCE_READER
-- Session: Primary=SALES_MANAGER, Secondary roles=NONE (disabled by session policy)

CURRENT_ROLE()             → 'SALES_MANAGER'
IS_ROLE_IN_SESSION('FINANCE_READER') → FALSE (not active)
ARRAY_CONTAINS('FINANCE_READER'::VARIANT, CURRENT_AVAILABLE_ROLES()) → TRUE (granted)
```

If your organisation has **secondary roles enabled** (recommended), `IS_ROLE_IN_SESSION()` and `CURRENT_AVAILABLE_ROLES()` will behave similarly. The difference matters when:
- Session policies restrict secondary roles for specific users
- You want to check potential access vs current access

---

## IS_DATABASE_ROLE_IN_SESSION - Database & Schema Access Roles

For database roles (`<schema>_R`, `<schema>_RW`, `DB_R`, `DB_RW`), use `IS_DATABASE_ROLE_IN_SESSION()`.

### Why Database Roles in Policies?

1. **Data Sharing**: Only function that works across shares - consumer's database role grants flow through
2. **Access Role Alignment**: Directly checks the schema/database access roles you've already provisioned
3. **Encapsulation**: Policy logic stays within the database, portable with the data

### Basic Usage

```sql
CREATE OR REPLACE ROW ACCESS POLICY schema_rap
AS (sensitivity STRING) RETURNS BOOLEAN ->
  IS_DATABASE_ROLE_IN_SESSION('MYDB', 'SENSITIVE_R')
  OR sensitivity = 'PUBLIC';
```

### With Schema Access Roles

```sql
-- Policy aligned with <schema>_R, <schema>_RW pattern
CREATE OR REPLACE ROW ACCESS POLICY sales.customer_rap
AS (region STRING) RETURNS BOOLEAN ->
  IS_DATABASE_ROLE_IN_SESSION('SALES_DB', 'CUSTOMERS_R')    -- Full read access
  OR IS_DATABASE_ROLE_IN_SESSION('SALES_DB', 'CUSTOMERS_RW') -- ETL access
  OR region = 'PUBLIC';
```

### Data Sharing Scenario

```sql
-- Provider account: Policy on shared table
CREATE OR REPLACE ROW ACCESS POLICY shared_data_rap
AS (tenant_id STRING) RETURNS BOOLEAN ->
  IS_DATABASE_ROLE_IN_SESSION('SHARED_DB', tenant_id || '_READER');

-- Consumer account: Database role granted via share
-- Policy automatically enforces tenant isolation
```

This checks if the database role is granted to any active role (primary or secondary) in the session.

---

## Role-Based vs Attribute-Based Policies

### Role-Based Policies

Policy conditions check roles directly:

```sql
CREATE OR REPLACE MASKING POLICY pii_mask
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val
    WHEN IS_ROLE_IN_SESSION('ANALYST') THEN SHA2(val)
    ELSE '***MASKED***'
  END;
```

| Pros | Cons |
|------|------|
| Simple to understand | Policy must list every role |
| Direct mapping to RBAC | Adding roles requires policy update |
| Easy to audit | Can become unwieldy at scale |

### Attribute-Based Policies

Policy conditions check user/session attributes or mapping tables:

```sql
CREATE OR REPLACE ROW ACCESS POLICY region_rap
AS (region STRING) RETURNS BOOLEAN ->
  region IN (
    SELECT allowed_region 
    FROM access_control.user_regions 
    WHERE user_name = CURRENT_USER()
  );
```

| Pros | Cons |
|------|------|
| Scales better | Requires mapping table maintenance |
| Policy unchanged when access changes | Harder to audit (check table, not policy) |
| Supports complex logic | Query in policy can impact performance |

### Hybrid: Role-to-Attribute Mapping

Best of both worlds - policy checks roles, but role membership determines attribute access:

```sql
CREATE OR REPLACE ROW ACCESS POLICY region_rap
AS (region STRING) RETURNS BOOLEAN ->
  EXISTS (
    SELECT 1 
    FROM access_control.role_regions rr
    WHERE rr.region = region
      AND IS_ROLE_IN_SESSION(rr.role_name)
  );
```

The mapping table:
```sql
CREATE TABLE access_control.role_regions (
  role_name STRING,
  region STRING
);

INSERT INTO access_control.role_regions VALUES
  ('SALES_EMEA_READER', 'EMEA'),
  ('SALES_APAC_READER', 'APAC'),
  ('SALES_AMER_READER', 'AMER'),
  ('SALES_GLOBAL_READER', 'EMEA'),
  ('SALES_GLOBAL_READER', 'APAC'),
  ('SALES_GLOBAL_READER', 'AMER');
```

---

## Row Access Policy Patterns

### Pattern 1: Simple Role Check

```sql
CREATE OR REPLACE ROW ACCESS POLICY sensitive_data_rap
AS (sensitivity_level STRING) RETURNS BOOLEAN ->
  CASE sensitivity_level
    WHEN 'PUBLIC' THEN TRUE
    WHEN 'INTERNAL' THEN IS_ROLE_IN_SESSION('INTERNAL_READER')
    WHEN 'CONFIDENTIAL' THEN IS_ROLE_IN_SESSION('CONFIDENTIAL_READER')
    WHEN 'RESTRICTED' THEN IS_ROLE_IN_SESSION('RESTRICTED_READER')
    ELSE FALSE
  END;
```

### Pattern 2: Hierarchical Roles

Higher roles can see lower sensitivity levels:

```sql
CREATE OR REPLACE ROW ACCESS POLICY tiered_rap
AS (tier INT) RETURNS BOOLEAN ->
  (tier <= 1)  -- PUBLIC
  OR (tier <= 2 AND IS_ROLE_IN_SESSION('TIER2_READER'))
  OR (tier <= 3 AND IS_ROLE_IN_SESSION('TIER3_READER'))
  OR IS_ROLE_IN_SESSION('FULL_ACCESS');
```

### Pattern 3: Owner Sees All

Data owners bypass restrictions:

```sql
CREATE OR REPLACE ROW ACCESS POLICY ownership_rap
AS (owner_domain STRING) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION(owner_domain || '_SYSADMIN')
  OR IS_ROLE_IN_SESSION(owner_domain || '_READER')
  OR owner_domain = 'SHARED';
```

---

## Masking Policy Patterns

### Pattern 1: Tiered Masking

```sql
CREATE OR REPLACE MASKING POLICY email_mask
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_FULL') THEN val
    WHEN IS_ROLE_IN_SESSION('PII_PARTIAL') THEN 
      REGEXP_REPLACE(val, '^(.{2}).*(@.*)$', '\\1***\\2')
    ELSE '***@***.***'
  END;

-- PII_FULL sees: john.smith@company.com
-- PII_PARTIAL sees: jo***@company.com
-- Others see: ***@***.***
```

### Pattern 2: Conditional Unmasking

```sql
CREATE OR REPLACE MASKING POLICY ssn_mask
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN IS_ROLE_IN_SESSION('HR_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION('HR_VIEWER') THEN 'XXX-XX-' || RIGHT(val, 4)
    ELSE 'XXX-XX-XXXX'
  END;
```

### Pattern 3: Context-Aware Masking

Mask based on both role AND data context:

```sql
CREATE OR REPLACE MASKING POLICY salary_mask
AS (val NUMBER, department STRING) RETURNS NUMBER ->
  CASE
    WHEN IS_ROLE_IN_SESSION('FINANCE_ADMIN') THEN val
    WHEN IS_ROLE_IN_SESSION(department || '_MANAGER') THEN val
    ELSE NULL
  END;
```

---

## Projection Policy Patterns

Projection policies control whether a column can be included in query results.

### Pattern: Role-Based Column Access

```sql
CREATE OR REPLACE PROJECTION POLICY salary_projection
AS () RETURNS PROJECTION_CONSTRAINT ->
  CASE
    WHEN IS_ROLE_IN_SESSION('HR_ADMIN') THEN PROJECTION_CONSTRAINT(ALLOW => TRUE)
    WHEN IS_ROLE_IN_SESSION('MANAGER') THEN PROJECTION_CONSTRAINT(ALLOW => TRUE)
    ELSE PROJECTION_CONSTRAINT(ALLOW => FALSE)
  END;
```

---

## Aggregation Policy Patterns

Aggregation policies require results to have a minimum group size (k-anonymity).

### Pattern: Role Bypass

```sql
CREATE OR REPLACE AGGREGATION POLICY min_group_policy
AS () RETURNS AGGREGATION_CONSTRAINT ->
  CASE
    WHEN IS_ROLE_IN_SESSION('RESEARCH_FULL') THEN AGGREGATION_CONSTRAINT(MIN_GROUP_SIZE => 0)
    ELSE AGGREGATION_CONSTRAINT(MIN_GROUP_SIZE => 10)
  END;
```

---

## Performance Considerations

### Avoid Complex Subqueries

Policy conditions execute for every row. Complex subqueries can devastate performance.

**Slow:**
```sql
CREATE OR REPLACE ROW ACCESS POLICY slow_rap
AS (region STRING) RETURNS BOOLEAN ->
  EXISTS (
    SELECT 1 FROM complex_view v
    JOIN another_table t ON v.id = t.id
    WHERE t.region = region
      AND v.user = CURRENT_USER()
  );
```

**Better:**
```sql
-- Pre-compute access in a simple mapping table
CREATE OR REPLACE ROW ACCESS POLICY fast_rap
AS (region STRING) RETURNS BOOLEAN ->
  EXISTS (
    SELECT 1 FROM access_control.user_regions
    WHERE user_name = CURRENT_USER()
      AND allowed_region = region
  );
```

### IS_ROLE_IN_SESSION Performance

`IS_ROLE_IN_SESSION()` is highly optimised - it does not query the role hierarchy at runtime. Multiple calls in a single policy are acceptable.

---

## Policy Role Design Guidelines

### 1. Policy Roles Have NO Data Grants

Policy roles are attributes, not access roles. They receive **zero grants to data objects**.

```sql
-- Policy roles - NO data access grants
CREATE ROLE PII_READER;        -- Can see PII data unmasked
CREATE ROLE RESTRICTED_READER; -- Can see restricted rows
CREATE ROLE RESEARCH_FULL;     -- Bypasses aggregation

-- These roles have NO grants to tables, views, or schemas
-- Data access comes from schema access roles (<schema>_R, <schema>_RW)

-- Grant policy roles to users/functional roles as attributes
GRANT ROLE PII_READER TO ROLE HR_ANALYST;
GRANT ROLE RESTRICTED_READER TO ROLE COMPLIANCE_TEAM;
```

### 2. SCIM Provisioning for Policy Roles

Policy roles are ideal for IdP provisioning via SCIM:

```
IdP Group: "PII-Authorised"  →  Snowflake Role: PII_READER
IdP Group: "Research-Team"   →  Snowflake Role: RESEARCH_FULL
```

The IdP manages **who has the attribute**; the policy manages **what the attribute means**.

### 3. Policy Roles in RBAC Hierarchy

Policy roles should be:
- **Owned** by a central governance/RBAC role
- **Granted** to functional roles or directly via SCIM
- **Never** granted data access

```
GOVERNANCE_RBAC (owns policy roles)
    ↓
PII_READER, RESTRICTED_READER, etc. (NO data grants)
    ↓ granted to (as attributes)
HR_ANALYST, COMPLIANCE_TEAM, or directly to users via SCIM
    ↓
Users (access data via schema roles, modified by policy roles)
```

### 4. Audit Policy Role Grants

```sql
-- Who has PII access?
SELECT grantee_name, granted_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE role = 'PII_READER'
  AND deleted_on IS NULL;
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| **Policy roles with data grants** | Conflates access and attributes | Policy roles have NO data grants - access via schema roles only |
| Using `CURRENT_ROLE()` | Ignores secondary roles | Use `IS_ROLE_IN_SESSION()` or `CURRENT_AVAILABLE_ROLES()` |
| Wrong function choice | `IS_ROLE_IN_SESSION` vs `CURRENT_AVAILABLE_ROLES` confusion | See Decision Matrix - active session vs granted roles |
| Hardcoding role lists | Policy needs update for new roles | Use mapping table or role hierarchy |
| Complex policy subqueries | Performance degradation | Pre-compute access in simple tables |
| Admin roles in policies | Over-privileged, hard to audit | Create purpose-built policy roles |
| No default deny | Missing roles = full access | Always include `ELSE FALSE/MASKED` |

---

## SQL Templates

### Create Policy Role Structure

```sql
-- Variables
SET policyDomain = 'DATA_GOVERNANCE';
SET rbacRole = $policyDomain || '_RBAC';

-- Create policy roles
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS PII_NONE;      -- No PII access (default)
CREATE ROLE IF NOT EXISTS PII_PARTIAL;   -- Partial PII (last 4 digits, etc.)
CREATE ROLE IF NOT EXISTS PII_FULL;      -- Full PII access

CREATE ROLE IF NOT EXISTS RESTRICTED_READER;
CREATE ROLE IF NOT EXISTS CONFIDENTIAL_READER;

-- Establish hierarchy
GRANT ROLE PII_NONE TO ROLE PII_PARTIAL;
GRANT ROLE PII_PARTIAL TO ROLE PII_FULL;

-- Transfer ownership to governance
GRANT OWNERSHIP ON ROLE PII_NONE TO ROLE IDENTIFIER($rbacRole) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON ROLE PII_PARTIAL TO ROLE IDENTIFIER($rbacRole) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON ROLE PII_FULL TO ROLE IDENTIFIER($rbacRole) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON ROLE RESTRICTED_READER TO ROLE IDENTIFIER($rbacRole) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON ROLE CONFIDENTIAL_READER TO ROLE IDENTIFIER($rbacRole) COPY CURRENT GRANTS;
```

### Standard Masking Policy

```sql
CREATE OR REPLACE MASKING POLICY governance.pii_string_mask
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_FULL') THEN val
    WHEN IS_ROLE_IN_SESSION('PII_PARTIAL') THEN 
      CASE 
        WHEN LENGTH(val) > 4 THEN REPEAT('*', LENGTH(val) - 4) || RIGHT(val, 4)
        ELSE REPEAT('*', LENGTH(val))
      END
    ELSE '***MASKED***'
  END;
```

### Standard Row Access Policy

```sql
CREATE OR REPLACE ROW ACCESS POLICY governance.sensitivity_rap
AS (sensitivity STRING) RETURNS BOOLEAN ->
  sensitivity = 'PUBLIC'
  OR (sensitivity = 'INTERNAL' AND IS_ROLE_IN_SESSION('INTERNAL_READER'))
  OR (sensitivity = 'CONFIDENTIAL' AND IS_ROLE_IN_SESSION('CONFIDENTIAL_READER'))
  OR (sensitivity = 'RESTRICTED' AND IS_ROLE_IN_SESSION('RESTRICTED_READER'))
  OR IS_ROLE_IN_SESSION('DATA_GOVERNANCE_ADMIN');
```

---

## Summary

| Do | Don't |
|----|-------|
| **Policy roles = attributes with NO data grants** | Grant data access to policy roles |
| Use schema access roles for data access | Mix access and attributes in one role |
| Use `IS_ROLE_IN_SESSION()` for active session checks | Use `CURRENT_ROLE()` |
| Use `IS_DATABASE_ROLE_IN_SESSION()` for schema/DB roles | Use account role functions for database roles |
| Use `CURRENT_AVAILABLE_ROLES()` for grant-based checks | Confuse active vs granted roles |
| Provision policy roles via SCIM as user attributes | Manually manage policy role membership |
| Include default deny (`ELSE FALSE`) | Assume missing condition = deny |
| Keep policy logic simple | Put complex joins in policies |
| Own policy roles under governance | Let policy roles float unowned |


### SCHEMA ACCESS ROLES

---
name: schema-access-roles
description: "Create and manage Schema Access Roles - Database Roles providing tiered Read/Write/Create access to a schema. Use when: setting up schema permissions, creating database roles for schema access, implementing RBAC at schema level."
---

# Schema Access Roles

Database Roles that provide three tiers of access to a schema's objects.

## Why Database Roles (Not Account Roles)

Schema Access Roles must be Database Roles, not Account Roles:

| Benefit | Explanation |
|---------|-------------|
| **Scoped by design** | Database Roles cannot see outside their database - enforces least-privilege automatically |
| **Clone-friendly** | Cloning a database clones its Database Roles - no role rebuilding required |
| **No UI clutter** | Database Roles don't appear in the account role list, keeping it clean |
| **Ownership safety** | Cannot be used with USE ROLE, preventing accidental object ownership via "creator owns" default |

**CRITICAL: Database Roles must NEVER own objects.** Because Database Roles cannot see outside their database, they cannot own:
- Views referencing other databases
- Dynamic tables (require warehouse, which lives outside the database)
- Tasks (require warehouse)
- Any object with cross-database dependencies

This was always suboptimal but now makes certain patterns completely impossible. Objects should be owned by an Account Role (typically a dedicated admin role), not the Database Role that has CREATE privileges.

**Why Schema level?** Object-level permissions are too granular (unmanageable at scale), database-level is too broad (violates least-privilege). Schema is the sweet spot - logical groupings of related objects with cohesive access needs.

## Access Tiers

| Tier | Prefix | Privileges | Use Case |
|------|--------|------------|----------|
| READ | `<schema>_R` | SELECT on data objects | Analysts, reporting |
| READ-WRITE | `<schema>_RW` | INSERT, UPDATE, DELETE + operational | ETL, applications |
| CREATE | `<schema>_C` | DDL (CREATE objects, but NOT ownership) | Developers, admins |

## Role Hierarchy

```
<schema>_C  (Create - highest)
    ↓ inherits
<schema>_RW  (Read-Write)
    ↓ inherits
<schema>_R  (Read - lowest)
```

## Workflow

### Step 1: Gather Requirements
Ask user for:
- Database name
- Schema name (new or existing)

**MANAGED ACCESS is required.** Always create schemas with MANAGED ACCESS. Without it, any user with CREATE privileges can grant access to objects they create, bypassing centralized access control. Only in rare legacy migration scenarios should non-MANAGED ACCESS be considered.

### Step 2: Generate SQL
Use the template below, substituting database and schema names.

### Step 3: Execute
Run SQL in Snowflake with appropriate admin privileges.

---

## SQL Template

### Variables
```sql
SET dbNm = '<DATABASE_NAME>';
SET scNm = '<SCHEMA_NAME>';
SET sarR = $scNm || '_R';
SET sarRW = $scNm || '_RW';
SET sarC = $scNm || '_C';
USE DATABASE IDENTIFIER($dbNm);
```

### Create Schema (MANAGED ACCESS)
```sql
CREATE SCHEMA IF NOT EXISTS IDENTIFIER($scNm) WITH MANAGED ACCESS;
```

### READ Role (<schema>_R)
```sql
CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($sarR);
GRANT USAGE, MONITOR ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);

-- Current objects: Tables and table-like
GRANT SELECT ON ALL TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL VIEWS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL MATERIALIZED VIEWS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL EXTERNAL TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL DYNAMIC TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL ICEBERG TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL EVENT TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON ALL STREAMS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);

-- Current objects: Executable/callable
GRANT USAGE ON ALL FUNCTIONS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);

-- Future objects: Tables and table-like
GRANT SELECT ON FUTURE TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE VIEWS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE MATERIALIZED VIEWS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE EXTERNAL TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE DYNAMIC TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE ICEBERG TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE EVENT TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
GRANT SELECT ON FUTURE STREAMS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);

-- Future objects: Executable/callable
GRANT USAGE ON FUTURE FUNCTIONS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarR);
```

### READ-WRITE Role (<schema>_RW)
```sql
CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($sarRW);

-- DML on tables
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON ALL TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON ALL ICEBERG TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT INSERT ON ALL EVENT TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Operational: Sequences, formats, stages
GRANT USAGE ON ALL SEQUENCES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT USAGE ON ALL FILE FORMATS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT USAGE ON ALL STAGES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT READ, WRITE ON ALL STAGES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Operational: Pipelines and automation
GRANT OPERATE, MONITOR ON ALL TASKS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE ON ALL DYNAMIC TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE ON ALL ALERTS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT MONITOR, OPERATE ON ALL PIPES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Operational: Secrets and Git
GRANT USAGE ON ALL SECRETS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT READ ON ALL GIT REPOSITORIES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Operational: SPCS (Snowpark Container Services)
GRANT READ ON ALL IMAGE REPOSITORIES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE, MONITOR ON ALL SERVICES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Future DML on tables
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON FUTURE TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT INSERT, UPDATE, DELETE, TRUNCATE ON FUTURE ICEBERG TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT INSERT ON FUTURE EVENT TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Future operational: Sequences, formats, stages
GRANT USAGE ON FUTURE SEQUENCES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT USAGE ON FUTURE FILE FORMATS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT USAGE ON FUTURE STAGES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT READ, WRITE ON FUTURE STAGES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Future operational: Pipelines and automation
GRANT OPERATE, MONITOR ON FUTURE TASKS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE ON FUTURE DYNAMIC TABLES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE ON FUTURE ALERTS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT MONITOR, OPERATE ON FUTURE PIPES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Future operational: Secrets and Git
GRANT USAGE ON FUTURE SECRETS IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT READ ON FUTURE GIT REPOSITORIES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Future operational: SPCS
GRANT READ ON FUTURE IMAGE REPOSITORIES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT OPERATE, MONITOR ON FUTURE SERVICES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);

-- Procedures (can modify data under owner's rights)
GRANT USAGE ON ALL PROCEDURES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT USAGE ON FUTURE PROCEDURES IN SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarRW);
```

### CREATE Role (<schema>_C)
**NOTE:** This role grants the ability to CREATE objects, but the Database Role must NEVER own them. Objects created while using an Account Role that has this Database Role granted will be owned by that Account Role, not the Database Role.

```sql
CREATE DATABASE ROLE IF NOT EXISTS IDENTIFIER($sarC);

-- Tables and table-like objects
GRANT CREATE TABLE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE VIEW ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE MATERIALIZED VIEW ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE EXTERNAL TABLE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE DYNAMIC TABLE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE ICEBERG TABLE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE EVENT TABLE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE STREAM ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Code objects
GRANT CREATE FUNCTION ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE PROCEDURE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Pipeline and automation
GRANT CREATE TASK ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE ALERT ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE PIPE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Data loading infrastructure
GRANT CREATE STAGE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE FILE FORMAT ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE SEQUENCE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Git and secrets
GRANT CREATE SECRET ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE GIT REPOSITORY ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- SPCS (Snowpark Container Services)
GRANT CREATE IMAGE REPOSITORY ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE SERVICE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE SNAPSHOT ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Applications
GRANT CREATE STREAMLIT ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE NOTEBOOK ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- AI/ML and Cortex
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE SEMANTIC VIEW ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE MODEL ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE DATASET ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);

-- Governance/policy (optional - may restrict to admin schemas)
GRANT CREATE NETWORK RULE ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE TAG ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE MASKING POLICY ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
GRANT CREATE ROW ACCESS POLICY ON SCHEMA IDENTIFIER($scNm) TO DATABASE ROLE IDENTIFIER($sarC);
```

### Establish Hierarchy
```sql
GRANT DATABASE ROLE IDENTIFIER($sarR) TO DATABASE ROLE IDENTIFIER($sarRW);
GRANT DATABASE ROLE IDENTIFIER($sarRW) TO DATABASE ROLE IDENTIFIER($sarC);
```

---

## Key Points

- **MANAGED ACCESS**: Required. Only schema owner can grant access, not object owners. Without MANAGED ACCESS, access control becomes fragmented and unauditable.
- **Future Grants**: Essential for new objects to automatically receive proper permissions.
- **Hierarchy**: Create role inherits Read-Write, Read-Write inherits Read. Grant at lowest needed level.
- **Naming**: `<schema>_R`, `<schema>_RW`, `<schema>_C` suffixes make roles self-documenting.
- **CREATE ≠ OWN**: The CREATE role grants DDL privileges, but Database Roles must NEVER own objects. Ownership stays with an Account Role.


### SECONDARY ROLES

---
name: secondary-roles
description: "Understanding and implementing secondary roles. Use when: users need privileges from multiple roles simultaneously, simplifying end-user experience, avoiding role-switching, dispelling security myths."
---

# Secondary Roles

## Dispelling the Myths

Secondary roles cause trepidation in some users. **This concern is unfounded.**

### "Secondary roles are a security weakness"
**FALSE.** A determined bad actor could accomplish the same outcomes without secondary roles as they could with them. Secondary roles do not grant any privileges the user doesn't already have - they simply allow using multiple granted roles simultaneously rather than switching between them.

### "Secondary roles bypass access control"
**FALSE.** All roles (primary and secondary) must first be **granted to the user** before they can be activated. Secondary roles aggregate privileges the user already has - nothing more.

### "We should disable secondary roles for security"
**COUNTERPRODUCTIVE.** Disabling secondary roles forces workarounds that are harder to audit:
- Creating custom "super roles" per user that aggregate privileges
- Building utilities to stitch together role grants
- Users switching roles mid-execution (harder to trace)

Secondary roles make access patterns **more transparent**, not less.

---

## What Are Secondary Roles

- Every Snowflake session has exactly one **primary role** (the current role in context)
- A set of **secondary roles** can be activated simultaneously
- Effective privileges = **primary role + all active secondary roles**
- Only roles **already granted to the user** can be activated

```
User: analyst_jane
├── Granted: SALES_READER, MARKETING_READER, FINANCE_READER
│
└── Session with secondary roles:
    Primary: SALES_READER
    Secondary: MARKETING_READER, FINANCE_READER
    ─────────────────────────────────────────────
    Effective: All privileges from all three roles
```

---

## Why Secondary Roles Matter

### The Problem They Solve
Users often need privileges from **multiple roles at once**:
- An analyst querying data from Sales, Marketing, AND Finance
- A dashboard pulling from multiple data products
- A report spanning multiple subject areas

### Without Secondary Roles
The workarounds are painful:
1. **One custom role per user** - Grants all needed privileges, but becomes unmanageable at scale
2. **Role switching mid-execution** - User runs `USE ROLE` repeatedly, breaking workflows
3. **Custom utilities** - Building tools to "stitch together" role grants

### With Secondary Roles
- Users work with **all their granted roles active** in a single session
- No need to know which role grants access to which object
- Simplified administration - grant roles normally, let secondary roles handle the rest

---

## When to Use Secondary Roles

### YES - Use For:
| Use Case | Why |
|----------|-----|
| **Analytics/Reporting users** | Access all data products without role switching |
| **Dashboard consumers** | Single session spans multiple subject areas |
| **Read-only access patterns** | Users with multiple READER roles |
| **Cross-domain data consumers** | Query data from multiple domains simultaneously |

### NO - Do Not Use For:
| Use Case | Why |
|----------|-----|
| **Admin/maintenance roles** | Powerful roles should be explicit, not aggregated |
| **Engineering/pipeline roles** | Clear ownership semantics required |
| **Object creation (DDL)** | Ownership tied to primary role only |

---

## How Secondary Roles Work

### User-Level Default (Recommended)
Set once, applies to every session:

```sql
-- Enable all granted roles as secondary by default
ALTER USER my_user SET DEFAULT_SECONDARY_ROLES = ('ALL');

-- Or specific roles only
ALTER USER my_user SET DEFAULT_SECONDARY_ROLES = ('SALES_READER', 'MARKETING_READER');
```

### Session-Level Commands
For ad-hoc activation:

```sql
-- Activate all eligible secondary roles
USE SECONDARY ROLES ALL;

-- Activate specific roles only
USE SECONDARY ROLES SALES_READER, MARKETING_READER;

-- Deactivate all secondary roles
USE SECONDARY ROLES NONE;
```

### Default Behaviour
As of BCR-1692 (2024_08 bundle), new users are created with `DEFAULT_SECONDARY_ROLES = ('ALL')` - secondary roles are active by default.

To change the default for a user:
```sql
-- Set default to no secondary roles
ALTER USER my_user SET DEFAULT_SECONDARY_ROLES = ();

-- Set default to specific roles only
ALTER USER my_user SET DEFAULT_SECONDARY_ROLES = ('SALES_READER', 'MARKETING_READER');
```

**Important**: This only sets the default. Users can still manually run `USE SECONDARY ROLES ALL` in their session. To truly prevent secondary roles, use a **session policy**.

### Enforcing with Session Policies
Session policies (Enterprise Edition) can restrict which secondary roles users can activate in-session:

```sql
-- Disallow all secondary roles
CREATE SESSION POLICY no_secondary_roles
  ALLOWED_SECONDARY_ROLES = ();

-- Allow only specific roles as secondary
CREATE SESSION POLICY limited_secondary_roles
  ALLOWED_SECONDARY_ROLES = ('SALES_READER', 'MARKETING_READER');

-- Block specific roles from being used as secondary (takes precedence over allowed)
CREATE SESSION POLICY block_admin_secondary
  BLOCKED_SECONDARY_ROLES = ('ACCOUNTADMIN', 'SECURITYADMIN', 'SYSADMIN');

-- Apply to account or user
ALTER ACCOUNT SET SESSION POLICY no_secondary_roles;
ALTER USER my_user SET SESSION POLICY limited_secondary_roles;
```

| Parameter | Effect |
|-----------|--------|
| `ALLOWED_SECONDARY_ROLES = ()` | Disallows all secondary roles |
| `ALLOWED_SECONDARY_ROLES = ('ALL')` | Allows all secondary roles (default) |
| `ALLOWED_SECONDARY_ROLES = ('role1', 'role2')` | Only these roles can be secondary |
| `BLOCKED_SECONDARY_ROLES = ('role1')` | These roles cannot be secondary (overrides allowed) |

---

## Ownership and DDL

**Critical distinction**: Secondary roles authorize **DML operations** but **ownership and DDL authority remain tied to the primary role**.

| Operation | Uses Secondary Roles? |
|-----------|----------------------|
| SELECT, INSERT, UPDATE, DELETE | Yes - aggregated privileges |
| CREATE TABLE, CREATE VIEW | **No** - primary role only |
| DROP, ALTER | **No** - primary role owns |
| GRANT, REVOKE | **No** - primary role context |

Objects created in a session are **owned by the primary role**, regardless of which secondary roles are active.

---

## Features That Don't Support Secondary Roles

Features using **owner's rights execution** do NOT use secondary roles:

| Feature | Reason |
|---------|--------|
| Dynamic Tables | Refreshes execute as owner role only |
| Streamlit Apps | Run with owner's rights execution model |
| Materialized Views | Background maintenance runs as owner role |
| Tasks (default) | Run with privileges of task owner role |
| Owner's Rights Stored Procedures | Execute with owner role privileges only |
| UDFs | Execute as owner by default |

### Exception: Tasks with EXECUTE AS USER
Tasks can be configured to run with a specific user's privileges using `EXECUTE AS USER`:
```sql
CREATE TASK my_task
  WAREHOUSE = my_wh
  SCHEDULE = '1 HOUR'
  EXECUTE AS USER service_user
  AS
    SELECT * FROM my_table;
```
When using `EXECUTE AS USER`:
- The task runs on behalf of the specified user
- **The user's DEFAULT_SECONDARY_ROLES are activated automatically**
- The owner role must have `IMPERSONATE` privilege on the user
- The user must be granted the task owner role

This is valuable when:
- Tasks need access across multiple roles
- Data masking/row access policies depend on the querying user
- Clear audit trails attributing activity to specific users are required
- Data Product architecture where a developer with access to many data products wants to combine and publish them without requiring each data provider to grant access to their domain SYSADMIN

### Solution for Other Features: Grant Required Roles to Owner
```sql
-- Owner role inherits privileges from child roles
GRANT ROLE SALES_READER TO ROLE DYNAMIC_TABLE_OWNER;
GRANT ROLE WAREHOUSE_USER TO ROLE DYNAMIC_TABLE_OWNER;
```

### Caller's Rights Procedures
Use caller's rights when secondary role access is needed:
```sql
CREATE PROCEDURE my_proc()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER  -- Will use caller's primary + secondary roles
AS
$$
  SELECT * FROM sales.data;
$$;
```

---

## API and Connector Support

### Connectors/Drivers
| Connector | Support | How |
|-----------|---------|-----|
| JDBC | ✓ | Execute `USE SECONDARY ROLES` after connection |
| ODBC | ✓ | Execute `USE SECONDARY ROLES` after connection |
| Python | ✓ | Execute `USE SECONDARY ROLES` after connection |
| Snowpark | ✓ | Session-based, supports secondary roles |

**No connector has a direct connection parameter for secondary roles.** Options:
1. Execute `USE SECONDARY ROLES` after connecting
2. Set `DEFAULT_SECONDARY_ROLES` at user level (recommended - applies automatically)
3. For scheduled workloads: Use Tasks with `EXECUTE AS USER` to inherit user's secondary roles

```python
import snowflake.connector

conn = snowflake.connector.connect(...)
conn.cursor().execute("USE SECONDARY ROLES ALL")
```

### REST API
- `X-Snowflake-Role` header sets **primary role only**
- No header exists for secondary roles
- **Workaround**: Execute `USE SECONDARY ROLES` via SQL API endpoint
- **Better**: Set `DEFAULT_SECONDARY_ROLES` at user level

```http
POST /api/v2/statements HTTP/1.1
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "statement": "USE SECONDARY ROLES ALL",
  "warehouse": "MY_WAREHOUSE",
  "role": "MY_PRIMARY_ROLE"
}
```

---

## OAuth Support

| OAuth Type | Secondary Roles | Configuration |
|------------|-----------------|---------------|
| External OAuth (Okta, Entra ID, PingFederate) | **Supported** | Set `EXTERNAL_OAUTH_ANY_ROLE_MODE` |
| Snowflake OAuth | **Not Supported** | N/A |

### External OAuth Configuration
```sql
ALTER SECURITY INTEGRATION my_oauth_integration
SET EXTERNAL_OAUTH_ANY_ROLE_MODE = 'ENABLE';
```

| Value | Behaviour |
|-------|-----------|
| `DISABLE` | Default. Users cannot switch roles |
| `ENABLE` | All users can switch roles |
| `ENABLE_FOR_PRIVILEGE` | Only users/roles with `USE_ANY_ROLE` privilege can switch |

---

## Integration with RBAC Hierarchy

Secondary roles work **with** your role hierarchy, not against it.

### Pattern: READER Roles with Secondary Roles
Each data product has a READER role. Users are granted the READER roles they need, then use secondary roles to access all simultaneously:

```sql
-- User granted multiple READER roles
GRANT ROLE SALES_READER TO USER analyst_jane;
GRANT ROLE MARKETING_READER TO USER analyst_jane;
GRANT ROLE FINANCE_READER TO USER analyst_jane;

-- Enable secondary roles for seamless access
ALTER USER analyst_jane SET DEFAULT_SECONDARY_ROLES = ('ALL');
```

Jane can now query Sales, Marketing, and Finance data in a single session without switching roles.

### Pattern: Domain Consumers
Users outside a domain who need read access bring their own compute and use secondary roles:

```sql
-- Marketing user needs Finance data
GRANT ROLE FINANCE.DB_R TO USER marketing_analyst;

-- Uses their own warehouse + secondary roles
ALTER USER marketing_analyst SET DEFAULT_SECONDARY_ROLES = ('ALL');
```

---

## SQL Templates

### Enable Secondary Roles for All Users (Bulk)
```sql
-- Enable for all users in a role
DECLARE
  c1 CURSOR FOR SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE deleted_on IS NULL;
BEGIN
  FOR record IN c1 DO
    EXECUTE IMMEDIATE 'ALTER USER ' || record.user_name || ' SET DEFAULT_SECONDARY_ROLES = (''ALL'')';
  END FOR;
END;
```

### Check Current Secondary Roles Status
```sql
-- Current session
SELECT CURRENT_SECONDARY_ROLES();

-- User defaults
SHOW PARAMETERS LIKE 'DEFAULT_SECONDARY_ROLES' IN USER my_user;
```

### Audit Secondary Roles Usage
```sql
-- Who has secondary roles enabled
SELECT user_name, default_secondary_roles
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND default_secondary_roles IS NOT NULL;
```

---

## Summary

| Myth | Reality |
|------|---------|
| Security weakness | No - aggregates existing grants only |
| Bypasses access control | No - all roles must be granted first |
| Should be disabled | No - makes access patterns MORE transparent |
| Complicates RBAC | No - simplifies end-user experience |

| Do | Don't |
|----|-------|
| Use for analytics/reporting users | Use for admin roles |
| Set `DEFAULT_SECONDARY_ROLES = ('ALL')` | Expect DDL from secondary roles |
| Grant READER roles liberally | Use for object ownership patterns |
| Rely on role hierarchy | Build custom per-user aggregation roles |


### WAREHOUSE ACCESS ROLES

---
name: warehouse-access-roles
description: "When to use (and not use) warehouse access roles. Challenges common over-application of this pattern."
---

# Warehouse Access Roles

## The Common Misconception

Many RBAC guides recommend creating warehouse access roles (`_WH_<warehouse>`) for every warehouse. **This is almost always unnecessary complexity.**

### Why This Pattern Gets Over-Applied
- Early Snowflake documentation showed access roles as a general pattern
- Consultants apply it universally without considering the actual requirement
- It feels "more secure" to have an intermediary role (it isn't)

### The Simple Rule
**If an access role contains a single grant, it shouldn't exist.**

Access roles exist to bucket multiple permissions together. A 1:1 mapping between access role and warehouse/privilege is just an extra layer of indirection with no benefit.

---

## When You DON'T Need Warehouse Access Roles

### Any Single Warehouse, Single Privilege Scenario

**Don't do this:**
```sql
CREATE ROLE _WH_FINANCE_TRNFRM;
GRANT USAGE ON WAREHOUSE FINANCE_TRNFRM TO ROLE _WH_FINANCE_TRNFRM;
GRANT ROLE _WH_FINANCE_TRNFRM TO ROLE FINANCE_SYSADMIN;
```

**Do this instead:**
```sql
GRANT USAGE ON WAREHOUSE FINANCE_TRNFRM TO ROLE FINANCE_SYSADMIN;
```

The access role adds nothing - it's a 1:1 wrapper around a single grant.

---

## The Cross-Domain Anti-Pattern

A common but **wrong** recommendation is to use warehouse access roles for cross-domain access.

**Example of what NOT to do:**
```
"Marketing needs to query Finance data, so grant them access to Finance's warehouse"
```

**Why this is wrong:**
- Creates an incentive AGAINST supplying useful data products
- Data producers become responsible for consumer compute costs
- Violates separation of concerns

**Correct approach:**
- Domains bring their own compute to data products
- Consumer grants themselves READ access to the data
- Consumer uses their OWN warehouse to query

```
MARKETING_ANALYST
    └── granted FINANCE.DB_R (read access to Finance data)
    └── uses MARKETING_QUERY warehouse (their own compute)
```

This creates the right incentives: data products are judged on data quality, not on providing free compute.

---

## When You DO Need Warehouse Access Roles

Access roles are warranted when **bucketing multiple permissions together**.

These roles must fit into the domain structure:
- **Ownership** transferred to the appropriate RBAC role (Domain or Data Product)
- **Granted** to the READER role at that level (SYSADMIN inherits through the hierarchy)

### Scenario 1: Suite of Specialized Warehouses
A set of Snowpark-optimized or high-compute warehouses available to a subset of users within a domain.

```sql
SET domain = 'DATASCIENCE';
SET domainRbac = $domain || '_RBAC';
SET domainReader = $domain || '_READER';
SET accessRole = $domain || '_WH_SNOWPARK_SUITE';

USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($accessRole);

GRANT USAGE ON WAREHOUSE SNOWPARK_M TO ROLE IDENTIFIER($accessRole);
GRANT USAGE ON WAREHOUSE SNOWPARK_L TO ROLE IDENTIFIER($accessRole);
GRANT USAGE ON WAREHOUSE SNOWPARK_XL TO ROLE IDENTIFIER($accessRole);
GRANT USAGE ON WAREHOUSE SNOWPARK_HCM TO ROLE IDENTIFIER($accessRole);

GRANT OWNERSHIP ON ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($domainRbac);
GRANT ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($domainReader);
```

One access role bundles multiple warehouses, owned by the domain's RBAC role, granted to domain's READER (SYSADMIN inherits).

### Scenario 2: Multiple Privileges on a Warehouse
When users need more than USAGE - e.g., MONITOR and OPERATE for warehouse management.

```sql
SET dataProduct = 'ETL_PLATFORM';
SET dpRbac = $dataProduct || '_RBAC';
SET dpReader = $dataProduct || '_READER';
SET accessRole = $dataProduct || '_WH_OPS';

USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($accessRole);

GRANT USAGE ON WAREHOUSE ETL_LOAD TO ROLE IDENTIFIER($accessRole);
GRANT MONITOR ON WAREHOUSE ETL_LOAD TO ROLE IDENTIFIER($accessRole);
GRANT OPERATE ON WAREHOUSE ETL_LOAD TO ROLE IDENTIFIER($accessRole);

GRANT OWNERSHIP ON ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($dpRbac);
GRANT ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($dpReader);
```

One access role bundles USAGE + MONITOR + OPERATE, owned by data product's RBAC role, granted to READER.

### Scenario 3: Combination - Multiple Warehouses, Multiple Privileges
High-compute cluster with full operational access for a specialized team within a domain.

```sql
SET domain = 'DATASCIENCE';
SET domainRbac = $domain || '_RBAC';
SET domainReader = $domain || '_READER';
SET accessRole = $domain || '_WH_HIGHCOMPUTE_OPS';

USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS IDENTIFIER($accessRole);

GRANT USAGE, MONITOR, OPERATE ON WAREHOUSE COMPUTE_4XL TO ROLE IDENTIFIER($accessRole);
GRANT USAGE, MONITOR, OPERATE ON WAREHOUSE COMPUTE_6XL TO ROLE IDENTIFIER($accessRole);

GRANT OWNERSHIP ON ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($domainRbac);
GRANT ROLE IDENTIFIER($accessRole) TO ROLE IDENTIFIER($domainReader);
```

---

## Decision Tree

```
How many grants would this access role contain?
│
├── ONE grant (single warehouse, single privilege)
│   └── DON'T create access role
│       └── Grant directly to the functional role
│
└── MULTIPLE grants
    ├── Multiple warehouses (suite/cluster)
    │   └── CREATE access role to bundle them
    │
    └── Multiple privileges (USAGE + MONITOR + OPERATE)
        └── CREATE access role to bundle them
```

---

## Warehouse Ownership

Warehouse **ownership** follows the infrastructure principle:

| Architecture | Warehouse Owner |
|--------------|-----------------|
| Single Account | SYSADMIN |
| Per Environment | ENV_SYSADMIN |
| Per Business Unit | BU_SYSADMIN or ENV_SYSADMIN |

Domain and Data Product roles **never own warehouses** - they receive USAGE only.

---

## Anti-Patterns

### 1. One Access Role Per Warehouse
Creating `_WH_<name>` for every warehouse regardless of need.
**Fix:** Only create when bucketing multiple grants.

### 2. Cross-Domain Warehouse Grants
Giving Domain B access to Domain A's warehouse.
**Fix:** Domain B brings their own compute.

### 3. Access Role with Single USAGE Grant
```sql
CREATE ROLE _WH_ANALYTICS;
GRANT USAGE ON WAREHOUSE ANALYTICS TO ROLE _WH_ANALYTICS;  -- Only one grant!
```
**Fix:** Grant USAGE directly to the functional role.

### 4. "Consistency" as Justification
Creating access roles because "we do it for all warehouses."
**Fix:** Consistency in unnecessary complexity is still unnecessary.

---

## Summary

| Situation | Use Access Role? |
|-----------|------------------|
| Single warehouse, USAGE only | **No** - grant directly |
| Cross-domain warehouse access | **No** - bring your own compute |
| Suite of specialized warehouses | **Yes** - bundles multiple warehouses |
| Multiple privileges (USAGE + MONITOR + OPERATE) | **Yes** - bundles multiple privileges |
| "Because best practice says so" | **No** |

---

## Related Notes

- [[Skill - Snowflake RBAC Patterns]]
- [[Workflow - Enterprise RBAC Implementation]]
- [[Cortex Code Skills Master Index]]
