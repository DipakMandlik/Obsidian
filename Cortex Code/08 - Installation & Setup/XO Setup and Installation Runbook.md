---
type: cortex-code-setup
title: "XO Setup and Installation Runbook"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/SETUP.md
category: setup
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - setup
  - xo
aliases:
  - "XO Setup Runbook"
  - "Install XO Plugin"
---

# XO Setup and Installation Runbook

**Type:** Installation & Setup Runbook
**Plugin:** `xo`
**Version:** `3.3.1`
**Source:** Snowflake-Labs/coco-skills
**Last Reviewed:** 2026-09-24

---

## Overview

XO provides a comprehensive operator workflow system (OODA loop), durable memory, and specialized subagents for Cortex Code. While the Cortex Code plugin catalog registers the plugin, running the bundled `setup.sh` script is required to configure environment variables, hook PATH resolution, and optionally create a private git-backed memory vault.

---

## Prerequisites

Ensure the following CLI tools are installed and present on your system `PATH`:
- `node` (v18+) — Required for executing XO JavaScript hooks
- `git` — Required for git-backed memory vault versioning
- `gh` — GitHub CLI (authenticated via `gh auth login`)
- `jq` — Lightweight JSON processor

---

## Installation Options

`✅ COPY-READY`

### Path A: Git-Backed Memory Vault (Recommended for Engineering Teams)

Creates a private repository named `xocortex` under your GitHub account, clones it locally, registers the plugin, configures `XOCORTEX_HOME`, and sets up hooks.

```bash
# 1. Clone official repository
git clone https://github.com/Snowflake-Labs/coco-skills.git
cd coco-skills/plugins/xo

# 2. Run initialization
./setup.sh init
```

### Path B: Standard Catalog Install

If you installed XO directly via Cortex Code Desktop / CLI catalog:

```bash
# Execute bundled installer from plugin catalog cache
~/.snowflake/cortex/plugins/xo/setup.sh install
```

### Path C: Custom Directory Setup

```bash
# Specify custom vault location and workspace
./setup.sh install --xocortex "$HOME/my-vault/xocortex" --target user-profile
```

---

## Environment Variable Configuration

Cortex Code executes hooks using `$SHELL -c` (non-interactive shell). Therefore, `XOCORTEX_HOME` must be declared in non-interactive shell configuration files:

| Shell | Target File | Export Command |
|---|---|---|
| **Zsh** | `~/.zshenv` | `echo 'export XOCORTEX_HOME="$HOME/xocortex"' >> ~/.zshenv` |
| **Bash** | `~/.bash_profile` or `/etc/environment` | `echo 'export XOCORTEX_HOME="$HOME/xocortex"' >> ~/.bash_profile` |
| **Fish** | `~/.config/fish/config.fish` | `set -Ux XOCORTEX_HOME $HOME/xocortex` |

---

## Verification & Health Check

Run the built-in status check to verify plugin registration and variable resolution:

```bash
./setup.sh status
```

Expected output:
- Plugin registered: Yes
- `XOCORTEX_HOME` resolves: Valid path
- Node executable: Found
- Hooks executable: Ready

---

## Related Notes

- [[Plugin - XO]]
- [[Skill - XO Operator Workflow]]
- [[Workflow - XO Operator OODA Loop]]
