# ITX — Execution Plan

Each step lists its UAT (how a human verifies it) and exit criteria (what must be true
to move on). Steps are sequential; a step's exit criteria gate the next.

## Step 1 — Repo restructure

Move the old product out of the way; scaffold the new one.

- `git mv .claude/skills archived/skills`; move `docs/ITX_WORKFLOW.md` and `.itx/`
  into `archived/`.
- `go mod init github.com/ric03uec/itx`; scaffold `cmd/itx/main.go`, `internal/`
  package dirs, Makefile (`build`, `test`, `lint`, `release-build`).
- cobra skeleton: `itx version` works; `session|todo|project|skill|config` subcommands
  registered as stubs.

**UAT**
- `make build && ./bin/itx version` prints a version.
- `git log --follow archived/skills/itx-execute/SKILL.md` shows history preserved.
- Repo root contains no live GitHub-workflow skill dirs.

**Exit criteria**
- `make build`, `make test`, `make lint` all pass (empty test suite OK).
- All legacy skills live under `archived/` and nothing references them from live docs.

## Step 2 — State core (store + session/todo/project commands)

- `internal/store`: config.yml load/save with defaults; work.json; todo.json;
  exclusive file lock + write-temp-then-rename; `XDG_CONFIG_HOME` respected.
- `internal/gitx`: project slug detection (git toplevel basename → cwd fallback,
  hash-suffix on collision).
- Commands: `session new [--from]`, `session show`, `session update`, `todo add`,
  `todo update`, `project status`, `config get|set`.
- Dependency validation on add/import: unknown `depends_on` ids and cycles rejected;
  todos stored topologically sorted.

**UAT** (in a scratch repo, `XDG_CONFIG_HOME` pointed at a temp dir)
- `itx session new` prints an id; todo.json + work.json exist with correct shape.
- `itx todo add` ×3 with `--depends-on`; `itx session show` renders order + statuses.
- `itx session new --from todos.json` bulk-imports; cyclic file is rejected with a
  clear error.
- `itx session update <id> --status complete` removes the id from work.json;
  `itx project status` no longer lists it.
- Two concurrent `itx todo update` invocations (shell loop) never corrupt todo.json.

**Exit criteria**
- Unit tests cover: lock/atomic write, slug detection, cycle rejection, work.json
  membership rules. `make test` green.
- All FR2 requirements (req.md) demonstrably met via CLI alone.

## Step 3 — Orchestrator (tmux + worktrees + harness + controller)

- `internal/tmux`: session/window create, send-keys, pane liveness, kill.
- `internal/gitx`: worktree add/remove (`<repo>-<sid-short>-<todo-id>`, branch
  `itx/<sid-short>/<todo-id>`); non-git fallback = shared dir + warning.
- `internal/harness`: adapter interface; claude/opencode/pi templates from config;
  PATH auto-detect (claude → opencode → pi); `--harness`/`--model` overrides;
  prompt generator (title + DoD + status-update contract).
- `internal/orchestrator`: controller loop per arch.md workflow 3 — spawn unblocked,
  poll, reconcile (dead pane ⇒ failed; complete ⇒ kill window, unblock dependents;
  failed ⇒ dependents blocked), terminal-state handling, `max_parallel`.
- Commands: `session execute` (idempotent), `session stop`, hidden `session run`.

**UAT** (fake harness: shell script that sleeps then calls `itx todo update`)
- 3 todos (t02,t03 depend on t01): `execute` spawns only t01; after t01 completes,
  t02+t03 windows appear in parallel; session ends `complete`; work.json empties.
- `tmux attach -t itx-<slug>-<sid>` shows controller in window 0 and live task windows.
- Kill a task pane mid-run: todo → `failed`, dependents → `blocked`, session →
  `blocked`, controller stands down with a report.
- `itx session stop` kills all windows; re-`execute` resumes only pending todos and
  reuses existing worktrees.
- `max_parallel: 1` in config serializes spawning.

**Exit criteria**
- Integration test (real tmux + fake harness) covering the happy path and the
  dead-pane reconcile path runs in CI. `make test` green.
- FR3 requirements demonstrably met; controller restart recovers from todo.json alone.

## Step 4 — Skill + embed

- Write `skills/itx/SKILL.md`: interview → build todo list with DoD/deps → CLI calls
  only → execute → monitor → report. No direct state-file edits.
- `go:embed` the skill; `itx skill install [claude|opencode|pi|all]` writes to each
  harness's skill dir (verify pi's dir during this step); `itx skill update`
  refreshes.

**UAT**
- `itx skill install claude` places the skill; `/itx` appears in a fresh Claude Code
  session.
- Drive a 2-task session end-to-end from Claude Code using only the skill.
- `itx skill install all` on a machine with only claude installed: installs claude,
  reports the others as not found (non-fatal).

**Exit criteria**
- Skill runs the full plan→execute→monitor loop in at least Claude Code and opencode.
- Installed skill content is byte-identical to the embedded copy (`itx skill update`
  is a no-op right after install).

## Step 5 — Distribution (installer + CI + release)

- `install.sh` per arch.md workflow 1: OS/arch dispatch (windows stub), tmux gate,
  checksum-verified download from GH Releases, install to `~/.local/bin`, PATH check,
  offer `itx skill install all`.
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

**Exit criteria**
- Fresh-machine install → skill install → 2-task session works on Linux and macOS
  using only the README one-liner.
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
