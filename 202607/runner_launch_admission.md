---
create_time: 2026-07-13 10:30:38
status: wip
prompt: 202607/prompts/runner_launch_admission.md
tier: tale
---
# Plan: Make runner admission account for newly launched immediate agents

## Problem and confirmed timeline

Agent `7u` used `%wait(runners=0)`, so it was intended to start at a genuinely quiet point. The persisted artifacts
confirm the user's suspicion about the observed behavior, with one timing nuance:

- `7q.w1` was the only admitted root runner, from `2026-07-13T13:41:07.119658Z` until `2026-07-13T14:01:56.839607Z`.
- While `7q.w1` was still running, it launched the `sase-5w` work chain. The immediate first phase, `sase-5w.1`, was
  launched at `10:01:15 EDT`; the remaining dependency-waiting phases and lander followed through `10:01:23 EDT`.
- `7u` did not literally overlap `7q.w1`: it claimed its slot at `2026-07-13T14:01:57.923559Z`, about one second after
  `7q.w1` stopped.
- However, `sase-5w.1` was already a live, immediate root launch. Its runner spent roughly 47 seconds preparing the
  primary and linked workspaces before reaching `wait_for_runner_slot()`. It claimed at `2026-07-13T14:02:02.047316Z`,
  after `7u` had started. Thus `7u` opened during a false quiet window while earlier launched work was still in
  pre-admission setup.

The evidence is in the completed metadata under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/13/20260713094059`, `.../20260713094153`, and
`.../20260713100115`, plus the corresponding workflow logs. No source files have been changed during this diagnosis.

## Root cause

The runner-slot system currently has two individually intentional rules that combine badly:

1. `running_root_agent_count()` counts a live root only after the atomic slot claim records `run_started_at`.
   Non-deferred agents do primary and linked-workspace preparation before that claim, leaving a potentially long
   `STARTING` interval that is neither counted as active nor represented in the slot queue.
2. Admission is strict FIFO across every waiter. An older `runners=0` waiter remains the queue head even while its own
   threshold is ineligible, so a later immediate launch with available capacity cannot claim early and make itself
   visible to the drain condition.

Marker writes already refresh the artifact index, and the slot gate performs a fresh filesystem scan under the global
`runner_slots.lock`. Adding another periodically refreshed counter would duplicate the source of truth, introduce
crash-recovery drift, and still leave the FIFO head-of-line problem. The fix should instead publish the existing atomic
claim earlier and make FIFO threshold-aware.

## Product contract

Treat `%wait(runners=N)` as a threshold, not a reservation of future capacity:

- A waiter is eligible when the current admitted root count is at most its effective threshold.
- Among waiters that are eligible at the current count, the oldest live waiter wins. An ineligible low-threshold waiter
  does not block later work whose higher threshold permits it to run.
- `%wait(runners=0)` therefore starts only at a real global lull. New immediate root launches may keep it waiting; this
  is a drain condition, not an exclusive fence. Once the barrier is admitted, later work may still start if its own
  threshold allows it, preserving the existing non-exclusive behavior.
- Immediate root agents become admitted before expensive workspace preparation. Dependency/time/fork waiters remain
  uncounted until their prerequisites resolve, preventing parent/child and dependency-chain deadlocks.
- Follow-up/child agents remain exempt from the root slot pool.

This deliberately replaces the current documentation promise that every later waiter queues behind a drain barrier. It
matches the reported expectation and avoids a low-threshold waiter idling otherwise available capacity.

## Design

### 1. Make the shared admission policy work-conserving

Extend the pure runner-slot waiter projection in `src/sase/core/runner_slots/` to retain each waiter's already-persisted
effective threshold. Change admission to select the first **currently eligible** live waiter rather than the first
waiter unconditionally. Keep timestamp/path tie-breaking deterministic and keep the decision inside the existing global
file lock so two processes still cannot oversubscribe a threshold.

Do not add a second counter or daemon-owned state. `agent_meta.run_started_at`, process liveness, `waiting.json`, and
the existing lock remain the authoritative cross-process protocol.

### 2. Claim immediate roots before workspace preparation

Refactor `src/sase/axe/run_agent_runner.py` so directive metadata and wait classification are established before any
expensive workspace work, then use one admission path:

- Immediate roots, including prompts with only `runners=`, call `wait_for_runner_slot()` immediately after metadata is
  published.
- Named/dependency/time/fork waits first resolve their prerequisites and repeat-stop decision, then call the same gate.
- Only an admitted, non-repeat-stopped root prepares or materializes its primary and linked workspaces and proceeds to
  execution.
- Preserve the current deferred-workspace behavior: no real workspace is claimed until dependencies and runner admission
  have both completed.

Ensure the gate is invoked exactly once per root lifecycle. Early workspace-preparation failures must still produce
terminal metadata (`stopped_at`/done or error markers) so the liveness scan releases the claimed slot. The visible
`run_started_at` and reported runtime will now include workspace preparation for ordinary roots, matching the meaning of
“active runner” used for concurrency control.

### 3. Keep status presentation consistent

Update the Agents-tab runner-slot context and explanatory copy so queue position reflects threshold-aware eligibility
rather than implying a single blocking FIFO head. Continue deriving display state from the already-loaded snapshot in
O(rows); do not add disk reads, subprocess work, or synchronous refreshes to the Textual event loop.

Update `docs/xprompt.md`, `docs/troubleshooting/runner-slots.md`, and the relevant ACE documentation to explain that a
drain barrier waits for true quiescence and can be delayed by newer eligible launches. Retain the existing `WAITING`
runner count/threshold rendering and make any queue wording unambiguous.

### 4. Preserve performance characteristics

The lifecycle move must not add a global artifact scan: an uncontended immediate root should still perform one
check-and-claim scan, only before preparation instead of after it. Parked agents keep the existing coarsely timed poll,
and child agents keep the zero-scan exemption. Threshold-aware selection should be folded into the existing in-memory
waiter traversal, adding at most O(waiters) cheap comparisons after the scan that already dominates the operation.

Counting workspace preparation against the configured root cap is intentional: it closes the false-idle window and also
bounds concurrent git/linked-repository preparation, which can otherwise create heavier I/O contention. Below the cap
there is no added launch delay.

## Tests

1. Extend the pure admission tests to cover:
   - an older ineligible `runners=0` waiter not blocking a later eligible default-threshold waiter;
   - FIFO ordering among multiple waiters that are eligible at the same running count;
   - the drain waiter becoming eligible and winning deterministically once the running count reaches zero;
   - stale/dead, done, non-agent, and child records remaining excluded.
2. Add a runner lifecycle regression that blocks ordinary primary/linked workspace preparation and proves
   `run_started_at` is already persisted and counted before preparation begins. Update the existing timestamp and
   preparation-failure assertions to the new active-runtime semantics.
3. Replace the existing fakey “drain barrier prevents later queue jump” scenario with the reported race:
   - one root is running;
   - a `runners=0` barrier parks;
   - a later immediate root with available threshold is admitted and remains active;
   - stopping the original root does not release the barrier;
   - the barrier starts only after the later root also stops.
4. Cover default-cap saturation, live cap changes, killed/preparation-failed roots, repeat-stop, dependency chains, and
   child exemptions to ensure early admission neither leaks slots nor creates deadlocks.
5. Assert the uncontended path performs one slot scan and the child path performs none. Keep TUI queue-context tests
   pure and linear; update visual snapshots only if user-facing queue wording actually changes.

## Verification

- Run focused runner-slot, runner-lifecycle, fakey, and Agents-tab tests first.
- Run `just install`, then the mandatory `just check` for the repository-wide static and test gates.
- Run `just test-visual` only if rendered status text changes, accepting snapshots only after inspecting the diffs.
- Perform a live post-review reproduction matching the timeline above and inspect persisted timestamps/markers: a later
  eligible immediate root must become active while the original root is still running, and the `runners=0` agent must
  remain parked until both have stopped.

## Scope boundary

The shared policy and lifecycle already live behind the Python runner-slot facade and use the Rust scanner wire; no new
Rust API or persistent schema is needed unless implementation reveals that the existing waiter threshold is not
available in the scan projection. Do not change memory files or unrelated AXE hook/mentor runner pools.
