---
title: Pi-Govern System Understanding
type: project-doc
created: 2026-09-16
updated: 2026-09-16
tags:
  - pi-govern
  - snowflake
  - fastapi
  - nextjs
  - data-governance
  - project-notes
---

# Pi-Govern — System Understanding

> [!info] Master Context Document
> Single source of truth for the Pi-Govern architecture audit. Snowflake-native data governance control plane: discovers a Snowflake account, mirrors it into its own catalog, and layers metadata, lineage, policy, RBAC, events, audit and an LLM assistant on top — all inside the customer's own Snowflake account.

## 1. One-Page Summary

**Purpose.** A [[Snowflake]]-native governance control plane. Discovers an account's data estate, persists a governance catalog inside a dedicated `PI_GOVERN` database in that same account, and layers metadata, lineage, policy, RBAC, events, audit and an LLM assistant on top. Nothing leaves Snowflake — catalog, audit log, AI conversations, and even LLM inference (`SNOWFLAKE.CORTEX.COMPLETE`) all live in the customer's account.

**Core architecture.** [[Next.js]] 15 / React 19 SPA-style app → axios → [[FastAPI]] (Python) → Snowpark session → Snowflake. No second datastore — no Postgres, no Redis, no ORM, no migrations framework. Every table PI-GOVERN owns is a Snowflake table in `PI_GOVERN.<SCHEMA>`.

**Modules.** 8 backend/frontend module pairs — metadata, lineage, rbac, policies, governance_events, audit, ai_assistant, dashboard — plus 3 cross-cutting services: auth, bootstrap (discovery + sync), bootstrap_init (environment provisioning).

**Snowflake footprint.** 8 schemas, 16 tables, 1 view, 2 stored procedures, 1 optional task. Reads `INFORMATION_SCHEMA` (metadata sync), `SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES` (lineage sync), `SHOW DATABASES` (discovery), `SNOWFLAKE.CORTEX.COMPLETE` (AI).

**Authentication.** Two mutually exclusive modes via `AUTH_MODE`: `service` (one process-wide Snowflake connection, identity from SPCS ingress header) and `login` (each user signs in with their own Snowflake credentials; backend opens a real Snowpark session per user). No password store, no JWT, no hashing anywhere (ADR-015).

**Data flow.** Snowflake system views → stored procedure `MERGE` → PI-GOVERN tables → repository → service → REST → react-query → component. Every module read is served from PI-GOVERN's own persisted tables, never live from Snowflake system views. Live reads happen only during sync and AI context assembly.

**Governance flow.** Sync populates the catalog → stewards classify columns and author policies → policy engine evaluates rules against asset attributes and writes `POLICY_EVALUATION_STATE` → failures plus critical/high governance events become score penalties → dashboard renders the score.

### Top 5 Known Risks

| # | Risk | Severity | Status |
|---|---|---|---|
| 1 | `session_store` holds live Snowpark sessions in process memory. Multi-replica deployment silently breaks auth without sticky sessions; a restart logs everyone out. | HIGH | CONFIRMED |
| 2 | Governance score anchored to hardcoded base 92, not 100, fallback 86. A perfectly governed account can never score above 92. Docstring says "100 - penalties"; code says 92. | HIGH (product credibility) | CONFIRMED |
| 3 | `/dashboard/trend`, `/dashboard/context`, all 7 `/ai/*` endpoints authenticated but not permission-checked. | HIGH | CONFIRMED |
| 4 | Discovery probes one `SELECT 1` per candidate DB; init check unions 4 `INFORMATION_SCHEMA` views. Caches only 5min/2min; `queue_database_syncs` re-probes uncached every sync. | MEDIUM-HIGH | CONFIRMED |
| 5 | Frontend persists auth to a single un-namespaced `localStorage` key; reset depends on `user?.id` only. Same username across two accounts can carry state across environments. | MEDIUM | CONFIRMED |

### Top 5 Known Gaps

| # | Gap | Class |
|---|---|---|
| 1 | RBAC module governs PI-GOVERN's own roles only — never reads Snowflake's native RBAC (no `SHOW GRANTS`, no `GRANTS_TO_ROLES`). | PRODUCT |
| 2 | Policies are metadata-only. Nothing creates a Snowflake `MASKING POLICY` or `ROW ACCESS POLICY`. "Governance execution" = evaluation/reporting, not enforcement. | PRODUCT |
| 3 | Governance events are never auto-generated — every row arrives via `POST /governance-events`. Score is driven by a table nobody writes automatically. | PRODUCT/DATA |
| 4 | `AUTHENTICATION_FAILED`, `PARTIAL_SETUP`, `REPAIR_REQUIRED` not modelled states — `GovernanceStatus` has exactly 3 values. | TECHNICAL/UX |
| 5 | Zero integration/component/E2E tests. Governance score formula itself untested. | TESTING |

---

## 2. Mental Model (Narrative)

PI-GOVERN installs itself into a Snowflake account: provisions `PI_GOVERN` with 8 schemas, mirrors the account's data estate into its own catalog tables, and governs from there. **The read path never touches Snowflake system views — it reads PI-GOVERN's mirror.** This is the single most important architectural fact: mirror-and-govern, not live-query. Freshness depends entirely on sync; lineage inherits `ACCOUNT_USAGE`'s ~3h latency.

**Login** — `POST /api/v1/auth/login` takes `{account, username, password, role?, warehouse?}`, opens a real Snowpark session with those credentials. Success = Snowflake accepted them. Backend runs `check_schema_init` + `identity_during_setup`, mints a 32-byte `secrets.token_urlsafe` token, stores token → (live session, identity) in an in-process dict, sets an httponly cookie. The token is *also* returned in the response body because the intended deployment (GitHub Pages → Render) is cross-site and the cookie is third-party-blocked. Frontend stores it in `localStorage["pi-govern-token"]`, sends `Authorization: Bearer`. Password never stored/hashed/logged.

**Snowflake connection** — one `SnowparkSessionManager` wraps one `Session`, blocking calls via `run_in_threadpool`. Three connection shapes: `spcs` (OAuth token file), `credentials` (account/user/role + password/key-pair), `auto`. One session per login (login mode, up to `SESSION_IDLE_MINUTES`=60) or one process-wide session (service mode). No connection pool.

**Setup** — `POST /bootstrap/init/run` takes a per-`(account, database)` lock, re-checks readiness, invalidates cache, walks stages in order: schemas → tables → procedures+view → seed_rbac → validation. Snapshots object inventory first, issues DDL only for absent objects. All DDL is `CREATE ... IF NOT EXISTS`; seeds are `INSERT ... WHERE NOT EXISTS`; procedures are `EXECUTE AS CALLER`. No `DROP`, no `TRUNCATE`, no `CREATE OR REPLACE` on the in-app path. `bootstrap_ddl.py` has a `_self_check()` asserting these invariants.

**Readiness** is determined by asking Snowflake what exists — not a flag, not a version table. `object_inventory()` unions `SCHEMATA/TABLES/VIEWS/PROCEDURES`, compares against the DDL manifest, confirms RBAC seed rows, smoke-tests `METADATA.V_ASSET_GOVERNANCE`. Positive results cached 2 min in `BOOTSTRAP_INIT_CACHE`; negative results are **never** cached.

**Metadata** comes from `CALL APP_CORE.SYNC_METADATA_DATABASE(?)` → `MERGE` from `<DB>.INFORMATION_SCHEMA.{TABLES,COLUMNS}` into `METADATA.{DATA_ASSETS,ASSET_COLUMNS}`. The `MERGE` refreshes type/row_count/size/description but **never overwrites** `classification`, `masked`, `masking_rule` — steward work survives re-sync. Classification is 100% human-entered; nothing auto-classifies PII.

**Lineage** comes from `CALL APP_CORE.SYNC_LINEAGE_DATABASE(?)` reading `SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES`, object-level only (no column lineage), rendered via `@xyflow/react` with a custom layered layout.

**RBAC** — two separate concepts: (a) the real Snowflake role the session runs as (walked via `SHOW GRANTS TO ROLE`, BFS capped at 100 roles, but *only* for setup pre-flight privilege checks); (b) PI-GOVERN's own `RBAC.ROLES/PERMISSIONS/ROLE_ASSIGNMENTS` mapping a Snowflake identity to an in-app role, cached 30s. The `/rbac` UI shows only (b).

**Policies** — a rule is `{match: all|any, conditions:[{field,operator,value}]}`, evaluated by a pure `engine.py` (8 operators, fails closed on unknown/type-mismatch). `POST /policies/{id}/evaluate` resolves targets (or all assets, capped at 500), upserts `POLICY_EVALUATION_STATE`. Nothing talks to Snowflake's policy DDL.

**Audit** — `audit_log_middleware` writes one row per successful (< 400) mutating request, excluding `/audit` and `/health`. Actor resolved from session store or ingress header; unresolvable actor = row skipped, never attributed to nobody. In login mode the row is written on the acting user's own session, so Snowflake `QUERY_HISTORY` attributes it correctly too.

**Governance score** — one SQL statement per request: `GENERATOR(ROWCOUNT=>7)` for last 7 days, joined against critical/high event counts and failed-evaluation counts per day. `penalty = events*3 + violations*5`; `score = max(0, min(100, 92 - penalty))`. Base 92, fallback 86, weights 3/5 — all hardcoded literals. Metadata/classification/lineage/quality coverage contribute nothing.

**AI assistant** — `context.py` runs 6 concurrent, individually fault-tolerant queries (databases, asset+drift counts, 25 sample assets, 10 recent policies, 10 open events, lineage counts) into a system prompt; last 20 messages flattened to text; `SELECT SNOWFLAKE.CORTEX.COMPLETE(?, ?)`. Read-only advisory — cannot execute governance actions. "Streaming" is simulated: full reply computed first, then replayed word-by-word.

**Dashboard** — `GET /dashboard/context` + `/dashboard/trend`, assembled server-side with `asyncio.gather`. 4 metric cards, 7-day posture chart, priority-events list, recent-activity list, policy-status breakdown. Every card is read-only; no drill-through routes.

---

## 3. Technology Stack

**Frontend:** Next.js 15 (App Router) · React 19 · TypeScript 5.7 · Tailwind CSS 3.4 · Radix UI + shadcn/ui · Zustand 5 (6 stores, 3 persisted) · TanStack Query 5 · axios · Recharts · `@xyflow/react` (lineage graph) · TanStack Table/Virtual · Zod · react-hook-form · sonner · nuqs · date-fns · next-themes. Test: vitest, Testing Library, Playwright installed but **zero specs**.

**Backend:** FastAPI 0.115 · uvicorn · `snowflake-snowpark-python` 1.53 (the only DB driver) · Pydantic 2 · slowapi (rate limiting) · structlog · sentry-sdk (`send_default_pii=False`) · orjson · tenacity. Test: pytest, pytest-asyncio, factory-boy, faker. Quality: ruff, mypy.

**Absent by design:** no SQLAlchemy/Alembic, no Postgres/Redis/Celery, no Anthropic/OpenAI SDK (Cortex instead, per ADR-014), no passlib/python-jose (Snowflake session identity instead, per ADR-015).

---

## 4. Repository Layout

```
backend/app/
  main.py            App factory, CORS, GZip, CSP, request-id, audit mw
  api/v1/             auth.py, bootstrap.py, health.py (+ router.py mounts 11 routers)
  core/               config, dependencies (DI), session_store, identity,
                      permissions, audit_middleware, exceptions, logging, limiter
  db/snowflake_base.py  THE generic parameterized repository
  services/           snowflake.py (session mgr), bootstrap.py (discovery+sync),
                      bootstrap_init.py (readiness), bootstrap_ddl.py (DDL manifest), cortex.py
  modules/<8>/        routes.py, services.py, repositories.py, schemas.py

frontend/src/
  app/                (auth)/login, (dashboard)/<9 routes>, (workspace) [passthrough]
  components/         auth/, bootstrap/ (4 gate components), layout/, navigation/, shell/, ui/
  modules/<8>/        components/, hooks/, services/, schemas/, types/, utils/
  providers/          workspace-provider (the orchestrator), query, theme, sidebar, toast
  store/              6 Zustand stores

deploy/standalone/setup.sql   Full account provisioning (545 lines)
deploy/spcs/                  01_infra.sql, 02_service.sql, deploy.sh, teardown.sh
native-app/                   manifest.yml, setup_script.sql (471 lines), service-spec.yaml
```

Layering rule: routes = HTTP + auth + status codes; services = business logic; repositories = SQL; `db/snowflake_base` = generic parameterized CRUD shared by all 8 modules. No module reimplements list/get/create/update/delete.

---

## 5. Application Architecture

```
Browser → Next.js (middleware.ts = NO-OP)
  → Provider stack: Theme > QueryClient > WorkspaceProvider > Sidebar
  → (dashboard)/layout gate chain: DashboardAccessGate > InitGate > SyncGate > AppShell
  → 8 feature modules / Zustand / react-query / axios (Bearer + 401 interceptor)
→ HTTPS /api/v1/* → FastAPI: SlowAPI > CORS > GZip > security+request-id mw > audit mw
  → DI: get_request_context() → (login: session_store) | (service: header/creds)
  → 8 module routers + auth/bootstrap + require_permission + health
  → repositories.py → SnowflakeTableRepository (parameterized ? binds)
  → SnowparkSessionManager → run_in_threadpool(session.sql())
→ Snowflake account: PI_GOVERN database (16 tables, 1 view, 2 procedures)
  + read-only surfaces: INFORMATION_SCHEMA, ACCOUNT_USAGE.OBJECT_DEPENDENCIES,
    SHOW DATABASES/GRANTS, SNOWFLAKE.CORTEX.COMPLETE
```

`WorkspaceProvider` mounting (not middleware.ts) triggers the entire readiness sequence, before any page renders. The AI assistant is a normal module behind the same API — not a parallel component. There is no separate "Governance Engine"; the policy engine is a pure function inside the policies module.

---

## 6. Frontend Architecture

**Component classes:** GLOBAL (AppShell, sidebar, header, status bar, command palette), GATES (DashboardAccessGate, InitGate, SyncGate, EnvironmentSetupCard, CubeLoader), SHARED (ModuleScaffold used by 5 modules, DataTable, ChartCard, Timeline), PAGE-SPECIFIC (one workspace per module), DUPLICATED (3 separate metric-card implementations; 2 status-pill implementations), UNUSED/DEAD (dashboard's own store/services/types — dashboard actually uses top-level `services/dashboard.service.ts`; rbac's own store; `lib/validations.ts` `loginSchema` that doesn't match the real form).

**Stores:** `auth-store` (persisted), `sidebar-store` (persisted), `preferences-store` (persisted), `workspace-store` (not persisted), `ui-store`. Plus `localStorage["pi-govern-token"/"pi-govern-theme"]`, `sessionStorage["pi-govern-session-resolved"/"pi-govern-login-approved"]`.

Every route group has `loading.tsx` + `error.tsx`. No fallback to demo data on error — demo mode is a build-time flag only.

---

## 7. Backend Architecture

```
FastAPI app
├ middleware: SlowAPI → CORS → GZip → security+req-id → audit_log_middleware (outermost, registered post-create_app, sees final response status)
├ get_request_context(): login → session_store (per-user SnowparkSessionManager)
│                         service → header/creds/dev (lru_cache get_snowpark_manager)
├ core/identity, core/permissions (require_permission/require_role) → RbacRepository
│   (30s WeakKeyDictionary cache keyed by session manager)
├ auth router → bootstrap_init (check/run_schema_init, setup_script, identity_during_setup)
├ bootstrap router → bootstrap service (discover, cache, sync_statuses, queue_database_syncs)
│   → metadata/sync + lineage/sync
├ 8 module routers → services → repositories → db/snowflake_base
├ ai router → ai_assistant/services → context.py (6 concurrent reads) → services/cortex
└ dashboard router → services (asyncio.gather×4) → delegates to metadata/policies/events repos
```

---

## 8. Snowflake Object Inventory (Summary)

8 schemas: `APP_CORE, RBAC, METADATA, LINEAGE, POLICIES, GOVERNANCE_EVENTS, AUDIT, AI_ASSISTANT` (+ `CONFIG` native-app only).

| Table/View/Proc | Schema | Purpose |
|---|---|---|
| `BOOTSTRAP_SYNC_STATUS` | APP_CORE | Per-DB sync state + counts, PK `database_name` |
| `BOOTSTRAP_DISCOVERY_CACHE` | APP_CORE | 5-min discovery cache, PK `(user, role)` |
| `BOOTSTRAP_INIT_CACHE` | APP_CORE | 2-min readiness cache, PK `(account, database, role)` |
| `SYNC_METADATA_DATABASE(STRING)` | APP_CORE | Procedure, `EXECUTE AS CALLER` |
| `SYNC_LINEAGE_DATABASE(STRING)` | APP_CORE | Procedure, `EXECUTE AS CALLER` |
| `ROLES / PERMISSIONS / ROLE_ASSIGNMENTS` | RBAC | In-app RBAC |
| `DATA_ASSETS / ASSET_COLUMNS` | METADATA | Catalog |
| `V_ASSET_GOVERNANCE` | METADATA | Rollup view — also the readiness smoke test |
| `LINEAGE_NODES / LINEAGE_EDGES` | LINEAGE | Object-level lineage |
| `POLICIES / POLICY_EVALUATION_STATE` | POLICIES | Definitions + per-asset results |
| `GOVERNANCE_EVENTS` | GOVERNANCE_EVENTS | Manually-entered events only |
| `AUDIT_LOGS` | AUDIT | Mutation audit |
| `CONVERSATIONS / MESSAGES` | AI_ASSISTANT | Chat history |

Native-app-only additions: `SYNC_ALL_GOVERNED_DATABASES()`, `SYNC_GOVERNED_DATABASES_TASK` (cron `0 */6 * * *`), `PI_GOVERN_SERVICE` (SPCS), `PI_GOVERN_EVENTS` (event table), `CONFIG.REGISTER_GOVERNED_DATABASE`.

**No STAGE and no STREAM anywhere.**

⚠️ **DDL is triplicated** across `bootstrap_ddl.py`, `deploy/standalone/setup.sql`, and `native-app/setup_script.sql` (+ upgrade scripts) — a hand-copied snapshot, not a generated artifact. 4 known divergences already exist (e.g. `setup.sql` still has 2 bugs `bootstrap_ddl.py` documents fixing). Largest maintainability risk in the codebase.

---

## 9. Security Findings

**Strengths:** no password/JWT storage anywhere (identity = Snowflake session, ADR-015); in login mode every query runs on the caller's own session (privilege escalation structurally impossible); all provisioned procedures `EXECUTE AS CALLER`; every SQL value is a `?` bind, identifiers regex-validated in Python *and* re-validated in-procedure; security headers + CSP on every response; docs disabled in production; rate limiting on login (10/min) and AI (30/min); Sentry with `send_default_pii=False`.

**Confirmed issues:**
- **S1 (HIGH):** 9 endpoints authenticated but not permission-checked — dashboard ×2, AI ×7. AI context returns 25 asset FQNs + 10 policy names + 10 event titles to any authenticated identity regardless of grants.
- **S2 (HIGH):** Session token in `localStorage`, readable by XSS; CSP weakened by `unsafe-inline unsafe-eval`. Forced by the cross-site Pages→Render deployment shape.
- **S3 (HIGH):** Live Snowpark sessions in a process-local dict — no shared store, no horizontal scaling, restart logs everyone out.
- **S4 (MEDIUM):** `SessionStore.sweep()` exists, is tested, but is **never called** — no lifespan hook invokes it. Idle sessions accumulate for process lifetime.
- **S5 (MEDIUM):** `middleware.ts` is a no-op — all route protection is client-side.
- **S6 (MEDIUM):** `ALLOW_FIRST_ADMIN_BOOTSTRAP` defaults `True`; first identity to arrive on a fresh install becomes admin (mitigated by `user == session_user` check in login mode, but automatic in service-mode credentials connections).
- **S9/S10 (LOW):** repository f-string SQL interpolation relies on caller discipline (no reachable injection today, no enforcement); AI accepts a caller-supplied `system_prompt` (prompt-injection channel, bounded impact — no tools, no write path).

---

## 10. Data Safety & Idempotency (Strongest Area)

- Setup can run twice safely — `check_schema_init` short-circuits before any DDL on a complete environment.
- All in-app DDL is `IF NOT EXISTS`; seeds are `INSERT ... WHERE NOT EXISTS`; syncs are `MERGE`. No `DROP`/`TRUNCATE`/`DELETE` in any setup path.
- The metadata `MERGE` specifically protects human classification/masking edits from being overwritten by re-sync.
- Only destructive path in the whole system: `deploy/spcs/teardown.sh --all` (drops the entire `PI_GOVERN` database, requires an explicit flag).
- `bootstrap_ddl.py` has a `_self_check()` that machine-asserts these invariants — not just documented, enforced.

---

## 11. Governance Score — Exact Formula

```
score = max(0, min(100, 92 - (events_last7d_critical_or_high * 3 + failed_evaluations_last7d * 5)))
```

- Recomputed on every request; **no history table** — the "7-day trend" is 7 recomputations of today's penalty data bucketed by date, not stored daily snapshots.
- Fallback if no trend data: hardcoded `86`.
- Ignores metadata coverage, classification coverage, `policy_bound_count` (computed in the view but unused), lineage completeness, ownership, quality — despite `V_ASSET_GOVERNANCE` computing several of these aggregates already.
- On a freshly synced account with no manually-entered events/policies, the score is a constant 92 regardless of actual governance posture.
- Untested — no test covers this arithmetic.

---

## 12. Testing & Technical Debt Snapshot

**Tested well:** bootstrap discovery/caching, init-check edge cases, phase recovery, session isolation, RBAC cache, session-store lifecycle, generic repository encode/decode, AI context fault isolation, policy engine (all 8 operators). All backend tests run against a `FakeSnowparkManager` — **no test touches real Snowflake.**

**Untested:** any real Snowflake interaction, the governance score formula, MERGE steward-preservation semantics, full login→session→authorized-request E2E, RBAC assignment API, audit middleware, policy evaluation flow, any frontend component/page, all of E2E (Playwright installed, zero specs), Native App install/upgrade.

**Top technical debt:** DDL triplication (HIGH), process-memory session store with `sweep()` never called (HIGH), governance score's hardcoded base/fallback (HIGH), 9 unpermissioned endpoints (HIGH), token-in-localStorage + weak CSP (HIGH), policy evaluation N+1 up to 500 round trips (MEDIUM), dead schema columns across 6 tables suggesting unbuilt features (`owner`, `steward`, `quality_score`, `sensitivity`, `masked`, `masking_rule`, `extra_metadata` ×3) (MEDIUM), lineage edges never pruned (MEDIUM), 3 metric-card implementations / 2 status-pill implementations (MEDIUM).

---

## Related
- [[Snowflake]]
- [[FastAPI]]
- [[Next.js]]
- [[Harness vs Model]]
- [[wiki/log]]
