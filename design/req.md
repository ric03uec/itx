# ITX — Requirements

Architecture and transition guards: [arch.md](arch.md). Verification: [uat.md](uat.md).

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
- FR2.3 `itx task update --session <id> <task-id> …` updates permitted fields through
  validated commands. JSON input cannot bypass transitions, evidence, or actor checks.
- FR2.4 Sessions/tasks use Pending, Ready, Running, Blocked, Succeeded, Failed,
  Cancelled (lowercase persisted/CLI values). Pending → Ready → Running is
  the normal launch path. Blocked means an agent needs user input, not a dependency
  wait. Running → Failed is explicit. After input, Blocked → Ready; after user repair
  and explicit re-arm, Failed → Ready. Neither transition bypasses scheduler admission.
  Users can declare Blocked or Failed work Succeeded with validated DoD, commit SHA,
  branch, PR, and no active worker. Succeeded and Cancelled are terminal; failed
  attempts retain their history even when the logical task is re-armed.
- FR2.5 `itx session show <id>` reports full session state — the session node plus its
  task subtree (the **manifest** view) — including per-task status, terminal-window
  liveness, worker observation, stop status, input requests, artifacts, and overall
  progress as **% completion** (succeeded tasks / total).
- FR2.6 `itx project status` reports all sessions of the current project not in a
  terminal state (a DAG query, not a separate file) so work can be resumed.
- FR2.7 State layout:
  - `~/.config/itx/dag.json` — single per-installation file holding the entire DAG
    (all projects, sessions, tasks, edges), attempts, action intents, approvals,
    pending history events, a schema version, and a revision counter.
  - `~/.config/itx/config.yml` — user configuration (default harness, storage
    backend, etc.).
- FR2.8 Concurrent writers (scheduler loop + N task windows) MUST NOT corrupt or lose
  state: every commit uses **optimistic locking** — it carries the revision it was
  read at; on mismatch the store rejects with a conflict and the writer re-reads and
  retries with bounded backoff/jitter. Writes use a separate stable lock file,
  temp-file fsync, atomic rename, and directory fsync. Retry closures contain only
  state changes, never external effects. Exhausted retries return a clear conflict.
- FR2.9 Storage is behind a **storage adapter** interface (load/commit-at-revision).
  v1 ships a single-JSON-file backend; the interface must allow swapping to sqlite
  (or other) with minimal effort and no kernel changes.
- FR2.10 Whenever a session or task is created or started — by a harness, an agent,
  or the user via the CLI — the **caller process id** is recorded on the node.
- FR2.11 Session and task nodes store their **workspace** records (`dir`,
  `is_worktree`, `branch`, input SHA) in the DAG. Reconciliation checks recorded
  ownership against actual resources before adoption or mutation.
- FR2.12 There is no `itx config` command; users edit `~/.config/itx/config.yml`
  directly.
- FR2.13 Immutable project/session/task/attempt IDs identify resources. Slugs are
  display labels; duplicate or renamed slugs cannot collide or change ownership.
- FR2.14 Maintain per-project append-only JSONL history with event-ID deduplication,
  actor/reason/action/attempt references, durable DAG outbox delivery, and torn-tail
  recovery. Version schemas and reject incompatible writers; quiesce for migration.

### FR3 — Execution (execution layer)

- FR3.1 `itx session execute <id>` durably registers work, ensures the singleton
  scheduler handshake, and returns. The scheduler lives in a dedicated control tmux
  session and holds a lifetime installation lock. Work sessions have separate
  terminals identified by project/session IDs. Closing the caller's terminal does
  not stop execution; repeat execute reconciles existing resources.
- FR3.2 One worker window per current task attempt, identified by task/attempt IDs,
  running the harness through a wrapper. Scheduler dispatches only Ready tasks;
  a task becomes Running after matching worker startup/resume acknowledgement,
  never just window existence. Global `max_parallel` includes reservations; 0 means
  unlimited. Persist launch intent before worktree/window/process side effects.
- FR3.3 Each session pins main's commit and creates its own **separate worktree and
  branch off main**, containing plans/session files and integrated results. Each
  task also has its own mandatory worktree/branch. Main's checkout stays untouched.
  Paths use immutable IDs under `~/.config/itx/projects/<project-id>/sessions/<session-id>/`.
  Root tasks use pinned main; dependent tasks pin upstream output commits. Non-git
  directories fail preflight before execution resources are created.
- FR3.4 The executor sends each task window a harness command with a generated
  prompt containing the task title, DoD, and the status-update contract. The
  calling harness that ran `execute` is instructed to report session progress as
  **% completion** while the session runs.
- FR3.5 Wrapper reports startup, input request, exit/outcome, and stop acknowledgement
  with attempt ID/control generation. Reconciliation uses receipts and process
  evidence: confirmed failure → Failed. Missing evidence retains the last confirmed
  state during bounded reconciliation of the specific action, with last evidence,
  error, and deadline visible. Expired acknowledgement/worker-observation deadlines
  become Failed with explicit timeout reasons, fence execution, and request stop.
  Retain ownership/capacity until exit or non-launch is confirmed; timeout alone
  never permits replacement. User-input waiting itself is not a timeout failure.
  Stale reports cannot overwrite controls or silently recover Failed. No duplicate
  worker on ambiguous launch and no additional catch-all lifecycle state.
- FR3.5a Exactly **one global scheduler loop** walks all registered sessions in
  bounded ticks. Slow preparation/git/launch runs asynchronously through the executor,
  without another scheduling loop or holding the DAG lock across external I/O.
- FR3.6 `itx session stop <id>` sets Suspended execution permission and propagates
  stops to unfinished children without stopping the global scheduler. Unstarted
  work becomes Pending; confirmed running interruption becomes Failed with reason
  Suspended. Preserve individual stop reasons and require explicit re-arm.
- FR3.7 Re-running execute reuses resources and never automatically retries Failed
  work or clears individual pauses. Failed → Ready requires explicit user repair/
  re-arm, prerequisite validation, and confirmed prior exit; create a new attempt.
- FR3.8 Harness selection: the supported set is a fixed v1 list — claude, opencode,
  pi. Default is auto-detected from PATH in that order; if none of the three is
  found, itx **fails fast** with a clear error. `~/.config/itx/config.yml`
  `default_harness` overrides detection;
  Effective precedence: task override → execute flag → session setting → config
  → auto-detect. Resolve and persist effective harness/model configuration per attempt.
- FR3.9 Child harnesses use the user's credentials (no separate auth) and explicitly
  supplied environment rather than stale tmux environment. Store secret references,
  not secret values. Workspace preparation is a simple deterministic, idempotent
  configured command with captured exit/logs; use safe argv/prompt-file construction.
- FR3.10 Terminal access goes through the terminal manager (tman) interface. v1 ships
  only the tmux backend; the interface (create session, add window, send command,
  liveness, kill) must allow future backends (wezterm, terminator, native terminals)
  without kernel/executor changes.
- FR3.11 Direct (non-interactive) LLM calls go through the LLM adapter. By default it
  uses the caller's harness provider/credentials; overridable in config.yml.
- FR3.12 Session cancellation sets the session and all unfinished children to
  Cancelled atomically, including Failed children; Succeeded history
  remains. Revoke launch/resume immediately and reconcile physical shutdown
  separately. A late completion cannot undo cancellation.
- FR3.13 The session records dependency-respecting merge order and PR base/head/merge
  revisions. A dependent PR initially targets the upstream task branch; retarget as
  ancestors integrate. Persist intents/reconcile ambiguous PR operations. Session
  output has its own final PR to main; final merge is user-controlled. Task merge
  authorization and stack/fan-in policy remain review decisions in arch.md.
- FR3.14 Succeeded always requires DoD + commit SHA + branch + PR and no active
  worker; this applies equally when a user marks a Blocked or Failed task done.
  Session success requires validated required child outputs and session artifacts.
- FR3.15 Cleanup requires explicit approval of task output or covering session
  output at its recorded revision, stopped workers, pushed results, no remaining
  consumers, and no dirty/untracked work. Changed output invalidates approval.
  Preserve branches/history; cleanup retries do not change successful status.

### FR4 — Skill behavior

- FR4.1 The skill guides the agent to build a session (clear DoD per task, explicit
  dependencies) interactively with the user, then persists it via the CLI.
- FR4.2 The skill NEVER edits the DAG file directly; every state change goes through
  the CLI.
- FR4.3 The skill can kick off, monitor (`session show`, `project status`), and stop
  execution. While a session runs, it reports progress to the user as % completion.

## Non-functional requirements

- NFR1 Deterministic kernel: given the same DAG, effective configuration, and recorded
  observations, the scheduler makes the same decisions. LLM calls go through the adapter,
  never inside scheduling/reconciliation logic.
- NFR2 Crash safety: recover durable intents from the DAG and reconcile wrapper
  receipts/external resources without duplicate dispatch or false completion.
- NFR3 Observability: user can `tmux attach` to watch any task live; `session show`
  works from any terminal.
- NFR4 Zero config to start: git, tmux, a supported harness, and authenticated `gh`
  for PR operations; config.yml optional. Preflight missing execution prerequisites.
- NFR5 Portability: POSIX installer (bash), Go binary with no cgo, no runtime deps
  beyond tmux/git/gh/harness binaries.
- NFR6 Validate concurrency/durability with multi-process contention and crash tests,
  including launch-before-ack, stop/completion races, and audit outbox recovery.

## Out of scope (v1)

- Windows support (structure for it, don't build it).
- Standalone agent wrapping the skill.
- Cross-machine/remote execution.
- Port, database, and container conflict isolation beyond simple workspace preparation.
- Squash/rebase stack rewriting (pending review of ancestry-preserving merge policy).
- sqlite storage backend (adapter interface only).
- The archived GitHub workflow skills (`archived/skills/`) — kept for reference only.
