---
tier: epic
title: Just-in-time bead claims for launched agents
goal: 'Phase and epic beads stay open while their responsible agents are waiting,
  then transition atomically to in_progress with the responsible assignee immediately
  before model execution; claim failures prevent execution and report a clear bead-specific
  error.

  '
phases:
- id: core_claim
  title: Atomic bead launch claim
  depends_on: []
  size: medium
  description: '''Atomic bead launch claim'' section: add the Rust-core mutation,
    locking, binding, facade, and tests needed to claim one bead for a named agent.'
- id: directive_runtime
  title: '%id(bead=...) runner lifecycle'
  depends_on:
  - core_claim
  size: medium
  description: '''%id(bead=...) runner lifecycle'' section: parse and preserve the
    new keyword, then claim after waits and workspace preparation but before model
    execution.'
- id: epic_work
  title: Migrate epic bead work to just-in-time claims
  depends_on:
  - directive_runtime
  size: medium
  description: '''Migrate epic bead work to just-in-time claims'' section: render
    phase and land bead associations, remove eager preclaiming, and cover lifecycle
    and failure behavior end to end.'
create_time: 2026-07-20 15:48:27
status: wip
bead_id: sase-8f
---

# Plan: Just-in-time bead claims for launched agents

## Context and target behavior

`sase bead work <epic>` currently computes every non-closed phase assignment, marks every phase `in_progress` with its
agent as assignee in one eager batch, launches the phase/land runner processes, and never marks the epic itself
`in_progress`. This makes downstream phases look active while their runners are still blocked on `%wait` dependencies.

Move responsibility for the transition into a new `bead=<bead_id>` keyword on `%id`. The directive associates exactly
one launched agent with exactly one bead. A runner may parse and publish that association early, but it must perform the
state transition only after dependency waits, runner-slot admission, deferred-workspace allocation, and workspace/SDD
preparation have succeeded. It then claims the bead immediately before starting model execution:

1. Render and spawn the runner with `%id(..., bead=<bead_id>)`.
2. Complete agent-name/bead waits and runner-slot admission without changing the target bead.
3. Prepare the real workspace and resolve its effective bead store.
4. Atomically set the bead to `in_progress` and assign it to the resolved agent name, persisting the mutation.
5. Start the model only if step 4 succeeds.

For epic work, every phase segment names its phase bead and the land segment names the parent epic bead. Therefore phase
beads remain open until their own workers pass their waits, and the epic remains open until its land agent is actually
ready to run.

## Behavioral contracts

- `bead=` is independent of `%id`'s mutually-exclusive `clan=`, `family=`, and `tribe=` membership choices, so it is
  valid on plain, auto-named, clan, family, and tribe forms. It accepts one non-empty bead ID; duplicate or malformed
  keywords retain the directive parser's existing targeted diagnostics.
- A launch claim accepts an open bead and supports retrying/reassigning a non-closed bead already marked `in_progress`.
  It rejects missing or closed beads and an empty resolved agent name. It never silently reopens a bead.
- The claim updates status and assignee in one backend mutation and emits a normal status/assignee update event so old
  event readers remain compatible. Same-store concurrent claims must serialize their read-modify-write sections so
  parallel phase workers cannot overwrite each other's event/projection updates.
- A successful claim is not rolled back merely because model execution later fails or the agent is killed: the worker
  did take responsibility, and the `in_progress` state makes the interrupted work visible. A retry can claim it again
  under its new agent name. A runner that exits through repeat-stop or fails before the claim remains non-mutating.
- The runner fails before model invocation when it cannot resolve, mutate, or durably persist the bead store. The error
  identifies the agent, bead, and underlying cause and is recorded through the normal runner error/done artifacts.
- If the launch environment already supplies `SASE_BEAD_ID`, it must match the `%id` value; a mismatch is a rendering or
  caller bug and fails before execution. A valid directive remains available as agent metadata and commit attribution,
  including for non-epic callers.
- Blocking waits may refresh/re-exec the runner. Claiming happens only on the post-wait execution pass. In-process model
  retries do not create extra claims, while an explicit spawn-on-retry performs an idempotent reassignment.
- Preserve existing bead push policy: local mutation/commit failures are fatal to the launch, while an already-created
  local commit is not undone solely because the configured remote push reports a warning. Use the existing managed SDD
  repository transaction/locking paths rather than ad hoc git commands.
- No `sase/memory/*.md`, generated agent-instruction shim, or other memory file is changed without separate explicit
  user approval. User-facing directive help and completion maintained in normal source files are in scope.

## Atomic bead launch claim

Implement the shared domain operation in the linked Rust core rather than reproducing lifecycle rules in the Python
runner:

- Add a single-bead `claim_for_agent_launch` mutation taking a bead ID, resolved agent name, and optional timestamp. It
  loads the current store under a bounded exclusive mutation lock, validates the target and status, sets
  `status=in_progress` plus `assignee=<agent>`, appends the compatible update event, regenerates projections, and
  returns the updated issue/outcome.
- Keep the lock across the store's read/validate/event/projection write critical section. Return typed, actionable
  errors for not found, closed target, invalid agent name, and lock/persistence failures. An already-`in_progress`
  target remains a successful claim so deterministic epic reruns and named retries can reassign it.
- Export the operation through `sase_core`, the PyO3 binding, and the Python bead mutation facade/`BeadProject` adapter.
  Keep the old batch-preclaim binding compatible if external callers may still use it, but remove it from the epic-work
  launch path in the final phase.
- Add Rust mutation/event tests, binding/facade parity tests, and a concurrent same-store regression proving two claims
  retain both event updates and projected statuses. Verify missing, closed, already-in-progress, and reassignment cases.

## `%id(bead=...)` runner lifecycle

Thread the association through every directive and runner layer without changing when other `%id` behavior occurs:

- Extend name-directive argument parsing and `PromptDirectives`/`AgentInfo` with an optional bead ID. Treat `bead=` as
  an orthogonal keyword while retaining the existing one-membership-axis validation. Validate and expand it consistently
  with other directive values, including auto-named `%id(bead=...)`.
- Update static-name inspection, fan-out naming, retry/relaunch edits, clan demotion, and family-attach rewriting so the
  bead keyword is preserved whenever the logical worker is preserved. Family retries should remove only the obsolete
  `family=` attachment rather than dropping `bead=` as collateral. Add `bead=` to interactive directive help and
  completion candidates.
- Persist the association in `agent_meta.json`, establish/validate `SASE_BEAD_ID`, and preserve it across the existing
  metadata refresh/follow-up paths. Keep epic-specific `epic_bead_id` and `phase_bead_id` role metadata intact rather
  than replacing those richer relationships with the generic field.
- Add a focused runner lifecycle helper that resolves the effective bead store only after deferred-workspace and linked
  SDD preparation, invokes the core claim, refreshes the local Python projection/cache as needed, and commits through
  the managed SDD transaction helpers. Call it after wait/repeat-stop handling and workspace preparation but before
  capturing the post-claim SDD base and invoking `run_execution_loop`.
- Cover parsing/diagnostics and text-preservation helpers, then add runner-order tests proving the bead remains open
  during dependency and runner-slot waits, repeat-stop never claims, code refresh does not double-claim, successful
  claim precedes model execution, and each resolution/mutation/commit/mismatch failure suppresses model execution with a
  useful error.

## Migrate epic bead work to just-in-time claims

Make `sase bead work` a producer of launch associations rather than the owner of every phase status transition:

- Render each phase identity as `%id(..., bead=<phase_id>)`, including the first clan-declaring segment and subsequent
  clan join forms. Render the land identity as `%id(..., bead=<epic_id>)`. Keep the existing `%wait` agent/bead edges,
  force-reuse handshake, role model selection, role metadata environments, and deterministic names unchanged.
- Remove the eager `preclaim_epic_work` call and its all-phase rollback payload from `launch_epic_bead_work`. Continue
  setting `is_ready_to_work` when the epic is launched because that flag records approval/launch eligibility, not active
  execution. The successful launch commit now persists that flag and any other parent-owned launch metadata only.
- Rework launch-failure cleanup around the new ownership boundary. If no runner was spawned, restore a readiness flag
  set by this attempt. If a partial bundle spawned runners, terminate those runners as today but retain
  `is_ready_to_work` and any claims already made, then persist that recoverable state so a rerun can launch the
  remaining non-closed work without pretending nothing started.
- Update the phase-worker prompt contract only where needed so it remains truthful when the model sees it: the runner
  has already claimed the phase by then. The land agent now sees its epic in progress and still closes it only after
  verification/integration.
- Replace eager-preclaim assertions in epic lifecycle, rollback, rendering, dry-run, JSON, retry, and collision tests.
  Add integration coverage for a dependency chain and parallel wave: downstream phases stay open while waiting, each
  phase becomes `in_progress` immediately before its worker executes, completed phases still unblock dependents, and the
  epic becomes `in_progress` only when the land agent crosses all phase waits. Verify closed-target land/phase claims
  fail without invoking their models and that dry runs never mutate bead status.

## Validation and rollout

- In `sase-core`, run targeted bead mutation/binding tests followed by `cargo fmt --all -- --check`,
  `cargo test --workspace`, and `cargo clippy --workspace --all-targets -- -D warnings`.
- In `sase`, run `just install` before targeted directive, runner, core-facade, and epic-work tests, then run the
  required `just check`. Exercise both the generic CWD launcher and the planned bead-work fast path because they share
  the runner lifecycle but prepare segments differently.
- Inspect generated prompts in dry-run tests to ensure every phase/land segment carries exactly one correct `bead=`
  association and no status mutation occurs in the parent. Confirm legacy prompts without `bead=` retain their current
  behavior.
