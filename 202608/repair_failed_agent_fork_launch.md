---
tier: tale
title: Repair failed-agent fork launch admission
goal:
  Failed-agent forks reach a real workspace and execute without losing explicit wait
  semantics or conflicting with the active shell-aware fork epic.
size: small
proposed_by: bbugyi200.athena.0d3
create_time: 2026-08-24 18:50:27
status: wip
---

# Repair failed-agent fork launch admission

## Diagnosis

The failed `sase-sq.7.1.2.f0` and `sase-sq.7.1.2.f0.f0` runs never reached a model. Both
were launched from workspace `#0` with `SASE_AGENT_DEFERRED_WORKSPACE=1` and failed in
`bootstrap_agent_run()` with:

> `SASE_AGENT_DEFERRED_WORKSPACE=1 but extracted wait metadata is empty`

This is a composition bug introduced by `e4534d265`
(`fix(agent): allow forking failed agents`):

1. Launch planning calls `has_deferred_start_directive()` and conservatively treats an
   explicit `#fork:<name>` as a deferred launch, so it provisions the placeholder
   workspace and sets `SASE_AGENT_DEFERRED_WORKSPACE=1`.
2. Runner directive extraction sees that the named parent is already terminally failed
   and correctly suppresses the normally implied wait through
   `fork_parent_wait_is_unreachable()`; explicit user-authored waits remain intact.
3. Bootstrap then rejects the now-legitimate state
   `deferred workspace + no remaining semantic wait`, even though the post-admission
   launch path already uses the deferred flag to claim and prepare a real workspace
   before model execution.

Trying to make the cheap launch scan consult current agent state is insufficient: the
parent can fail between launch planning and runner extraction. The robust invariant is
that launch detection may be conservative, but a deferred run must always claim a real
workspace before execution. The existing launch phase already enforces that invariant.

## Scope and active-epic coordination

This is deliberately a small compatibility repair, not a competing fork architecture.
The active `sase-t8` epic, especially phase `sase-t8.2`, is replacing ordinary implied
fork waits with durable typed, terminal-aware fork dependencies. Before editing code,
re-read `sase-t8` and its phase statuses and append a `DISCOVERED ISSUE:` note to the
`sase-t8` epic describing the two failed runs, the preflight/extraction mismatch, the
files this tale expects to touch, and the intended regression. Also leave a focused note
on `sase-t8.2` so its worker sees the launch-boundary invariant. If phase 2 has already
landed an equivalent fix, preserve its design and add or strengthen only the missing
regression instead of restoring the old failure shortcut.

After verification, append a second epic note naming the final implementation and test
coverage, or explain that phase 2 superseded the code change and how the regression was
adapted. Do not alter phase ownership, dependencies, or status.

## Implementation

1. In `src/sase/axe/run_agent_runner_bootstrap.py`, replace the unconditional fatal
   assertion for `deferred_workspace and not has_wait` with the safe conservative
   admission behavior: retain `RunnerBootstrap.deferred_workspace=True`, skip dependency
   waiting when `has_wait=False`, proceed through runner-slot admission, and let
   `_prepare_workspace_and_repos()` claim the real workspace before any execution.
   Document why a fork parent becoming terminal between preflight and extraction makes
   this a valid state. Do not clear the deferred flag and do not allow model execution
   in workspace `#0`.
2. Keep the failed-parent semantic behavior from `e4534d265` intact:
   `fork_parent_wait_is_unreachable()` still suppresses only an implicit fork wait;
   explicit `%wait:<failed-parent>` remains success-gated, and mixed/live/group/tribe
   fork sources continue to wait according to current behavior. Do not add mutable
   agent-state lookup to `has_deferred_start_directive()` merely to hide the mismatch.
3. Update `tests/test_axe_run_agent_runner_deferred_workspace_outcomes.py` so the old
   “deferred without extracted wait fails” assertion becomes a regression proving the
   conservative path claims a nonzero real workspace, updates metadata, and reaches the
   run loop without calling dependency-wait machinery. Keep or add an assertion that
   execution never observes the placeholder path/number.
4. Add the smallest composition-level failed-fork regression needed to connect the
   existing pieces: a terminal failed parent makes launch preflight classify the
   explicit fork as deferred, directive extraction suppresses its implicit wait, and
   runner admission still claims a real workspace and proceeds. Reuse existing artifact
   fixtures and runner helpers rather than launching a real provider. Cover the same
   no-transcript early-failure shape represented by `sase-sq.7.1.2.f0`, and ensure a
   user-authored explicit wait is not accidentally bypassed.
5. Reconcile against any concurrently landed `sase-t8.1`/`sase-t8.2` changes before
   finalizing. Prefer the new typed fork-dependency model when present; the invariant
   and regression remain: conservative placeholder provisioning must never fail merely
   because semantic fork dependencies are already terminal, and model execution must
   never occur in the placeholder workspace.

## Verification

Run `just install` first, then the focused directive/extraction, fork history, and
deferred-runner suites covering the changed path. Run `just check` for the required
whole-repository gates. If the concurrent `sase-t8` merge broadens the change into the
Rust/Python wait wire or `just check` escalates, run `just check-full` only through
`/sase_monitor` with `TESTING`/`TESTED` statuses and a concrete follow-up action.

Finally, exercise the built-in fork resolver/renderer against an archived terminal
failed agent with no transcript and confirm it produces `PARENT AGENT FAILED` context;
the automated runner regression is the authoritative proof that this context can now
reach a child without the bootstrap crash.

## Completion criteria

- The failure mode from both named runs is reproduced by a regression that fails on
  `e4534d265` and passes after the repair.
- A failed-parent `#fork` with no explicit wait reaches model execution only after a
  real workspace is claimed; it neither parks forever nor dies in bootstrap.
- Explicit waits and live/group/tribe fork waits preserve their current semantics.
- `sase-t8` and `sase-t8.2` carry clear before/after coordination notes, with any
  concurrent overlap reconciled rather than overwritten.
- Focused tests and `just check` pass.
