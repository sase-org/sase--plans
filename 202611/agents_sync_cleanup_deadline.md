---
tier: tale
title: Bound publication cleanup after the drain alarm
goal:
  Fix sase-1qt.2 by bounding post-timeout cleanup and git recovery while retaining the
  queued request and the existing five-second regression bound.
size: medium
proposed_by: bbugyi200.athena.sase-1qt.2
proposed_with: codex/gpt-6-astra
bead: sase-1qt.2
create_time: 2026-11-01 18:19:43
status: wip
---

- **PARENT:**
  [202611/task_backlog_epic_5.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_5.md)
- **BEAD:**
  [sase-1qt.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qt/sase-1qt.2.md)

# Bound publication cleanup after the drain alarm

Complete assigned phase `sase-1qt.2` of `sase-1qt`, addressing task `sase-xo`. Read the
phase and task with `sase bead read`, and read the approved parent design with
`sase artifact read plan:202611/task_backlog_epic_5.md "Need parent scope"`. The
parent's final decision is `optional_scope = keep_five_seeds`: this work fixes the drain
flake only. No memory edits, new options, unrelated features, task-bead creation,
timeout increases, skips, xfails, or flake-baseline exemptions.

One coding agent can implement this bounded change: the failure is reproduced, the
existing git-lock wire already carries the necessary deadlines, and the publication
transaction and failure-accounting contracts remain in place.

## Evidence and remaining uncertainty

Investigation used clean sase `9066546900ccafffcd3846f1ff56b81580c1f515` on athena with
Python 3.14.7. Opened sase-core was `e60f3e1658d34f1345bc4bef8bf112a396749c87`. The
previous fixes `1a69c8947d` and `4adee846d3` are present; their implementation must not
be redone.

- The original
  `tests/agents_sync/test_commit_publication_bounded_drain.py::test_blocked_render_is_bounded_and_leaves_the_request_queued`
  passed in isolation, taking 1.30 seconds in the test call.
- The complete bounded-drain module beside `tests/agents_sync/test_git_sync.py` passed
  with four workers: 30 tests in 11.13 seconds. The blocked-render call took 1.23
  seconds. This modest parallel run did not spontaneously reproduce the historical
  loaded-lane failure.
- A deterministic probe left a fresh owner-free `.git/index.lock` in the temporary
  sidecar immediately before entering the test's existing blocked renderer. The
  unchanged assertion failed at **15.6448 seconds** against `< 5.0`. Rendering entered
  0.155 seconds after transaction start and unwound at 0.830 seconds with
  `alarm_fired=True`. The locked drain unwound at 15.625 seconds;
  `DrainTiming.stage_durations["cleanup"]` was **14.6253 seconds**. The first alarm
  stage was `hood_publish:foo`.
- Git recovery logged all six backoffs, 0.1 through 3.2 seconds, followed by a
  7.666-second wait for the lock to become stale and then lock removal.

The probe used an in-memory pytest plugin, not source edits: wrap
`_commit_publication_locked.publish_queued_transaction`, replace its frozen
`PublicationTransactionHooks` with `dataclasses.replace`, and wrap `publish_hood` to
write the sidecar lock before calling the original hook. Wrap
`_commit_publication_drain.publish_queued_locked` to print the supplied timer's stage
durations on unwind. This is a fault-injected reproduction of the post-alarm cleanup
mechanism; it does not establish how every historical full lane created its lock.
Preserve that distinction in the completion evidence.

## Root cause and relevant code

`_commit_publication_drain._bounded_publication_drain` arms a one-shot SIGALRM.
`_commit_publication_locked.publish_queued_locked` catches the resulting exception and
runs `git_sync.clean_publication_worktree` before propagating it. By then the alarm is
spent. Both regenerable path families can invoke git reset/checkout and git-lock
recovery without a remaining cleanup deadline.

`src/sase/agents_sync/git.py` calls `run_with_git_lock_retry` without its existing
`remaining_timeout` argument and calls local `run_sdd_git` with `timeout=None`. The
eager stale-lock probe in `_git_sync_ops_payload.py` also has its own probe budget.
Merely bounding backoff sleeps would leave these other waits exposed.

There is also a core-policy gap: in sase-core
`crates/sase_core/src/git_lock/decision.rs`, `maybe_backoff` ignores
`remaining_command_deadline_ns` and `remaining_recovery_budget_ns`, while the young-lock
age-wait path consults them. A caller deadline therefore needs policy enforcement as
well as propagation. Keep this policy in Rust; Python owns clocks, signals,
subprocesses, and filesystem observations.

## Implementation

1. **Make the existing Rust recovery policy honor its budgets consistently.** Open
   sase-core with `/sase_repo` and read its `AGENTS.md`. Use the existing wire fields to
   stop when the effective budget is exhausted and prevent an ordinary retry, stale-age
   wait, disappearance retry, or removal/retry decision from authorizing work beyond
   that budget. If the next scheduled delay cannot fit, stop and preserve the last
   failure rather than granting the full delay. Keep the current behavior when no
   tighter caller deadline applies. Preserve every lock ownership, path, identity, and
   stale-age safety rule. Add focused Rust cases for expired and
   shorter-than-next-backoff budgets and the aged-lock branch, plus a Python binding
   round trip. No wire-version bump is needed just to enforce existing fields.

2. **Give exception cleanup one short, shared deadline.** Use a maximum of one second
   for the complete best-effort cleanup after a drain timeout; this is a cleanup cap,
   not an increase to the existing drain timeout or test bound. Put any new shared
   default/policy in the Rust publication-recovery domain with a thin binding; keep OS
   execution in Python. Establish one monotonic deadline around both regenerable path
   families, with no fresh allowance per git command or path family. Attempt both
   families when there is budget, and report unfinished cleanup when the shared deadline
   expires.

   Keep the original publication exception primary. Attach cleanup failure or timeout as
   diagnostic evidence, preserve the first alarm stage and keys, and allow existing
   failure accounting to leave the request queued with exactly one charged attempt for a
   hood-render timeout. Always release the sidecar lock. A later drain must still
   clean/retry successfully. Preserve append-only `files/objects` and the normal scratch
   cleanup tested by the manifest-timeout regression.

3. **Enforce the deadline at execution boundaries.** Thread the remaining cleanup budget
   through a scoped Python adapter or runner closure, preserving existing injected
   `GitRunner` compatibility. Pass it to git-lock recovery and cap local subprocess
   timeouts by the remaining time, never substituting `None` or a fresh default when the
   budget is zero. Bound the eager owner probe too. Check again after waits/probes
   before starting more work. For main-thread arbitrary Python stalls, use a separate
   bounded cleanup alarm that cannot overwrite the original publication timeout
   attribution; preserve prior signal handlers and timers. Cooperative git deadlines
   must still work where SIGALRM cannot be used. Reuse existing process
   termination/reaping machinery; leave no detached worker mutating the sidecar after
   return. Do not expand this into a general publication architecture rewrite or promise
   a hard bound against arbitrary uninterruptible kernel stalls.

4. **Turn the probe into regressions.** Add a real temporary-sidecar test that creates
   the young index lock as the blocked renderer starts, invokes the actual publication
   path, and keeps `elapsed < 5.0`, `queued`, `not published`, one queued row with
   `attempts == 1`, timeout attribution, and released-lock assertions. Add a
   deterministic policy/executor test with fake clocks showing that retry/age-wait and
   cumulative cleanup operations cannot reset or overrun the remaining budget. Test a
   stalled cleanup and verify its secondary error does not replace the render timeout.
   Verify later retry success after the test-owned contention is removed. Keep the
   original blocked-render test and all its assertions. Do not make the regression pass
   by mocking cleanup away, shortening the original blocked render, or disabling retries
   globally.

If loaded reruns expose a different failure, use the captured stages to refine this same
fix before claiming resolution. Record out-of-scope findings only as
`PROPOSED FOLLOW-UP:` notes on `sase-1qt.2`.

## Verification and completion

Read `lint_and_test.md` through `/sase_memory_read`. Before any source edit, retain a
failing run of the contention regression against the clean base; the probe above already
demonstrates the mechanism and supplies comparable stage timings. After the fix:

- Run the Rust git-lock and publication-recovery policy/binding tests through the core
  repository's prescribed recipes, never bare cargo. Build/use the updated binding in
  the isolated workspace environment, without a global install. Ensure the host advances
  `sase-core-revision.txt` past the core change; never invent a SHA. Consult
  `docs/rust_backend.md` through the required memory workflow if needed for the existing
  build/pin procedure.
- Run the bounded-drain module, drain-accounting tests, queue tests, `test_git_sync.py`,
  and relevant git-lock recovery tests. Exercise the existing alarm-restoration,
  manifest cleanup, cleanup-error, and successful-retry cases.
- Run at least three modest parallel repetitions of the bounded-drain module alongside
  the git-heavy agents-sync tests, retaining the actual granted worker count and
  per-stage timings. Include a broader agents-sync neighbor run if needed to reproduce
  the original contention. Use governed worker admission; do not create artificial
  host-wide CPU saturation. Use `/sase_monitor` before starting a command expected to
  outlast the turn.
- Run `sase tool run check` in each changed repository. Do not run `just check-full`.
  Diagnose any failure; if it reproduces identically on the clean base, record
  `PROPOSED FOLLOW-UP:` with the existing task ID when known, and do not leave the
  completed phase open solely for that baseline failure.

Record base/revisions, host, runtime, before/after elapsed and cleanup timings,
loaded-run results, and any limits on `sase-1qt.2`. Add
`TASK RESOLVED: sase-xo — <verified evidence>` only after the acceptance above is met.
Then run `sase bead epic-symbols sase-1qt.2` (there were no entries during planning),
resolving entries or re-keying them to a still-open parent/later phase before closing.
Close only `sase-1qt.2` with `sase bead close sase-1qt.2 --note "<what was verified>"`.
Do not close `sase-xo`, `sase-1qt`, or an ancestor plan: the parent lander owns those
closures. Finish through `/sase_final`; host finalizers own commits.
