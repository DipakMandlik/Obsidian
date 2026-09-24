# PI-GOVERN - System Architecture Specification

## 1. Executive Architecture Summary

PI-GOVERN is a Snowflake-native data governance control plane, intelligence layer, and execution engine. It installs directly inside a customer's Snowflake account, discovers the account's data estate, persists a governance catalog within a dedicated `PI_GOVERN` Snowflake database, and layers metadata, lineage, policy evaluation, in-app RBAC, audit logging, and AI capabilities on top.

### Foundational Architectural Principles
- **Snowflake as the Sole Data Store**: No PostgreSQL, MySQL, Redis, DynamoDB, or external datastore. All catalog metadata, policy rules, audit trails, and session states are persisted directly in Snowflake.
- **Mirror-and-Govern Data Flow**: Real-time read operations serve from PI-GOVERN mirror tables rather than repeatedly scanning Snowflake `ACCOUNT_USAGE` or `INFORMATION_SCHEMA`.
- **Snowflake Session as the Security Boundary**: User credentials authenticate against Snowflake directly, creating a live Snowpark session. All query execution inherits caller Snowflake permissions.
- **Zero Data Exfiltration**: AI inference runs via `SNOWFLAKE.CORTEX.COMPLETE` inside the Snowflake perimeter. No customer metadata or data leaves Snowflake.
- **Non-Destructive Idempotency**: All DDL uses `CREATE ... IF NOT EXISTS`, all seeds use `INSERT ... WHERE NOT EXISTS`, and stored procedures execute as caller (`EXECUTE AS CALLER`).

---

## 2. End-to-End System Topology

```
+-----------------------------------------------------------------------------------+
| BROWSER RUNTIME (Next.js 15 App Router / React 19)                                 |
|                                                                                   |
|  Providers: ThemeProvider > QueryClientProvider > WorkspaceProvider > Sidebar    |
|  Gate Sequence: DashboardAccessGate -> InitGate -> SyncGate -> AppShell           |
|  State Management: Zustand Stores (auth, workspace, sidebar, preferences, ui)     |
|  API Transport: Axios HTTP client with Bearer token & 401 interceptor             |
+------------------------------------------+----------------------------------------+
                                           | HTTPS /api/v1/*
                                           v
+-----------------------------------------------------------------------------------+
| FASTAPI BACKEND RUNTIME (Python 3.11+ / Uvicorn)                                  |
|                                                                                   |
|  Middleware Pipeline:                                                             |
|   1. SlowAPI (Rate limiting: 10/min login, 30/min AI)                             |
|   2. CORS Middleware (Configurable allowed origins)                               |
|   3. GZip Middleware (Response compression for payloads > 1KB)                    |
|   4. Security Headers & Request ID Middleware (CSP, Strict-Transport-Security)    |
|   5. Audit Log Middleware (Intercepts successful mutations, writes to AUDIT)      |
|                                                                                   |
|  Dependency Injection (DI) & Context Assembly:                                    |
|   - get_request_context(): Maps Bearer token -> SnowparkSessionManager            |
|   - require_permission(resource, action): In-app RBAC permission evaluator        |
|                                                                                   |
|  Service & Repository Layer:                                                      |
|   - Core Services: Bootstrap, BootstrapInit, SnowflakeSession, Cortex             |
|   - Domain Modules: Metadata, Lineage, Policies, RBAC, Events, Audit, AI, Dashboard|
|   - db/snowflake_base.py: Parameterized CRUD operations using ? binds             |
+------------------------------------------+----------------------------------------+
                                           | Snowpark Python Connection
                                           v
+-----------------------------------------------------------------------------------+
| SNOWFLAKE CLOUD DATA PLATFORM (Customer Account)                                   |
|                                                                                   |
|  Dedicated Database: PI_GOVERN                                                     |
|  Schemas (8):                                                                     |
|   - APP_CORE           : Sync state, discovery caches, sync stored procedures     |
|   - RBAC               : Application roles, permissions, user role assignments     |
|   - METADATA           : Data assets, asset columns, V_ASSET_GOVERNANCE view      |
|   - LINEAGE            : Lineage nodes, lineage edges (dependency graph)          |
|   - POLICIES           : Governance policy definitions, asset evaluation states   |
|   - GOVERNANCE_EVENTS  : Issue tracking, risk findings, governance alerts          |
|   - AUDIT              : Mutation audit logs with user/actor attribution          |
|   - AI_ASSISTANT       : Chat conversations and contextual message history        |
|                                                                                   |
|  Native Snowflake Compute & Services:                                             |
|   - Stored Procedures  : APP_CORE.SYNC_METADATA_DATABASE, SYNC_LINEAGE_DATABASE   |
|   - Native Inference   : SNOWFLAKE.CORTEX.COMPLETE                                |
|   - System Views (Read): <DB>.INFORMATION_SCHEMA, ACCOUNT_USAGE.OBJECT_DEPENDENCIES|
+-----------------------------------------------------------------------------------+
```

---

## 3. Conceptual Layering & Data Pipeline

PI-GOVERN operates across three conceptual layers:

```
+-------------------------------------------------------------------------+
|                          INTELLIGENCE LAYER                             |
|  Discovery Engine -> Metadata Scanner -> Dependency Graph Analyzer      |
|  Risk & Drift Detection -> Cortex Context Assembler                    |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
|                            CONTROL LAYER                                |
|  In-App RBAC Engine -> Policy Definition & Matching Evaluator           |
|  Ownership & Stewardship Registry -> Governance State Machine           |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
|                           EXECUTION LAYER                               |
|  Idempotent Provisioner (DDL) -> Procedure Merge Engine                 |
|  Audit Logger -> (Future: Native Masking & Row Access DDL Compiler)    |
+-------------------------------------------------------------------------+
```

### 6-Stage Governance Pipeline
1. **Data Sources**: Monitored Snowflake databases, schemas, tables, and views.
2. **Metadata Intelligence Engine**: Scheduled extraction of structural definitions, data types, row counts, and storage volumes.
3. **Governance Knowledge Graph**: Construction of object dependency graphs (lineage nodes and edges) across database boundaries.
4. **Policy + RBAC Orchestration**: Evaluation of governance policies against asset attributes; enforcement of operational roles.
5. **Lineage + Observability Engine**: Visual representation of upstream/downstream data flows and impact analysis.
6. **Automation + Risk Intelligence**: Event detection, Cortex AI contextual reasoning, and compliance auditing.

---

## 4. Technology Stack Specification

### Frontend Architecture
- **Framework**: Next.js 15 (App Router architecture, React Server Components where applicable)
- **UI Library**: React 19, TypeScript 5.7
- **Styling**: Tailwind CSS 3.4, PostCSS, Class Variance Authority (CVA), clsx, tailwind-merge
- **Component Primitives**: Radix UI Primitives, shadcn/ui design tokens
- **Client State**: Zustand 5 (modular stores with selective persistence)
  - `auth-store`: Active user identity, session state, environment parameters (persisted)
  - `sidebar-store`: Expanded sidebar vs. collapsed rail state (persisted)
  - `preferences-store`: Theme, display density, default filters (persisted)
  - `workspace-store`: Current database, discovery inventory, active asset selection
  - `ui-store`: Modals, drawer states, command palette visibility
- **Server Cache & Queries**: TanStack Query 5 (React Query) with optimistic updates and cache invalidation
- **Visualization & Graphs**: Recharts (metric trends), `@xyflow/react` (interactive lineage DAG with layered layout)
- **Tables & Virtualization**: `@tanstack/react-table`, `@tanstack/react-virtual` for enterprise data density
- **Form Handling**: React Hook Form, Zod validation schemas

### Backend Architecture
- **Framework**: FastAPI 0.115+ (ASGI framework with async route handlers)
- **Language Runtime**: Python 3.11+
- **Database Driver**: `snowflake-snowpark-python` 1.53+ (sole database connector)
- **Schema Validation & Settings**: Pydantic 2, Pydantic Settings
- **Security & Rate Limiting**: SlowAPI, Starlette Security headers, Cryptographic token generation (`secrets.token_urlsafe`)
- **Observability & Logging**: Structlog (JSON structured logs), Sentry SDK (`send_default_pii=False`)
- **Serialization**: orjson for high-throughput JSON processing
- **Resilience**: Tenacity retry mechanics on network connection drops

### Snowflake Cloud Native Components
- **Runtime Footprint**: Database `PI_GOVERN`
- **Warehouse**: Configurable virtual warehouse (X-Small is sufficient for standard catalog operations)
- **Inference Service**: Snowflake Cortex (`cortex-complete` using models such as `mistral-large`, `claude-3-5-sonnet`, or `llama3`)

---

## 5. Snowflake Database Schema & Footprint

The platform provisions and manages eight schemas in the `PI_GOVERN` database:

### 1. `APP_CORE` (Infrastructure & Orchestration)
- `BOOTSTRAP_SYNC_STATUS`: Tracks per-database synchronization progress, asset counts, errors, and timestamps.
- `BOOTSTRAP_DISCOVERY_CACHE`: Caches discovered databases per Snowflake user and role (5-minute TTL).
- `BOOTSTRAP_INIT_CACHE`: Caches database readiness validation states (2-minute TTL for positive checks).
- `SYNC_METADATA_DATABASE(DB_NAME STRING)`: Stored procedure. Reads `<DB>.INFORMATION_SCHEMA.TABLES` and `COLUMNS`, executing a safe `MERGE` into `METADATA.DATA_ASSETS` and `METADATA.ASSET_COLUMNS`. Preserves steward classifications and custom descriptions.
- `SYNC_LINEAGE_DATABASE(DB_NAME STRING)`: Stored procedure. Reads `SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES`, extracting unique source and target entities into `LINEAGE.LINEAGE_NODES` and mapping dependency connections into `LINEAGE.LINEAGE_EDGES`.

### 2. `RBAC` (Application Access Control)
- `ROLES`: Internal roles (`admin`, `viewer`, custom roles).
- `PERMISSIONS`: Granular resource-action combinations (e.g. `metadata:read`, `policies:write`, `*:*`).
- `ROLE_ASSIGNMENTS`: Maps Snowflake identities (`username`) to internal PI-GOVERN roles.

### 3. `METADATA` (Governance Catalog)
- `DATA_ASSETS`: Core catalog table. Stores tables, views, materialized views, and stages. Tracks FQN, source database/schema, row count, byte size, owner, steward, quality score, sensitivity, and governance lifecycle state.
- `ASSET_COLUMNS`: Column definitions. Foreign-keyed to `DATA_ASSETS`. Tracks data type, steward classification (PII, SENSITIVE, FINANCIAL), masking status, masking rules, and business descriptions.
- `V_ASSET_GOVERNANCE`: Analytical rollup view. Pre-aggregates column count, PII column count, sensitive column count, and policy attachment count per asset. Also acts as the database health smoke test.

### 4. `LINEAGE` (Data Dependency Graph)
- `LINEAGE_NODES`: Distinct data entities participating in lineage relationships (qualified object name, node type, source database, health status).
- `LINEAGE_EDGES`: Directed dependencies linking source node ID to target node ID with transformation details.

### 5. `POLICIES` (Governance Rules & Evaluation)
- `POLICIES`: Policy definitions containing name, description, status (`draft`, `active`), policy type, rule definitions (JSON variant), and target criteria (`applies_to` JSON variant).
- `POLICY_EVALUATION_STATE`: Evaluation outcomes mapping policy ID to asset ID. Tracks status (`compliant`, `violation`), evaluation timestamp, and evaluation match details.

### 6. `GOVERNANCE_EVENTS` (Alerts & Risk Tracking)
- `GOVERNANCE_EVENTS`: Governance findings, audit exceptions, and drift alerts. Tracks severity (`info`, `warning`, `critical`, `high`), status (`open`, `resolved`), affected resource ID, and resolution timestamps.

### 7. `AUDIT` (Compliance Mutation Log)
- `AUDIT_LOGS`: Append-only compliance log. Records action name, resource type, resource ID, actor identity, timestamp, client IP, and contextual metadata.

### 8. `AI_ASSISTANT` (Conversation History)
- `CONVERSATIONS`: Chat sessions bound to individual user identities.
- `MESSAGES`: Message logs within conversations, storing user inputs, assistant responses, and contextual metadata.

---

## 6. Mirror-and-Govern Data Flow

PI-GOVERN adheres strictly to the mirror-and-govern architecture:

```
[ Snowflake Raw Objects ]
         |
         | (Periodic / On-demand Sync)
         v
[ APP_CORE Stored Procedures ]
   - MERGE TABLES & COLUMNS
   - MERGE OBJECT_DEPENDENCIES
         |
         | (Writes System Metrics, Retains Steward Edits)
         v
[ PI_GOVERN Catalog Mirror ]
   - METADATA.DATA_ASSETS
   - METADATA.ASSET_COLUMNS
   - LINEAGE.LINEAGE_NODES / EDGES
         |
         | (Fast, Indexed Queries / Aggregations)
         v
[ FastAPI Repository Layer ]
         |
         | (REST APIs via JSON)
         v
[ Frontend Dashboard & Workspaces ]
```

### Ingestion Semantics
- **Non-Destructive Synchronization**: The `MERGE` procedure updates dynamic metrics (`row_count`, `size_bytes`, `data_type`), but explicitly preserves human steward additions (`classification`, `masked`, `masking_rule`, custom `description`).
- **Read Path Isolation**: Under no circumstances does a user dashboard render execute a query against `INFORMATION_SCHEMA` or `ACCOUNT_USAGE`.
- **Latency Decoupling**: Lineage data reflects the refresh latency of `SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES` (typically 1 to 3 hours). The UI displays last-synced timestamps to maintain audit transparency.

---

## 7. Authentication & Session Architecture

PI-GOVERN supports two mutually exclusive modes governed by the `AUTH_MODE` configuration:

```
                                  AUTH_MODE Selection
                                           |
                  +------------------------+------------------------+
                  |                                                 |
                  v                                                 v
        [ 'login' Mode ]                                  [ 'service' Mode ]
  - User submits credentials                        - Single process connection
  - Backend opens per-user Snowpark Session         - SPCS Ingress token or static creds
  - Direct Snowflake authentication                 - Identity inferred from HTTP header
  - Session stored in SessionStore                  - Shared SnowflakeSessionManager
  - Mints 32-byte opaque URL-safe token             - Global warehouse / role context
  - Token in Cookie & Response Body                 - Centralized service execution
```

### Session Token Lifecycle (Login Mode)
1. **Authentication**: `POST /api/v1/auth/login` accepts `{account, username, password, role, warehouse}`.
2. **Snowflake Validation**: Snowpark establishes a live session. If Snowflake rejects credentials, authentication fails immediately.
3. **Pre-flight Health**: The system runs `check_schema_init` to verify `PI_GOVERN` database readiness.
4. **Token Generation**: Backend generates a high-entropy 32-byte token via `secrets.token_urlsafe(32)`.
5. **Session Registration**: Token maps to `(SnowparkSessionManager, UserIdentity, last_activity)` in `core/session_store.py`.
6. **Transport**: Returned via HTTP-only cookie and JSON response body (to support cross-site setups where cookies are blocked).
7. **Execution Boundary**: Every subsequent request bearing `Authorization: Bearer <token>` executes SQL queries directly through that user's Snowpark session.

---

## 8. Security, Authorization & Audit Subsystems

### In-App RBAC vs. Snowflake Native RBAC
- **In-App RBAC**: PI-GOVERN manages functional application permissions (`metadata:read`, `policies:write`, `audit:read`) in `RBAC.PERMISSIONS`. Internal roles are evaluated by FastAPI dependencies (`require_permission`) and cached for 30 seconds using a `WeakKeyDictionary` keyed by the session manager.
- **Snowflake Native RBAC**: In login mode, all SQL queries executed against Snowflake run with the privileges of the caller's Snowflake role. Privilege escalation is structurally impossible because unauthorized database objects cannot be accessed by the Snowpark session.

### Authorization Pipeline
Every request must execute through the strict pipeline:
`Authentication -> Identity Verification -> In-App Permission Check -> Snowflake Object Privilege -> Data Access`

### Audit Middleware
- Installed as outer ASGI middleware on FastAPI.
- Intercepts all successful mutating HTTP requests (`POST`, `PUT`, `PATCH`, `DELETE`) with status code < 400.
- Skips non-governance routes (`/audit`, `/health`, `/docs`).
- Extracts actor identity directly from the authenticated session context.
- Writes an immutable audit row into `AUDIT.AUDIT_LOGS` on the caller's session, ensuring identical attribution in Snowflake's internal `QUERY_HISTORY`.

---

## 9. AI Assistant Architecture (Snowflake Cortex)

```
[ User Prompt + Selected Asset FQN ]
                 |
                 v
[ backend/app/modules/ai_assistant/context.py ]
   Concurrent Async Assembly (asyncio.gather):
   1. Discovered Databases Summary
   2. Asset & Schema Drift Counts
   3. 25 Sample Data Assets (Filtered by Permissions)
   4. 10 Active Policies
   5. 10 Open High/Critical Governance Events
   6. Lineage Graph Topological Counts
                 |
                 v
[ Synthesized Context Envelope & System Prompt ]
                 |
                 v
[ Snowpark Session SQL: SELECT SNOWFLAKE.CORTEX.COMPLETE(?, ?) ]
                 |
                 v
[ Cortex LLM Inference (Inside Snowflake Cloud) ]
                 |
                 v
[ Word-by-Word Streamed Response to Frontend ]
```

- **Advisory Only**: The AI assistant is read-only. It has no write permissions and cannot execute DDL or modify database tables.
- **Data Perimeter**: Prompts, table names, and metadata are submitted directly to Cortex inside the customer Snowflake account. No data is sent to OpenAI, Anthropic, or external API endpoints.

---

## 10. Frontend Application Architecture

### Viewport & Master-Detail Design
- **Non-Scrolling Primary Shell**: The main viewport (`h-screen overflow-hidden`) avoids global page scrollbars. High-density interfaces fit the complete operational workspace inside the screen.
- **Master-Detail Layout**: Used across Metadata Intelligence and Policy modules:
  - Left Panel: High-density, searchable, filterable Asset Explorer.
  - Right Panel: Asset Intelligence workspace featuring the 6 canonical governance tabs (Overview, Access History, Governance, Metadata, Change History, AI Assistant).

### Provider & Gate Sequence
```
RootLayout
  └─ ThemeProvider (Light theme default)
      └─ QueryClientProvider (TanStack Query)
          └─ WorkspaceProvider (Global orchestrator)
              └─ DashboardAccessGate (Auth verification)
                  └─ InitGate (Validates PI_GOVERN schema existence)
                      └─ SyncGate (Ensures initial sync has run)
                          └─ AppShell (Sidebar, Header, Main Content)
```

---

## 11. Deployment Topologies

PI-GOVERN supports three production deployment topologies:

### 1. Standalone Container / VM
- Frontend built as standalone Next.js container or static export.
- FastAPI backend deployed to container runtime (Docker, ECS, Kubernetes, Render).
- Connects to Snowflake over HTTPS via credentials or key-pair authentication.

### 2. Snowpark Container Services (SPCS)
- FastAPI and Next.js containers run directly inside Snowflake compute pools.
- Authentication uses SPCS native ingress OAuth token files (`/snowflake/session/token`).
- Lowest network latency; zero external ingress required.

### 3. Snowflake Native App
- Packaged as a Snowflake Native App with a `manifest.yml` and `setup_script.sql`.
- Runs completely inside the consumer account under consumer authorization.
- Uses `CONFIG.REGISTER_GOVERNED_DATABASE` to grant access to consumer data.

---

## 12. Known Architectural Constraints & Technical Debt

1. **In-Memory Session Store**: `core/session_store.py` stores Snowpark sessions in local process memory. Horizontal scaling across multiple container replicas requires sticky sessions until distributed session coordination (PEND-005) is implemented.
2. **DDL Triplication**: Schema DDL is maintained in three locations: `bootstrap_ddl.py`, `deploy/standalone/setup.sql`, and `native-app/setup_script.sql`. Changes must be manually synchronized.
3. **Manual Governance Events**: `GOVERNANCE_EVENTS` rows are currently written via REST API calls rather than automatically emitted by background drift detection daemons (PEND-002).
4. **Policy Enforcement Boundary**: Policy evaluation runs in Python and writes compliance state, but does not compile or push native Snowflake masking or row-level security DDL (PEND-004).
