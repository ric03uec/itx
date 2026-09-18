# ITX — Domain Terminology

> Revision in progress: the proposed state and ownership contracts in
> [PLAN.md](PLAN.md) supersede conflicting baseline definitions below for planning.

Single source of truth for terms. All docs, CLI surfaces, code identifiers, and file
names use these terms exactly.

## The DAG (core data structure)

The entire system state is one **DAG** (directed acyclic graph), stored in a single
file per installation. Everything else — insertion, management, execution — is a
layer operating on this structure.

| Term | Definition |
|---|---|
| **Root** | The single system node. Created when itx is installed and initialized (`itx init`). Parent of all projects. |
| **Node** | An element of the DAG: `root`, `project`, `session`, or `task`. Every node has an id, type, timestamps; project/session/task nodes carry a goal and (session/task) a status. |
| **Edge** | A directed link between nodes. Two kinds: **child** edges (root→project→session→task hierarchy) and **dependency** edges (task→task ordering within a session). |
| **Project** | Child of the root. A working directory (usually a git repo) where itx operates, identified by a **project slug**. |
| **Session** | Child of a project. An **objective** — an outcome the user wants to achieve. Correlates 1:1 with a harness session at execution time, and 1:1 with a terminal session named `{project-name}-{session-slug}`. |
| **Task** | Child of a session. A **sub-objective** required to complete the session; carries a definition of done and dependency edges to sibling tasks. Correlates with the subtasks the harness works through. Executes in its own terminal window named `{session-slug}-{task-order}-{task-slug}`. |
| **Goal** | The objective text on a node: for a session, the user-provided outcome; for a task, the sub-objective plus its Definition of Done. |
| **Definition of Done (DoD)** | Per-task acceptance statement. A task may only transition to `done` when its DoD is met. |
| **Manifest** | Logical view, not a file: a session node plus its task subtree (what `itx session show` renders). |
| **Workspace** | Per-task working-directory record stored **on the task node** (`dir`, `is_worktree`, `branch`) — never inferred from disk. Always a git worktree, no exceptions: auto-created when the task is scheduled, auto-removed when the session reaches `done` (branch kept). |
| **Caller PID** | Process id of whatever invoked the CLI to create or start a session/task (harness, agent, or user shell). Recorded on the node. |

## State machine

Sessions and tasks share the same state transition diagram:

| State | Meaning |
|---|---|
| `waiting` | Created; not yet running (dependencies unmet or not yet scheduled). |
| `running` | Actively being executed. |
| `paused` | Halted by the user (`itx session stop`); resumable. |
| `blocked` | Cannot proceed — upstream failure or needs intervention. |
| `done` | Objective met (task: DoD satisfied). **Terminal.** |
| `failed` | Died or gave up (e.g. dead window while running). **Terminal.** Cascades `blocked` to dependents. |

`done` and `failed` are the only terminal states.

## Storage

| Term | Definition |
|---|---|
| **DAG file** | `~/.config/itx/dag.json` — the single per-installation file holding the entire DAG (all projects, sessions, tasks, edges). |
| **Optimistic locking** | Every commit carries the revision it was read at; the store rejects the write (`conflict`) if the revision moved, and the writer re-reads and retries. Makes concurrent writes predictable. |
| **Storage adapter** | The narrow interface the kernel uses to load/commit the DAG. v1 backend: single JSON file. Swappable to sqlite (or other) with minimal effort. |

## Layers

| Term | Definition |
|---|---|
| **Insertion layer** | Everything that adds nodes to the DAG: `itx init` (root), project registration, `itx session new`, `itx task add`, bulk import — driven by the CLI directly or by the skill. |
| **Management layer** | The **kernel**: state transitions, dependency resolution, queries (`session show`, `project status`), reconciliation, optimistic-locking commits. Sole writer of the DAG. |
| **Execution layer** | Materializes `running` nodes into real work: the kernel's **executor** (work queue — picks the next task node whose dependencies are `done`), harness adapter, LLM adapter, and tman. |

## Components

| Term | Definition |
|---|---|
| **Kernel** | The system core: management layer + execution ownership. Scheduler loop, executor, storage adapter, project/session/task/config management. Deterministic. Touches terminals only through tman, harnesses only through the harness adapter. |
| **Scheduler loop** | THE single main loop of the system (window 0). Each tick it walks sessions, tasks, and window liveness in one pass: reconciles, schedules, commits. There is no other loop. |
| **Executor** | Kernel subcomponent **called by the scheduler loop each tick — not a loop itself**. Decides which task node runs next (work queue), then materializes each scheduled task — worktree + branch, launch command via the harness adapter — and runs it through the tman interface. |
| **Terminal Manager (tman)** | Abstraction over terminal multiplexers/emulators. Small interface: create terminal session, add terminal window, send command, check liveness, kill. v1 backend: tmux. Future: wezterm, terminator, native OS terminals. |
| **Harness Adapter** | Builds the interactive agent-harness launch command from a command template + generated task prompt. Fixed v1 list: claude, opencode, pi. Selection precedence: task override → session → `--harness` flag → config `default_harness` → PATH auto-detect; none of the three found ⇒ fail fast. |
| **LLM Adapter** | Direct (non-interactive) LLM calls the system needs (e.g. slug generation, summaries, failure triage). Defaults to the caller's harness provider/credentials; overridable in config. |

## Terminal terms (tman vocabulary)

| tman term | tmux equivalent | Notes |
|---|---|---|
| **Terminal session** | tmux session (`tmux new-session`, i.e. what `<prefix>:new` creates) | One per itx session. Name: `{project-name}-{session-slug}`. |
| **Terminal window** | tmux window (`tmux new-window`, i.e. what `<prefix>c` creates) | One per task. Name: `{session-slug}-{task-order}-{task-slug}`. Window 0 is the scheduler loop. |

Other backends map these to their native concepts (e.g. wezterm: window/tab).

## Naming conventions

| Thing | Pattern | Example |
|---|---|---|
| Project slug | sanitized repo/dir basename | `itx` |
| Session id | `s-<date>-<rand>` | `s-20260917-a1b2` |
| Session slug | short human slug from session goal | `auth-refactor` |
| Task id | `t-<rand>` | `t-c3d4` |
| Terminal session | `{project-name}-{session-slug}` | `itx-auth-refactor` |
| Terminal window (task) | `{session-slug}-{task-order}-{task-slug}` | `auth-refactor-01-add-store` |
| Worktree dir | `~/.config/itx/projects/{project}/sessions/{session-id}/worktrees/{task-slug}` | `…/projects/itx/sessions/s-20260917-a1b2/worktrees/add-store` |
| Task branch | `itx/{session-slug}/{task-slug}` | `itx/auth-refactor/add-store` |
| DAG file | `~/.config/itx/dag.json` | |
| Config | `~/.config/itx/config.yml` | |

## Deprecated terms

| Don't say | Say instead |
|---|---|
| todo / todo list / todo.json | task / manifest (logical view) / DAG |
| manifest.json, work.json (as files) | dag.json — sessions/tasks are DAG nodes; "active work" is a DAG query |
| pending / inprogress / complete | waiting / running / done |
| panel | terminal window |
| tmux wrapper | terminal manager (tman) |
| store (as a standalone component) | storage adapter (inside the kernel) |
| orchestrator loop / kernel loop / executor loop | scheduler loop (there is only one loop) |
| itx skill update / itx config get,set | itx update / edit config.yml directly |
