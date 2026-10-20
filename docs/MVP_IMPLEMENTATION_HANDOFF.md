# Implementation Handoff — orKistRAtor MVP

**Status:** APPROVED decisions (user Q1–Q7)  
**Mode for Cursor:** IMPLEMENTATION after this handoff  
**Product binary:** `orKistRAtor.exe`  
**Stack:** Electron + React + TypeScript (Windows-first)

Refs (read-only inspiration, do not copy as base):
- `e:\Work\ORCH` — CLI adapters / process spawn patterns
- `e:\Work\ouroboros` — sequential stage loop + review restart
- `e:\Work\everything-claude-code` — harness wrapping (post-MVP only)

---

## A. Goal and boundaries

### Observable behavior
1. User launches `orKistRAtor.exe` → desktop window with minimal UI.
2. Pages: **Active tasks**, **Completed tasks**, **Settings**.
3. Create Task: choose `projectPath`, enter task text, set task-level toggles (see settings rules), submit → flow starts (or queues).
4. Task = ordered history/queue of **stages**. Open task → stage list. Open stage → **read-only** CLI chat stream (no user input in MVP).
5. Flow stages: `interview → plan → implement → review`. On review `FAIL` → new cycle starting at `plan` (skip `interview`). On review `PASS` → task completed.
6. Only **one task running** at a time. Other active tasks wait. If queue-continue is effective → after finish, start next eligible task.
7. Settings: Cursor connection check; global `communicate`; global `continueNextTaskAfterFinish`.
8. No plugins installed into the user project or into Cursor. Orchestrator spawns `cursor-agent` CLI directly.

### Unchanged / out of scope (MVP)
- Parallel running tasks
- Presets / model-per-stage profiles
- Enabling Codex / Claude Code in UI (adapter interfaces may exist stubbed; runtime+UI = Cursor only)
- ECC harness injection into project / dynamic harness
- Writing into stage chat
- Linux/macOS packaging (same codebase OK; MVP gate = Windows exe)
- Push/merge/deploy automation

### Forbidden without new agreement
- Forking ORCH/ouroboros/ECC into this repo as the app base
- Installing skills/hooks into `.cursor/` of the target project
- Changing public contracts after they ship without bump + handoff update
- Production secrets in repo

---

## B. Baseline

| Item | Value |
|---|---|
| Repo | `e:\Work\orKistRator` → `origin` `https://github.com/BoGGuN/orKistRAtor.git` |
| Reset | Greenfield after commit that removed ORCH fork tree (history kept) |
| Dirty policy | Implement only against this handoff |

### Target architecture (greenfield)

```
apps/desktop (Electron main + preload)
  └── renderer (React pages)
core/
  domain/          Task, Stage, Settings, Verdict
  application/     FlowEngine, TaskQueue, SettingsService
  infrastructure/
    adapters/      CursorAdapter (+ stubs Codex/Claude unused)
    process/       spawn, kill, stream
    storage/       app-data JSON/SQLite
```

**Do not duplicate:** ORCH org/teams/worktree-as-core; ouroboros Seed/ontology/MCP; ECC installers.

**May study:** ORCH `cursor` adapter spawn flags (`cursor-agent` / `agent`, stream-json); ouroboros loop semantics; ECC multi-harness *ideas* only.

---

## C. Decisions and invariants

### C1. Desktop stack
- **Decision:** Electron + React + TypeScript
- **SoT:** this handoff (Q2=A)
- **Why:** fastest Windows MVP + embedded read-only terminal/stream UI
- **Invariant:** Win/Linux/mac builds share one UI; MVP ships Windows `orKistRAtor.exe`

### C2. Task ↔ project path
- **Decision:** `projectPath` chosen at Create Task (required)
- **SoT:** Q3=A
- **Invariant:** every stage run uses that task’s `projectPath` as cwd/workspace for `cursor-agent`
- **Error:** missing/invalid path → block start, show error; task stays draft/failed-to-start

### C3. Stage flow
- **Stages:** `interview` | `plan` | `implement` | `review`
- **First cycle:** interview → plan → implement → review
- **After review FAIL:** append new stages plan → implement → review (no interview)
- **After review PASS:** task → completed
- **Max cycles (default):** 3 review failures then task → `failed` (configurable later; hardcode 3 in MVP)
- **SoT:** product intent + ouroboros-like loop (simplified)

### C4. Review verdict
- **Decision:** stage prompt requires machine-readable verdict at end:
  ```
  VERDICT: PASS
  ```
  or
  ```
  VERDICT: FAIL
  ```
  (optional JSON line also accepted: `{"verdict":"PASS"|"FAIL"}`)
- **SoT:** Q4=A
- **Parse:** scan tail of stage output; last matching verdict wins
- **If missing/ambiguous:** treat as `FAIL`, stop auto-advance, surface error on stage (user can Continue only if `communicate` effective — see C5); do not invent PASS
- **Invariant:** never mark task completed without explicit PASS

### C5. Setting `communicate` (ex-pause-between-stages)
- **Name:** `communicate`
- **Meaning:** when **effective true**, after a stage finishes successfully wait for user **Continue** before next stage; when **false**, auto-chain next stage
- **Global default:** `false`
- **Task field:** `communicate` (set only on create UI; no separate task settings page)
- **Inheritance / ceiling rules (SoT: user Q5–Q6):**
  1. On create, task toggles **copy** current global values.
  2. If global `communicate === false`, task UI control is **disabled**; task value forced `false` (cannot enable).
  3. If global `communicate === true`, task may set `true` or `false` (opt-out allowed).
  4. **Effective value** for a running task = task’s stored value (already constrained by rules above). Changing global later does **not** rewrite existing tasks (MVP).
- **Also applies between cycles** (after FAIL review → before next plan) when effective true.

### C6. Setting `continueNextTaskAfterFinish`
- **Global default:** `false`
- **Task field:** same name; create-time only; same ceiling rules as `communicate`
- **Meaning when effective true:** when current task reaches terminal status (`completed` or `failed`), if another active (queued) task exists → start the next one (FIFO by createdAt)
- **When effective false:** stop; user starts next manually
- **Invariant:** never two tasks `running` simultaneously (Q6=B∧C)

### C7. Queue / concurrency
- **Active** = not completed/cancelled (includes `queued`, `running`, `waiting_communicate`)
- **Completed list** = `completed` | `failed` | `cancelled` (product may show failed under completed; label “Completed” = finished terminal — document in UI as “Finished” if needed; MVP: page name “Completed tasks” includes failed)
- Exactly one `running` task app-wide
- Create while another runs → new task `queued`

### C8. Cursor-only runtime (MVP)
- Adapter interface supports future `cursor` | `codex` | `claude`
- Registry: only `cursor` enabled
- Settings → Connections: Cursor doctor (`cursor-agent`/`agent` `--version` or equivalent test)
- **Invariant:** no write of plugins into user project

### C9. Stage chat UI
- Read-only stream of adapter stdout/stderr (and parsed assistant text if stream-json)
- No input box / send in MVP
- Historical stages: show persisted log

### C10. Persistence
- App data under OS app-data dir (e.g. `%APPDATA%/orKistRAtor/`)
- Store: settings.json + tasks (JSON files or SQLite — prefer SQLite if simple; JSON OK for MVP)
- Each stage: status, startedAt, endedAt, log path, verdict

### C11. Harness
- MVP: orchestrator-owned system prompts per stage only
- Post-MVP: ECC-like harness takes priority over ouroboros-like prompts — **not in this MVP**

---

## D. Change matrix

| Where | What | Why | Preserve | Prove with |
|---|---|---|---|---|
| `apps/desktop` | Electron shell, nav, pages | UX path | — | smoke open window |
| `core/domain` | Task/Stage/Settings models + ceiling rules | SoT settings | — | unit tests inheritance |
| `core/application/flow-engine` | stage sequence, review loop, communicate wait | ouroboros-like flow | single runner | unit flow tests |
| `core/application/task-queue` | one running + continueNext | Q6 | — | unit queue tests |
| `core/infrastructure/adapters/cursor` | spawn cursor-agent | direct CLI | no project plugins | adapter unit + doctor |
| `core/infrastructure/verdict` | parse VERDICT | Q4 | never false PASS | unit parse tests |
| Settings UI | global toggles + Cursor check | MVP settings | — | component/integration |
| Create Task UI | path, text, task toggles | Q3/Q6 | ceiling lock | unit + UI |
| Stage terminal UI | RO stream | Q8/9 | no input | UI test |

---

## E. Implementation plan (Cursor sequence)

1. Scaffold Electron+React+TS monorepo; productName `orKistRAtor`; Windows target.
2. Domain models + settings ceiling helpers + **tests first**.
3. Verdict parser + **tests**.
4. FlowEngine (mock adapter) + communicate + review loop + **tests**.
5. TaskQueue (single runner, continueNext) + **tests**.
6. CursorAdapter + ProcessManager + doctor.
7. Storage persistence.
8. Wire IPC main↔renderer.
9. UI pages: Settings, Active, Completed, Create, Stages, StageChat RO.
10. End-to-end with mock adapter; manual script with real cursor-agent if available.
11. Package Windows `orKistRAtor.exe`.
12. Return Implementation Report for Codex validation review.

---

## F. Acceptance matrix

| Criterion | Evidence | Check |
|---|---|---|
| Exe opens to 3 pages | manual / e2e | packaged Win build |
| Create task requires projectPath + text | test + UI | unit/e2e |
| Stages created in order interview→plan→implement→review | flow test | vitest |
| Review FAIL appends plan→implement→review (no interview) | flow test | vitest |
| Review PASS → task completed | flow test | vitest |
| Missing VERDICT → not PASS | verdict test | vitest |
| Global communicate=false → task cannot enable | settings test | vitest |
| Global communicate=true → task can true/false; create copies global | settings test | vitest |
| Effective communicate=true waits for Continue | flow test | vitest |
| Effective communicate=false auto-next stage | flow test | vitest |
| Only one running task | queue test | vitest |
| continueNext effective true starts next queued | queue test | vitest |
| continueNext false leaves next queued | queue test | vitest |
| Settings Cursor doctor reports ok/fail | adapter.test | unit + manual |
| Stage chat has no input | UI review | e2e/manual |
| No files written to project for harness/plugins | code review | grep / test |
| Artifact name orKistRAtor.exe | electron-builder config | build |

---

## G. Risks (applicable)

| Risk | Mitigation |
|---|---|
| cursor-agent CLI missing / renamed | doctor + clear Settings error; try `cursor-agent` then `agent` |
| stream-json parse drift | persist raw log; display raw fallback |
| Verdict format ignored by model | strong stage prompt + FAIL on missing |
| Global settings change mid-flight | MVP: existing tasks keep snapshot; document |
| Electron security | contextIsolation, no nodeIntegration in renderer |
| Accidental ORCH copy-paste | greenfield only; review forbids large ORCH dumps |
| Queue + communicate deadlock | Continue only unblocks stage; queue start separate |

---

## H. Contract for Cursor

1. Work only in approved MVP scope.
2. Do not change architecture/contracts without BLOCKER report.
3. Tests first for domain/flow/queue/verdict.
4. Cover error/disabled/communicate/queue paths, not only happy path.
5. If handoff conflicts with reality → stop with BLOCKER.
6. Do not mark done without docs/config artifacts in scope (builder productName).
7. Do not claim CI/runtime evidence you did not run.

### Implementation Report required
- files changed
- behavior summary
- acceptance mapping
- tests added
- commands + results
- refs/commits
- skipped checks
- deviations / questions for Codex

---

## I. UI map (MVP)

```
[Active tasks]  [Completed tasks]  [Settings]
       │                │               │
       ▼                ▼               ▼
  list queued/        list terminal    communicate
  running/waiting     tasks            continueNextTaskAfterFinish
       │                               Cursor connection (test)
       ▼
  Task stages (history)
       │
       ▼
  Stage chat (read-only stream)

[+ Create task]
  - projectPath (folder picker)
  - prompt text
  - communicate (locked if global false)
  - continueNextTaskAfterFinish (locked if global false)
  - Submit → enqueue/start
```

---

## J. Resolved answers log

| Q | Answer |
|---|---|
| Q1 wipe | A — delete files, commit, push, keep history |
| Q2 stack | A — Electron + React + TS |
| Q3 project | A — select on create |
| Q4 verdict | A — machine-readable VERDICT |
| Q5 pause | rename `communicate`, default **false** |
| Q6 queue | B+C; global+task toggles; ceiling rules; create copies globals |
| Q7 name | `orKistRAtor.exe` |
