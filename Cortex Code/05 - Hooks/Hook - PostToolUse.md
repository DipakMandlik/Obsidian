---
type: cortex-code-hook
name: PostToolUse
title: "Hook - PostToolUse"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/hooks/posttooluse.js
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/posttooluse.js
category: hook
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - hook
  - posttooluse
aliases:
  - "Hook - PostToolUse"
  - "PostToolUse"
---

# Hook - PostToolUse

**Type:** Hook
**Hook Name:** `PostToolUse`
**Event:** `PostToolUse`
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo/hooks/posttooluse.js`
**Source URL:** [plugins/xo/hooks/posttooluse.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/posttooluse.js)
**Status:** Active
**Last Reviewed:** 2026-09-24

---

## Trigger

Fires immediately after Cortex Code executes any tool (read, write, bash, sql, etc.).

---

## Purpose

Monitors context expansion resulting from large tool outputs and warns the operator before context window exhaustion forces auto-compaction.

---

## Execution Flow

Reads tool execution payload -> Checks context token estimate -> Evaluates delta -> If threshold is crossed, logs event and surfaces advisory message.

---

## Inputs

Tool execution result payload (tool_name, parameters, output_size).

---

## Outputs

JSON response { "continue": true } or warning.

---

## Side Effects

Updates context state file in $XOCORTEX_HOME/state.

---

## Dependencies

- node, plugins/xo/hooks/_common.js, plugins/xo/hooks/_context-monitor.js
- [[Plugin - XO]]

---

## Configuration

`⚙️ CONFIG`

```json
{\n  "type": "command",\n  "command": "node ${CORTEX_PLUGIN_ROOT}/hooks/posttooluse.js"\n}
```

---

## Copy-Ready Configuration & Implementation

`🪝 HOOK` `✅ COPY-READY`

```javascript
#!/usr/bin/env node
'use strict';
const { readInput, buildPaths } = require('./_common.js');
const { checkContext } = require('./_context-monitor.js');

function main() {
    let input;
    try {
        input = readInput();
    } catch (e) {
        process.stdout.write('{"continue": true}');
        return;
    }

    const ctx = buildPaths(input);
    const { SHORT_SID, LOG_FILE, CONTEXT_STATE_DIR, SESSION_ID } = ctx;

    const r = checkContext(input, CONTEXT_STATE_DIR, SESSION_ID, {
        promptTokensEst: 0,
        responseBuffer: 0,
        hookSource: 'posttool',
        logFile: LOG_FILE,
        shortSid: SHORT_SID,
        incrementTurns: false,
    });

    // When a skill loads, append a short reminder that the skill is domain knowledge
    // for the current XO stage, not a replacement for the XO process. This lands at the
    // tail of the skill tool result (after the skill body), so it is a supplement to the
    // stronger reminder the UserPromptSubmit hook emits at the top of the next turn.
    const skillNudge = input.tool_name === 'skill'
        ? `XO [silent]: you just loaded skill '${(input.tool_input && input.tool_input.command) || '?'}' as domain knowledge for your CURRENT XO stage/pathway — it does not replace the XO process. Keep your stage, gates, and recording in force; if the skill's instructions pulled your focus, re-read your task/pathway before proceeding. Surface nothing about this note itself.`
        : '';

    let checkpointMsg = '';
    if (r.result === 'fire') {
        switch (r.level) {
            case 'routine':
                checkpointMsg = `Context ${r.projectedPct}% (trigger: ${r.trigger}; output since last ${r.outputSinceLast}, turns ${r.turnsSinceLast}) — consider checkpointing at your next milestone.`;
                break;
            case 'urgent':
                checkpointMsg = `Context ${r.projectedPct}% [urgent] (trigger: ${r.trigger}; output since last ${r.outputSinceLast}, turns ${r.turnsSinceLast}) — write a checkpoint soon. Update your notes file and task file.`;
                break;
            case 'critical':
                checkpointMsg = `Context ${r.projectedPct}% [critical] (trigger: ${r.trigger}; output since last ${r.outputSinceLast}, turns ${r.turnsSinceLast}) — checkpoint NOW. Write notes file, task file, and diary before continuing.`;
                break;
            case 'final':
                checkpointMsg = `Context ${r.projectedPct}% [FINAL] (trigger: ${r.trigger}; output since last ${r.outputSinceLast}, turns ${r.turnsSinceLast}) — compaction imminent. STOP and write checkpoint immediately: notes file, task file, diary, /memories/.`;
                break;
            default:
                checkpointMsg = `Context ${r.projectedPct}% [${r.level}] — checkpoint may be needed.`;
        }
    }

    const parts = [skillNudge, checkpointMsg].filter(Boolean);
    if (parts.length === 0) {
        process.stdout.write('{"continue": true}');
        return;
    }

    process.stdout.write(JSON.stringify({ additionalContext: parts.join('\n\n') }));
}

main();
```

---

## Source

- **Repository File:** `plugins/xo/hooks/posttooluse.js`
- **GitHub URL:** [Snowflake-Labs/coco-skills/plugins/xo/hooks/posttooluse.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/posttooluse.js)
- **Manifest Reference:** `plugins/xo/hooks/hooks.json`

---

## Official Repository Knowledge

Hooks in Cortex Code run in a non-interactive shell environment (`$SHELL -c`). They provide behavioral guardrails, automatic documentation, and operational monitoring without human intervention.

---

## My Operational Notes

- Ensure `node` is located on the global environment `PATH` so hook scripts execute without failure.
- Review `tasks/index.md` regularly to track active and archived work items across projects.
