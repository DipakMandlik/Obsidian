---
type: cortex-code-hook
name: UserPromptSubmit
title: "Hook - UserPromptSubmit"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/hooks/userpromptsubmit.js
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/userpromptsubmit.js
category: hook
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - hook
  - userpromptsubmit
aliases:
  - "Hook - UserPromptSubmit"
  - "UserPromptSubmit"
---

# Hook - UserPromptSubmit

**Type:** Hook
**Hook Name:** `UserPromptSubmit`
**Event:** `UserPromptSubmit`
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo/hooks/userpromptsubmit.js`
**Source URL:** [plugins/xo/hooks/userpromptsubmit.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/userpromptsubmit.js)
**Status:** Active
**Last Reviewed:** 2026-09-24

---

## Trigger

Fires every time the user submits a prompt before the model begins reasoning.

---

## Purpose

Logs prompt submission timestamp to the session diary, checks accumulated context size against warning thresholds, and injects context pressure warnings if compaction is imminent.

---

## Execution Flow

Reads user prompt payload -> Appends prompt record to diary -> Calls checkContext() -> Evaluates token thresholds -> Returns continue: true.

---

## Inputs

Prompt payload (prompt text, session_id, conversation turn count).

---

## Outputs

JSON response { "continue": true } with optional warning message.

---

## Side Effects

Appends to local session diary, writes to session context state log.

---

## Dependencies

- node, plugins/xo/hooks/_common.js, plugins/xo/hooks/_context-monitor.js
- [[Plugin - XO]]

---

## Configuration

`⚙️ CONFIG`

```json
{\n  "type": "command",\n  "command": "node ${CORTEX_PLUGIN_ROOT}/hooks/userpromptsubmit.js",\n  "timeout": 5\n}
```

---

## Copy-Ready Configuration & Implementation

`🪝 HOOK` `✅ COPY-READY`

```javascript
#!/usr/bin/env node
'use strict';
const fs = require('fs');
const path = require('path');
const { readInput, localWallClock, buildPaths } = require('./_common.js');
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
    const { SHORT_SID, LOG_FILE, CONTEXT_STATE_DIR, SESSION_ID, DIARY_PATH } = ctx;

    if (DIARY_PATH) {
        try { fs.mkdirSync(path.dirname(DIARY_PATH), { recursive: true }); } catch (_) {}
    }

    const prompt = input.prompt ?? '';
    const promptLen = prompt.length;
    const promptTokensEst = Math.floor(promptLen / 4);
    const responseBuffer = 2000;

    const r = checkContext(input, CONTEXT_STATE_DIR, SESSION_ID, {
        promptTokensEst,
        responseBuffer,
        hookSource: 'userprompt',
        logFile: LOG_FILE,
        shortSid: SHORT_SID,
    });

    const localPromptLine = `TIME: User Local Time at Prompt Submit: ${localWallClock()}`;

    if (r.result !== 'fire') {
        // Not a checkpoint threshold. When a live reading is available, surface a minimal
        // SILENT context indicator every turn so the agent always has the CURRENT number and
        // never carries a stale checkpoint figure forward (e.g. after compaction frees the
        // window). Informational only — the wording forbids idle discussion of it.
        // Primary reminder that keeps XO discipline present through skill loads.
        // Conditional wording self-suppresses for reactive / no-pathway sessions.
        const xoLine = `XO [silent]: if you're working under an XO pathway, treat any skill you invoke as domain knowledge for your CURRENT stage — keep following your XO stages, gates, and recording; don't let a loaded skill replace the XO process. Surface nothing about this unless it yields a real action.`;
        if (r.reportedPct !== undefined) {
            const ctxLine = `CONTEXT [silent — informational; surface to the user ONLY when context is genuinely decision-relevant, never as idle chatter]: ~${r.reportedPct}% of the window used as of last turn. May jitter after compaction.`;
            process.stdout.write(JSON.stringify({ additionalContext: `${localPromptLine}\n\n${ctxLine}\n\n${xoLine}` }));
        } else {
            process.stdout.write(JSON.stringify({ additionalContext: `${localPromptLine}\n\n${xoLine}` }));
        }
        return;
    }

    const diaryHint = DIARY_PATH ? `Write your checkpoint to the diary at: ${DIARY_PATH}` : '';

    const factLine = `FACT: After the previous turn, context was ${r.reportedPct}% (${r.reportedTotal}/${r.contextWindow} tokens) — a point-in-time reading that falls after compaction; do not cite it on later turns. Last recorded was ${r.lastContextPct}% (delta: +${r.contextDelta}%). Output since last checkpoint: ${r.outputSinceLast} tokens. Turns since last checkpoint: ${r.turnsSinceLast}.`;
    const predictionLine = `PREDICTION: Your current prompt is approximately ${promptTokensEst} tokens. With an estimated ${responseBuffer}-token response buffer, projected context after this turn: ~${r.projectedPct}%.`;
    const recommendationLine = `RECOMMENDED ACTION: [${r.level}] checkpoint (triggered by: ${r.trigger}).`;

    let urgencyDetail, actionDetail;
    switch (r.level) {
        case 'routine':
            urgencyDetail = 'Record now. Append a diary line; if this session is tracked under a WI, also update notes + task NextAction. Then continue.';
            actionDetail = `Before your next tool call:\n1. DIARY (append) — always, even for a lightweight WI-less session: ${diaryHint}\n   Format: ## [${SHORT_SID}] <short description> (add \`WI-N:\` right after the session id only if a WI is being tracked; 5-10 lines max)\n2. Only if this session is tracked under a WI: NOTES (notes/{YYYY-MM}/wi{N}-*.md) — current understanding, decisions, what to try next; and TASK (tasks/current/wi{N}-*.md) — Status + NextAction.\nSee the \`save-protocol\` reference in the \`xo\` skill for the full recording protocol.\nThen continue with the user's request.`;
            break;
        case 'urgent':
            urgencyDetail = 'Checkpoint overdue. Write NOW before continuing work.';
            actionDetail = `Checkpoint NOW.\n1. DIARY (append) — always, even for a lightweight WI-less session: ${diaryHint}\n   Format: ## [${SHORT_SID}] description (add \`WI-N:\` after the session id only if a WI is being tracked; 5-10 lines max)\n2. Only if this session is tracked under a WI: NOTES (notes/{YYYY-MM}/wi{N}-*.md) — write thoroughly, a recovering agent reads this first; and TASK (tasks/current/wi{N}-*.md) — Status, StatusNote, NextAction.\nAlso update /memories/ with recovery context.\nSee the \`save-protocol\` reference in the \`xo\` skill for the full protocol if needed.`;
            break;
        case 'critical':
            urgencyDetail = 'Context is nearly full. STOP current work and write IMMEDIATELY. Compaction is imminent.';
            actionDetail = `STOP. Checkpoint IMMEDIATELY.\n1. DIARY (append) — always, even for a lightweight WI-less session: ${diaryHint}\n   Format: ## [${SHORT_SID}] description (add \`WI-N:\` after the session id only if a WI is being tracked; 5-10 lines max)\n2. Only if this session is tracked under a WI: NOTES (notes/{YYYY-MM}/wi{N}-*.md) — write EVERYTHING: all reasoning, decisions, evidence, what works, what doesn't, what to try next. A cold-start agent must continue from this alone. TASK (tasks/current/wi{N}-*.md) — exact Status, StatusNote, NextAction.\nUpdate /memories/ with full recovery context.`;
            break;
        case 'final':
            urgencyDetail = 'Context will likely exceed 90% after this turn, triggering compaction. LAST CHANCE to preserve state. Do this BEFORE responding to the user.';
            actionDetail = `LAST CHANCE — checkpoint BEFORE doing anything else.\n1. DIARY (append) — always, even for a lightweight WI-less session: ${diaryHint}\n   Format: ## [${SHORT_SID}] description (add \`WI-N:\` after the session id only if a WI is being tracked; 5-10 lines max)\n2. Only if this session is tracked under a WI: NOTES (notes/{YYYY-MM}/wi{N}-*.md) — write your COMPLETE knowledge state. Everything. This file IS your memory through compaction. TASK (tasks/current/wi{N}-*.md) — exact Status, StatusNote, NextAction.\nUpdate /memories/ with everything needed to resume from zero context.`;
            break;
        default:
            urgencyDetail = '';
            actionDetail = '';
    }

    const processCheckLine = `PROCESS CHECK (silent): confirm you're still following the XO process — save first, plus the stages, slices, and reflexive gates (cold critic / Prove / Review on produced work) the work warrants. Consult the task's Provisional Pathway if set: when the current stage looks complete — especially right after delivering or shipping something — do your recording (advance the (here) marker), then offer the natural next stage once. Offer any skipped stage or gate once, unless the user already declined it. If this session has become substantive but is NOT tracked under a work item, treat that as a skipped gate: still append the diary line as above, and also offer the user a work item (Work Item Gate options, including "keep it lightweight — no WI"); offer once, and if the user declines, record the decline, keep diarising the session, and don't re-ask. Keep this private — surface nothing unless it yields a real offer.`;

    const msg = `${localPromptLine}\n\nCHECKPOINT [${r.level}] — Session ${SHORT_SID}\n\n${factLine}\n\n${predictionLine}\n\n${recommendationLine}\n${urgencyDetail}\n\n${actionDetail}\n\n${processCheckLine}`;

    process.stdout.write(JSON.stringify({ additionalContext: msg }));
}

main();
```

---

## Source

- **Repository File:** `plugins/xo/hooks/userpromptsubmit.js`
- **GitHub URL:** [Snowflake-Labs/coco-skills/plugins/xo/hooks/userpromptsubmit.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/userpromptsubmit.js)
- **Manifest Reference:** `plugins/xo/hooks/hooks.json`

---

## Official Repository Knowledge

Hooks in Cortex Code run in a non-interactive shell environment (`$SHELL -c`). They provide behavioral guardrails, automatic documentation, and operational monitoring without human intervention.

---

## My Operational Notes

- Ensure `node` is located on the global environment `PATH` so hook scripts execute without failure.
- Review `tasks/index.md` regularly to track active and archived work items across projects.
