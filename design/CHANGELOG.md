# Architecture Changelog

Append-only record of architecture inputs and decisions. Preserve prompts verbatim.

## 2026-09-18 — Revised orchestration plan

Artifact: [PLAN.md](PLAN.md). Status: proposed plan; no implementation performed.

### Original review request

```text
review the architecture for the new version of itx in this branch. all docs are in design/ folder. give me feedback on any gaps inthe architecture and design. goals are already defined. i need this to be used by all engineers who're tryign to predictably scale up their agent workflos both on their machiens and on remote boxes.
```

### User architecture feedback (verbatim)

```text
1. good point. there is only way to do this. the session will start a worktree and all tasks (parallel or sequentials) will build stacked prs agains that worktree branch. in thie example, session will pin to a version of main. task a will create a worktree offf of main, and send pr when done. task b will do the same against task a pr. merges will happen in order and the merge order will be stored in the parent session. parente session is the orchestrator 2. ok. makes sense. persist launch intent the way you mentioned so that states are preserved correctly and thre are no gaps in execution on failure 3. i dont get why this is contradictory. ther can be one scheduler loop for everything. its like the event loop which will scan the full tree and spawn actions for registration. so instead of spawiniing loops for sessions, sessions are registring themselves on the main event loop. tell me why this wont work with examples. 4. this is intentional. the only way to get dependenct to be unblocked is to fix the failed job. or explicity mark the stauts as done. running to blcoked must be there(fix it). stopped tasks are not scheduled by default, the user will put htem in waiting state. there needs to be a state after paused which will mark the task as ready so that it get picked up again in the next scheudler loop. not sure that in indendent runngin gtask means here? when session isupdated, all children get updated. if session is cancelled. the scheudler cancells all childern. work on this part. state transition is the most improtant part of this whole project. 5. correct on this. need a worker wrappre to capture the states of the spawned session. 6. the definsion of done must be a commit, branch and a pr. this is the durable record of complteion. the workflow is that cleanup only occurs when a user has approved the task output or sesion output/outcoe. 7. ignore remote boxes for now. lets keep this localized. 8. yes, its expected that sessions will have unique goals but you're right. lets include immutable session and task identifiers and use slug as disaplay only. 9. yes, do this. workspace prep command shoud be there (determinsitc). dont overindex on this. i want to see this run before improving this. ignore port and db and other conflicts. i undersand the risk but this is a future problem once the core orchestration is working fine 10. makes sense. this will need retries, backoffs, history(jsonl files in projects), schema versioning. json is good but will ned durable mechanisms. also tests to make sure this works wiht multiple processes. give me updated plan based on my feedback and then we'll geto gaps and acceptance criteria. think about this (and preserve my prompts in the changelog for architecture)
```

### Follow-up (verbatim)

```text
what re yo doing this has been goin on forever
```

### Confirmed direction from user

- Parent session owns a pinned-main worktree, stacked task PRs, and persisted merge order.
- One global scheduler accepts session registration; no per-session scheduler loops.
- Persist launch intent and supervise harnesses through a worker wrapper.
- Failure requires explicit repair/completion; paused work must be explicitly re-armed.
- Include running-to-blocked and parent-to-child propagation, especially cancellation.
- Completion requires commit, branch, PR; cleanup requires user approval.
- Immutable identifiers; slugs are display-only.
- Minimal deterministic workspace preparation. Defer runtime-resource conflicts and remote execution.
- JSON storage with durable writes, retries/backoff, per-project JSONL history,
  schema versions, and real multi-process tests.
- State transitions are the highest-priority design work. Review plan before
  revisiting gaps and acceptance criteria.

### Proposed details, not yet user-ratified

- Explicit `ready` and `cancelled` states; waiting-to-ready is scheduler promotion.
- User-only failed-to-waiting repair and failed-to-done resolution retain attempts.
- Session-wide blocking on first task failure, with state-aware child propagation.
- Merge-commit ancestry for v1 stacks; integration barrier for multi-dependency fan-in.
- Global scheduler control terminal, action receipts, and snapshot-to-history outbox.
- Exact merge authorization policy remains to be settled; merge order is not authorization.

### Scope of this edit

Save the revised plan and prompt provenance. Mark the original documents as a
superseded baseline where they conflict. Detailed rewrites and the next acceptance
matrix follow review of this plan rather than delaying the requested feedback.

## 2026-09-18 — Kubernetes-aligned taxonomy and separate session worktree

Artifacts: [STATE_MODEL.md](STATE_MODEL.md), [PLAN.md](PLAN.md).
Status: proposal only; waiting for user review.

### User prompt (verbatim)

```text
what does ready mean? im not happy with these states. do a reserch on what state names are used in kubernetes for pods (use subagent) and propose a taxonomy that alighs with it. wiht each state, also mention 1. what does it mean to be in that state and 2. trigger to move from this to possible states. no. sesson worktree is another worktree OFF off main. this might contain lan and other files so keep it separate. main shoud be clean. definition of done is fine. update this first and wait for me ato review
```

### Research and proposal

- Delegated official Kubernetes documentation research to a subagent as requested.
- Distinguished Pod phases from conditions, container states, and kubectl display status.
- Withdrew the previous Ready phase; proposed Pending/Running/Succeeded/Failed/Unknown,
  separate explicit user controls, and explanatory conditions/reasons.
- Documented meanings and outgoing triggers for every task/session phase and control.
- ITX-specific policies (including intentional interruption classification and explicit
  task repair) are identified separately from Kubernetes behavior.
- Clarified session worktree: a separate checkout/new branch off pinned main, allowed
  to contain plans and session files; never the main checkout or a task worktree.
- Kept commit/branch/PR completion and approval-gated cleanup unchanged.
- Stopped at the proposal; implementation and acceptance work await user review.

## 2026-09-18 — Explicit Ready, Blocked, and Cancelled states

Artifacts: [STATE_MODEL.md](STATE_MODEL.md), [PLAN.md](PLAN.md).

### User prompt (verbatim)

```text
there should be cancelled as well when the user canclesl the sessions. and i need blocked as well. blcoked is when the agent is waitnf for user inputs and cnnot move forward. from blocked it'll go to ready and from ready to running. flow should aso go from pending to ready and from ready to running. ready is when scheduler can pick up the tsk . update this
```

### Updated contract

- Added Ready, Blocked, and Cancelled as explicit ITX lifecycle states, extending
  Kubernetes terminology rather than claiming these are Pod phases.
- Pending → Ready → Running is the normal scheduling path; Ready means eligible
  for scheduler pickup, including waiting for available scheduler capacity.
- Running → Blocked means required user input prevents progress. Resolving the
  request routes through Ready before scheduler-authorized Running.
- Ordinary dependency waits remain Pending; they are not user-input blocks.
- Session cancellation sets Cancelled on the parent and every unfinished child,
  with worker shutdown observed separately. Preserve successful outputs/history.
- Updated task/session meanings, outgoing triggers, and examples. The broader
  proposal remains under review; no implementation changes were made.

## 2026-09-18 — Show Running → Failed explicitly

### User prompt (verbatim)

```text
no there should be failed as well afte rrunning.
```

Updated the STATE_MODEL.md flow diagram and PLAN.md summary to explicitly show
Running → Failed on execution failure or failed completion validation, alongside
Running → Succeeded and Running → Blocked. The transition table already includes
this path; the diagram now matches it.

## 2026-09-18 — Failed → Ready after user repair

### User prompt (verbatim)

```text
but failed can go back to ready once user fixes the failure reasons
```

Updated task/session transition tables, flow diagram, example, and PLAN.md summary:
Failed → Ready once the user resolves failure reasons and explicitly re-arms work,
prior execution is stopped, and scheduling prerequisites are satisfied. Ready →
Running occurs on scheduler dispatch/acknowledgement with a new attempt; retain the
failed attempt history. This replaces the previous Failed → Pending recovery path.

## 2026-09-18 — Publish design branch for Mermaid review

### User prompt (verbatim)

```text
ok. now push to branch. i will review the mermmaid diagram
```

Replaced the text flow with Mermaid diagrams for execution/user recovery,
cancellation propagation, and uncertain-worker reconciliation in STATE_MODEL.md.
Publish the accumulated design documentation to the existing design-itx-v2 branch
for user review.

## 2026-09-18 — Consolidate existing design files and state transitions

### User prompt (verbatim)

```text
cancleled needs to be part of state trnasition. and state can go from blocked to successful as wel. user can say this is done. dont call this kubernes-aligned state dude. its just state transision. alos why're you cretaing new files whne old ones already existi for arch etc. yu've just created dupllated. fucking updat the same files. i can go back when i want
```

### Changes

- Consolidated the architecture and one combined state-transition diagram into
  [arch.md](arch.md#state-transitions). Cancelled is part of that diagram, reachable
  from every unfinished state through task/session cancellation.
- Added Blocked → Succeeded for explicit user completion with the existing
  DoD/commit/branch/PR evidence and confirmed worker shutdown. Kept Failed → Ready
  after user repair and Running → Failed visible in the same diagram.
- Named the contract simply **State transitions**, with no external taxonomy framing.
- Updated [DOMAIN.md](DOMAIN.md), [req.md](req.md), [execution.md](execution.md), and
  [uat.md](uat.md) in place to match the current design rather than leaving baseline
  content behind superseding-document banners.
- Removed duplicate PLAN.md and STATE_MODEL.md. Historical entries above retain
  their original artifact references and prompts; those removed documents can be
  inspected at commit `d8fa15e`. The current architecture is arch.md.
- Unresolved failure/merge policies remain explicitly marked for review. This is a
  documentation revision, not implementation or acceptance sign-off.

## 2026-09-18 — Remove the catch-all state

### User prompt (verbatim)

```text
there cannot be any unknown state. this is a recipe for disaster and becomes a kitchnn sink for anything that doenst have a home. remove it
```

### Changes

- Removed Unknown from the current state definitions, Mermaid diagram, requirements,
  execution plan, and verification scenarios. Historical entries above remain unchanged.
- The seven states are Pending, Ready, Running, Blocked, Succeeded, Failed, Cancelled.
- Missing evidence retains the last confirmed state during bounded reconciliation
  of a specific action. Record last evidence, error, and deadline separately.
- Deadline expiry uses Failed with a concrete acknowledgement/observation timeout
  reason, fencing execution and requesting stop. Preserve ownership/capacity until
  exit or non-launch is confirmed; neither timeout nor late receipts authorize retry.
  User-input waiting itself does not expire. Deadline defaults remain to be defined
  before implementation.
