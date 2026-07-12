---
create_time: 2026-07-10 15:24:22
status: done
prompt: 202607/prompts/wait_deps_never_cancel.md
tier: tale
---
# Waiting Agents Must Never Fail Because a Dependency Failed

## Problem

Killing (or dismissing, or otherwise failing) an agent that other agents are `%wait`ing on currently causes those
waiting agents to FAIL almost immediately. Observed live: `bob-cli-9` (an epic lander waiting on
`%w:bob-cli-9.1,...,bob-cli-9.8`) flipped to FAILED with "Queued launch cancelled because dependency failed" seconds
after `bob-cli-9.5` was killed from the Agents tab.

This is wrong. The desired semantics are:

- A wait dependency can only ever be **satisfied** — it must never be fatal to the waiter.
- Waiting agents keep waiting, **potentially forever**, until every dependency has completed successfully.
- A dependency may not exist yet (waiter launched first), may fail, may be killed, and may be restarted any number of
  times. The waiter resolves once a (possibly restarted) run of each dependency completes.

Killing the _waiting agent itself_ must still stop it — that is the user's escape hatch and already works via the
runner's `was_killed()` checks.

## Root Cause — Three Cancellation Vectors

Wait machinery recap: a waiting agent's runner (`src/sase/axe/run_agent_wait.py`) writes `waiting.json` into its
artifact dir and polls for `ready.json`. The `waits` lumberjack chop (`src/sase/scripts/sase_chop_wait_checks.py`) scans
all `waiting.json` markers, resolves dependencies via `src/sase/core/wait_dependency_resolution/`, and writes
`ready.json`. When the runner reads a `ready.json` with `"cancelled": true`, it calls `_finalize_cancelled_wait()`
(`src/sase/axe/run_agent_runner.py`), which writes a `done.json` with outcome `stopped` + `queue_cancelled` — displayed
as FAILED in the TUI.

Three code paths produce that cancellation:

1. **TUI kill/dismiss/cleanup** — `_resolve_waiters_before_artifact_delete()` in
   `src/sase/ace/tui/actions/agents/_killing_utils.py`. Every kill/dismiss path funnels through
   `delete_agent_artifacts()`, which scans all `waiting.json` markers in the project and, when the deleted agent's
   `done.json` outcome is not `completed` (e.g. `killed`), writes
   `ready.json {"cancelled": true, "reason": "dependency_failed"}` into every waiter that references the deleted agent
   by name or identity. This is the path that fired in the observed failure (the `waits` chop was not even running at
   the time).

2. **wait_checks chop failed-dependency branch** — `sase_chop_wait_checks.py` writes a cancelled `ready.json` when
   `dependency_resolution_status()` reports `failed`. That status can only arise from _identity_ dependencies
   (`wait_for_artifacts`, created by family-attach/fork launches): `WaitDependencyIndex.identity_status()` returns
   `failed` when the pinned artifact's `done.json` outcome is in `FAILURE_OUTCOMES` (`failed`/`killed`/`stopped`) or has
   `repeat_stopped`. (Name-based waits already have the correct semantics in the chop: a failed newest run simply does
   not resolve, and a later same-named completed run does — see
   `tests/test_axe_chop_wait_checks.py::test_later_same_name_completed_agent_resolves_after_killed`.)

3. **Runner-side checks** — `run_agent_wait.py`:
   - `_initial_dependency_result()` performs the same failed-identity-dep check at launch and returns a cancelled result
     before waiting even starts.
   - `_read_ready_result()` parses the cancelled `ready.json` shape and propagates it.
   - Additionally, `_WAIT_MAX_TIMEOUT` (24h) makes the runner "proceed anyway" after a day, which also contradicts
     wait-forever semantics (it starts an agent whose dependencies never completed).

## Design

### New resolution semantics (never "failed")

- **Named dependency** (`waiting_for`): unchanged mechanics — satisfied when the newest same-named candidate has a
  successful `done.json`. Missing name → keep waiting. Newest run failed/killed → keep waiting (a same-named relaunch
  will resolve it; the TUI kill-and-edit retry flow reuses the agent name, so this is the primary restart path).
- **Identity dependency** (`wait_for_artifacts`): satisfied when the pinned artifact (or its family generation, for
  family-attach roots) reaches identity success (`completed`/`plan_rejected`). If the pinned artifact has a failure
  outcome (`failed`/`killed`/`stopped`/`repeat_stopped`) or its artifacts were deleted, **fall back to name-based
  resolution** using the dependency's recorded `name`: a newer same-named run that completes satisfies the dependency
  (restart semantics). Otherwise keep waiting. `identity_status()` never returns `failed`.
- `WaitDependencyStatus` loses its `failed` state and `failed_dependencies` payload (grep confirms the only consumers
  are the cancellation paths being removed). Keep the internal `is_failed`/`FAILURE_OUTCOMES` helpers — they are still
  needed to decide when the identity → name fallback applies (a pinned run that is merely still running must NOT fall
  back).

### Cancellation machinery removal

- `sase_chop_wait_checks.py`: delete the `status.failed` branch (cancelled `ready.json` write) and the `cancelled`
  summary counter. Failed deps count as `unresolved`.
- `_killing_utils.py`:
  - Remove the `cancelled` ready-marker write for failed/killed dependencies — deleting a failed dependency leaves
    waiters untouched (they return to "dependency does not exist yet" state).
  - Keep the `dependency_succeeded` branch: dismissing a _completed_ dependency must still write a resolved `ready.json`
    when the waiter's full dependency set is satisfied, since deletion erases the completion record from the index.
    Inside `_ready_data_for_completed_dependency()`, a `failed` status is no longer possible; only fully-`resolved`
    produces a marker.
- `run_agent_runner.py`: remove `_finalize_cancelled_wait()`, `_CancelledWaitFinished`, and the cancelled-result call
  site (including the `queue_cancelled` done-marker field and its notification).
- `run_agent_wait.py`:
  - `_initial_dependency_result()`: drop the cancelled path — the launch-time check only distinguishes "already
    satisfied" from "must wait".
  - `_read_ready_result()`: a `ready.json` with `"cancelled": true` is now only possible as a stale marker from a
    previous SASE version (or a crashed mid-kill write). Treat it as _not ready_: delete the stale marker and keep
    polling. This keeps old on-disk state from failing waiters after upgrade.
  - Remove `_WAIT_MAX_TIMEOUT` and the "Wait timeout exceeded, proceeding anyway" branch — the dependency poll loop runs
    until `ready.json` appears or the waiter is killed.
  - `_WaitDependencyResult` simplifies accordingly (likely to a plain bool / removal).

### Preserving completion records when artifacts are deleted (anti-strand memo)

With forever-waits, one pre-existing gap becomes a real hang: dismissing/cleaning up a _completed_ dependency while the
waiter still has other unresolved deps silently erases the completion record (the index only sees live artifacts;
today's code documents this in
`tests/test_agent_dismiss_persistence.py::test_delete_agent_artifacts_keeps_waiter_with_other_unresolved_dependency`).
Bulk "cleanup done agents" makes this easy to trigger.

Fix: in `_resolve_waiters_before_artifact_delete()`, when the deleted dependency succeeded but the waiter's full set is
_not_ yet satisfied, merge the dependency into a `resolved_deps` memo list in the waiter's `waiting.json` (names for
name deps; enough identity to match for identity deps). `dependency_resolution_status()` treats memoized entries as
satisfied. The runner already re-writes `waiting.json` only for time floors, and the chop reads it fresh each pass, so a
memo field is safe; `update_agent_artifact_index_for_marker_mutation()` must be called after the memo write like every
other marker mutation.

Chosen tradeoff: first observed completion wins for memoized deps (a later relaunch of the same name does not
"un-resolve" the memo). This matches the existing race behavior of chop-based resolution.

### Intentional behavior changes to call out

- A queued family-attach/fork follow-up whose parent is killed no longer cancels — it stays WAITING until the parent is
  relaunched and completes, or the user kills the waiter. The "Queued agent stopped / Parent dependency failed"
  notification disappears with the code path.
- `%repeat` STOP cascades are unaffected: STOP-skipped slots finalize as `completed` (+ `repeat_stopped`), so downstream
  name-based chain waits still resolve and the STOP output variable still propagates. However, an _identity_ waiter
  attached to a `repeat_stopped` slot previously cancelled; it now keeps waiting (with the name fallback available).
  This is the desired wait-forever behavior.
- The 24h "proceed anyway" wait cap is removed for dependency waits.
- No TUI changes needed: WAITING agents keep their bucket, and the per-dependency status badges in the detail header
  (`_agent_display_header.py`) already derive live from status buckets — a failed dep correctly shows ✗ next to a
  still-WAITING waiter.

## Implementation Steps

1. **Core resolution** (`src/sase/core/wait_dependency_resolution/`):
   - `_types.py`: drop the `failed` state / `failed_dependencies` from `WaitDependencyStatus`; keep `FAILURE_OUTCOMES` +
     candidate `is_failed` flags for fallback detection.
   - `_index.py`: rework `identity_status()` — never `failed`; add the name-based restart fallback (only when the pinned
     candidate/family generation has a failure outcome or is missing).
   - `_resolution.py`: `dependency_resolution_status()` returns only `resolved`/`waiting`; support the `resolved_deps`
     memo from the waiting marker (new parameter or part of the wait payload).
   - `_artifact_state.py`: remove `failed_dependency_record()` once unused.
2. **Chop** (`src/sase/scripts/sase_chop_wait_checks.py`): remove the cancel branch + counter; pass the waiter's
   memoized `resolved_deps` into resolution.
3. **Runner** (`src/sase/axe/run_agent_wait.py`, `src/sase/axe/run_agent_runner.py`): remove cancellation handling,
   stale-cancelled-marker cleanup in `_read_ready_result()`, remove the 24h cap, delete
   `_finalize_cancelled_wait()`/`_CancelledWaitFinished`.
4. **TUI kill path** (`src/sase/ace/tui/actions/agents/_killing_utils.py`): remove cancelled marker writes; add the
   `resolved_deps` memo write for succeeded-dependency deletion when the waiter is not yet fully satisfied.
5. **Tests** — update and extend:
   - `tests/test_axe_chop_wait_checks.py`: flip `test_identity_wait_failed_parent_cancels` and
     `test_identity_wait_repeat_stopped_parent_cancels` to assert the waiter keeps waiting; add "identity dep restart
     (newer same-named run) resolves" coverage.
   - `tests/test_run_agent_wait.py`: replace `test_cancelled_wait_returns_failed_dependency_result` with
     stale-cancelled-marker cleanup coverage; update/remove the 24h-timeout test.
   - `tests/test_run_agent_runner_wait_queue.py`: remove
     `test_queued_child_failed_parent_finalizes_stopped_and_notifies`; assert a queued child with a killed parent stays
     waiting.
   - `tests/test_agent_dismiss_persistence.py`: killed-dependency dismissal leaves waiters untouched;
     completed-dependency dismissal with remaining deps writes the `resolved_deps` memo, and resolution honors it.
   - `tests/test_agent_artifact_marker_mutation_audit.py`: update the marker-writer audit for the removed/added mutation
     sites.
   - New chop test: dependency that does not exist yet keeps the waiter unresolved indefinitely and resolves once the
     (late-launched) dependency completes.

## Out of Scope / Notes

- Retry-with-new-name (`<base>.r-N` from the retry-edit flow) still does not satisfy a wait on `<base>` — same-name
  relaunch (kill-and-edit, which reuses the name) is the supported restart path, and the W wait-edit modal covers manual
  re-pointing. No hood-level name matching is added.
- Waiter processes that die (reboot, OOM) are still marked DEAD by the scheduler's process checks; reviving them is the
  existing revive flow's job.
- Optional follow-up (not in this change): a one-shot notification when a dependency fails while agents wait on it
  ("bob-cli-9 is waiting on failed bob-cli-9.5 — retry it or kill the waiter"), so forever-waits stay visible without
  re-introducing cancellation.
- All logic touched here is Python-only backend behavior in this repo (the runner, the chop, and TUI-hosted persistence
  helpers); the Rust core exposes no wait-dependency APIs, so no cross-repo changes are required.
