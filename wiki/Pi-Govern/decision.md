# PI-GOVERN Decision Log

## Purpose

This document is the single source of truth for all important PI-GOVERN product, architecture, engineering, security, UX, and implementation decisions.

---

## Decision Rules

- Record every meaningful project decision.
- Never silently overwrite an important historical decision.
- If a decision changes, mark the previous decision as superseded.
- Record why the decision changed and the superseding decision ID.
- Link related decisions where useful.
- Distinguish confirmed decisions (Status: Accepted) from proposals or roadmapped items (Status: Pending).
- Do not record transient implementation trivia that lacks long-term architectural significance.
- Before starting significant implementation work, read this decision log.
- When an accepted decision is modified or a new architectural choice is settled, update this log.

---

# Product Decisions

## PD-001: PI-GOVERN Product Positioning

Status: Accepted

Decision:
PI-GOVERN is a Snowflake-native governance platform, not merely a dashboard, a metadata catalog, or a passive reporting application. The product is positioned across three cohesive capabilities:
1. Governance Control Plane
2. Governance Intelligence Layer
3. Active Governance Execution Engine

Long-term positioning: PI-GOVERN is evolving toward a complete governance ecosystem and governance operating system around the Snowflake data estate. The product enables users to:
- Discover
- Understand
- Classify
- Govern
- Control
- Detect
- Decide
- Act
- Verify
- Audit

Rationale:
Treating governance as a static dashboard or simple catalog fails to address real operational risk. Enterprises need an active control plane that operates directly within their data warehouse, bridges the gap between observation and enforcement, and retains complete auditability without data exfiltration.

Implications:
- Every feature must align with active governance, intelligence, or control rather than passive display.
- UI and backend models must be structured to eventually support policy enforcement, automated event detection, and remediation workflows.
- Marketing and UX copy must reflect control-plane and intelligence capabilities, avoiding passive "viewer" or "catalog-only" framing.

Related: AD-001, AD-003, PD-002, PD-004

---

## PD-002: Metadata Intelligence Value Proposition (USP)

Status: Accepted

Decision:
The primary unique selling proposition (USP) for Metadata Intelligence is:
"PI-GOVERN turns Snowflake metadata into actionable governance intelligence."

PI-GOVERN does not compete with Snowflake by building another generic catalog UI. Differentiation comes from providing comprehensive governance context around every Snowflake asset, answering:
- What exists?
- Who owns it?
- Who can access it?
- What data does it contain?
- How sensitive is it?
- Where did it come from?
- Where does it go?
- What policies apply?
- What changed?
- What is risky?
- What should I do next?
- Can PI-GOVERN help me act?

Rationale:
Snowflake already provides basic object navigation and catalog metadata. PI-GOVERN adds enterprise value by synthesizing metadata, access posture, policy coverage, data lineage, risk signals, and historical changes into unified asset intelligence.

Implications:
- Metadata views must present governance context (sensitivity, policies, access, lineage, drift) prominently, not just column lists and data types.
- Empty states and feature layouts must reinforce governance decision-making.

Related: PD-001, MD-001, MD-002

---

## PD-003: Progressive Governance Object Model

Status: Accepted

Decision:
Every supported Snowflake object discovered by PI-GOVERN progressively becomes a canonical Governance Object. The supported object universe includes:
- Database
- Schema
- Table
- View
- Stage
- Column
- Other Snowflake objects as backend support is introduced

The user experience evolves from a traditional database tree hierarchy:
Database -> Schema -> Table -> Columns

into a rich, multi-dimensional governance entity:
Governance Object -> Metadata -> Classification -> Ownership -> Access -> Lineage -> Policies -> Governance -> Risk -> Change -> AI -> Actions -> Verification -> Audit.

Each Governance Object carries a canonical identity referenced consistently across all modules.

Rationale:
A flat or purely relational catalog treats tables as passive data containers. By establishing a canonical Governance Object abstraction, all governance dimensions (RBAC, lineage, policies, events, audit logs, AI conversations) bind to the same entity.

Implications:
- Object identifiers must be deterministic and cross-referenced across modules (e.g. fully qualified names and canonical IDs).
- Object detail views must provide tabs for every governance dimension rather than isolating them into disconnected pages.

Related: MD-001, MD-002, AD-003

---

## PD-004: Governance Action Framework (Observe to Act)

Status: Accepted

Decision:
The long-term metadata and governance experience progresses through five operational stages:
Observe -> Understand -> Decide -> Act -> Verify

Governance actions must be exposed in the UI only when the backend genuinely supports and authorizes them. Fake, stubbed, or cosmetic action buttons are strictly prohibited.

Rationale:
Exposing non-functional action controls destroys user trust in an enterprise governance platform. Users must be confident that every visible action is backed by real execution and verification logic.

Implications:
- Read-only states must be explicitly presented as read-only.
- Action buttons (e.g. apply mask, assign steward, trigger re-evaluation) must remain hidden or explicitly disabled with capability explanations until real execution paths exist.

Related: DATA-002, SEC-002, PEND-004

---

# Architecture Decisions

## AD-001: Snowflake-Native Backend

Status: Accepted

Decision:
Snowflake is the primary and only persistent backend data platform for PI-GOVERN. There is no dependency on PostgreSQL, MySQL, Redis, Celery, or external databases. All PI-GOVERN state lives inside the `PI_GOVERN` Snowflake database across dedicated schemas (`APP_CORE`, `RBAC`, `METADATA`, `LINEAGE`, `POLICIES`, `GOVERNANCE_EVENTS`, `AUDIT`, `AI_ASSISTANT`).

The architectural request flow is:
Frontend (Next.js 15) -> FastAPI (Python) -> Snowpark Session -> Snowflake

Rationale:
Customers choose Snowflake-native solutions because sensitive metadata, audit trails, and data governance policies remain entirely within their own Snowflake security and governance boundary. Eliminating external databases simplifies deployment, avoids data exfiltration concerns, and satisfies strict enterprise compliance requirements.

Implications:
- No SQLAlchemy, Alembic, or external ORM.
- Database operations use parameterized Snowpark SQL executions via `db/snowflake_base.py`.
- Application state must be persisted in Snowflake tables, not in local application memory or external caches.

Related: AD-002, SEC-001, OPS-001

---

## AD-002: Mirror-and-Govern Data Flow Architecture

Status: Accepted

Decision:
PI-GOVERN follows a "mirror-and-govern" architecture rather than a purely live-query model. Snowflake metadata is discovered and synchronized into PI-GOVERN catalog tables (`METADATA.DATA_ASSETS`, `METADATA.ASSET_COLUMNS`, `LINEAGE.LINEAGE_NODES`, etc.) via stored procedures and sync services.

All application and dashboard read paths must query the PI-GOVERN mirror tables, never Snowflake system views (`INFORMATION_SCHEMA`, `ACCOUNT_USAGE`) on live UI request paths. Metadata freshness and synchronization timestamps must be displayed honestly to the user.

Rationale:
Querying `ACCOUNT_USAGE` and `INFORMATION_SCHEMA` on live page renders introduces unacceptable latency, high warehouse credit consumption, and potential permission errors. Mirroring allows millisecond-level responsive UI renders and enables stewards to enrich metadata (classifications, descriptions, tags, policies) without risk of having user enrichment overwritten by sync routines.

Implications:
- Sync procedures (`SYNC_METADATA_DATABASE`, `SYNC_LINEAGE_DATABASE`) must use safe `MERGE` logic that updates system attributes (row count, bytes, data types) but preserves steward classifications, tags, and custom descriptions.
- UI must clearly communicate last-synced timestamps and sync status.

Related: AD-001, PERF-001, DATA-001

---

## AD-003: Core Layered Conceptual Architecture

Status: Accepted

Decision:
The conceptual system architecture comprises six pipeline stages grouped into three operational layers:

Conceptual Pipeline:
Data Sources
  -> Metadata Intelligence Engine
  -> Governance Knowledge Graph
  -> Policy + RBAC Orchestration
  -> Lineage + Observability Engine
  -> Automation + Risk Intelligence

Three Operational Layers:
1. Intelligence Layer: Discovery, metadata extraction, classification, lineage synthesis, risk scoring, AI context.
2. Control Layer: Policy authoring, evaluation engine, RBAC permission models, access boundaries, stewardship assignment.
3. Execution Layer: DDL deployment, policy enforcement, sync procedures, audit generation, automated remediation.

The pillars are interconnected through metadata, lineage, policies, RBAC, governance state, audit, AI, and verified actions.

Rationale:
Separating intelligence, control, and execution prevents tight coupling between analysis and destructive enforcement, enabling safe preview and governance workflows.

Implications:
- Modules must expose clear boundaries between observing an issue (intelligence), defining the rule (control), and applying the change (execution).

Related: PD-001, AD-002, MD-001

---

## AD-004: Structural Environment Isolation

Status: Accepted

Decision:
PI-GOVERN state is structurally isolated by Snowflake account and database boundary. An artificial `environment_id` column must not be introduced into PI-GOVERN tables when structural isolation (separate Snowflake account, separate warehouse, or dedicated `PI_GOVERN` database instance) already establishes the multi-tenant or multi-environment boundary.

Backend caches (discovery cache, init cache) and synchronization tasks must remain strictly scoped by account, database, and role. Frontend state must not leak between Snowflake accounts or sessions.

Rationale:
Adding artificial tenant/environment identifiers inside a single database creates accidental leakage risk, complicates query constraints, and contradicts Snowflake's native isolation model where separate accounts or databases represent discrete environments.

Implications:
- Frontend storage keys must be scoped to the authenticated account/identity or cleared upon logout/switching.
- Backend session managers and caches must key on `(account, database, user, role)` tuples.

Related: SEC-001, AD-006

---

## AD-005: Idempotent Resumable Bootstrap and Setup

Status: Accepted

Decision:
PI-GOVERN environment initialization must be fully idempotent, safe, and resumable. Already initialized environments must never undergo a multi-minute setup or loading sequence on application boot.

Readiness is determined by validating actual Snowflake object existence (`SCHEMATA`, `TABLES`, `VIEWS`, `PROCEDURES`, RBAC seeds, smoke-testing `METADATA.V_ASSET_GOVERNANCE`), not by checking an unverified database flag.

Setup stages must execute in strict order:
schemas -> tables -> procedures and views -> RBAC seed -> validation

Strict safety invariants:
- Never issue `DROP`, `TRUNCATE`, or destructive `CREATE OR REPLACE` commands during in-app setup.
- All DDL statements must use `CREATE ... IF NOT EXISTS`.
- All seed data statements must use `INSERT ... WHERE NOT EXISTS`.
- All stored procedures must execute as caller (`EXECUTE AS CALLER`).
- Invariants must be verified via programmatic self-checks (e.g. `bootstrap_ddl.py:_self_check`).

Rationale:
Destructive DDL risks enterprise data loss and wipes steward work. Non-idempotent setup causes failed restarts, deployment friction, and degraded user experience.

Implications:
- Bootstrap checks must cache positive readiness results (e.g. 2 minutes) but never cache negative results, ensuring immediate detection of newly provisioned setups.
- DDL definitions across standalone scripts, SPCS setups, and Python manifests must remain strictly synchronized.

Related: UX-004, OPS-001

---

## AD-006: Session Architecture and Scalability

Status: Accepted

Decision:
The current in-memory process dictionary session store (`core/session_store.py`) is recognized as a known single-replica constraint. The future production architecture must address horizontal scalability (multi-replica deployments, container restarts, session persistence) without weakening Snowflake-native authentication or session authority.

In login mode, the application must not replace the real Snowpark session with synthetic JWT tokens that bypass Snowflake authorization.

Rationale:
In-process session storage logs users out on container restart and fails under round-robin multi-replica load balancing without sticky sessions. However, substituting mock tokens would eliminate Snowflake's native query attribution and authorization model.

Implications:
- Current deployments must use single-instance or sticky-session routing.
- The path toward multi-replica support is tracked as a dedicated pending decision (PEND-005) rather than patched with insecure client-side session storage.

Related: SEC-001, PEND-005

---

# Authentication & Security Decisions

## SEC-001: Snowflake Session as Authority and Access Boundary

Status: Accepted

Decision:
PI-GOVERN supports two mutually exclusive authentication modes configured via `AUTH_MODE`:
1. `service`: Single process-wide connection using service credentials or SPCS ingress tokens.
2. `login`: Each user authenticates directly against Snowflake with their own credentials; the backend opens an authentic Snowpark session for that user identity.

In login mode, Snowflake session authority is the true data access boundary. Every query executed on behalf of the user runs directly through their Snowpark session.

Security mandates:
- Never store, hash, or persist user passwords in PI-GOVERN tables, caches, or files.
- Never issue synthetic JWT credentials that divorce application identity from Snowflake session identity.
- Never weaken Snowflake-native authorization checks.

Rationale:
Delegating authentication and query authority directly to Snowflake ensures zero credential exposure, native multi-factor authentication enforcement, and flawless attribution in Snowflake's `QUERY_HISTORY`.

Implications:
- In login mode, all repository queries execute through the caller's session manager, guaranteeing that Snowflake access control policies apply automatically.
- Session tokens passed to the frontend are opaque, high-entropy tokens mapping to active backend sessions, not decoded authorization claims.

Related: AD-001, AD-006, SEC-002

---

## SEC-002: Consistent Authorization Enforcement Across Endpoints

Status: Accepted

Decision:
Authorization must follow a strict sequential pipeline:
Authentication -> Identity -> Permission -> Resource Scope -> Data

Endpoints across all modules (including Dashboard and AI Assistant) must enforce explicit permission checks (`require_permission` / `require_role`). An endpoint must never return governance data or metadata solely because the requesting caller has a valid authenticated session.

Rationale:
Authentication establishes identity, but does not grant unrestricted access. Returning sensitive asset lists, policy details, or governance events without verifying in-app or Snowflake permissions violates the principle of least privilege.

Implications:
- Audit and remediate endpoints that currently check only authentication to ensure permission guards are applied.
- AI context assembly must filter retrieved asset summaries through the user's active permissions.

Related: SEC-001, AI-001, DATA-002

---

# Metadata Decisions

## MD-001: Metadata Intelligence as Governance Foundation

Status: Accepted

Decision:
Metadata Intelligence is the foundational governance object layer of PI-GOVERN, not a simple metadata listing.

The user interface follows a lightweight master-detail workspace:
- Left: Asset Explorer (compact enterprise list with multi-asset filtering)
- Right: Asset Intelligence (deep multi-tab governance details for the selected asset)

Page header direction:
- Title: Metadata Intelligence
- Subtitle: Discover, understand, and govern your Snowflake data estate.
- Avoid oversized, bulky headers such as "Snowflake governance catalog".

Rationale:
A master-detail layout allows users to browse and filter thousands of assets without context switching or pagination disruption, keeping the selected asset anchored as the focus of governance analysis.

Implications:
- The UI must avoid large, repetitive card grids that waste screen real estate.
- The workspace must fit cleanly within the viewport without global vertical page scrolling.

Related: PD-002, PD-003, UX-001, MD-002

---

## MD-002: Asset Detail Tab Structure

Status: Accepted

Decision:
The Asset Intelligence panel contains exactly six purposeful governance tabs:
1. Overview: FQN, object type, owner, steward, row count, size, quality score, governance state, tags.
2. Access History: Who has accessed or queried this asset, access patterns, and associated role exposure.
3. Governance: Active policies applied, evaluation results, classification status, masking rules.
4. Metadata: Detailed column definitions, data types, steward classifications, descriptions, and comments.
5. Change History: Structural schema drift, column modifications, ownership updates, and metadata revisions.
6. AI Assistant: Contextual assistant pre-loaded with the selected asset's governance context.

When no asset is selected, the Asset Intelligence panel displays an intentional, clean empty state:
"Select an asset - Choose an asset to explore metadata, access, governance, lineage, changes, and AI insights."

Tabs must represent genuine governance dimensions. Tabs must never be added solely for visual filler.

Rationale:
These six tabs map directly to the questions enterprise data stewards and governance officers ask about sensitive data assets.

Implications:
- Each tab must consume canonical PI_GOVERN metadata or honestly declare data absence when backend feeds are unavailable.
- Selecting an asset immediately contextualizes all six tabs without requiring separate navigational jumps.

Related: PD-002, PD-003, AI-001

---

## MD-003: Metadata Estate-Level KPIs

Status: Accepted

Decision:
The Metadata Intelligence workspace features estate-level aggregate metrics:
- Databases
- Schemas
- Tables
- Views
- Stages
- Sensitive Assets

Every metric must be derived dynamically from actual synchronized PI_GOVERN metadata (`METADATA.DATA_ASSETS`, `METADATA.ASSET_COLUMNS`). Hardcoded numbers, mock data, or arbitrary fallback metrics are strictly prohibited.

Rationale:
Governance teams require an accurate snapshot of their governed estate. Displaying fabricated or static metrics undermines the credibility of the entire platform.

Implications:
- Aggregates must be served from database rollup views or efficient summary queries (e.g. `V_ASSET_GOVERNANCE`), not computed by loading all records into the browser.

Related: DATA-001, DATA-002, PERF-001

---

# UX Decisions

## UX-001: Non-Scrolling Primary Viewport Workspace

Status: Accepted

Decision:
The primary PI-GOVERN application experience, particularly the Dashboard and Metadata Intelligence modules, must fit within the viewport as a compact, non-scrolling workspace.

Internal scrolling is permitted and expected within data tables, asset lists, and detail tabs, but the outer page layout must not scroll vertically. Global vertical scrolling, oversized heroes, and decorative page padding that forces page scrollbars are forbidden.

Rationale:
Enterprise control planes and operational consoles demand high information density and situational awareness. Non-scrolling single-viewport layouts prevent cognitive disorientation and keep primary navigation, filters, and status controls visible at all times.

Implications:
- Main containers must use strict height constraints (e.g. `h-[calc(100vh-...)]` or flex-1 with `overflow-hidden`).
- Headers, metric bars, and status banners must remain compact.

Related: UX-002, UX-003, MD-001

---

## UX-002: Business-Class Enterprise Light Theme Design Language

Status: Accepted

Decision:
The official design language of PI-GOVERN is a business-class, premium enterprise SaaS interface adhering to:
- Clean white/light theme as the primary visual mode.
- Deep blue as the primary brand and action accent.
- Restrained semantic accents: subtle purple (intelligence/AI), green (healthy/compliant), orange/red (risk/violations).
- Rounded cards with subtle borders (`border-slate-200/80` or equivalent).
- Soft enterprise drop shadows and restrained glows.
- Crisp typographic hierarchy with high readability.
- Restrained, purposeful micro-interactions and transitions.
- High visual density with efficient spacing.

Explicitly rejected styles:
- Dark theme as a default or unfinished toggle.
- Neon colors, excessive gradients, or cyberpunk accents.
- Heavy, opaque glassmorphism that obscures text readability.
- Gaming-style or decorative UI without functional purpose.

Rationale:
Enterprise data governance buyers and practitioners work in corporate environments where clarity, professionalism, and accessibility are paramount. Flashy consumer or gaming aesthetic styles detract from operational trustworthiness.

Implications:
- Components must align with Radix/shadcn clean light-theme tokens.
- Color must communicate semantic status (risk, health, classification) rather than mere decoration.

Related: UX-001, UX-003

---

## UX-003: Dual Navigation System (Expanded Sidebar and Collapsed Rail)

Status: Accepted

Decision:
The navigation system supports two first-class modes:
1. Expanded Sidebar: Full navigation hierarchy with labels, badges, and group headers.
2. Collapsed Navigation Rail: High-density icon rail.

The Collapsed Navigation Rail is an intentional, first-class navigation experience, not a crude CSS-clipped sidebar. It must preserve:
- Logical navigation grouping and separator lines.
- Active route state indicators.
- Instant, accessible tooltip labels on hover and focus.
- Touch/click hit areas adhering to accessibility guidelines (minimum 40x40px).
- Smooth, responsive collapse and expand transitions.

The existing approved expanded sidebar must not be redesigned without an explicit decision.

Rationale:
Users working on data-dense tasks (lineage graphs, metadata explorers) need maximum horizontal screen width while retaining one-click access to all platform modules.

Implications:
- Collapsed states must be implemented with Radix Tooltip primitives to guarantee accessibility and screen reader support.
- Navigation state must persist across route transitions via user preferences.

Related: UX-001, UX-002

---

## UX-004: Explicit Setup and Error State Modeling

Status: Accepted

Decision:
The application user interface must explicitly distinguish and model distinct environment and error states:
1. `AUTHENTICATION_FAILED`: Invalid credentials, expired session, or account lockout.
2. `MISSING_SETUP`: Database `PI_GOVERN` or required schemas/tables do not exist.
3. `PERMISSION_DENIED`: Valid credentials, but insufficient Snowflake grants to read or operate.
4. `READY`: Fully provisioned, validated, and operational.
5. `PARTIAL_SETUP`: Objects exist but tables, views, or procedures are incomplete.
6. `REPAIR_REQUIRED`: Drift or schema incompatibility detected.

Under no circumstances should the UI display a generic "Error loading data" or spin indefinitely when an environment simply lacks setup.

When setup is missing, the UI must provide:
- A clear, human-readable explanation of the missing objects.
- The exact idempotent SQL setup script required.
- A one-click Copy action for the setup script.
- A Recheck action to immediately re-test readiness without full page reload.
- Zero fake progress bars or artificial wait times.

Rationale:
Misleading or generic error messages cause support escalations and frustrate administrators during onboarding. Giving administrators the exact script to run turns a blocker into an immediate self-service resolution.

Implications:
- Backend bootstrap APIs must return structured state codes matching these six scenarios.
- Setup gates in the frontend must route users to the setup remediation card rather than failing silently.

Related: AD-005, DATA-002

---

## UX-005: Global Environment Display Fidelity

Status: Accepted

Decision:
The application must display the genuine connected Snowflake environment context (account name, active role, current database) wherever an environment indicator is required. Hardcoding demo strings such as `INTERACTIVE_DEMO` on production dashboards is strictly prohibited.

The Dashboard header does not require an oversized environment badge if it adds visual clutter. Environment isolation remains the responsibility of the backend and session layer.

Rationale:
Data stewards managing multiple Snowflake environments (development, staging, production) must never be confused about which estate they are inspecting or governing.

Implications:
- Environment badges must bind directly to `workspace-store` or `/api/v1/bootstrap/status` context.
- Fallbacks must clearly indicate disconnected or unconfigured states rather than displaying fake environment identifiers.

Related: AD-004, DATA-002

---

# Data & KPI Decisions

## DATA-001: Dynamic Dashboard KPI Architecture and Semantics

Status: Accepted

Decision:
The primary KPI banner on the Executive Dashboard replaces the legacy metric quartet (Total Assets, Active Policies, Governance Score, Open Events) with four estate-centric metrics:
1. Databases Governed
2. Schemas Governed
3. Tables & Views Governed
4. Sensitive Assets Identified

Semantic standards:
- Always use truthful terminology: "Governed", "Discovered", "Cataloged", "Identified".
- Never use "Created" unless PI-GOVERN itself created the underlying Snowflake object.
- Never display "Snowflake Roles Governed" until native Snowflake role and grant synchronization is genuinely implemented.
- Every metric must be dynamically aggregated from canonical `PI_GOVERN` tables.
- Static visual placeholders (e.g. 12, 48, 1,284, 167) must never be used in production builds.

Rationale:
Governance executives need to understand the scale of their governed estate and the footprint of sensitive data. Legacy metrics were vague or dependent on incomplete modules (such as un-automated governance events).

Implications:
- The `/dashboard/context` API endpoint must serve these four counts computed directly from `METADATA.DATA_ASSETS` and `METADATA.ASSET_COLUMNS`.
- Frontend metric cards must handle loading skeletons cleanly without flashing hardcoded numbers.

Related: PD-001, DATA-002, PEND-001

---

## DATA-002: Strict Data Integrity and Anti-Fabrication Rule

Status: Accepted

Decision:
PI-GOVERN enforces a strict data integrity mandate: never fabricate data under any circumstances. This applies across:
- KPI numbers and trend percentages
- Asset lists and schemas
- Object owners and data stewards
- Governance policies and evaluation results
- Access history records and query logs
- Data classifications and sensitivity tags
- Schema change history and drift records
- Governance events and violation findings
- Snowflake roles, grants, and privileges
- Remediation actions and execution statuses

When a capability or data feed is unsupported by the connected Snowflake environment or not yet implemented in PI-GOVERN:
1. Do not invent mock data or generate artificial records.
2. Clearly document the capability gap in code and UI empty states.
3. Provide clean, decoupled architectural extension points.
4. Track the missing capability as a formal Pending Decision or roadmap work item.

Rationale:
Governance software is audited by security, compliance, and risk officers. Fabricating a single finding, role, or access log completely invalidates the product's credibility and legal reliability.

Implications:
- Mock generators must be restricted to standalone offline unit tests, never imported into production execution paths.
- Empty states must inform users how to enable or synchronize the missing data.

Related: PD-004, SEC-002, UX-004

---

# AI Assistant Decisions

## AI-001: Context-Aware Governance Assistant

Status: Accepted

Decision:
The AI Assistant module operates in two distinct, context-sensitive modes:
1. Global Data Estate Mode (no asset selected): Operates as a strategic governance advisor answering estate-level questions regarding overall coverage, policy gaps, compliance posture, and general Snowflake governance best practices.
2. Selected Asset Mode (asset selected): Automatically receives the selected Governance Object's complete contextual envelope (fully qualified name, classification, sensitivity, column schema, active policies, access history, lineage dependencies, recent changes, and risk indicators).

Key assistant operational boundaries:
- The AI Assistant is advisory and read-only. It cannot execute mutations or apply DDL policies without an authorized, verified user confirmation workflow.
- The assistant must never fabricate governance facts, policies, owners, or security findings.
- The assistant relies on Snowflake Cortex (`SNOWFLAKE.CORTEX.COMPLETE`) executed within the customer's Snowflake account, ensuring enterprise data and metadata never leave the Snowflake security perimeter.

Representative contextual queries supported:
- "Why is this asset classified as PII?"
- "Who can access this asset and through which roles?"
- "Which governance policies currently protect this asset?"
- "What schema or ownership changes occurred recently?"
- "What downstream tables and views depend on this object?"
- "What governance risks or compliance gaps exist here?"

Rationale:
A generic chatbot disconnected from the active workspace forces users to copy-paste table schemas and questions. Context-aware injection grounds the LLM in verified metadata, eliminating hallucinations and delivering instant, actionable insights.

Implications:
- Context assembly in `backend/app/modules/ai_assistant/context.py` must filter context based on the acting user's permissions and selected asset ID.
- LLM inference calls must remain inside Snowflake via Cortex per ADR-001.

Related: PD-002, PD-003, SEC-002, DATA-002

---

# Performance & Engineering Discipline Decisions

## PERF-001: Avoid N+1 and Redundant System Scans

Status: Accepted

Decision:
PI-GOVERN backend services and frontend queries must prioritize rapid rendering and minimal warehouse credit consumption:
- Use set-based aggregate queries and analytical rollups (e.g. `V_ASSET_GOVERNANCE`).
- Implement cursor-based or offset pagination for large asset, column, and audit listings.
- Cache stable bootstrap discovery metadata and readiness validations with short, safe TTLs (2 to 5 minutes).
- Perform incremental metadata and lineage synchronization rather than full estate scans whenever possible.

Strictly avoid:
- N+1 database queries across child assets, columns, or policy associations.
- Live queries against Snowflake `INFORMATION_SCHEMA` or `ACCOUNT_USAGE` on user page loads.
- Triggering background metadata synchronization on page render.
- Redundant or uncontrolled Snowflake Cortex LLM calls.

Rationale:
Snowflake query execution incurs monetary compute costs and latency. Inefficient querying produces sluggish UI response times and inflates customer Snowflake compute bills.

Implications:
- Data tables must use virtualized lists (`@tanstack/react-virtual`) and paginated API endpoints.
- Backend services must assemble dashboard and workspace contexts via concurrent `asyncio.gather` queries against mirror tables.

Related: AD-001, AD-002, DATA-001

---

## OPS-001: Controlled Incremental Engineering Discipline

Status: Accepted

Decision:
All PI-GOVERN modifications must follow a disciplined, controlled engineering methodology:
1. Inspect the existing architecture and codebase before proposing or writing code.
2. Identify the authoritative source of truth for the affected data or logic.
3. Map internal and cross-module dependencies.
4. Evaluate blast radius, security boundaries, and backward compatibility.
5. Implement the smallest correct, self-contained change.
6. Test and verify behavior with automated or reproducible checks.
7. Update `Pi-Govern/decision.md` whenever an implementation establishes, refines, or alters a meaningful architectural decision.

Massive, uncontrolled rewrites and undocumented architectural drift are strictly forbidden.

Rationale:
PI-GOVERN is a complex governance platform integrating deep Snowflake SQL, FastAPI services, and reactive Next.js state. Uncontrolled changes introduce subtle regressions, security bypasses, and DDL synchronization drift.

Implications:
- PRs and commits must remain focused and traceable.
- Decisions documented in this log must be respected during implementation and review.

Related: AD-005, SEC-001, DATA-002

---

# Future Decisions

This section tracks explicitly pending decisions, architectural proposals, and roadmap initiatives. Nothing in this section represents an accepted decision.

## PEND-001: Evidence-Based Governance Score Model

Status: Pending

Context:
The legacy Governance Score implementation uses a hardcoded base score of 92, an arbitrary fallback of 86, and a rigid deduction formula:
`score = max(0, min(100, 92 - (critical_events * 3 + failed_evaluations * 5)))`
This model has known deficiencies: a perfectly governed account can never exceed 92, and positive governance dimensions (metadata completeness, classification coverage, lineage health, policy attachment, ownership assignment) contribute zero points.

Proposed Direction:
A future dedicated decision will replace or redesign the Governance Score into a multi-dimensional, evidence-based measurement framework:
- Metadata coverage ratio
- Sensitive data classification coverage
- Lineage completeness and dependency tracking
- Asset ownership and stewardship assignment
- Active policy attachment and evaluation success rate
- Access risk and excessive privilege exposure
- Unresolved governance violations and event trends

Until this decision is formalized, the legacy Governance Score is demoted from primary dashboard KPI status, but the legacy calculation logic remains intact in the codebase to prevent breaking secondary dependencies.

Related: DATA-001, DATA-002

---

## PEND-002: Automated Governance Event Generation Engine

Status: Pending

Context:
Currently, rows in `GOVERNANCE_EVENTS.GOVERNANCE_EVENTS` are written exclusively via manual REST API calls (`POST /governance-events`). There is no automated daemon or background process detecting governance drift. Consequently, an empty events table indicates an absence of recorded events, not a healthy data estate.

Proposed Direction:
Design and implement an automated governance event generation engine that continuously or periodically analyzes:
- Metadata drift (unexpected dropped tables, altered columns, missing comments).
- Classification drift (unclassified columns in sensitive tables).
- Policy violations (failing evaluation states from `POLICIES.POLICY_EVALUATION_STATE`).
- Access anomalies (new unreviewed grants to broad roles like `PUBLIC`).
- Ownership gaps (unassigned owners or departed stewards).

Related: PD-001, AD-003, DATA-002

---

## AD-007: Native Snowflake RBAC Governance Control Plane

Status: Accepted (Supersedes PEND-003)

Decision:
PI-GOVERN establishes an asset-centric, explainable Snowflake RBAC control plane while preserving Snowflake as the sole enforcement authority.
The architecture enforces a strict two-layer RBAC separation:
1. Layer 1 (PI-GOVERN Application RBAC): Internal platform permissions stored in RBAC.ROLES, RBAC.PERMISSIONS, and RBAC.ROLE_ASSIGNMENTS controlling who can operate PI-GOVERN itself (admin, viewer).
2. Layer 2 (Snowflake Data RBAC): Native Snowflake access control governing who can access Snowflake data assets (users, roles, role hierarchies, object grants).

Key architectural invariants:
- Snowflake is the source of truth: Users, roles, grants, and role hierarchies are queried live from Snowflake metadata (SHOW USERS, SHOW ROLES, SHOW GRANTS TO ROLE, ACCOUNT_USAGE) with no synthetic or fabricated mock data.
- Effective access engine: Traces access paths through direct grants, schema/database inheritance, and role-to-role inheritance DAGs.
- Action safety and verification: All mutations (create user, create role, grant privilege, revoke privilege) execute real Snowflake SQL, re-query Snowflake to verify state changes, and log an audit event to AUDIT.AUDIT_LOGS.
- Zero credential leakage: User creation passwords exist only transiently during the DDL operation and are never logged, persisted, or returned in API responses.
- Grounded AI intelligence: RBAC AI queries are strictly bounded to verified Snowflake context (selected role, user, or asset) without hallucinated grants or permissions.

Rationale:
Enterprises need visibility, explainability, and operational control over complex Snowflake role hierarchies without replacing Snowflake's native authorization engine.

Implications:
- Backend provides dedicated Snowflake RBAC endpoints under /api/v1/rbac/snowflake/...
- Frontend renders the approved command-center layout: KPIs, role explorer, role details (overview, asset access, users, inherited roles, policies, activity, AI assistant).
- Existing internal RBAC tables remain untouched for platform authorization.

Related: PD-001, PD-003, SEC-001, SEC-002, DATA-002, AI-001

---

## PEND-003: Native Snowflake RBAC Governance (Superseded by AD-007)

Status: Superseded (by AD-007)

Context:
The legacy `/rbac` module only managed internal PI-GOVERN roles and permissions in `RBAC.ROLES` and `RBAC.PERMISSIONS`. Superseded by AD-007 which implements the full native Snowflake RBAC Control Plane.

---

## PEND-004: Snowflake Native Policy Enforcement (Masking and Row Access)

Status: Pending

Context:
PI-GOVERN's policy engine (`modules/policies/engine.py`) evaluates attribute rules in Python and writes results to `POLICIES.POLICY_EVALUATION_STATE`. It does not generate, alter, or apply native Snowflake DDL policies (`CREATE MASKING POLICY`, `CREATE ROW ACCESS POLICY`, `ALTER TABLE ... MODIFY COLUMN ... SET MASKING POLICY`).

Proposed Direction:
Establish a complete policy lifecycle spanning:
1. Definition: Authoring business and data protection policies in PI-GOVERN.
2. Evaluation: Simulating and testing policy match conditions against assets.
3. Enforcement: Compiling verified policies into native Snowflake masking and row access DDL and applying them via caller permissions.
4. Verification: Querying sample records to cryptographically verify data masking.
5. Audit: Logging all enforcement actions to `AUDIT.AUDIT_LOGS`.

Related: PD-001, PD-004, DATA-002

---

## PEND-005: Multi-Replica Distributed Session Store

Status: Pending

Context:
In login authentication mode, active Snowpark sessions are held in a local Python dictionary in process memory (`core/session_store.py`). In a multi-replica container deployment (e.g. SPCS or Kubernetes behind a standard load balancer), subsequent HTTP requests from the same user fail if routed to a different container replica. Furthermore, restarting the container logs out all active users.

Proposed Direction:
Evaluate enterprise session distribution patterns that preserve Snowflake security boundaries:
- Option A: Redis or distributed cache storing encrypted session tokens with sticky session routing at the ingress controller.
- Option B: SPCS native ingress authentication headers where Snowflake handles container authentication transparently.
- Option C: Stateless request-scoped Snowpark connections using short-lived session authorization tokens or key-pair authentication.

Any chosen solution must uphold SEC-001 and strictly avoid storing user passwords or issuing unverified tokens.

Related: AD-006, SEC-001
