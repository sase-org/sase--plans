---
tier: epic
title: Evaluate project %if predicates in claimed workspaces
goal: "Project-scoped %if predicates run only after admission claims and prepares a
  numbered workspace, so stale source checkouts cannot admit obsolete agents.

  "
phases:
  - id: condition_workspace_runtime
    title: Claimed-workspace condition runtime
    depends_on: []
    description:
      "condition_workspace_runtime: acquire, recover, and release a prepared operational
      lease around each project-scoped condition check."
    size: medium
  - id: sase_regressions
    title: SASE admission regressions and documentation
    depends_on:
      - condition_workspace_runtime
    description:
      "sase_regressions: exercise ordering, synchronization, cleanup, and failure
      behavior and document the revised typed-admission contract."
    size: small
  - id: toobig_contract
    title: toobig split integration contract
    depends_on:
      - condition_workspace_runtime
    description:
      "toobig_contract: update the external chop's stale-check regression and
      documentation for temporary claimed-workspace evaluation."
    size: small
  - id: integrated_verification
    title: Cross-repository end-to-end verification
    depends_on:
      - sase_regressions
      - toobig_contract
    description:
      "integrated_verification: run the required SASE and chop verification lanes and
      confirm stale queued work skips before agent dispatch."
    size: xsmall
proposed_by: bbugyi200.athena.0dd
bead_id: sase-tk
create_time: 2026-09-09 19:50:06
status: wip
---

- **PROMPT:**
  [prompts/202608/claimed_workspace_if.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/claimed_workspace_if.md)
- **BEAD:**
  [sase-tk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tk/README.md)

# Plan: Evaluate `%if` only in a claimed workspace

## Problem and evidence

Typed admission currently resolves logical waits and then evaluates `%if` from the
launch request's `source_cwd`. Only an eligible Agent unit reaches the ordinary launch
path, where runner admission eventually claims and prepares a numbered workspace. The
ordering is therefore:

```text
waits -> %if in source_cwd -> dispatch -> runner wait -> claim/prepare workspace
```

That source checkout is not necessarily synchronized when earlier agents commit and push
work. The overnight `toobig` runs demonstrate the consequence:

- `toobig-48.agent_chat_from_name.0` launched and immediately reported that the target
  had already been split by an earlier agent.
- `toobig-48.project_mutations.0` did the same for a second previously split target.
- The chop's real predicate checks the target's line count with a relative path. It was
  therefore reading the stale chop/source checkout, not a freshly prepared numbered
  workspace that could see the preceding agent's pushed result.

The Rust admission planner already has the correct logical ordering—waits precede the
`check` action—and the Rust condition evaluator already accepts an explicit cwd. The
missing behavior is host-side workspace acquisition and cleanup around that check; no
new Rust journal phase or wire-schema change is required.

## Intended semantics

For a conditioned unit with a selected managed SASE project:

```text
waits
  -> claim a numbered operational workspace
  -> materialize and synchronize its checkout from the configured upstream
  -> journal checking and run %if with that checkout as cwd
  -> release the temporary claim
  -> journal skipped/error, or dispatch the eligible unit normally
```

The condition lease is deliberately short-lived and is not handed to the admitted Agent
or Proc unit. In particular, an Agent carrying `%wait(runners=...)` must not hold a
numbered workspace while waiting for runner capacity; after an eligible predicate, the
existing deferred launch path claims and prepares its execution workspace at the usual
time. This keeps runner-slot semantics intact while ensuring the eligibility decision
observes a real, synchronized project checkout.

The behavioral contract is:

- Workspace allocation happens only after all typed logical/external waits for the unit
  settle and before the predicate process is started.
- The predicate's cwd is the root of the claimed, prepared checkout. The launch
  request's `source_cwd` is never used as a fallback for a selected project.
- A busy workspace pool is backpressure, not a condition failure: synchronous admission
  returns incomplete when appropriate and the detached coordinator retries without
  starting the predicate.
- Allocation, materialization, or preparation failures fail closed. The predicate must
  not run against `source_cwd`; non-contention failures settle visibly as condition
  errors with the lease operation in the message.
- Eligible, skipped, errored, cancelled, interrupted, and recovered checks all release
  the condition lease exactly once. Persist enough lease settlement metadata beside the
  condition work directory to clean up after coordinator recovery; retain the existing
  orphan-claim cleanup as the final crash backstop.
- False conditions still allocate no runner, agent identity, proc identity, or model
  request, but they now temporarily allocate a workspace. Eligible units continue
  through the existing dispatchers unchanged after that temporary claim is released.
- Home/unmanaged conditions have no claimable numbered project workspace and retain
  their existing explicit source-cwd behavior. Document this boundary rather than
  inventing or claiming workspace `0`.

## Phase: `condition_workspace_runtime`

Implement the condition-scoped lease at the Python admission/runtime boundary in the
`sase` repository.

1. Add a focused condition-workspace helper rather than embedding project and lease
   mechanics in the admission loop. It should derive a stable workflow/holder identity
   from the request and logical unit, acquire through the existing operational workspace
   lease API, expose the prepared checkout cwd, persist an idempotent settlement marker,
   and release/recover that marker safely.
2. Pass the request identity and selected project into the admission engine's condition
   context. For project-scoped units, acquire the lease before writing the `checking`
   journal entry, then invoke the existing sandboxed evaluator with the lease checkout
   as `source_cwd`. Keep the Rust evaluator's private script, environment sanitization,
   timeout, cancellation, result, and replay behavior unchanged.
3. Teach the engine to distinguish workspace-pool contention from a completed action. An
   inline `until_blocked` run should return an incomplete receipt so the normal detached
   coordinator can continue; a detached run should sleep/retry without a busy loop.
   Replan after each resourceful check so an eligible unit dispatches, or a terminal
   unit releases, before admission speculatively leases additional workspaces.
4. Settle the marker in `finally` paths and recovery paths, including cancellation,
   interrupted `checking`, evaluator exceptions, and a persisted result discovered by a
   restarted coordinator. Never fall back to the primary/source checkout after a lease
   error.
5. Keep the condition lease separate from Agent and `%proc` execution leases. Do not
   change the Rust admission action vocabulary or transfer a condition claim into a
   deferred runner.

Likely implementation surfaces include `src/sase/agent/launch_admission_engine.py`,
`src/sase/agent/launch_condition_runtime.py`, a focused sibling module for condition
workspace state, and a small public extension to `src/sase/workspace_provider/lease.py`
if typed contention/recovery helpers are needed.

## Phase: `sase_regressions`

Add focused and integration coverage in the `sase` repository, then update the public
contract.

Tests should prove:

- a predicate cannot start before its logical wait resolves;
- while the predicate runs, its cwd is a numbered checkout with a live operational
  claim, not the request's stale source cwd;
- a prepared lease observes an upstream commit that the source checkout has not pulled,
  causing a realistic `toobig`-style line-count predicate to skip without invoking the
  Agent dispatcher;
- workspace exhaustion leaves admission pending and later recovery evaluates exactly
  once after a workspace becomes available;
- eligible, skipped, condition-error, cancellation, evaluator exception, and replayed
  result paths release the lease and do not leak RUNNING claims;
- a lease preparation failure records a condition error and never executes in
  `source_cwd`;
- home/unmanaged conditions retain their explicitly documented behavior; and
- `%wait(runners=...)` remains in the eligible Agent prompt, proving the temporary
  condition lease did not collapse deferred runner admission.

Prefer real temporary git repositories and project specifications for the stale-source
regression, with small injected fakes for contention and crash/recovery edges. Extend
the existing launch-admission and condition-runtime suites instead of creating a second
admission harness.

Update `docs/architecture.md`, `docs/xprompt.md`, and the typed AXE discussion in
`docs/axe.md`: replace the old “resource-free/no workspace” wording, explain the
temporary prepared condition lease and home boundary, and preserve the guarantees that
false predicates allocate no runner/identity/model request and that waits precede the
check.

## Phase: `toobig_contract`

Use `/sase_repo` to open `gh:bbugyi200/bugyi-chops`; do not locate or fetch that
repository another way. The production predicate does not need to change. Update its
integration contract and docs so they match the fixed SASE behavior:

1. Replace the current same-checkout stale mutation fixture with the incident-shaped
   topology: the chop/source checkout still contains an oversized file while a separate
   writer has pushed the already-split version to the configured upstream. Configure a
   real claimable SASE project/workspace pool for the bridge test.
2. Assert that the leased predicate checkout synchronizes the newer upstream content,
   the typed admission result is `skipped`, and the supplied agent launcher is never
   called. Retain the eligible and clan-promotion assertions so the new setup does not
   weaken dispatch coverage.
3. Update the README language from “no workspace allocation” to “a temporary claimed
   workspace, but no runner/agent/model allocation,” and explain that the lease is why a
   queued condition sees earlier pushed splits even when the chop checkout is stale.

Keep this phase independent of the SASE regression-writing phase so both can proceed
after the runtime contract is available.

## Phase: `integrated_verification`

Verify the assembled behavior without broadening the feature:

1. In `sase`, run `just install` and the required `just check`. Because this is the
   combined epic tree and the launch-admission/lease behavior has broad lifecycle
   consequences, run `just check-full` through `/sase_monitor` as required by project
   instructions, with the prescribed `TESTING`/`TESTED` statuses and a concrete
   follow-up action.
2. Run the focused launch-admission, condition-runtime, workspace-lease, and AXE typed
   chop tests explicitly so their output is easy to audit even if the scoped selector
   would omit one.
3. In `bugyi-chops`, follow that repository's own install/lint/test instructions and run
   the focused `toobig_split` integration tests against the updated SASE checkout.
4. Inspect RUNNING claims after the skip, eligible, error, and cancellation exercises;
   no `lease(launch-if:...)` claim may remain. Confirm the stale-upstream scenario
   produces zero agent launches and a completed skipped admission receipt.

## Non-goals and risks

- Do not move `%if` into the agent runner: that would allocate the very agent/runner the
  predicate is intended to suppress and would still create repeated no-op agent rows.
- Do not retain or transfer the condition lease through dispatch: runner-capacity waits
  must remain workspace-free, and execution workspace ownership stays with the existing
  launch/proc paths.
- Do not evaluate against workspace `0`, the primary checkout, or `source_cwd` when a
  selected project's lease cannot be acquired.
- Do not change `%if` exit-code classification, sandboxing, timeout, context schema,
  typed wait semantics, clan promotion, or dispatch fingerprints.
- There is an unavoidable time-of-check/time-of-dispatch interval because the temporary
  condition lease is released before normal dispatch. The goal is a current,
  claim-backed eligibility observation without reserving a workspace across runner
  waits, not an atomic repository snapshot handoff.

## Acceptance criteria

- A project-scoped `%if` process starts only while a numbered, prepared workspace has a
  live operational claim and uses that checkout as cwd.
- A stale primary/chop checkout plus a newer upstream split causes the real line-count
  pattern to skip before any agent launcher runs.
- Workspace contention delays admission; preparation failures fail closed; neither path
  executes the predicate in the stale source checkout.
- Every terminal and recovery path releases the temporary claim, and no runner-capacity
  wait holds it beyond condition evaluation.
- Existing typed wait ordering, condition replay, condition classification, Agent/Proc
  dispatch, and AXE clan-promotion tests remain green.
- SASE and `bugyi-chops` documentation and integration tests describe and enforce the
  same claimed-workspace semantics.
