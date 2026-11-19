---
tier: epic
title: Finish measured Agents interaction latency recovery
goal: Make selection, detail enrichment, and small roster updates responsive without
  rebuilding or retaining unrelated agent data.
parent_bead: sase-1t3.13
phases:
- id: baseline
  title: Measure the current interaction paths
  size: medium
  depends_on: []
  description: 'baseline: capture comparable post-heap j/k, detail-selection, unread-jump,
    runtime-tick, exact-delta-copy, and bead-warmup measurements; record the environment
    and identify which existing optimizations already meet the scope.'
- id: selection
  title: Bound detail enrichment and preserve immediate selection
  size: medium
  depends_on:
  - baseline
  description: 'selection: preserve the existing immediate chrome and streamed lanes,
    bound selection-driven enrichment, reuse stable per-agent lane content identities,
    and verify j/k and comma-j remain highlight plus debounced-detail operations.'
- id: visible-work
  title: Limit periodic work to surfaced rows and relevant waiters
  size: medium
  depends_on:
  - baseline
  description: 'visible-work: share a viewport-aware row selection between runtime
    ticks and bead warmup, retain wait-status correctness, and make stranded sweeps
    lazy; preserve the existing Rust interval-union semantics and cache bounds.'
- id: delta-copy
  title: Detach only rows affected by exact artifact deltas
  size: medium
  depends_on:
  - baseline
  description: 'delta-copy: introduce a conservative exact-delta copy-on-write preparation
    path that owns every mutated row and relationship while sharing untouched presentation
    rows; preserve full-load and bounded-history semantics.'
- id: integration
  title: Verify combined latency and prepare parent-phase completion
  size: medium
  depends_on:
  - selection
  - visible-work
  - delta-copy
  description: 'integration: repeat the baseline workload on the combined changes,
    verify responsiveness and graph isolation under interleaved refreshes, record
    evidence on sase-1t3.13, and prepare completion for this child epic''s land agent.'
proposed_by: bbugyi200.athena.sase-1t3.13
proposed_with: codex/gpt-6-astra
create_time: 2026-11-18 15:37:10
status: done
bead_id: sase-1t3.13.1
---

- **PROMPT:** [prompts/202611/interaction_latency.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/interaction_latency.md)
- **PARENT:** [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:** [sase-1t3.13.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.13.1.md)

# Finish measured Agents interaction latency recovery

## Scope and tier

This is a child epic for `sase-1t3.13`, the `interaction-pass` phase of
`plan:202611/tui_storm_trust_heap_recovery.md`. Read that artifact with
`sase artifact read` and read the assigned bead before implementation. The phase has no
earlier-work notes as of this investigation. Its dependencies `sase-1t3.1`,
`sase-1t3.10`, and `sase-1t3.11` are closed.

An epic is appropriate because safe graph ownership, selection-worker lifetime, and
viewport-driven refresh work are distinct changes with separate regression surfaces.
Each phase below is bounded direct implementation work. The baseline phase prevents
rebuilding optimizations that earlier epics already delivered; the three implementation
phases can proceed independently after it.

The outer epic's final decisions remain binding: `hotfix_reinstall=yes`,
`trust_ack_compat=cutover`, and `tui_perf_memory=no`. The first two concern other
phases; this plan introduces no reinstall or trust work. **No memory files may be
edited.** Do not re-ask these decisions.

## Evidence already in the checkout

Investigation used sase HEAD `5b08952c6d`; recheck the current checkout before
implementing because other phases continue to land.

- `widgets/prompt_panel/_agent_display_async_agent.py` already launches header work with
  Textual `run_worker(thread=True)`. `_agent_display_async.py` streams lane batches
  through `call_from_thread`. This is already asynchronous, but canceling a worker on
  each selection change does not prove its underlying thread has stopped. The plan must
  test actual outstanding computations.
- `_agent_display_header_summary_cache.py` already has a bounded per-agent summary cache
  and lane refresh intervals. Its `detail_header_summary_cache_key` still computes
  `repr(summary)`, encodes the whole summary, and hashes it on each key lookup.
- `actions/agents/_unread_navigation.py` and
  `tests/ace/tui/test_agent_unread_jump_fast_path.py` already implement and test
  immediate selection, deferred detail, and target-panel-only expansion. Preserve this
  behavior instead of replacing the navigation pipeline.
- `models/_agent_time_aggregate.py` already caches member wires, interval results, and
  clan fast hits. Shared arithmetic delegates to Rust through
  `sase.core.agent_runtime_facade`. `AgentList.patch_active_runtime_rows` nevertheless
  loops over its whole `_agents` collection. `visible_agents()` returns rendered tree
  entries, not necessarily the clipped scroll viewport.
- `actions/agents/_loading_graph.py` avoids unnecessary graph copies on authoritative
  replacement loads, but classifies exact artifact deltas as requiring the cached
  roster. `own_required_apply_snapshot` then calls `copy_agent_graph` on that roster.
  `_loading_compute_merge.py` performs relationship normalization after merging, so
  deleting the copy would race with live UI rendering.
- `_loading_bead_warmup.py` already coalesces work off the pump and applies patches by
  identity. Its candidate collection still walks `self._agents`, and stranded-project
  collection walks `_agents_with_children`. Scheduling probes the candidates before the
  coalescing checks and the worker probes them again. This is not a viewport-bounded
  workload.

All source paths above are relative to `src/sase/ace/tui/` unless stated otherwise.
Existing behavior is evidence of landed work, not evidence that the latency targets have
been achieved; this planning turn measured no timings.

## Invariants for every phase

- Read `ace.md`, `tui.md`, and `tui_perf.md` through `/sase_memory_read` before TUI
  changes. Follow pump-free scheduling, last-request-wins application, navigation gates,
  and incremental row patching. Timers only schedule work.
- No disk reads, stat/glob, subprocesses, archive queries, large summary serialization,
  or blocking awaits on a keystroke/render path. Preserve the existing 150 ms detail
  debounce and immediate highlight.
- Cache expensive lane results per agent with source-version invalidation obtained
  off-thread. Use existing targeted query APIs from the prior phase; do not recreate
  full-file log caches or retain multiple roster generations.
- Capture row inputs before background computation and recheck selected identity,
  attempt view, tab, and request validity after awaits. Bound pending work and retained
  results. A changed roster generation alone must not throw away still-valid content for
  an unchanged agent.
- Keep Rust backend semantics authoritative. This plan primarily changes Python
  presentation scheduling and mutable row ownership. Any necessary shared query or
  aggregation change belongs in `sase-core`, opened with `/sase_repo`; read its
  `AGENTS.md`, update bindings/tests, and advance `sase-core-revision.txt` beyond the
  landed core commit without moving it back.
- Do not add threads to hide latency, change GC thresholds, delete archives, restart the
  live TUI, or run global installation. A bounded executor may replace uncontrolled
  selection-worker submissions; it is not extra capacity.
- Each phase records measurements, environment, changed paths, and verification in its
  bead notes. Record unrelated discoveries only as `PROPOSED FOLLOW-UP:` notes, not new
  beads. A check failure proven identical on the clean base is recorded with its
  evidence and does not strand a completed phase.

## Phase baseline

Use the current checkout and a separate TUI instance or the existing headless harness on
a quiet host. Record Python, Textual, sase, and `sase_core_rs` versions/import locations
and the core revision, along with host load, roster size, viewport size, and cold/warm
state. Match the installed TUI environment before treating measurements as
production-comparable. Do not infer live-code versions from the on-disk checkout of an
already-running TUI.

Enable `SASE_TUI_PERF=1` and `SASE_TUI_TRACE=1`. Reuse:

- `tests/ace/tui/bench_tui_jk.py`, especially the Agents, unread, and keypath cases;
- `tests/perf/bench_tui_trace.py` and `bench_detail_header_summary.py`;
- `tui_jk.jsonl`, trace spans, agent-load prep timings, and memory heartbeats.

Capture p50/p95/max and sample counts for j/k paint, comma-j visible and
collapsed-target branches, selection-to-first-detail-chrome, and cold/warm header lane
completion. Measure selection chrome separately from intentional 150 ms debounce and
final lane completion; never label worker duration as key-to-paint or count debounced
completion as first paint.

For deltas, use 2- and 8-row changes against representative 150- and 600-row graphs,
including clan/session edges and overlapping capacity rows. Count copied Agent objects
and identify which preparation passes mutate them. For warmup and ticks, keep the
viewport fixed while increasing hidden/offscreen rows and count lookups, aggregate
calls, and rendered row work. Include idle, scrolling, collapsed panels, WAITING rows,
and selection changes during a slow enrichment. Add only the focused counters/fixtures
missing from existing harnesses; no new general telemetry subsystem.

Publish reusable commands, fixture parameters, and retained evidence in the phase note
and on `sase-1t3.13`. Mark each lane `remaining` or `already satisfied` with evidence.
Later phases may finish with verification only when the target cost is demonstrably
gone. Use `/sase_monitor` for commands that outlast the turn.

## Phase selection

Own the prompt-panel header cache/async modules, selection display glue, and their
tests. Avoid graph-preparation and runtime/warmup files.

1. Preserve immediate cheap chrome, cached lane display, and cheapest-first streamed
   results. Verify artifact reads, memory reads, hosted page URL, and skill uses use
   targeted per-agent queries with version/mtime invalidation off-thread. Empty/missing
   results must also be cached and revalidated.
2. If the stress fixture confirms overlapping canceled computations, replace the header
   submission mechanism with one app-owned bounded worker, one running compute and one
   replaceable pending selection. Reuse the storm worker's lifecycle pattern without
   sharing its executor or decision queue. Cancellation suppresses obsolete display and
   skips unneeded later batches; it must not launch another thread while synchronous
   work is still running. Cache a completed valid lane even when selection moved away.
   Teardown cancels pending work and prevents late widget updates.
3. Store a semantic content token/digest when lane data is resolved or merged, not
   during each hint-render cache lookup. Equal content keeps the token; genuine changes
   to any displayed field or lane readiness change it. Do not replace semantic equality
   with transient dataclass identity. Cache hits skip both large summary hashing and
   renderable construction.
4. Exercise comma-j's visible, collapsed clan/tribe, stopped/stranded fallback, and miss
   branches. Preserve acknowledgment semantics and use only the minimum structural
   reveal when needed; unrelated panels never rebuild. Fix only regressions the new
   measurements or tests identify.

Extend the existing header enrichment/lane tests, `test_agent_detail_two_phase`,
`test_agent_display_defer_detail`, and unread fast-path suites. Use a blocked fake
resolver and rapid A/B/A selection to prove bounded compute count, unchanged-content
reuse, source drift refresh, error recovery, and no stale paint. A large summary must
not be serialized on repeated hot cache reads.

## Phase visible-work

Own `_loading_bead_warmup.py`, runtime tick scheduling, AgentList viewport selection,
and related tests. Leave detail worker/cache and graph-copy files to their phases.

1. Expose a cheap row selection based on actual visible option geometry for each active,
   non-retiring panel, skipping banners and deduplicating agents. Account for multiline
   options, scrolling, folding, rail mode, and resize. Do not silently redefine
   `visible_agents()` for unrelated consumers.
2. Use that selection for time-driven row patches. Existing runtime-wire and interval
   caches remain in place. An unchanged offscreen clan must not trigger an aggregate
   call every tick; entering the viewport or expanding a panel produces a catch-up
   render. Membership/status changes still invalidate the relevant result. Preserve
   gate/monitor exclusions and interval-union arithmetic; call the existing Rust facade.
3. Warm bead display for surfaced rows and the selected detail. Maintain a separately
   updated set of WAITING dependency/follow-progress candidates so hidden waiters that
   affect visible state remain correct, without scanning every historical row on each
   schedule request. Batch and deduplicate work, honor existing TTLs, and coalesce
   before constructing expensive candidates.
4. Make automatic stranded-project work lazy: prioritize surfaced projects, deduplicate,
   use the existing report cadence and bounded batches. Preserve explicit
   `stranded_resolve` behavior and the stopped/stranded comma-j fallback. Background
   discovery must eventually converge; skipping hidden work forever is not an acceptable
   speedup.
5. After an await, recheck activity and current row identity; patch only changed rows
   and update the selected detail where needed. Navigation and typing gates apply to
   scheduled work and application. Do not turn off-tab warmup into a busy timer loop.

Extend `test_agents_bead_warmup.py`, runtime tick/cache tests, and relevant
stranded/navigation tests. Hold viewport and relevant waiter count constant while
growing unrelated rows and assert bounded resolver/aggregate calls. Test scroll
catch-up, missing-result expiry, bead changes, deduplication, worker failure, pending
requests, removed rows, and hidden-tab behavior.

## Phase delta-copy

Own `_loading_graph.py`, `_loading_compute*.py`, `_loading_disk_delta.py`,
`models/_agent_graph.py`, and graph/merge ownership tests.

1. Add a conservative exact-artifact-delta fast path before full cached-graph
   detachment. Reuse known exact identities/deletions and the current roster lookup
   structures. Distinguish scalar updates from relationship/membership, dedup, capacity,
   query, or ordering changes. Full loads and uncertain or broad changes retain the
   existing safe path.
2. Determine the affected relationship closure for old and new rows: parents,
   session/clan containers, retry peers, wait/status display references, and
   capacity/fleet aliases. Copy each affected Agent once through a shared memo; detach
   each mutable field actually written. Relink owned references to the copies. Unrelated
   components remain shared and read-only. Preserve cycles and alias consistency across
   visible/capacity rosters.
3. Do not call whole-roster mutating relationship normalization on shared live rows.
   Normalize only the owned affected component on the fast path and splice the result
   into a new roster sequence. Audit slot annotation, named-proc carryover, fold/fleet
   projection, and finalization as well as the merge: a later writer must own its target
   too. Broad changes may fall back before mutation, with an explicit trace reason.
4. Retain only current-generation lookup/ownership state. Do not introduce a second
   historical graph cache. Stale or canceled worker results cannot mutate the live
   graph; selection is recaptured by the existing apply path.

Test equivalence to the existing full-detach preparation for scalar updates, terminal
transitions, child add/remove/reparent, deletions, overlapping capacity rows, same-PID
dedup, named procs, fleet rows, and incomplete history. Snapshot live graph
relationships and mutable leaf values before worker work; prove they remain unchanged
until apply, including cancellation and exceptions. For unrelated components, prove
unchanged identity. Count allocations for the baseline 2/8-row deltas: copying must
scale with the affected closure, not the full roster. A full-detach fallback is
acceptable for genuinely broad or unsupported structural changes, never as the outcome
of every representative small delta. Record fallbacks and their reasons in measurements.

## Phase integration and land responsibilities

Run the same fixture sizes, environment, and sample counts from baseline on the combined
implementation. Exercise navigation while a detail lane is slow, a small delta applies,
and bead warmup completes. Record before/after p50/p95/max, copy counts, resolver
counts, queue bounds, and memory behavior. Targets remain j/k p95 toward 16 ms and
selection-to-first-detail-chrome below 100 ms. Report every missed target with
attribution; do not hide it behind a fast synthetic subcase. Confirm eventual
lane/wait/stranded correctness alongside speed.

Each phase that changes tracked files reads `lint_and_test.md` and runs focused tests
plus `sase tool run check` (the required `just check` wrapper). Use `/sase_monitor` when
needed, and run formatting first. Do not initiate `just check-full`. If rendering or
screenshot coverage changes, read `tui_screenshot.md`, run the relevant snapshot
workflow, and inspect changes. Do not rewrite memory or the outer soak phase's
documentation as part of this child epic. The separate one-hour soak remains
`sase-1t3.14`'s responsibility.

The integration phase records evidence on its own bead and `sase-1t3.13` and closes only
its assigned phase. It must not close any ancestor. Before any phase closes, run
`sase bead epic-symbols <assigned-id>` and resolve or re-key every leftover entry to a
still-open owner.

This child epic's **land agent** reviews all phase evidence, finishes its own child-epic
closure obligations, then runs `sase bead epic-symbols sase-1t3.13` and closes **only
the delegated parent phase** with
`sase bead close sase-1t3.13 --note "<verified behavior and measurements>"`. It must not
close `sase-1t3` or any other ancestor. Record clean-base check failures and remaining
out-of-scope target misses as `PROPOSED FOLLOW-UP:` notes with evidence; proven baseline
failures alone do not prevent closure.
