# Snowflake Classification & Tagging for PI-GOVERN

> **Classification answers:** “What kind of data is this?”  
> **Tagging answers:** “What governance label should this object carry?”  
> **Policy answers:** “What should Snowflake do because of that label?”

**PI-GOVERN sits above all three as the governance intelligence and control plane.** Snowflake remains the runtime authority for native enforcement.

---

## Core concepts

Consider a `CUSTOMER` table:

```text
CUSTOMER_ID | CUSTOMER_NAME | EMAIL | PHONE | DOB | ANNUAL_INCOME | CITY
```

### 1. Technical metadata: data type

```text
CUSTOMER_ID    NUMBER
CUSTOMER_NAME  VARCHAR
EMAIL          VARCHAR
PHONE          VARCHAR
DOB            DATE
ANNUAL_INCOME  NUMBER
CITY           VARCHAR
```

Data types are technical metadata. A `VARCHAR` data type does **not** mean `EMAIL = PII`.

### 2. Sensitive-data classification: what the data represents

Snowflake sensitive-data classification identifies semantic and privacy categories, for example:

```text
EMAIL          → semantic category: EMAIL           → privacy category: IDENTIFIER
CUSTOMER_NAME  → semantic category: NAME            → privacy category: IDENTIFIER
PHONE          → semantic category: PHONE_NUMBER    → privacy category: IDENTIFIER
DOB            → semantic category: DATE_OF_BIRTH   → privacy category: SENSITIVE
```

- **Semantic category** describes what the value represents.
- **Privacy category** describes its privacy classification, such as `IDENTIFIER`, `QUASI_IDENTIFIER`, or `SENSITIVE`.

Classification is not merely an inspection of column names. Snowflake analyzes sampled values and returns information such as semantic category, privacy category, confidence, coverage, valid-value ratio, details, and tags.

### 3. Governance tagging: what label the organization assigns

Tags are governance objects that can be applied to databases, schemas, tables, views, columns, and other Snowflake objects.

```text
PII
CONFIDENTIAL
FINANCE_DATA
CUSTOMER_DATA
GDPR
PCI
HR_DATA
```

Examples:

```text
EMAIL          → PII = TRUE
ANNUAL_INCOME  → DATA_SENSITIVITY = HIGH
```

Tags reflect organizational governance decisions; Snowflake cannot infer every company-specific taxonomy.

### 4. Policy and enforcement: what Snowflake should do

```text
PII = TRUE
    ↓
MASKING POLICY
    ↓
Authorized role: raw email
Other roles: masked email
```

Snowflake can associate tags with masking policies, allowing matching tagged columns to receive native protection automatically.

---

## The operating model

```text
DATA
 ↓
CLASSIFICATION — What is this data?
 ↓
TAGGING        — What governance label should it carry?
 ↓
POLICY         — What action is required?
 ↓
ENFORCEMENT    — Snowflake runtime applies it
```

PI-GOVERN should model the distinction strictly. **Classification is not tagging, and neither is policy.**

---

## Who performs classification?

### Snowflake automatic classification

Snowflake can classify sensitive data without requiring a steward to inspect every column. For a column such as `CUSTOMER.EMAIL`, the outcome can be:

```text
SEMANTIC_CATEGORY = EMAIL
PRIVACY_CATEGORY  = IDENTIFIER
CONFIDENCE        = HIGH | MEDIUM | LOW
```

`SYSTEM$CLASSIFY` can classify an individual object and return the classification result. The result reflects sampled data and alignment with the classifier; it is not a human certification of truth.

### Snowflake custom classifiers

Native categories cannot cover every proprietary business concept. Custom classifiers support custom semantic categories, a privacy category, regular expressions, optional column-name matching, and a scoring threshold.

Examples of organization-specific concepts:

```text
EMPLOYEE_ID
POLICY_NUMBER
CLAIM_NUMBER
PATIENT_ID
INTERNAL_RISK_SCORE
CUSTOMER_ID_INTERNAL
```

Example pattern:

```text
EMPLOYEE_IDENTIFIER
Pattern: EMP-[0-9]{5}
Privacy category: IDENTIFIER
```

A column containing mostly values such as `EMP-10293` can then be recommended as `EMPLOYEE_IDENTIFIER`. Snowflake documents a default custom-classifier threshold of `0.8`.

### Manual governance classification

Automation cannot determine all business context. PI-GOVERN should let data stewards add or review governance metadata such as:

```text
Business Domain     Customer
Criticality         High
Regulatory Scope    GDPR
Data Owner          Customer Data Office
Retention Class     7 Years
Certification       Approved
```

Manual governance enriches Snowflake classification; it does not replace it.

---

## Classification does not automatically mean “PII”

Do not collapse classification into a single PII field.

```text
Semantic Classification: EMAIL
Privacy Classification:  IDENTIFIER
Governance Classification: RESTRICTED
Tags: PII = TRUE, GDPR = TRUE
```

These are separate dimensions:

| Dimension | Question answered | Example |
|---|---|---|
| Technical metadata | How is it stored? | `VARCHAR` |
| Semantic classification | What does it represent? | `EMAIL` |
| Privacy classification | How is it categorized for privacy? | `IDENTIFIER` |
| Governance classification | How does the organization govern it? | `RESTRICTED`, `GDPR` |
| Tag | Which executable governance label applies? | `PII = TRUE` |
| Policy | What protection or action follows? | `PII_MASKING_POLICY` |

---

## Two classification systems PI-GOVERN must correlate

### Snowflake data classification

Answers:

> What kind of sensitive information is this?

Examples:

```text
EMAIL
NAME
PHONE_NUMBER
BANK_ACCOUNT
PAYMENT_CARD
```

### Organization governance classification

Answers:

> How does our organization want to govern this data?

Examples:

```text
GOVERNANCE_CLASSIFICATION = PUBLIC | INTERNAL | CONFIDENTIAL | RESTRICTED
DOMAIN = CUSTOMER
CRITICALITY = HIGH
REGULATORY_SCOPE = GDPR
DATA_OWNER = CUSTOMER_DATA_TEAM
RETENTION_CLASS = 7_YEARS
```

PI-GOVERN’s value is correlating both systems:

```text
Column: CUSTOMER.EMAIL

Snowflake classification: EMAIL
Privacy category:          IDENTIFIER
Governance classification: RESTRICTED
Regulatory scope:          GDPR
Business domain:           CUSTOMER
Data owner:                Customer Data Office
```

---

## Classification provenance

PI-GOVERN should persist provenance for every classification and tag assignment.

| Source | Example |
|---|---|
| `SNOWFLAKE_NATIVE` | `EMAIL`, `NAME`, `BANK_ACCOUNT` |
| `SNOWFLAKE_CUSTOM_CLASSIFIER` | `EMPLOYEE_ID`, `POLICY_NUMBER`, `CLAIM_NUMBER` |
| `PI_GOVERN_MANUAL` | `CRITICALITY = HIGH`, `DOMAIN = FINANCE` |
| `INHERITED` | Metadata inherited from a parent object |
| `PROPAGATED` | Metadata propagated through supported dependencies or data movement |

Snowflake tag application methods can include `MANUAL`, `CLASSIFIED`, `INHERITED`, and `PROPAGATED`. The UI must expose this context rather than merely showing `Tag: PII`.

Recommended object view:

```text
Tag:                PII = TRUE
Source:             Snowflake Classification
Applied method:     CLASSIFIED
Classification:     EMAIL
Privacy category:   IDENTIFIER
Last classified:    <timestamp>
Policy:             PII_MASKING_POLICY
Policy status:      ACTIVE
```

---

## Snowflake classification profiles and tag maps

A `CLASSIFICATION_PROFILE` defines how automatic classification operates for associated databases. Depending on configuration, it can control items such as:

```text
minimum object age
maximum classification validity
automatic tagging
view classification
custom classifiers
semantic-category subset
tag mapping
AI mode
```

Conceptual flow:

```text
DATABASE
  ↓
CLASSIFICATION PROFILE
  ↓
TABLES / VIEWS
  ↓
COLUMN SAMPLING
  ↓
CLASSIFICATION RESULTS
```

### System classification tags

Snowflake maintains system tags such as:

```text
SNOWFLAKE.CORE.SEMANTIC_CATEGORY
SNOWFLAKE.CORE.PRIVACY_CATEGORY
```

With automatic tagging enabled, classified columns can receive recommended system classification tags.

### Tag mapping

A classification profile can map semantic categories to organization-defined tags:

```text
EMAIL ────────────┐
NAME ─────────────┼──→ PII = TRUE
PHONE_NUMBER ─────┘
```

This enables the automation chain:

```text
New table
  ↓
Classification profile classifies columns
  ↓
Semantic / privacy categories
  ↓
Tag map applies user-defined governance tags
  ↓
Tag-based policy protects the columns
```

---

## Review, approval, and certification

Detection should not imply approval. PI-GOVERN should layer a governance workflow above Snowflake detection:

```text
DETECTED
  ↓
REVIEW
  ↓
APPROVED
  ↓
GOVERNED
```

Example:

```text
Snowflake detection:    EMAIL
Confidence:             HIGH
Suggested governance:   PII = TRUE; Sensitivity = RESTRICTED
PI-GOVERN status:       Pending Review
Steward decision:       Approved | Rejected | Exception
```

This distinguishes a classifier’s probabilistic result from governance certification.

Snowflake classification can use its native mechanisms and, where configured, AI-assisted classification. Record the actual provenance; PI-GOVERN must not imply every result was generated by AI.

---

## Scope by asset level

The database hierarchy is important, but each level supports a different kind of governance context.

| Level | Best governance use |
|---|---|
| Database | Domain, business unit, environment, regulatory scope, governance owner |
| Schema | Data product, business function, sensitivity boundary, owner |
| Table | Dataset criticality, certification, data owner, retention |
| Column | Sensitive semantic classification, privacy category, PII tag, masking state |

Example:

```text
FINANCE_DB
  Domain: Finance
  Regulatory scope: SOX

FINANCE.CUSTOMER
  Domain: Customer
  Criticality: High

CUSTOMER_MASTER
  Criticality: Critical
  Certification: Certified

EMAIL
  Semantic: EMAIL
  Privacy: IDENTIFIER
  PII: TRUE
  Masking: ACTIVE
```

---

## Examples

### Banking

```text
CUSTOMER_MASTER
├── CUSTOMER_ID
├── NAME
├── EMAIL
├── PHONE
├── PAN
├── ACCOUNT_NUMBER
├── CREDIT_SCORE
└── CUSTOMER_RISK_LEVEL
```

Snowflake may identify supported categories such as `NAME`, `EMAIL`, `PHONE`, PAN, or bank-account information based on the actual data. `CUSTOMER_RISK_LEVEL` and similar internal concepts require a custom classifier or manual governance classification.

### Healthcare

```text
PATIENT
├── PATIENT_ID
├── PATIENT_NAME
├── EMAIL
├── DOB
├── DIAGNOSIS
└── INSURANCE_NUMBER
```

Possible governance mapping:

```text
NAME, EMAIL, DOB → PII = TRUE
DIAGNOSIS        → PHI = TRUE
PHI = TRUE       → Healthcare masking policy
```

Snowflake discovers sensitive categories; the organization determines the governance and enforcement consequence.

### Financial services

```text
CUSTOMER_ACCOUNT
├── ACCOUNT_NUMBER
├── CUSTOMER_NAME
├── PAN
├── BANK_BALANCE
├── CREDIT_SCORE
└── RISK_CATEGORY
```

Native classification can cover supported sensitive categories. `CREDIT_SCORE` and `RISK_CATEGORY` may need a custom classifier or manual business governance because their meaning is organization-specific.

---

## PI-GOVERN architecture

```text
                         PI-GOVERN
                 Governance Intelligence
                           │
          ┌────────────────┼─────────────────┐
          ↓                ↓                 ↓
    Classification       Tags             Policies
          │                │                 │
          └────────────────┼─────────────────┘
                           ↓
                    Governance Context
                           ↓
                    Risk / Compliance
                           ↓
                    Policy Decisions
                           ↓
                   Snowflake Compiler
                           ↓
                   Native Snowflake Enforcement
```

PI-GOVERN should understand relationships across:

```text
classification + tags + owners + lineage + roles + policies + regulations + business context
```

It should not duplicate Snowflake’s classifier. Its differentiated role is business governance, provenance, review, policy intelligence, validated deployment, verification, auditing, and drift management.

---

## End-to-end lifecycle

For a new `CRM.CUSTOMER` table containing `CUSTOMER_ID`, `NAME`, `EMAIL`, `PHONE`, and `ANNUAL_INCOME`:

1. **Discovery** — PI-GOVERN discovers the database, schema, table, and columns.
2. **Classification** — Snowflake detects `NAME`, `EMAIL`, and `PHONE` where the sampled data matches its classifiers.
3. **System tags** — Snowflake records semantic and privacy category tags.
4. **Tag map** — the profile maps qualifying categories to `PII = TRUE`.
5. **Enrichment** — PI-GOVERN adds `Domain = Customer`, `Criticality = High`, `Regulatory Scope = GDPR`, and ownership.
6. **Policy intelligence** — PI-GOVERN determines required protection from the combined governance state.
7. **Policy compilation** — governance intent becomes a validated deployment plan, approval, and Snowflake policy DDL; never arbitrary AI-generated SQL executed directly.
8. **Enforcement** — Snowflake applies the native tag-based masking policy.
9. **Verification** — PI-GOVERN verifies the policy exists, is assigned, protects the intended column, and matches desired state.
10. **Continuous governance** — PI-GOVERN detects new assets, classification/tag/policy/role/lineage changes, and governance drift.

```text
Governance intent
  ↓
Policy model
  ↓
Validation and impact analysis
  ↓
Approval
  ↓
Snowflake policy DDL
  ↓
Native enforcement
```

---

## PI-GOVERN UI model

The Classification & Tagging experience should display three separate sections.

### Detected Classification

```text
Snowflake detected: EMAIL
Privacy category:   IDENTIFIER
Confidence:         HIGH
```

### Governance Classification

```text
Business domain:    Customer
Sensitivity:        Restricted
Criticality:        High
Regulatory scope:   GDPR
```

### Tags and enforcement

```text
PII = TRUE
CUSTOMER_DATA = TRUE
GDPR = TRUE

For each tag:
- Source: Manual | Classified | Inherited | Propagated
- Linked policy: <policy name>
- Status: Active | Unprotected | Drifted
```

---

## Freshness and metadata caveat

Classification and tag metadata can have different freshness characteristics. `ACCOUNT_USAGE` views generally have latency, unlike the Information Schema model. Snowflake documents latency that can be up to two hours for `TAGS` and up to three hours for `TAG_REFERENCES`.

PI-GOVERN must distinguish:

```text
1. Snowflake has not classified or tagged the object yet.
2. Snowflake has classified or tagged it, but PI-GOVERN's mirrored metadata has not refreshed yet.
```

Expose data-source and last-refreshed timestamps so users do not mistake replication latency for an ungoverned asset.

---

## Principle to retain

> **Snowflake can detect what sensitive data appears to be; the organization decides what that classification means for governance; tags carry that governance context; policies turn it into enforcement.**

## Snowflake references

1. [Classifying sensitive data](https://docs.snowflake.com/en/user-guide/classify-intro)
2. [Object tagging](https://docs.snowflake.com/en/user-guide/object-tagging)
3. [Tag-based masking policies](https://docs.snowflake.com/en/user-guide/tag-based-masking-policies)
4. [`SYSTEM$CLASSIFY`](https://docs.snowflake.com/en/sql-reference/functions/system_classify)
5. [Custom classifiers](https://docs.snowflake.com/en/user-guide/classify-custom-classifier)
6. [Tag references and application methods](https://docs.snowflake.com/en/sql-reference/functions/tag_references_all_columns)
7. [Classification profiles](https://docs.snowflake.com/en/user-guide/classify-auto)
8. [Classification tag mapping](https://docs.snowflake.com/en/user-guide/classify-auto#tag-mapping)
9. [AI-assisted classification](https://docs.snowflake.com/en/user-guide/classify-auto#ai-mode)
10. [Account Usage: TAGS](https://docs.snowflake.com/en/sql-reference/account-usage/tags) and [TAG_REFERENCES](https://docs.snowflake.com/en/sql-reference/account-usage/tag_references)
