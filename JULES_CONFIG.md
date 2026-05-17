# VibecOPS Jules Configuration

## Primary Account

| Setting | Value |
|---|---|
| **Primary Google Account** | `daniel.trejo.eclectic@gmail.com` |
| **Plan** | Jules Pro (100 tasks/day) |
| **GitHub Org** | `RxFit` |
| **Interfaces** | Desktop GUI (local) + `vibecops.rx-fit.com` (remote) |

## Architecture

Both the **Desktop GUI Launcher** and **vibecops.rx-fit.com** route through the same VibecOPS bridge server (`vibecops_server.js`). Jules tasks are triggered via GitHub issues and PRs — the interface used to trigger a task is irrelevant to the Jules task limit. The 100-task/day limit is enforced at the Jules account level (`daniel.trejo.eclectic@gmail.com`).

## Workflow Files

All active repos deploy three canonical workflow files:

| Workflow | File | Trigger |
|---|---|---|
| **Assign Jules** | `assign-jules.yml` | Push to `STATE.md` |
| **Auto-Merge** | `auto-merge-jules.yml` | PR opened by Jules |
| **Smoke Tests** | `jules-pr-tests.yml` | PR opened on labeled branches |

## Jules Actor Detection

The auto-merge workflow detects Jules PRs by:
1. PR author login: `google-labs-jules`, `julesbot`, `jules-google`
2. PR labels: `jules`, `vibecops`, `auto-merge`

## Repos Covered

| Repo | Tag | Workflows |
|---|---|---|
| `RxFit/RxFit-Concierge` | MAIN | ✅ Deployed |
| `RxFit/AppRxFitai` | MOBILE | ✅ Deployed |
| `RxFit/notebookrx` | AI | ✅ Deployed |
| `RxFit/notebookparser` | AI | ✅ Deployed |
| `RxFit/SDM-Headless-Enterprise` | OPS | ✅ Deployed |
| `RxFit/jade-cos` | AGENT | ✅ Deployed |
| `RxFit/Xana.AI` | TOOLS | ✅ Deployed |
| `RxFit/Jules_Auto_Merge` | VIBECOPS | ✅ Deployed |

## STATE.md Flow

```
VibecOPS GUI / vibecops.rx-fit.com
         │
         ▼
  Push STATE.md to repo
         │
         ▼
  assign-jules.yml triggers
         │
         ▼
  GitHub Issue created → Jules assigned
         │
         ▼
  Jules works → opens PR
         │
         ▼
  jules-pr-tests.yml → smoke test
         │
         ▼
  auto-merge-jules.yml → squash merge to main
         │
         ▼
  Next task in STATE.md queued
```

---
*Maintained by Antigravity (local agent). Last updated: 2026-05-17.*
