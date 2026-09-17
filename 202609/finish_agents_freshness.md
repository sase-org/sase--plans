---
tier: epic
title: Finish Agents freshness correctness and acceptance
goal: "Cached-roster capacity stays correct across hidden rows, empty rosters, and
  interleaved loads; attention polling never delays local refreshes or loses a requested
  network refresh; fresh measurements establish the integrated sase-124 freshness
  contract.

  "
parent_bead: sase-124
phases:
  - id: capacity-ordering
    title: Correct capacity inputs and asynchronous result ordering
    depends_on: []
    size: medium
    description: "capacity-ordering: use the canonical capacity roster including hidden
      and empty cases, guard all changing inputs, and keep stale-result recomputation
      off the UI thread with deterministic race regressions.

      "
  - id: attention-scheduling
    title: Preserve attention refresh intent without delaying local surfaces
    depends_on:
      - capacity-ordering
    size: medium
    description: "attention-scheduling: detach cache and network attention work from
      local ticks, preserve stronger pending network requests, and record poll duration
      and mode with deterministic scheduling regressions.

      "
  - id: acceptance
    title: Prove freshness on the integrated athena tree
    depends_on:
      - capacity-ordering
      - attention-scheduling
    size: medium
    description:
      "acceptance: integrate intervening refresh changes, capture fresh busy and idle
      session evidence plus scripted marker and capacity latencies, rerun relevant
      benches, and attribute every inherited acceptance proposal."
proposed_by: bbugyi200.athena.sase-124.land
create_time: 2026-09-17 17:43:26
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_agents_freshness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_agents_freshness.md)
- **PARENT:**
  [202609/agents_tab_freshness.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_freshness.md)

# Finish Agents freshness correctness and acceptance

This is the remaining work from the landing audit of sase-124, not a repeat of its seven
completed phases. Read the original accepted plan with
`sase artifact read plan:202609/agents_tab_freshness.md` and the latest notes on
sase-124 before implementation. Every worker must read `tui_perf.md` and
`lint_and_test.md` through `sase memory read`. The parent link returns completion to the
interrupted landing. Parent close, Symvision cleanup, and parent-plan status changes are
not implementation phases in this plan.

## Evidence and limits

The audit at 155aeee2ef reviewed every parent/child note, the six implementation commits
(43ddcf15f5, 14403c1594, 980de1487a, 0af5b08151, 26a43d29f4, 739caf01ff), and
intervening history. Fetched master matched that tree. All 305 focused tests passed, but
deterministic additional checks exposed untested failures. The complete audit and
proposal outcomes are in `file:explicit:12bc47aaa562488f61e61ce6`; read it through
`sase artifact read`.

Read reproduction artifact `file:explicit:7149138b09eabe6ff5ba5226` through
`sase artifact read`. Its Python script uses current production mixins, existing narrow
test harnesses, and mocked I/O ordering. Run it from the repo with the repo root on
Python's import path (for example using `runpy.run_path`). It reproduces:

- Capacity uses the display roster: one displayed runner plus a second hidden canonical
  runner reports one occupied slot instead of two.
- An empty loaded roster skips the tab-entry refresh entirely, retaining an old limit
  even though capacity configuration and holds still matter.
- An older cached-roster worker can overwrite a newer roster's capacity because both
  carry the same limit/hold generation.
- A stale-capacity boundary is recomputed on MainThread, including hold-store work.
- A network refresh requested during a cache-only poll produces fetch modes
  `[true, true]`; the pending boolean loses the required network mode.
- A blocked cache-only attention poll prevents local agents refresh until released.

The saved phase-7 real-archive benchmark was checked: 11,516 artifacts, production
bounded p50/p95 1421/1693 ms, full-history 2232/2561 ms, and unchanged refresh 249/952
ms over five ordinary refresh samples. It used query `not machine:apollo` and
requested_limit 100. These are useful benchmark results, not the missing 30-minute live
acceptance session or marker/capacity timings.

## Phase: capacity-ordering

Relevant code: `actions/agents/_loading_filter.py`, `_loading_apply.py`,
`_loading_compute.py`, `_loading_compute_types.py`, `_loading_disk_full.py`,
`_loading_disk_delta.py`, `event_refresh/_auto_refresh.py`, `_app_watchers.py`, and
state initialization under `src/sase/ace/tui/`.

1. Base cached capacity computation on `_agents_capacity_with_children`, which the
   normal load retains from `prep.capacity_agents` before hiding/fleet projection. An
   explicitly empty canonical roster must be valid; distinguish uninitialized state from
   empty state. Do not count remote display rows as local runners.
2. Snapshot row inputs on the UI thread before handing them to a worker. Guard results
   with revisions covering the canonical roster as well as limits/holds. Bump/invalidate
   at full and exact-delta roster application and relevant local mutations. Recheck the
   current revision after every await. A capacity-only refresh must never restore counts
   from a roster that has since changed.
3. Keep the newer roster from an in-flight load whose capacity inputs are stale, retain
   the last valid capacity display temporarily, and schedule a coalesced recomputation
   with current inputs. Do not call `prepare_loaded_agents_apply_boundary`, hold-store
   reads, config disk reads, or the expensive capacity projection on the UI thread to
   repair that race. Preserve existing query/fold stale-boundary behavior while
   separating the new capacity repair from it.
4. Limit/hold token changes must invalidate and schedule cheap capacity refresh even
   during an in-flight broad load; watcher dirty flags must not short-circuit the needed
   invalidation. Empty occupancy must still refresh the displayed limit. Preserve
   launch-armers, hold expiry and admission semantics from ff08843798 by using existing
   Rust-backed facades, not new Python policy.
5. Add deterministic tests for hidden/local versus remote rosters, empty rosters, both
   stale-result arrival orders, input drift during a load, and rapid coalesced tab
   switches. Assert capacity-related hold/config work is off-thread in the stale-load
   case. Assert the cheap path performs no broad agents load. Keep row mutation and any
   queue presentation consistent with the normal load contract.

No new capacity scan, timer, feature flag, or shared backend policy is needed. If a
shared domain change proves necessary, open sase-core through `sase repo open` and
implement it there with the binding and adapter updated together.

## Phase: attention-scheduling

Relevant code: `actions/agents/_remote_attention.py`,
`actions/event_refresh/_auto_refresh.py`, and `_state_init_agents.py` under the TUI.

1. Put both cache-only fetch/reconciliation and network recomputation in the existing
   pump-free coalesced worker path. The local tick must not await either mode.
   Cache-only still invokes federation supervisor health/start/configuration IPC and
   notification-store work, so it is not guaranteed cheap. Reuse the existing
   notifications dirty/snapshot scheduling path when results change.
2. Preserve request strength while coalescing: a network request arriving during a cache
   poll must cause a subsequent network fetch. A cache request must not downgrade a
   pending network request. Reserve scheduled/running state before spawning, release it
   on spawn failure, exception, and cancellation, and avoid unbounded trailing polls.
   Keep the default 60-second network cadence and the explicit after-answer full refresh
   behavior.
3. Record mode, duration, outcome and coalescing counts without extending the local
   tick. Since the work is detached, associate completed poll measurements with a
   clearly documented completion span and/or latest-completed counters on
   `refresh.auto_tick`; do not label pending work as a completed network poll.
4. Use blocked fake fetches and events to prove local Agents/AXE/notification surfaces
   proceed while either attention mode is blocked. Cover network-during- cache and
   cache-during-network overlap, after-answer refresh, changed inbox convergence,
   cold/error paths, cancellation, and the long recompute cadence. Verify teardown
   through the existing pump-free registry.

## Phase: acceptance

This phase supplies the missing evidence and integration, with fixes limited to
regressions from this epic. Re-read current source and post-audit commits first.

1. Preserve concurrent changes: sase-126's Rust floor and scan/notification hydration
   optimization (02fc83e11a, 1f2d2ff99a), sase-127.2's stable-search incremental display
   (155aeee2ef), and the grouping/layout picker changes (73e4318edf, 5e4c866eb5).
   Recheck sase-127.1/.3/.4 for visible-set and no-op fleet projection changes and
   include their state in attribution. Do not fork their visible-set or projection
   fixes.
2. Recheck sase-zr.7.3/.5. At audit time they were still in progress. Current durable
   failure records live in gate bundles (`journal.jsonl`, `errors/*.json`), not a new
   top-level agent artifact marker. Only extend marker polling if a real loader-visible
   marker contract has landed. Preserve exact receipt/pulse routing and stale-row guards
   when those changes arrive; do not guess a marker filename.
3. Launch a controlled traced TUI session on athena using `docs/perf_runbook.md`. Record
   process identity, commit/core revisions, flags, query, archive size and concurrent
   relevant epic revisions for every capture. Use fresh trace/perf/stall files and
   verify their timestamps advance. Use `sase_monitor` for long captures and benchmarks;
   do not modify or drive the user's existing interactive session.
4. Measure at least 30 minutes of ordinary busy-host operation and an explicitly idle
   ten-minute window. Use isolated scripted marker/settlement fixtures while the TUI
   runs to measure marker-write to visible-row latency with and without watcher
   delivery. Assert the original targets: tick p95 below 1000 ms, no tick over two
   seconds attributable to attention polling, marker latency below three seconds with
   watcher delivery and twenty seconds without it, capacity correct within one second of
   tab entry, no unread/countdown main-thread stall over 500 ms, and zero slow full
   loads during the controlled idle window. Identify periodic sanity/revalidation events
   instead of silently omitting them.
5. Rerun trace, j/k and agent-load-tiering benches; record p50/p95/max and actual sample
   counts. Check the phase-4 unchanged-input warm-load improvement against its baseline
   and warmup skip/coalescing behavior. Separate production viewport results from
   full-history parity and identify improvements due to sase-126/127. Do not raise
   latency budgets or treat an unchanged rerun as proof of a fix.
6. Resolve inherited proposals with evidence: sase-124.1 #1 is existing task sase-zc,
   already fixed in source; sase-124.7 #1 needs controlled idle-load attribution; #2
   needs current stack/revision attribution outside the fixed unread/countdown paths; #3
   selected-tribe is already sase-lx (40 ms budget, inherited numbers recorded there),
   while AXE/Fleet excursions need separate attribution; #4 stale tracing is an evidence
   gap, not a diagnosed tracing bug. sase-zn owns its specific lock-busy
   source-scan/dismissal-parity defects and sase-127 owns no-op fleet repaint. Do not
   assign all similar symptoms to them. Repair confirmed epic-caused regressions. As
   phase workers, record any genuinely independent residual as `PROPOSED FOLLOW-UP:`
   with root cause, measurements and proposing-bead identity for the land agent; do not
   create task beads.
7. Report every original target as met, missed with attributed evidence, or still
   unverified; never turn missing evidence into success. Original-plan permission to
   propose measured independent gaps remains in effect. Run focused regressions and the
   required `just check` after changes. The child land agent runs combined
   `just check-full` through `sase_monitor` after `just fix` and reevaluates readiness.

The completed child must provide enough evidence for sase-124's land agent to recheck
all descendants/notes and new drift before deciding whether the original epic can close
normally.
