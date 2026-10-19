---
tier: tale
title: Keep session waiters parked while an approved gate launches its coder
goal:
  A %w:<session> waiter stays parked through a plan gate's approval-to-coder launch
  window and only releases when the session's coder and later turns are done.
size: medium
proposed_by: bbugyi200.athena.11u
proposed_with: claude/opus
create_time: 2026-10-19 10:33:59
status: wip
---

# Plan: Stop `%w:<session>` waiters releasing while an approved gate is still launching its coder

## Problem

Research report `research:202610/11q_w0_early_wait_release.md` traced why `11q.w0`
(armed with `%w:11q`) started while `11q` was still running. After `11q--gate` (a plan
gate) was approved, there was an ~18 s window before the coder it approved (`11q--code`)
existed. During that window the gate's `agent_meta.json` said
`gate_followup_outcome: "failed"`. The wait resolver reads a non-success outcome as "no
successor is coming", so the `11q` session aggregate looked resolved and done, and the
`wait_checks` tick released the waiter. The gate's own `done.json` write wakes
`wait_checks`, so the tick almost always lands in that window. In practice, **any
`%w:<session>` on a session whose plan gate gets approved is released at approval
time**, not after the coder, monitor, and retry turns finish.

### Root cause (verified in code)

1. **Settlement persists a terminal-looking outcome before it launches anything.**
   `launch_or_record_followup` in `src/sase/gate_turn/handoff_launch.py` calls
   `classify_gate_handoff`, then `apply_decision` (`src/sase/gate_turn/handoff.py`), and
   only then calls `persist_attempt(preparing/launching)` and
   `settle_turn_claim_and_followup`. On the normal first-pass settle input (gate
   `answered`, follow-up requested, nothing recorded yet), the Rust classifier
   (`decide_missing_disposition` → `failure_or_resume` in sase-core
   `crates/sase_core/src/gate_followup/classify.rs`) returns `persist_outcome: "failed"`
   together with `launch_allowed: true`. `apply_decision` writes that `"failed"`
   immediately. The real outcome (`launched` plus `gate_followup_agent`) is written only
   after the launch returns. Every launch exit already records its own final outcome:
   - success: `launched` or `launched-degraded`, through `record_turn_followup_outcome`
     in `src/sase/turns/settlement.py`
   - launcher declined: `not-launchable`
   - `settle_error`: `failed`, through `persist_attempt(..., error=...)`
   - exception: `failed`, through `persist_attempt(..., error=...)`

   So the only effect of the pre-launch write is the misleading interim value.

2. **The resolver treats any non-success outcome as "no successor".**
   `turn_followup_handoff_agent` in
   `src/sase/core/wait_dependency_resolution/_artifact_state.py` returns the
   `<pending-turn-followup>` placeholder only when `followup_outcome is None`. That
   placeholder is what keeps a session open, through
   `_agent_session_turn_handoffs_have_successors` in `_index_entities.py`. For any other
   non-success outcome it returns `None`, so `handoffs_present` becomes true and the
   session aggregate resolves.
3. Releases are one-shot. The confirmation re-scan only looks for new member
   directories, and the successor's directory did not exist yet. So nothing took the
   release back.

The same window also exists on the explicit resume path (`sase gate answer ... --resume`
→ `settle_already_terminal_handoff`). There, the stale `"failed"` from the earlier
attempt stays in both `agent_meta.json` and `done.json` while the new attempt launches.

## Approach

All changes are in this repo (sase). There are two independent layers of defense plus
regression tests. **No sase-core change is made.** The classifier's `persist_outcome` is
still correct as the fallback disposition, "what to record if the launch path records
nothing". The bug is that the host writes it _before_ the launch, which is host
sequencing glue in `handoff_launch.py`. Fixing it in Python takes effect with every
`sase-core-rs` in sase's accepted minor window, needs no `sase-core-revision.txt` pin
move, and leaves every final on-disk state exactly as it is today.

### 1. Defer the classifier's provisional outcome until the launch attempt ends

In `src/sase/gate_turn/handoff.py` / `src/sase/gate_turn/handoff_launch.py`:

- Let `apply_decision` defer the outcome write. For example, add a keyword such as
  `defer_outcome: bool = False`. When it is set, the function skips persisting
  `persist_outcome` and returns that value to the caller. The other fields
  (`adopt_agent`, `needs_attention`) are still applied as today. For launch-allowed
  decisions those are `None`/`False` anyway, but keep the behavior general.
- In `launch_or_record_followup`, compute
  `launch_allowed = bool(decision.get("launch_allowed"))` once and call
  `apply_decision(..., defer_outcome=launch_allowed)`. The non-launch paths (`adopt`,
  `wait`, diagnose, and the `not launch_allowed` early return) must behave exactly as
  they do now.
- Every exit of the launch branch must apply the deferred outcome, but only if no
  outcome has been recorded by then. The exits are: the `except` branch, the
  `settle_error` branch, and the success path after `merge_followup_fields`.
  Re-read/merge the on-disk meta first. The launcher and `persist_attempt` write fields
  to disk, and `merge_followup_fields` already pulls `gate_followup_*` keys back. Write
  the deferred value only when `meta.get("gate_followup_outcome")` is still empty.
  Writing it unconditionally would overwrite `launched`. A small private helper
  (`_record_deferred_outcome(artifacts_dir, meta, deferred)`) or a `try/finally` around
  the launch branch are both fine. Choose whichever reads cleanest; it must also cover
  the early `return` in the `except` branch.
- Do not touch the resume path's stale outcome (no clearing). Layer 2 covers that
  window.
- Update the `launch_or_record_followup` docstring or add a short comment explaining why
  the outcome is deferred: an interim terminal outcome releases `%w:<session>` waiters
  early.

Crash semantics after this change: if the settling process dies hard mid-launch, the
gate is left terminal with `gate_next_action` set and no outcome. The resolver keeps the
session pending (existing placeholder) until the periodic reconcile pass
(`reconcile_incomplete_gate_handoffs` → `_diagnose_one` in
`src/sase/gate_turn/reclaim.py`) classifies the stale attempt (diagnose mode:
`decide_missing_disposition`). That pass persists `"failed"` with `needs_attention`, and
then the session resolves the same way a recorded failure does today. This is the
intended behavior; do not add other fallbacks.

### 2. Resolver guard: a live handoff attempt keeps the session open

- Add one small shared predicate so the resolver and the classifier snapshot cannot
  drift. Put it in a new lightweight module, for example
  `src/sase/gate_turn/attempt_liveness.py`. It must not import `sase.gate_turn.handoff`,
  the agent-scan facade, or `sase.ace` at module import time, because the wait resolver
  imports it on every `wait_checks` tick. Suggested API:
  `gate_followup_attempt_is_live(meta: Mapping[str, Any]) -> bool`. It returns `True`
  when the flat on-disk `gate_followup_attempt_live` is `True` and the recorded
  `gate_followup_attempt_owner_pid` is either absent or still running. That is exactly
  the current rule in `handoff._attempt_snapshot`:
  `recorded_live and (owner_pid is None or is_process_running(owner_pid))`. Lazily
  import `is_process_running` from `sase.ace.hooks.processes` inside the function, the
  same way `_member_runner_is_live` lazily imports its liveness check. Move the attempt
  field-name constants this needs (`ATTEMPT_LIVE_FIELD`, `ATTEMPT_OWNER_PID_FIELD`) into
  the new module, have `handoff.py` import them from there, and keep `handoff.py`'s
  public names and `__all__` working. Move or duplicate the small `_optional_int`
  parsing as needed, without importing privates across modules (Symvision).
- Make `handoff._attempt_snapshot` compute `live` through the new predicate.
- In `turn_followup_handoff_agent`
  (`src/sase/core/wait_dependency_resolution/_artifact_state.py`), extend the existing
  gate guard. For a `kind == "gate"` member whose state is not `lost`/`stopped` and
  whose `next_action` is non-blank, return `_PENDING_TURN_FOLLOWUP_AGENT` when:
  - `followup_outcome is None` (existing behavior), **or**
  - `followup_outcome` is not in `SUCCESSFUL_TURN_FOLLOWUP_OUTCOMES` **and**
    `gate_followup_attempt_is_live(meta)` is true (new).

  Successful outcomes keep returning the recorded follow-up agent, unchanged. Read
  attempt liveness from `meta` (the on-disk `agent_meta.json`), never from `done_data`.
  `persist_attempt` writes only meta, and `done.json` does not carry the live flag or
  the owner pid. Rows built from `asdict(AgentMetaWire)` snapshots
  (`sase.agents._wait_live_rows`) have no flat live key, so they keep today's behavior.
  Document that fail-open in the docstring. The authoritative release path
  (`wait_checks` → `WaitDependencyIndex.add_many`) reads on-disk meta and gets the
  guard. The pid check only runs for gate members that have a non-success outcome _and_
  a recorded-live attempt, so it adds no meaningful cost to the index build.

- Leave `turn_followup_outcome()` (raw outcome for terminal-blocker detection)
  unchanged.

### 3. Regression tests

Add the tests next to the existing coverage, following each file's fixture style.

- `tests/test_gate_wait_dependency_pending_followup.py` (uses `make_agent` and
  `_write_gate_done` from `tests/_gate_wait_dependency_helpers.py`). Build the same
  `gate-lane--plan` → `gate-lane--gate` session as the existing tests, with no successor
  directory, and assert on `index.agent_session_candidate("gate-lane")` and on
  `dependency_resolution_status(index, ["gate-lane"])`. Together these reproduce the
  report's verification table.
  - **Mid-launch with an interim failed outcome stays parked.** Gate meta has
    `gate_followup_outcome: "failed"`, `gate_followup_attempt_live: True`,
    `gate_followup_attempt_stage: "launching"`, and
    `gate_followup_attempt_owner_pid: os.getpid()`. The session is not resolved and the
    `%w` status is not resolved. Parametrize over (a) `"failed"` in meta only, which is
    the settle window with today's write order, and (b) `"failed"` in both meta and
    `done.json`, which is the resume window.
  - **A settled failure still releases.** The same shape with
    `gate_followup_attempt_live: False` and stage `"failed"` resolves the session. Add a
    second case with a recorded-live attempt whose owner is dead: monkeypatch the
    liveness check, for example `sase.ace.hooks.processes.is_process_running`, to return
    `False`. Do not rely on a guessed-dead pid. This proves the guard only holds while
    an attempt is actually live and does not change today's failure semantics.
- `tests/gate_turn/test_settlement_followup.py` (fixtures: `create_gate`,
  `make_gate_turn_member`, `execute_gate_selection`, monkeypatched
  `sase.gate_turn.handoff_launch.launch_gate_followup_agent`).
  - **No terminal outcome during launch.** Use an observing launcher, in the style of
    `test_launcher_only_runs_after_the_turn_is_terminal_and_indexed`, that reads
    `agent_meta.json` and `done.json` from disk while it runs. Assert that neither
    carries a `gate_followup_outcome`, and that the meta records a live attempt
    (`gate_followup_attempt_live is True`, stage `launching`). If it is straightforward
    with the sandbox's projects root, also build the wait index inside the launcher and
    assert that the `lane` session is unresolved. This is the end-to-end form of "an
    approved plan gate mid-launch must keep a `%w:<session>` waiter parked".
  - **The deferred fallback still lands.** Monkeypatch `settle_turn_claim_and_followup`
    in `sase.gate_turn.handoff_launch` to return `None` without recording an outcome.
    Assert that the final meta and `done.json` carry the classifier's provisional
    `"failed"`, the same final state as today.
  - The existing final-state tests must stay green unchanged:
    `test_preparation_exception_persists_failure_without_placeholder_launch`,
    `test_resume_recovers_incomplete_terminal_handoff`,
    `test_tale_approve_commit_settlement_launches_coder_followup`,
    `test_done_marker_carries_the_followup_outcome_and_agent`, the
    `tests/gate_turn/test_answered_handoff_resume.py` suite, and the reclaim/reconcile
    suites.
- If `_attempt_snapshot` behavior is covered anywhere (for example
  `tests/gate_turn/test_successor_evidence.py` or `test_inflight_creation.py`), keep
  those tests passing. Add a focused unit test for `gate_followup_attempt_is_live`
  covering: recorded live with no pid, live with a running pid, live with a dead pid,
  and not recorded live.

## Non-goals

- **No change to the sase-core classifier or its wire.** Its `persist_outcome` stays the
  fallback disposition, and diagnose/reconcile behavior is unchanged.
- **No change to the one-shot release or to the confirmation re-scan.** Removing the
  false signal at its source, plus the live-attempt guard, is enough. Revocable releases
  would be a much larger design change.
- **No change to `gate_turn_followup_needs_attention` / ACE GATE projection** during the
  launch window.
- **Monitor turns are out of scope.** `settle_monitor_followup` in
  `src/sase/monitor/proc_adapter.py` writes the monitor's `done.json` only _after_ the
  follow-up launch, so a monitor member is not terminal during its launch window and has
  no equivalent early-release window.

## Verification

- Targeted while iterating: `tests/test_gate_wait_dependency_pending_followup.py`,
  `tests/gate_turn/`, and the wait-dependency resolution tests (`tests/` files matching
  `*wait_dependency*`).
- Before finishing, follow the repo's lint-and-test memory note (`lint_and_test`). Run
  the standard `just check` through `sase tool run`, and fix any Symvision findings for
  the new module, moved constants, and new test helpers.
- Manual sanity check is optional. Approving a plan gate while a `%w:<session>` waiter
  is parked should leave the waiter parked until the session's coder and its turns
  finish.
