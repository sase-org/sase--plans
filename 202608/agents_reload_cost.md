---
tier: tale
title: Reduce full Agents-tab reload cost
goal:
  Routine Tier-1 Agents-tab reloads consume the read-only artifact-index projection without source-tree filesystem
  probes or writer-lock contention while exact deltas and Tier-2 reconciliation preserve freshness.
proposed_by: bbugyi200.athena.sase-e4.6
bead: sase-e4.6
create_time: 2026-08-02 09:31:25
status: wip
---

- **PROMPT:**
  [prompts/202608/agents_reload_cost.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agents_reload_cost.md)
- **PARENT:**
  [202608/tui_freeze_render_path_git.md](https://github.com/sase-org/sase--plans/blob/main/202608/tui_freeze_render_path_git.md)
- **BEAD:** [sase-e4.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e4/sase-e4.6.md)

# Reduce full Agents-tab reload cost

## Goal

Make the routine Tier-1 Agents-tab reload consume the already-maintained SQLite projection without walking agent
artifact directories, so post-launch, starting-poll, notification, tab-switch, and auto-refresh updates complete in well
under the current multi-second disk stage. Preserve exact artifact-delta updates, full-history reconciliation, dismissed
visibility, clan context, and the existing strict self-healing artifact-index API.

## Evidence and diagnosis

The approved parent design records 3,049 slow-load events with disk-stage p50 2.62 s, p95 9.45 s, and max 166.7 s. The
current log now contains 3,056 events and still shows ordinary startup/launch/auto-refresh loads around 2.1–3.0 s, with
recent outliers of 8.1–16.7 s. Prep and apply remain tens of milliseconds, confirming that the disk stage is the target.

The reporter's current index is 80,367,616 bytes and contains 5,695 artifact rows plus 21,351 dismissed identities. The
Tier-1 query asks for at most 1,000 active and 200 recent completed rows. In the Rust implementation, however,
`query_agent_artifact_index()` does more than query the projection:

1. `repair_stale_rows_for_query()` selects every `hidden = 1` row before applying the visible filter and calls
   `MarkerSignatures::from_artifact_dir()` for each one.
2. `select_records()` calls the same signature collector for every selected active and completed row.
3. A changed signature invokes `scan_agent_artifact_dir()` and an index upsert, so this nominal read path can take
   SQLite write locks as well as filesystem I/O.

That explains the measured ~14.3k `lstat` calls per refresh and why the cost scales with historical artifacts rather
than the 130–180 rows rendered by ACE. The Python facade additionally serializes the query behind the same unbounded
process-local `RLock` used by index rebuild and dismissed-projection maintenance, so a background maintenance pass can
delay a routine read even though SQLite WAL supports concurrent readers.

The large dismissed set is not a meaningful hot-path target: copying the current 20,720-entry JSON-derived set takes
about 0.66 ms on this machine. Keep its existing snapshot semantics rather than complicating ownership for a
sub-millisecond saving.

## Existing correctness machinery to preserve and rely on

- Artifact mutations are projected through `update_agent_artifact_index_for_marker_mutation()`/upsert/delete helpers.
  The fail-closed AST audit in `tests/test_agent_artifact_marker_mutation_audit.py` guards this contract.
- Filesystem watcher events are mapped to exact artifact directories and consumed through
  `_load_agent_artifact_delta_async()`; launch results also schedule exact artifact deltas.
- ProjectSpec `RUNNING` claims are loaded separately on every broad refresh, so a just-launched agent is visible even
  while its artifact projection is converging.
- A deliberate Tier-2/full-history refresh still scans source artifacts and is the strict reconciliation path.
- The existing `query_agent_artifact_index()` has non-TUI callers and tested self-healing behavior. Do not weaken or
  silently redefine it.

## Design

Add a second, explicit Rust-core query operation for consumers that want the current materialized projection. It will
open the existing SQLite database read-only and deserialize the rows selected by the same active/recent/full-history,
hidden, dismissed, project-filter, ordering, and clan-context SQL as the strict query. It will not stat artifact
directories, rescan marker JSON, upsert rows, migrate schema, create directories, or acquire the Python process-local
writer lock.

The semantic split must be unmistakable in names and docstrings:

- `query_agent_artifact_index()` remains the strict, source-revalidating query.
- `query_agent_artifact_index_projected()` is the fast read-only materialized-view query.

Only the ordinary ACE Tier-1 loader switches to the projected operation. Exact-delta ingestion remains responsible for
freshness during normal TUI operation; full-history loads remain source-revalidating. Other CLI, verification, repair,
and lifecycle callers keep the strict operation by default.

This belongs in `sase-core` because artifact-index query semantics are shared backend behavior. The Python work is a
thin facade and TUI policy selection, consistent with the Rust core boundary.

## Implementation

### 1. Add the projected query to `sase-core`

In `crates/sase_core/src/agent_scan/index.rs`:

- Refactor the common query body to accept an already-open `rusqlite::Connection` and an internal revalidation policy.
  The strict entrypoint opens through today's migration-capable `open_index()`, runs `repair_stale_rows_for_query()`,
  and lets `select_records()` compare source signatures exactly as it does now.
- Add `query_agent_artifact_index_projected()`. Open with SQLite read-only flags, install a short busy timeout, and run
  the common query body with source revalidation disabled. In that mode, `select_records()` deserializes stored
  `record_json` directly and performs only database-backed filtering/context resolution.
- Keep SQL selection, deduplication, sort order, dismissed matching, project lifecycle filters, and clan-context
  resolution shared between modes. A projected query must observe one atomic WAL snapshot if dismissed maintenance is
  concurrently replacing rows.
- Do not bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`: this adds an operation but changes no persisted columns or JSON
  projection shape.
- Export the operation through `agent_scan/mod.rs` and `lib.rs`.

In `crates/sase_core_py/src/lib.rs`:

- Add the matching `query_agent_artifact_index_projected` PyO3 function with the same query/options arguments and
  scanner-shaped return value as the strict binding.
- Continue to release the GIL around the Rust call.

### 2. Expose the read-only facade and select it for ACE Tier 1

In `src/sase/core/agent_scan_facade.py`:

- Add `query_agent_artifact_index_projected()` using the existing query/options wire converters.
- Deliberately do not enter `agent_artifact_index_operation_lock()`: the core entrypoint is read-only and WAL-safe, and
  bypassing the writer serialization is what prevents a maintenance pass from turning a refresh into a long wait.
- Preserve normal binding/error propagation so the loader's existing bounded source-scan fallback handles a missing,
  stale, corrupt, or briefly unavailable index.

In `src/sase/ace/tui/models/agent_loader.py` and `_agent_loader_artifacts.py`:

- Route `_query_artifact_index_for_loader()` to the projected facade for non-full-history Tier 1.
- Keep full-history behavior unchanged: it never queries the index and scans source artifacts.
- Keep the existing fallback and `AgentLoadState` repair metadata. Add a small trace/load-state field only if needed to
  make `projected` versus `strict` visible in existing `agents.load_from_disk` telemetry; do not create another refresh
  path.
- Document that normal freshness comes from exact artifact-delta/lifecycle projection updates and that the Tier-2 path
  remains the source-of-truth reconcile.

### 3. Lock correctness and compatibility with tests

Rust coverage in `crates/sase_core/src/agent_scan/index.rs` and/or `crates/sase_core/tests/agent_scan_parity.rs`:

- The projected query returns the same records, ordering, dismissed filtering, project filters, and clan context as a
  strict query when the index is current.
- After mutating a source marker without updating the index, the projected query returns the stored projection while the
  strict query self-heals and returns the mutation. This pins the API distinction instead of relying on timing.
- Removing or making source artifact directories unreadable after indexing does not make the projected query touch or
  rewrite them.
- The read-only operation works concurrently with a writer transaction and never attempts DDL/upserts.
- Existing strict-query self-heal tests remain unchanged and passing.

Python coverage:

- Extend facade tests to pin the projected binding's query/options dictionary and scanner conversion.
- Hold `agent_artifact_index_operation_lock()` in one thread and prove the projected facade enters its fake Rust binding
  immediately, while the existing strict-query serialization test continues to prove strict calls wait.
- Extend the Tier-1 loader contract test to assert it calls the projected facade with active limit 1,000, recent limit
  200, and hidden rows excluded; assert the strict facade/source scan are not called on the healthy path.
- Preserve artifact-delta, fallback, startup schema-bypass, full-history, dismissed visibility, and marker-mutation
  audit tests.

## Validation

1. Before measuring, run `just install` so the local Python environment contains the modified linked `sase-core`
   binding.
2. Run focused Rust tests for `sase_core` and `sase_core_py`, then focused Python facade/loader/delta/startup tests.
3. Run the required `just check` from the primary `sase` workspace; it rebuilds/validates the linked core as needed.
4. Benchmark the same current production index before and after with the Tier-1 query shape. Capture wall time and
   syscall counts (`strace -c` or equivalent). Acceptance criteria:
   - projected-query source-file `stat`/`lstat` count is zero (SQLite's own file metadata calls are excluded);
   - routine Tier-1 disk stage is below 1 second on the reporter's machine and no longer scales with hidden historical
     rows;
   - the strict query still detects an intentionally unprojected marker mutation;
   - the projected query completes while the process-local writer lock is held;
   - exact artifact delta and full-history test paths remain green.
5. Review both repository diffs, rerun the focused performance probe after the full check, and verify no generated
   version fields or memory/instruction files changed.

## Non-goals

- Do not prune the dismissed archive or change dismissed identity semantics in this phase.
- Do not cache mutable `Agent` objects or add a third UI refresh scheduler.
- Do not weaken the strict artifact-index query used by verification, repair, or non-TUI callers.
- Do not alter Tier-2/full-history completeness behavior.
