---
tier: tale
goal: Root agents paused for user answers stop blocking runner-slot drain barriers,
  then reacquire capacity safely before resuming.
create_time: 2026-07-15 13:04:05
status: done
prompt: 202607/prompts/question_paused_runner_slots.md
---

# Plan: Yield runner slots while agents await answers

## Diagnosis and intended behavior

The `runners` keyword is parsed and persisted correctly. Both observed queue heads carry an explicit `wait_runners: 0`
and valid FIFO timestamps. The unexpected occupancy comes later: runner admission treats every live root with a
historical `run_started_at` as still running. The existing `sase-5y.5` root has such a timestamp and a live
`pending_question.json`, so it counted as one runner before this investigation launched and continues to block both
drain barriers while it is paused for the user.

Change the slot lifecycle so a live root that is blocked in the explicit question-response loop temporarily yields its
runner slot. Preserve the global-cap guarantee by making that root reacquire through the same locked, FIFO admission
gate after an answer arrives and before it performs follow-up work. A yielded root must therefore be absent from the
occupied count while `QUESTION`, visible as a normal runner-slot waiter if its answer is ready but capacity is not, and
counted again only after an atomic claim. Child agents remain exempt, and pre-execution dependency/time/runner waits
keep their current ordering and threshold semantics.

## Implementation

1. Refine the shared runner-slot occupancy predicate to distinguish a live admitted root from one whose authoritative
   pending-question marker says it is blocked on user input. Reuse the existing Rust-backed agent-scan projection for
   `pending_question.json`; no new scan wire or marker schema should be introduced unless implementation proves the
   existing marker cannot make the transition race-free.
2. Extend the question-response handoff so the pending-question marker remains authoritative throughout the pause. Once
   an answer is available, route the root through the global runner-slot gate and remove the pause marker as part of the
   successful locked claim. If capacity is unavailable, publish the normal runner-slot waiting state and preserve FIFO
   ordering until admission. Killing the agent during either pause must clean up its markers without wedging the queue.
3. Align every occupancy projection with admission: the ACE Agents-tab context and integration/CLI JSON must exclude an
   unanswered `QUESTION` root, count an admitted/resumed root, and show answered-but-queued work with the effective
   threshold, occupied count, and queue position. Keep status precedence deterministic when question and runner-wait
   markers briefly coexist.
4. Update runner-slot and xprompt documentation to state that unanswered question pauses yield capacity and that
   answering may place the agent in the runner queue before work resumes. Preserve the distinction between this
   temporary yield and the documented non-exclusive nature of `%wait(runners=0)` after admission.

## Validation

- Add focused pure-admission tests proving unanswered-question roots do not occupy slots while ordinary started roots,
  answered/resumed roots, and all existing root/child/stale cases retain their behavior.
- Add runtime coverage for the full lifecycle: a capped running root enters `QUESTION`, a `%wait(runners=0)` root is
  admitted, the answered root waits rather than oversubscribing, and it resumes in FIFO order after capacity is freed.
  Cover kill/cleanup while the answered root is queued.
- Add ACE and integration/CLI projection tests so displayed occupancy and queue context match the scheduler's decisions.
- Re-run the existing runner-slot concurrency, drain-barrier, live-config, crash, and child-exemption suites, then run
  the repository-required `just install` followed by `just check`.

## Risks and compatibility

The critical race is answer arrival versus another root claiming the newly yielded capacity. The pause marker must not
be cleared before the resumed root owns a slot under the global lock. Existing persisted question and waiting markers
must remain readable, and an upgraded waiter should immediately stop counting an already-paused legacy root without
requiring manual marker edits.
