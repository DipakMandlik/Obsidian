---
title: DBT
type: concept
created: 2026-09-01
updated: 2026-09-01
tags: [data-engineering, dbt]
---

# DBT

## DBT Test

Testing in dbt is a core engineering practice designed to bring software rigor to data pipelines. At a high level, all dbt tests compiled by dbt are SQL queries where any returned rows indicate a failure. If the query returns zero rows, the test passes.

In dbt, tests are broadly divided into Data Testing (validating the quality of your production data) and Unit Testing (validating your SQL transformation logic using mock inputs). Within data testing, you can implement Generic Tests, Singular Tests, and Source Freshness Tests.

### 1. Generic Tests (Schema Tests)

Generic tests are parameterised, reusable test templates defined in YAML configuration files. Once defined, you can apply them to columns across models, sources, seeds or snapshots.

dbt ships with four built-in generic tests out of the box:
- **unique**: Asserts that every value in a column is distinct.
- **not_null**: Asserts that a column contains no missing or null values.
- **accepted_values**: Asserts that a column's values fall within a predefined list.
- **relationships**: Asserts referential integrity by validating that values in a column exist in a target field of a parent model.

`models/staging/jaffle_shop/schema.yml`:
```yaml
version: 2

models:
  - name: stg_orders
    description: "A staging model cleaning up order data"
    columns:
      - name: order_id
        description: "Primary key of the model"
        data_tests:
          - unique
          - not_null
      - name: status
        description: "The current fulfillment status of the order"
        data_tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']
      - name: customer_id
        description: "Foreign key mapping back to the customer"
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customers')
                field: customer_id
```

**⚠️ Edge cases & pitfalls**
- **The null-value loophole in referential integrity**: by default, if a foreign key column contains `NULL`, the `relationships` test still passes for those rows. If referential integrity must be strict, pair `relationships` with an explicit `not_null` test on that column.
- **Strict case sensitivity**: `accepted_values` matches strictly per the database engine's case rules. If Snowflake loads `'Placed'` but the YAML specifies `'placed'`, the test compiles, runs, and fails on the capitalized rows.

### 2. Singular Tests (Custom Data Tests)

Singular tests are one-off SQL queries written to validate complex, domain-specific business rules. They are saved as `.sql` files directly inside the project's `tests/` directory. They require no registration in schema YAML — dbt automatically compiles and runs any SQL file in `tests/` during `dbt test`.

`tests/assert_positive_value_for_total_amount.sql`:
```sql
-- Refunds may have a negative amount, but the total order amount should always be >= 0.
-- Return records where this rule is violated to fail the test.
select
    order_id,
    sum(amount) as total_amount
from {{ ref('stg_payments') }}
group by 1
having not(total_amount >= 0)
```

**⚠️ Edge cases & pitfalls**
- **Poor scaling performance**: singular tests allow raw SQL joins, but complex multi-table joins on massive production tables run on every deployment, slowing model runtimes and inflating compute costs. Keep the testing footprint lean.
- **Hardcoded fragility**: singular tests reference specific tables/columns directly, so they don't scale. Copy-pasted logic across tables is a signal to refactor into a reusable Custom Generic Test via a Jinja macro.

### 3. Timeliness Tests (Source Freshness)

Unlike standard model assertions, Source Freshness tests evaluate metadata to ensure raw datasets are loaded into the warehouse within a target SLA.

`models/sources.yml`:
```yaml
version: 2

sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _etl_loaded_at
    tables:
      - name: orders
```

**⚠️ Edge cases & pitfalls**
- **Timezone discrepancies**: freshness relies on the database's internal clock or timestamp casting. Mismatched extraction vs. database timezones (UTC vs. local) can cause erroneous flags — always standardize timestamps to UTC.

### 4. Unit Testing

Unit tests evaluate the transformation logic itself, not the underlying production data. A mock ("toy") input dataset is run through the model and the output is compared against a static, expected dataset to verify the SQL logic before pushing to production.

Native unit testing is supported in modern dbt versions; teams on older projects often rely on open-source packages like `dbt-unit-testing` or `dbt-datamocktool` to run TDD (Test-Driven Development) workflows.

**⚠️ Edge cases & pitfalls**
- **Masking latent SQL bugs**: unit tests excel at verifying calculations (e.g., `amount / 100`) against controlled mock data, but they cannot catch runtime errors on production data — incompatible datatypes, sorting errors, duplicate record volumes. They must complement, not replace, standard data quality tests.

## Related
