I’m going to inspect the React pages and key domain components first, then translate each interaction into backend agent requirements and FastAPI endpoint patterns. I’ll start with a focused codebase exploration to extract concrete UI intents before drafting the architecture mapping.

Map UI to agent needs

Read [](file:///home/ubuntu/docket_poc/Frontend/README.md)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/app/routes/pageRegistry.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/package.json)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Home/HomePage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Intake/IntakePage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Docket/DocketPage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Context/ContextPage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Execution/ExecutionPage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Lineage/LineagePage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Audit/AuditPage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Workflow)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Review/ReviewerHomePage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Review/ReviewPage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Admin/AdminConsolePage.jsx#1-1), lines 1 to 100

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/execution/hooks/useExecution.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/hooks/useContextQuery.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/hooks/useDocket.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#150-150), lines 150 to 200

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/execution/services/pipeline.service.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/store/useReferenceStore.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/auth/services/auth.service.js#1-1), lines 1 to 150

Searched for regex `httpClient\.get|httpClient\.post|httpClient\.put|httpClient\.delete`, 3 results

Read [](file:///home/ubuntu/docket_poc/Frontend/src/shared/services/httpClient.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/store/docketSlice.js)

Searched for regex `export const.*=.*async|fetch|api|service` (`**/services/**`), 27 results

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#300-300), lines 300 to 350

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#400-400), lines 400 to 553

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Settings/SettingsPage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Workspace/PersonaWorkspacePage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Auth/AuthPage.jsx#1-1), lines 1 to 150

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/AgentPipeline.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/AIInsightBlock.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/ExecutionState.jsx#1-1), lines 1 to 100

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/LineageGraphPanel.jsx#1-1), lines 1 to 100

Searched for text `recordAuditLog|recordExecutionHistory|recordRevisionHistory`, 18 results

Read [](file:///home/ubuntu/docket_poc/Frontend/utils/executionLifecycle.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/RoleSelectionCards.jsx#1-1), lines 1 to 100

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/EvidenceDrawer.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/layouts/AppLayout.jsx)

Read memory

Created memory file [](file:///home/ubuntu/.vscode-server/data/User/workspaceStorage/0fa15931395fabdd4b337f3d0341d415/GitHub.copilot-chat/memory-tool/memories/ZThiOTQ3MDMtYWJkZi00YmE1LWFmMWEtOTQ1ZTUzNTdlOTNi/frontend_exploration_findings.md)

Optimizing tool selection...I finished a full repository scan and now I’m extracting the findings file to turn it into an implementation-ready backend architecture spec aligned to your requested format.

Read [](file:///home/ubuntu/.vscode-server/data/User/workspaceStorage/0fa15931395fabdd4b337f3d0341d415/GitHub.copilot-chat/chat-session-resources/e8b94703-abdf-4ba5-af1a-945e5357e93b/call_1MSbyhaCfJjR89jbKYV63RFM__vscode-1776151419887/content.txt#1-1), lines 1 to 400

Read [](file:///home/ubuntu/.vscode-server/data/User/workspaceStorage/0fa15931395fabdd4b337f3d0341d415/GitHub.copilot-chat/chat-session-resources/e8b94703-abdf-4ba5-af1a-945e5357e93b/call_1MSbyhaCfJjR89jbKYV63RFM__vscode-1776151419887/content.txt#401-401), lines 401 to 583

Next I’ll verify the most important findings directly in core route, page, and service files so the backend design is grounded in actual code paths rather than assumptions.

Read [](file:///home/ubuntu/docket_poc/Frontend/src/app/routes/pageRegistry.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Execution/ExecutionPage.jsx#1-1), lines 1 to 320

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Intake/IntakePage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Review/ReviewPage.jsx#1-1), lines 1 to 380

I validated key page behavior and now I’m checking hooks/services to map actual API boundaries versus mocked local workflows, so endpoint recommendations are precise.

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/hooks/useDocket.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/execution/hooks/useExecution.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/services/api.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#1-1), lines 1 to 420

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/docket/services/docket.service.js#420-420), lines 420 to 553

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/execution/services/pipeline.service.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Context/ContextPage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Audit/AuditPage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Settings/SettingsPage.jsx)

I’ve confirmed core service behavior is mostly mocked/local; I’ll now quickly sample remaining pages to complete the page-by-page intent and interaction coverage you requested.

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Home/HomePage.jsx#1-1), lines 1 to 320

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Review/ReviewerHomePage.jsx#1-1), lines 1 to 320

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Docket/DocketPage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Lineage/LineagePage.jsx)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Workspace/PersonaWorkspacePage.jsx#1-1), lines 1 to 340

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Auth/AuthPage.jsx#1-1), lines 1 to 360

Read [](file:///home/ubuntu/docket_poc/Frontend/src/features/auth/services/auth.service.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/shared/services/httpClient.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/hooks/useContextQuery.js)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Admin/AdminConsolePage.jsx#1-1), lines 1 to 420

I have enough page coverage; I’m now validating the execution and review action handlers specifically because they define the real interaction contracts that the backend must honor.

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Execution/ExecutionPage.jsx#320-320), lines 320 to 860

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Review/ReviewPage.jsx#380-380), lines 380 to 980

Read [](file:///home/ubuntu/docket_poc/Frontend/src/components/domain/ExecutionState.jsx#1-1), lines 1 to 260

Searched for regex `httpClient\.(get|post|put|patch|delete)\(` (`**/src/**/*.js`), 1 result

Searched for regex `httpClient\.(get|post|put|patch|delete)\(` (`**/src/**/*.{js,jsx}`), 3 results

I’ve confirmed only three real HTTP reads exist; I’ll capture exact line-level anchors for key behaviors so the architecture mapping is traceable and implementation-ready.

Searched for text `const pageMap =` (`**/src/app/routes/pageRegistry.jsx`), 1 result

Searched for regex `setActivePage\('execution'\)` (`**/src/pages/Intake/IntakePage.jsx`), 1 result

Searched for regex `const \{ triggerExecution, pollExecutionStatus \} = useExecution\(\)` (`**/src/pages/Execution/ExecutionPage.jsx`), 1 result

Searched for text `askIntelligenceQuestion` (`**/src/pages/Execution/ExecutionPage.jsx`), 2 results

Searched for regex `recordAuditLog\(` (`**/src/pages/Execution/ExecutionPage.jsx`), 1 result

Searched for regex `ARTIFACT_STORE_KEY|EXECUTION_HISTORY_KEY|AUDIT_LOG_KEY|REVISION_HISTORY_KEY` (`**/src/features/docket/services/docket.service.js`), 12 results

Searched for text `fetchContextByReference = async` (`**/src/features/docket/services/docket.service.js`), 1 result

Searched for regex ``httpClient.get\(`/docket/user/\$\{userId\}`\)`` (`**/src/pages/Home/HomePage.jsx`), 1 result

Searched for regex ``httpClient.get\(`/docket/reviewer/\$\{reviewerId\}`\)`` (`**/src/pages/Review/ReviewerHomePage.jsx`), 1 result

Searched for regex `getGlobalIntelligenceLedger\(|recordAuditLog\(|recordExecutionHistory\(` (`**/src/pages/Review/ReviewPage.jsx`), 7 results

Searched for regex `validateYAML\(|writeYamlConfig\(|readYamlConfig\(` (`**/src/pages/Settings/SettingsPage.jsx`), 7 results

Searched for text `window.localStorage` (`**/src/features/auth/services/auth.service.js`), 6 results

Searched for regex `recordAuditLog\(` (`**/src/pages/Admin/AdminConsolePage.jsx`), 2 results

Searched for text `useContextQuery\(referenceId\)` (`**/src/pages/Context/ContextPage.jsx`), 1 result

Searched for regex `getGlobalIntelligenceLedger\(` (`**/src/pages/Audit/AuditPage.jsx`), 1 result

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Governance)

Read [](file:///home/ubuntu/docket_poc/Frontend/src/pages/Workflow)

I’ve completed the reverse-engineering and verified key flows against page and service code. I’m now compiling the final backend architecture spec in your exact 8-part format, including interaction-to-agent mapping, schemas, endpoint design, memory, complexity split, and observability.## 1. UI Intent Summary

| UI Surface | Primary user intent | Problem being solved | Nature |
|---|---|---|---|
| Intake page | Start an investigation from a reference plus optional note; validate YAML changes before run | Convert ambiguous business issue into a structured execution request | Data submission + setup |
| Execution page | Run staged pipeline, watch progress, ask AI before final publish, delegate reviewers | Human-in-the-loop control over agent pipeline before dossier finalization | Monitoring + decision-support + submission |
| Context page | View synthesized summary, anomalies, missing data, and evidence links | Turn multi-domain raw records into analyst-understandable context | Analysis + decision-support |
| Docket page | Edit draft narrative/report sections | Allow manual shaping of generated intelligence artifact | CRUD-style editing on generated artifact |
| Reviewer home | Prioritize and open assigned review queue | Operationalize review workload and SLA throughput | Retrieval + workflow triage |
| Review page | Approve/reject/revise with comments, delegate to approver | Governance and quality gate before final decision | Decision-support + submission |
| Lineage page | Inspect evidence-to-assertion trace paths | Trust, explainability, and debugging of claims | Debugging/error analysis + decision-support |
| Home page | Track personal docket portfolio by status and action items | Visibility into lifecycle state and required actions | Retrieval + monitoring |
| Workspace page | View cross-reference ledger, grouped versions, latest activity | Portfolio-level operations and auditability | Monitoring + retrieval |
| Audit page | Search/filter activity stream and reference activity | Incident/debug timeline and accountability | Monitoring + debugging |
| Settings page | Edit and validate YAML extraction/mapping config | Safe configuration changes before new runs | Configuration/setup |
| Admin console | Manage users/roles, workflow controls, overrides, SLA visibility | Governance, access control, and operational control plane | Configuration + monitoring + decision support |
| Auth page | Register/login with role/profile defaults | Identity and role-based entry into workflow | Setup + submission |

Evidence anchors:
- Routing and page inventory: pageRegistry.jsx
- Intake to execution transition: IntakePage.jsx
- Execution orchestration hooks: ExecutionPage.jsx
- Context query usage: ContextPage.jsx
- Home and reviewer queue API reads: HomePage.jsx, ReviewerHomePage.jsx

---

## 2. Interaction → Agent Mapping Table

| Interaction | UI location | Classification | Agent type | Required inputs | Required outputs |
|---|---|---|---|---|---|
| Submit investigation request | Intake | Data Submission | Intake Agent | reference_id, user_query, user_role, yaml_version, tenant/env metadata | execution_id, accepted/rejected, normalized scope, validation warnings |
| Trigger pipeline run | Execution | Data Submission | Orchestrator Agent | execution_id, reference_id, run_mode, user context | run_state, stage plan, task ids |
| Fetch execution stage progress | Execution | System Monitoring | Simple API (deterministic) | execution_id | current_stage, percent, ETA, warnings/errors |
| Retrieve CRM/Salesforce/MES/QMS/Proc data | Hidden stage | Data Retrieval | Domain Retrieval Agents | reference_id, source connectors, time bounds, schema mapping | normalized domain payloads, retrieval confidence, missing entities |
| Build cross-domain context graph | Hidden stage | Decision Support | Correlation Agent | normalized domain payloads, join heuristics, mapping rules | graph edges/nodes, confidence_by_domain, anomaly candidates |
| Generate narrative and insights | Hidden stage + Context | Decision Support | Synthesis Agent | context graph, prior versions, policy constraints | summary, anomalies, missing_data, recommendations, evidence citations |
| Ask follow-up AI question | Execution chat | Decision Support | Analyst Copilot Agent | docket_id, question, context snapshot, lineage pointers | explanation, answer, evidence links, confidence |
| Publish docket artifact | End of execution | Data Submission | Publisher Agent | synthesis output, lineage refs, status policy | docket_id, version, state, publish timestamp |
| Approve or reject docket | Review | Data Submission | Workflow Governance Agent | docket_id, decision, reviewer/approver id, comment | state transition result, next assignee, audit event ids |
| Request revision and restart from SLM stage | Review | Debugging/Decision Support | Revision Agent | docket_id, target_section, revision_instruction, previous output | revision_run_id, locked_stages, new version draft |
| Delegate to reviewers/approvers | Execution/Review | Configuration + Submission | Assignment Agent | docket_id, assignee ids, delegator, reason | assignment records, notifications, queue updates |
| Query audit and activity | Audit/Workspace/Admin | System Monitoring | Simple API (deterministic) | filters: actor, action, ref, time | event list, aggregates, correlations |
| Validate/apply YAML | Settings | Configuration/Setup | Config Validation Agent (hybrid) | yaml text, schema version, actor | valid/invalid, line errors, semantic warnings, applied config version |
| Auth and role initialization | Auth | Configuration/Setup | Simple API | credentials/profile + desired role | token/session, role claims, defaults |

---

## 3. Required Agent Types

1. Intake Agent  
Purpose: normalize and validate incoming investigation requests.

2. Pipeline Orchestrator Agent  
Purpose: manage stage sequencing, retries, and stage transition policy.

3. Domain Retrieval Agents  
Purpose: source-specific retrieval/normalization from Salesforce, Snowflake-facing marts, MES/QMS/ERP systems.

4. Correlation Agent  
Purpose: multi-domain entity/time correlation and confidence scoring.

5. Synthesis Agent  
Purpose: produce explainable summary, anomalies, gaps, recommendations.

6. Analyst Copilot Agent  
Purpose: interactive Q&A over execution context and lineage.

7. Revision Agent  
Purpose: targeted re-processing using revision instructions while preserving locked upstream stages.

8. Workflow Governance Agent  
Purpose: reviewer/approver decisions, policy enforcement, routing/escalation.

9. Assignment Agent  
Purpose: reviewer/approver delegation and queue updates.

10. Config Validation Agent  
Purpose: YAML syntax + semantic rule validation and versioning.

11. Publisher Agent  
Purpose: persist immutable docket artifact and evidence linkage.

---

## 4. Input/Output Schema Design

### 4.1 Canonical agent input envelope

{
  request_id: string,
  user_query: string,
  context: {
    reference_id: string,
    docket_id: string optional,
    execution_id: string optional,
    table_name: string optional,
    error_logs: array optional,
    timestamps: {
      from: ISO datetime optional,
      to: ISO datetime optional
    },
    evidence_ids: array optional,
    target_section: string optional
  },
  metadata: {
    environment: dev|qa|prod,
    tenant_id: string,
    pipeline_name: string,
    user_id: string,
    user_role: BUSINESS_USER|DATA_ANALYST|REVIEWER|APPROVER|ADMIN,
    yaml_version: string optional,
    correlation_id: string,
    trace_id: string
  },
  memory_hints: {
    include_previous_errors: boolean,
    include_prior_versions: boolean,
    include_user_history: boolean
  }
}

### 4.2 Canonical agent output envelope

{
  request_id: string,
  explanation: string,
  root_cause: {
    summary: string,
    factors: array of strings,
    confidence: 0.0-1.0
  },
  resolution_steps: array of {
    step: string,
    owner: string,
    priority: low|medium|high,
    eta_hours: number optional
  },
  suggested_code: array of {
    target_system: string,
    snippet: string,
    rationale: string
  },
  risk_analysis: {
    overall: low|medium|high|critical,
    risks: array of {
      risk: string,
      likelihood: low|medium|high,
      impact: low|medium|high,
      mitigation: string
    }
  },
  evidence: array of {
    source: string,
    reference: string,
    confidence: 0.0-1.0
  },
  next_actions: array of strings,
  observability: {
    trace_id: string,
    model_used: string,
    token_usage: number optional,
    latency_ms: number
  }
}

### 4.3 Stage-specific additions

- Debugging-focused requests: require context.error_logs, context.timestamps, metadata.pipeline_name.
- Data engineering enhancement requests: include context.table_name, schema diffs, SLA targets.
- Monitoring requests: include execution_id, stage history, source latency metrics.

---

## 5. Suggested FastAPI Endpoint Design

### 5.1 Replace traditional endpoints with agent endpoints

| Instead of | Use | Why |
|---|---|---|
| GET pipeline status | POST /agent/analyze-pipeline | Status plus reasoning, bottleneck detection, projected impact |
| GET context by reference | POST /agent/synthesize-context | Deterministic retrieval plus intelligent cross-domain synthesis |
| POST simple approve | POST /agent/govern-decision | Policy-aware routing, escalation logic, risk notes |
| POST ask ai (thin) | POST /agent/assist-review | Context-aware answer with evidence/lineage trace |
| POST revision | POST /agent/revise-docket | Instruction-aware partial rerun and versioned artifact |

### 5.2 Recommended endpoint set

1. POST /agent/intake  
2. POST /agent/orchestrate-execution  
3. POST /agent/retrieve-domain-data  
4. POST /agent/correlate-context  
5. POST /agent/synthesize-docket  
6. POST /agent/assist-review  
7. POST /agent/govern-decision  
8. POST /agent/revise-docket  
9. POST /agent/assign-work  
10. POST /agent/validate-config  
11. GET /state/execution/{execution_id}  
12. GET /state/docket/{docket_id}  
13. GET /state/audit-events

### 5.3 Complexity split (simple vs agent vs hybrid)

- Simple API enough:
1. Session/auth token issuance
2. Fetch immutable artifacts by id
3. Fetch audit logs with filters
4. Poll execution state snapshot

- Agent reasoning required:
1. Root-cause analysis for delays/failures
2. Cross-domain context synthesis
3. Revision instruction application
4. Interactive review Q&A
5. Risk-aware governance recommendations

- Hybrid:
1. YAML validation (syntax deterministic + semantic recommendations agent-assisted)
2. Workflow decisioning (state transition deterministic + escalation/risk reasoning)
3. Monitoring (raw metrics deterministic + anomaly interpretation by agent)

---

## 6. Context & Memory Requirements

### 6.1 What must persist

1. Execution memory per reference  
Includes stage outputs, intermediate context graph, anomalies, and errors.

2. Artifact memory per docket version  
Immutable snapshot, lineage links, approval metadata.

3. User/role memory  
Default assignees, role claims, prior actions affecting routing.

4. Error memory  
Prior failure signatures should influence retrieval and synthesis prompts on retry.

5. Config memory  
Active YAML version and diff history tied to execution runs.

### 6.2 Memory policy

- Short-term working memory: per execution run.
- Episodic memory: per reference and docket lineage.
- Organizational memory: cross-run incident patterns, source reliability, recurring root causes.

### 6.3 Behavioral requirement

Yes, previous errors should influence future responses.  
Example: if procurement connector repeatedly times out, retrieval and orchestration agents should automatically lower confidence, surface risk, and propose fallback retrieval path.

Evidence of current local-only persistence:
- Docket/audit/execution/revision keys: docket.service.js
- Auth/session local storage: auth.service.js

---

## 7. Observability Plan

### 7.1 Required trace model

Every request and stage event must carry:
1. request_id
2. correlation_id
3. trace_id
4. reference_id
5. execution_id and docket_id where applicable
6. agent_name and model_version
7. latency, token usage, retry count

### 7.2 Event taxonomy

1. intake.accepted / intake.rejected
2. retrieval.started / retrieval.partial_miss / retrieval.failed / retrieval.completed
3. correlation.completed / correlation.low_confidence
4. synthesis.started / synthesis.completed / synthesis.guardrail_block
5. governance.approved / governance.rejected / governance.revision_requested
6. assignment.created / assignment.failed
7. config.validated / config.applied / config.rejected
8. execution.stage_transition
9. execution.error
10. alert.sla_breach

### 7.3 Debuggable artifacts to expose

1. Agent thought summary (not chain-of-thought, but decision rationale)
2. Evidence map used for each recommendation
3. Confidence per domain and per claim
4. Prompt/version metadata for reproducibility
5. Versioned config snapshot used during run

Frontend surfaces indicating this need:
- Audit and ledger views: AuditPage.jsx, PersonaWorkspacePage.jsx
- Review decision/audit recording: ReviewPage.jsx

---

## 8. Gaps & Assumptions

### 8.1 Gaps

1. Most pipeline behavior is currently simulated in frontend services and local storage, not backend agents.
2. Only three direct HTTP reads are present today:
   - HomePage.jsx
   - ReviewerHomePage.jsx
   - docket.service.js
3. Governance and Workflow page folders are empty, so future UI contracts are undefined:
   - Governance
   - Workflow
4. No backend-enforced RBAC visible from frontend integration patterns.
5. No external data connector contract for Salesforce or Snowflake yet.

### 8.2 Assumptions

1. Reference id is the primary correlation key across systems.
2. Docket versioning is required and immutable after approval.
3. Review process supports reviewer and approver roles with delegation.
4. YAML is expected to materially alter extraction/correlation behavior.
5. Agentic backend should support both synchronous response and async run handles.

---

If useful, I can produce the next step as a concrete FastAPI contract pack:
1. Pydantic request/response models for all agent endpoints.
2. Orchestrator state machine definition.
3. Event schema for OpenTelemetry plus audit store.
4. Minimal implementation skeleton for one full path: intake → orchestrate → retrieve → synthesize → publish.
