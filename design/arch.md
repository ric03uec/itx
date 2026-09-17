# ITX — Architecture & Design

Terminology: see [DOMAIN.md](DOMAIN.md). Requirements: see [req.md](req.md).

## The core idea

The entire system is a **DAG stored in one file per installation**. The root node is
the system, created at install/init. Projects are children of the root; sessions
(user objectives) are children of projects; tasks (sub-objectives) are children of
sessions, with dependency edges between sibling tasks. Everything else is one of
three layers operating on this data structure:

- **Insertion layer** — adds nodes (init, session new, task add, bulk import)
- **Management layer** — the kernel: state transitions, queries, reconciliation,
  optimistic-locking commits
- **Execution layer** — materializes `running` task nodes into harness sessions in
  terminal windows (executor → harness adapter → tman)

Scaling stays deterministic because scheduling is a pure function of the DAG: same
graph, same decisions, regardless of how many sessions and tasks exist.

## The DAG

```mermaid
flowchart TB
    R(("root<br/>(system)")) --> P1["project: itx"]
    R --> P2["project: blog"]
    P1 --> S1["session: auth-refactor<br/>goal: 'ship refactored auth'<br/>status: running"]
    P1 --> S2["session: fix-ci<br/>status: waiting"]
    S1 --> T1["task 01: add-store<br/>DoD + status: done"]
    S1 --> T2["task 02: wire-cli<br/>status: running"]
    S1 --> T3["task 03: docs<br/>status: waiting"]
    T2 -. depends on .-> T1
    T3 -. depends on .-> T2
```

Solid edges = hierarchy (child). Dotted edges = task dependency. Sessions correlate
1:1 with harness sessions; tasks correlate with the subtasks the harness works
through.

## State machine (sessions and tasks share it)

```mermaid
stateDiagram-v2
    [*] --> waiting: node inserted
    waiting --> running: scheduled (deps done)
    running --> done: goal met / DoD satisfied
    running --> paused: user stop
    paused --> running: resume
    running --> failed: died or gave up
    waiting --> blocked: upstream failed
    blocked --> waiting: unblocked / intervention
    done --> [*]
    failed --> [*]
```

`done` and `failed` are terminal. A `failed` task cascades `blocked` to its
dependents; the session goes `blocked` (or `failed` if nothing can proceed).

## High-level product diagram

```mermaid
flowchart TB
    subgraph user["User"]
        U["Terminal / Agent harness"]
    end

    subgraph insertion["INSERTION LAYER"]
        SK["itx skill<br/>installed per harness"]
        CLI["itx CLI<br/>init · session new · task add · import"]
    end

    subgraph kernel["KERNEL — management + execution layers"]
        LOOP["scheduler loop — THE single main loop<br/>walk sessions + tasks · schedule · reconcile · commit"]
        subgraph executor["executor (execution layer — called per tick, not a loop)"]
            QUEUE["work queue<br/>next task node with deps done"]
            LAUNCH["task launch<br/>worktree · prompt · command assembly"]
        end
        MGMT["node mgmt<br/>project · session · task · config"]
        SA["storage adapter<br/>optimistic locking"]
    end

    subgraph adapters["External adapters"]
        HAR["harness adapter<br/>claude · opencode · pi"]
        LLM["llm adapter<br/>default: caller's provider"]
    end

    subgraph tman["TERMINAL MANAGER (tman)"]
        TIF["interface:<br/>CreateSession · AddWindow<br/>SendCommand · IsAlive · Kill"]
        TMUX["tmux backend v1"]
        FUT["future: wezterm · terminator · native"]
    end

    subgraph storage["Storage"]
        DAG["~/.config/itx/dag.json<br/>single file · whole DAG"]
        SQL["future: sqlite"]
        CFG["config.yml"]
    end

    subgraph term["terminal session {project}-{session-slug}"]
        W0["window 0: scheduler loop<br/>itx session run"]
        W1["window {slug}-01-{task}"]
        W2["window {slug}-02-{task}"]
    end

    U -->|slash command| SK --> CLI
    U -->|direct| CLI
    CLI -->|insert nodes / transition states| kernel
    SA --> DAG
    SA -.-> SQL
    LOOP -->|unblocked tasks| QUEUE --> LAUNCH
    LAUNCH -->|build launch command| HAR
    kernel -.->|non-interactive calls| LLM
    LAUNCH -->|run command in new window| TIF
    TIF --> TMUX
    TMUX --> term
    W1 & W2 -->|itx task update| CLI
```

Component boundaries:

- **Kernel** — implements the management layer and owns the execution layer.
  There is exactly **one loop** in the whole system: the scheduler loop. Each tick
  it walks the DAG (sessions and tasks alike), reconciles reality (window liveness,
  reported statuses), schedules ready tasks, and commits. Everything else —
  executor, node management, storage adapter — is a function called from this loop.
  Sole writer of the DAG. Deterministic. Touches terminals only through tman,
  harnesses only through the harness adapter.
- **Executor (kernel subcomponent, not a loop)** — invoked by the scheduler loop
  each tick: picks the next task nodes whose dependency edges are all `done` (the
  work queue), then materializes each one (worktree, prompt, harness command via
  the harness adapter) and runs it through the tman interface.
- **Storage adapter** — narrow load/commit interface with optimistic locking. v1:
  single JSON file. Swappable to sqlite with minimal effort; no kernel changes.
- **tman (terminal manager)** — narrow interface (`CreateSession`, `AddWindow`,
  `SendCommand`, `IsAlive`, `Kill`); tmux is the only v1 backend; wezterm/terminator/
  native terminals slot in behind the same interface later.
- **Harness adapter** — command construction for the fixed v1 list: claude /
  opencode / pi. Auto-detect probes PATH in that order; if none of the three is
  found, execution **fails fast** with a clear error (no fallback).
- **LLM adapter** — direct non-interactive LLM calls (slug generation, summaries,
  failure triage). Defaults to the caller's harness provider; overridable in config.

## Key workflows

### 1. Install + init

```mermaid
flowchart LR
    A["curl install.sh | bash"] --> B{"OS?"}
    B -->|linux / darwin| C{"tmux present?"}
    B -->|windows| Z["print 'planned', exit"]
    C -->|no| Y["print install cmd<br/>brew/apt/dnf, exit 1"]
    C -->|yes| D["detect arch<br/>amd64/arm64"]
    D --> E["download binary + checksums<br/>from GitHub Releases"]
    E --> F["verify sha256"]
    F --> G["install to ~/.local/bin/itx"]
    G --> H["itx init — create DAG root"]
    H --> I["offer: itx skill install all"]
```

### 2. Plan a session (insertion layer)

```mermaid
sequenceDiagram
    actor User
    participant Agent as Agent (skill)
    participant K as Kernel (via CLI)

    User->>Agent: /itx "achieve outcome X"
    Agent->>User: interview: goal? DoD per task? dependencies?
    Agent->>K: itx session new --goal "…"
    K->>K: insert session node under project (project node auto-inserted under root on first use)
    Note over K: session-status=waiting · caller_pid recorded
    K-->>Agent: <session-id>
    loop each task (session already exists, id in hand)
        Agent->>K: itx task add --session <sid> --title … --dod … --depends-on …
        K->>K: insert task node + dependency edges (cycle check)
        Note over K: task-status=waiting
    end
    Agent->>K: itx session show <sid>
    Agent->>User: confirm plan (goal, tasks, DoD, deps)
    User->>Agent: go
    Agent->>K: itx session execute <sid>
    loop while session-status=running
        Agent->>K: itx session show <sid>
        Agent->>User: report progress — % completion (done tasks / total)
    end
```

`itx session new` always comes first and returns the id; tasks are then attached to
that id one CLI call at a time (or all at once with `--from`, where the file carries
the goal plus the full task list). After `execute`, the calling harness is
instructed to keep reporting session progress as **% completion**.

### 3. Execute / orchestrate (management + execution layers)

```mermaid
sequenceDiagram
    participant EXE as itx session execute
    participant T as tman
    participant K as scheduler loop (window 0)
    participant X as executor (called by loop)
    participant W as task windows (harness)
    participant D as DAG (via storage adapter)

    Note over EXE: every step below is idempotent — re-running execute is always safe
    EXE->>T: CreateSession {project}-{session-slug} (no-op if exists)
    EXE->>T: window 0 ← itx session run <sid> (no-op if already running)
    EXE->>T: IsAlive(window 0)? — verify the loop is actually live
    EXE->>D: session → running · record caller_pid
    Note over D: session-status=running — set only AFTER the loop is live.<br/>A halfway failure leaves session-status=waiting, re-execute restarts cleanly
    EXE-->>EXE: return (execution detached)

    loop THE single scheduler loop — every N sec, sessions + tasks + everything
        K->>D: load DAG (revision R)
        K->>X: reconcile + schedule in one pass
        alt task ready (deps done, under max_parallel)
            X->>X: create worktree under session dir + branch, build prompt + harness cmd
            X->>T: AddWindow {session-slug}-{order}-{task-slug}, SendCommand
            X->>T: IsAlive(window)? — verify spawn
            X->>D: task → running · record window + caller_pid (commit @R, retry on conflict)
            Note over D: task-status=running — set only AFTER the window is live
        end
        W->>D: itx task update --status done / failed
        K->>T: IsAlive(window)?
        alt window dead while task-status=running
            K->>D: task-status=failed
        end
        alt task-status=done
            K->>T: Kill(window)
            K->>K: dependents become schedulable
        end
        alt task-status=failed
            K->>D: dependents task-status=blocked · session-status=blocked
        end
        K->>K: print progress — % completion (done tasks / total)
    end
    K->>D: all tasks done → session-status=done · remove worktrees (branches kept)
```

One loop, not two: the scheduler loop in window 0 is the only loop in the system —
it walks sessions, tasks, liveness, and terminal states in a single pass per tick.
The executor is a function it calls, not a second loop. Every operation (`execute`,
CreateSession, AddWindow, state commits) is idempotent, so a crash at any point is
recovered by simply running `execute` again.

### 4. Resume / pause

```mermaid
flowchart LR
    A["itx project status"] --> B["DAG query:<br/>sessions of this project not done/failed"]
    B --> C{"user picks"}
    C -->|resume| D["itx session execute sid<br/>idempotent: reuse terminal session/worktrees,<br/>schedule only waiting tasks<br/>session-status=running"]
    C -->|pause| E["itx session stop sid<br/>kill windows + scheduler loop<br/>session-status=paused, running tasks → paused"]
```

## Storage

### Storage adapter interface

```go
type Revision uint64

type Store interface {
    Load(ctx context.Context) (*DAG, Revision, error)
    // Commit persists the DAG iff the stored revision still equals expected.
    // Returns ErrConflict otherwise; caller re-loads, re-applies, retries.
    Commit(ctx context.Context, dag *DAG, expected Revision) error
}
```

Optimistic locking: every writer follows load → mutate → commit-at-revision; on
`ErrConflict` it re-loads and retries. The JSON backend implements Commit with an
exclusive flock + revision check + write-temp-then-rename, so writes are atomic and
predictable under N concurrent task windows. A sqlite backend implements the same
interface with a transaction — no kernel changes.

### `~/.config/itx/dag.json` (single file per installation)

```json
{
  "revision": 42,
  "nodes": [
    { "id": "root", "type": "root", "created_at": "2026-09-17T09:00:00Z" },
    {
      "id": "p-itx", "type": "project",
      "name": "itx", "slug": "itx",
      "dir": "/home/devashish/workspace/ric03uec/itx",
      "created_at": "…"
    },
    {
      "id": "s-20260917-a1b2", "type": "session",
      "slug": "auth-refactor",
      "goal": "ship refactored auth with tests",
      "status": "running",
      "harness": "claude", "model": "",
      "caller_pid": 40100,
      "created_at": "…", "started_at": "…", "finished_at": ""
    },
    {
      "id": "t-c3d4", "type": "task",
      "order": 1, "slug": "add-store",
      "title": "Add store package",
      "goal": "storage adapter package",
      "definition_of_done": "adapter with optimistic locking; unit tests pass",
      "status": "done",
      "harness": "", "model": "",
      "workspace": {
        "dir": "~/.config/itx/projects/itx/sessions/s-20260917-a1b2/worktrees/add-store",
        "is_worktree": true,
        "branch": "itx/auth-refactor/add-store"
      },
      "window": "auth-refactor-01-add-store",
      "caller_pid": 43210,
      "created_at": "…", "started_at": "…", "finished_at": "…",
      "notes": ""
    }
  ],
  "edges": [
    { "from": "root",             "to": "p-itx",            "type": "child" },
    { "from": "p-itx",            "to": "s-20260917-a1b2",  "type": "child" },
    { "from": "s-20260917-a1b2",  "to": "t-c3d4",           "type": "child" },
    { "from": "t-e5f6",           "to": "t-c3d4",           "type": "dependency" }
  ]
}
```

Rules:
- `root` is created once by `itx init`; project nodes are inserted on first use in a
  directory; the graph is append-mostly (nodes are never re-parented).
- Statuses per the shared state machine: `waiting | running | paused | blocked |
  done | failed`; only session and task nodes carry status.
- `dependency` edges only between tasks of the same session; cycle-checked on insert.
- Per-task `harness`/`model` empty ⇒ inherit session ⇒ config ⇒ auto-detect
  (claude → opencode → pi only; none found ⇒ fail fast).
- `workspace` is tracked in the node itself — `dir`, `is_worktree`, `branch` — never
  inferred from disk. Worktrees are the only way of working (no exceptions): created
  automatically under
  `~/.config/itx/projects/{project}/sessions/{session-id}/worktrees/{task-slug}`
  when a task is scheduled, removed automatically when the session reaches `done`
  (branches are kept for merging). A project dir that is not a git repo fails fast
  at `execute`.
- `caller_pid` is recorded whenever a session or task is created or started —
  the CLI captures its parent process id, so it works identically for a harness,
  an agent, or a user driving the CLI directly.
- "Active work" is a query (sessions not `done`/`failed`), not a separate file.

### `~/.config/itx/config.yml`

```yaml
default_harness: ""        # "" → auto-detect PATH: claude → opencode → pi (fixed list; none found → fail fast)
max_parallel: 0            # 0 = unlimited
poll_interval_seconds: 30
terminal: tmux             # tman backend; only tmux in v1
storage: json              # storage adapter backend; sqlite later
harnesses:                 # command templates; user-fixable without a new binary
  claude:   'claude --dangerously-skip-permissions {{prompt}}'
  opencode: 'opencode {{prompt}}'
  pi:       'pi {{prompt}}'
llm:                       # LLM adapter override; empty → caller's provider
  provider: ""
  model: ""
```

## CLI surface

```
itx init                                 # create DAG root (run once, by installer)
itx session new --goal G [--slug S] [--harness X] [--model Y]
                                         # create empty session with a goal, print id
itx session new --from plan.json         # bulk create: file carries the session goal
                                         # PLUS the full task list (title/DoD/deps) —
                                         # session + all tasks inserted in one call
itx session show <id>                    # manifest view + % completion
itx session update <id> --status [waiting|running|paused|blocked|done|failed]
itx session execute <id> [--harness X] [--model Y]   # idempotent
itx session stop <id>                    # session-status: running → paused
itx session run <id>                     # hidden; scheduler loop (window 0)
itx task add --session <id> --title T --dod D [--depends-on a,b] [--json '{…}']
itx task update --session <id> <task-slug> [--status S] [--json '{…}']
itx project status                       # DAG query: this project's non-terminal sessions
itx skill install [claude|opencode|pi|all]
itx update                               # update binary + installed skills to latest
                                         # release; idempotent — no-op when current
itx version
```

No `itx config` command: configuration is `~/.config/itx/config.yml`, edited
directly by the user.

## tman interface

```go
type TerminalManager interface {
    CreateSession(name string) error            // idempotent
    HasSession(name string) bool
    AddWindow(session, window, dir string) error
    SendCommand(session, window, cmd string) error
    IsAlive(session, window string) bool
    KillWindow(session, window string) error
    KillSession(name string) error
}
```

v1: `tmuxBackend` implements this with `tmux new-session / new-window / send-keys /
list-windows / kill-window / kill-session`. Backend chosen by `terminal:` in
config.yml; adding wezterm/terminator/native = new implementation, no caller changes.

## Package layout

```
cmd/itx/main.go
internal/cli/        # cobra command wiring only (insertion + query surface)
internal/kernel/     # THE core: single scheduler loop, state machine, node mgmt,
                     # dep resolution, reconciler, config load
internal/kernel/executor/  # work queue: next task node with deps done; task
                           # materialization (worktree, prompt, launch via tman)
internal/kernel/store/     # storage adapter interface + JSON backend (OCC);
                           # future: sqlite backend
internal/tman/       # TerminalManager interface + tmux backend
internal/harness/    # harness adapters + PATH auto-detect
internal/llm/        # LLM adapter (caller-default, config override)
internal/gitx/       # git helpers: toplevel, slug, worktree add/remove
skills/itx/SKILL.md            # go:embed → itx skill install
skills/itx-release/SKILL.md
install.sh
.github/workflows/{ci,release}.yml
archived/skills/     # former GitHub-workflow skills (reference only)
```

## Design decisions

| Decision | Choice | Why |
|---|---|---|
| Core data structure | single DAG: root → projects → sessions → tasks (+ dependency edges) | scheduling is a pure function of the graph — deterministic scaling as work grows |
| Storage | one file per installation (`dag.json`), optimistic locking, behind a storage adapter | predictable concurrent writes; human-readable v1; sqlite swap without kernel changes |
| State model | shared 6-state machine (waiting/running/paused/blocked/done/failed) for sessions and tasks | one transition diagram to reason about; failed vs blocked separates terminal from recoverable |
| CLI language | Go, static binary | curl-install from GH Releases; real JSON; clean Windows path later |
| Core shape | single kernel with ONE scheduler loop (executor/node mgmt/storage adapter are functions it calls) | one writer, one loop, one source of truth |
| Terminal access | tman interface, tmux backend v1 | swap in wezterm/terminator/native without touching kernel |
| Parallelism | 1 terminal session / itx session; 1 window / task | watchable, killable, survives terminal close |
| Scheduler loop home | window 0 of the terminal session | no daemon plumbing; user-visible; deterministic Go loop |
| Isolation | mandatory git worktree + branch per task, under `~/.config/itx/projects/…/sessions/…/worktrees/` — auto-created, auto-removed on session done (branches kept); non-git dirs fail fast | no exceptions to reason about; parallel edits can't collide; workspace tracked in the node (`is_worktree`), never inferred |
| Status truth | tasks report via CLI + kernel reconciles | robust to children forgetting; dead window → failed |
| Running means running | `running` committed only after terminal session / window is verified live; every execute step idempotent | halfway failures leave clean state; re-execute always safe |
| Caller identity | `caller_pid` recorded on create/start of sessions and tasks | traceable whether driven by harness, agent, or raw CLI |
| Updates | one `itx update` — binary + installed skills together, no-op when current | versions never drift; no separate skill update to forget |
| Concurrency | unlimited default, `max_parallel` opt-in | user asked for max parallelism by default |
| LLM calls | separate LLM adapter, defaults to caller's provider | keeps kernel deterministic; direct calls isolated and overridable |
| Skill distribution | embedded in binary, `itx skill install` | skill version always matches binary |
| Releases | atx model: push v-X.Y → CI tags YY.MM.PP + binaries | proven pipeline; installer reads GH Releases |
| Windows | stubs + interfaces only | requirement: structure for it, don't build it |
