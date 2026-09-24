---
type: cortex-code-prompts
title: "Plugin Installation Commands"
category: commands
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - plugin
  - installation
---

# Plugin Installation Commands

**Type:** Copy-Ready Plugin Commands
**Last Reviewed:** 2026-09-24

---

## XO Plugin Installation

`✅ COPY-READY`

```bash
# Full Git-Backed Initialization (Creates private vault repo and registers plugin)
git clone https://github.com/Snowflake-Labs/coco-skills.git
cd coco-skills/plugins/xo
./setup.sh init

# Standard Workspace Install
./setup.sh install --target workspace

# Inspect Installation Status
./setup.sh status

# Set Environment Variable Manually (Zsh)
echo 'export XOCORTEX_HOME="$HOME/xocortex"' >> ~/.zshenv
source ~/.zshenv
```
