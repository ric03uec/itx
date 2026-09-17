# ITX — Domain Terminology

Single source of truth for terms. All docs, CLI surfaces, code identifiers, and file
names use these terms exactly.

## Core domain

| Term | Definition |
|---|---|
| **Project** | A working directory (usually a git repo) where itx operates. Identified by a **project slug** derived from the repo/directory name. State root: `~/.config/itx/projects/<project-slug>/`. |
| **Session** | One orchestrated unit of work within a project — typically corresponds to one project-level task (a feature, fix, or initiative). Identified by a **session slug**. Maps 1:1 to a terminal session named `{project-name}-{session-slug}`. |
| **Manifest** | The session's complete definition of work: the ordered task list plus session metadata (status, harness, model, timestamps). Everything that needs to be done in the session. Persisted as `manifest.json`; the single source of truth the kernel orchestrates from. |
| **Task** | One item in the manifest: title, task slug, definition of done, dependencies (`depends_on`), status, optional harness/model override, isolation mode. Executes in its own terminal window named `{session-slug}-{task-order}-{task-slug}`. |
| **Definition of Done (DoD)** | Per-task acceptance statement. A task may only be marked `complete` when its DoD is met. |
| **Status** | `pending | inprogress | blocked | complete | failed`. Applies to both sessions and tasks. `complete` and `failed` are terminal for tasks; a session is `complete` only when every task is `complete`. |
| **Work index** | `work.json` — per-project list of session ids not yet `complete`; the resume surface for `itx project status`. |

## Components

| Term | Definition |
|---|---|
| **Kernel** | The system core: orchestrator loop, state store, file locking, project management, task management, config management. Sole reader/writer of manifest, work index, and config. Decides *what* runs *when* (dependency graph, reconciliation, terminal states). |
| **Executor** | Kernel subcomponent that decides which node in the dependency tree runs next: owns the work queue, then materializes each scheduled task — prepares isolation (git worktree + branch), builds the launch command via the harness adapter, and runs it through the tman interface. |
| **Terminal Manager (tman)** | Abstraction over terminal multiplexers/emulators. Small interface: create terminal session, add terminal window, send command, check liveness, kill. v1 backend: tmux. Future backends: wezterm, terminator, native OS terminals. |
| **Harness Adapter** | Builds the interactive agent-harness launch command (claude, opencode, pi) from a command template + generated task prompt. Selection precedence: task override → session → `--harness` flag → config `default_harness` → PATH auto-detect. |
| **LLM Adapter** | Direct (non-interactive) LLM calls the system needs (e.g. slug generation, summaries, failure triage). Defaults to the caller's harness provider/credentials; overridable in config. |

## Terminal terms (tman vocabulary)

| tman term | tmux equivalent | Notes |
|---|---|---|
| **Terminal session** | tmux session (`tmux new-session`, i.e. what `<prefix>:new` creates) | One per itx session. Name: `{project-name}-{session-slug}`. |
| **Terminal window** | tmux window (`tmux new-window`, i.e. what `<prefix>c` creates) | One per task. Name: `{session-slug}-{task-order}-{task-slug}`. Window 0 is the kernel's controller loop. |

Other backends map these to their native concepts (e.g. wezterm: window/tab).

## Naming conventions

| Thing | Pattern | Example |
|---|---|---|
| Project slug | sanitized repo/dir basename | `itx` |
| Session id | `s-<date>-<rand>` | `s-20260917-a1b2` |
| Session slug | short human slug from session intent | `auth-refactor` |
| Terminal session | `{project-name}-{session-slug}` | `itx-auth-refactor` |
| Terminal window (task) | `{session-slug}-{task-order}-{task-slug}` | `auth-refactor-01-add-store` |
| Worktree dir | `{repo}-{session-slug}-{task-slug}` (sibling of repo) | `itx-auth-refactor-add-store` |
| Task branch | `itx/{session-slug}/{task-slug}` | `itx/auth-refactor/add-store` |
| Manifest file | `~/.config/itx/projects/<project-slug>/sessions/<session-id>/manifest.json` | |
| Work index | `~/.config/itx/projects/<project-slug>/work.json` | |
| Config | `~/.config/itx/config.yml` | |

## Deprecated terms

| Don't say | Say instead |
|---|---|
| todo / todo list / todo.json | task / manifest / manifest.json |
| panel | terminal window |
| tmux wrapper | terminal manager (tman) |
| store (as a standalone component) | kernel (store is inside the kernel) |
