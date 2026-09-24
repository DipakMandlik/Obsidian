---
type: cortex-code-setup
title: "Cortex Code Skills Installation Guide"
source_repo: Snowflake-Labs/coco-skills
source_path: README.md
category: setup
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - setup
  - skills
aliases:
  - "Skills Installation Guide"
  - "Install CoCo Skills"
---

# Cortex Code Skills Installation Guide

**Type:** Installation & Setup Guide
**Source:** Snowflake-Labs/coco-skills
**Last Reviewed:** 2026-09-24

---

## How Cortex Code Skills Work

Cortex Code skills are modular bundles of operational knowledge, deterministic SQL inspection scripts, specialized agent prompts, and reference architectures. Skills are discovered and loaded into Cortex Code sessions either globally (user profile) or locally (project workspace).

---

## Installation Methods

`✅ COPY-READY`

### Method 1: Local Project Workspace Install (Recommended)

Place skills in your project's `.snowflake/cortex/skills/` directory:

```bash
# Clone the official repo or copy required skill directories
git clone https://github.com/Snowflake-Labs/coco-skills.git

# Copy specific skill into current project
mkdir -p .snowflake/cortex/skills
cp -r coco-skills/skills/rbac .snowflake/cortex/skills/
cp -r coco-skills/skills/well-architected-framework-assessment .snowflake/cortex/skills/
```

### Method 2: Global User-Profile Install

Make skills available across all projects on your workstation:

```bash
# Create global skills directory
mkdir -p ~/.snowflake/cortex/skills

# Copy skills globally
cp -r coco-skills/skills/* ~/.snowflake/cortex/skills/
```

### Method 3: Cortex Plugin Installation

Plugins (such as `xo`) contain bundled skills, hooks, and agents:

```bash
mkdir -p ~/.snowflake/cortex/plugins
cp -r coco-skills/plugins/xo ~/.snowflake/cortex/plugins/
~/.snowflake/cortex/plugins/xo/setup.sh install
```

---

## Verifying Installed Skills

In Cortex Code CLI or Desktop, list available skills:

```text
/skills
```

---

## Related Notes

- [[Cortex Code Skills Master Index]]
- [[CoCo Command Cheat Sheet]]
- [[Prerequisites and Troubleshooting Guide]]
