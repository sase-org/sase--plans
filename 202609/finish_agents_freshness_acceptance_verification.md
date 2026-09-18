---
tier: epic
title: Finish Agents freshness acceptance verification
goal:
  Prove the sase-124.8.4 trace-writer remediation on the live/scripted path it changed,
  classify the previously failed trace benchmark on the integrated tree, and obtain the
  required combined-tree verification without absorbing independently owned latency or
  marker work.
parent_bead: sase-124.8.4
phases:
  - id: trace-remeasurement
    title: Remeasure the trace remediation and settle the failed benchmark
    depends_on: []
    size: medium
    description:
      "trace-remeasurement: prove the off-loop trace writer on a dedicated live/scripted
      TUI, rerun the failed trace benchmark, and repair only evidence-backed regressions
      caused by sase-124.8.4."
  - id: combined-verification
    title: Verify the integrated Agents freshness cohort
    depends_on:
      - trace-remeasurement
    size: small
    description:
      "combined-verification: audit the remeasurement result and post-phase drift, rerun
      the focused integration cohort, and complete monitored combined-tree verification."
proposed_by: bbugyi200.athena.sase-124.8.4.land
create_time: 2026-09-18 01:13:20
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_agents_freshness_acceptance_verification.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_agents_freshness_acceptance_verification.md)
- **PARENT:**
  [202609/complete_agents_freshness_acceptance.md](https://github.com/sase-org/sase--plans/blob/main/202609/complete_agents_freshness_acceptance.md)

# Finish Agents freshness acceptance verification

## Context

The landing audit for parent epic `sase-124.8.4` confirmed the controlled acceptance
capture and the implementation commit `43d66775935583c3ea525bf9bf750a357438830d`. That
commit moves trace JSONL directory creation and append writes off a running event loop
via the `sase-tui-trace-writer` thread, adds an explicit flush seam for immediate
readers, and updates the trace tests and benchmark reader. Current focused coverage is
green: `tests/ace/tui/test_tui_trace.py` plus
`tests/ace/tui/test_event_handlers_auto_refresh_dirty_flags.py` passed 54 tests.

The evidence artifacts `file:explicit:f5f58a0ec40e12dd1aa9bebc` and
`file:explicit:144230fdd9b2d38483af7ac9` identify two verification obligations that the
closed remediation phase did not discharge:

1. The original live watchdog captured 0.7-0.8 second countdown/info-panel hitches with
   stacks in `trace.py` at `Path.mkdir` and `open`. Because the source changed, the
   accepted plan requires a current live/scripted measurement of that affected target,
   not only a monkeypatched unit test.
2. `tests.perf.bench_tui_trace` failed to settle within 20 seconds during the
   loaded-host capture. The remediation note did not rerun it, and the parent plan
   requires the child land path to run combined `just check-full` through
   `/sase_monitor`.

Do not repeat the already-valid 30-minute busy capture or manufacture a quiet-host
result. The ten-minute candidate was honestly classified `missed_busy_host`, and marker
latency remains external because `sase-zr.7.3` and `sase-zr.7.5` are still in progress.
Preserve the existing capacity/attention behavior and the `sase-127.1/.3/.4`
bounded-load, forced-fleet-source, and stable-search integrations.

The landing agent already triaged every phase proposal. AXE j/k evidence was routed to
active epic `sase-zn`; Fleet fault evidence was routed to `sase-xe.16.11`, corroborating
reopened phase `.3`; the two full-lane/pass-isolation Commits/Stitches failures were
routed to `sase-j7` after an exact current-tree rerun passed; the trace-benchmark
startup failure stays in this plan because the trace I/O fix could affect it. Do not
create duplicates or absorb those three externally owned scopes.

## Phase: trace-remeasurement

1. Re-read `tui.md`/`tui_perf.md`, `lint_and_test.md`, the parent epic and both phase
   notes, the two evidence artifacts above, current `trace.py`, and commit `43d6677593`.
   Review commits after that revision before editing. Preserve all concurrent
   screenshot, gate, memory, Agents-refresh, and Fleet changes; fix only a regression
   caused by this epic if verification exposes one.
2. Exercise the changed trace path in a dedicated test-owned TUI/session with fresh,
   explicit trace and watchdog output paths. Confirm the pane PID and output timestamps
   advance. Drive the countdown/info-panel path with tracing enabled long enough to
   produce multiple records, flush/read the JSONL, and report sample counts plus
   p50/p95/max. Inspect watchdog stacks and show that no main-thread stall above 500 ms
   is attributable to trace `mkdir`/`open`/write work. Host-contention stalls from other
   stacks must be reported separately, not relabeled as a pass or fixed by raising a
   budget. Do not touch the user's interactive TUI session.
3. Rerun the bounded `tests.perf.bench_tui_trace` scenario against fresh output files
   through `/sase_monitor`. If startup now settles, report all scenario results. If it
   still fails, preserve the exact timing, host load, and event-loop/pump stacks and
   decide from evidence whether the failure is trace-fix work or independently owned
   benchmark/host-load work. Repair only an epic-caused defect and add deterministic
   regression coverage for any source change; otherwise record the deliberate routing
   outcome on the child bead.
4. If the remeasurement requires source changes, run `just fix` and focused tests for
   every affected behavior. Close this phase with the fresh trace/watchdog measurements,
   benchmark outcome, current revision, and any source/test commit. Record genuinely
   independent residuals as `PROPOSED FOLLOW-UP:` notes rather than creating tasks.

## Phase: combined-verification

1. Read the `trace-remeasurement` note and actual resulting source/tests, then inspect
   commits that landed after that phase started. Integrate only changes that should use
   or preserve the off-loop trace/freshness behavior; do not reimplement concurrent
   work.
2. Re-run the focused trace/auto-refresh tests and the capacity/attention plus
   `sase-127.1/.3/.4` integration cohort named by the parent plan. Run `just fix`
   inline. Then run the required combined `just check-full` through `/sase_monitor`,
   using the `TESTING`/`TESTED` handoff expected by `lint_and_test.md`. Triage unrelated
   failures precisely and rerun exact nodes on the unchanged tree when needed; do not
   claim a green gate from a partial lane.
3. Close only this phase after checking the child epic's symbols and recording exact
   focused and combined verification results plus every phase proposal disposition. The
   child land agent must re-read both phase notes and actual commits, integrate
   post-phase drift, and close only this child epic. Return control through
   `parent_bead: sase-124.8.4`. Do not close the parent epic, alter either parent plan's
   status, perform the parent's final Symvision cleanup, or close `sase-124.8`.

## Acceptance

- Trace JSONL remains complete and readable while its filesystem work is demonstrably
  absent from the live TUI event-loop stall stacks.
- The previously failed trace benchmark has a current, evidence-backed result and any
  epic-caused failure is repaired with deterministic coverage.
- Focused integration coverage and monitored `just check-full` have current outcomes;
  every failure is either fixed in scope or deliberately attributed without
  greenwashing.
- No independently owned marker, AXE, Fleet, Commits/Stitches, or generic host-load work
  is duplicated in this child.
