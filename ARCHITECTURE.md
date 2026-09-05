# VibecOPS Architecture — Pillar 1: The Jules Auto-Merge Assembly Line

> **Historical design notice (2026-09-04):** The repo-local
> `auto-merge-jules.yml` workflow described by the original design is retired
> under T-163. Do not redeploy it. Current merge execution belongs to the
> centralized exact-head executor in `RxFit/rxfit-command-center` (PR #173).

> **Repository:** [RxFit/Jules_Auto_Merge](https://github.com/RxFit/Jules_Auto_Merge)
> **Status:** RETIRED — repo-local merge workflow superseded
> **Last Updated:** 2026-09-04
> **Maintained by:** Antigravity (local agent) + Jules (GitHub-connected agent)

---

## 1. Executive Summary

This document describes the historical **Pillar 1** VibecOPS workflow — the *Bite-Sizer Assembly Line*. It transforms Jules from an unconstrained, context-flooded marathon runner into a disciplined, **assembly-line worker** that executes one micro-task at a time, submits the result to the centralized reviewed merge path, marks it done only after success, and chains the next task automatically.

This pillar directly solves three critical failure modes:

| Problem | Root Cause | Pillar 1 Solution |
|---|---|---|
| Jules runs for 4 hours and crashes | Massive, unconstrained prompts blow context limits | `STATE.md` micro-scoping via the Bite-Sizer |
| Lost work on timeout/API error | No task checkpoint recovery | `STATE.md` is the persistent checkpoint in GitHub |
| "What's missing?" requires manual auditing | No live task dashboard | `STATE.md` is a real-time GitHub-hosted dashboard |
| Agent Wars (AI overwriting AI) | No concurrency guard | Git Collision Detector (Pre-Dispatch Hook) |
| Cost overruns on Jules compute | Redundant full-context re-feeds | Micro-prompts use a fraction of the token budget |

---

## 2. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       YOUR LOCAL MACHINE                        │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              antigravity_watchdog.js (PM2)               │  │
│  │                                                          │  │
│  │  ┌────────────┐     ┌──────────────────────────────┐    │  │
│  │  │  chokidar  │────▶│  handleFileEvent()            │    │  │
│  │  │  /handoff/ │     │                              │    │  │
│  │  └────────────┘     │  1. Parse ticket JSON        │    │  │
│  │                     │  2. PRE-HOOK: Collision Check │    │  │
│  │                     │  3. PRE-HOOK: Bite-Sizer      │    │  │
│  │                     │  4. Rewrite ticket → re-drop  │    │  │
│  │                     │  5. triggerDispatch()         │    │  │
│  │                     └──────────────┬───────────────┘    │  │
│  │                                    │                     │  │
│  │                     ┌──────────────▼───────────────┐    │  │
│  │                     │  dispatch_to_antigravity.js   │    │  │
│  │                     │  (calls Jules via GitHub API)│    │  │
│  │                     └──────────────┬───────────────┘    │  │
│  │                                    │                     │  │
│  │                     ┌──────────────▼───────────────┐    │  │
│  │                     │  POST-HOOK: Domino Effect     │    │  │
│  │                     │  - Mark [x] in STATE.md       │    │  │
│  │                     │  - Spawn next .json ticket    │    │  │
│  │                     └──────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   /handoff/          /logs/            /repos/App_N/           │
│   ├─ TSK_001.json   ├─ watchdog.log   └─ STATE.md             │
│   └─ TSK_002.json   └─ dispatch.lock                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                    GitHub API (Jules)
                              │
                  ┌───────────▼──────────┐
                  │   RxFit/App_1..12    │
                  │   - Jules PR/commit  │
                  │   - Reviewed merge   │
                  │   - STATE.md commit  │
                  └──────────────────────┘
```

---

## 3. The `STATE.md` Contract (Source of Truth)

Every one of the 12 repositories under management **MUST** contain a `STATE.md` file at the root level. This file is the only persistent memory the system requires.

### 3.1 Schema

```markdown
# FEATURE: [Short Feature Name]
**Status:** IN_PROGRESS | COMPLETE | BLOCKED
**Global Context:** [One paragraph describing the app and current feature goal]

## TASKS
- [x] TASK 1: [Completed task description]
- [x] TASK 2: [Completed task description]
- [ ] TASK 3: [Next task — this is what Jules will receive]
- [ ] TASK 4: [Future task]
- [ ] DAILY AUDIT: [Injected automatically by daily scheduler]
```

### 3.2 Rules

- Tasks are processed **strictly top-to-bottom**. The Bite-Sizer picks the first `- [ ]` line.
- The `**Global Context:**` block is always appended to Jules' micro-prompt so Jules always knows *why* it is doing what it is doing — just not *everything* it needs to do.
- Task text after `[ ]` is used as the **exact task key** for the mark-off operation. Do not use duplicate task descriptions.
- **Generated by Replit.** Use Replit to produce the initial `STATE.md` to reduce Antigravity compute cost. Paste the generated spec; Replit outputs the checklist.

### 3.3 Daily Audit Injection

A separate lightweight scheduler (Task Scheduler or PM2 cron) appends audit tasks at 2:00 AM every day:

```
- [ ] DAILY AUDIT: Check for broken links across all pages.
- [ ] DAILY AUDIT: Audit mobile padding on dashboard.
- [ ] DAILY AUDIT: Verify all API endpoints return 200.
```

When the watchdog loop wakes up, it processes these exactly like regular tasks.

---

## 4. The Handoff Ticket Schema

Every job delivered to the watchdog is a JSON file dropped into `/handoff/`. The Bite-Sizer requires one additional field beyond the original schema.

### 4.1 Required Fields

```json
{
  "TaskID": "TSK_20260516_001",
  "Target": "ANTIGRAVITY",
  "Status": "PENDING",
  "RepoPath": "C:\\AG_Workspaces\\repos\\App_4",
  "Prompt": "Build App 4's Stripe billing microservice."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `TaskID` | `string` | ✅ | Unique ID for deduplication and logging |
| `Target` | `string` | ✅ | Must be `"ANTIGRAVITY"` to be picked up |
| `Status` | `string` | ✅ | Must be `"PENDING"` to be dispatched |
| `RepoPath` | `string` | ✅ **NEW** | Absolute local path to the git repo. Used by Collision Detector and Bite-Sizer |
| `Prompt` | `string` | ✅ | Original high-level prompt. The Bite-Sizer preserves this as `_VibecOpsOriginal` |

### 4.2 Internal Fields (Added by Watchdog)

| Field | Description |
|---|---|
| `_VibecOpsOriginal` | The original prompt, preserved before micro-task injection |
| `_VibecOpsProcessed` | Boolean flag — set to `true` after hooks run to prevent re-processing |

### 4.3 Error States

| `Status` Value | Meaning | Resolution |
|---|---|---|
| `ERROR_COLLISION` | Agent War detected — too many edits on same files | Human review of git log required |
| `ERROR_DISPATCH` | `dispatch_to_antigravity.js` threw an error | Check logs; PM2 will retry on next restart |

---

## 5. The Three VibecOPS Hooks

### 5.1 Pre-Hook A — Git Collision Detector (Pillar 4 Preview)

**Trigger:** Fires on every new `PENDING` ticket before any dispatch.

**Logic:**

1. Checks if `RepoPath` contains a `.git` directory.
2. Runs `git log -n 15 --name-only` to get the list of files touched in the last 15 commits.
3. Counts occurrences per file. If any single file appears **4 or more times**, an Agent War is assumed.
4. Sets `ticket.Status = "ERROR_COLLISION"` and writes it back — **no dispatch occurs**.

**Why 4?** Three rapid edits to the same file is normal during active development. Four in 15 commits signals AI agents are overwriting each other.

```
Threshold: 4 edits on same file across last 15 commits → HALT
```

**Failure Mode:** If `git` is not installed or `RepoPath` has no `.git`, the check is skipped and execution continues (fail-open). This ensures a misconfigured `RepoPath` never blocks work.

---

### 5.2 Pre-Hook B — The Bite-Sizer

**Trigger:** Fires on every new `PENDING` ticket that passes the Collision Detector.

**Logic:**

1. Reads `STATE.md` from `RepoPath`.
2. Scans line-by-line for the first `- [ ]` entry using the regex `/^(\s*[-*]\s*\[\s*\]\s*)(.+)/`.
3. Extracts the task text (e.g., `TASK 3: Add signature verification`).
4. Overwrites `ticket.Prompt` with the micro-prompt template:

```
[VIBECOPS STRICT DIRECTIVE]
Your ONLY objective for this run is: TASK 3: Add signature verification to the webhook endpoint.
Do not touch anything outside of this scope. Output code, execute your operations, submit the result for reviewed merge, and exit.

---
Global Context:
[Original high-level prompt from _VibecOpsOriginal]
```

5. Saves the current task text to `activeJobs[]` for the Post-Hook.

**If no `STATE.md` exists:** The original prompt is sent to Jules unchanged. The system degrades gracefully.

---

### 5.3 Post-Hook — The Domino Effect

**Trigger:** Fires only when `dispatch_to_antigravity.js` exits with **no error** (success path).

**Logic:**

1. Iterates over `activeJobs[]` — the jobs that were dispatched in this cycle.
2. For each job, reads `STATE.md` and finds the line matching `[ ]` + `taskText`.
3. Replaces `[ ]` with `[x]` in-place and writes `STATE.md` back to disk.
4. Calls `getNextTask()` again. If a new `[ ]` task exists:
   - Generates a new ticket: `TSK_AUTO_<timestamp>.json`
   - Writes it to `/handoff/`
   - `chokidar` detects the new file within milliseconds and the cycle restarts automatically.
5. If `STATE.md` is fully checked off — logs `"STATE.md is 100% complete for [RepoPath]!"` and halts gracefully.

**Critical Safety:** The Domino Effect only fires on **successful dispatch**. A Jules API timeout or network error means `exec()` returns an error, the post-hook never fires, and the task is **never marked complete**. The existing ticket stays in `/handoff/` and will be re-processed on the next PM2 restart.

---

## 6. File & Directory Structure

```
C:\AG_Workspaces\
├── antigravity_watchdog.js       ← Main daemon (VibecOps Edition)
├── dispatch_to_antigravity.js    ← Jules dispatcher (unchanged)
├── handoff/                      ← Ticket drop zone (chokidar watches here)
│   ├── TSK_20260516_001.json
│   └── TSK_AUTO_1715900000.json  ← Auto-generated by Domino Effect
├── logs/
│   ├── antigravity_watchdog.log
│   └── antigravity_dispatch.lock
└── repos/
    ├── App_1/
    │   └── STATE.md              ← Per-repo task checklist
    ├── App_2/
    │   └── STATE.md
    └── ...App_12/
        └── STATE.md
```

---

## 7. Process Management (PM2)

The watchdog runs as a persistent PM2 process. Key operational commands:

```bash
# Start / Restart after updating watchdog.js
pm2 restart antigravity_watchdog

# View live logs
pm2 logs antigravity_watchdog

# View process status
pm2 status

# Save process list across reboots
pm2 save
pm2 startup
```

### PM2 Ecosystem Config (recommended)

```js
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'antigravity_watchdog',
    script: './antigravity_watchdog.js',
    watch: false,               // chokidar handles watching internally
    autorestart: true,
    max_memory_restart: '500M',
    env: {
      NODE_ENV: 'production'
    }
  }]
}
```

---

## 8. The Hardened `antigravity_watchdog.js` (Full Source)

```javascript
/**
 * antigravity_watchdog.js — Autonomous Daemon (VibecOps Edition)
 *
 * Objective: Monitors the handoff directory for PENDING tickets and
 * instantly wakes Antigravity by triggering dispatch_to_antigravity.js.
 * Powered by chokidar, managed by PM2.
 *
 * Upgrades Added:
 * - PILLAR 4: Git Collision Detector (Agent War Prevention)
 * - PILLAR 1: The Bite-Sizer (STATE.md Pre-Dispatch Micro-Scoping)
 * - PILLAR 1: The Domino Effect (STATE.md Post-Dispatch Auto-Advancement)
 */

const chokidar = require('chokidar');
const { exec, execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

const HANDOFF_DIR = path.join(__dirname, 'handoff');
const DISPATCH_SCRIPT = path.join(__dirname, 'dispatch_to_antigravity.js');
const LOG_FILE = path.join(__dirname, 'logs', 'antigravity_watchdog.log');
const LOCK_FILE = path.join(__dirname, 'logs', 'antigravity_dispatch.lock');

// Ensure directories exist
if (!fs.existsSync(HANDOFF_DIR)) fs.mkdirSync(HANDOFF_DIR, { recursive: true });
const logDir = path.dirname(LOG_FILE);
if (!fs.existsSync(logDir)) fs.mkdirSync(logDir, { recursive: true });

function log(msg) {
    const entry = `[${new Date().toISOString()}] [Watchdog] ${msg}\n`;
    fs.appendFileSync(LOG_FILE, entry);
    console.log(entry.trim());
}

// ============================================================================
// VIBECOPS PILLAR 4 & 1: PRE-DISPATCH & POST-DISPATCH HOOKS
// ============================================================================

function isCollisionDetected(repoPath) {
    try {
        if (!fs.existsSync(path.join(repoPath, '.git'))) return false;
        const gitLogCmd = `git -C "${repoPath}" log -n 15 --name-only --format="COMMIT"`;
        const stdout = execSync(gitLogCmd, { encoding: 'utf8', stdio: ['ignore', 'pipe', 'ignore'] });
        const lines = stdout.split('\n').map(l => l.trim()).filter(l => l && l !== 'COMMIT');
        const fileCounts = {};
        for (const file of lines) {
            fileCounts[file] = (fileCounts[file] || 0) + 1;
            if (fileCounts[file] >= 4) {
                log(`[COLLISION] Agent War Detected: File ${file} modified ${fileCounts[file]} times recently.`);
                return true;
            }
        }
        return false;
    } catch (e) {
        return false; // Fail open — no git config should not block work
    }
}

function getNextTask(repoPath) {
    const statePath = path.join(repoPath, 'STATE.md');
    if (!fs.existsSync(statePath)) return null;
    const lines = fs.readFileSync(statePath, 'utf8').split('\n');
    for (let i = 0; i < lines.length; i++) {
        const match = lines[i].match(/^(\s*[-*]\s*\[\s*\]\s*)(.+)/);
        if (match) {
            return { index: i, line: lines[i], text: match[2].trim() };
        }
    }
    return null;
}

function markTaskAndAdvance(repoPath, taskText, originalPrompt) {
    const statePath = path.join(repoPath, 'STATE.md');
    if (!fs.existsSync(statePath)) return;
    let lines = fs.readFileSync(statePath, 'utf8').split('\n');
    let taskFound = false;
    for (let i = 0; i < lines.length; i++) {
        if (lines[i].includes('[ ]') && lines[i].includes(taskText)) {
            lines[i] = lines[i].replace(/\[\s*\]/, '[x]');
            taskFound = true;
            break;
        }
    }
    if (taskFound) {
        fs.writeFileSync(statePath, lines.join('\n'));
        log(`[VIBECOPS] Marked task complete in STATE.md: ${taskText}`);
        const nextTask = getNextTask(repoPath);
        if (nextTask) {
            log(`[VIBECOPS] Spawning next ticket for: ${nextTask.text}`);
            const newTicket = {
                TaskID: `TSK_AUTO_${Date.now()}`,
                Target: 'ANTIGRAVITY',
                Status: 'PENDING',
                RepoPath: repoPath,
                Prompt: originalPrompt || '',
                _VibecOpsOriginal: originalPrompt || ''
            };
            const ticketPath = path.join(HANDOFF_DIR, `${newTicket.TaskID}.json`);
            fs.writeFileSync(ticketPath, JSON.stringify(newTicket, null, 2));
        } else {
            log(`[VIBECOPS] STATE.md is 100% complete for ${repoPath}!`);
        }
    }
}

// ============================================================================
// CORE WATCHDOG ENGINE
// ============================================================================

let isDispatching = false;
let dispatchDebounce = null;
let activeJobs = [];

function triggerDispatch(reason) {
    if (fs.existsSync(LOCK_FILE)) {
        log(`Lock file present. Deferring trigger: ${reason}`);
        return;
    }
    if (isDispatching) return;
    clearTimeout(dispatchDebounce);
    dispatchDebounce = setTimeout(() => {
        isDispatching = true;
        log(`Triggering Dispatcher. Reason: ${reason}`);
        const jobsToProcess = [...activeJobs];
        activeJobs = [];
        exec(`node "${DISPATCH_SCRIPT}"`, (error, stdout, stderr) => {
            isDispatching = false;
            if (error) {
                log(`[ERROR] Dispatcher failed: ${error.message}`);
                return;
            }
            if (stdout) log(`[Dispatcher Output]\n${stdout.trim()}`);
            // POST-DISPATCH: Domino Effect
            for (const job of jobsToProcess) {
                if (job.repoPath && job.taskText) {
                    markTaskAndAdvance(job.repoPath, job.taskText, job.originalPrompt);
                }
            }
        });
    }, 1000);
}

const watcher = chokidar.watch(HANDOFF_DIR, {
    ignored: /(^|[\/\\])\../,
    persistent: true,
    awaitWriteFinish: { stabilityThreshold: 500, pollInterval: 100 }
});

log('VibecOps Watchdog initialized. Listening for JADE handoff tickets...');

watcher
    .on('add', filePath => handleFileEvent(filePath, 'Added'))
    .on('change', filePath => handleFileEvent(filePath, 'Modified'));

function handleFileEvent(filePath, eventType) {
    if (!filePath.endsWith('.json')) return;
    try {
        const fileContent = fs.readFileSync(filePath, 'utf8');
        if (!fileContent.trim()) return;
        const ticket = JSON.parse(fileContent);
        if (ticket.Target === 'ANTIGRAVITY' && ticket.Status === 'PENDING') {
            if (ticket._VibecOpsProcessed) {
                triggerDispatch(`Ticket ${ticket.TaskID} ready`);
                return;
            }
            const repoPath = ticket.RepoPath || ticket.ProjectDir;
            let activeTaskText = null;
            if (repoPath && fs.existsSync(repoPath)) {
                // PRE-HOOK 1: Collision Check
                if (isCollisionDetected(repoPath)) {
                    ticket.Status = 'ERROR_COLLISION';
                    ticket.Error = 'Agent War Detected. Human review required.';
                    fs.writeFileSync(filePath, JSON.stringify(ticket, null, 2));
                    log(`[CRITICAL] Halted ${ticket.TaskID} due to git collision in ${repoPath}.`);
                    return;
                }
                // PRE-HOOK 2: Bite-Sizer
                const nextTask = getNextTask(repoPath);
                if (nextTask) {
                    const originalPrompt = ticket.Prompt || ticket._VibecOpsOriginal || '';
                    ticket._VibecOpsOriginal = originalPrompt;
                    ticket.Prompt = `[VIBECOPS STRICT DIRECTIVE]\nYour ONLY objective for this run is: ${nextTask.text}\nDo not touch anything outside of this scope. Output code, execute your operations, submit the result for reviewed merge, and exit.\n\n---\nGlobal Context:\n${originalPrompt}`;
                    activeTaskText = nextTask.text;
                    log(`[BITE-SIZER] Scoped ticket to micro-task: ${activeTaskText}`);
                }
            }
            ticket._VibecOpsProcessed = true;
            if (activeTaskText) {
                activeJobs.push({ repoPath, taskText: activeTaskText, originalPrompt: ticket._VibecOpsOriginal });
            }
            fs.writeFileSync(filePath, JSON.stringify(ticket, null, 2));
        }
    } catch (err) {
        log(`[WARNING] Failed to parse ${path.basename(filePath)}: ${err.message}`);
    }
}
```

---

## 9. Deployment Runbook

### Step 1 — Backup Original Watchdog

```bash
cp antigravity_watchdog.js antigravity_watchdog.BACKUP.js
```

### Step 2 — Deploy VibecOps Edition

Replace the full contents of `antigravity_watchdog.js` with the source in Section 8.

### Step 3 — Add `STATE.md` to Target Repo

Go to any one of your 12 repos and create `STATE.md` in the root:

```markdown
# FEATURE: Stripe Billing Microservice
**Status:** IN_PROGRESS
**Global Context:** We are building a Stripe webhook listener for App #4.

## TASKS
- [ ] TASK 1: Initialize database schema for user subscriptions.
- [ ] TASK 2: Build POST endpoint `/api/webhooks/stripe`.
- [ ] TASK 3: Add signature verification to the webhook endpoint.
- [ ] TASK 4: Write DB update logic for `subscription.active` events.
```

### Step 4 — Restart PM2

```bash
pm2 restart antigravity_watchdog
pm2 logs antigravity_watchdog
```

### Step 5 — Drop the Kick-Off Ticket

Create one `.json` file in `/handoff/`:

```json
{
  "TaskID": "TSK_KICKOFF_001",
  "Target": "ANTIGRAVITY",
  "Status": "PENDING",
  "RepoPath": "C:\\AG_Workspaces\\repos\\App_4",
  "Prompt": "Global context: Building Stripe billing microservice for App 4. See STATE.md for the full task list."
}
```

**That's it.** Drop this single ticket and walk away. The watchdog will execute all 4 tasks sequentially, auto-merging each one, until `STATE.md` reports 100% complete.

---

## 10. Failure Recovery

| Failure Scenario | What Happens | Recovery |
|---|---|---|
| Jules API timeout mid-task | `exec()` returns error → Domino Effect skips → task never marked `[x]` | Ticket stays in `/handoff/`; runs on next PM2 restart |
| GitHub API rate limit | Same as above | Auto-retry on next cycle |
| `STATE.md` not found | Bite-Sizer skips; original prompt sent | Add `STATE.md` to repo and retry |
| Agent War detected | Ticket set to `ERROR_COLLISION`; no dispatch | Human reviews git log, manually resets ticket Status to `PENDING` |
| Power loss / PM2 crash | Ticket still on disk in `/handoff/` | PM2 autorestart re-reads folder; work resumes |
| Task completed but `STATE.md` write fails | Ticket was already dispatched | Manually mark `[x]` in `STATE.md` and drop new ticket |

---

## 11. Future Pillars (Not In Scope — Documented for Reference)

| Pillar | Name | Status |
|---|---|---|
| 1 | Bite-Sizer Assembly Line | ✅ **This document** |
| 2 | Replit Blueprint Generator | 🔜 Planned |
| 3 | Cross-Repo Context Relay | 🔜 Planned |
| 4 | Collision Detector (Full) | 🔜 Planned (Preview in Pillar 1) |
| 5 | Daily Audit Scheduler | 🔜 Planned |

---

## 12. Contributing / Maintaining `STATE.md`

- **Who writes initial `STATE.md`?** Replit — paste the feature spec, ask Replit to output a `- [ ]` checklist. This saves Antigravity compute.
- **Who updates `STATE.md`?** The watchdog does it automatically via the Domino Effect.
- **Who reads `STATE.md`?** You — open it on GitHub for a live dashboard of any of your 12 apps.
- **What if I need to re-order tasks?** Edit `STATE.md` directly on GitHub. The watchdog reads it fresh on every ticket.

---

*Architecture authored by Antigravity. Validated against the existing PM2 + chokidar deployment environment on Windows. No cron required.*
