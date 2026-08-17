---
tier: tale
title: Deflake the monitor idle-timeout liveness bound
goal:
  The idle-timeout supervisor test retains a real five-second no-hang guard and its
  0.2-second output-stall contract while passing under a contended full parallel lane.
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.4
bead: sase-ns.6.6.4
create_time: 2026-08-17 04:22:42
status: wip
---

- **PARENT:** [202608/backlog_top5_gates_green.md](backlog_top5_gates_green.md)
- **BEAD:**
  [sase-ns.6.6.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.4.md)

# Deflake the monitor idle-timeout liveness bound

## Goal

Complete phase bead `sase-ns.6.6.4` by making
`test_run_supervisor_idle_timeout_fires_after_output_stalls` distinguish an actually
hung supervisor from a pytest worker that was merely descheduled under full-lane
contention. Keep the production idle-timeout behavior, the 0.2-second idle budget, the
timeout-kind assertions, and a real five-second liveness failure.

## Evidence and root cause

The durable full-run store contains three failures of this node across unrelated
workspaces and change sets. The representative failure is solely
`assert 5.825556540999969 < 5.0`; the preceding `exit_status == 1` assertion passes, so
the supervisor has already fired the idle timeout and returned before the test rejects
the result. The two post-`b569cbdc2` recurrences show that this is distinct from the
fixed `BoundedLogPipe.close()` five-second join defect.

The current assertion measures total wall time in the pytest process. When that process
is descheduled, it cannot tell whether the supervisor finished before the five-second
deadline. Running the supervisor as a child process with
`subprocess.run(..., timeout=5.0)` provides the principled distinction: Python observes
the child's state before declaring its deadline expired, so a child that completed while
the parent was starved passes, while a genuinely live/hung child at the deadline raises
`TimeoutExpired` and fails the test.

## Implementation

1. Add a focused test helper in `tests/monitor/test_monitor_supervise.py` that invokes
   the real `sase.monitor.supervise` module in a child process with the existing
   `_NO_HANG_TIMEOUT` as its hard subprocess timeout. Preserve the test sandbox
   environment and avoid changing the production supervisor solely to accommodate
   scheduler contention.
2. Change only the idle-stall regression to use that helper. Assert the timeout exit
   status, `monitor_state`, `monitor_timeout_kind`, and both persisted timeout messages
   exactly as today; retain `idle_timeout_seconds=0.2` and `timeout_seconds=30.0` so the
   test still proves that output idleness, not the total runtime budget, terminates the
   command.
3. Add or retain an assertion that the initial `started` output reached the durable log
   if repeated contention runs show it is stable; do not hide a genuine premature-idle
   defect. If diagnostics instead show the supervisor remains alive beyond the hard
   subprocess deadline, stop treating this as test-only and fix the measured production
   path with focused regression coverage.
4. Leave `tests/reproducible_flake_baseline.txt` unchanged until the fix has an actual
   commit hash. Report the required post-fix `# fixed-at:` entry with the landed commit
   to the epic land agent, which can add the convention-required follow-up commit
   without fabricating a hash in this phase workspace.

## Verification

1. Run `just install`, then repeat the target test serially and with xdist contention;
   also run the complete monitor-supervisor test file.
2. Run `just check` and inspect its selected-test explanation for unexpected broadening
   or failures.
3. Run `just selection-health --fail-on-new-flake` and record its exact remaining node
   list, explicitly distinguishing historical pre-fix evidence from a new live failure.
4. Launch `just check-full` only through the SASE monitor workflow with a `--next`
   continuation. Require a durable full-run record in which the target node passes under
   the repository's full parallel lane; account for any remaining failures as other epic
   phases rather than broadening this phase.
5. Reinspect the diff, confirm no idle-timeout behavior was weakened, add any discovered
   out-of-scope work as `PROPOSED FOLLOW-UP:` on `sase-ns.6.6.4`, and close only that
   phase bead with a note naming the focused, repository, selection-health, and
   full-lane evidence.
