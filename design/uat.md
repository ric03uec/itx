# ITX — Acceptance & Verification (UAT)

Final acceptance scenarios for the whole product. Run on both Linux (amd64) and macOS
(arm64) unless noted. All scenarios use only public surfaces: `install.sh`, the `itx`
CLI, the skill, tmux.

## A. Installation & init

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| A1 | Fresh install, tmux present | `curl …/install.sh \| bash` on a clean machine with tmux | Binary at `~/.local/bin/itx`; `itx version` prints version; installer offers init + skill install |
| A2 | tmux missing | Hide tmux from PATH, run installer | Exits non-zero; prints platform-correct install command (brew/apt/dnf); nothing installed |
| A3 | Unsupported OS | Run installer under simulated Windows (`OSTYPE` override) | Clear "Windows support planned" message; clean exit; no partial install |
| A4 | Checksum tamper | Corrupt downloaded binary before verify (test hook) | Install aborts loudly; no binary placed on PATH |
| A5 | Init | `itx init`; run again | dag.json created with root node; second run is a no-op; commands before init fail with a clear "run itx init" error |
| A6 | Skill install | `itx skill install all` with claude + opencode present, pi absent | Skill lands in both harness dirs; pi reported not-found, exit 0 |

## B. DAG lifecycle (CLI only)

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| B1 | Create + inspect | `itx session new`; `itx task add` ×3 (t02,t03 depend on t01); `itx session show` | Session id printed; dag.json holds root→project→session→task child edges + dependency edges; show renders tasks in dependency order with statuses/DoD |
| B2 | Bulk import | `itx session new --from manifest.json` | Same result as B1; invalid deps/cycles rejected with clear error, no nodes created |
| B3 | Status updates | `itx task update … --status running` then `done`; `itx session update … --status done` | Timestamps set; illegal transitions (e.g. `done` → `running`) rejected; `itx project status` no longer lists the session |
| B4 | Resume index | Create 2 sessions, complete 1 | `itx project status` (DAG query) lists exactly the non-terminal one |
| B5 | Concurrent writes | 2 parallel loops of `itx task update` on different tasks | dag.json valid JSON throughout; no lost updates — conflicting commits rejected by revision check and retried |
| B6 | Failure cascade | Mark t01 `failed` | t02, t03 → `blocked`; session → `blocked` |

## C. Orchestrated execution

Fake harness (script that sleeps, then `itx task update --status done`) unless
stated; real harness in C7.

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| C1 | Dependency-ordered parallel run | B1 session; `itx session execute <id>` | tmux session `{project}-{session-slug}` exists; window 0 = kernel loop; only t01 spawns first; t02+t03 spawn in parallel after t01 → `done` (windows named `{session-slug}-{order}-{task-slug}`); session → `done`; `itx project status` empties |
| C2 | Worktree isolation | During C1, inspect worktrees | Each task ran in `{repo}-{session-slug}-{task-slug}` worktree on branch `itx/{session-slug}/{task-slug}`; main tree untouched |
| C3 | Reconciler: dead window | Kill t01's window mid-run | t01 → `failed`; t02,t03 → `blocked`; session → `blocked`; kernel loop reports and stands down |
| C4 | Pause + resume | `itx session stop` mid-run; then `execute` again | Stop kills all windows, session + in-flight tasks → `paused`; re-execute reuses tmux/worktrees, spawns only non-terminal tasks, completes |
| C5 | Concurrency cap | `max_parallel: 1`; session with 2 independent tasks | Second window appears only after first → `done` |
| C6 | Harness selection | No config → auto-detect; then `--harness opencode`; then config `default_harness: pi` | Precedence flag > config > auto-detect observable in spawned command lines |
| C7 | Real harness | 2-task session with real `claude` | Children call `itx task update` per contract; session → `done`; results on task branches |
| C8 | Non-git project | Run C1 in a non-git dir | Warning about shared-dir isolation; tasks run in project dir; otherwise same lifecycle |
| C9 | Crash recovery | Kill the kernel-loop window mid-run; re-`execute` | Kernel recovers from dag.json alone; no duplicate windows; run completes |

## D. Skill-driven flow (per harness)

Run once per harness: Claude Code, opencode, pi (pi best-effort in v1).

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| D1 | Plan via skill | Invoke `/itx` with a 3-task goal | Agent interviews for DoD + deps; creates session/tasks via CLI only (no direct DAG-file edits); shows plan before executing |
| D2 | Execute + monitor | Confirm; agent runs `itx session execute`, then `itx session show` | Agent reports progress from CLI output; user can `tmux attach` independently |
| D3 | Blocker surfacing | Force a task failure | Agent surfaces failed/blocked state and options, doesn't silently retry forever |

## E. Release pipeline

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| E1 | Cut release | Push to `v-X.Y` | CI: tests pass → tag `YY.MM.PP` → GH Release with 4 binaries + checksums.txt |
| E2 | Installer ↔ release | Run README one-liner after E1 | Installs exactly the E1 binaries; checksums verify |
| E3 | Release skill | `/itx-release` in this repo | CHANGELOG archived to `docs/releases/<ver>/`; root reset to `[Unreleased]`; curated notes on the GH Release |

## Sign-off

Product accepted when:
1. All A–E scenarios pass on Linux amd64 and macOS arm64.
2. `make test && make lint` green on main; CI enforces both on PRs.
3. A newcomer, using only README, goes from zero → installed → skill-driven 2-task
   parallel session → completed, in under 15 minutes.
