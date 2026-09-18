# ITX — Revised Architecture Plan

Date: 2026-09-18

Status: proposed revision based on the architecture review and user feedback.
This plan supersedes conflicting choices in the original design documents for
planning purposes. It is not an implementation or acceptance sign-off.
Original inputs and decision provenance: [CHANGELOG.md](CHANGELOG.md).

## 1. One installation-wide scheduler

There is exactly one scheduler process and one scheduling loop for the entire
installation. Sessions register durable execution requests in the DAG. They never
start their own scheduler loops. The parent session is the logical orchestrator of
its tasks; the shared scheduler executes its orchestration policy.

- Host the scheduler in a dedicated ITX control tmux session, separate from all
  user-work sessions. Bootstrap it on first execute; verify a process handshake,
  not just the existence of a window.
- Hold an exclusive installation-wide scheduler lock for the process lifetime.
  Concurrent bootstrap calls converge on the same owner.
- Each tick reads the tree, reconciles worker observations, applies session
  controls, computes eligible actions, and persists action intents.
- Dispatch slow preparation, git/PR operations, and worker launches asynchronously.
  They return durable outcomes to the same scheduler; they are not additional
  scheduling loops. Do not hold the DAG write lock during external operations.
- `max_parallel` is installation-wide; count launch reservations as well as running
  workers. Stable ordering makes admission predictable.
- Pausing or cancelling one session does not stop the global scheduler.

This model works. The original contradiction was between “one global loop” and
“start a scheduler in every session,” not a limitation of global scheduling.
Examples: two simultaneous execute commands must not create two owners; a long
preparation command in session A must not prevent cancellation of session B; a
scheduler restart must recover registered sessions without launching duplicates.

## 2. State model first

See [STATE_MODEL.md](STATE_MODEL.md) for delegated Kubernetes research and the
user-directed ITX extensions, with meanings and outgoing transition triggers for
every task/session phase and user control.

- Phases: **Pending, Ready, Running, Blocked, Succeeded, Failed, Cancelled, Unknown**.
- Normal execution: **Pending → Ready → Running**. Ready means the scheduler can
  pick up the task; capacity waits remain Ready, dependency waits remain Pending.
- **Running → Failed** on execution failure or failed completion validation;
  **Running → Succeeded** when the definition of done validates.
- **Running → Blocked** when the agent needs user input and cannot proceed;
  **Blocked → Ready → Running** after required input is supplied and scheduling is
  permitted. No direct Blocked → Running shortcut.
- **Cancelled** is a terminal state. Session cancellation cancels all unfinished
  children immediately; worker shutdown is tracked separately and success is preserved.
- Execution control: **Active / Suspended** for user pause/re-arm. Ready, Blocked,
  and Cancelled are explicit ITX states, not Kubernetes Pod phases.
- **Failed → Ready** after the user fixes failure reasons and explicitly re-arms
  execution, with prior workers stopped and scheduling prerequisites satisfied.
  Scheduler dispatch then moves Ready → Running with a new attempt. Failed attempts
  never restart automatically; preserve their history. Manual completion still
  requires the unchanged completion artifacts.
- Parent controls propagate to unfinished children; confirmed worker state changes
  only after observation. Completion and approval remain separate.

This is a proposal awaiting user review, not a finalized state contract.

## 3. Session worktree, PR stacks, and merge order

- The session pins an exact main commit and creates **another, separate worktree
  on a new session branch off main**. The session worktree is not the main checkout
  and is not any task's worktree. It may hold the plan and other session-owned
  files as well as the integrated result. ITX must not write plans, switch branches,
  or apply task changes in the main checkout; main stays clean. Main moving later
  does not change the pinned base.
- Root tasks create separate worktrees from that pinned commit. Their PRs target
  the session integration branch.
- A task depending on A starts from A's recorded output commit and initially opens
  its PR against A's branch. Thus B can build on A before A is merged.
- The parent stores a dependency-respecting merge order using immutable task IDs,
  along with PR base/head revisions, merge status, and merge commits.
- Merge in that order into the session branch; reconcile/retarget dependent PRs
  after their parent merges. Persist intents before PR creation, retargeting, or
  merge, and reconcile the remote result before retrying an ambiguous operation.
- Proposed v1 stack policy: preserve ancestry with merge commits. Squash/rebase
  support requires explicit restacking behavior rather than silently changing SHAs.
- Proposed fan-in rule: a task requiring multiple branches waits for those outputs
  to integrate into the session branch, then pins that integrated commit. Conflicts
  block the session for intervention. One PR cannot have multiple base branches.
- Session completion requires all task outputs plus the integrated session result
  and its final PR against main. Final merge into main remains user-controlled.

Task completion, user approval, and PR merge status are separate records. The exact
merge authorization policy is a follow-up decision; ordering does not itself grant
permission to merge. Neither Succeeded nor successful merge alone authorizes cleanup.

```text
main@pinned-commit                     main checkout stays clean
├── session branch / separate worktree  plan + session files + integrated outputs
└── task A branch / separate worktree   PR targets session branch
    └── task B branch / worktree        PR initially targets task A branch
```

Root tasks still pin the selected main commit; any session planning files needed by
workers are passed as explicit inputs, not assumed to exist in their worktrees.

## 4. Durable launch protocol and worker wrapper

Persist `{action_id, session_id, task_id, attempt_id, control_generation, input_sha,
workspace, effective_launch_config, phase}` before any launch side effect.

Sequence: reserve → prepare/reconcile workspace → run preparation → launch tagged
wrapper → startup acknowledgement → running → outcome/exit receipt.

- Internal launch phases do not require additional public task states.
- Wrapper captures startup, harness exit code, stop acknowledgement, and outcome.
  A live shell/window is not worker liveness.
- Worker reports carry attempt ID and control generation. Cancellation/re-arm
  invalidates stale reports without deleting prior attempt evidence.
- Per-attempt ownership and durable receipts prevent repeated commands into an
  existing window. Scheduler restart adopts live work rather than rerunning it.
- Fast completion is accepted through the same acknowledgement/receipt protocol;
  it cannot race a late scheduler `running` commit and get overwritten.
- Unknown external outcomes are reconciled, not blindly replayed. This promises
  recoverable execution state, not uninterrupted execution or exactly-once effects
  for arbitrary commands issued by an agent.

## 5. Durable completion and approval-gated cleanup

Every completed task records its commit SHA, branch, and PR identity/URL alongside
the task-specific DoD result. Manual failure resolution has the same artifact
requirement. Approval records identify the exact reviewed revision; changing that
revision invalidates the applicable approval.

Cleanup requires explicit user approval of the task output or a session approval
covering that output revision. It also requires worker exit, durable pushed
outputs, and no remaining workspace consumer. Dirty/untracked work blocks removal
rather than being discarded. Keep branches/PR records and history; do not delete
stack dependencies while they are still needed. Cleanup has its own durable,
retryable action status and does not rewrite a task's successful outcome.

## 6. Minimal workspace and identity contract

- All session/task IDs are immutable. Slugs are display labels only. Include IDs
  in terminal, worktree, branch, and action identities.
- Add one deterministic configured workspace preparation command, executed by the
  wrapper in the recorded worktree before the harness, with exit status and logs.
  The command must tolerate rerun after interruption; do not silently retry an
  ambiguous non-idempotent preparation step.
- Record effective harness/model, working directory, and launch configuration.
  Pass environment explicitly; avoid depending on a tmux server's stale environment.
  Store secret references, not credential values. Use argument arrays/prompt files.
- Keep current harness support: claude, opencode, pi. Existing detection and
  override configuration remain; the wrapper adds supervision.
- Port, database, container, and other shared runtime resource isolation is deferred.
  Remote/cross-host execution is deferred. This revision is local-only.

## 7. JSON durability and project history

- Retain the JSON snapshot with revision-based optimistic concurrency and a narrow
  storage adapter. Add explicit schema version and migration handling.
- Use a separate stable lock file, revision validation under lock, temporary-file
  write and fsync, atomic rename, and directory fsync.
- Conflict retries reload and reapply only the state operation, with bounded
  exponential backoff/jitter and a surfaced exhaustion error. Never rerun external
  effects inside a storage retry closure.
- Keep per-project append-only JSONL history under the configured ITX project
  directory, keyed by immutable project ID. Events include action/attempt IDs,
  transitions, reasons, actors, and result references.
- Commit pending history events atomically with their state mutation in the DAG
  (an outbox). Flush them to JSONL with event-ID deduplication and fsync before
  acknowledging delivery. Interrupted final lines are detected/repaired under a
  history lock; pending events remain available for redelivery. History is an audit
  record, not a competing scheduler authority.
- Version history records as well as the snapshot. Reject incompatible writers;
  quiesce scheduling during incompatible migrations.
- Add real multi-process tests for lost updates, retry exhaustion, singleton
  ownership, interrupted writes, and snapshot/history recovery. JSON is the v1
  backend; SQLite remains future work.

## 8. Implementation order

1. **State contract:** settle task/session transition tables, propagation, explicit
   recovery, approval versus completion, and stale-worker behavior. Build the pure
   reducer and transition tests first.
2. **Durable state:** versioned DAG, optimistic transactions, bounded retries,
   file durability, JSONL outbox/history, and multi-process tests.
3. **Global scheduler:** registration, singleton lifecycle, deterministic admission,
   session controls, and nonblocking action dispatch with fake executors.
4. **Worker execution:** launch intents, wrapper receipts, process supervision,
   minimal workspace preparation, immutable resource IDs, real tmux + fake harness.
5. **Git/PR orchestration:** session worktree, pinned inputs, task PR stacks,
   persisted merge order, reconciliation, durable output validation, approval-gated
   cleanup. PR support now requires an explicit authenticated GitHub adapter;
   use `gh` for the initial implementation and preflight it before execution.
6. **Thin skill and distribution:** wire real harnesses, embed/install/update the
   skill, finish installer/releases and user-facing docs.

Existing execution.md and uat.md describe the earlier baseline. Reconcile their
detailed steps and acceptance matrix after this revised plan/state contract is
reviewed. The next review covers gaps and acceptance criteria, not additional
runtime-isolation or remote scope.
