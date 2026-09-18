# ITX — Domain Terminology

Shared vocabulary for docs, CLI, and code. State transitions and guards are defined
in [arch.md](arch.md#state-transitions).

## DAG and execution records

| Term | Definition |
|---|---|
| **Root** | Single installation/system node, created idempotently by `itx init`. |
| **Node** | Root, project, session, or task with immutable ID/type/timestamps. Sessions/tasks carry status. |
| **Edge** | Child hierarchy (root → project → session → task) or dependency between sibling tasks. |
| **Project** | Repository registered under root, identified by immutable ID; slug is a display label. |
| **Session** | User objective under a project; logical orchestrator of tasks, worktree, PR stack, and merge order. Registers with the global scheduler. |
| **Task** | Sub-objective with DoD/dependencies; executes in its own worktree and worker window. |
| **Attempt** | One execution of a task with immutable identity, launch configuration, receipts, and retained outcome. Failure repair creates another attempt. |
| **Action intent** | Durable record authorizing a specific launch, resume, git/PR operation, stop, or cleanup before its side effects. |
| **Goal** | User-desired session outcome or task sub-objective. |
| **Definition of Done (DoD)** | Acceptance statement plus durable commit SHA, branch, and PR evidence required for Succeeded, including manual completion. |
| **Manifest** | Logical session/task subtree rendered by `session show`, not a separate state file. |
| **Workspace** | Stored path/worktree/branch/input-SHA record for a session or task; removal requires approved output and cleanup guards. |
| **Approval** | Explicit user acceptance of a particular output revision; separate from completion and merge. |
| **Caller PID** | Process provenance on create/start; not sufficient for worker identity or liveness. |
| **Control generation** | Fencing value that prevents stale attempt reports from overriding newer user controls. |

## States

Display names are capitalized; persisted/CLI values are lowercase.

| State | Meaning |
|---|---|
| **Pending** | Waiting for prerequisites or execution permission; cannot be dispatched. |
| **Ready** | Eligible for scheduler pickup, including waiting for capacity. |
| **Running** | Worker/orchestration startup or resumption acknowledged. |
| **Blocked** | Required user input prevents progress; not an ordinary dependency wait. |
| **Succeeded** | DoD and durable commit/branch/PR validated; no active work. Terminal. |
| **Failed** | Definitive failure; user can fix reasons and re-arm into Ready or explicitly complete with evidence. No automatic retry. |
| **Cancelled** | User abandoned task/session; no future dispatch, shutdown reconciled separately. Terminal. |

Blocked can move to Ready after input or directly to Succeeded when the user declares
completion and evidence validates. Failed can move to Ready after repair or to
Succeeded through explicit evidenced completion. Succeeded and Cancelled are the
terminal logical-task states; individual attempts retain terminal history.

**Active/Suspended** is the separate permission control for user pause/re-arm.
**Stopping** describes pending physical shutdown, not an additional lifecycle state.
Missing evidence is tracked against a specific action and bounded reconciliation
deadline, retaining the last confirmed state. Deadline expiry becomes Failed with
an explicit timeout reason; it does not prove worker exit or permit duplicate dispatch.

## Layers and components

| Term | Definition |
|---|---|
| **Insertion layer** | CLI/skill operations adding validated nodes and edges. |
| **Management layer** | Kernel reducer, queries, dependency resolution, reconciliation, and transactions. |
| **Execution layer** | Executor, wrapper, harness adapter, git/PR adapter, and tman materializing Ready work. |
| **Kernel** | Owns state semantics and execution policy. CLI writers and scheduler share the same transactional validation. |
| **Scheduler** | Exactly one installation-wide loop in a dedicated control tmux session, protected by lifetime lock. Sessions register with it. |
| **Executor** | Asynchronous action execution dispatched by the scheduler; no separate scheduling loop. |
| **Worker wrapper** | Reports startup, input requests, exit/outcome, and stop acknowledgements for an identified attempt. |
| **Terminal manager (tman)** | Terminal resource interface; tmux in v1, other backends later. Window existence is not agent liveness. |
| **Harness adapter** | Builds claude/opencode/pi commands. Precedence: task override → execute flag → session → config → PATH detection. |
| **LLM adapter** | Optional direct calls outside scheduling; caller-provider default, config override. |
| **Storage adapter** | Load/commit-at-revision interface; versioned JSON in v1, SQLite later. |
| **Optimistic locking** | Revision conflict detection plus bounded reload/reapply/backoff; no external side effects inside retries. |
| **History outbox** | Audit events committed with DAG changes before durable delivery to per-project append-only JSONL. |

## Resource identity and naming

Names include immutable IDs; slugs only improve display. Renaming a goal/slug cannot
change worker, branch, or workspace identity.

| Resource | Pattern |
|---|---|
| Project/session/task/attempt IDs | Unique stable IDs, e.g. `p-…`, `s-…`, `t-…`, `a-…` |
| Work terminal session | `itx-{project-id}-{session-id}` |
| Task window | `{task-id}-{attempt-id}` |
| Global scheduler terminal | Dedicated installation control session, not a work-session window |
| Session branch | `itx/{session-id}` |
| Task branch | `itx/{session-id}/{task-id}` |
| Session worktree | `~/.config/itx/projects/{project-id}/sessions/{session-id}/worktree` |
| Task worktree | `~/.config/itx/projects/{project-id}/sessions/{session-id}/worktrees/{task-id}` |
| Snapshot/config | `~/.config/itx/dag.json`, `config.yml` |
| Project history | `~/.config/itx/projects/{project-id}/history.jsonl` |

Honor `XDG_CONFIG_HOME` for the configured ITX directory. Work terminals have one
worker window per current task attempt; the scheduler has a separate terminal.

## Deprecated terms

| Don't say | Say instead |
|---|---|
| todo.json / manifest.json as state files | DAG; manifest is a logical query |
| waiting / done as lifecycle values | pending / succeeded |
| blocked for ordinary dependency wait | pending with dependency reason |
| panel | terminal window |
| executor loop / per-session scheduler | global scheduler; executor action |
| itx skill update / itx config get,set | itx update / edit config.yml |
