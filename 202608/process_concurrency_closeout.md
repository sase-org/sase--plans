---
tier: tale
title: Stabilize process-concurrency and isolation tests
goal:
  Complete sase-rm.11 with deterministic process, lease, cache, and workspace-isolation
  coverage.
size: medium
proposed_by: bbugyi200.athena.sase-rm.11
bead: sase-rm.11
create_time: 2026-08-20 15:23:59
status: wip
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.11.md)

# Plan: Stabilize process-concurrency and isolation tests

## Outcome

Complete phase bead `sase-rm.11` by repairing or conclusively re-evaluating its nine
assigned tasks (`sase-lk`, `sase-n6`, `sase-nc`, `sase-nr`, `sase-or`, `sase-qk`,
`sase-qp`, `sase-qs`, and `sase-r4`). The resulting tests must distinguish real process,
slot, and lease failures from scheduler delay; must not fork the multi-threaded pytest
worker; must not reuse state across temporary SASE homes; and must not read the
launching agent's ambient workspace occupancy.

The implementation remains in the primary `sase` repository. Do not close any of the
nine task beads: record one close-ready evidence block per task on `sase-rm.11` for the
parent epic's land agent. Close only `sase-rm.11`, after the required epic-symbol audit.

## Implementation

1. Re-establish the test environment and baseline evidence.
   - Re-query the phase and all nine task beads before editing so a newly claimed,
     snoozed, or closed target is not raced.
   - Run `just install`, then reproduce deterministic failures and run isolated
     baselines for the named monitor, runner-slot, process-group, suite-gate,
     research-swarm, and linked-repository nodes.
   - For `sase-nr`, inspect `tools/run_pytest`'s workspace-scoped base-temp handling and
     make a bounded attempt to reproduce the reported disappearing `pytest-current`
     teardown failure. Change cleanup only if the actual traceback identifies a narrow
     repository-owned race; otherwise retain the current cleanup behavior and record
     detailed canceled-close evidence rather than hiding third-party teardown errors.

2. Separate liveness verdicts from host scheduling delay.
   - In `tests/monitor/test_monitor_supervise.py`, move the no-hang verdict for the
     subprocess-driven supervisor cases into the child/driver boundary so the parent
     transport wait cannot mistake an unscheduled pytest worker for a supervisor hang.
     Preserve the readable output-tail assertions and add a proof that a deliberately
     wedged supervisor still trips the child-owned liveness verdict. Retire the live
     `sase-lk` baseline row only through the repository's supported fixed-evidence
     mechanism when a valid landing instant is available; do not merely delete it.
   - In `tests/fakey/test_runner_slots_e2e.py`, replace short wall-clock barrier failure
     with semantic start/park/release verdicts and diagnostics that distinguish a true
     runner-slot deadlock from a descheduled Fakey subprocess. Preserve the child
     exemption, family slot retention, FIFO repeat admission, and root-cap assertions.
   - Add a module-owned process-start seam in `src/sase/noninteractive_subprocess.py` so
     `tests/test_noninteractive_subprocess.py` can start a real child group, observe its
     flushed PGID deterministically, then exercise timeout cleanup without assuming the
     child reaches `print()` within 0.2 seconds.

3. Remove unsafe fork-after-threads test sites while retaining real contention.
   - Replace the `multiprocessing.get_context("fork")` sites in prompt-artifact staging,
     run-log append, and agents-publication-outbox tests with in-process thread/barrier
     seams over the same production locks and files. Keep each concurrency assertion
     (well-formed manifest, complete JSONL records, and no lost/duplicate outbox
     requests).
   - Replace the direct `os.fork()` dead-PID helper in the occupancy-guard tests with a
     completed subprocess PID.
   - For monitor/proc group-kill tests that genuinely require a process boundary, have
     an already isolated child subprocess spawn its grandchild via `subprocess.Popen` in
     the same process group. Document why that boundary is required and avoid any fork
     call in the multi-threaded pytest process.
   - In `tests/test_procs_supervisor.py`, publish the grandchild PID atomically (same
     directory temporary file plus `os.replace`) and fail explicitly if the complete PID
     never appears before teardown. This jointly satisfies `sase-or` and `sase-qk`
     without replacing one race with a parse retry.

4. Make suite-gate and registry cache tests deterministic.
   - Extend the suite-gate holder/reclaim seam with an explicit `now` value or clock
     callable and rewrite `test_fresh_heartbeat_is_not_reclaimed` to judge a known
     heartbeat at a known instant. Keep a real token-lock conflict, assert the fresh
     holder is never signaled/reclaimed, and retain separate stale/max-hold coverage.
   - Scope the agent-name registry scan caches to their source root/signature or expose
     one production reset that clears all derived registry scan state. Use that API in
     test home isolation rather than clearing three private registry fields. Add both
     triggering two-file orders plus isolated coverage for the second research-swarm
     allocation, proving `research.0` advances to `research.1` in each temporary home.

5. Eliminate ambient linked-repository workspace reads.
   - Reject an empty `primary_workspace_dir` when
     `prepare_linked_repo_workspaces_if_needed` is about to prepare a retained linked
     repository, since resolving `""` to the current working directory is unsafe in
     production as well as tests.
   - Update every relevant test caller in
     `tests/test_run_agent_runner_setup_linked_repos.py` to pass an explicit scratch
     primary workspace and workspace number. Assert the occupancy guard receives that
     scratch identity, and keep skip/fresh-sidecar paths that never need the guard
     working.
   - Replace the three live `sase-r4` baseline rows with valid fixed-evidence directives
     only when their fix instant is available; do not erase history.

## Verification and handoff

1. Run focused serial tests for every touched module and deterministic order tests for
   the `sase-qs` cache poisoners.
2. Run repeated contention lanes for the monitor supervisor, Fakey runner slots,
   noninteractive/process supervisors, suite gate, research-swarm ordering, and
   linked-repository setup. Confirm a repository-wide search finds no unsafe
   `multiprocessing.get_context("fork")` or direct test-process `os.fork()` sites;
   explicitly account for any process-boundary source strings that remain.
3. Run `just selection-health --explain` and `just selection-health --fail-on-new-flake`
   to validate baseline retirement without suppressing post-fix evidence. Run
   `just check` after all primary-repository edits.
4. Because the phase requires a real full parallel lane, run `just check-full` through
   `/sase_monitor` with `TESTING`/`TESTED` statuses and a follow-up that inspects the
   complete result. Re-run focused failures and fix any caused regression; record
   unrelated discoveries only as `PROPOSED FOLLOW-UP:` notes on `sase-rm.11`.
5. Append one evidence-rich close-ready note per assigned task to `sase-rm.11`, naming
   the cause or bounded non-reproduction, relevant files, exact commands, and results.
   Re-query all target task states, run `sase bead epic-symbols sase-rm.11`, resolve or
   re-key every returned Justfile entry, verify the working tree and checks one final
   time, then close only the phase with
   `sase bead close sase-rm.11 --note "<what was verified>"`.
