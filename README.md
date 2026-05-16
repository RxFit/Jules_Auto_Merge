# Jules Auto-Merge — VibecOPS Pillar 1

> **The Bite-Sizer Assembly Line** | Powered by `antigravity_watchdog.js` + PM2 + chokidar

This repository is the home of the **Jules Auto-Merge** automation — the engine that transforms Jules from a 4-hour marathon runner into a precision assembly-line worker that:

- Reads `STATE.md` and feeds Jules **one micro-task at a time**
- Auto-merges each result to production (YOLO merge)
- Marks the task `[x]` in `STATE.md` automatically
- Spawns the next ticket instantly via the Domino Effect
- Detects Agent Wars before they happen (Collision Detector)

## Quick Start

1. Add `STATE.md` to any of your 12 repos (see template below)
2. Deploy `antigravity_watchdog.js` (VibecOps Edition) — see `ARCHITECTURE.md`
3. Drop one kick-off ticket into `/handoff/`
4. Walk away

## `STATE.md` Template

```markdown
# FEATURE: [Feature Name]
**Status:** IN_PROGRESS
**Global Context:** [One paragraph describing the app and what you are building]

## TASKS
- [ ] TASK 1: [First task for Jules]
- [ ] TASK 2: [Second task]
- [ ] TASK 3: [Third task]
```

## Full Documentation

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the complete system design, ticket schema, hook logic, PM2 config, and failure recovery playbook.

## Pillars

| # | Pillar | Status |
|---|---|---|
| 1 | Bite-Sizer Assembly Line | ✅ Active |
| 2 | Replit Blueprint Generator | 🔜 Planned |
| 3 | Cross-Repo Context Relay | 🔜 Planned |
| 4 | Collision Detector (Full) | 🔜 Planned |
| 5 | Daily Audit Scheduler | 🔜 Planned |

---
*Maintained by Antigravity (local agent). Architecture doc generated 2026-05-16.*
