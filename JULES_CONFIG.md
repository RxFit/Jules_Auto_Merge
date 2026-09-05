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

The repo-local `auto-merge-jules.yml` workflow was retired under T-163 on
2026-09-04. The affected copies were invalid, base64-encoded files, and their
unreviewed merge behavior has been superseded by the centralized, exact-head
merge executor in `RxFit/rxfit-command-center` (PR #173).

Active repositories may still use the task-assignment and smoke-test workflows.
Merge decisions and execution no longer live in each repository:

| Workflow | File | Trigger |
|---|---|---|
| **Assign Jules** | `assign-jules.yml` | Push to `STATE.md` |
| **Reviewed Merge** | Central command-center executor | Eligible PR passes exact-head review and repository gates |
| **Smoke Tests** | `jules-pr-tests.yml` | PR opened on labeled branches |

## Legacy Jules Actor Detection (Retired)

The deleted workflow used the following broad actor/label detection. It is
retained here only as migration history and must not be redeployed:
1. PR author login: `google-labs-jules`, `julesbot`, `jules-google`
2. PR labels: `jules`, `vibecops`, `auto-merge`

## Historical Repos Covered

This table records the prior rollout footprint. It does not indicate that the
retired auto-merge workflow remains deployed.

| Repo | Tag | Legacy rollout |
|---|---|---|
| `RxFit/RxFit-Concierge` | MAIN | Historical |
| `RxFit/AppRxFitai` | MOBILE | Historical |
| `RxFit/notebookrx` | AI | Historical |
| `RxFit/notebookparser` | AI | Historical |
| `RxFit/SDM-Headless-Enterprise` | OPS | Historical |
| `RxFit/jade-cos` | AGENT | Historical |
| `RxFit/Xana.AI` | TOOLS | Historical |
| `RxFit/Jules_Auto_Merge` | VIBECOPS | Historical |

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
  command-center executor → exact-head validation and governed merge
         │
         ▼
  Next task in STATE.md queued
```

---
*Maintained by Antigravity (local agent). Last updated: 2026-09-04.*
