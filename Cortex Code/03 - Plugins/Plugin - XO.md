---
type: cortex-code-plugin
name: xo
title: "XO Operator Workflow System"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo
source_url: https://github.com/Snowflake-Labs/coco-skills/tree/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo
category: operator-harness
version: 3.3.1
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - plugin
  - xo
  - workflow
aliases:
  - "XO Plugin"
  - "XO Operator Workflow"
  - "XO Harness"
---

# XO Operator Workflow System

**Type:** Cortex Code Plugin
**Plugin Name:** `xo`
**Version:** `3.3.1`
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo`
**Source URL:** [plugins/xo](https://github.com/Snowflake-Labs/coco-skills/tree/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Purpose

XO (*/ˈɛksoʊ/* — from *exocortex*, "smarter rocks") is an operator workflow system (an "AI harness") for **Cortex Code** — Desktop and CLI. It helps an AI agent operate as a reliable partner across multi-session, multi-step knowledge work — documentation, problem-solving, analysis, coding, testing, and research — with structure, durable memory, and human supervision.

Key superpowers encoded into XO:
1. **Automatic, durable memory:** Session-recording hooks keep a local markdown trail (`diary/`, `notes/`, `tasks/`, checkpoints) so context survives summarisation and compaction, and work resumes cleanly across sessions. Every significant session receives a local Work Item (`WI-*`) number.
2. **Composable OODA stages:** A builder's tool belt mapped to an OODA loop (**Observe → Orient → Decide → Act**), ensuring the agent reaches for the right step at the right moment: never forcing heavyweight process onto a small job, never skipping discipline on a large one.
3. **Specialised subagents:** Proposer / critic / verifier roles for scoped, reviewable, auditable work.

---

## Repository Path

`plugins/xo/`

---

## Manifest

`🧩 PLUGIN` `⚙️ CONFIG`

```json
{
  "name": "xo",
  "description": "XO operator workflow system for Cortex Code: composable workflow stages with quality gates, specialised subagents (proposer/critic/verifier), behavioural-enforcement hooks (automatic session recording, context checkpointing, and process guidance), and persistent cross-session memory. Prereqs: node, git, gh, jq. After install, run the bundled plugin/setup.sh to set XOCORTEX_HOME, put node on the hooks' PATH, and optionally create a git-backed vault.",
  "version": "3.3.1"
}
```

---

## Included Skills

- [[Skill - XO Operator Workflow]] (`plugins/xo/skills/xo/SKILL.md`) — The primary single entry point skill for the XO system. Triages requests, routes to stage references, and composes multi-stage execution paths.

---

## Included Agents

XO includes 4 specialised subagents designed to separate proposal generation from critical verification:

1. [[Agent - XO Bounded Writer]] (`plugins/xo/agents/xo-bounded-writer.agent.md`) — Write authority without codebase exploration; writes strictly from supplied findings and raw evidence; zero hallucination.
2. [[Agent - XO Cold Fast]] (`plugins/xo/agents/xo-cold-fast.agent.md`) — Fast, read-only surveyor and test runner for high-volume discovery and test execution.
3. [[Agent - XO Cold Smart]] (`plugins/xo/agents/xo-cold-smart.agent.md`) — High-intelligence evaluator, critic, and verifier; binary compliance gates and foundation checking.
4. [[Agent - XO Generator]] (`plugins/xo/agents/xo-generator.agent.md`) — Full write authority; generates code and tests strictly satisfying a bounded specification with scope discipline.

---

## Included Hooks

XO hooks enforce session recording, task reindexing, context threshold monitoring, and standing directives:

1. [[Hook - SessionStart]] (`plugins/xo/hooks/sessionstart.js`) — Initializes standing directives, reindexes active tasks, checks context state, and establishes local Work Item logging.
2. [[Hook - UserPromptSubmit]] (`plugins/xo/hooks/userpromptsubmit.js`) — Tracks prompt timestamps, records momentum to diary, and monitors token consumption thresholds.
3. [[Hook - PostToolUse]] (`plugins/xo/hooks/posttooluse.js`) — Evaluates context limits after each tool call to emit warnings before compaction.
4. [[Hook - Reindex Tasks]] (`plugins/xo/hooks/reindex-tasks.js`) — Automatically regenerates `tasks/index.md` whenever tasks change.
5. [[Hook - Architecture and Context Monitor]] (`plugins/xo/hooks/_context-monitor.js`) — Multi-tiered context health monitoring (`routine`, `urgent`, `critical`, `final`).

---

## Setup

Running the plugin's bundled `setup.sh` script configures the full operating environment:
- Creates a git-backed `xocortex` vault (if using `init`).
- Exports `XOCORTEX_HOME` into shell startup profiles (`~/.zshenv`, `~/.config/fish/config.fish`, or system environment).
- Ensures `node` is on the hook execution `PATH` so JavaScript hooks fire reliably.

---

## Installation

`✅ COPY-READY`

### Option 1: Git-Backed Vault (Recommended)
```bash
# 1. Clone official repository
git clone https://github.com/Snowflake-Labs/coco-skills.git

# 2. Run initialization (creates private xocortex repo on your GitHub and registers plugin)
coco-skills/plugins/xo/setup.sh init
```

### Option 2: Standard Catalog Install
```bash
# If installed via Cortex Code Catalog:
~/.snowflake/cortex/plugins/xo/setup.sh install
```

### Option 3: Manual Environment Setup
```bash
# Export the vault directory in your shell profile (e.g., ~/.zshenv or ~/.bashrc)
export XOCORTEX_HOME="$HOME/xocortex"
export PATH="/usr/local/bin:$PATH"
```

---

## Usage

Once installed, XO activates on any task. You can invoke it explicitly or use standard natural-language triggers:

```text
Start a work item for migrating our Databricks pipelines to Snowflake
```
```text
/xo investigate why query performance degraded on WAREHOUSE_ANALYTICS_XL
```
```text
Resume work on WI-14
```

### Vault Layout Structure
When initialized, XO maintains your work in `$XOCORTEX_HOME`:
```text
xocortex/
  diary/{YYYY-MM}/{YYYY-MM-DD}.md       session momentum log
  tasks/current/{project}-wi{N}-*.md    active work items
  tasks/archive/{YYYY-MM}/              completed work items
  notes/{YYYY-MM}/{project}-wi{N}-*.md  deep investigation notes
  tmp/                                  scratch space (gitignored)
```

---

## Copy-Ready Plugin Configuration

`⚙️ CONFIG`

### Hook Registration (`plugins/xo/hooks/hooks.json`)
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ${CORTEX_PLUGIN_ROOT}/hooks/sessionstart.js",
            "timeout": 15
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ${CORTEX_PLUGIN_ROOT}/hooks/userpromptsubmit.js",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ${CORTEX_PLUGIN_ROOT}/hooks/posttooluse.js"
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting

1. **Hooks not firing:** CoCo executes hooks using `$SHELL -c` (non-interactive). Ensure `node` and `XOCORTEX_HOME` are exported in `~/.zshenv` (Zsh) or system environment, not just in interactive `.zshrc` or `.bashrc`.
2. **tgrep search index:** XO utilizes Cortex Code's `tgrep` for semantic search if `tgrep.enabled` is active and account has `arctic-embed` model access. If 403 occurs, search falls back cleanly to `grep`.
3. **Verify status:** Run `setup.sh status` to verify plugin registration, path resolution, and environment variables.

---

## Source

- **Repository Directory:** `plugins/xo/`
- **GitHub URL:** [Snowflake-Labs/coco-skills/plugins/xo](https://github.com/Snowflake-Labs/coco-skills/tree/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo)
- **Manifest:** [plugin.json](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/.cortex-plugin/plugin.json)

---

## Official Repository Knowledge

XO represents Snowflake Labs' official operator workflow framework. It is artifact-neutral, applying equally well to code, documentation, Snowflake SQL objects, infrastructure as code, and data engineering pipelines.

---

## My Operational Notes

- Always initialize with a private GitHub repository (`setup.sh init`) so work item notes and architecture decisions sync across development machines.
- For high-stakes architectural changes, rely on the **Cold Smart Critic** to verify findings against the written spec before executing changes.

---

## Client Demonstration Notes

- Highlight how XO gives Cortex Code persistent memory, meaning it never "forgets" what was decided three turns ago or in yesterday's session.
- Demonstrate how subagents prevent hallucinations: the **Bounded Writer** literally lacks search tools, making it impossible for it to fabricate facts not in the research brief.
