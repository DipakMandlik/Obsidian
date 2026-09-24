---
type: cortex-code-skill
name: setup-openflow-privatelink
title: "OpenFlow PrivateLink Setup"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/openflow-spcs-privatelink/SKILL.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/openflow-spcs-privatelink/SKILL.md
category: networking-spcs
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - skill
  - networking
aliases:
  - "OpenFlow PrivateLink Setup"
  - "setup-openflow-privatelink"
---

# OpenFlow PrivateLink Setup

**Type:** Skill
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/openflow-spcs-privatelink/SKILL.md`
**Source URL:** [skills/openflow-spcs-privatelink/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/openflow-spcs-privatelink/SKILL.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

Set up AWS PrivateLink between OpenFlow on SPCS and private data sources like RDS or on-prem databases.

Use when configuring private connectivity from Snowflake OpenFlow (running on SPCS) to AWS-hosted or on-premises data sources via AWS PrivateLink. Covers NLB + VPC Endpoint Service on AWS, then SYSTEM$PROVISION_PRIVATELINK_ENDPOINT, network rules, and External Access Integration on Snowflake. AWS only (Public Preview also exists on Azure but is out of scope here). Triggers: OpenFlow PrivateLink, SPCS private connectivity, NLB for OpenFlow, EAI for OpenFlow, SYSTEM$PROVISION_PRIVATELINK_ENDPOINT, OpenFlow RDS, OpenFlow MySQL, OpenFlow PostgreSQL, private ingestion OpenFlow, SPCS egress.

---

## When To Use

Set up AWS PrivateLink between OpenFlow on SPCS and private data sources like RDS or on-prem databases.

**Verified Triggers:** `OpenFlow PrivateLink, SPCS private connectivity, NLB for OpenFlow, EAI for OpenFlow, SYSTEM$PROVISION_PRIVATELINK_ENDPOINT, OpenFlow RDS, OpenFlow MySQL, OpenFlow PostgreSQL, private ingestion OpenFlow, SPCS egress`

---

## When Not To Use

Do not use when outside the designated Snowflake or Cortex Code domain boundary.

---

## What It Enables

This skill provides Cortex Code with deterministic, domain-specific execution patterns for **OpenFlow PrivateLink Setup**, enforcing safety, verification standards, and optimal Snowflake resource utilization without hallucinating commands or grants.

---

## Dependencies

- **Platform:** Snowflake Cortex Code CLI / Desktop
- **Category:** `networking-spcs`
- **Related Notes:** [[Cortex Code Skills Master Index]], [[CoCo Quick Access]], [[How to Choose a Cortex Code Skill]]

---

## Workflow

1. **Invocation:** Activate the skill explicitly via prompt or natural-language trigger.
2. **Context Inspection:** Inspect the environment, objects, and local files according to the skill instructions.
3. **Execution:** Execute deterministic actions or code generation following the skill protocol.
4. **Verification & Proof:** Validate outputs, grants, or generated code before concluding.

---

## Activation

```text
Help me connect OpenFlow on SPCS to my private RDS MySQL instance using AWS PrivateLink.
```

---

## Copy-Ready Skill

`✅ COPY-READY`

```markdown
---
name: setup-openflow-privatelink
title: OpenFlow PrivateLink Setup
summary: Set up AWS PrivateLink between OpenFlow on SPCS and private data sources like RDS or on-prem databases.
description: |
  Use when configuring private connectivity from Snowflake OpenFlow (running on SPCS) to AWS-hosted or on-premises data sources via AWS PrivateLink. Covers NLB + VPC Endpoint Service on AWS, then SYSTEM$PROVISION_PRIVATELINK_ENDPOINT, network rules, and External Access Integration on Snowflake. AWS only (Public Preview also exists on Azure but is out of scope here). Triggers: OpenFlow PrivateLink, SPCS private connectivity, NLB for OpenFlow, EAI for OpenFlow, SYSTEM$PROVISION_PRIVATELINK_ENDPOINT, OpenFlow RDS, OpenFlow MySQL, OpenFlow PostgreSQL, private ingestion OpenFlow, SPCS egress.
tools:
  - snowflake_sql_execute
  - Bash
  - Read
  - Write
  - Edit
prompt: Help me connect OpenFlow on SPCS to my private RDS MySQL instance using AWS PrivateLink.
language: en
status: Published
author: Snowflake Solutions Team
type: snowflake
---

## Overview

Connect OpenFlow on SPCS to private data sources (RDS, on-prem DBs reachable via Direct Connect/VPN) through AWS PrivateLink. The flow:

```
OpenFlow (SPCS) → EAI → Network Rule (PRIVATE_HOST_PORT)
                  → Outbound PrivateLink Endpoint
                  → AWS VPC Endpoint Service → Internal NLB
                  → Target Group(s) → RDS / on-prem DB (via TCP proxy if needed)
```

Each NLB listener uses a unique port (any port > 1024) so multiple instances sharing the same backend port (e.g., two MySQL on 3306) can be disambiguated.

## Prerequisites

- AWS permissions: NLB, target groups, endpoint services, security groups
- Snowflake `ACCOUNTADMIN`
- OpenFlow already deployed on SPCS
- One or more data sources in a private VPC (or on-prem with VPC connectivity)

## Phase 1 — AWS infrastructure

1. **Target groups (IP type)** — one per database, TCP on the DB port, registered with the private IP of each instance (or a TCP proxy EC2 IP if the source is on-prem with non-RFC1918 addresses).
2. **Internal NLB** — TCP listeners on unique custom ports (e.g., 3301, 3302), each forwarding to its target group. Enable cross-zone load balancing. Set `dns_record.client_routing_policy=any_availability_zone`.
3. **VPC Endpoint Service** — fronts the NLB. Keep manual acceptance enabled (default).

⚠️ STOPPING POINT: Confirm `describe-target-health` shows `healthy` and the NLB state is `active` before continuing. Record `nlb_dns_name`, `endpoint_service_name`, and listener ports.

On-prem targets outside RFC1918/RFC6598 ranges require a single TCP proxy EC2 (NGINX/HAProxy/Envoy in TCP mode) registered into the target groups.

## Phase 2 — Snowflake side

1. Get Snowflake's account principal:
   ```sql
   SELECT key, value FROM TABLE(FLATTEN(INPUT => PARSE_JSON(SYSTEM$GET_PRIVATELINK_CONFIG())));
   ```
   Add the `privatelink-account-principal` ARN to the endpoint service's allowed principals via `aws ec2 modify-vpc-endpoint-service-permissions`.

2. Provision the endpoint:
   ```sql
   USE ROLE ACCOUNTADMIN;
   SELECT SYSTEM$PROVISION_PRIVATELINK_ENDPOINT('<endpoint_service_name>', '<nlb_dns_name>');
   ```

3. Accept the pending connection in AWS (`aws ec2 accept-vpc-endpoint-connections`).

⚠️ STOPPING POINT: Run `SELECT SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO();` and wait until `status` is `available` (30–40s) before creating the network rule.

4. Network rule + EAI:
   ```sql
   CREATE OR REPLACE NETWORK RULE <rule_name>
     MODE = EGRESS
     TYPE = PRIVATE_HOST_PORT
     VALUE_LIST = ('<nlb_dns_name>:<port1>', '<nlb_dns_name>:<port2>');

   CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION <eai_name>
     ALLOWED_NETWORK_RULES = (<rule_name>)
     ENABLED = TRUE;
   ```

5. In OpenFlow UI: Runtime console → `...` → **Associate External Access Integration** → pick `<eai_name>`.

6. Validate by pointing a connector at `<nlb_dns_name>:<listener_port>`.

## Common Mistakes

- **Reusing a backend port as the NLB listener port** when multiple instances share it — listeners must be unique.
- **Uppercase letters in `VALUE_LIST`** — the NLB FQDN must be lowercase, otherwise `Invalid VALUE_LIST`.
- **Forgetting to accept the AWS endpoint connection** — the endpoint stays in `pendingAcceptance` forever.
- **Skipping the account-principal allowlist** — `SYSTEM$PROVISION_PRIVATELINK_ENDPOINT` will fail.
- **Security group on RDS not allowing the NLB's SG** on the DB port — target health flips unhealthy.
- **Deleting the NLB before deprovisioning** the Snowflake endpoint — leaves a dangling endpoint.
- **On-prem IPs outside RFC1918/RFC6598** registered directly — NLB rejects; use a TCP proxy EC2 instead.
- **Using `PRIVATE_HOST_PORT` against the DB hostname** — use the NLB DNS name, not the database's.

## Troubleshooting quick reference

| Symptom | Check |
|---|---|
| Endpoint stuck `pendingAcceptance` | Accept it in AWS VPC > Endpoint Services |
| Targets unhealthy | RDS/proxy SG inbound from NLB SG on DB port |
| `Invalid VALUE_LIST` | NLB DNS must be lowercase |
| Endpoint `failed` | `SYSTEM$DEPROVISION_PRIVATELINK_ENDPOINT` then re-provision; `SYSTEM$RESTORE_PRIVATELINK_ENDPOINT` recovers within 7 days |
| Traffic drops despite healthy targets | VPC NACLs: inbound listener ports, outbound 1024–65535 |

## Cleanup

⚠️ STOPPING POINT: Confirm with the user before running any `DROP` / `delete-*` commands below.

Order matters — Snowflake first, then AWS:

1. OpenFlow UI: disassociate EAI from runtime.
2. `DROP EXTERNAL ACCESS INTEGRATION <eai_name>; DROP NETWORK RULE <rule_name>;`
3. `SELECT SYSTEM$DEPROVISION_PRIVATELINK_ENDPOINT('<endpoint_service_name>');`
4. `aws ec2 delete-vpc-endpoint-service-configurations`, then delete NLB, then target groups.

## Stopping Points

- Phase 1 — wait for healthy targets and active NLB before creating the endpoint service
- Phase 2 step 3 — wait until `SYSTEM$GET_PRIVATELINK_ENDPOINTS_INFO()` returns `available` before creating the network rule
- Cleanup — confirm with the user before running any destructive `DROP` / `delete-*` commands

## Cost note

NLB (hourly + LCU), VPC Endpoint Service (hourly), PrivateLink endpoint (hourly + per-GB), and cross-zone data transfer all incur AWS charges. Check current AWS pricing.
```

---

## Example Prompts

- `Help me connect OpenFlow on SPCS to my private RDS MySQL instance using AWS PrivateLink.`
- `OpenFlow PrivateLink`
- `SPCS private connectivity`
- `NLB for OpenFlow`

---

## Related

- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]
- [[How to Choose a Cortex Code Skill]]

---

## Official Source

- **Repository Path:** `skills/openflow-spcs-privatelink/SKILL.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/openflow-spcs-privatelink/SKILL.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/openflow-spcs-privatelink/SKILL.md)

---

## Version Tracking

- **Repository Commit:** `28b549f48da9994307081a9d4d2f16379f5a9c16`
- **Branch:** `main`
- **Sync Date:** 2026-09-24

---

## Official Repository Knowledge

This skill is directly extracted from the official Snowflake Labs Cortex Code skills repository (`Snowflake-Labs/coco-skills`). It reflects official Snowflake best practices and validated agent behaviors.

---

## My Operational Notes

- Store local artifacts generated by this skill in designated project subfolders.
- When running in production Snowflake accounts, ensure warehouse size and role privileges align with the operation.
- For multi-step workflows, verify intermediate outputs before proceeding to downstream phases.

---

## Client Demonstration Notes

- Highlight that Cortex Code executes verified, repository-backed skills rather than guessing.
- Demonstrate evidence capture and deterministic output validation live for clients.
