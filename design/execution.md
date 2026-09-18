# ITX — Execution Plan

> Superseded planning order: see [PLAN.md §8](PLAN.md#8-implementation-order).
> The detailed steps below are the earlier baseline, pending revised-plan review.

Each step lists its UAT (how a human verifies it) and exit criteria (what must be true
to move on). Steps are sequential; a step's exit criteria gate the next.

## Step 1 — Repo restructure

Move the old product out of the way; scaffold the new one.

- `git mv .claude/skills archived/skills`; move `docs/ITX_WORKFLOW.md` and `.itx/`
  into `archived/`.
- `go mod init github.com/ric03uec/itx`; scaffold `cmd/itx/main.go`, `internal/`
  package dirs, Makefile (`build`, `test`, `lint`, `release-build`).
- cobra skeleton: `itx version` works; `init|session|task|project|skill|update`
  subcommands registered as stubs (no `config` command — users edit config.yml
  directly).

**UAT**
- `make build && ./bin/itx version` prints a version.
- `git log --follow archived/skills/itx-execute/SKILL.md` shows history preserved.
- Repo root contains no live GitHub-workflow skill dirs.

**Exit criteria**
- `make build`, `make test`, `make lint` all pass (empty test suite OK).
- All legacy skills live under `archived/` and nothing references them from live docs.

## Step 2 — Kernel state core (DAG store + node management)

- `internal/kernel/store`: storage adapter interface (`Load` → DAG + revision,
  `Commit` at expected revision → conflict error on mismatch) + JSON backend:
  single `dag.json`, exclusive file lock, write-temp-then-rename, revision bump per
  commit; `XDG_CONFIG_HOME` respected.
- `internal/kernel`: DAG node/edge model (root, project, session, task; child +
  dependency edges; per-node `caller_pid`; per-task `workspace` record with
  `dir`/`is_worktree`/`branch`); shared state machine
  (waiting/running/paused/blocked/done/failed) with transition validation;
  config.yml load with defaults (no config command); commit-retry helper
  (re-read on conflict).
- `internal/gitx`: project slug detection (git toplevel basename → cwd fallback,
  hash-suffix on collision); fail fast if the project dir is not a git repo.
- Commands: `init` (create root, idempotent), `session new --goal|--from`,
  `session show` (manifest + % completion), `session update`, `task add`,
  `task update`, `project status`. Every create/start records the caller pid.
- Dependency validation on add/import: unknown `depends_on` slugs and cycles rejected;
  `session show` renders tasks in topological order.

**UAT** (in a scratch repo, `XDG_CONFIG_HOME` pointed at a temp dir)
- `itx init` creates `dag.json` with a root node; running it again is a no-op.
- `itx session new --goal "…"` prints an id; dag.json contains root→project→session
  child edges and the caller pid on the session node.
- `itx task add` ×3 with `--depends-on`; `itx session show` renders order + statuses;
  a cyclic or unknown dependency is rejected with a clear error.
- `itx session new --from plan.json` bulk-imports the same shape (goal + tasks).
- Illegal transition (e.g. `done` → `running`) is rejected; `itx session update <id>
  --status done` removes it from `itx project status` output.
- Two concurrent `itx task update` loops (shell) never corrupt dag.json and never
  lose an update: conflicting commits are rejected by revision check and retried.

**Exit criteria**
- Unit tests cover: lock/atomic write, optimistic-lock conflict + retry, state-machine
  transition rules, slug detection, cycle rejection, project-status query.
  `make test` green.
- Storage adapter interface has no JSON-specific leakage (kernel compiles against the
  interface only).
- All FR2 requirements (req.md) demonstrably met via CLI alone.

## Step 3 — Scheduler loop + executor + tman + adapters

- `internal/tman`: `TerminalManager` interface (CreateSession, AddWindow, SendCommand,
  IsAlive, KillWindow, KillSession) + tmux backend. All calls idempotent.
- `internal/gitx`: worktree add/remove under
  `~/.config/itx/projects/{slug}/sessions/{session-id}/worktrees/{task-slug}`, branch
  `itx/{session-slug}/{task-slug}`. Worktrees are mandatory — non-git project dir ⇒
  `execute` fails fast; worktrees auto-removed when the session reaches `done`
  (branches kept). Workspace record (`dir`/`is_worktree`/`branch`) written to the
  task node.
- `internal/harness`: adapter interface; fixed list claude/opencode/pi, templates
  from config; PATH auto-detect in that order — none found ⇒ fail fast;
  `--harness`/`--model` overrides.
- `internal/kernel/executor`: called by the scheduler loop each tick (NOT a loop):
  work queue (next task node whose dependency edges are all `done`) + task
  materialization — worktree, prompt generation (title + DoD + status-update
  contract), harness command assembly, window request via tman; task → `running`
  only after its window is verified live.
- `internal/llm`: LLM adapter interface; caller-provider default, config override.
  (v1 wiring only; no scheduling logic may call it.)
- `internal/kernel`: THE single scheduler loop per arch.md workflow 3 — one pass per
  tick over sessions, tasks, and liveness: schedule via executor, reconcile (dead
  window ⇒ `failed`; `done` ⇒ kill window, unblock dependents; `failed` ⇒ dependents
  `blocked`), print % completion each tick, every state change committed with
  optimistic locking (retry on conflict), terminal-state handling (session `done` ⇒
  clean up worktrees), `max_parallel`. No second loop anywhere.
- Commands: `session execute` (idempotent end-to-end; session → `running` only after
  the terminal session + loop are verified live, caller pid recorded),
  `session stop` (→ `paused`), hidden `session run`.

**UAT** (fake harness: shell script that sleeps then calls `itx task update`)
- 3 tasks (t2,t3 depend on t1): `execute` spawns only t1; after t1 → `done`,
  t2+t3 windows appear in parallel; window 0 prints % completion each tick; session
  ends `done`; worktrees are removed (branches remain); `itx project status` empties.
- `tmux attach -t {project}-{session-slug}` shows the scheduler loop in window 0 and
  live task windows named `{session-slug}-{order}-{task-slug}`; worktrees live under
  `~/.config/itx/projects/…/sessions/…/worktrees/` and each task node carries its
  workspace record (`is_worktree: true`).
- Kill the loop between CreateSession and the `running` commit (test hook):
  session stays out of `running`; re-`execute` completes cleanly (idempotency).
- Kill a task window mid-run: task → `failed`, dependents → `blocked`, session →
  `blocked`, scheduler loop stands down with a report.
- `itx session stop` kills all windows, session + in-flight tasks → `paused`;
  re-`execute` resumes only non-terminal tasks and reuses existing worktrees.
- In a non-git dir, `execute` fails fast with a clear error; with no harness on
  PATH and no config, `execute` fails fast listing claude/opencode/pi.
- `max_parallel: 1` in config serializes spawning.

**Exit criteria**
- Integration test (real tmux + fake harness) covering the happy path and the
  dead-window reconcile path runs in CI. `make test` green.
- FR3 requirements demonstrably met; scheduler loop restart recovers from dag.json
  alone.
- Kernel (including its executor) has zero direct tmux calls (everything through tman) —
  enforced by review/grep in CI.

## Step 4 — Skill + embed

- Write `skills/itx/SKILL.md`: interview → build session with DoD/deps → CLI calls
  only → execute → monitor → report. No direct DAG-file edits.
- Skill instructs the calling harness to monitor after `execute` and report session
  progress as % completion.
- `go:embed` the skill; `itx skill install [claude|opencode|pi|all]` writes to each
  harness's skill dir (verify pi's dir during this step). Refresh happens via
  `itx update` (Step 5), not a separate skill-update command.

**UAT**
- `itx skill install claude` places the skill; `/itx` appears in a fresh Claude Code
  session.
- Drive a 2-task session end-to-end from Claude Code using only the skill.
- `itx skill install all` on a machine with only claude installed: installs claude,
  reports the others as not found (non-fatal).

**Exit criteria**
- Skill runs the full plan→execute→monitor loop in at least Claude Code and
  opencode, reporting % completion while the session runs.
- Installed skill content is byte-identical to the embedded copy (re-running
  `itx skill install` is a no-op).

## Step 5 — Distribution (installer + CI + release)

- `install.sh` per arch.md workflow 1: OS/arch dispatch (windows stub), tmux gate,
  checksum-verified download from GH Releases, install to `~/.local/bin`, PATH check,
  offer `itx init` + `itx skill install all`.
- `itx update`: check latest GH Release; update binary AND every installed skill in
  one idempotent step; no-op with a clear message when already current.
- `.github/workflows/ci.yml`: test + lint on PR.
- `.github/workflows/release.yml`: push to `v-X.Y` → test → cross-compile
  linux/darwin × amd64/arm64 → next `YY.MM.PP` tag → GH Release + checksums.txt.
- `skills/itx-release/SKILL.md`: adapted from atx-release (drop R2/PostHog/updater
  phases).

**UAT**
- On a clean Linux box/container without tmux: installer exits non-zero with apt/dnf
  instructions. With tmux: installs, `itx version` works.
- Same flow on macOS (arm64).
- Tamper with a downloaded binary → checksum verification fails loudly.
- Cut `v-26.09` and push: CI produces tag `26.09.01`, 4 binaries + checksums on the
  GH Release; installer one-liner from README installs that release.
- `itx update` right after install: reports up-to-date, changes nothing; after a
  newer release exists: replaces binary and refreshes installed skills.

**Exit criteria**
- Fresh-machine install → init → skill install → 2-task session works on Linux and
  macOS using only the README one-liner.
- ci.yml green on PRs; release.yml produced at least one real release.

## Step 6 — Docs

- Rewrite README.md (product statement, install one-liner, quick start, CLI
  reference), AGENTS.md (CLI contract for agents + skill behavior), CLAUDE.md
  pointer; prune/redirect old docs to `archived/`.

**UAT**
- A newcomer following only README gets from zero to a completed 2-task session.
- No live doc references archived skills as current functionality.

**Exit criteria**
- README, AGENTS.md, design/ mutually consistent with shipped behavior; uat.md
  scenarios all pass (see design/uat.md).
