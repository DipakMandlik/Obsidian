---
type: cortex-code-hook
name: reindex-tasks
title: "Hook - Reindex Tasks"
source_repo: Snowflake-Labs/coco-skills
source_path: plugins/xo/hooks/reindex-tasks.js
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/reindex-tasks.js
category: hook
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - hook
  - reindex-tasks
aliases:
  - "Hook - Reindex Tasks"
  - "reindex-tasks"
---

# Hook - Reindex Tasks

**Type:** Hook
**Hook Name:** `reindex-tasks`
**Event:** `Internal hook & helper utility`
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `plugins/xo/hooks/reindex-tasks.js`
**Source URL:** [plugins/xo/hooks/reindex-tasks.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/reindex-tasks.js)
**Status:** Active
**Last Reviewed:** 2026-09-24

---

## Trigger

Triggered by SessionStart hook and whenever task files are modified, moved, or closed.

---

## Purpose

Scans $XOCORTEX_HOME/tasks/current/ and archive/, parses frontmatter status, and deterministically regenerates the root tasks/index.md Kanban/table.

---

## Execution Flow

Reads all WI-*.md files in tasks/current/ -> Parses state and status -> Sorts by priority and timestamp -> Writes updated tasks/index.md atomically.

---

## Inputs

Markdown files in tasks directory.

---

## Outputs

Regenerated tasks/index.md markdown file.

---

## Side Effects

Overwrites tasks/index.md with fresh task status index.

---

## Dependencies

- plugins/xo/hooks/_common.js
- [[Plugin - XO]]

---

## Configuration

`⚙️ CONFIG`

```json
node ${CORTEX_PLUGIN_ROOT}/hooks/reindex-tasks.js
```

---

## Copy-Ready Configuration & Implementation

`🪝 HOOK` `✅ COPY-READY`

```javascript
'use strict';
const fs = require('fs');
const path = require('path');
const { isoUtc, atomicWrite, appendLog } = require('./_common.js');

function reindexTasks(tasksDir, logFile, shortSid) {
    const currentDir = path.join(tasksDir, 'current');
    const archiveDir = path.join(tasksDir, 'archive');
    const indexFile = path.join(tasksDir, 'index.md');

    if (!fs.existsSync(currentDir)) return;

    const ts = isoUtc();

    const stateOrd = { Now: 1, Next: 2, Later: 3 };
    const statusOrd = s => (s && s.startsWith('InProgress') ? 1 : s && s.startsWith('Shipped') ? 2 : s === 'Ready' ? 3 : 4);

    const entries = [];
    const wiGlob = fs.readdirSync(currentDir).filter(n => /(?:^|-)wi\d+.*\.md$/.test(n));
    for (const fname of wiGlob) {
        const fpath = path.join(currentDir, fname);
        if (!fs.statSync(fpath).isFile()) continue;
        const base = path.basename(fname, '.md');
        const wiNumMatch = base.match(/(?:^|-)wi0*(\d+)/);
        const wiNum = wiNumMatch ? parseInt(wiNumMatch[1], 10) : 0;

        const lines = fs.readFileSync(fpath, 'utf8').split('\n');
        let title = '', state = '', wi_status = '', status_note = '';
        for (const line of lines) {
            if (!title && /^# WI-/.test(line)) {
                title = line.replace(/^# WI-\d+: /, '');
            } else if (/^- Priority: /.test(line)) {
                state = line.replace(/^- Priority: /, '');
            } else if (/^- Status: /.test(line)) {
                wi_status = line.replace(/^- Status: /, '');
            } else if (/^- StatusNote: /.test(line)) {
                status_note = line.replace(/^- StatusNote: */, '');
                break;
            } else if (/^- Blocker: /.test(line)) {
                break;
            }
        }

        const so = stateOrd[state] ?? 3;
        const sto = statusOrd(wi_status);
        const wiPad = String(wiNum).padStart(4, '0');
        entries.push({ so, sto, wiPad, wiNum, state, wi_status, status_note, title });
    }

    entries.sort((a, b) => a.so - b.so || a.sto - b.sto || a.wiPad.localeCompare(b.wiPad));

    const archiveEntries = [];
    if (fs.existsSync(archiveDir)) {
        const archiveDirs = fs.readdirSync(archiveDir).filter(n => {
            return fs.statSync(path.join(archiveDir, n)).isDirectory();
        });
        for (const subdir of archiveDirs) {
            const subdirPath = path.join(archiveDir, subdir);
            const files = fs.readdirSync(subdirPath).filter(n => /(?:^|-)wi\d+.*\.md$/.test(n));
            for (const fname of files) {
                const fpath = path.join(subdirPath, fname);
                if (!fs.statSync(fpath).isFile()) continue;
                const base = path.basename(fname, '.md');
                const wiNumMatch = base.match(/(?:^|-)wi0*(\d+)/);
                const wiNum = wiNumMatch ? parseInt(wiNumMatch[1], 10) : 0;
                const wiPad = String(wiNum).padStart(4, '0');
                const lines = fs.readFileSync(fpath, 'utf8').split('\n');
                let title = '';
                for (const line of lines) {
                    if (/^# WI-/.test(line)) {
                        title = line.replace(/^# WI-\d+: /, '');
                        break;
                    }
                }
                archiveEntries.push({ wiPad, wiNum, title });
            }
        }
    }
    archiveEntries.sort((a, b) => a.wiPad.localeCompare(b.wiPad));

    const header = [
        '---',
        `generated: ${ts}`,
        `active_count: ${entries.length}`,
        `archived_count: ${archiveEntries.length}`,
        '---',
        '',
        '# Work Items Index',
        '',
        '| WI | Priority | Status | StatusNote | Description |',
        '|-----|-------|--------|------------|-------------|',
    ].join('\n');

    const activeRows = entries
        .map(e => `| ${e.wiNum} | ${e.state} | ${e.wi_status} | ${e.status_note} | ${e.title} |`)
        .join('\n');

    let content = header + '\n' + activeRows + '\n';

    if (archiveEntries.length > 0) {
        const archiveRows = archiveEntries
            .map(e => `| ${e.wiNum} | - | Done | | ${e.title} |`)
            .join('\n');
        content += '\n### Archived\n| WI | Priority | Status | StatusNote | Description |\n|-----|-------|--------|------------|-------------|\n' + archiveRows + '\n';
    }

    atomicWrite(indexFile, content);

    if (logFile && shortSid) {
        appendLog(logFile, `[${isoUtc()}] [${shortSid}] reindexed ${entries.length} active + ${archiveEntries.length} archived work items -> ${indexFile}`);
    }
}

module.exports = { reindexTasks };
```

---

## Source

- **Repository File:** `plugins/xo/hooks/reindex-tasks.js`
- **GitHub URL:** [Snowflake-Labs/coco-skills/plugins/xo/hooks/reindex-tasks.js](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/plugins/xo/hooks/reindex-tasks.js)
- **Manifest Reference:** `plugins/xo/hooks/hooks.json`

---

## Official Repository Knowledge

Hooks in Cortex Code run in a non-interactive shell environment (`$SHELL -c`). They provide behavioral guardrails, automatic documentation, and operational monitoring without human intervention.

---

## My Operational Notes

- Ensure `node` is located on the global environment `PATH` so hook scripts execute without failure.
- Review `tasks/index.md` regularly to track active and archived work items across projects.
