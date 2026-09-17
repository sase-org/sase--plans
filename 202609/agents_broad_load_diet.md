---
tier: tale
title: Cut broad Agents load and post-apply warmup cost
goal:
  Warm unchanged Agents refreshes avoid redundant archive-wide reads and expensive
  overlapping enrichment work while preserving freshness and repair guarantees.
size: medium
proposed_by: bbugyi200.athena.sase-124.4
bead: sase-124.4
create_time: 2026-09-17 12:02:09
status: wip
---

- **PARENT:**
  [202609/agents_tab_freshness.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_freshness.md)
- **BEAD:**
  [sase-124.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-124/sase-124.4.md)

# Cut Broad Agents Load and Post-Apply Warmup Cost

## Outcome

Make an unchanged, warm Agents refresh cheap and predictable on a large archive: avoid
repeated full bead-store reductions and artifact-link projections, prevent deferred
enrichment workers from competing with a foreground Agents load, and bound periodic Tier
1 index revalidation without weakening eventual repair. Preserve stale-while-revalidate
rendering and all existing row, relation, and bead-resolution semantics.

## Baseline and invariants

- Capture the current large-archive timings with
  `tests/perf/bench_agent_load_tiering.py`, the existing detail-header benchmark, and
  focused timing around patch relation-index construction and confirmed-bead warmup.
  Keep the before/after JSON outside the repository and report archive size, candidate
  counts, p50/p95 (where repeated samples are practical), and index counters.
- Use the current profile as the prioritization guard: ordinary cached Tier 1 spends
  most of its time in the Rust index query and filtering; periodic revalidation
  currently signature-checks thousands of hidden rows; confirmed-bead warmup performs a
  resolve plus show store reduction per candidate; and an unchanged patch relation
  projection rescans every aggregate link row. Do not add a broad whole-result Agents
  cache whose invalidation would risk stale rows.
- Keep filesystem and store work off Textual's message pump. Cached values remain
  stale-while-revalidate, explicit watcher/delta events can force prompt refresh, and a
  failed background probe must retain the last successful display state.

## Implementation

1. Add a tolerant batched bead lookup to `sase-core` and expose it through the Python
   binding.
   - In the Rust bead read API, reduce a bead store once, resolve each requested full or
     shorthand ID against that one snapshot, and return only uniquely resolved issues. A
     missing or ambiguous candidate is omitted instead of aborting the rest of the
     batch, matching the passive TUI lookup's existing fail-closed behavior.
   - Export the operation from `sase_core`, register its PyO3 binding, and add Rust
     tests for full IDs, shorthand IDs, duplicates, missing IDs, ambiguity, and
     one-snapshot batch behavior.
   - Add the corresponding Python facade and `BeadProject` query method with
     wire-conversion/delegation tests. Keep shared bead resolution in the Rust core;
     Python should only group contexts and format display strings.

2. Convert post-apply bead enrichment from N+1 store reads to grouped batch reads.
   - Extend the bead lookup/session helper with a batch path that computes each
     candidate's existing ordered workspace/project/local store search path, groups
     unresolved IDs per store, performs at most one batched core read per participating
     store, and preserves first-store-wins precedence.
   - Update confirmed-bead warmup to resolve all unique cache keys through that path and
     populate the existing positive/missing TTL cache without changing glyph or
     deleted-bead behavior. Reuse the same batch facility from family-plan preview
     enrichment where a preview batch requests bead context, so that adjacent post-apply
     lanes cannot independently repeat whole-store reductions.
   - Add focused tests proving duplicate rows and multiple IDs in one store cause one
     core batch call, fallback-store order is preserved, missing/ambiguous IDs do not
     suppress other results, repeated unchanged applies produce no candidates, and
     expired/deleted entries still patch the affected row correctly.

3. Cache immutable artifact-link edge projections and reuse patch relation indexes when
   their inputs are unchanged.
   - Add a small bounded, thread-safe cache around the I/O-free `artifact_link_edges`
     projection. Key only snapshots with a stable `source_key`, plus the compiled
     relation declaration shape, sorted known targets, project hint, and any
     agent-identity snapshot that affects ref resolution; bypass caching for
     anonymous/synthetic snapshots. This makes aggregate mtime/size changes and semantic
     target changes invalidate naturally.
   - Let the patch load path reuse the resulting immutable relation index for an
     equivalent patch semantic signature (project/name/parent and other fields that
     affect graph edges) plus the artifact-link snapshot key, rather than keying reuse
     on the transient list object's identity. Do not retain `Patch` objects in a
     long-lived cache.
   - Cover cache hits, aggregate changes, patch parent/name changes, bounded eviction,
     synthetic snapshot bypass, and unchanged relation output. Extend the perf coverage
     with a large aggregate-link fixture/counter so an unchanged second build
     demonstrates that the aggregate rows are not rescanned.

4. Introduce one quiet-work coordinator for deferred Agents enrichment and give
   foreground loads priority.
   - Centralize the repeated “scheduled/running/pending” checks used by bead
     confirmation, family preview, live hints, and monitor reconciliation into a
     lightweight generation/signature gate. A worker may start only when no full/delta
     Agents load or artifact-index maintenance is active or queued; if foreground work
     arrives first, retain one latest pending request and retry from the normal
     idle/timer path.
   - Key bead/family work by their existing cache candidates. Key live-hint work by the
     resolved active workspace candidates plus a short refresh deadline so consecutive
     unchanged applies collapse while filesystem changes still converge. Gate monitor
     reconciliation by the relevant active monitor signature and a bounded cadence;
     monitor watcher/lifecycle events may invalidate it immediately. Never run monitor
     reconciliation or live VCS probes concurrently with an in-flight Agents disk/index
     load.
   - Preserve the navigation gate and pump-free execution. Add deterministic async tests
     for unchanged apply bursts, a load arriving before a worker starts, a request
     arriving while a worker runs, forced invalidation, expiry, task-spawn failure, and
     a settled monitor's follow-up refresh.

5. Bound periodic Tier 1 hidden-row repair in the Rust index while retaining eventual
   coverage.
   - Change the Tier 1 `revalidate` repair path so it no longer signature-checks every
     excluded hidden row in one query. Persist or derive a deterministic rotation cursor
     and inspect a fixed-size hidden-row batch per pass; always revalidate the
     active/recent rows selected for the current query, and leave full-history
     reconciliation as the authoritative unbounded discovery/removal pass.
   - Ensure the rotation advances even when rows are unchanged, survives process
     restarts where practical, wraps after the last row, and does not starve any hidden
     row. Keep explicit artifact-index mutation hooks and watcher-driven exact updates
     as the fast freshness path.
   - Add Rust tests for the per-pass cap, wraparound/eventual repair of a
     hidden-to-visible change, project filters, deletion, cached-read zero-write
     behavior, and full-history completeness. Surface counters needed by the existing
     benchmark to demonstrate that bounded revalidation checks the selected rows plus
     only the configured repair batch.

6. Verify the detail-header and unwindowed project/Patch sweeps against the new
   contention profile.
   - Run `tests/perf/bench_detail_header_summary.py` and the existing lane/debounce
     tests. The current per-lane TTL, cheapest-first streaming, cancellation, and
     stationary-selection debounce should make repeated warm rebuilds memory-only; add a
     regression test only if the benchmark reveals a repeated resolver call. Avoid a
     second cache when the existing one already proves the requirement.
   - Profile `get_all_project_files`, RUNNING-claim resolution, and
     HOOKS/MENTORS/COMMENTS projection again after background contention is removed. If
     they remain material, add mtime/size-keyed caches at their owning source boundary
     while retaining PID liveness checks and stale-claim cleanup on every load;
     otherwise record their measured cost and leave the simpler path intact.

## Verification and handoff

- Run focused Rust tests for bead batching and incremental index revalidation,
  rebuild/install the local Rust binding as required by the repository workflow, and run
  focused Python tests for the bead facade/project delegation, Agents warmups, monitor
  reconciliation, live hints, patch relation wiring, artifact-link relations,
  detail-header lanes, and Tier 1 reconcile behavior.
- Extend/run the Agents load benchmark on an athena-shaped archive for multiple warm
  samples. Require at least a 50% reduction in warm/no-roster-change broad Tier 1 disk
  time versus the captured baseline, zero repeated bead/family store reductions on an
  unchanged apply, bounded periodic revalidation counters, and cache-hit patch relation
  construction that does not rescan aggregate rows. Confirm normal refresh, watcher
  invalidation, stale-while-revalidate display, and monitor settlement still converge.
- Run the repository-prescribed fast verification (`just check`) after reviewing the
  lint/test memory instructions, plus the `sase-core` checks required for the Rust
  changes. Record any genuinely out-of-scope discovery only as `PROPOSED FOLLOW-UP:`
  notes on `sase-124.4`; do not create beads.
- Before completion, run `sase bead epic-symbols sase-124.4` and resolve every remaining
  symbol or re-key its Justfile ownership to the parent epic or a still-open later
  phase. Then close only `sase-124.4` with a note summarizing the verified timings,
  counters, focused tests, and `just check`; do not close `sase-124` or any ancestor.
