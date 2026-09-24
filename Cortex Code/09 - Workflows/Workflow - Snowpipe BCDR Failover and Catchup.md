---
type: cortex-code-workflow
title: "Workflow - Snowpipe BCDR Failover and Catchup"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/snowpipe-bcdr/
category: data-engineering
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - workflow
  - bcdr
  - snowpipe
aliases:
  - "Snowpipe BCDR Workflow"
---

# Workflow - Snowpipe BCDR Failover and Catchup

**Type:** Disaster Recovery Operational Workflow
**Source:** Snowflake-Labs/coco-skills
**Primary Skill:** [[Skill - Snowpipe BCDR on Azure]]
**Last Reviewed:** 2026-09-24

---

## Workflow Overview

Provides step-by-step procedures for failing over Snowpipe ingestion on Azure ADLS Gen2 from a primary region to a secondary disaster recovery region, executing catchup ingestion, and safely failing back.

---

## Workflow Stages

1. **Pattern Selection:** Evaluate active-active, dual pipe, RA-GRS, or Failover Group replication.
2. **Failover Execution:** Halt ingestion on primary, update DNS/event grid subscriptions, activate secondary pipes.
3. **Catchup Ingestion:** Run timestamp-bounded catchup queries to load data queued during transition.
4. **Data Reconciliation:** Validate row count and deduplication across regions.
5. **Failback Procedure:** Re-establish replication and return primary pipes to active ingestion.

---

## Related Notes

- [[Skill - Snowpipe BCDR on Azure]]
