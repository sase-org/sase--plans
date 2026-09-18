---
tier: epic
title: Complete Agents freshness acceptance
goal: Supply the controlled freshness evidence still missing from sase-124.8, attribute
  every missed target, and repair only regressions caused by the Agents freshness
  work before returning to its land agent.
parent_bead: sase-124.8
phases:
- id: controlled-acceptance
  title: Capture and attribute the missing live freshness evidence
  depends_on: []
  size: medium
  description: 'controlled-acceptance: gather the required busy, idle, marker, capacity,
    stall, and navigation measurements and attribute every target.'
- id: acceptance-remediation
  title: Repair confirmed epic regressions and complete acceptance verification
  depends_on:
  - controlled-acceptance
  size: medium
  description: 'acceptance-remediation: fix only evidence-backed freshness regressions,
    add deterministic coverage, and reverify the affected acceptance targets.'
proposed_by: bbugyi200.athena.sase-124.8.land
create_time: 2026-09-17 22:34:28
status: wip
bead_id: sase-124.8.4
---

- **PROMPT:** [prompts/202609/complete_agents_freshness_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/complete_agents_freshness_acceptance.md)
- **PARENT:** [202609/finish_agents_freshness.md](https://github.com/sase-org/sase--plans/blob/main/202609/finish_agents_freshness.md)
- **BEAD:** [sase-124.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-124/sase-124.8.4.md)

# Complete Agents freshness acceptance

This is only the acceptance work left after auditing `sase-124.8`; it does not repeat
the capacity-ordering or detached-attention implementation. Read
`plan:202609/finish_agents_freshness.md`, the latest notes on `sase-124.8` and its three
phases, and the parent plan `plan:202609/agents_tab_freshness.md`. Every worker must
read `tui.md`, `tui_perf.md`, and `lint_and_test.md` through `sase memory read` before
capturing or changing the TUI. Long captures, benchmarks, and `just check-full` must run
through `sase_monitor`, never as a blocking inline command. Do not drive or alter the
user's existing interactive TUI session.

The child land agent returns through `parent_bead: sase-124.8`. Closing that parent,
cleaning its Symvision entries, changing either parent plan's status, and closing
`sase-124` are landing duties, not phases in this plan.

## Verified starting point

At `c320b2b6ca`, the landing audit read the current source and the three epic commits:
`f1616c505e` corrected canonical/empty/stale capacity ordering, `9a1d5d67a` detached and
mode-coalesced attention polling, and `76df54778f` supplied wire IDs for no-attempt gate
failures encountered during the acceptance phase. The saved deterministic reproducer
`file:explicit:7149138b09eabe6ff5ba5226` now reports the intended capacity results, no
UI-thread stale-boundary call, attention modes `[true, false]`, and local Agents refresh
before a blocked cache poll releases. The focused capacity/attention suite passed 84
tests, and 52 load-tier, fleet-projection, and active-search integration tests passed.

Concurrent work is already present and must remain intact:

- `7058f16ceb` (`sase-127.1`) preserves visible rosters and panel keys across bounded
  loads and exact tombstones.
- `5b7c4553cc` (`sase-127.3`) skips unchanged fleet reprojections while forcing real
  remote-attention and remote-mutation repaint sources.
- `677ed7d8e4` (`sase-127.4`) guards stable active-search refreshes against full panel
  rebuilds. Its 30-minute athena run is useful attribution evidence, but it did not
  exercise every `sase-124.8` target.
- Gate failures from `1d14218a3c` and `76df54778f` remain in receipt-scoped gate bundles
  and notification flows. `sase-zr.7.3` and `.5` are still in progress; do not invent a
  top-level marker filename or duplicate their exact receipt/pulse routing.
- The local/remote screenshot commands from `729fe7cae1`, `139f6aa263`, and `c320b2b6ca`
  may establish and inspect a dedicated tmux-backed TUI, but a PNG is not performance
  evidence and must not replace trace, perf, stall, or process samples.

The phase-3 note records a busy traced window from 2026-09-17T23:47:11Z through
2026-09-18T00:23:43Z with `refresh.auto_tick` p50/p95/max of 124/3108/3906 ms. Attention
work completed separately (183 cache and 34 network polls, network p95 1199 ms), but the
tick target was missed and not causally attributed. It also records `bench_tui_trace`
passing 5/5, a real-archive load-tiering run over 11,554 artifacts, and six loaded-host
`bench_tui_jk` failures. It explicitly leaves these required items unverified: the idle
ten-minute window, scripted marker latency with and without watcher delivery,
capacity-on-entry latency, tick p95 below 1000 ms, no attention-attributable tick over
two seconds, and no unread/countdown main-thread stall over 500 ms. Missing evidence is
not success.

The sole independent phase proposal is already routed to ready flaky-test task
`sase-129`, which names
`tests/ace/tui/test_loader_cleanup_decoupling.py::test_rows_apply_and_loading_clears_while_cleanup_is_blocked`.
It failed once in the full parallel lane and passed in immediate exact-node and file
reruns. Do not absorb that task into this plan. Its required typed relation to retired
umbrella `sase-ct` was attempted, but the write was refused by the dirty hidden-plans
clone; the independent recurrence was recorded on existing task `sase-10y`.

## Phase: controlled-acceptance

1. Re-read current source and commits since `c320b2b6ca` before capture. Recheck
   `sase-zr.7.3` and `.5`; use a real receipt/pulse or loader-visible marker contract
   only if it has landed. Preserve the load-tier, stable-search, and no-op fleet
   projection behavior listed above. Record commit and linked-core revisions, flags,
   query, archive size, host load, process identity, capture paths, and exact start/end
   times for every sample.
2. Launch a dedicated traced/perf-enabled TUI from this checkout on athena using
   `docs/perf_runbook.md`. Use fresh explicit output paths and prove their timestamps
   and process IDs advance. A new tmux-backed window may be created with the supported
   TUI/screenshot workflow, but never reuse, drive, kill, or reconfigure the user's
   interactive session.
3. Capture at least 30 minutes of ordinary busy-host operation and a separately
   identified ten-minute idle window. For each window report sample counts and
   p50/p95/max for `refresh.auto_tick`, completed attention cache/network work, slow
   loader stages, key-to-paint records, and watchdog hitch/stall rows. Identify sanity
   refreshes, tier revalidations, exact deltas, fleet reprojections, and full-history
   loads rather than grouping them under generic refresh cost.
4. Use isolated scripted fixtures while that dedicated TUI runs to measure the original
   observable contracts: marker/settlement write to visible-row convergence below three
   seconds with watcher delivery and below twenty seconds without it; capacity
   correctness within one second of Agents-tab entry; and unread/countdown UI work with
   no main-thread stall above 500 ms. Exercise both watcher-delivered and polling paths
   without mutating real user gates, notifications, holds, runner limits, or agents.
   Preserve exact receipt/pulse routing and stale-row guards.
5. Build an explicit target matrix. Report each original target as met, missed with
   trace/stack attribution, or still unverified. In particular, determine what produced
   the prior 3108 ms tick p95 and 3906 ms maximum, prove whether any tick above two
   seconds awaited attention work, and distinguish unread/countdown paths from broad
   load, fleet projection, AXE, or host-contention costs. The idle window must state the
   count of slow full loads, with zero as the target.
6. Reproduce and classify the six phase-3 j/k failures. Selected-tribe latency is
   already tracked by `sase-lx`; do not duplicate it. Attribute AXE and Fleet failures
   independently, preserving `sase-127.3`'s genuine-change/forced-source behavior and
   avoiding a common-cause claim without stack or trace evidence. As a phase worker,
   record genuinely independent residuals as `PROPOSED FOLLOW-UP:` notes with exact
   measurements and root-cause evidence; do not create task beads.

## Phase: acceptance-remediation

1. Consume the controlled-acceptance target matrix and raw evidence. Repair every
   confirmed regression caused by `sase-124`/`sase-124.8`, including any path that
   restores attention awaits to `refresh.auto_tick`, performs capacity/hold/config work
   on the UI thread, loses canonical roster ordering, or defeats bounded/exact refresh
   and incremental display behavior. Do not broaden this phase to independent host, AXE,
   Fleet, gate, or generic flake work; record those as precisely attributed proposals on
   this phase.
2. Add deterministic regressions for each source fix. Re-run the capacity and attention
   tests from `sase-124.8`, the `sase-127.1/.3/.4` integration tests, and affected
   receipt/pulse tests. Preserve the existing 60-second attention network cadence,
   completed-only attention counters, exact-delta guards, bounded roster convergence,
   forced remote repaint sources, and stable-search incremental path.
3. Repeat only the live/scripted measurements affected by a fix until every target has a
   current result. If no source fix is warranted, retain the phase-1 measurements and
   explain why each miss belongs to an existing task/epic or is a measured acceptance
   miss rather than silently declaring it green. Missing or stale evidence remains
   unverified.
4. Re-run the affected trace, j/k, and agent-load-tiering benches after any code change;
   otherwise verify and cite the phase-3 current-tree runs instead of manufacturing a
   redundant success. Always record p50/p95/max, sample counts, query, requested limit,
   archive size, and production-bounded versus full-history results separately. Do not
   raise latency budgets.
5. Run `just fix` and the focused suites in the phase. The child land agent must review
   both phase notes, every `PROPOSED FOLLOW-UP:`, post-phase drift, and the final target
   matrix, then run combined `just check-full` through `sase_monitor` before returning
   control to `sase-124.8`. Do not close or edit the parent epic or either parent plan.
