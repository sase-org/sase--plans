---
tier: epic
title: Make Artifacts sub-tab queries fast, starting with the Agent pane
goal: "Every Artifacts sub-tab paints its default `limit:100` view in well under half a
  second on a live-scale corpus, and the corpus work that only a wider query needs
  happens after first paint instead of before it.

  "
phases:
  - id: bench
    title: Honest first-paint benchmarks for the Artifacts panes
    depends_on: []
    size: small
    description: "bench: build a pane-level first-paint benchmark over a live-scale
      synthetic corpus and repair the agent-catalog bench, whose fixture structurally
      omits the two costs that actually dominate the real registry load.

      "
  - id: registry
    title: Stop revalidating the agent-name registry on every load
    depends_on:
      - bench
    size: medium
    description: "registry: make the process-cached registry stop paying a full
      source-signature stat sweep and a full per-entry owner-existence sweep on every
      `load_name_registry()` call, which is 905ms of the Agent pane's 1529ms load.

      "
  - id: agent-paint
    title: Two-stage Agent pane load
    depends_on:
      - bench
    size: medium
    description: "agent-paint: paint the Agent pane from a bounded head slice and move
      the full-corpus query-index build into a background extension worker, using the
      shipped Files-pane two-stage pattern.

      "
  - id: core-corpus
    title: Direct dict-to-QueryRow corpus construction in sase-core
    depends_on:
      - bench
    size: medium
    description: "core-corpus: remove the serde_json::Value intermediate from
      `compile_corpus_with_profile` in the sase-core repo so corpus indexing stops
      materializing every row twice on the Rust side, then release and raise the floor.

      "
  - id: entry-projection
    title: Cut the Python-side corpus marshalling cost
    depends_on:
      - bench
    size: medium
    description: "entry-projection: stop materializing each query row three times on the
      Python side and shrink the per-row projection work in the agent catalog's
      query-entry adapter, without changing the row wire shape.

      "
  - id: plans
    title: Defer plan metadata reads past the inventory slice
    depends_on:
      - bench
    size: medium
    description: "plans: stop reading and YAML-parsing every archived plan file to
      produce fifty rows, and stop computing the rejected section for callers that only
      asked for proposed plans.

      "
  - id: beads
    title: Take the external-issue network call off the Bead first-paint path
    depends_on:
      - bench
    size: medium
    description: "beads: move the in-band `gh` subprocess out of the Bead snapshot
      worker into a background refresh, and remove the repeated project-alias and
      external-ref work in the same load.

      "
  - id: verify
    title: End-to-end verification and the perf recipe
    depends_on:
      - registry
      - agent-paint
      - core-corpus
      - entry-projection
      - plans
      - beads
    size: small
    description:
      "verify: confirm the combined first-paint numbers against a live-scale corpus,
      record the recipe in the perf README, and land the epic's combined tree green."
proposed_by: bbugyi200.athena.0do
status: done
bead_id: sase-tt
create_time: 2026-09-09 19:49:59
---

- **PROMPT:**
  [prompts/202608/artifacts_query_performance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_query_performance.md)
- **BEAD:**
  [sase-tt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-tt/README.md)

# Plan: Make Artifacts sub-tab queries fast, starting with the Agent pane

## 1. Problem

Every Artifacts sub-tab defaults to a `limit:100` query, but no pane except Stitch lets
that cap reach the work that produces the rows. Each pane loads its entire corpus,
builds an entire query index over it, and only then slices to 100 for display. On a
live-scale corpus the user waits seconds for a hundred rows.

### 1.1 Measured baseline

Timings below are steady-state (warm process, warm page cache) against a live corpus of
12,667 registry names, 8,099 artifact-index rows, 6,868 top-level dismissed bundles,
1,950 archived plan files, and 4,346 beads, scoped to a single project. Cold-process
numbers are meaningfully worse; the Agent pane's first load in a fresh process measured
5,635ms.

| Sub-tab | Snapshot load                       | Query index | First paint     |
| ------- | ----------------------------------- | ----------- | --------------- |
| Agent   | 1,529ms                             | 1,535ms     | ~3,100ms        |
| Bead    | 2,057ms                             | 486ms       | ~2,500ms        |
| Plan    | 2,536ms                             | 12ms        | ~2,500ms        |
| File    | 273ms (500-row head)                | 519ms       | ~800ms          |
| Stitch  | pushes `limit` into the git backend | —           | already bounded |

The whole of each pane's first-paint cost is serial inside one worker: every pane's
`_build_snapshot` (`src/sase/ace/tui/widgets/artifacts/snapshot_pane.py`) returns only
after both the load and the index are done, and `_apply_snapshot` renders after that.

### 1.2 Where the Agent pane's 3.1 seconds goes

`load_agents_snapshot` (`.../artifacts/agents_data.py`) → `build_agent_catalog_snapshot`
(`src/sase/agents/catalog/_build.py`), 1,529ms:

- **905ms — `load_name_registry()`, all of it revalidation.** The 16MB registry JSON
  parse is already process-cached; `_cached_registry` then calls
  `registry_file_is_stale` on every hit (`src/sase/agent/names/_registry_store.py`).
  That costs 590ms in `_source_signature()`, which stats 37,872 source paths (116,947
  `posix.stat` calls total), plus 307ms sweeping `entry_owner_missing` across 12,673
  entries (72,559 `is_file` calls). `name_registry_load_session()` exists to memoize the
  signature for a bounded loop, but no Artifacts caller opens one, and even inside a
  session the first signature still costs 590ms.
- **263ms** — the `_build_row` join loop over 12,673 entries.
- **213ms** — two `load_dismissed_bundle_summaries` queries.
- **89ms** — the projected artifact-index read.

`build_agents_query_index` (`.../artifacts/query_rows.py` →
`compile_artifact_query_index` in `src/sase/core/query_profile_corpus_facade.py`),
1,535ms over 11,783 rows:

- **408ms** — `agent_catalog_query_entry` builds one dict per row
  (`src/sase/agents/catalog/_query.py`).
- **277ms** — `coerce_artifact_query_rows` turns each dict into an `ArtifactQueryRow`.
- **245ms** — `_row_wire` turns each `ArtifactQueryRow` back into a plain dict.
- **716ms** — Rust `compile_corpus_with_profile`, which converts each incoming dict to a
  `serde_json::Value` before building a `QueryRow` from it.
- **66ms** — `_observed_facets`.

Evaluation itself is not the problem: once the index exists, a query runs in 1–10ms.

**None of the 1,535ms index build is needed for first paint.**
`_filtered_agents_snapshot` (`.../artifacts/agents_query.py`) short-circuits when the
query's remainder is blank — the default `limit:100` case — and slices `snapshot.rows`
directly without consulting the index at all.

Project scoping does not help: 11,783 of 12,667 rows survive it, and it is applied in
Python after all rows are built.

### 1.3 Where the other panes' time goes

**Plan, 2,536ms.** `load_proposals` (`.../artifacts/plans_data_sources.py`) calls
`build_plan_inventory(limit=50, statuses=("proposed",))` and reads only
`inventory.proposed` — which is empty in the common case. But `build_plan_inventory`
(`src/sase/main/plan_inventory.py`) always computes `rejected`, and
`collect_rejected_plans` (`src/sase/main/plan_inventory_collectors.py`) reads and
YAML-parses all 1,900 unrepresented archived plan files through `plan_metadata_for_path`
→ `parse_plan_frontmatter` → `yaml.safe_load`, sorts by mtime, then keeps 50. Measured:
`build_plan_inventory` 1,966ms, of which `collect_rejected_plans` alone is 1,401ms and
is discarded whole. `collect_approved_plans` has the same shape — it calls
`plan_metadata_for_path` per candidate before slicing to the target.

**Bead, 2,057ms.** `_load_external_issue_caches` (`.../artifacts/beads_data.py`) runs a
`gh` subprocess inside the snapshot worker: 355ms, network-bound, unbounded in the worst
case. Its 60s TTL means every ACE session's first Bead paint pays it, every paint after
60s idle pays it, and the explicit-refresh path (`force=True`) bypasses the TTL and
always pays it. Beyond that: `_build_external_issue_links` makes 17,154
`normalize_external_ref` calls (133ms), and `resolve_project_alias_ref` reloads the
project alias map and lifecycle records 20–23 times per load (110ms). The Rust bead
reads (`bead_list` 300ms, `bead_ready` 203ms, `bead_blocked` 203ms) are the honest
floor.

**File, ~800ms.** Already the fastest, because it is the only pane that implements a
bounded first page (`FILES_FIRST_PAGE_LIMIT = 500`) with a background extension to the
full index and an on-demand grow when the cap outruns the loaded page
(`_schedule_full_extension`, `_maybe_grow_files_snapshot`). This epic generalizes that
pattern rather than inventing one.

### 1.4 Why the existing benchmark did not catch this

`tests/perf/bench_agent_catalog.py` asserts a 400ms budget over a 12,525-name synthetic
registry and passes. It passes because its fixture removes both dominant real costs: it
runs under an isolated `SASE_HOME` with no artifact tree, so `_source_signature()` has
almost nothing to stat, and its synthetic entries deliberately omit `source` (the
docstring says so) so `entry_owner_missing` never touches the filesystem. The bench
measures a 273ms build of a corpus whose real-world equivalent takes 1,529ms. Fixing the
bench is a prerequisite for trusting any of this epic's numbers, which is why it is
phase one.

## 2. Approach

Four independent levers, applied where each pane's measurement says it belongs:

1. **Do not revalidate what has not changed** (`registry`). The registry's process cache
   is correct but its freshness proof is more expensive than the parse it protects.
2. **Let `limit:` reach the work** (`agent-paint`). First paint needs a sorted head
   slice; the full corpus index is only needed once the user types a filter term or
   raises the cap. That is the Files pane's shipped contract.
3. **Stop materializing the same row four times** (`core-corpus`, `entry-projection`).
   Every pane's index build pays it; the Agent pane pays it 11,783 times.
4. **Do not read files to produce rows you then throw away** (`plans`), and **do not put
   the network in the paint path** (`beads`).

Phases 2–7 are independent of each other and can run in parallel once `bench` lands.
`core-corpus` and `entry-projection` are explicitly scoped not to collide: `core-corpus`
changes only how the Rust binding consumes the existing wire shape, and
`entry-projection` must keep that wire shape byte-identical.

### 2.1 Targets

Measured against the same live-scale corpus, first paint of the default `limit:100`
view:

| Sub-tab | Now      | Target  |
| ------- | -------- | ------- |
| Agent   | ~3,100ms | ≤ 400ms |
| Bead    | ~2,500ms | ≤ 700ms |
| Plan    | ~2,500ms | ≤ 400ms |
| File    | ~800ms   | ≤ 500ms |

Background completion (full corpus index available for wide queries) should land within
~1.5s of first paint on the Agent pane, and the pane must never present a stale index as
a fresh one — the existing generation/digest guards in `ArtifactQueryCacheKey` and
`_accept_snapshot` already enforce that and must keep doing so.

### 2.2 Non-goals

- Changing the `limit:` semantics, the query dialects, or the saved-query and
  query-history wire formats.
- Moving the agent name registry to a Rust-owned store.
  `src/sase/agents/catalog/__init__.py` documents that promotion trigger; this epic
  makes the registry cheap to re-read, which is the smaller change, and leaves the
  trigger intact.
- Pruning the artifact index. That is `sase-kh` and is separate.
- Reducing what the Rust bead reads cost. `bead_list`/`bead_ready`/`bead_blocked` are
  the Bead pane's honest floor here.

## 3. Honest first-paint benchmarks for the Artifacts panes

Repair `tests/perf/bench_agent_catalog.py` so it measures what the real load measures:

- Give the synthetic fixture a real artifact tree under its isolated `SASE_HOME`, sized
  so `source_signature_paths()` returns a path count in the same order as the live
  37,872, and give its entries a `source` (`artifact` / `dismissed_bundle`) with
  `artifacts_dir` / `bundle_path` values that exist, so `entry_owner_missing` does the
  filesystem work it does in production. Keep the existing no-artifact-tree variant as a
  separate parametrization so the split between parse cost and revalidation cost stays
  visible.
- Re-derive the budget from the repaired fixture. The current `_BUDGET_MS = 400` is a
  measurement of the wrong thing; record the new pre-fix baseline as the number
  `registry` has to beat, and set the post-fix gate in that phase.

Add `tests/perf/bench_artifacts_first_paint.py`, a `slow`-marked bench that reports, per
pane (Agent, Bead, Plan, File), the split of snapshot-load time versus query-index time
versus time-to-first-renderable-rows for a default `limit:100` query over a live-scale
synthetic corpus. It asserts structural invariants that cannot flake — that first paint
does not depend on the full-corpus index, that per-pane work does not scale with corpus
size beyond the head slice — and prints wall-clock p50/p95 for humans to compare, in the
style of the Admin Center bench described in `tests/perf/README.md`.

Reuse `tests/perf/fixtures.py` where it fits. This phase changes no production code.

## 4. Stop revalidating the agent-name registry on every load

In `src/sase/agent/names/_registry.py` and `src/sase/agent/names/_registry_store.py`,
make a cache hit cheap:

- Gate the full `registry_file_is_stale` proof behind a bounded validation memo. A cache
  hit already checks the registry file's own `(st_mtime_ns, st_size)` via
  `_file_signature`; the expensive part is proving the _sources_ have not moved
  underneath it. Memoize that proof against the in-process
  `agent_name_registry_freshness_token()` generation plus a short wall-clock TTL, so a
  burst of loads inside one ACE paint pays it at most once, and any registry mutation
  (which already calls `invalidate_agent_name_registry_freshness()`) invalidates it
  immediately.
- Make `_source_signature()` itself cheaper. `source_signature_paths()` already caches
  its directory walks (`_directory_entries_for_signature`,
  `_artifact_dirs_for_signature`); the 590ms is the per-path `stat` loop over the
  returned 37,872 paths. Artifact directory names are immutable once created and the
  scan deliberately excludes live run contents, so a directory whose entry set is
  unchanged does not need every child re-statted. Fold the stat into the same
  signature-keyed cache the walk already uses.
- Bound the `entry_owner_missing` sweep the same way. Its purpose is detecting entries
  whose backing artifact or bundle was deleted out from under the registry; that does
  not need to be re-proven on every read within a TTL window.

Correctness bar: a registry that genuinely went stale must still be detected and
rebuilt, and the existing tests in `tests/` covering registry staleness, rebuild, and
`reset_name_registry_caches_for_tests` must pass unchanged. Add coverage for the new
memo: a mutation invalidates it, TTL expiry re-proves, and a deleted artifact directory
is still caught on the next proof.

Expected: `load_name_registry()` on a warm cache drops from ~905ms to single-digit ms;
the Agent pane's snapshot load drops from 1,529ms to ~600ms. This also speeds every
other registry consumer, including ACE startup.

Verify with the repaired `bench_agent_catalog` from `bench`.

## 5. Two-stage Agent pane load

Mirror the shipped Files-pane contract in `.../artifacts/agents_pane.py`,
`agents_data.py`, and `agents_query.py`.

- Give `AgentsSnapshot` a `complete: bool`, matching `FilesSnapshot.complete`, and give
  `load_agents_snapshot` a `limit` parameter. The head slice must be the same
  newest-first ordering the pane already renders (descending `started_at`, then name),
  so a bounded load and a full load agree on the first N rows.
- `_build_snapshot` takes `request.full` the way `files_pane.py` does: on the bounded
  pass, load the head slice, skip the full-corpus index build entirely, and return a
  renderable result. `_filtered_agents_snapshot` already short-circuits on a blank query
  remainder, so the default `limit:100` view paints correctly with `_query_index` still
  `None`.
- After first paint, `_schedule_full_extension`-style: yield the event loop, then run
  the full load plus `build_agents_query_index` in the background, keyed on the same
  `_load_generation` guard `_on_agents_query_index_worker_changed` already uses.
- Add the Agent equivalent of `_maybe_grow_files_snapshot`: when the user raises the cap
  past the loaded head, or types a filter term while the index is still building,
  request the full pass. `AgentsQueryMixin` already has the states this needs — it
  returns a `pending` flag when `_query_index is None`, and `_sync_agent_query_bar`
  already accepts `lower_bound` and `coverage_label` for exactly this "count is not
  final yet" case.
- Keep the compatibility constant `AGENTS_DEFAULT_LIMIT` meaningful or remove it
  deliberately; its current docstring asserts the pane does _not_ cap the loaded
  snapshot, which this phase reverses. Update the comment rather than leaving it lying.

Choose the head-slice size from the `bench` numbers. `FILES_FIRST_PAGE_LIMIT` is 500
against a default cap of 100; the same 5× headroom is a reasonable starting point, and
the bench should show what it costs.

Behavior that must not regress: relation-panel targets, jump mode, saved-query and
query-history application, revival of dismissed agents, and `set_project_scope`
reloading. Each of those reads the snapshot, and each needs a defined answer for "the
full corpus has not arrived yet." Extend the existing Agent-pane tests under
`tests/ace/tui/widgets/artifacts/` to cover the bounded-then-extended sequence,
including a filter typed during the window before the index lands.

## 6. Direct dict-to-QueryRow corpus construction in sase-core

Open the core repo with `sase repo open sase-core -r "<reason>"` and work there.

`py_compile_corpus_with_profile` in `crates/sase_core_py/src/lib.rs` currently does, per
row, `py_to_json_value(&item)` to build a full `serde_json::Value` and then
`QueryRow::from_wire(&json, &profile)`. That is a complete second materialization of
every row before the corpus is built, and it is most of the measured 716ms for 11,783
Agent rows.

Read the `PyDict` directly into `QueryRow` — the same fields, the same validation, the
same errors including the `rows[{idx}]: {error}` prefix — without the intermediate
`Value`. Keep `QueryRow::from_wire` as-is for the JSON callers that need it; add the
direct path alongside. The accepted wire shape does not change, so this phase is
invisible to Python.

Add a Rust-side bench over a corpus in the same order as the Agent pane's (~12k rows,
~20 fields) and record before/after. Then release sase-core and raise the `sase-core-rs`
floor in this repo's `pyproject.toml`, following the pattern of `sase-rt.2` and the
`just _core-overrides-arg` local-override flow.

Read `sase memory read rust_core_backend_boundary` before starting; this phase is
squarely on the core side of that boundary.

## 7. Cut the Python-side corpus marshalling cost

Two independent costs in `src/sase/core/query_profile_corpus_facade.py` and
`src/sase/agents/catalog/_query.py`, together ~930ms for the Agent pane:

**Stop building each row three times.** `compile_artifact_query_index` goes entry-dict →
`ArtifactQueryRow` (277ms) → wire-dict (245ms). `_row_wire` rebuilds as a plain dict
exactly what `coerce_artifact_query_row` just finished normalizing. Produce the wire
dict once. The constraint is that `ArtifactQueryRow` is also what `_observed_facets` and
the parity-only Python reference evaluator consume, and that
`coerce_artifact_query_rows` carries a genuine corpus-level pass for the `patches` pane
(transitive ancestry) that must survive. Keep the emitted wire shape byte-identical so
this composes with `core-corpus` in either order; a test asserting the wire dict is
unchanged for a fixed corpus is the cheapest way to hold that line.

**Shrink the per-row projection.** `agent_catalog_query_entry` costs 408ms over 11,783
rows. `_text_values` calls `_label_values`, and `agent_catalog_query_entry` calls both
`_label_values` and `_text_values` and then joins `_text_values` again for
`searchable_text` — three overlapping `_distinct` passes per row where one would do.
`agent_catalog_runtime_seconds` calls `parse_local` twice per enriched row. Neither is
subtle; both are hot only because the corpus is large.

This is shared machinery: `sase agent search` (`src/sase/agents/cli_search.py`) uses the
same adapter, and every Artifacts pane uses the same facade. Do not change what any
query matches. The parity suite between the Rust and Python evaluators is the guard; run
it.

## 8. Defer plan metadata reads past the inventory slice

In `src/sase/main/plan_inventory.py` and `src/sase/main/plan_inventory_collectors.py`:

- **Sort first, read metadata second.** `collect_rejected_plans` already has each
  candidate's mtime from a cheap `stat` before it needs any file content. Sort by mtime,
  slice to `limit`, and call `plan_metadata_for_path` only on the survivors. The one
  case that genuinely needs metadata for every candidate is a non-empty `tiers` filter,
  since tier comes from frontmatter; keep the current behavior on that path and take the
  fast path when `tiers` is empty. Apply the same shape to `collect_approved_plans`,
  which calls `plan_metadata_for_path` inside `_approved_plan_from_meta` before its own
  slice. Measured: 1,401ms → tens of ms for the 50 rows actually returned, and this
  helps `sase plan list` as much as it helps the pane.
- **Honor `statuses`.** `build_plan_inventory(statuses=("proposed",))` currently
  computes approved and rejected regardless; `status_filter` is only consulted at render
  time. The comment explains that approvals feed `represented_paths` and improve
  rejected inference — that reasoning holds for approved, not for rejected when rejected
  is not requested. Skip the rejected collection when the caller did not ask for it, and
  keep `total_archived_proposals` accurate (it comes from `archived_plan_paths()`, 9ms,
  not from the collector).
- **Make `plan_metadata_for_path` cheap when it does run.** It reads the entire plan
  file with `read_text` and hands the whole string to `parse_plan_frontmatter`, which
  `yaml.safe_load`s the frontmatter block for two scalar keys, `title` and `tier`. Read
  only as far as the closing `---`. `src/sase/sdd/plan_tiers.py` already has a bounded,
  signature-keyed cache (`_PLAN_TIER_CACHE`) for the tier half; extend that idea to
  cover title, or route this call through it.

`sase plan list` output must be identical before and after for the same corpus; a golden
comparison on a fixture archive is the check. Existing tests under `tests/` covering
plan inventory and `sase plan list` must pass unchanged.

## 9. Take the external-issue network call off the Bead first-paint path

In `.../artifacts/beads_data.py` and `.../artifacts/beads_pane.py`:

- **Move `_load_external_issue_caches` out of the snapshot worker.** Paint from whatever
  cache exists — including none — and refresh external issues in a separate background
  worker that applies its result as a follow-up update, the way the Agent pane's
  query-index worker applies its own result under a generation guard. The Bead pane
  already renders external-issue links as an enrichment layer over local beads, so an
  empty-then-populated sequence is a presentation state it can express; give it an
  explicit one rather than a silent blank. The refresh worker must be cancelled on
  unmount and on project-scope change, and its result rejected if the generation moved.
- **Keep `force=True` meaningful without making it blocking.** An explicit refresh
  should trigger the external refresh immediately, but still off the paint path.
- **Memoize per-load lookups.** `resolve_project_alias_ref` reloads the project alias
  map and lifecycle records 20–23 times inside one load (110ms). Resolve once per load
  and pass it down. Similarly, `_build_external_issue_links` calls
  `normalize_external_ref` 17,154 times (133ms) across 4,346 beads; hoist the per-bead
  ref parsing out of the inner loop.

Expected: Bead first paint drops from ~2,500ms to ~700ms, bounded by the Rust bead
reads, and stops varying with network latency. Extend the Bead pane tests to cover paint
with no external cache, the arrival of a refresh, a refresh rejected on a scope change,
and an external-issue provider error surfacing as it does today.

## 10. End-to-end verification and the perf recipe

With every phase landed on the combined tree:

- Re-run `tests/perf/bench_artifacts_first_paint.py` and record the four panes' before
  and after numbers against §2.1's targets. State plainly which targets were met and
  which were not; a missed target is a result, not a failure to hide.
- Re-run the repaired `bench_agent_catalog` and confirm the registry revalidation win
  held.
- Sanity-check on a real machine corpus, not only the synthetic one, using the "Agent
  Artifact Startup" recipe already in `tests/perf/README.md` as the model.
- Add an "Artifacts Sub-Tab First Paint" section to `tests/perf/README.md` describing
  how to re-measure, so the next person changing any of these paths has the recipe.
- Run `just check-full` through `/sase_monitor`, and `just test-visual` if any pane's
  loading or empty-state rendering changed; a new "index still building" affordance on
  the Agent pane would need golden updates.

Any real defect found in an unrelated area during this epic goes in a
`PROPOSED FOLLOW-UP:` note on the phase bead, not a new task bead.
