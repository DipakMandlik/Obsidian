# DDL Script

Creates the dedicated `PIQLENS_DQ_DB` database, `PIQLENS_DQ` schema, core DQ platform tables, and custom reporting tables & views.

## Part 1: Core Platform Data Quality Tables

```sql
CREATE DATABASE IF NOT EXISTS PIQLENS_DQ_DB;
USE DATABASE PIQLENS_DQ_DB;
CREATE SCHEMA IF NOT EXISTS PIQLENS_DQ;
USE SCHEMA PIQLENS_DQ_DB.PIQLENS_DQ;

-- 1. Scan Run Lifecycle
CREATE TABLE IF NOT EXISTS DQ_RUN_CONTROL (
    RUN_ID                        VARCHAR(100) PRIMARY KEY,
    EXECUTION_MODE                VARCHAR(20),       -- 'MANUAL', 'SCHEDULED', 'API'
    TRIGGER_TYPE                  VARCHAR(20),       -- 'USER', 'CRON', 'WEBHOOK'
    TRIGGERED_BY                  VARCHAR(100),
    START_TS                      TIMESTAMP_NTZ,
    END_TS                        TIMESTAMP_NTZ,
    DURATION_SECONDS              NUMBER(10, 2),
    RUN_STATUS                    VARCHAR(50),       -- 'RUNNING', 'COMPLETED', 'FAILED'
    TOTAL_DATASETS                NUMBER(38, 0) DEFAULT 0,
    TOTAL_CHECKS                  NUMBER(38, 0) DEFAULT 0,
    PASSED_CHECKS                 NUMBER(38, 0) DEFAULT 0,
    FAILED_CHECKS                 NUMBER(38, 0) DEFAULT 0,
    WARNING_CHECKS                NUMBER(38, 0) DEFAULT 0,
    SKIPPED_CHECKS                NUMBER(38, 0) DEFAULT 0,
    ERROR_MESSAGE                 VARCHAR(4000),
    CREATED_TS                    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- 2. Rule Evaluation Results (Per-Rule Metrics)
CREATE TABLE IF NOT EXISTS DQ_CHECK_RESULTS (
    CHECK_ID              NUMBER AUTOINCREMENT PRIMARY KEY,
    RUN_ID                VARCHAR(100) NOT NULL REFERENCES DQ_RUN_CONTROL(RUN_ID),
    CHECK_TIMESTAMP       TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    DATABASE_NAME         VARCHAR(100) NOT NULL,
    SCHEMA_NAME           VARCHAR(100) NOT NULL,
    TABLE_NAME            VARCHAR(100) NOT NULL,
    COLUMN_NAME           VARCHAR(100),
    RULE_NAME             VARCHAR(100) NOT NULL,
    RULE_TYPE             VARCHAR(50),
    RULE_LEVEL            VARCHAR(20),               -- 'TABLE', 'COLUMN'
    METRIC_DATE           DATE,                      -- Partition date from dateColumn
    CURRENT_VALUE         FLOAT,
    PREVIOUS_VALUE        FLOAT,
    CHANGE_PCT            FLOAT,
    TOTAL_RECORDS         NUMBER(38, 0),
    VALID_RECORDS         NUMBER(38, 0),
    INVALID_RECORDS       NUMBER(38, 0),
    PASS_RATE             NUMBER(10, 2),
    THRESHOLD             NUMBER(10, 2),
    CHECK_STATUS          VARCHAR(50) NOT NULL,      -- 'PASSED', 'FAILED', 'WARNING', 'SKIPPED'
    FAILURE_REASON        VARCHAR(500),
    EXECUTION_TIME_MS     NUMBER(38, 0),
    REPORT_TAG            VARCHAR(200),
    REPORT_RULE_TYPE      VARCHAR(20),
    CREATED_TS            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- 3. In-Warehouse Failed Records Forensics
CREATE TABLE IF NOT EXISTS DQ_FAILED_RECORDS (
    FAILURE_ID            NUMBER AUTOINCREMENT PRIMARY KEY,
    CHECK_ID              NUMBER REFERENCES DQ_CHECK_RESULTS(CHECK_ID),
    RUN_ID                VARCHAR(100) NOT NULL,
    DATABASE_NAME         VARCHAR(100) NOT NULL,
    SCHEMA_NAME           VARCHAR(100) NOT NULL,
    TABLE_NAME            VARCHAR(100) NOT NULL,
    COLUMN_NAME           VARCHAR(100),
    RULE_NAME             VARCHAR(100) NOT NULL,
    ROW_ID_TYPE           VARCHAR(20) NOT NULL DEFAULT 'ROW_HASH', -- PRIMARY_KEY or ROW_HASH
    PRIMARY_KEY_COLUMN    VARCHAR(500),
    FAILED_ROW_PK         VARCHAR(500),              -- Source PK, or ROW_<SHA256(row JSON)> when no PK exists
    FAILED_VALUE          VARCHAR(4000),             -- Violating cell value
    FAILURE_REASON        VARCHAR(500),              -- Diagnostic reason
    ROW_DATA_JSON         VARIANT,                   -- Full source row snapshot; duplicates use FAILURE_ID
    DETECTED_TS            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
-- Optimize cluster key for sub-second pushdown pagination
ALTER TABLE DQ_FAILED_RECORDS CLUSTER BY (CHECK_ID, DETECTED_TS);
-- Upgrade existing installations created before row-identity metadata existed.
ALTER TABLE DQ_FAILED_RECORDS ADD COLUMN IF NOT EXISTS ROW_ID_TYPE VARCHAR(20) DEFAULT 'ROW_HASH';
ALTER TABLE DQ_FAILED_RECORDS ADD COLUMN IF NOT EXISTS PRIMARY_KEY_COLUMN VARCHAR(500);

-- 4. Daily Quality Summary (6 Dimensions & SLA Compliance)
CREATE TABLE IF NOT EXISTS DQ_DAILY_SUMMARY (
    SUMMARY_ID            NUMBER AUTOINCREMENT PRIMARY KEY,
    SUMMARY_DATE          DATE NOT NULL,
    DATABASE_NAME         VARCHAR(100) NOT NULL,
    SCHEMA_NAME           VARCHAR(100) NOT NULL,
    TABLE_NAME            VARCHAR(100) NOT NULL,
    BUSINESS_DOMAIN       VARCHAR(100),
    TOTAL_CHECKS          NUMBER(38, 0) DEFAULT 0,
    PASSED_CHECKS         NUMBER(38, 0) DEFAULT 0,
    FAILED_CHECKS         NUMBER(38, 0) DEFAULT 0,
    WARNING_CHECKS        NUMBER(38, 0) DEFAULT 0,
    SKIPPED_CHECKS        NUMBER(38, 0) DEFAULT 0,
    DQ_SCORE              NUMBER(10, 2),             -- Composite Score (0 - 100)
    COMPLETENESS_SCORE    NUMBER(10, 2),
    UNIQUENESS_SCORE      NUMBER(10, 2),
    VALIDITY_SCORE        NUMBER(10, 2),
    CONSISTENCY_SCORE     NUMBER(10, 2),
    FRESHNESS_SCORE       NUMBER(10, 2),
    VOLUME_SCORE          NUMBER(10, 2),
    TRUST_LEVEL           VARCHAR(20),               -- 'HIGH', 'MEDIUM', 'LOW'
    QUALITY_GRADE         VARCHAR(10),               -- 'A+', 'A', 'B', 'C', 'F'
    IS_SLA_MET            BOOLEAN DEFAULT TRUE,
    TOTAL_RECORDS         NUMBER(38, 0),
    FAILED_RECORDS_COUNT  NUMBER(38, 0),
    LAST_RUN_ID           VARCHAR(100),
    CREATED_TS            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- 5. Weekly Aggregated Summary
CREATE TABLE IF NOT EXISTS DQ_WEEKLY_SUMMARY (
    ID                    NUMBER AUTOINCREMENT PRIMARY KEY,
    WEEK_ID               VARCHAR(10),               -- e.g. '2026-W35'
    WEEK_START_DATE       DATE,
    WEEK_END_DATE         DATE,
    DATABASE_NAME         VARCHAR(100),
    SCHEMA_NAME           VARCHAR(100),
    TABLE_NAME            VARCHAR(100),
    AVG_DQ_SCORE          NUMBER(10, 2),
    AVG_COMPLETENESS      NUMBER(10, 2),
    AVG_UNIQUENESS        NUMBER(10, 2),
    AVG_VALIDITY          NUMBER(10, 2),
    AVG_CONSISTENCY       NUMBER(10, 2),
    AVG_FRESHNESS         NUMBER(10, 2),
    SCORE_TREND           VARCHAR(20),
    QUALITY_GRADE         VARCHAR(10),
    CREATED_TS            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- 6. Column Profiling Metrics
CREATE TABLE IF NOT EXISTS DQ_COLUMN_PROFILE (
    PROFILE_ID            NUMBER AUTOINCREMENT PRIMARY KEY,
    RUN_ID                VARCHAR(100),
    DATABASE_NAME         VARCHAR(100),
    SCHEMA_NAME           VARCHAR(100),
    TABLE_NAME            VARCHAR(100),
    COLUMN_NAME           VARCHAR(100),
    DATA_TYPE             VARCHAR(50),
    TOTAL_RECORDS         NUMBER(38, 0),
    NULL_COUNT            NUMBER(38, 0),
    DISTINCT_COUNT        NUMBER(38, 0),
    MIN_VALUE             VARCHAR(16777216),
    MAX_VALUE             VARCHAR(16777216),
    AVG_VALUE             NUMBER(38, 4),
    STDDEV_VALUE          NUMBER(38, 4),
    PROFILE_TS            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

## Part 2: Custom Client Reporting Pipeline

```sql
-- 7. Custom Rule Results Table (For business reporting)
CREATE TABLE IF NOT EXISTS PIQLENS_RULE_RESULTS (
    METRIC_DATE           DATE,
    TAG                   VARCHAR(200),
    BASE_TAG              VARCHAR(200),
    RULE_TYPE             VARCHAR(20),
    CURRENT_VALUE         FLOAT,
    PREVIOUS_VALUE        FLOAT,
    CHANGE_PCT            FLOAT,
    VALIDATION_STATUS     VARCHAR(20),
    SCANNED_AT            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
-- Upgrade installations created before SCANNED_AT became the canonical
-- report-result timestamp. Existing CREATED_AT data is left intact.
ALTER TABLE PIQLENS_RULE_RESULTS
    ADD COLUMN IF NOT EXISTS SCANNED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP();

-- 8. Customer Report Config Table
CREATE TABLE IF NOT EXISTS PIQLENS_REPORT_CONFIG (
    BASE_TAG              VARCHAR(200) NOT NULL PRIMARY KEY,
    SECTION_NAME          VARCHAR(200) NOT NULL,
    PROGRAM_NAME          VARCHAR(100) NOT NULL,
    APPLICATION_NAME      VARCHAR(200) NOT NULL,
    DISPLAY_ORDER         INT NOT NULL,
    IS_EXCLUDED           BOOLEAN NOT NULL DEFAULT FALSE,
    EXCLUDE_REASON        VARCHAR(500),
    FORMAT_TYPE           VARCHAR(20) NOT NULL DEFAULT 'CURRENCY'
);

-- 9. Custom Client Master Report View
CREATE OR REPLACE VIEW PIQLENS_REPORT_VIEW AS
WITH latest_per_tag AS (
  SELECT BASE_TAG, MAX(METRIC_DATE) AS LATEST_DATE
  FROM PIQLENS_RULE_RESULTS
  GROUP BY BASE_TAG
),
merged AS (
  SELECT
    r.BASE_TAG,
    MAX(r.METRIC_DATE)                                                        AS METRIC_DATE,
    MAX(CASE WHEN r.RULE_TYPE = 'MTD'    THEN r.CURRENT_VALUE  END)           AS MTD_CURRENT,
    MAX(CASE WHEN r.RULE_TYPE = 'MTD'    THEN r.PREVIOUS_VALUE END)           AS MTD_PREVIOUS,
    MAX(CASE WHEN r.RULE_TYPE = 'MTD'    THEN r.CHANGE_PCT     END)           AS MTD_PCT,
    MAX(CASE WHEN r.RULE_TYPE = 'DAILY'  THEN r.CURRENT_VALUE  END)           AS DAILY_CURRENT,
    MAX(CASE WHEN r.RULE_TYPE = 'DAILY'  THEN r.PREVIOUS_VALUE END)           AS DAILY_PREVIOUS,
    MAX(CASE WHEN r.RULE_TYPE = 'DAILY'  THEN r.CHANGE_PCT     END)           AS DAILY_PCT,
    MAX(CASE WHEN r.RULE_TYPE = 'CHANGE' THEN r.CURRENT_VALUE  END)           AS CHG_CURRENT,
    MAX(CASE WHEN r.RULE_TYPE = 'CHANGE' THEN r.PREVIOUS_VALUE END)           AS CHG_PREVIOUS,
    MAX(CASE WHEN r.RULE_TYPE = 'CHANGE' THEN r.CHANGE_PCT     END)           AS CHG_PCT,
    CASE
      WHEN BOOLOR_AGG(r.VALIDATION_STATUS = 'FAILED')   THEN 'FAILED'
      WHEN BOOLOR_AGG(r.VALIDATION_STATUS = 'EXCLUDED') THEN 'EXCLUDED'
      ELSE 'PASSED'
    END AS VALIDATION_STATUS,
    CASE
      WHEN MAX(CASE WHEN r.RULE_TYPE='DAILY' THEN r.CHANGE_PCT END) IS NOT NULL
       AND MAX(CASE WHEN r.RULE_TYPE='MTD'   THEN r.CHANGE_PCT END) IS NOT NULL
      THEN
        'Daily: '   || TO_CHAR(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CURRENT_VALUE END),'999,999,990.00') ||
        ' ('  || IFF(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CHANGE_PCT END) >= 0, '+', '') ||
        TO_CHAR(ROUND(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CHANGE_PCT END), 1)) || '%) | ' ||
        'MTD: '     || TO_CHAR(MAX(CASE WHEN RULE_TYPE='MTD'   THEN CURRENT_VALUE END),'999,999,990.00') ||
        ' ('  || IFF(MAX(CASE WHEN RULE_TYPE='MTD'   THEN CHANGE_PCT END) >= 0, '+', '') ||
        TO_CHAR(ROUND(MAX(CASE WHEN RULE_TYPE='MTD'   THEN CHANGE_PCT END), 1)) || '%)'
      WHEN MAX(CASE WHEN r.RULE_TYPE='MTD' THEN r.CHANGE_PCT END) IS NOT NULL
      THEN
        'MTD: ' || TO_CHAR(MAX(CASE WHEN RULE_TYPE='MTD' THEN CURRENT_VALUE END),'999,999,990.00') ||
        ' (' || IFF(MAX(CASE WHEN RULE_TYPE='MTD' THEN CHANGE_PCT END) >= 0, '+', '') ||
        TO_CHAR(ROUND(MAX(CASE WHEN RULE_TYPE='MTD' THEN CHANGE_PCT END), 1)) || '%)'
      WHEN MAX(CASE WHEN r.RULE_TYPE='DAILY' THEN r.CHANGE_PCT END) IS NOT NULL
      THEN
        'Daily: ' || TO_CHAR(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CURRENT_VALUE END),'999,999,990.00') ||
        ' (' || IFF(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CHANGE_PCT END) >= 0, '+', '') ||
        TO_CHAR(ROUND(MAX(CASE WHEN RULE_TYPE='DAILY' THEN CHANGE_PCT END), 1)) || '%)'
      WHEN MAX(CASE WHEN r.RULE_TYPE='CHANGE' THEN r.CHANGE_PCT END) IS NOT NULL
      THEN
        'Change: ' || IFF(MAX(CASE WHEN RULE_TYPE='CHANGE' THEN CHANGE_PCT END) >= 0, '+', '') ||
        TO_CHAR(ROUND(MAX(CASE WHEN RULE_TYPE='CHANGE' THEN CHANGE_PCT END), 1)) || '%'
      ELSE NULL
    END AS COMMENT_TEXT
  FROM PIQLENS_RULE_RESULTS r
  JOIN latest_per_tag l ON r.BASE_TAG = l.BASE_TAG AND r.METRIC_DATE = l.LATEST_DATE
  GROUP BY r.BASE_TAG
)
SELECT
  c.SECTION_NAME,
  c.PROGRAM_NAME,
  c.APPLICATION_NAME,
  m.METRIC_DATE,
  IFF(c.IS_EXCLUDED, 'EXCLUDED',        COALESCE(m.VALIDATION_STATUS, 'MISSING')) AS FINAL_STATUS,
  IFF(c.IS_EXCLUDED, NULL,              COALESCE(m.MTD_CURRENT, m.CHG_CURRENT, m.DAILY_CURRENT)) AS CURRENT_VALUE,
  IFF(c.IS_EXCLUDED, NULL,              COALESCE(m.MTD_PREVIOUS, m.CHG_PREVIOUS, m.DAILY_PREVIOUS)) AS PREVIOUS_VALUE,
  IFF(c.IS_EXCLUDED, c.EXCLUDE_REASON,  m.COMMENT_TEXT) AS COMMENT,
  c.FORMAT_TYPE,
  c.DISPLAY_ORDER,
  c.IS_EXCLUDED
FROM PIQLENS_REPORT_CONFIG c
LEFT JOIN merged m ON c.BASE_TAG = m.BASE_TAG
ORDER BY c.DISPLAY_ORDER;
```

## Part 3: PostgreSQL - Metadata, Config & RBAC Layer

Per the hybrid model below: Postgres holds **config/metadata/RBAC** only. It does **not** re-store bulk scan output. `dq_check_results`, `dq_failed_records`, `dq_daily_summary`, `dq_weekly_summary`, `dq_monthly_summary`, `dq_quarterly_summary`, and `dq_column_profile` were dropped from the original draft because they duplicate the Snowflake tables in Part 1 (same grain, same purpose) — keeping both copies means double-writing on every scan and two sources of truth to reconcile, exactly what the hybrid split is meant to avoid. `dq_run_control` is the one intentional exception: kept here as the **live-status cache** (polled by the UI on every run) and mirrored to its Snowflake counterpart at run completion for historical audit — see the sync note under Deployment Architecture.

```sql
-- CreateSchema
CREATE SCHEMA IF NOT EXISTS "public";

-- ── Rule & Dataset Configuration ──────────────────────────────────
CREATE TABLE "rule_master" (
    "rule_id" SERIAL NOT NULL,
    "rule_name" VARCHAR(100) NOT NULL,
    "rule_type" VARCHAR(50) NOT NULL,
    "rule_level" VARCHAR(20) NOT NULL,
    "display_name" VARCHAR(120),
    "ui_category" VARCHAR(50),
    "subcategory" VARCHAR(80),
    "default_threshold" DOUBLE PRECISION NOT NULL DEFAULT 90.0,
    "description" VARCHAR(500),
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "is_custom" BOOLEAN NOT NULL DEFAULT false,
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_by" VARCHAR(100),
    "modified_ts" TIMESTAMP(3),
    CONSTRAINT "rule_master_pkey" PRIMARY KEY ("rule_id")
);

CREATE TABLE "rule_sql_template" (
    "template_id" SERIAL NOT NULL,
    "rule_id" INTEGER NOT NULL,
    "sql_template" TEXT NOT NULL,
    "template_version" INTEGER NOT NULL DEFAULT 1,
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_by" VARCHAR(100),
    "modified_ts" TIMESTAMP(3),
    CONSTRAINT "rule_sql_template_pkey" PRIMARY KEY ("template_id")
);

CREATE TABLE "dataset_config" (
    "dataset_id" VARCHAR(100) NOT NULL,
    "source_database" VARCHAR(100) NOT NULL,
    "source_schema" VARCHAR(100) NOT NULL,
    "source_table" VARCHAR(100) NOT NULL,
    "business_domain" VARCHAR(100),
    "criticality" VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_by" VARCHAR(100),
    "modified_ts" TIMESTAMP(3),
    "incremental_enabled" BOOLEAN NOT NULL DEFAULT true,
    "incremental_column" VARCHAR(100),
    "last_incremental_ts" TIMESTAMP(3),
    "primary_key_column" TEXT,
    "connector_type" VARCHAR(20) NOT NULL DEFAULT 'snowflake',
    "connection_id" VARCHAR(100),
    "data_connection_id" VARCHAR(100),
    "source_fingerprint" VARCHAR(64),
    CONSTRAINT "dataset_config_pkey" PRIMARY KEY ("dataset_id")
);

CREATE TABLE "dataset_columns" (
    "id" VARCHAR(100) NOT NULL,
    "dataset_id" VARCHAR(100) NOT NULL,
    "column_name" VARCHAR(200) NOT NULL,
    "data_type" VARCHAR(200) NOT NULL,
    "ordinal_position" INTEGER,
    "is_nullable" BOOLEAN,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_ts" TIMESTAMP(3),
    CONSTRAINT "dataset_columns_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "dataset_schema_snapshot" (
    "id" VARCHAR(100) NOT NULL,
    "dataset_id" VARCHAR(100) NOT NULL,
    "column_name" VARCHAR(200) NOT NULL,
    "data_type" VARCHAR(200) NOT NULL,
    "ordinal_position" INTEGER,
    "is_nullable" BOOLEAN,
    "baseline_version" INTEGER NOT NULL DEFAULT 1,
    "is_active_baseline" BOOLEAN NOT NULL DEFAULT true,
    "captured_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "captured_by_run_id" VARCHAR(100),
    CONSTRAINT "dataset_schema_snapshot_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "dataset_rule_config" (
    "config_id" SERIAL NOT NULL,
    "dataset_id" VARCHAR(100) NOT NULL,
    "rule_id" INTEGER NOT NULL,
    "column_name" VARCHAR(100),
    "incremental_column" VARCHAR(100),
    "threshold_value" DECIMAL(10,2) NOT NULL DEFAULT 90.00,
    "config_json" JSONB,
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_by" VARCHAR(100),
    "modified_ts" TIMESTAMP(3),
    CONSTRAINT "dataset_rule_config_pkey" PRIMARY KEY ("config_id")
);

CREATE TABLE "weights_mapping" (
    "weight_id" SERIAL NOT NULL,
    "rule_type" VARCHAR(50) NOT NULL,
    "business_domain" VARCHAR(100),
    "weight" DECIMAL(5,2) NOT NULL DEFAULT 1.00,
    "priority" INTEGER NOT NULL DEFAULT 3,
    "effective_date" DATE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "expiry_date" DATE,
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "weights_mapping_pkey" PRIMARY KEY ("weight_id")
);

CREATE TABLE "allowed_values_config" (
    "id" SERIAL NOT NULL,
    "dataset_id" VARCHAR(100),
    "column_name" VARCHAR(100),
    "allowed_values" VARCHAR(2000),
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_ts" TIMESTAMP(3),
    CONSTRAINT "allowed_values_config_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "fk_config" (
    "id" SERIAL NOT NULL,
    "dataset_id" VARCHAR(100),
    "column_name" VARCHAR(100),
    "parent_database" VARCHAR(100),
    "parent_schema" VARCHAR(100),
    "parent_table" VARCHAR(100),
    "parent_key" VARCHAR(100),
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_ts" TIMESTAMP(3),
    CONSTRAINT "fk_config_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "schedule_config" (
    "schedule_id" VARCHAR(100) NOT NULL,
    "dataset_id" VARCHAR(100),
    "cron_expression" VARCHAR(100) NOT NULL,
    "scan_type" VARCHAR(20) NOT NULL DEFAULT 'FULL',
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "last_run_ts" TIMESTAMP(3),
    "next_run_ts" TIMESTAMP(3),
    "last_run_status" VARCHAR(50),
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3),
    "max_retries" INTEGER NOT NULL DEFAULT 3,
    "retry_delay_seconds" INTEGER NOT NULL DEFAULT 60,
    CONSTRAINT "schedule_config_pkey" PRIMARY KEY ("schedule_id")
);

-- ── Run Orchestration (live cache - mirrored to Snowflake DQ_RUN_CONTROL on completion) ──
CREATE TABLE "dq_run_control" (
    "run_id" VARCHAR(100) NOT NULL,
    "execution_mode" VARCHAR(20),
    "trigger_type" VARCHAR(20),
    "triggered_by" VARCHAR(100),
    "start_ts" TIMESTAMP(3),
    "end_ts" TIMESTAMP(3),
    "duration_seconds" DECIMAL(10,2),
    "run_status" VARCHAR(50),
    "total_datasets" INTEGER,
    "datasets_processed" INTEGER,
    "datasets_skipped" INTEGER,
    "total_checks" INTEGER,
    "passed_checks" INTEGER,
    "failed_checks" INTEGER,
    "warning_checks" INTEGER,
    "skipped_checks" INTEGER,
    "error_message" VARCHAR(4000),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "run_type" VARCHAR(20),
    "row_level_records_processed" BIGINT,
    "total_rows_in_scope" BIGINT,
    "data_connection_id" VARCHAR(100),
    CONSTRAINT "dq_run_control_pkey" PRIMARY KEY ("run_id")
);

-- ── Operational / Reporting Support ───────────────────────────────
CREATE TABLE "report_jobs" (
    "report_id" VARCHAR(100) NOT NULL,
    "owner_id" VARCHAR(100) NOT NULL,
    "connection_id" VARCHAR(100) NOT NULL,
    "data_connection_id" VARCHAR(100),
    "connector_type" VARCHAR(20) NOT NULL,
    "status" VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    "format" VARCHAR(20),
    "file_path" TEXT,
    "report_date" DATE,
    "generated_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "generated_by" VARCHAR(100),
    "scope" VARCHAR(500),
    "metadata" JSONB,
    "download_count" INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT "report_jobs_pkey" PRIMARY KEY ("report_id")
);

CREATE TABLE "table_load_history" (
    "load_id" VARCHAR(100) NOT NULL,
    "database_name" VARCHAR(100) NOT NULL,
    "schema_name" VARCHAR(100) NOT NULL,
    "table_name" VARCHAR(100) NOT NULL,
    "load_type" VARCHAR(50),
    "load_status" VARCHAR(50),
    "rows_loaded" BIGINT NOT NULL DEFAULT 0,
    "rows_updated" BIGINT NOT NULL DEFAULT 0,
    "rows_deleted" BIGINT NOT NULL DEFAULT 0,
    "bytes_loaded" BIGINT NOT NULL DEFAULT 0,
    "load_start_time" TIMESTAMP(3),
    "load_end_time" TIMESTAMP(3),
    "duration_seconds" DECIMAL(10,2),
    "error_message" VARCHAR(4000),
    "triggered_by" VARCHAR(100),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "table_load_history_pkey" PRIMARY KEY ("load_id")
);

CREATE TABLE "data_lineage" (
    "id" VARCHAR(100) NOT NULL,
    "upstream_database" VARCHAR(100),
    "upstream_schema" VARCHAR(100),
    "upstream_table" VARCHAR(100),
    "downstream_database" VARCHAR(100),
    "downstream_schema" VARCHAR(100),
    "downstream_table" VARCHAR(100),
    "lineage_type" VARCHAR(50),
    "transformation_logic" VARCHAR(4000),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "data_lineage_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "runtime_state" (
    "state_key" VARCHAR(100) NOT NULL,
    "state_value" JSONB NOT NULL,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "runtime_state_pkey" PRIMARY KEY ("state_key")
);

-- ── Users, Connections & RBAC ──────────────────────────────────────
CREATE TABLE "users" (
    "id" VARCHAR(100) NOT NULL,
    "email" VARCHAR(255) NOT NULL,
    "name" VARCHAR(255),
    "password_hash" VARCHAR(255),
    "snowflake_username" VARCHAR(255),
    "is_active" BOOLEAN NOT NULL DEFAULT true,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "last_login_ts" TIMESTAMP(3),
    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "data_connections" (
    "id" VARCHAR(100) NOT NULL,
    "connector_type" VARCHAR(50) NOT NULL,
    "display_name" VARCHAR(200) NOT NULL,
    "owner_id" VARCHAR(100),
    "provider_config_id" VARCHAR(100),
    "is_active" BOOLEAN NOT NULL DEFAULT false,
    "status" VARCHAR(50),
    "resource_snapshot" JSONB,
    "resource_snapshot_ts" TIMESTAMP(3),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3),
    "retired_ts" TIMESTAMP(3),
    CONSTRAINT "data_connections_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "data_connection_grants" (
    "id" VARCHAR(100) NOT NULL,
    "user_id" VARCHAR(100) NOT NULL,
    "data_connection_id" VARCHAR(100) NOT NULL,
    "access_level" VARCHAR(20) NOT NULL,
    "granted_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3),
    "revoked_ts" TIMESTAMP(3),
    CONSTRAINT "data_connection_grants_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "data_connection_grant_audit" (
    "id" VARCHAR(100) NOT NULL,
    "user_id" VARCHAR(100) NOT NULL,
    "data_connection_id" VARCHAR(100) NOT NULL,
    "action" VARCHAR(30) NOT NULL,
    "access_level" VARCHAR(20),
    "actor_user_id" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "data_connection_grant_audit_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "data_connection_resource_grants" (
    "id" VARCHAR(100) NOT NULL,
    "user_id" VARCHAR(100) NOT NULL,
    "data_connection_id" VARCHAR(100) NOT NULL,
    "resource_type" VARCHAR(20) NOT NULL,
    "database_name" VARCHAR(200) NOT NULL,
    "schema_name" VARCHAR(200),
    "table_name" VARCHAR(200),
    "access_level" VARCHAR(20) NOT NULL,
    "granted_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3),
    "revoked_ts" TIMESTAMP(3),
    CONSTRAINT "data_connection_resource_grants_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "data_connection_resource_grant_audit" (
    "id" VARCHAR(100) NOT NULL,
    "user_id" VARCHAR(100) NOT NULL,
    "data_connection_id" VARCHAR(100) NOT NULL,
    "resource_type" VARCHAR(20) NOT NULL,
    "database_name" VARCHAR(200) NOT NULL,
    "schema_name" VARCHAR(200),
    "table_name" VARCHAR(200),
    "action" VARCHAR(30) NOT NULL,
    "access_level" VARCHAR(20),
    "actor_user_id" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "data_connection_resource_grant_audit_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "roles" (
    "role_id" SERIAL NOT NULL,
    "role_name" VARCHAR(50) NOT NULL,
    "description" VARCHAR(255),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "roles_pkey" PRIMARY KEY ("role_id")
);

CREATE TABLE "user_roles" (
    "id" SERIAL NOT NULL,
    "user_id" VARCHAR(100) NOT NULL,
    "role_id" INTEGER NOT NULL,
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "user_roles_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "snowflake_connections" (
    "id" TEXT NOT NULL,
    "data_connection_id" VARCHAR(100),
    "name" VARCHAR(200) NOT NULL,
    "account" VARCHAR(200) NOT NULL,
    "username" VARCHAR(200) NOT NULL,
    "warehouse" VARCHAR(100) NOT NULL,
    "default_database" VARCHAR(100),
    "default_schema" VARCHAR(100),
    "role" VARCHAR(100),
    "auth_method" VARCHAR(50) NOT NULL,
    "encrypted_creds" TEXT,
    "is_active" BOOLEAN NOT NULL DEFAULT false,
    "last_test_status" VARCHAR(20),
    "last_test_message" VARCHAR(500),
    "last_test_ts" TIMESTAMP(3),
    "last_used_ts" TIMESTAMP(3),
    "created_by" VARCHAR(100),
    "created_ts" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "modified_ts" TIMESTAMP(3),
    "source_fingerprint" VARCHAR(64),
    "retired_ts" TIMESTAMP(3),
    CONSTRAINT "snowflake_connections_pkey" PRIMARY KEY ("id")
);

-- ── Indexes ────────────────────────────────────────────────────────
CREATE UNIQUE INDEX "rule_master_rule_name_key" ON "rule_master"("rule_name");
CREATE INDEX "dataset_config_data_connection_id_idx" ON "dataset_config"("data_connection_id");
CREATE INDEX "dataset_config_connector_type_source_fingerprint_idx" ON "dataset_config"("connector_type", "source_fingerprint");
CREATE UNIQUE INDEX "dataset_config_connector_type_connection_id_source_database_key" ON "dataset_config"("connector_type", "connection_id", "source_database", "source_schema", "source_table");
CREATE INDEX "dataset_columns_dataset_id_ordinal_position_idx" ON "dataset_columns"("dataset_id", "ordinal_position");
CREATE UNIQUE INDEX "dataset_columns_dataset_id_column_name_key" ON "dataset_columns"("dataset_id", "column_name");
CREATE INDEX "dataset_schema_snapshot_dataset_id_is_active_baseline_idx" ON "dataset_schema_snapshot"("dataset_id", "is_active_baseline");
CREATE INDEX "dataset_schema_snapshot_dataset_id_baseline_version_idx" ON "dataset_schema_snapshot"("dataset_id", "baseline_version");
CREATE INDEX "dataset_rule_config_dataset_id_rule_id_column_name_idx" ON "dataset_rule_config"("dataset_id", "rule_id", "column_name");
CREATE INDEX "dq_run_control_data_connection_id_idx" ON "dq_run_control"("data_connection_id");
CREATE INDEX "report_jobs_owner_id_generated_at_idx" ON "report_jobs"("owner_id", "generated_at");
CREATE INDEX "report_jobs_data_connection_id_idx" ON "report_jobs"("data_connection_id");
CREATE INDEX "report_jobs_status_generated_at_idx" ON "report_jobs"("status", "generated_at");
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");
CREATE UNIQUE INDEX "users_snowflake_username_key" ON "users"("snowflake_username");
CREATE INDEX "data_connections_owner_id_connector_type_idx" ON "data_connections"("owner_id", "connector_type");
CREATE INDEX "data_connections_is_active_retired_ts_idx" ON "data_connections"("is_active", "retired_ts");
CREATE UNIQUE INDEX "data_connections_connector_type_provider_config_id_key" ON "data_connections"("connector_type", "provider_config_id");
CREATE INDEX "data_connection_grants_user_id_revoked_ts_idx" ON "data_connection_grants"("user_id", "revoked_ts");
CREATE INDEX "data_connection_grants_data_connection_id_revoked_ts_idx" ON "data_connection_grants"("data_connection_id", "revoked_ts");
CREATE UNIQUE INDEX "data_connection_grants_user_id_data_connection_id_key" ON "data_connection_grants"("user_id", "data_connection_id");
CREATE INDEX "data_connection_grant_audit_user_id_created_ts_idx" ON "data_connection_grant_audit"("user_id", "created_ts");
CREATE INDEX "data_connection_grant_audit_data_connection_id_created_ts_idx" ON "data_connection_grant_audit"("data_connection_id", "created_ts");
CREATE INDEX "data_connection_resource_grants_user_id_data_connection_id__idx" ON "data_connection_resource_grants"("user_id", "data_connection_id", "revoked_ts");
CREATE INDEX "data_connection_resource_grants_data_connection_id_database_idx" ON "data_connection_resource_grants"("data_connection_id", "database_name", "schema_name", "table_name");
CREATE INDEX "data_connection_resource_grant_audit_user_id_data_connectio_idx" ON "data_connection_resource_grant_audit"("user_id", "data_connection_id", "created_ts");
CREATE INDEX "data_connection_resource_grant_audit_data_connection_id_dat_idx" ON "data_connection_resource_grant_audit"("data_connection_id", "database_name", "schema_name", "table_name");
CREATE UNIQUE INDEX "roles_role_name_key" ON "roles"("role_name");
CREATE UNIQUE INDEX "user_roles_user_id_role_id_key" ON "user_roles"("user_id", "role_id");
CREATE UNIQUE INDEX "snowflake_connections_data_connection_id_key" ON "snowflake_connections"("data_connection_id");
CREATE INDEX "snowflake_connections_created_by_source_fingerprint_idx" ON "snowflake_connections"("created_by", "source_fingerprint");

-- ── Foreign Keys ───────────────────────────────────────────────────
ALTER TABLE "rule_sql_template" ADD CONSTRAINT "rule_sql_template_rule_id_fkey" FOREIGN KEY ("rule_id") REFERENCES "rule_master"("rule_id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "dataset_columns" ADD CONSTRAINT "dataset_columns_dataset_id_fkey" FOREIGN KEY ("dataset_id") REFERENCES "dataset_config"("dataset_id") ON DELETE CASCADE ON UPDATE CASCADE;
ALTER TABLE "dataset_schema_snapshot" ADD CONSTRAINT "dataset_schema_snapshot_dataset_id_fkey" FOREIGN KEY ("dataset_id") REFERENCES "dataset_config"("dataset_id") ON DELETE CASCADE ON UPDATE CASCADE;
ALTER TABLE "dataset_rule_config" ADD CONSTRAINT "dataset_rule_config_dataset_id_fkey" FOREIGN KEY ("dataset_id") REFERENCES "dataset_config"("dataset_id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "dataset_rule_config" ADD CONSTRAINT "dataset_rule_config_rule_id_fkey" FOREIGN KEY ("rule_id") REFERENCES "rule_master"("rule_id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "allowed_values_config" ADD CONSTRAINT "allowed_values_config_dataset_id_fkey" FOREIGN KEY ("dataset_id") REFERENCES "dataset_config"("dataset_id") ON DELETE SET NULL ON UPDATE CASCADE;
ALTER TABLE "fk_config" ADD CONSTRAINT "fk_config_dataset_id_fkey" FOREIGN KEY ("dataset_id") REFERENCES "dataset_config"("dataset_id") ON DELETE SET NULL ON UPDATE CASCADE;
ALTER TABLE "data_connection_grants" ADD CONSTRAINT "data_connection_grants_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "data_connection_grants" ADD CONSTRAINT "data_connection_grants_data_connection_id_fkey" FOREIGN KEY ("data_connection_id") REFERENCES "data_connections"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "data_connection_resource_grants" ADD CONSTRAINT "data_connection_resource_grants_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "data_connection_resource_grants" ADD CONSTRAINT "data_connection_resource_grants_data_connection_id_fkey" FOREIGN KEY ("data_connection_id") REFERENCES "data_connections"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "user_roles" ADD CONSTRAINT "user_roles_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "user_roles" ADD CONSTRAINT "user_roles_role_id_fkey" FOREIGN KEY ("role_id") REFERENCES "roles"("role_id") ON DELETE RESTRICT ON UPDATE CASCADE;
```

**Dropped from your draft (Snowflake already owns these - see Part 1):** `dq_check_results`, `dq_failed_records`, `dq_daily_summary`, `dq_weekly_summary`, `dq_column_profile`, plus their indexes (`dq_check_results_check_timestamp_table_name_idx`, `dq_check_results_data_connection_id_idx`, `dq_failed_records_detected_ts_table_name_idx`, `dq_daily_summary_summary_date_table_name_idx`) and FKs (`dq_check_results_run_id_fkey`, `dq_failed_records_check_id_fkey`, `dq_daily_summary_dataset_id_fkey`). `dq_monthly_summary` and `dq_quarterly_summary` were dropped too - same rollup family as `dq_daily_summary`/`dq_weekly_summary`, so they belong next to those in Snowflake (add there if/when monthly/quarterly reporting is needed, not duplicated here).

## Deployment Architecture - Where Data Lives

PiQlens can be deployed 3 ways. Choice depends on the client's use-case (data volume, budget, compliance).

### Option A - Full Snowflake-Native
Everything — scan results, failed records, rule config, RBAC - lives in Snowflake.
- ✅ Single platform, simplest ops.
- ❌ Every small UI read (permission check, rule lookup, live run status) spins a Snowflake warehouse and burns credits. Expensive at scale because Snowflake bills per compute-second regardless of query size.

### Option B - Full PostgreSQL-Native
All scan data, results and failed records move out of Snowflake into Postgres.
- ✅ Cheap, predictable fixed infra cost.
- ❌ Loses compute-next-to-data: scan engine has to pull rows out of the client's Snowflake tables into Postgres before checking them - adds ETL/egress latency and duplicates storage for large tables.

### Option C - Hybrid (Recommended)
Split by **access pattern**, not by "all data": bulk/scan-volume data stays where it's scanned (Snowflake); small, frequently-hit application data moves to Postgres.



| Layer                    | Lives in               | Tables                                                                                 | Why                                                                                                                                                      |
| ------------------------ | ---------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Scan results & forensics | **Snowflake**          | `DQ_CHECK_RESULTS`, `DQ_FAILED_RECORDS`, `DQ_COLUMN_PROFILE`                           | High-volume, written once per scan, compute happens where the source data already sits no data movement.                                                 |
| Reporting / aggregates   | **Snowflake**          | `DQ_DAILY_SUMMARY`, `DQ_WEEKLY_SUMMARY`, `PIQLENS_RULE_RESULTS`, `PIQLENS_REPORT_VIEW` | Rolls up from the tables above via SQL aggregation  cheapest to compute in-place.                                                                        |
| App metadata / config    | **PostgreSQL**         | `PIQLENS_REPORT_CONFIG`, rule definitions, RBAC (users/roles/permissions)              | Small rows, read/written on every UI request a Snowflake warehouse spin-up for a "does this user have access" check is wasteful.                         |
| Run orchestration        | **PostgreSQL** (cache) | `DQ_RUN_CONTROL`                                                                       | Polled repeatedly by the UI for live run status; kept as a lightweight cache row, mirrored/archived to Snowflake at run completion for historical audit. |

**Rule of thumb:** if it's scanned/aggregated in bulk → Snowflake. If it's read on every page load or gates access → Postgres.

⚠️ Open item: `DQ_RUN_CONTROL` currently exists only in the Snowflake DDL above. For the hybrid model it needs a companion Postgres table (live cache) + a sync/archive job writing final run state back to the Snowflake copy for audit history. Confirm before build starts so product doesn't discover this mid-sprint.

## Master Data Model — Full Table Index

Every table/view across both databases, in one place, so nobody has to hunt through Parts 1–3 to answer "where does X live."

### Database: `PIQLENS_DQ_DB` (Snowflake)

| # | Table / View | Type | Primary Key | Grain | Purpose |
|---|---|---|---|---|---|
| 1 | `DQ_RUN_CONTROL` | Table | `RUN_ID` | 1 row per scan run | Scan run lifecycle — status, timing, pass/fail counts |
| 2 | `DQ_CHECK_RESULTS` | Table | `CHECK_ID` | 1 row per rule evaluation | Per-rule pass/fail metrics for a run |
| 3 | `DQ_FAILED_RECORDS` | Table | `FAILURE_ID` | 1 row per failing source row | Row-level forensics for failed checks |
| 4 | `DQ_DAILY_SUMMARY` | Table | `SUMMARY_ID` | 1 row per table per day | Daily DQ score across 6 dimensions + SLA flag |
| 5 | `DQ_WEEKLY_SUMMARY` | Table | `ID` | 1 row per table per week | Weekly rollup of daily scores + trend |
| 6 | `DQ_COLUMN_PROFILE` | Table | `PROFILE_ID` | 1 row per column per run | Column-level profiling stats (nulls, distinct, min/max) |
| 7 | `PIQLENS_RULE_RESULTS` | Table | *(none — append-only)* | 1 row per tag per rule-type per date | Business metric results feeding the client report |
| 8 | `PIQLENS_REPORT_CONFIG` | Table | `BASE_TAG` | 1 row per report line item | Maps metric tags to report sections/display order |
| 9 | `PIQLENS_REPORT_VIEW` | View | — | 1 row per config line | Final client-facing report, joins config + latest results |

### Database: PostgreSQL (`public` schema — app metadata, config & RBAC)

| # | Table | Primary Key | Grain | Purpose |
|---|---|---|---|---|
| 1 | `rule_master` | `rule_id` | 1 row per rule definition | Catalog of all available DQ rules |
| 2 | `rule_sql_template` | `template_id` | 1 row per rule SQL version | Versioned SQL templates behind each rule |
| 3 | `dataset_config` | `dataset_id` | 1 row per source table registered | Which client tables are onboarded for scanning |
| 4 | `dataset_columns` | `id` | 1 row per column per dataset | Current column list/types for a dataset |
| 5 | `dataset_schema_snapshot` | `id` | 1 row per column per baseline version | Schema-drift baseline history |
| 6 | `dataset_rule_config` | `config_id` | 1 row per rule applied to a dataset | Which rules run on which dataset/column + threshold |
| 7 | `weights_mapping` | `weight_id` | 1 row per rule-type/domain | Scoring weight & priority per rule type |
| 8 | `allowed_values_config` | `id` | 1 row per column allow-list | Valid-value sets for validity checks |
| 9 | `fk_config` | `id` | 1 row per referential rule | Parent/child column mapping for FK checks |
| 10 | `schedule_config` | `schedule_id` | 1 row per scheduled scan | Cron schedule + retry policy per dataset |
| 11 | `dq_run_control` | `run_id` | 1 row per in-flight/recent run | **Live cache** of run status, mirrored to Snowflake `DQ_RUN_CONTROL` on completion |
| 12 | `report_jobs` | `report_id` | 1 row per generated report file | Report generation status & download tracking |
| 13 | `table_load_history` | `load_id` | 1 row per data load | Ingestion/load audit trail |
| 14 | `data_lineage` | `id` | 1 row per upstream→downstream edge | Table-level lineage graph |
| 15 | `runtime_state` | `state_key` | 1 row per app state key | Generic key-value app runtime state |
| 16 | `users` | `id` | 1 row per user | User accounts |
| 17 | `data_connections` | `id` | 1 row per connected data source | Client connection registry (any connector type) |
| 18 | `data_connection_grants` | `id` | 1 row per user×connection grant | RBAC — connection-level access |
| 19 | `data_connection_grant_audit` | `id` | 1 row per grant change | RBAC audit trail — connection-level |
| 20 | `data_connection_resource_grants` | `id` | 1 row per user×db/schema/table grant | RBAC — fine-grained resource access |
| 21 | `data_connection_resource_grant_audit` | `id` | 1 row per resource-grant change | RBAC audit trail — resource-level |
| 22 | `roles` | `role_id` | 1 row per role | Role catalog |
| 23 | `user_roles` | `id` | 1 row per user×role | User-to-role assignment |
| 24 | `snowflake_connections` | `id` | 1 row per Snowflake connection | Snowflake-specific connection credentials/config |

**Total: 9 Snowflake objects (8 tables + 1 view) + 24 PostgreSQL tables = 33 objects.**

![[Qlens-Deployment Approaches 1.png]]



![[PiQlens_PostgreSQL_ERD_PRODUCTION.drawio.png]]

![[PIQLENS_DQ_Snowflake_DB_ERD.drawio.png]]