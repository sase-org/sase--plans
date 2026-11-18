---
tier: epic
title: Bound retained log objects with targeted backend queries
goal: Complete sase-1t3.11 by removing project-wide Python log and index snapshots
  from TUI queries while preserving results, attribution, freshness, and responsiveness.
parent_bead: sase-1t3.11
phases:
- id: artifact_files
  title: Stream artifact-file queries before Python conversion
  size: medium
  depends_on: []
  description: 'artifact_files: replace the persistent full artifact-file index cache
    with targeted Rust association queries, preserve full-reader compatibility, and
    prove retention and result parity.'
- id: memory_reads
  title: Query memory-read history for the selected context
  size: medium
  depends_on:
  - artifact_files
  description: 'memory_reads: add the Rust streamed audit-query contract and memory
    decoder, switch the memory lane to bounded results, and preserve schema and attribution
    semantics.'
- id: bead_touches
  title: Restrict touch queries to exact actors and session aliases
  size: medium
  depends_on:
  - memory_reads
  description: 'bead_touches: identify the surviving touch owner, push actor matching
    into Rust, remove full touch/view snapshots, and preserve legacy-view and session
    attribution.'
- id: artifact_reads
  title: Reuse targeted audit queries for artifact-read history
  size: medium
  depends_on:
  - bead_touches
  description: 'artifact_reads: extend the audit-query contract to artifact events,
    remove the full artifact-read snapshot, and prove event, ordering, and context
    parity.'
- id: stranded_reports
  title: Collect stranded facts and titles from one targeted snapshot
  size: medium
  depends_on:
  - artifact_reads
  description: 'stranded_reports: remeasure the indexed bead reader, eliminate repeated
    candidate collection, and retain only the issue facts required by stranded reports.'
- id: verify_retention
  title: Verify combined heap retention and query costs
  size: medium
  depends_on:
  - stranded_reports
  description: 'verify_retention: integrate cache census coverage, compare retained
    objects and query latency on fixed workloads, and prepare evidence for the child
    epic land agent.'
proposed_by: bbugyi200.athena.sase-1t3.11
proposed_with: codex/gpt-6-astra
create_time: 2026-11-17 12:30:46
status: done
bead_id: sase-1t3.11.1
---

- **PROMPT:** [prompts/202611/bounded_log_queries.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/bounded_log_queries.md)
- **PARENT:** [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:** [sase-1t3.11.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.11.1.md)

# Bound retained log objects with targeted backend queries

This child epic implements the whole bounded-log-caches phase, `sase-1t3.11`, of
`plan:202611/tui_storm_trust_heap_recovery.md`. It needs separate coding agents because
five sources have different wire formats and correctness contracts, four need Rust
query/binding work, and their combined memory behavior needs independent verification.
Every phase is direct implementation work sized for one coding agent. Dependencies
serialize the migrations in census order and avoid concurrent changes to shared bindings
and revision pins.

## Evidence and existing work

Planning inspected sase `487825956a` and sase-core
`d20108657804cfc257342b0bfb4617ea8bc61a0f`, also the current `sase-core-revision.txt`.
The assigned bead was in progress with no prior attempt notes. Its target caches are
still present; the completed work in `sase-1t3.9` is measurement tooling and census
evidence, not these migrations.

Read the parent plan and phase-9 notes through `sase artifact read` and
`sase bead read`. Phase 9's post-storm samples recorded:

| Owner                 |                About 10 minutes |                                  About 2 hours | Current anchor                             |
| --------------------- | ------------------------------: | ---------------------------------------------: | ------------------------------------------ |
| Artifact-file rows    |                     47,466 live |                    47,658 live; 47,625 indexed | `src/sase/core/artifact_file_explicit.py`  |
| Memory-read events    |                      6,262 live |                         23,457 live and cached | `src/sase/ace/tui/memory_reads.py`         |
| Bead touches          |                      7,332 live | 16,307 live; named snapshot cache reports zero | `src/sase/ace/tui/_bead_touches_loader.py` |
| Artifact-read events  |                   13,566 cached |              27,587 cached across two projects | `src/sase/ace/tui/artifact_reads.py`       |
| Stranded issue parses | Earlier research estimated 8.5k |        Not independently attributed by phase 9 | `src/sase/bead/stranded_facts.py`          |

The 2-hour run had RSS 3.65 GB, 13.04 million allocated blocks, and gen-2 p95 2.03 s.
Those are whole-app observations, not bytes attributable solely to this phase. The
10-minute and 2-hour samples came from separate instances; do not treat their delta as a
controlled growth experiment.

Current mechanisms and useful prior work:

- `_artifact_file_index_cache` keeps up to 32 paths, each containing every parsed
  `ArtifactFile`. `list_indexed_artifact_files()` filters only after that allocation.
  Existing Rust `query_artifact_files()` and `query_artifact_context()` also start by
  loading the entire file. A targeted API name alone does not prove streaming.
- The memory/artifact lanes each keep eight whole-project snapshots plus two 128-entry
  result caches, whose values are capped at 50 rows. Only the result caches are an
  appropriate long-lived owner.
- `query_touches_for_agent()` explicitly calls `query_touch_index()` without actors.
  Rust `bead::touch_index::query_bead_touches()` accepts exact actors but parses the
  full JSON index before filtering. The TUI also retains legacy bead-view snapshots.
- `load_stranded_report()` collects candidates once for facts and again for titles.
  Current Rust `bead::read::list_issues()` already uses the SQLite read model when
  available. Do not implement a second bead index based on the older research.
- `BoundedLRUCache` and deferred stranded-report refresh already exist. Preserve them
  where they serve bounded presentation state.

## Shared implementation contract

1. Read the required `ace.md`, `tui.md`, `tui_perf.md`, and `lint_and_test.md` memories
   through `/sase_memory_read`. Open linked `sase-core` through `/sase_repo` and read
   its `AGENTS.md`. Paths below are relative to their named repository.
2. Shared parsing, filtering, ordering, and bounded query behavior belongs in
   `sase_core`, with typed requests/results and registered PyO3 bindings. Python owns
   host path discovery, typed adapters, and TUI presentation labels. Keep existing
   public readers available; removing an internal snapshot is not permission to change
   CLI results, audit writes, or persistent source formats.
3. Prefer streamed reads of the existing source formats to a new index subsystem. Filter
   before constructing Python objects. Do not retain project-wide decoded rows in Rust
   globals as a substitute. A query may scan the source on a result-cache miss, but it
   must release unrelated decoded rows as it scans; measure the cost. Existing SQLite
   bead queries should be reused. Source-scale duplicate tracking, where necessary for
   exact semantics, must be query-local and measured separately.
4. Cache only the bounded answers needed by the caller. Audit/touch presentation caches
   retain at most their existing 128 entries per cache and 50 rows per entry; do not
   enlarge those bounds. Uncapped public bulk readers may return all requested rows to
   their caller but must not install a global full-source snapshot.
5. Include the source path/project, selector/context identity, query options, and source
   version in cache validity. Preserve the 0.5-second reread throttle. Revalidate on
   append, truncation, deletion/recreation, and atomic replacement, including
   replacement with equal size/mtime. Use file identity/change metadata and explicit
   same-process invalidation where needed; do not hash the whole source per lookup. Do
   not publish data read from one version under a newer version's token.
6. Preserve existing lock and missing/corrupt-source behavior. No refresh, repair, event
   replay, or index write may be introduced by a touch-panel read. Failed queries must
   not poison an unchanged-token cache. Error results should remain distinguishable
   internally from successful empty results where retries need it.
7. Render-only paths remain memory-only. Keep disk work inside existing background
   loading paths and keep the per-agent/context loader APIs stable for `sase-1t3.13`.
   This child epic does not redesign detail scheduling, roster DTOs, startup, GC, or
   executors. If an affected call site needs isolation to meet the existing contract,
   use established pump-free and last-request-wins mechanisms.
8. Follow `docs/contribute/rust-backend.md` for bindings and the source revision pin.
   Declare both changed repositories through `/sase_final`; the host commits core first
   and updates the pin before the sase commit. When consuming an already-landed core
   change, ratchet the pin forward. Never hand-create commits or downgrade a newer pin
   from another phase. Do not edit package versions or release changelogs.
9. Inherited decisions are final: `hotfix_reinstall=yes`, `trust_ack_compat=cutover`,
   `tui_perf_memory=no`. No memory file edits are authorized. The first two decisions
   concern other parent phases and add no reinstall/trust work here. Never restart the
   live TUI or run a global install. Do not delete archives or change retention.
10. Record new findings as `PROPOSED FOLLOW-UP:` notes on the worker's assigned phase;
    do not create task beads. Skill-use and NotificationPlusOne retention were already
    reported by phase 9 and remain outside this child epic. PosixPath/DirEntryInfo work
    is limited to allocations caused by these queries, not unrelated walks.

## Phase artifact_files

Implement in sase-core's `artifact_file` domain and its existing PyO3 binding domain,
then the sase artifact-file adapters.

- Add an additive association query that streams supported index envelopes and returns
  only matching rows. Match the current `artifact_file_helpers.matches_association()`:
  exact artifact directory OR non-null equal project and raw timestamp. A plain
  agent-name filter or the exact-directory-only wait-context query is not equivalent.
- Preserve file order, explicit/default inclusion, schema 1/2 compatibility, optional
  fields, and VCS-backed rows. Characterize the Python reader's tolerance before reusing
  Rust's existing parser; do not silently tighten it on the hot path.
- Route `list_indexed_artifact_files()` and `list_explicit_artifact_files()` through
  that query. Remove the whole-index module cache; preserve the bulk reader API as an
  uncached explicit bulk operation. Keep writer locking, atomic replace, and unknown
  line preservation intact. Add only bounded targeted-result caching if repeated callers
  need it, with an explicit row/byte admission budget for oversized answers.
- Audit direct bulk-reader callers to ensure a frequent TUI path cannot recreate the
  full snapshot. Retention/doctor bulk operations need not be redesigned, and their
  returned tuples must retain current behavior.
- Extend `tests/artifact_file_facade/test_listing.py`, persistence/storage tests, and
  `tests/agent/test_artifact_files_cache.py`; add Rust and binding parity tests. Cover
  association fallback, same-name unrelated runs, malformed/unsupported envelopes,
  VCS-only records, writer invalidation, concurrent reads, and replacement.
- On a fixture with at least 50,000 unrelated rows and a small fixed matching set,
  demonstrate that only the matching Python `ArtifactFile` objects are constructed and
  that no full-index owner survives. Record cold/warm query timing and temporary memory
  before/after. Keep output completeness for agents with many artifacts.

## Phase memory_reads

Introduce a small shared audit-query facility in core, initially for memory reads. Its
input is a source path, an ordered set of exact member selectors (artifact dir and
optional name), a result cap, and the required single-agent/context duplicate policy.
Return selected event data and the matching member ordinal; TUI role text stays in
Python. The artifact-read phase will reuse this traversal/selection contract.

- Stream JSONL and validate the actual memory schemas in
  `src/sase/memory/_read_log_events.py` and `_read_log_models.py`, including legacy
  rows, web/strand fields, batch `resolved_targets`, and `included_targets`. Do not
  alter audit-write attribution or requested-versus-expanded target semantics.
- Preserve artifact-directory precedence: normalize consistently with
  `util/normalized_dirs.py`; events carrying a directory cannot fall back to a name when
  that directory mismatches. A missing directory permits an exact name match. Cover
  symlinked paths and missing path components; do not use string-prefix matching.
- Preserve first-member attribution, stable timestamp-descending order, and ties.
  Context queries deduplicate event IDs after matching, retaining the first matched
  source occurrence; ordinary queries retain their existing duplicate behavior. Apply
  the final cap after matching and duplicate handling, not to a raw log tail. Preserve
  any public `limit` behavior or validate callers before changing it.
- Replace `_memory_reads_snapshot_cache` with targeted queries feeding the existing
  per-agent and context caches. Keep the public loader shape and result labels. The
  unfiltered audit API remains available for explicit history consumers.
- Replace tests that require a global parse-once snapshot with bounded-result and
  revalidation assertions. Extend `test_memory_read_log.py` and the memory loader,
  context, cache, and display suites. Test duplicate IDs straddling the cap, unordered
  timestamps, equal timestamps, unrelated projects, name collisions, malformed rows, an
  older matching event beyond a long unrelated tail, and all source mutations.
- Measure fixed selected contexts with 25,000 and 100,000 unrelated events. Python event
  construction and retained rows must depend on returned matches/cache bounds, not
  source length. Record scan latency and transient backend allocations too.

## Phase bead_touches

First resolve the census discrepancy: inspect ownership in a short-lived isolated query
workload and, if needed, a separate TUI instance. A zero counter from the named cache is
not evidence that 16,307 live touches are fixed. Look for module aliases, derived
entries, retained workers, and facade consumers; record the actual referrer.

- Extend the existing Rust touch query to accept the precise agent-identity selectors
  required by `touch_matches_agent()`, including owned/global names and bare legacy
  actors. Reuse core identity normalization. Match only exact normalized identities;
  preserve whitespace handling and reject unrelated dotted names or email actors.
- Preserve member precedence and the trailing unlabeled session-container alias.
  Ordinary non-root turns must not inherit that alias. Keep CLI
  `query_touches_for_agent()` semantics and existing touch wire metadata, including
  creation reason, note preview/count, verbs, title, and close/reopen information.
- Read only the touch-query fields of the JSON index and select during decoding; avoid
  retaining all stream signatures and touches merely to answer one actor query. Preserve
  generation/schema reporting and whole-document corruption behavior: do not return
  partial rows from a truncated or wrong-schema index.
- Query legacy `bead_views.jsonl` for the same selectors without a whole-project
  snapshot. Fold durable and viewed rows in the established order, preserve counts and
  tie behavior, then apply the display cap. Do not cap either source prematurely when
  doing so would change the final merged view.
- Replace `_bead_touches_snapshot_cache` and `_bead_views_snapshot_cache`, ensuring
  result-cache hits occur before expensive source queries. Update the real owner
  identified above if it is another consumer of this same source.
- Extend core touch-query tests, `tests/core/test_bead_touch_index_facade.py`, CLI
  touched tests, and the TUI touch loader/session-alias/merge suites. Cover invalid
  indexes, actor collisions, all row roles, retries, source replacement, and views.
- Measure on at least 20,000 unrelated touches plus a fixed selected context. Keep only
  the selected results in Python and identify remaining touch owners explicitly.

## Phase artifact_reads

- Reuse the audit query facility from memory_reads, adding artifact event validation
  matching `src/sase/artifact_read_log.py`: schema, required strings, boolean
  `recorded_link`, optional directory, and normalized optional `resolved_path`.
- Switch `src/sase/ace/tui/artifact_reads.py` to targeted results and remove
  `_artifact_reads_snapshot_cache`. Preserve the single/context duplicate distinction,
  member ordinal-to-label conversion, timestamp ordering, and the 50-row cap. Keep
  `ArtifactReadRefSpec` and click/open behavior intact; querying must not resolve refs,
  publish read edges, or perform audited artifact reads as a side effect.
- Preserve the unfiltered public reader for explicit callers; it must not install a
  permanent full-source cache. Keep source errors and writer locking compatible.
- Extend `tests/test_artifact_read_log.py`, artifact loader/cache/context tests, and
  `tests/ace/tui/widgets/test_agent_artifact_reads.py`. Run both memory and artifact
  query tests when changing their shared facility. Use the same adversarial selector,
  duplicate, limit, mutation, and retention fixtures as memory_reads.
- Measure 30,000 and 100,000 unrelated rows with a fixed selected context, including
  multiple projects. Record bounded retained event counts plus cold/warm latency.

## Phase stranded_reports

Remeasure before changing the backend. The current implementation already has indexed
status/type filtering; this is not an instruction to replace the bead store.

- Trace one `load_stranded_report()` and distinguish Python issue objects, Rust query
  rows, read-model freshness/replay work, and cached report state. Record whether the
  old whole-store parse still exists on warm current stores. If already gone, explicitly
  mark that part satisfied and narrow the patch to the remaining duplicate collection
  and unnecessary hydration.
- Collect candidate facts and titles from the same candidate/context snapshot rather
  than running `_candidate_and_context_issues()` twice. Preserve existing helper APIs
  used by callers, but the combined report path should collect once.
- Where measurement confirms unnecessary hydration, add a targeted core projection over
  the existing read model for candidate beads and necessary child-plan context. Select
  parent relationships in core instead of fetching every plan into Python. Return only
  the fields consumed by the existing stranded reducer/fact collector; avoid unused
  descriptions, note histories, and artifact links. Keep replay/error behavior for
  unsupported or unhealthy stores and keep it off the UI thread.
- Preserve candidate membership: in-progress eligible tasks/phases/epics and open
  eligible beads with completion failures. Keep child-epic blocking, settle grace,
  retained-workspace evidence, finalizers, delivered handoffs, successors, monitor
  ownership, and all other liveness semantics unchanged. Python host probes stay in
  Python. The `bead_ids` option must still include required dependency context.
- Keep `_STRANDED_REPORT_CACHE` a cache of final report data. Release intermediate
  issue/session graphs after collection; repair any demonstrated stale snapshot owner
  without changing the TTL or refresh scheduling contract.
- Extend stranded collector batch/status/recovery/staged-handoff tests and
  `tests/ace/tui/models/test_agent_stranded.py`. Prove facts and titles use one snapshot
  and remain coherent if the store changes between would-be old queries. With many
  irrelevant closed beads and a fixed candidate set, Python materialization should scale
  with candidates and required context, not all history.

## Phase verify_retention

This phase integrates and measures; it does not absorb unfinished migrations.

- Update `src/sase/ace/tui/util/_gc_policy_census.py` and its tests to report actual
  bounded result owners after old caches disappear. Include the touch owner found above.
  Missing old symbols must not falsely report a successful memory reduction. Keep
  census-only inspection out of normal render and heartbeat paths.
- Run a reproducible fixed workload against the base and combined implementation: small
  and large source histories, the same selected agents/session members, repeated
  selection, more keys/projects than cache capacity, append, and source replacement.
  Release results and collect garbage before counting retained objects. Compare
  `ArtifactFile`, `MemoryReadEvent`, `ArtifactReadEvent`, `BeadTouch`, and stranded
  candidate/report graphs. Do not rely on `gc.get_objects()` alone for untracked types;
  add explicit cache row totals and scoped construction counters.
- Require no project-wide Python source snapshots, bounded result cache occupancy, and
  Python conversion proportional to matching/returned rows. Record transient Rust peak
  memory so moving the same retained graph into Rust cannot pass unnoticed.
- Record cold, warm-cache-hit, and post-invalidation query latency with data sizes,
  Python/core/Textual versions, revisions, host load, and retained counts. Diagnose
  regressions on repeated context switching; do not trade memory savings for repeated
  source scans on each paint. Verify existing render-only APIs make no stat/glob/read
  calls. If a performance target still misses, record the measured remaining cost.
- Use a separate TUI instance for an approximately 10-minute integration census with
  source lanes exercised. Record RSS, allocated blocks, gen-2 pauses, cache totals, and
  executor gauges. Coordinate longer waits through `/sase_monitor`; do not restart the
  live TUI. The parent `sase-1t3.14` still owns the long end-to-end soak.
- Put the before/after table and reproducible commands in this phase's notes, with a
  concise evidence summary on `sase-1t3.11` for the parent interaction and soak workers.

## Verification and landing

Each implementation phase adds the focused behavioral and scale tests described above,
then runs the standard checks in every changed repository. Read current verification
memory before running them. Use `sase tool run check` for the guarded `just check`
recipe; never bare cargo in core and never `just check-full` unless explicitly
requested. Use `/sase_monitor` for long checks/builds and wait for each handoff command
itself to exit. Formatting precedes final verification. Bindings must be exercised with
the locally built core, not an older installed extension. No screenshot changes are
expected; if rendered behavior does change, inspect targeted screenshot evidence under
the established workflow rather than silently accepting new goldens.

Before closing its own phase, each worker runs `sase bead epic-symbols <assigned-id>`
and resolves or re-keys all exemptions to a still-open owner. Record reproducible
clean-base check failures as `PROPOSED FOLLOW-UP:` notes and follow the launch rule that
identical base failures do not leave completed phases open. Never weaken tests or
classify a failure as unrelated without evidence.

The child epic land agent verifies all six phases, combined tests, revision pins,
cache-owner evidence, and the final object/latency table. It owns completing
`sase-1t3.11`: run `sase bead epic-symbols sase-1t3.11`, resolve any remaining entries,
then close that bead with the verification evidence when its child work is complete
under the host's land workflow. Phase workers close only their own assigned beads; this
paragraph is preparation for the land agent, not ancestor-close authorization. Do not
close `sase-1t3` or any higher ancestor. End all root turns through `/sase_final`.
