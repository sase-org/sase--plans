---
tier: epic
title: Move artifact-link indexing into the Rust core
goal: Complete sase-1kj.8 with one shared native index, lazy link chips, and indexed
  Patch projections while preserving current behavior
phases:
- id: parity-baseline
  title: Capture artifact-link parity and performance baselines
  depends_on: []
  size: small
  description: 'parity-baseline: freeze current chips, targets, and Patch-edge outputs
    in fixtures and record reproducible CPU and wall-time baselines on a saved real
    aggregate before either implementation changes.'
- id: native-index
  title: Build the shared Rust artifact-link index and opaque binding
  depends_on:
  - parity-baseline
  size: medium
  description: 'native-index: implement the immutable core index, indexed row and
    projection queries, and GIL-releasing opaque Python handle against the captured
    parity fixtures.'
- id: python-integration
  title: Route link chips and Patch projections through the shared native handle
  depends_on:
  - native-index
  size: medium
  description: 'python-integration: install the thin single-flight adapters, preserve
    Patch deferral, remove the Python aggregate walks, verify the source pin and parity,
    and measure the improvement.'
proposed_by: bbugyi200.athena.sase-1kj.8
parent_bead: sase-1kj.8
create_time: 2026-10-13 17:50:20
status: done
bead_id: sase-1kj.8.1
---

- **PROMPT:** [prompts/202610/artifact_link_inverted_index.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/artifact_link_inverted_index.md)
- **PARENT:** [202610/apollo_tui_startup.md](https://github.com/sase-org/sase--plans/blob/main/202610/apollo_tui_startup.md)
- **BEAD:** [sase-1kj.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kj/sase-1kj.8.1.md)

# Artifact-link inverted index

This is the child implementation epic for **sase-1kj.8**, the `core-link-index` phase of
`plan:202610/apollo_tui_startup.md`. Submission from the assigned phase lets SASE stamp
the parent relationship. The phase had no prior-work notes when read. Its prerequisites,
`sase-1kj.5` and `sase-1kj.6`, are closed.

An epic is appropriate because freezing the compatibility oracle, implementing a new
native handle, and migrating concurrent Python consumers have distinct verification
boundaries. The phases are intentionally sequential. There are no new reviewer
decisions: retain the outer epic's final `fallback_seconds = sec_45` choice. No memory
edits are authorized or needed.

## Scope and existing behavior

`src/sase/ace/tui/relations/link_index.py` currently builds all chips eagerly in Python.
Each raw row invokes native relation/canonicalization helpers, then each endpoint pays
further label, target, and alias work. Its scope-bounded LRU releases its lock between
lookup and build, permitting duplicate concurrent builds.

`src/sase/ace/tui/relations/artifact_links.py:artifact_link_edges` walks all rows again
for each pane. `relations/patches.py:build_patches_relation_index` uses this projection
in addition to the Patch ancestry/children/sibling graph. The existing memo in
`actions/patch/_relation_index.py` preserves a cheap targets-only hidden view and counts
full builds; retain those semantics. It too currently releases its lock before
construction.

The two walks have different duplicate policies. Link chips canonicalize endpoints and
select the highest nonnegative `uses` row, with first occurrence winning ties and
determining position. Pane edges preserve the first matching row after resolving
endpoints into pane identities. Cross-project duplicates can consequently carry
different metadata and targets. An index containing only winning chip rows would
silently change pane behavior.

All backend parsing, canonicalization, aliasing, adjacency, and target resolution belong
in `sase_core`. Python keeps snapshot loading, process-local cache ownership, Textual
scheduling, presentation declarations, colors/icons, and conversion into existing
dataclasses. No Python backend fallback is permitted.

## Implementation contract

### Immutable native index

Add the implementation under `crates/sase_core/src/artifact_link/`, keeping `mod.rs` a
facade. Reuse `wire.rs`, `relation.rs`, `row_resolution.rs`, and the existing agent
identity functions. Avoid implementing another ref grammar.

Compile once from an already-loaded `ArtifactLinksSnapshot`'s ordered aggregate rows,
including each row's `_project`, plus one captured owner/alias context and the assembled
relation metadata required by current fallback behavior. Accept this through a versioned
build request. One bulk conversion into owned Rust input is acceptable; per-row Python
binding calls and returning every chip are not. Do not reopen aggregate paths during
compilation: that could index newer bytes under an older snapshot signature. Existing
Python snapshot loading and its error reporting remain intact.

Use a permissive read-model input compatible with the existing aggregate loader, not the
strict mutation-only `ArtifactLinkRowWire`: tests and historical aggregates can omit
schema/metadata, have malformed refs, and contain non-integer or negative `uses`. Pin
the existing coercion, skip, and error behavior with fixtures. A corrupt row must not
start rejecting an entire snapshot where the old path tolerated it.

The immutable structure stores compact row records and ordinal adjacency lists:

- A canonical chip view deduplicates exactly as the current `_build_link_index` does.
  Preserve both perspectives, labels, direction, metadata, writability, and stable
  semantic-first ordering. Semantic relations are `related`, `supersedes`, `implements`,
  and `derives-from`; then order by label and public neighbor ref.
- A raw-row view preserves original order and metadata for pane projection. Index both
  endpoints under `artifact_row_ref_lookup_keys`, using each row's project hint and
  captured identity context. Candidate queries union/deduplicate row ordinals and return
  original order without walking unrelated rows.
- Canonical ref entries take precedence over aliases. Alias insertion is a second,
  first-wins pass; preserve owner-aware agent spellings, seven-character stitch aliases,
  `ref:plan:` aliases, and `chop:`/`job:` compatibility. Public chop neighbors remain
  `job:` and have no selectable target.
- Native target synthesis preserves the current `target_for_ref_kind` behavior,
  including project hints, document/archive identities, malformed stitch refs, unknown
  document kinds, and the non-selectable bug/chat/job kinds. For chips, preserve the
  distinction between the first indexed target for a ref and a neighbor target resolved
  with that particular row's project.
- Do not change the link store, aggregate schema, relation registry, or persistent
  publication paths. This is a read index.

Provide a frozen PyO3 handle owning an `Arc` to the core index. Follow the existing
`AgentArchiveCorpusHandle` binding pattern in
`crates/sase_core_py/src/agent_custody/mod.rs`; keep domain logic out of PyO3. Register
additive bindings in the artifact-links domain. Suggested operations:

1. `artifact_link_index_build(request)` returns the opaque handle.
2. `artifact_link_index_chips(handle, ref)` returns only that ref's native chip records,
   with target wire identities and parsed neighbor kinds for styling.
3. `artifact_link_index_target(handle, ref)` returns membership and an optional target,
   distinguishing an indexed non-selectable ref from an absent ref.
4. `artifact_link_index_rows(handle, lookup_keys)` returns only matching raw rows in
   original order. Empty keys yield no rows.
5. `artifact_link_index_project(handle, request)` accepts known row identities,
   permitted relation names, and the fallback project hint. Build known-target lookup
   keys once, select candidates through the inverted index, resolve both endpoints with
   project-first/deterministic-fallback behavior, and return only valid edges touching
   known targets. Preserve the old first-row deduplication.

The projection operation must use the same indexed candidate selection as the rows
operation, not hide another full scan. Python supplies relation declarations and
decorates returned edge records with their kind/label. Existing native
`artifact_row_index_keys` and `artifact_row_ref_lookup_keys` own key semantics. Preserve
exact-first agent/file resolution and stitch-prefix matches, because the shared
projection helper also serves non-Patch panes.

Release the GIL for core compilation and potentially substantial queries after
converting inputs to owned Rust values. Return only requested output across the
boundary. Keep no Python objects or callbacks inside native state. Tests may enumerate
refs for parity, but production construction must never enumerate and materialize the
entire chip map.

### Python cache and consumers

Put the typed binding adapter in a new `src/sase/core/artifact_link_index_facade.py`.
Put the TUI snapshot/cache adapter beside `relations/artifact_links.py`, so LinkIndex
and pane projection use the same native handle. Keep the core facade independent of TUI
imports.

Memoize by `snapshot.source_key` within project scope, with the current bounded
eight-scope LRU and one live signature per scope. Include the captured owner/alias
context and supplied relation metadata in cache validity, or invalidate on their
existing refresh/reset boundary; do not serve an index built for a different identity.
For synthetic or empty-signature snapshots with actual rows, use snapshot identity with
a strong reference, or bypass global reuse, so two distinct test or in-memory snapshots
cannot collide. Do not compute an O(rows) hash on a cache hit.

Use a per-scope lock or in-flight result with a cache recheck after acquisition. Two
callers for the same snapshot must compile once. A failed build releases all waiters and
allows retry; a slow older build must not replace a newer published generation. Eviction
must not invalidate handles still held by callers. Cache clear hooks must also be safe
with pending builds. Avoid holding the cache registry lock across native calls or
waiting on another scope.

Keep `LinkChip`, `LinkIndex.chips_for`, `target_for`, and `source_key` behavior. Use
lazy Mapping adapters for `by_ref` and `targets_by_ref` if that is the least disruptive
way to preserve existing constructor/test support. In particular,
`actions/hints/_files.py` checks membership in `targets_by_ref`, including refs whose
target is `None`; tests also iterate `by_ref`. Membership and ordinary lookups must not
enumerate the graph. Bound the requested-chip cache (for example, 128 refs per handle)
and cache styling by neighbor kind/pane; do not eagerly create Python chips or reparse
each neighbor. Capture identity/configuration context on the existing worker path; lazy
Mapping operations must perform no filesystem I/O.

Route `artifact_link_edges` through the shared handle's indexed projection. Keep Patch
graph construction and the existing `build_relation_index` contract. Preserve source
identity, relation ordering, known targets, dangling-target handling, metadata, and the
outputs of other pane callers. Delete only Python walk helpers that have no remaining
callers; standalone ref-navigation helpers may remain.

Make the full Patch memo build single-flight as well, preserving strong Patch object
identity comparisons, signature invalidation, full-over-targets-only precedence,
deferred upgrade on reveal, and exactly one real-build count. A hidden targets-only
request must not compile the native graph. Keep cache lock ordering acyclic and do not
add waits to the UI pump.

Add a `links.index_build` trace span through the existing trace utility, emitted only
for real native builds, with raw/deduplicated row counts and elapsed time. Measure
compilation/conversion separately from chip-query and projection work. Any first query
that could materialize many chips must occur on the existing worker path; prefetch the
selected ref there if necessary before the UI uses it. After awaits, re-read current UI
selection before applying results. Preserve the startup gate, pump-free task/coalescing
guards, and activity gates.

## Phase: parity-baseline

Work in sase only. Read `tui_perf.md` and the applicable repo instructions. Add small
sanitized fixtures and expected normalized outputs to the existing link-index and
relation test coverage. Capture expectations by running the current code before any
replacement. Include directed/inverse and symmetric relations, unequal and equal
duplicate uses, reversed related rows, duplicates in different projects, self-edges,
aliases colliding with canonical entries, projected writability, metadata, invalid/empty
refs, tolerant counts, target project hints, and known-target fallbacks. Pin all
`LinkChip` fields and final Patch `RelationIndex` edges/targets, not just counts. Fixed
identity/style inputs make these fixtures deterministic.

Add a focused reusable comparison/benchmark helper under `tests/perf/` that uses the
current public snapshot and index entry points. With `perf_counter` and `thread_time`,
record cold `_build_link_index`, full Patch projection, warm reuse, and representative
chips/target queries. Record input hash, row/ref counts, host load, interpreter/core
versions, and separate input-loading time. The benchmark must measure equivalent
eager-consumer work as well as lazy construction after the migration, so deferred work
is not reported as eliminated work.

Load a real aggregate through the established snapshot loader and save one stable copy
for both measurements. Keep real data and full expected outputs out of git; register
durable evidence using `sase artifact create`, then record artifact refs and
reproducible commands in the phase note. Later phases consume it with
`sase artifact read`, never a direct sidecar-file read. If no nonempty real aggregate is
available, record the limitation and provide a representative synthetic 45k-row fixture;
do not claim real-data parity from synthetic data.

Run the focused tests and `just fmt`, then `sase tool run check`. This phase changes
tests/evidence only and hands a stable parity oracle to the two implementation phases.
No production walk is removed here.

## Phase: native-index

Open `sase-core` with `sase repo open` and read its printed `AGENTS.md`. Implement the
native structure and all five query operations above with additive wire types, core
tests, and binding registration/round-trip tests. Port the baseline fixture expectations
into Rust parity tests; do not keep a second production Python implementation as an
oracle. Include row-order/dedup differences, alias collisions, project-hint resolution,
and empty/malformed-input cases.

Prove that a rows/project query with a fixed Patch set does not scan unrelated rows,
using a focused core operation-count assertion or equivalent deterministic test. Test
ownership/lifetime and repeated concurrent reads of the frozen handle. Measure native
build/selected query costs with input/output conversion included when using the binding.
Do not wire TUI consumers yet.

Keep new files below the core repo's size limit and import core symbols by module path
instead of adding root/prelude aliases. Use `just` recipes, never bare cargo. Run
focused core and binding tests, `just fmt`, and `sase tool run check` in the linked
checkout. The finalizer must include the changed core repository. Record binding names
and verification evidence for the integration phase; SASE owns commits, and the next
phase uses the resulting landed core revision.

## Phase: python-integration

Implement the adapters and caller changes specified above. Before calling the new
binding, verify that the opened core checkout contains the native phase and rebuild the
checkout-local extension through supported recipes. Follow `docs/rust_backend.md`: for
an already-landed core commit, ratchet `sase-core-revision.txt` to a revision containing
it and verify `tools/check_sase_core_rs_bindings`. If this phase also changes core,
declare both repos so host finalization commits core first and writes its SHA into the
primary pin. Never guess an uncommitted SHA or create commits manually.

The published `sase-core-rs` dependency window in `pyproject.toml` and `uv.lock` belongs
to the release workflow. Do not edit it for unpublished bindings; the release floor
smoke gate enforces publication. Do not edit Rust package versions or changelogs.

Extend the regression coverage in:

- `tests/ace/tui/test_link_index.py` and
  `tests/sdd/test_artifact_link_store_link_index.py` for parity and lazy lookups.
- `tests/ace/tui/test_artifacts_relation_sources_artifact_links.py` and
  `tests/ace/tui/artifacts_contract/test_relation_goldens.py` for projection parity
  across panes, including Patch edges and project ambiguities.
- `tests/ace/tui/test_patch_relation_index.py` for hidden deferral, reveal,
  invalidation, single-flight builds, and exact real-build counting.
- The existing LinkSubject, link-follow, and Record deck tests for worker-side
  selected-ref materialization, membership, and navigation behavior.

Add deterministic barrier/event-based concurrency tests: LinkSubject and Record deck
requests for one signature compile once; failure unblocks waiters and retry works;
signature replacement and scope eviction do not publish stale results; different
synthetic snapshots with identical placeholder keys stay separate. Ensure the Patch path
no longer invokes per-ref bindings on every aggregate row, and unrelated rows do not
increase the projected candidate set. Avoid flaky wall time thresholds in unit tests.

Compare the complete fixture outputs and saved real-aggregate outputs to the baseline,
then repeat the same timing experiment on that saved input. Report cold build, selected
chip query, full Patch projection, warm hit, and combined costs, including both CPU and
wall time and host load. Correct parity differences and any replacement full-graph
Python walk before closing. Query output is naturally proportional to its returned
neighbors; report that cost honestly.

Run focused tests, `just fmt`, and `sase tool run check` in sase; run core's wrapped
check too if it changed. No rendered output change is intended; if output or visual
coverage does change, follow the screenshot memory and required targeted workflow. Read
`symvision.md` before fixing unused-symbol failures. Remove dead walk code without
removing independently used helpers or masking failures with blanket whitelists.

## Verification, closure, and child-epic landing

Every worker reads required memory through `sase memory read`. Long verification
commands use `sase_monitor` with a concrete continuation and are allowed to finish;
never end a turn with an unmonitored command still running. Run the wrapped
`sase tool run check` gate for each changed repo. `just check-full` is not part of this
plan. Failures reproduced identically on a clean base get a `PROPOSED FOLLOW-UP:` note
with evidence on the worker's assigned phase; that preexisting failure alone does not
leave a completed phase open.

Workers close only their own assigned child phase after running its
`sase bead epic-symbols <id>` check. Resolve or re-key every remaining exemption to a
still-open owner before closing. Record out-of-scope discoveries as
`PROPOSED FOLLOW-UP:` notes; workers never create task beads or close ancestors.

The child epic's land agent verifies that all three phases are closed, both repo gates
and the binding pin are accounted for, parity evidence is complete, and before/after
measurements include any remaining conversion cost. It records the combined outcome on
**sase-1kj.8**, runs `sase bead epic-symbols sase-1kj.8`, resolves any leftovers, and
closes **sase-1kj.8** with a note citing checks, parity evidence, timings, and
limitations. Use the host's normal child-epic settlement order where descendant guards
apply. Only that land agent owns the enclosing phase closure; child workers merely
prepare its evidence. **Never close sase-1kj or an older ancestor plan.**

Completion requires one native build per aggregate generation for concurrent consumers,
no Python full-aggregate chip/projection walk, lazy Python chip creation, indexed Patch
selection, preserved startup/hidden-tab behavior, unchanged fixture and real-data
outputs, and a core revision pin that exposes every called binding.

The optional agent-catalog facet walk stays outside this migration: it currently counts
raw touching rows and has different semantics from chip deduplication. Other outer-epic
phases, global installs, remote-host operations, feature flags, memory, keymaps, and
persistent artifact-link storage are outside this child epic.
