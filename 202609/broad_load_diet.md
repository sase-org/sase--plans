---
tier: tale
title: Cut repeated Agents-tab broad-load and warmup work
goal:
  Unchanged warm Tier 1 loads and their follow-on warmups avoid repeated archive-scale
  disk work while preserving refresh correctness.
size: medium
proposed_by: bbugyi200.athena.sase-124.4
bead: sase-124.4
create_time: 2026-09-17 13:31:14
status: wip
---

- **PARENT:**
  [202609/agents_tab_freshness.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_freshness.md)
- **BEAD:**
  [sase-124.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-124/sase-124.4.md)

# Cut repeated Agents-tab broad-load and warmup work

## Objective

Make an unchanged, warm Tier 1 Agents-tab broad load cheap on an Athena-sized archive
and prevent consecutive load applies from restarting disk-heavy warmups. Preserve row
freshness by invalidating on authoritative input changes and by leaving explicit index
revalidation uncached.

## Evidence and constraints

- The approved epic design is `plan:202609/agents_tab_freshness.md`, phase
  `broad-load-diet`.
- A current real-archive baseline over 11,486 artifact directories measured a production
  bounded load at 1.48 s and settled ordinary refreshes at roughly 0.90-1.00 s. A warm
  profile attributes about 0.67 s to the Rust index query plus wire conversion, 0.22 s
  to query filtering, and 0.15 s to source projection/normalization.
- Current trace data still shows `agents.load_from_disk` at 1.2-2.4 s under contention.
  `agents.bead_confirmation_warmup` reaches 7-22 s for 11-14 candidates because a lookup
  session opens stores once but still performs a separate Rust store read for each
  candidate. Live-hint refreshes are about 0.5-1.2 s and monitor reconcile about 0.2-0.3
  s after many applies.
- The existing Patch snapshot cache already keys parsed project files by
  `(path, mtime_ns, size)`, and the current detail-header lane cache reduces warm
  rebuilds to tens of milliseconds in recent traces. Do not replace these working
  mechanisms or add another refresh path.
- Keep all filesystem and VCS work off Textual's event loop/message pump. Explicit
  `freshness="revalidate"` loads must still inspect source state, and artifact/index
  mutations must invalidate any reused snapshot before it can be served.

## Implementation

1. Add a small process-local cache at the TUI artifact-snapshot boundary in
   `src/sase/ace/tui/models/_agent_loader_artifacts.py` (with a focused helper module if
   that keeps the contract clearer). Key it by the complete bounded query inputs and by
   a stable signature of the SQLite database and WAL. Cache only `freshness="cached"`
   index reads, compare the signature before and after the query before publishing an
   entry, bound the cache, and expose a clear hook for tests. Explicit revalidation,
   source-scan fallbacks, and artifact deltas bypass this cache. Reuse the immutable
   scanner-shaped snapshot so each load still rebuilds mutable `Agent` rows and applies
   current project, Patch, status, dismissal, and query projections.

2. Remove the remaining avoidable unwindowed project work without caching liveness
   decisions. Cache parsed RUNNING-claim snapshots per project spec using
   `(path, mtime_ns, size)`, but perform PID liveness checks and stale claim cleanup on
   every load. Invalidate the entry after cleanup mutates its project file. Continue
   using the established Patch snapshot cache; do not add a second Patch parser cache
   for the already-cheap HOOKS/MENTORS/COMMENTS projection loop unless the new benchmark
   demonstrates a material residual cost.

3. Make bead-confirmation warmup one store read per store, not one store replay per
   candidate. Extend `BeadIssueLookupSession` to materialize and index one issue
   snapshot per opened store for batched sessions, preserving canonical full-ID and
   unambiguous suffix lookup semantics. Keep unavailable-store and missing-bead behavior
   fail-closed. Make the TUI scheduler decline to spawn a warmup task when its
   memory-only candidate scan is empty, while rechecking candidates in the worker to
   handle races.

4. Suppress redundant post-apply work and disk contention. Give live-hint and
   monitor-reconcile scheduling a bounded freshness/input guard so bursts and unchanged
   consecutive applies do not rerun identical work; roster changes still trigger an
   immediate pass, and monitor liveness retains a periodic backstop. At worker entry,
   defer while an agents load is in flight and keep the existing navigation and
   coalescing guards intact. Do not weaken event-driven refreshes or allow an older
   worker result to overwrite a newer roster.

5. Extend the existing agent-load tiering performance harness under `tests/perf/` with
   an Athena-shaped, settled repeated-broad-load scenario. Report first-load versus
   unchanged-refresh timing plus cache hit/miss and revalidation counters. Keep
   correctness assertions count/signature based; use measured timing for the documented
   performance comparison rather than a flaky CI wall-clock floor.

## Tests and verification

- Unit-test snapshot cache hit, database/WAL invalidation, query-key isolation,
  mutation-during-read refusal, bounded eviction, clear-hook behavior, and unconditional
  bypass for revalidation/fallback paths.
- Unit-test RUNNING-claim cache hits and mtime invalidation while proving PID death is
  rechecked and stale cleanup invalidates the cache.
- Unit-test one bead-store list/read for many candidates, cross-store grouping, full-ID
  and suffix resolution, missing IDs, and no task spawn after an unchanged apply with
  fresh bead caches.
- Extend live-hint and monitor-reconcile tests for burst suppression, changed roster
  inputs, periodic monitor fallback, load-in-flight deferral, and last-current-roster
  application.
- Run the focused unit suites and performance smoke tests, then the Athena-scale
  benchmark before and after. The unchanged warm broad-load result must improve by at
  least 50% relative to the captured baseline without changing visible rows; repeated
  unchanged applies must record no bead warmup and no redundant live-hint/monitor work.
- Run `just check`. Before closing `sase-124.4`, inspect
  `sase bead epic-symbols sase-124.4`, resolve or re-key every remaining symbol, record
  the incremental-revalidation decision and measured before/after data on this phase
  bead, and close only this phase with the required verification note.

## Out of scope

- Hidden-row index pruning (`sase-kh`), changing the five-second full-load floor, the
  parent epic's other refresh phases, and a new feature flag.
- A new incremental source-discovery algorithm unless the measured explicit Tier 1
  revalidation remains a dominant regression after the safe cached-path and warmup
  fixes. Record that evaluation on the phase bead instead of expanding this tale without
  evidence.
