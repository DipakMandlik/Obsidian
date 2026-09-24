---
type: cortex-code-setup
title: "Prerequisites and Troubleshooting Guide"
source_repo: Snowflake-Labs/coco-skills
category: setup
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - troubleshooting
aliases:
  - "CoCo Troubleshooting"
---

# Prerequisites and Troubleshooting Guide

**Type:** Troubleshooting Runbook
**Last Reviewed:** 2026-09-24

---

## Common Issues & Verified Resolutions

### 1. Hook Fails with "command not found: node"
- **Cause:** Cortex Code executes hooks using `$SHELL -c`, which bypasses interactive profile files (`.zshrc`, `.bashrc`).
- **Fix:** Add Node.js to your system-wide path or `~/.zshenv`:
  ```bash
  echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshenv
  ```

### 2. tgrep Returns HTTP 403 Forbidden
- **Cause:** `tgrep` uses Cortex `arctic-embed` model. If your Snowflake account does not have model privileges enabled in the current region, Snowflake rejects the embedding request.
- **Fix:** XO automatically falls back to ripgrep (`grep`). To silence the warning, set `tgrep.enabled: false` in your Cortex Code configuration or grant Cortex model access.

### 3. Snowpark Connect (SCOS) Module Missing
- **Cause:** Workload validation requires `snowpark_connect` runtime.
- **Fix:** Install via `uv` or `pip`:
  ```bash
  pip install snowpark-connect
  ```

### 4. Privilege Gaps on Cortex Search or Semantic Models
- **Cause:** Role executing the skill lacks `USAGE` on Cortex Search service or database/schema.
- **Fix:** Run [[Skill - Audit Cortex Agent Access]] to identify missing grants and auto-generate the exact remediation SQL.

---

## Related Notes

- [[XO Setup and Installation Runbook]]
- [[Cortex Code Skills Installation Guide]]
