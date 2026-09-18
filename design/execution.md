# ITX — Execution Plan

Build in this order: state transitions → durable storage → global scheduler →
wrapper/tmux execution → Git/PR completion → skill/distribution. Each step's exit
criteria gate the next. Contracts live in [arch.md](arch.md), terminology in
[DOMAIN.md](DOMAIN.md), product checks in [uat.md](uat.md).

## Step 1 — State contract and minimal scaffold

- Scaffold Go module `github.com/ric03uec/itx`, CLI/package skeleton, and Makefile
  build/test/lint/release-build targets. `itx version` works.
- Preserve legacy workflow history under `archived/`: old skills, workflow docs,
  and `.itx/` artifacts; inspect existing layout before moving resources.
- Implement pure state reducer for Pending, Ready, Running, Blocked, Succeeded,
  Failed, Cancelled; shared actor/guard validation for CLI and scheduler.
- Cover Pending → Ready → Running, Running → Blocked → Ready → Running,
  Running → Failed → Ready, and user Blocked/Failed → Succeeded with evidence.
- Model immutable task/attempt identity, control generations, Active/Suspended
  permission, parent cancellation/suspension, session aggregation, and physical
  stopping separately from durable state.
- Before locking affected behavior: review first-failure session policy, task merge
  authorization, ancestry-preserving merge/fan-in rules, and evidence/control payloads.

**UAT:** Walk the Mermaid diagram and actor/guard table in arch.md; include cancelled
sessions containing Pending/Ready/Running/Blocked/Failed/Succeeded tasks.

**Exit:** Reducer tests cover legal/illegal edges, manual completion, terminal-state
fencing, parent propagation, and stale outcomes. First-failure/control semantics
are explicitly settled before scheduler integration. No terminal/Git side effects
are needed to prove state transitions.

## Step 2 — Durable DAG, history, and CLI node management

- Store interface Load/Commit-at-revision; versioned JSON snapshot, stable lock file,
  temp fsync + rename + directory fsync, revision checks, bounded backoff/jitter.
- DAG nodes/edges, attempts, action intents, workspace/input revisions, evidence,
  approvals, and history outbox. Respect `XDG_CONFIG_HOME`.
- Per-project JSONL audit delivery: dedup event IDs, fsync before acknowledgement,
  torn-tail recovery, schema compatibility and quiesced migration.
- Commands: init, session new/bulk import/show/update, task add/update, project
  status. Stable IDs, caller provenance, cycle/unknown-reference checks, legal
  field/actor updates, % completion, input requests, worker/stop observations.
- Config-file defaults and immutable effective launch configuration; no config CLI.

**UAT:** Create a scratch DAG/session/tasks, reject cycles and illegal state writes,
verify Failed appears as recoverable work and manual success needs evidence.

**Exit:** Multi-process tests prove no lost updates/corrupt JSON under contention;
crash injection covers rename/fsync and outbox append/ack boundaries. External
effects are absent from retry closures. Duplicate slugs and renamed labels preserve
resource identity. Store interface has no JSON leakage.

## Step 3 — Global scheduler with fake executors

- One lifetime-locked installation-wide owner, session registration, process
  handshake, idempotent execute, hidden `itx scheduler run`.
- Bounded deterministic ticks, reconcile before scheduling, Ready admission,
  global capacity including reservations, asynchronous side-effect actions.
- Persist action/attempt/generation/input/config before dispatch; reconcile missing
  evidence against action-specific deadlines while retaining the last confirmed
  state. Deadline expiry → Failed with a concrete reason and fenced stop request;
  retain ownership/capacity until exit or non-launch is confirmed. Define deadline
  defaults before implementation; no replacement based solely on timeout and no
  second scheduler in a session.
- Integrate parent controls, cancellation fencing, explicit re-arm, and blocked
  input/resume admission. Keep unaffected sessions progressing.

**UAT:** Two simultaneous execute callers register different sessions with one owner;
slow preparation in one session does not prevent cancellation/progress in another.

**Exit:** Fake-executor tests cover duplicate execute, owner crash/restart, reserved
capacity, ambiguous launch, input resumption, first-failure policy, stop/re-arm,
and stale completion. No scheduling path uses an LLM or holds the DAG lock over I/O.

## Step 4 — Worker wrapper, tman, harnesses, and preparation

- tmux-backed tman for control/work sessions and task windows, identified by IDs.
- Wrapper startup, exit, input-request, and stop receipts with process/attempt
  ownership. Retained shell/window is not evidence of a live worker.
- Adapters for claude/opencode/pi; task → execute → session → config → auto-detect
  precedence, fail-fast preflight, safe argv/prompt files, explicit environment.
- Simple idempotent workspace preparation with exit/log capture; LLM adapter wiring
  stays outside scheduling. Defer port/database/container isolation.

**UAT:** Real tmux + fake harness: startup failure, quick success, worker exit leaving
a shell, lost acknowledgement, input request, cancellation during startup, scheduler
restart with a live worker, and explicitly repaired Failed → Ready.

**Exit:** Receipt reconciliation prevents duplicate execution at every tested launch
boundary. Stop acknowledgement precedes replacement/manual success/cleanup. Kernel
has no direct tmux calls. Fake artifact adapter can isolate worker tests until Step 5.

## Step 5 — Worktrees, PR stacks, completion, and cleanup

- Session worktree/branch off exact main SHA for plans/session files and integration;
  separate task worktrees; immutable IDs in paths/branches; main remains unchanged.
- Root tasks pin main; dependent tasks pin recorded upstream output SHA. Pass session
  planning inputs explicitly. Implement reviewed fan-in and stack merge policies.
- Authenticated gh adapter; PR identities, expected bases/heads, ordered integration,
  durable operation intents, and reconciliation of ambiguous remote success.
- Task and session completion validate DoD/commit/branch/PR; user manual completion
  from Blocked or Failed has identical evidence requirements and no active worker.
- Revision-scoped task/session approval; cleanup checks worker exit, pushed outputs,
  consumers, and dirty/untracked work. Preserve branches/history.

**UAT:** A → B stack and A+B → C fan-in consume the recorded code, integrate in order,
produce session PR, and leave main clean. Complete blocked work manually. Succeeded
worktrees persist before approval and dirty work prevents removal even after approval.

**Exit:** Git/PR adapter tests and sandbox integration verify ordered merges,
authorization, conflicts, stale heads, ambiguous PR operations, approval invalidation,
and cleanup retries. Final main merge remains user-controlled.

## Step 6 — Thin skill and embed

- `skills/itx/SKILL.md`: interview → DoD/deps → CLI insertion → plan confirmation →
  execute → monitor/report, surface input requests, explicit recovery and approval.
- `go:embed`; install claude/opencode/pi/all (verify actual harness directories).
  Never edit DAG directly. Refresh through `itx update`.

**UAT:** Run a two-task session via skill in Claude Code and opencode; best-effort pi.

**Exit:** CLI is sole state contract; installed content matches embedded version;
skill reports progress and never silently retries Failed or completes without evidence.

## Step 7 — Installer, CI, release, and documentation

- Installer: Linux/macOS amd64/arm64, Windows stub, tmux gate, GitHub Release checksum
  verification, `~/.local/bin`, PATH check, init/skill-install offer.
- Update binary and installed skills together, no-op if current; enforce schema
  compatibility and quiesce incompatible upgrades.
- CI test/lint; push `v-X.Y` → tests → four cross-compiled binaries → `YY.MM.PP`
  release + checksums. Adapt release skill from existing atx release workflow.
- Update README, AGENTS, CLAUDE pointer, CLI reference, and archive redirects to the
  shipped contracts. Design files remain the canonical design, not parallel plans.

**UAT:** Missing tmux, checksum tamper, clean Linux/macOS install, no-op/new release
update, and newcomer README-only skill-driven session. See uat.md for full matrix.

**Exit:** Build/test/lint pass; at least one real release; current docs match shipped
behavior; all applicable acceptance checks pass with results recorded.
