# ITX — Acceptance & Verification (UAT)

Verification targets for [arch.md](arch.md) and [req.md](req.md), not a claim that
these behaviors are implemented or tested. Run product scenarios on Linux amd64
and macOS arm64. Use public CLI/skill/tmux surfaces; concurrency and crash checks
also use controlled integration-test hooks. Settle marked architecture policy
decisions before finalizing the corresponding tests.

## A. Installation and distribution

| # | Scenario | Pass criteria |
|---|---|---|
| A1 | Fresh install with tmux | Checksum-verified binary installed at `~/.local/bin/itx`; version works; init/skill install offered. |
| A2 | Missing tmux / unsupported OS | Platform-correct brew/apt/dnf guidance; missing tmux exits non-zero; Windows planned-support message, no partial install. |
| A3 | Corrupt release binary | Checksum mismatch aborts installation. |
| A4 | Init twice / commands before init | One root and no duplicate initialization; clear prerequisite error before init. |
| A5 | Skill install all | Installed harnesses receive embedded skill; absent harnesses reported without fatal error. |
| A6 | Update | Current version no-op; new release refreshes binary and installed skills; incompatible schema requires quiesced migration. |
| A7 | Release pipeline | `v-X.Y` produces tested `YY.MM.PP` release, four binaries/checksums; README installer retrieves that release. |
| A8 | Release skill | Release notes/archive follow repository release workflow and published assets match version. |

## B. DAG and state transitions

| # | Scenario | Pass criteria |
|---|---|---|
| B1 | Session create, task add, bulk import | IDs returned, hierarchy/dependencies and caller provenance recorded; invalid edges/cycles reject atomically. |
| B2 | Pending → Ready → Running | Prerequisites gate Ready; only scheduler dispatch and worker acknowledgement permit Running. Window existence alone does not. |
| B3 | Agent needs input | Running → Blocked with request; accepted response → Ready; scheduler dispatch → Running; no autonomous bypass. |
| B4 | User completes blocked work | Blocked → Succeeded after explicit user declaration, verified DoD/commit/branch/PR, and worker shutdown; missing evidence or active worker rejects success. Audit records manual resolution. |
| B5 | Failure and user repair | Running → Failed, stays Failed without user action; fixed reasons + explicit re-arm + satisfied prerequisites + prior exit → Ready; new attempt retains old failure. |
| B6 | User resolves failed work | Failed → Succeeded only with the same completion evidence and no active worker; never invent successful attempt history. |
| B7 | Session cancellation | Cancel a session containing Pending, Ready, Running, Blocked, Failed, and Succeeded tasks: all unfinished children become Cancelled atomically, Succeeded preserved, admission revoked, physical stops reconciled. |
| B8 | Cancellation race | Late startup/success/failure receipts cannot undo Cancelled. Confirm stopped before replacement or cleanup; terminal children retain history. |
| B9 | Terminal and invalid writes | Succeeded/Cancelled cannot return to execution; arbitrary JSON/status writes cannot bypass actor/evidence checks. |
| B10 | Dependency versus input wait | Unsatisfied dependencies stay Pending, capacity waits stay Ready; neither becomes Blocked. Failed prerequisite cannot silently re-arm itself. |
| B11 | Parent aggregation and completion | Blocked child does not block an independent runnable sibling; no-progress input wait surfaces at session. Session manual success requires validated required child/session outputs and no active work. |
| B12 | Query and naming | Show includes reasons/requests/artifacts/stopping and succeeded/total progress; project status includes Failed recovery; duplicate/renamed slugs do not collide or alter IDs/resources. |

## C. Global scheduling and worker recovery

Use a fake harness with real wrapper receipts; artifact-validation fakes may isolate
worker tests. Use real Git/PR evidence for section D and end-to-end skill tests.

| # | Scenario | Pass criteria |
|---|---|---|
| C1 | Concurrent execute across sessions | Exactly one lifetime-lock owner and one global loop; sessions register separately; no per-session scheduler. |
| C2 | Slow side effect | Slow preparation/git call does not prevent another session's cancellation or scheduler reconciliation. |
| C3 | Global capacity | Reservations plus executing work obey max_parallel across sessions; blocked resumes require fresh admission; 0 allows all eligible work. |
| C4 | Launch crash boundaries | Crash after intent, worktree, window, process launch, or acknowledgement: restart reconciles same action/attempt, never creates duplicate workers. |
| C5 | Scheduler death | Existing workers survive where possible; replacement owner adopts identified attempts from DAG/receipts/resources without losing controls. |
| C6 | Shell outlives harness | Exit receipt determines outcome despite live window; missing evidence retains last confirmed state during bounded reconciliation, with specific action/error/deadline visible. Expired deadline becomes Failed with concrete timeout reason, never inferred success. |
| C7 | Fast finish / startup failure | Fast finish records acknowledged Running then validated outcome; definitive preparation/start failure becomes Failed without invented execution. |
| C8 | Missing worker evidence | Reconcile executing/parked/completed worker using recorded action/attempt. Acknowledgement or observation timeout → Failed, fenced stop requested, ownership/capacity retained until exit or non-launch confirmed; late receipts do not silently recover Failed. User-input wait alone never times out. No extra lifecycle state or blind replacement. |
| C9 | Suspension and explicit re-arm | Session stops do not kill global loop; Pending/Ready admission removed, interrupted worker becomes Failed/Suspended after exit; explicit re-arm preserves individual stops and success history. |
| C10 | First failure | Verify the reviewed session-failure policy, retaining failed attempt/history and never manufacturing Blocked input requests for dependency waits. Other sessions progress. |
| C11 | Harness/environment/preparation | Task → execute → session → config → detection precedence; safe argv/prompt quoting; explicit environment despite stale tmux env; idempotent preparation logs/exit. |
| C12 | Missing prerequisites | Non-git dir, missing harness, or unavailable GitHub auth fail preflight before dispatch; diagnostics name missing prerequisite. |

## D. Workspaces, stacked results, and cleanup

| # | Scenario | Pass criteria |
|---|---|---|
| D1 | Session and task isolation | Separate session worktree/branch pins main SHA and holds plans/session files; every task has another worktree; main checkout unchanged. |
| D2 | A → B stacked PR | A pins main, B pins A output SHA; B initially targets A; parent records merge order and retargets as ancestors integrate; B receives actual upstream code. |
| D3 | Fan-in A+B → C | Reviewed integration policy supplies both prerequisite outputs at a recorded commit; C never starts against an arbitrary dependency head. |
| D4 | Integration conflicts / authorization | Conflicts requiring input surface Blocked; expected head/base checked; merge order does not substitute for permission; final main merge user-controlled. |
| D5 | Ambiguous PR operation | Crash after remote create/merge but before local acknowledgement reconciles existing result rather than blindly duplicating mutation. |
| D6 | Completion | Every successful task has DoD/commit/branch/PR evidence and stopped worker; session has required integrated outputs and final PR to main. |
| D7 | Success without approval | Worktrees retained after Succeeded; completion never implies cleanup authorization. |
| D8 | Approved cleanup guards | Task approval or covering session approval bound to revision; changed output invalidates approval; active workers, consumers, unpushed or dirty/untracked work prevent removal. |
| D9 | Cleanup retry | Partial cleanup safely reconciled; branches/history retained; Succeeded not rewritten because removal failed. |

## E. Storage and audit durability

| # | Scenario | Pass criteria |
|---|---|---|
| E1 | Multi-process writers | Many concurrent CLI/scheduler writers produce no corrupt JSON or lost updates; revision conflicts use bounded backoff/jitter and explicit exhaustion. |
| E2 | Snapshot crash points | Stable lock, temp fsync, rename, directory fsync recover a valid committed snapshot at documented durability boundaries. |
| E3 | No side effects in retries | Forced repeated conflicts never relaunch a worker or duplicate a Git/PR effect. |
| E4 | History outbox crash points | Crash before append/after append/before acknowledgement: committed events eventually delivered, deduplicated by ID, ordered consistently per project. |
| E5 | Torn JSONL tail / schema mismatch | Tail repaired under lock without discarding valid records; incompatible writers rejected; audit never overrides authoritative DAG state. |

## F. Skill-driven flow and sign-off

| # | Scenario | Pass criteria |
|---|---|---|
| F1 | Plan and execute | Agent interviews DoD/dependencies, uses CLI only, confirms plan, executes, reports progress. Run Claude Code/opencode; pi best-effort in v1. |
| F2 | User input / manual completion | Agent surfaces Blocked request and can submit response or user's explicit evidenced completion; does not force every blocked task to rerun. |
| F3 | Failure / cancellation / approval | No silent failure retries; cancellation reports pending physical stops; approval is requested before eligible cleanup. |
| F4 | Newcomer flow | README alone supports install → skill-driven two-task session → evidenced completion; target under 15 minutes with prerequisites ready. |

Sign-off requires applicable A–F checks on Linux/macOS, `make test && make lint`
passing with CI enforcement, and current README/AGENTS/design contracts matching
shipped behavior. Record actual results during implementation; policy-dependent
checks remain pending until their architecture decisions are settled.
