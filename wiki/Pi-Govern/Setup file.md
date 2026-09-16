

CREATE SCHEMA IF NOT EXISTS "APP_CORE";

CREATE SCHEMA IF NOT EXISTS "RBAC";

CREATE SCHEMA IF NOT EXISTS "METADATA";

CREATE SCHEMA IF NOT EXISTS "LINEAGE";

CREATE SCHEMA IF NOT EXISTS "POLICIES";

CREATE SCHEMA IF NOT EXISTS "GOVERNANCE_EVENTS";

CREATE SCHEMA IF NOT EXISTS "AUDIT";

CREATE SCHEMA IF NOT EXISTS "AI_ASSISTANT";


        CREATE TABLE IF NOT EXISTS RBAC.ROLES (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          name VARCHAR(100) NOT NULL UNIQUE,
          description VARCHAR(2000),
          is_system BOOLEAN NOT NULL DEFAULT FALSE,
          created_at TIMESTAMP_TZ NOT NULL,
          updated_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS RBAC.PERMISSIONS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          resource VARCHAR(255) NOT NULL,
          action VARCHAR(100) NOT NULL,
          role_id VARCHAR(36) NOT NULL,
          conditions VARIANT
        )
    


        CREATE TABLE IF NOT EXISTS RBAC.ROLE_ASSIGNMENTS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          snowflake_identity VARCHAR(255) NOT NULL UNIQUE,
          role_id VARCHAR(36) NOT NULL,
          assigned_by VARCHAR(255),
          assigned_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS AUDIT.AUDIT_LOGS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          action VARCHAR(100) NOT NULL,
          resource_type VARCHAR(100) NOT NULL,
          resource_id VARCHAR(255),
          resource_name VARCHAR(255),
          actor_id VARCHAR(255) NOT NULL,
          actor_email VARCHAR(255),
          extra_metadata VARIANT,
          timestamp TIMESTAMP_TZ NOT NULL,
          ip_address VARCHAR(50)
        )
    


        CREATE TABLE IF NOT EXISTS AI_ASSISTANT.CONVERSATIONS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          user_id VARCHAR(255) NOT NULL,
          title VARCHAR(255),
          created_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS AI_ASSISTANT.MESSAGES (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          conversation_id VARCHAR(36) NOT NULL,
          role VARCHAR(20) NOT NULL,
          content VARCHAR(16777216) NOT NULL,
          created_at TIMESTAMP_TZ NOT NULL,
          FOREIGN KEY (conversation_id) REFERENCES AI_ASSISTANT.CONVERSATIONS(id)
        )
    


        CREATE TABLE IF NOT EXISTS POLICIES.POLICIES (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          name VARCHAR(255) NOT NULL,
          description VARCHAR(2000),
          status VARCHAR(20) NOT NULL DEFAULT 'draft',
          policy_type VARCHAR(100) NOT NULL,
          rules VARIANT,
          applies_to VARIANT,
          created_by VARCHAR(255),
          created_at TIMESTAMP_TZ NOT NULL,
          updated_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS POLICIES.POLICY_EVALUATION_STATE (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          policy_id VARCHAR(36) NOT NULL,
          asset_id VARCHAR(36) NOT NULL,
          status VARCHAR(50) NOT NULL,
          last_evaluated_at TIMESTAMP_TZ NOT NULL,
          details VARIANT,
          UNIQUE (policy_id, asset_id)
        )
    


        CREATE TABLE IF NOT EXISTS METADATA.DATA_ASSETS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          name VARCHAR(255) NOT NULL,
          description VARCHAR(4000),
          asset_type VARCHAR(100) NOT NULL,
          source VARCHAR(255),
          schema_name VARCHAR(255),
          database_name VARCHAR(255),
          tags VARIANT,
          owner VARCHAR(255),
          steward VARCHAR(255),
          quality_score FLOAT,
          sensitivity VARCHAR(50),
          row_count NUMBER,
          size_bytes NUMBER,
          governance_state VARCHAR(20) NOT NULL DEFAULT 'review',
          created_at TIMESTAMP_TZ NOT NULL,
          updated_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS METADATA.ASSET_COLUMNS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          asset_id VARCHAR(36) NOT NULL,
          name VARCHAR(255) NOT NULL,
          data_type VARCHAR(100) NOT NULL,
          classification VARCHAR(50),
          masked BOOLEAN NOT NULL DEFAULT FALSE,
          masking_rule VARCHAR(255),
          description VARCHAR(4000),
          FOREIGN KEY (asset_id) REFERENCES METADATA.DATA_ASSETS(id)
        )
    


        CREATE TABLE IF NOT EXISTS LINEAGE.LINEAGE_NODES (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          name VARCHAR(255) NOT NULL,
          node_type VARCHAR(100) NOT NULL,
          source VARCHAR(255),
          status VARCHAR(20) NOT NULL DEFAULT 'healthy',
          extra_metadata VARIANT
        )
    


        CREATE TABLE IF NOT EXISTS LINEAGE.LINEAGE_EDGES (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          source_id VARCHAR(36) NOT NULL,
          target_id VARCHAR(36) NOT NULL,
          transformation VARCHAR(4000),
          created_at TIMESTAMP_TZ NOT NULL,
          FOREIGN KEY (source_id) REFERENCES LINEAGE.LINEAGE_NODES(id),
          FOREIGN KEY (target_id) REFERENCES LINEAGE.LINEAGE_NODES(id)
        )
    


        CREATE TABLE IF NOT EXISTS GOVERNANCE_EVENTS.GOVERNANCE_EVENTS (
          id VARCHAR(36) NOT NULL PRIMARY KEY,
          title VARCHAR(255) NOT NULL,
          description VARCHAR(4000),
          severity VARCHAR(20) NOT NULL DEFAULT 'info',
          status VARCHAR(20) NOT NULL DEFAULT 'open',
          resource_id VARCHAR(255),
          resource_type VARCHAR(100),
          created_at TIMESTAMP_TZ NOT NULL,
          resolved_at TIMESTAMP_TZ
        )
    


        CREATE TABLE IF NOT EXISTS APP_CORE.BOOTSTRAP_SYNC_STATUS (
          database_name VARCHAR(255) NOT NULL PRIMARY KEY,
          status VARCHAR(20) NOT NULL,
          last_synced_by VARCHAR(255),
          last_synced_at TIMESTAMP_TZ,
          assets_synced NUMBER NOT NULL DEFAULT 0,
          columns_synced NUMBER NOT NULL DEFAULT 0,
          nodes_synced NUMBER NOT NULL DEFAULT 0,
          edges_synced NUMBER NOT NULL DEFAULT 0,
          error_message VARCHAR(4000),
          updated_at TIMESTAMP_TZ NOT NULL
        )
    


        CREATE TABLE IF NOT EXISTS APP_CORE.BOOTSTRAP_DISCOVERY_CACHE (
          snowflake_user VARCHAR(255) NOT NULL,
          role_name VARCHAR(255) NOT NULL,
          discovered_databases VARIANT NOT NULL,
          last_discovered_at TIMESTAMP_TZ NOT NULL,
          PRIMARY KEY (snowflake_user, role_name)
        )
    


        CREATE PROCEDURE IF NOT EXISTS APP_CORE.SYNC_METADATA_DATABASE(DB_NAME STRING)
        RETURNS VARIANT
        LANGUAGE SQL
        EXECUTE AS CALLER
        AS
        $$
        DECLARE
          assets_synced NUMBER := 0;
          columns_synced NUMBER := 0;
          tables_ref VARCHAR;
          columns_ref VARCHAR;
          invalid_db_name EXCEPTION (-20001, 'Invalid database name');
        BEGIN
          IF (DB_NAME NOT REGEXP '^[A-Za-z_][A-Za-z0-9_$]*$') THEN
            RAISE invalid_db_name;
          END IF;

          tables_ref := '"' || DB_NAME || '".INFORMATION_SCHEMA.TABLES';
          columns_ref := '"' || DB_NAME || '".INFORMATION_SCHEMA.COLUMNS';

          -- Assets: upsert one row per table/view in the governed database.
          MERGE INTO METADATA.DATA_ASSETS t
          USING (
            SELECT
              table_schema,
              table_name,
              CASE UPPER(table_type)
                WHEN 'BASE TABLE' THEN 'TABLE'
                WHEN 'VIEW' THEN 'VIEW'
                WHEN 'MATERIALIZED VIEW' THEN 'MATERIALIZED_VIEW'
                WHEN 'EXTERNAL TABLE' THEN 'EXTERNAL_TABLE'
                WHEN 'EVENT TABLE' THEN 'EVENT_TABLE'
                ELSE 'TABLE'
              END AS asset_type,
              row_count,
              bytes,
              comment
            FROM IDENTIFIER(:tables_ref)
            WHERE table_schema <> 'INFORMATION_SCHEMA'
          ) s
          ON t.database_name = :DB_NAME AND t.schema_name = s.table_schema AND t.name = s.table_name
          WHEN MATCHED THEN UPDATE SET
            asset_type = s.asset_type,
            row_count = s.row_count,
            size_bytes = s.bytes,
            description = COALESCE(s.comment, t.description),
            updated_at = CURRENT_TIMESTAMP()
          WHEN NOT MATCHED THEN INSERT
            (id, name, description, asset_type, source, schema_name, database_name,
             row_count, size_bytes, governance_state, created_at, updated_at)
            VALUES
            (UUID_STRING(), s.table_name, s.comment, s.asset_type,
             :DB_NAME || '.' || s.table_schema, s.table_schema, :DB_NAME,
             s.row_count, s.bytes, 'review', CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP());

          -- Columns: upsert per asset, preserving steward-set classification/masking.
          MERGE INTO METADATA.ASSET_COLUMNS c
          USING (
            SELECT a.id AS asset_id, col.column_name AS name, col.data_type, col.comment
            FROM METADATA.DATA_ASSETS a
            JOIN IDENTIFIER(:columns_ref) col
              ON col.table_schema = a.schema_name AND col.table_name = a.name
            WHERE a.database_name = :DB_NAME
          ) s
          ON c.asset_id = s.asset_id AND c.name = s.name
          WHEN MATCHED THEN UPDATE SET
            data_type = s.data_type,
            description = COALESCE(s.comment, c.description)
          WHEN NOT MATCHED THEN INSERT
            (id, asset_id, name, data_type, classification, masked, masking_rule, description)
            VALUES
            (UUID_STRING(), s.asset_id, s.name, s.data_type, NULL, FALSE, NULL, s.comment);

          assets_synced := (SELECT COUNT(*) FROM METADATA.DATA_ASSETS WHERE database_name = :DB_NAME);
          columns_synced := (
            SELECT COUNT(*) FROM METADATA.ASSET_COLUMNS c
            JOIN METADATA.DATA_ASSETS a ON a.id = c.asset_id
            WHERE a.database_name = :DB_NAME
          );

          RETURN OBJECT_CONSTRUCT('assets_synced', assets_synced, 'columns_synced', columns_synced);
        END;
        $$
    


        CREATE PROCEDURE IF NOT EXISTS APP_CORE.SYNC_LINEAGE_DATABASE(DB_NAME STRING)
        RETURNS VARIANT
        LANGUAGE SQL
        EXECUTE AS CALLER
        AS
        $$
        DECLARE
          nodes_synced NUMBER := 0;
          edges_synced NUMBER := 0;
          node_type_of STRING;
          invalid_db_name EXCEPTION (-20001, 'Invalid database name');
        BEGIN
          IF (DB_NAME NOT REGEXP '^[A-Za-z_][A-Za-z0-9_$]*$') THEN
            RAISE invalid_db_name;
          END IF;

          -- Nodes: one per distinct object referenced from / referencing the DB.
          MERGE INTO LINEAGE.LINEAGE_NODES n
          USING (
            SELECT qualified_name, node_type, db FROM (
              SELECT
                referenced_database || '.' || referenced_schema || '.' || referenced_object_name AS qualified_name,
                CASE UPPER(referenced_object_domain)
                  WHEN 'VIEW' THEN 'VIEW'
                  WHEN 'MATERIALIZED VIEW' THEN 'VIEW'
                  WHEN 'EXTERNAL TABLE' THEN 'TABLE'
                  WHEN 'STAGE' THEN 'STAGE'
                  WHEN 'FUNCTION' THEN 'FUNCTION'
                  ELSE 'TABLE'
                END AS node_type,
                referenced_database AS db
              FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
              WHERE referenced_database = :DB_NAME OR referencing_database = :DB_NAME
              UNION
              SELECT
                referencing_database || '.' || referencing_schema || '.' || referencing_object_name AS qualified_name,
                CASE UPPER(referencing_object_domain)
                  WHEN 'VIEW' THEN 'VIEW'
                  WHEN 'MATERIALIZED VIEW' THEN 'VIEW'
                  WHEN 'EXTERNAL TABLE' THEN 'TABLE'
                  WHEN 'STAGE' THEN 'STAGE'
                  WHEN 'FUNCTION' THEN 'FUNCTION'
                  ELSE 'TABLE'
                END AS node_type,
                referencing_database AS db
              FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
              WHERE referenced_database = :DB_NAME OR referencing_database = :DB_NAME
            )
            WHERE qualified_name IS NOT NULL
          ) s
          ON n.name = s.qualified_name
          WHEN MATCHED THEN UPDATE SET node_type = s.node_type, source = s.db
          WHEN NOT MATCHED THEN INSERT (id, name, node_type, source, status)
            VALUES (UUID_STRING(), s.qualified_name, s.node_type, s.db, 'healthy');

          -- Edges: one per dependency, keyed by resolved node ids.
          MERGE INTO LINEAGE.LINEAGE_EDGES e
          USING (
            SELECT ns.id AS source_id, nt.id AS target_id
            FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES od
            JOIN LINEAGE.LINEAGE_NODES ns
              ON ns.name = od.referenced_database || '.' || od.referenced_schema || '.' || od.referenced_object_name
            JOIN LINEAGE.LINEAGE_NODES nt
              ON nt.name = od.referencing_database || '.' || od.referencing_schema || '.' || od.referencing_object_name
            WHERE (od.referenced_database = :DB_NAME OR od.referencing_database = :DB_NAME)
              AND ns.id IS NOT NULL AND nt.id IS NOT NULL
          ) s
          ON e.source_id = s.source_id AND e.target_id = s.target_id
          WHEN NOT MATCHED THEN INSERT (id, source_id, target_id, transformation, created_at)
            VALUES (UUID_STRING(), s.source_id, s.target_id, 'OBJECT_DEPENDENCIES', CURRENT_TIMESTAMP());

          nodes_synced := (SELECT COUNT(*) FROM LINEAGE.LINEAGE_NODES WHERE source = :DB_NAME);
          edges_synced := (
            SELECT COUNT(*) FROM LINEAGE.LINEAGE_EDGES e
            JOIN LINEAGE.LINEAGE_NODES ns ON ns.id = e.source_id
            JOIN LINEAGE.LINEAGE_NODES nt ON nt.id = e.target_id
            WHERE ns.source = :DB_NAME OR nt.source = :DB_NAME
          );

          RETURN OBJECT_CONSTRUCT('nodes_synced', nodes_synced, 'edges_synced', edges_synced);
        END;
        $$
    


        CREATE VIEW IF NOT EXISTS METADATA.V_ASSET_GOVERNANCE AS
        SELECT
          a.id,
          a.name,
          a.description,
          a.asset_type,
          a.source,
          a.schema_name,
          a.database_name,
          a.tags,
          a.owner,
          a.steward,
          a.quality_score,
          a.sensitivity,
          a.row_count,
          a.size_bytes,
          a.governance_state,
          a.created_at,
          a.updated_at,
          COALESCE(cs.column_count, 0)    AS column_count,
          COALESCE(cs.pii_count, 0)       AS pii_count,
          COALESCE(cs.sensitive_count, 0) AS sensitive_count,
          COALESCE(pb.policy_bound_count, 0) AS policy_bound_count
        FROM METADATA.DATA_ASSETS a
        LEFT JOIN (
          SELECT
            asset_id,
            COUNT(*)                                                              AS column_count,
            COUNT_IF(UPPER(classification) = 'PII')                              AS pii_count,
            COUNT_IF(UPPER(classification) IN ('SENSITIVE', 'CONFIDENTIAL', 'FINANCIAL'))
                                                                                 AS sensitive_count
          FROM METADATA.ASSET_COLUMNS
          GROUP BY asset_id
        ) cs ON cs.asset_id = a.id
        LEFT JOIN (
          SELECT f.value::VARCHAR AS asset_id, COUNT(DISTINCT p.id) AS policy_bound_count
          FROM POLICIES.POLICIES p,
               LATERAL FLATTEN(input => PARSE_JSON(p.applies_to::VARCHAR)) f
          GROUP BY 1
        ) pb ON pb.asset_id = a.id
    


        INSERT INTO RBAC.ROLES (id, name, description, is_system, created_at, updated_at)
        SELECT UUID_STRING(), 'admin', 'Full platform access', TRUE, CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP()
        WHERE NOT EXISTS (SELECT 1 FROM RBAC.ROLES WHERE name = 'admin')
    


        INSERT INTO RBAC.ROLES (id, name, description, is_system, created_at, updated_at)
        SELECT UUID_STRING(), 'viewer', 'Read-only access', TRUE, CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP()
        WHERE NOT EXISTS (SELECT 1 FROM RBAC.ROLES WHERE name = 'viewer')
    


        INSERT INTO RBAC.PERMISSIONS (id, resource, action, role_id)
        SELECT UUID_STRING(), '*', '*', r.id FROM RBAC.ROLES r
        WHERE r.name = 'admin'
          AND NOT EXISTS (
            SELECT 1 FROM RBAC.PERMISSIONS p WHERE p.role_id = r.id AND p.resource = '*' AND p.action = '*'
          )
    


        INSERT INTO RBAC.PERMISSIONS (id, resource, action, role_id)
        SELECT UUID_STRING(), resource, 'read', r.id
        FROM RBAC.ROLES r,
             (SELECT 'metadata' AS resource UNION ALL SELECT 'lineage' UNION ALL SELECT 'policies'
              UNION ALL SELECT 'governance_events' UNION ALL SELECT 'audit' UNION ALL SELECT 'rbac') v
        WHERE r.name = 'viewer'
          AND NOT EXISTS (
            SELECT 1 FROM RBAC.PERMISSIONS p WHERE p.role_id = r.id AND p.resource = v.resource AND p.action = 'read'
          )






Cannot perform CREATE SCHEMA. This session does not have a current database. Call 'USE DATABASE', or use a qualified name.