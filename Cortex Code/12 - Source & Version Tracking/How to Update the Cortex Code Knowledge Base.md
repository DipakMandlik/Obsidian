---
type: cortex-code-guide
title: "How to Update the Cortex Code Knowledge Base"
category: tracking
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - maintenance
  - update
aliases:
  - "Knowledge Base Update Guide"
---

# How to Update the Cortex Code Knowledge Base

**Type:** Maintenance Runbook
**Last Reviewed:** 2026-09-24

---

## When to Update

Run this update process whenever the official repository (`https://github.com/Snowflake-Labs/coco-skills`) publishes new commits, skills, or plugin versions.

---

## Step-by-Step Update Procedure

`✅ COPY-READY`

```bash
# Step 1: Fetch the latest changes from official repository
cd /tmp/coco-skills || git clone https://github.com/Snowflake-Labs/coco-skills.git /tmp/coco-skills
cd /tmp/coco-skills
git fetch origin main
NEW_COMMIT=$(git rev-parse origin/main)

# Step 2: Compare against recorded commit in COCO Knowledge Manifest
# Current Recorded Commit: 28b549f48da9994307081a9d4d2f16379f5a9c16
git log --oneline 28b549f48da9994307081a9d4d2f16379f5a9c16..$NEW_COMMIT

# Step 3: Identify structural changes
git diff --name-status 28b549f48da9994307081a9d4d2f16379f5a9c16..$NEW_COMMIT
```

### Update Protocol
1. **Preserve Personal Notes:** Never overwrite sections labeled `## My Operational Notes` or `## Client Demonstration Notes`.
2. **Update Official Blocks:** Refresh the `## Copy-Ready Skill` code fence from the updated repository file.
3. **Register New Capabilities:** Add newly added skills or agents to [[Cortex Code Skills Master Index]] and [[Cortex Code Skills Source Registry]].
4. **Update Manifest:** Update `repository_commit` and `last_sync` in [[COCO Knowledge Manifest]].

---

## Related Notes

- [[Cortex Code Skills Master Index]]
- [[COCO Knowledge Manifest]]
- [[Cortex Code Skills Source Registry]]
