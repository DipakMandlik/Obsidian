# Cortex Code (CoCo) — Killer Live Demo Commands

Type the command and let CoCo demonstrate it. Don't explain the feature first.

## 1. Discover the entire Snowflake environment
```text
/explore the entire Snowflake environment I currently have access to. Do not ask me what to inspect. Discover the databases, schemas, tables, views, stages, warehouses, roles, grants, pipelines, and important dependencies. Build a concise architecture map and identify the 10 most interesting findings.
```

## 2. Find a problem you don't tell it about
```text
Find something genuinely wrong or suspicious in this Snowflake environment. Do not rely on obvious metadata alone. Investigate query history, object relationships, permissions, warehouse behavior, and data patterns. Prove the finding with evidence and explain the root cause.
```

## 3. Find expensive SQL
```text
Analyze recent Snowflake workload and identify the queries that are consuming disproportionate compute. Don't simply rank queries by credits. Determine WHY they are expensive, inspect their SQL, identify inefficient patterns, and propose the smallest code or architecture changes that would reduce cost without degrading correctness.
```

## 4. Investigate a mysterious data-quality issue
```text
Assume the business reports that today's customer numbers are incorrect. You have no information about where the problem originates. Investigate the complete data path, trace dependencies backward from the final analytical objects, identify where the discrepancy originates, and prove your conclusion with SQL evidence.
```

## 5. Recursive dependency challenge
```text
Take the most important analytical table in this environment and recursively trace everything it depends on. Continue until you reach the original source objects. Then identify any broken, suspicious, duplicated, or unnecessary dependency in the chain.
```

## 6. Act like a security auditor
```text
Perform a Snowflake access-control investigation. Identify users and roles with potentially excessive privileges, trace inherited privileges through role hierarchies, identify sensitive objects that are broadly accessible, and explain each finding with the exact grants that support it. Do not modify anything.
```

## 7. Make agents fight (multi-agent orchestration)
```text
Investigate the current Snowflake environment using specialist subagents.

Create separate investigators for:
1. Security
2. FinOps
3. Performance
4. Data Quality
5. Governance
6. Architecture

Allow them to investigate independently. Then act as the lead investigator and reconcile their findings. If two investigators disagree, do not choose one arbitrarily. Run additional investigation to determine which conclusion is supported by evidence.

Return only findings that can be demonstrated with evidence.
```

## 8. Discover something and then fix it
```text
Find one real, reproducible engineering problem in this Snowflake environment.

First investigate and prove the problem.

Then create a remediation plan.

Do not change anything yet.

After presenting the plan, wait for approval.
```
Then, after approval:
```text
Execute the approved remediation. After making the change, independently verify that the original problem is resolved and run regression checks to ensure you did not introduce a new problem.
```

## 9. Investigate its own answer (adversarial self-review)
```text
Analyze this environment and give me your 5 most important findings.

Now assume your first answer may be wrong.

Create an adversarial review of your own findings. Attempt to disprove every finding using additional Snowflake queries and metadata. Remove anything that cannot be independently verified.
```

## 10. "I don't know what happened"
```text
Something changed in this Snowflake environment recently and I don't know what.

Investigate recent changes across objects, permissions, workloads, schemas and data pipelines. Identify unusual changes, correlate them with query and access history, and reconstruct the most likely sequence of events.

Separate confirmed evidence from inference.
```

## 11. Investigate a role like a forensic analyst
```text
Pick the most privileged non-admin role available to me.

Reconstruct exactly what this role can access, including inherited privileges. Identify the most sensitive objects it can reach and explain the privilege path that gives it access.

Do not make any changes.
```

## 12. Governance inventory
```text
Build a governance inventory of this Snowflake account.

For every important database, schema and object, determine:
- owner
- object type
- row count where available
- approximate size
- last activity
- grants
- dependent objects
- sensitivity indicators
- tags/classification where available

Highlight missing governance metadata rather than inventing values.
```

## 13. Find unused stuff
```text
Find Snowflake objects and warehouses that appear to be unused or underutilized.

Do not simply use creation dates.

Correlate object metadata with query/access history and warehouse workload. Distinguish genuinely unused resources from resources that are infrequently but legitimately used.
```

## 14. Write actual SQL for you
```text
Identify the 3 most valuable investigations you can perform against this Snowflake environment that I have not explicitly asked for.

For each investigation:
1. Explain why it matters.
2. Execute the investigation.
3. Show the SQL used.
4. Show the result.
5. Explain the conclusion.
```

## 15. Build a Snowflake architecture diagram
```text
Reverse engineer this Snowflake environment.

Infer the logical architecture from databases, schemas, tables, views, stages, transformations, query history and dependencies.

Then generate a clean architecture representation showing source, ingestion, transformation, serving and consumption layers.

Clearly distinguish observed relationships from inferred relationships.
```

---

## The "HOLY SH*T" command
```text
Act as the lead data engineer for this Snowflake environment.

Your objective is to discover something I don't already know.

You have permission to investigate the environment using the tools available to you.

Do not wait for me to specify what to inspect.

Spawn specialist subagents where useful.

Have them independently investigate:
- security
- cost
- performance
- data quality
- governance
- architecture
- operational anomalies

Then consolidate their findings.

For every finding:
1. Explain what you discovered.
2. Show the evidence.
3. Show the SQL or tool investigation that produced the evidence.
4. Determine the likely root cause.
5. Explain the business/engineering impact.
6. Identify whether it is confirmed or inferred.
7. Recommend the next action.

Do not manufacture findings.

Your goal is to tell me something about this Snowflake environment that I would not have discovered by simply looking at the dashboard.
```
Then say nothing. Let CoCo work.

## Final command — autonomous remediation boundary
After it finishes:
```text
You have now investigated the environment.

Take your most significant confirmed finding.

Without asking me to explain the implementation, determine everything required to remediate it safely.

Inspect the relevant code, objects, dependencies and permissions.

Create a step-by-step remediation plan.

Identify every risk associated with the change.

Then tell me exactly which actions you can safely execute yourself and which actions require human approval.
```

This moves the demo from "it can answer questions" to "it can investigate, reason, delegate, gather evidence, understand dependencies, and determine the boundary between autonomous and human-controlled action."
