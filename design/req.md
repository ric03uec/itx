# ITX — Requirements

## Product statement

ITX is a parallel task orchestrator. The entire system state is a single **DAG**
(root → projects → sessions → tasks, plus task dependency edges). ITX turns a
**session** — an objective broken into tasks with a clear definition of done (DoD)
and explicit dependencies — into parallel, isolated agent-harness executions inside
tmux. Terminology is defined in [DOMAIN.md](DOMAIN.md). It ships as:

1. A **Go CLI** (`itx`) — the deterministic core. All state lives in the DAG file,
   all state changes go through the CLI (kernel).
2. A **skill** that wraps the CLI so any agent harness (Claude Code, opencode, pi) can
   drive it. The skill uses the CLI for every deterministic call.
3. (Later, out of scope for v1) an **agent** that wraps the skill and runs standalone.

## Functional requirements

### FR1 — Installation

- FR1.1 Installable via `curl <url from this repo> | bash` on Linux and macOS
  (amd64 and arm64).
- FR1.2 The installer MUST verify `tmux` is available. If missing, it prints
  per-platform install instructions and exits non-zero.
- FR1.3 Windows is not supported in v1, but the installer and code MUST be structured
  so Windows support can be added (OS dispatch in installer, tman/harness behind
  interfaces in Go).
- FR1.4 The installer downloads a prebuilt binary from this repo's GitHub Releases,
  verifies its checksum, and installs to the user's PATH.
- FR1.5 `itx init` creates the DAG root node (idempotent; the installer offers to run
  it). All other commands require an initialized DAG.
- FR1.6 After binary install, the user can install the skill into any harness via
  `itx skill install [claude|opencode|pi|all]`. The SKILL.md is embedded in the binary
  so skill and binary versions never drift.
- FR1.7 `itx update` updates the binary AND all installed skills to the latest
  release in one idempotent step — it skips (no-op) when everything is already at
  the latest version. There is no separate skill-update command.

### FR2 — DAG state management (insertion + management layers)

- FR2.1 `itx session new --goal <text>` creates a session node under the current
  project (creating the project node under the root on first use) and returns its
  id — always the first call; tasks are attached afterwards using that id.
  `itx session new --from <file.json>` is the bulk form: the file carries the
  session goal PLUS the full task list (title, DoD, dependencies), inserting the
  session and all its tasks in one call.
- FR2.2 `itx task add` adds a task node as a child of the session; each task carries a
  title, a definition of done, and a `depends_on` list (dependency edges to sibling
  tasks). Unknown dependencies and cycles are rejected.
- FR2.3 `itx task update --session <id> <task-slug> …` updates a task node (status or
  arbitrary fields via JSON). Only legal state transitions are accepted.
- FR2.4 `itx session update <id> --status [waiting|running|paused|blocked|done|failed]`
  updates session status. Sessions and tasks share the same state machine
  (see DOMAIN.md); `done` and `failed` are terminal; `failed` cascades `blocked` to
  dependents.
- FR2.5 `itx session show <id>` reports full session state — the session node plus its
  task subtree (the **manifest** view) — including per-task status, terminal-window
  liveness, and overall progress as **% completion** (done tasks / total).
- FR2.6 `itx project status` reports all sessions of the current project not in a
  terminal state (a DAG query, not a separate file) so work can be resumed.
- FR2.7 State layout:
  - `~/.config/itx/dag.json` — single per-installation file holding the entire DAG
    (all projects, sessions, tasks, edges) plus a revision counter.
  - `~/.config/itx/config.yml` — user configuration (default harness, storage
    backend, etc.).
- FR2.8 Concurrent writers (scheduler loop + N task windows) MUST NOT corrupt or lose
  state: every commit uses **optimistic locking** — it carries the revision it was
  read at; on mismatch the store rejects with a conflict and the writer re-reads and
  retries. Writes are atomic (write-temp + rename) under an exclusive file lock.
- FR2.9 Storage is behind a **storage adapter** interface (load/commit-at-revision).
  v1 ships a single-JSON-file backend; the interface must allow swapping to sqlite
  (or other) with minimal effort and no kernel changes.
- FR2.10 Whenever a session or task is created or started — by a harness, an agent,
  or the user via the CLI — the **caller process id** is recorded on the node.
- FR2.11 Each task node stores its **workspace** record (`dir`, `is_worktree`,
  `branch`) in the DAG itself; whether a directory is a worktree is read from state,
  never inferred from disk.
- FR2.12 There is no `itx config` command; users edit `~/.config/itx/config.yml`
  directly.

### FR3 — Execution (execution layer)

- FR3.1 `itx session execute <id>` starts execution: creates one terminal session
  (named `{project-name}-{session-slug}`) via tman, runs the scheduler loop in
  window 0, and returns. Execution survives the user's terminal closing. The session
  is set to `running` **only after** the terminal session and scheduler loop are
  verified live — a halfway failure leaves the session restartable, and every step
  of `execute` is idempotent (safe to re-run at any point).
- FR3.2 One terminal **window per task** (named
  `{session-slug}-{task-order}-{task-slug}`), each running its own harness instance.
  The executor schedules every task whose dependency edges are all `done` (unlimited
  by default; optional `max_parallel` cap in config). A task is set to `running`
  only after its window is verified live.
- FR3.3 Task isolation is a **git worktree + branch per task — mandatory, no
  exceptions**. Worktrees live under
  `~/.config/itx/projects/<slug>/sessions/<session-id>/worktrees/<task-slug>`,
  are created automatically when the task is scheduled, and are removed
  automatically when the session reaches `done` (branches kept for merging).
  A project directory that is not a git repo fails fast at `execute`.
- FR3.4 The executor sends each task window a harness command with a generated
  prompt containing the task title, DoD, and the status-update contract. The
  calling harness that ran `execute` is instructed to report session progress as
  **% completion** while the session runs.
- FR3.5 Status updates are belt & suspenders:
  - Task harnesses report via `itx task update` (running → done/failed).
  - The scheduler loop polls and reconciles: dead window while `running` ⇒ `failed`;
    `done` task ⇒ tear down window, unblock and spawn dependents; `failed` task ⇒
    dependents `blocked`.
- FR3.5a There is exactly **one loop** in the system: the scheduler loop in window 0
  walks sessions, tasks, and window liveness in a single pass per tick. The executor
  is a function it calls, never a second loop.
- FR3.6 `itx session stop <id>` pauses execution (kills scheduler loop and task windows;
  in-flight tasks and the session move to `paused`).
- FR3.7 Re-running `execute` on a partially-done session is idempotent: reuses the
  terminal session and existing worktrees, spawns only tasks not in a terminal state.
- FR3.8 Harness selection: the supported set is a fixed v1 list — claude, opencode,
  pi. Default is auto-detected from PATH in that order; if none of the three is
  found, itx **fails fast** with a clear error. `~/.config/itx/config.yml`
  `default_harness` overrides detection;
  `itx session execute --harness pi|opencode|claude --model <slug>` overrides both.
  Per-task harness/model overrides are honored.
- FR3.9 Child harnesses run with the same user credentials/environment as the user's
  main harness (no separate auth).
- FR3.10 Terminal access goes through the terminal manager (tman) interface. v1 ships
  only the tmux backend; the interface (create session, add window, send command,
  liveness, kill) must allow future backends (wezterm, terminator, native terminals)
  without kernel/executor changes.
- FR3.11 Direct (non-interactive) LLM calls go through the LLM adapter. By default it
  uses the caller's harness provider/credentials; overridable in config.yml.

### FR4 — Skill behavior

- FR4.1 The skill guides the agent to build a session (clear DoD per task, explicit
  dependencies) interactively with the user, then persists it via the CLI.
- FR4.2 The skill NEVER edits the DAG file directly; every state change goes through
  the CLI.
- FR4.3 The skill can kick off, monitor (`session show`, `project status`), and stop
  execution. While a session runs, it reports progress to the user as % completion.

## Non-functional requirements

- NFR1 Deterministic kernel: given the same DAG, the scheduler loop makes the same
  spawn/reconcile decisions. LLM calls only ever happen through the LLM adapter,
  never inside scheduling/reconciliation logic.
- NFR2 Crash safety: scheduler loop restart recovers from the DAG file alone.
- NFR3 Observability: user can `tmux attach` to watch any task live; `session show`
  works from any terminal.
- NFR4 Zero config to start: only tmux + one harness required; config.yml optional.
- NFR5 Portability: POSIX installer (bash), Go binary with no cgo, no runtime deps
  beyond tmux/git/harness binaries.

## Out of scope (v1)

- Windows support (structure for it, don't build it).
- Standalone agent wrapping the skill.
- Merging/PR automation for task branches (user merges results).
- Cross-machine/remote execution.
- sqlite storage backend (adapter interface only).
- The archived GitHub workflow skills (`archived/skills/`) — kept for reference only.
