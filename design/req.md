# ITX — Requirements

## Product statement

ITX is a parallel task orchestrator. It turns a **session manifest** — tasks with a
clear definition of done (DoD) and an explicit dependency hierarchy — into parallel,
isolated agent-harness executions inside tmux. Terminology is defined in
[DOMAIN.md](DOMAIN.md). It ships as:

1. A **Go CLI** (`itx`) — the deterministic core. All state lives on disk, all state
   changes go through the CLI.
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
  so Windows support can be added (OS dispatch in installer, harness/tmux behind
  interfaces in Go).
- FR1.4 The installer downloads a prebuilt binary from this repo's GitHub Releases,
  verifies its checksum, and installs to the user's PATH.
- FR1.5 After binary install, the user can install the skill into any harness via
  `itx skill install [claude|opencode|pi|all]`. The SKILL.md is embedded in the binary
  so skill and binary versions never drift.

### FR2 — Sessions and manifests (state management)
- FR2.1 `itx session new` creates a session and returns its id. Supports bulk creation
  from a JSON file (`--from manifest.json`).
- FR2.2 `itx task add` appends a task to the session manifest; each task carries a
  title, a definition of done, and a `depends_on` list.
- FR2.3 `itx task update --session <id> <task-slug> …` updates a task (status or
  arbitrary fields via JSON).
- FR2.4 `itx session update <id> --status [inprogress|pending|blocked|complete|failed]`
  updates session status.
- FR2.5 `itx session show <id>` reports full session state including per-task status
  and terminal-window liveness.
- FR2.6 `itx project status` reports all non-complete sessions for the current project
  (from `work.json`) so work can be resumed.
- FR2.7 State layout:
  - `~/.config/itx/projects/<slug>/sessions/<session-id>/manifest.json` — the session
    manifest: tasks in a sorted (dependency-ordered) array with status, DoD,
    harness/model used, timestamps, and granular per-task status.
  - `~/.config/itx/projects/<slug>/work.json` — only the ids of sessions not in
    `complete` state.
  - `~/.config/itx/config.yml` — user configuration (default harness, etc.).
- FR2.8 Concurrent writers (kernel loop + N task windows) MUST NOT corrupt state:
  writes are serialized with a file lock and are atomic (write-temp + rename).

### FR3 — Execution (orchestration)
- FR3.1 `itx session execute <id>` starts execution: creates one terminal session
  (named `{project-name}-{session-slug}`) via tman, runs the kernel loop in window 0,
  and returns. Execution survives the user's terminal closing.
- FR3.2 One terminal **window per task** (named
  `{session-slug}-{task-order}-{task-slug}`), each running its own harness instance. All
  unblocked tasks run in parallel (unlimited by default; optional `max_parallel` cap in
  config).
- FR3.3 Task isolation: in a git repo, each task gets its own **git worktree + branch**.
  Outside a git repo, tasks share the project directory and itx warns.
- FR3.4 The executor sends each task window a harness command with a generated
  prompt containing the task title, DoD, and the status-update contract.
- FR3.5 Status updates are belt & suspenders:
  - Task sessions report via `itx task update` (inprogress → complete/failed).
  - The kernel loop polls and reconciles: dead window while inprogress ⇒ failed;
    completed task ⇒ tear down window, unblock and spawn dependents.
- FR3.6 `itx session stop <id>` halts execution (kills kernel loop and task windows;
  in-flight tasks marked blocked).
- FR3.7 Re-running `execute` on a partially-done session is idempotent: reuses the
  terminal session and existing worktrees, spawns only what is still pending.
- FR3.8 Harness selection: default is auto-detected from PATH (claude → opencode → pi);
  `~/.config/itx/config.yml` `default_harness` overrides detection;
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
- FR4.1 The skill guides the agent to build a session manifest (clear DoD per task,
  explicit dependencies) interactively with the user, then persists it via the CLI.
- FR4.2 The skill NEVER edits state files directly; every state change goes through the
  CLI.
- FR4.3 The skill can kick off, monitor (`session show`, `project status`), and stop
  execution.

## Non-functional requirements

- NFR1 Deterministic kernel: given the same manifest.json, the kernel loop makes the
  same spawn/reconcile decisions. LLM calls only ever happen through the LLM adapter,
  never inside scheduling/reconciliation logic.
- NFR2 Crash safety: kernel loop restart recovers from manifest.json alone.
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
- The archived GitHub workflow skills (`archived/skills/`) — kept for reference only.
