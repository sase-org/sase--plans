---
tier: tale
size: medium
title: Make accepted gate execution failures durable and recoverable
goal:
  Ensure an accepted gate decision cannot be replaced while its verified execution owner
  is live, while dead or durably failed attempts can be recovered without rerunning
  completed commands and all waiting surfaces observe the failure.
proposed_by: bbugyi200.apollo.sase-zr.7.1
bead: sase-zr.7.1
create_time: 2026-09-16 14:33:40
status: wip
---

- **PARENT:**
  [202609/sase_zr_close_out.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_zr_close_out.md)
- **BEAD:**
  [sase-zr.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zr/sase-zr.7.1.md)

# Gate decision integrity and failure recovery

## Context

The accepted epic design requires one coherent change across `sase-core` and `sase`.
Today the Rust acceptance policy treats every differing receipt as a conflict, while
Python bypasses that conflict whenever any journal attempt is merely incomplete. That
makes a still-running command supersedable. At the same time, failures after acceptance
are written only as diagnostic error files: polling stays pending, cancellation remains
blocked by the receipt, archive failure is misclassified as a completed attempt, and no
actionable notification is restored after the original review notification was
dismissed.

Keep `response.json` as the execution-complete record, preserve `%auto`, and never put
submitted input values into the new durable outcome. Shared accept/replay/supersede
policy stays in `sase-core`; Python supplies observed owner liveness and performs I/O.

## Implementation

1. Extend the `sase-core` gate-decision wire and deterministic policy with explicit
   existing-owner liveness and durable-failure facts. Require every newly accepted
   receipt to carry a non-empty execution owner, keep identical replay idempotent,
   reject a differing answer while owner state is live or unknown, and allow replacement
   only for a proven-dead owner or the currently recorded failed attempt. Reuse the same
   core replacement decision for cancellation. Add Rust policy tests and PyO3 round-trip
   and error tests, retaining compatibility when reading legacy receipts that lack owner
   metadata.

2. Add a thin Python ownership adapter that records either the supervising
   `SASE_PROC_ID` or a boot/start-time-qualified host process identity, and verifies it
   through the durable proc store or local process identity helpers. Feed the observed
   state and the journal's current failure outcome into the Rust policy from acceptance
   and cancellation. Remove the `incomplete_attempt` conflict bypass so a blocked live
   command cannot be superseded; allow a new decision or cancellation only after the
   owner is dead or the attempt has a durable failure event.

3. Extend the execution journal with a redacted failed-attempt outcome containing the
   attempt id, stage (`command`, `terminal_prepare`, `side_effects`, or `follow_up`),
   stable error code, safe message, and timestamp. Preserve incomplete-attempt replay
   data for resume/restart. Move `attempt_completed` until after terminal preparation
   succeeds so resume after archive failure reuses completed option results and retries
   only terminal preparation. Record every exception raised after acceptance, including
   command, adapter preparation, response-adjacent side effects, and other `GateError`
   paths, without copying raw inputs or command output into the outcome.

4. Project the failed outcome through `poll_gate`/`wait_for_gate` as a terminal failure
   result, so a waiter never reports pending or `already_answered` after execution has
   failed. Teach cancellation and gate-shell reclaim to recognize live receipt owners
   and recorded failures: live owned work is left alone without repeated error logging,
   while failed or dead-owned work is recoverable under the same core policy.

5. Publish one notification per gate and attempt through the existing notification
   upsert/dedup store when a failed outcome is recorded. Include bounded redacted
   failure details plus bundle/request identity and explicit resume, restart, and cancel
   action data for the existing gate-action consumers. Dismiss that failure notification
   after a later successful response, while keeping the original review notification
   settled. Make notification write failures diagnostic-only so they cannot obscure the
   authoritative journal outcome.

6. Add focused Python regression coverage for: a conflicting answer rejected while the
   first command is barrier-blocked; owner death allowing replacement; failure allowing
   replacement and cancellation; archive failure followed by resume without command
   rerun; side-effect and post-acceptance `GateError` outcomes; failure polling/waiting;
   notification dedup and success dismissal; and reclaim ignoring a live owner. Update
   affected CLI/result serialization assertions as needed without changing existing
   successful response behavior.

## Verification

- Run the focused `sase-core` gate-decision and PyO3 binding tests, then `just check` in
  `sase-core`.
- Install the matching local Rust binding for the `sase` workspace using the
  repository's supported install target; do not edit release versions or dependency pins
  manually.
- Run the focused gate acceptance, durability, executor, archive recovery, CLI answer,
  poller/waiter, and reclaim tests in `sase`.
- Read the canonical lint/test guidance, then run `just check` in `sase` and address all
  failures caused by this phase.
- Run `sase bead epic-symbols sase-zr.7.1`, resolve or re-key every remaining symbol,
  and close only `sase-zr.7.1` with a note summarizing the verified owner-conflict,
  recovery, notification, polling, and test evidence.
