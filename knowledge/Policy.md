# Snowflake Policy Model for PI-GOVERN

Before we design the **Policy** feature in Pi-Govern, we should first get the Snowflake policy model exactly right. The key is to separate **access control** from **data protection policies**.

---

## 1. The fundamental Snowflake model

Think of Snowflake governance as two layers:

```text
                    SNOWFLAKE GOVERNANCE
                           │
             ┌─────────────┴─────────────┐
             │                           │
       ACCESS CONTROL              DATA PROTECTION
             │                           │
       Who can access?             What can they see/do?
             │                           │
     Roles + Privileges       Policies evaluated at query time
```

### Access control

This answers:

> **"Does this user/role have permission to access this object?"**

For example:

```text
User
 ↓
Role
 ↓
DATABASE USAGE
 ↓
SCHEMA USAGE
 ↓
TABLE SELECT
```

If the user doesn't have the necessary privileges, the query doesn't get access to the object.

### Data protection policies

These answer a different question:

> **"Even if the user can access this object, what data are they allowed to see or how can they use it?"**

Snowflake currently provides several data-protection policy types, including masking, row access, aggregation, projection, and join policies.

This distinction is **extremely important for Pi-Govern**.

---

## 2. The five major data-protection policies

The easiest way to understand them is:

| Policy | Controls | Example |
| :--- | :--- | :--- |
| **Masking Policy** | What value a user sees | Show `XXXX-1234` instead of SSN |
| **Row Access Policy** | Which rows a user sees | India manager sees India rows |
| **Aggregation Policy** | Minimum group size in results | Don't allow statistics for groups < 10 |
| **Projection Policy** | Whether a column can appear in final output | Prevent `SSN` from being projected |
| **Join Policy** | How data can be joined | Restrict joins involving sensitive datasets |

Snowflake documents these as separate policy objects with different enforcement behavior.

---

## 3. Masking Policy

This is the primary column-level policy type implemented in Pi-Govern.

Imagine:

```text
CUSTOMER
────────────────────────────
CUSTOMER_ID
NAME
EMAIL
PHONE
SSN
```

A normal analyst might have:

```sql
SELECT *
FROM CUSTOMER;
```

But the policy could produce:

```text
CUSTOMER_ID     NAME       EMAIL             SSN
---------------------------------------------------------
101             John       john@abc.com      *********
102             Sarah      sarah@abc.com     *********
```

While an authorized role could see:

```text
CUSTOMER_ID     NAME       EMAIL             SSN
---------------------------------------------------------
101             John       john@abc.com      123-45-6789
102             Sarah      sarah@abc.com     987-65-4321
```

The important thing is:

**The user can have SELECT on the table but still receive masked values.**

Snowflake evaluates the masking policy at query runtime.

Conceptually:

```text
User
  │
  │ SELECT
  ▼
TABLE
  │
  ├── Access check
  │
  ▼
MASKING POLICY
  │
  ├── Authorized → Original value
  │
  └── Unauthorized → Masked value
```

This is very different from simply denying access to the column.

---

## 4. Row Access Policy

Now imagine the table contains:

```text
EMPLOYEE
────────────────────────────
EMPLOYEE_ID
NAME
REGION
SALARY
```

Suppose:

```text
Manager A → India
Manager B → USA
```

Both managers may have:

```text
SELECT on EMPLOYEE
```

But they shouldn't see the same rows.

The row access policy evaluates:

```text
Current user/role
        ↓
Determine permitted region
        ↓
Filter rows
```

So Manager A effectively receives:

```text
EMPLOYEE_ID | NAME  | REGION
-----------------------------
1           | Rahul | India
2           | Amit  | India
```

while Manager B receives:

```text
EMPLOYEE_ID | NAME  | REGION
-----------------------------
7           | John  | USA
8           | Mike  | USA
```

Snowflake describes row access policies as row-level security that determines which rows are returned at query time.

---

## 5. Masking vs Row Access

This distinction is critical in the Pi-Govern UI.

### Masking

**Column-level protection**

```text
                 CUSTOMER
                     │
        ┌────────────┼────────────┐
        │            │            │
       NAME        EMAIL         SSN
                                  │
                           MASKING POLICY
                                  │
                         ┌────────┴────────┐
                         │                 │
                     Authorized       Unauthorized
                         │                 │
                    123-45-6789       *********
```

### Row access

**Row-level protection**

```text
                    CUSTOMER
                       │
                  ROW ACCESS
                    POLICY
                       │
             ┌─────────┴─────────┐
             │                   │
          User A               User B
             │                   │
          India rows          USA rows
```

They can also work together. Snowflake evaluates a row access policy before masking policies when both apply.

---

## 6. Projection Policy

Projection is different again.

Suppose:

```text
CUSTOMER

ID
NAME
EMAIL
PHONE
SSN
```

A projection policy can prevent a column from being projected into the **final query result**.

For example:

```sql
SELECT
    ID,
    NAME,
    SSN
FROM CUSTOMER;
```

If `SSN` is projection-constrained and the current role isn't permitted to project it, Snowflake prevents that column from appearing in the final output.

This is different from masking:

```text
MASKING
SSN → ********

PROJECTION
SSN → cannot be projected
```

Projection-constrained columns can still be used in inner queries or `WHERE` clauses, so projection policies should not be treated as equivalent to complete column secrecy.

---

## 7. Aggregation Policy

This addresses a differential privacy problem.

Imagine a table:

```text
EMPLOYEE
```

and somebody queries:

```sql
SELECT
    department,
    AVG(salary)
FROM employee
GROUP BY department;
```

If a department has only one employee, the aggregate could effectively reveal that person's salary.

An aggregation policy can require a minimum group size.

Conceptually:

```text
Minimum group size = 10

GROUP BY department

Department A → 1 employee   ❌
Department B → 8 employees  ❌
Department C → 43 employees ✅
```

Snowflake describes aggregation policies as enforcing minimum group sizes to reduce the possibility of isolating an individual's data.

---

## 8. Join Policy

A join policy controls whether a protected table/view must be joined and, where configured, which columns may be used for joins.

Conceptually:

```text
CUSTOMER
   │
   │ permitted relationship
   ▼
TRANSACTIONS
```

This prevents arbitrary correlation and triangulation of sensitive datasets.

---

## 9. Tag-Based Policies

Snowflake supports **tag-based policies**, which associate a policy with a tag and then apply that tag to objects.

For example:

```text
Tag:
PII

Value:
SENSITIVE
```

Then:

```text
PII Tag
   │
   ▼
Masking Policy
   │
   ▼
Objects carrying the tag
```

Workflow:

```text
CREATE TAG
     ↓
CREATE POLICY
     ↓
SET POLICY ON TAG
     ↓
SET TAG ON OBJECT
     ↓
Policy automatically protects applicable objects
```

This is where **Classification + Tagging + Policy** in Pi-Govern becomes one coherent governance workflow.

---

## 10. Example for Pi-Govern

Imagine your platform discovers:

```text
DATABASE
└── CUSTOMER_DB
    └── SALES
        └── CUSTOMER
            ├── CUSTOMER_ID
            ├── NAME
            ├── EMAIL
            ├── PHONE
            └── SSN
```

Your classification engine identifies:

```text
EMAIL → PII
PHONE → PII
SSN → HIGHLY_SENSITIVE
```

Then your tagging system produces:

```text
EMAIL
  └── tag: PII

PHONE
  └── tag: PII

SSN
  └── tag: HIGHLY_SENSITIVE
```

Then your policy layer maps:

```text
PII
 │
 └── Masking Policy

HIGHLY_SENSITIVE
 │
 ├── Masking Policy
 └── Projection Policy
```

Now the overall architecture becomes:

```text
                PI-GOVERN
                    │
        ┌───────────┴───────────┐
        │                       │
 Classification              Tagging
        │                       │
        └───────────┬───────────┘
                    │
                    ▼
                 POLICY
                    │
       ┌────────────┼─────────────┐
       │            │             │
    Masking     Row Access    Projection
       │            │             │
       └────────────┼─────────────┘
                    │
                    ▼
                SNOWFLAKE
                    │
                    ▼
             Query-time enforcement
```

This elevates the model from a basic CRUD screen into an active governance control plane.

---

## 11. Direct policy vs tag-based policy

There are two primary ways policies attach to objects:

### Direct

```text
TABLE
 │
 └── MASKING POLICY
```

or:

```text
TABLE
 │
 └── ROW ACCESS POLICY
```

### Tag-based

```text
TABLE
 │
 └── TAG
      │
      └── POLICY
```

The tag-based model is ideal for centralized governance because you define the protection once and apply the tag to objects. Snowflake also supports inheritance when tags are applied higher in the object hierarchy (database or schema).

---

## 12. Policy Intelligence UI Model

Rather than a simple flat table of policy records, the Pi-Govern Policy module should answer:

```text
POLICY INTELLIGENCE

What policies exist?
        ↓
Where are they applied?
        ↓
What data do they protect?
        ↓
Which tags/classifications trigger them?
        ↓
Which roles/users are affected?
        ↓
What happens at query time?
        ↓
Is the policy actually enforced?
```

Underlying lifecycle:

```text
Classification
      ↓
Tag
      ↓
Policy
      ↓
Object / Column
      ↓
Role / Context
      ↓
Query
      ↓
Policy Evaluation
      ↓
Result
```

### Version & Capability Notes

Snowflake tag-based masking policies are generally available on Enterprise Edition. Tag-based aggregation, row access, projection, and join policies were introduced as public-preview capabilities in 2026.
