# ITX — Acceptance & Verification (UAT)

Final acceptance scenarios for the whole product. Run on both Linux (amd64) and macOS
(arm64) unless noted. All scenarios use only public surfaces: `install.sh`, the `itx`
CLI, the skill, tmux.

## A. Installation

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| A1 | Fresh install, tmux present | `curl …/install.sh \| bash` on a clean machine with tmux | Binary at `~/.local/bin/itx`; `itx version` prints version; installer offers skill install |
| A2 | tmux missing | Hide tmux from PATH, run installer | Exits non-zero; prints platform-correct install command (brew/apt/dnf); nothing installed |
| A3 | Unsupported OS | Run installer under simulated Windows (`OSTYPE` override) | Clear "Windows support planned" message; clean exit; no partial install |
| A4 | Checksum tamper | Corrupt downloaded binary before verify (test hook) | Install aborts loudly; no binary placed on PATH |
| A5 | Skill install | `itx skill install all` with claude + opencode present, pi absent | Skill lands in both harness dirs; pi reported not-found, exit 0 |

## B. Session & todo lifecycle (CLI only)

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| B1 | Create + inspect | `itx session new`; `itx todo add` ×3 (t02,t03 depend on t01); `itx session show` | Session id printed; todo.json dependency-ordered; show renders statuses/DoD |
| B2 | Bulk import | `itx session new --from todos.json` | Same result as B1; invalid deps/cycles rejected with clear error, no session created |
| B3 | Status updates | `itx todo update … --status inprogress` then `complete`; `itx session update … --status complete` | Timestamps set; session drops out of work.json; `itx project status` empty |
| B4 | Resume index | Create 2 sessions, complete 1 | `itx project status` lists exactly the incomplete one |
| B5 | Concurrent writes | 2 parallel loops of `itx todo update` on different todos | todo.json valid JSON throughout; no lost updates |

## C. Orchestrated execution

Fake harness (script that sleeps, then `itx todo update --status complete`) unless
stated; real harness in C7.

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| C1 | Dependency-ordered parallel run | B1 session; `itx session execute <id>` | tmux session `itx-<slug>-<sid>` exists; window 0 = controller; only t01 spawns first; t02+t03 spawn in parallel after t01 completes; session → `complete`; work.json empties |
| C2 | Worktree isolation | During C1, inspect worktrees | Each task ran in `<repo>-<sid>-<todo-id>` worktree on branch `itx/<sid>/<todo-id>`; main tree untouched |
| C3 | Reconciler: dead pane | Kill t01's pane mid-run | t01 → `failed`; t02,t03 → `blocked`; session → `blocked`; controller reports and stands down |
| C4 | Stop + resume | `itx session stop` mid-run; then `execute` again | Stop kills all windows, inprogress → blocked; re-execute reuses tmux/worktrees, spawns only non-terminal todos, completes |
| C5 | Concurrency cap | `max_parallel: 1`; session with 2 independent todos | Second window appears only after first completes |
| C6 | Harness selection | No config → auto-detect; then `--harness opencode`; then config `default_harness: pi` | Precedence flag > config > auto-detect observable in spawned command lines |
| C7 | Real harness | 2-task session with real `claude` | Children call `itx todo update` per contract; session completes; results on task branches |
| C8 | Non-git project | Run C1 in a non-git dir | Warning about shared-dir isolation; tasks run in project dir; otherwise same lifecycle |

## D. Skill-driven flow (per harness)

Run once per harness: Claude Code, opencode, pi (pi best-effort in v1).

| # | Scenario | Steps | Pass criteria |
|---|----------|-------|---------------|
| D1 | Plan via skill | Invoke `/itx` with a 3-task goal | Agent interviews for DoD + deps; creates session/todos via CLI only (no manual state edits); shows plan before executing |
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
