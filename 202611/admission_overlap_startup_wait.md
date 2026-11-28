---
tier: tale
title: Wait for holder children before measuring admission overlap
goal:
  Fix sase-1vy.5's startup observation race while preserving the two-run admission limit
  checks.
size: medium
proposed_by: bbugyi200.athena.sase-1vy.5
proposed_with: codex/gpt-6-astra
bead: sase-1vy.5
status: done
---

- **PARENT:**
  [202611/backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_2.md)
- **BEAD:**
  [sase-1vy.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vy/sase-1vy.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1vy.5](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1vy.5.md)
- **COMMITS:**
  - [4c9ccea](https://github.com/sase-org/sase/commit/4c9ccea4b2b63d9d7af6aadd3ac751575860c55e)
    — fix(tool): wait for holder children before measuring admission overlap

# Wait for holder children before measuring admission overlap

Complete phase `sase-1vy.5` of `plan:202611/backlog_epic_2.md`, addressing the failure
tracked by `sase-1vw`. This is one bounded coding task: correct the acceptance test's
startup synchronization and verify that its concurrency assertions remain effective. The
assigned phase has no prior implementation or remainder notes; a child epic is
unnecessary.

## Evidence and governing decisions

- At planning time, the main checkout is at `53acb952ef`. The overlap test still
  immediately asserts `len(_started_children(w)) == 2` after observing two admitted
  records and two waiting records.
- The original failure is recoverable with
  `sase tool show 7d9af4cef7ba6b82066e96437690602e -l`. It reports
  `AssertionError: assert 1 == 2` at that startup assertion in
  `tests/tool/test_admission_processes.py`. It never reached the final
  `_max_overlap(lines) <= 2` assertion. Thus this report is evidence of premature
  startup observation, not evidence that three children executed concurrently.
- `src/sase/tool/executor_run.py::run_recorded_body` calls `admit_before_spawn`, then
  fingerprints inputs, prepares the child environment, and calls `spawn_child`. A
  durable admission record therefore precedes the child's `start` stamp; two admitted
  records do not guarantee two stamps yet.
- Related `sase-1ve` is already closed. Commit `42255c3cce` added the analogous semantic
  wait to `test_nested_agent_check_keeps_parent_and_never_queues`; preserve that landed
  work.
- In the linked `sase-core` repository,
  `crates/sase_core/src/tool_run/store/admission.rs` grants inside
  `TransactionBehavior::Immediate` transactions in both `admission_enter` and
  `admission_poll`. Its adjacent `store/tests/admission.rs` already has
  `racing_pollers_never_exceed_the_limit`. Inspection supports the test-race diagnosis
  but does not replace verification of real child overlap.
- Follow `decisions:rust-core-required`: any demonstrated shared admission defect
  belongs in Rust, with binding and pin updates if needed. A test synchronization
  correction requires no product admission change.
- Follow `decisions:check-full-is-explicit` and `decisions:guarded-recipes`: verify with
  `sase tool run check`; do not request `check-full`.
- Follow `decisions:triage-annotates-does-not-change-exit-codes`,
  `decisions:host-owned-completion`, and
  `decisions:handoffs-end-through-host-completion` for failure evidence and completion.
  No memory edits or additional plan decisions are needed.

## Implementation

1. Re-read the assigned phase and `sase-1vw`, and check the current test and history for
   intervening fixes. Retain the original failure's full run id in the phase's eventual
   verification note.
2. In `test_limit_two_with_four_runs_never_overlaps_more_than_two`, keep the waits for
   two admitted holders and two queued waiters. Before touching `w.release`, use the
   existing `_wait_for` and `LOAD_TOLERANT_TIMEOUT` to wait until at least two
   child-start stamps exist, then assert that the count is exactly two. Explain briefly
   that admission is committed before child startup. Waiting for at least two followed
   by the exact assertion should fail promptly if a third child appears. Do not add a
   fixed sleep, increase timeouts, introduce retries around assertions, or remove the
   child-start assertion.
3. Keep the real four-process exercise and its final checks: all wrappers succeed, all
   four children start, maximum observed overlap stays at most two, admission records
   settle appropriately, at least two runs waited, and the existing wait/grant output
   expectations hold. Include stamp lines in relevant assertion diagnostics where
   helpful. This existing acceptance test is the regression coverage; avoid adding
   mock-only tests that merely mirror the polling implementation.
4. If verification instead reveals peak overlap above two, preserve the stamps, wrapper
   logs, and admission records. Distinguish ordinary slot grants from the documented
   max-wait or unavailable bypasses before changing admission. Open `sase-core` through
   `/sase_repo` before further access; repair any demonstrated shared grant defect there
   and run its required checks. Do not mask a genuine over-limit result with the startup
   wait or relax the overlap assertion.

## Verification and completion

1. Run the focused node five times as independent pytest invocations, stopping and
   investigating any failure:
   `.venv/bin/python -m pytest tests/tool/test_admission_processes.py::test_limit_two_with_four_runs_never_overlaps_more_than_two`.
   The historical five green reruns did not fix the race; explain the synchronization
   change using the recovered traceback and executor ordering, not the pass count.
2. Run the whole module serially. Then exercise the module with a bounded two-worker
   request through the repository's governed pytest lane, for example
   `SASE_PYTEST_WORKERS=2 just test tests/tool/test_admission_processes.py`. Confirm the
   actual grant and record whether concurrent execution occurred. This tests contention
   among real wrappers without launching an unbounded host stressor. Preserve all
   exact-count and overlap checks in every run.
3. Read `lint_and_test.md` through `/sase_memory_read`, run `just fix`, and run
   `sase tool run check`. Route a long final check through `/sase_monitor` if needed,
   staging it before finalization and recording the required follow-up actions. Run the
   linked repository's wrapped check too if it was changed. Treat UNKNOWN failures as
   unresolved until investigated. A failure reproduced identically on the clean base is
   recorded as `PROPOSED FOLLOW-UP:` on this phase, with any existing owner bead cited;
   it does not prevent completing this assigned phase.
4. Record the root cause, repeated and contention test results, check run id, and any
   remaining follow-up on `sase-1vy.5`. Do not create task beads. The current user
   instruction permits closing only `sase-1vy.5`; leave the `sase-1vw` task's closure to
   the parent epic's land agent despite the older phase description's broader closing
   instruction. Never close `sase-1vy` or any ancestor plan bead.
5. Run `sase bead epic-symbols sase-1vy.5` immediately before closing. Planning found no
   entries; resolve or re-key any that appear before completion. Close only the assigned
   phase with `sase bead close sase-1vy.5 --note "<verification evidence>"`, then finish
   through `/sase_final`. Do not manually change its status or reopen it.

Acceptance requires a semantic startup wait, preserved detection of excess child
overlap, repeated focused verification plus module contention coverage, and the wrapped
repository check with any independently established base failures documented.
