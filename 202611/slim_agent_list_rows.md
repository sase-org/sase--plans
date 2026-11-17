---
tier: epic
title: Slim agent list rows and bound retained snapshots
goal: Complete sase-1t3.10 with compact Rust list projections, selected-only detail
  hydration, and one retained snapshot generation without changing visible agent behavior.
parent_bead: sase-1t3.10
phases:
- id: slim-core-contract
  title: Add compact list contracts and Rust bindings
  size: medium
  depends_on: []
  description: 'slim-core-contract: inventory list-required fields and add additive
    compact agent list DTOs, index and source-scan projection entry points, and PyO3
    bindings in sase-core. Preserve selection, ordering, clan context, and completeness
    metadata; prove parity and absence of nested full records.'
- id: scalar-row-adapters
  title: Decode compact rows and construct ordinary agents
  size: medium
  depends_on:
  - slim-core-contract
  description: 'scalar-row-adapters: advance the sase core source pin, add thin Python
    compact-wire adapters, and construct done and running agent rows directly from
    list facts with metadata and status parity. Keep production acquisition unchanged
    until the workflow cutover phase completes the consumer set.'
- id: workflow-list-cutover
  title: Consume slim rows throughout TUI loading
  size: medium
  depends_on:
  - scalar-row-adapters
  description: 'workflow-list-cutover: adapt workflow and prompt-step rows, switch
    TUI indexed, fallback, full-history, and delta acquisition to compact projections,
    and extend selected-record hydration for all newly deferred fields. Preserve grouping,
    live filters, row patching, and authoritative load-state semantics.'
- id: bounded-retention
  title: Bound snapshot and detail retention and measure the result
  size: medium
  depends_on:
  - workflow-list-cutover
  description: 'bounded-retention: retain one compact artifact snapshot generation,
    release obsolete wire and detail payloads, add ownership regression tests, and
    record comparable load and heap measurements. Prepare evidence for this child
    epic''s land agent to complete sase-1t3.10; do not close an ancestor as a phase
    worker.'
proposed_by: bbugyi200.athena.sase-1t3.10
proposed_with: codex/gpt-6-astra
create_time: 2026-11-17 12:28:06
status: wip
bead_id: sase-1t3.10.1
---

- **PROMPT:** [prompts/202611/slim_agent_list_rows.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/slim_agent_list_rows.md)
- **PARENT:** [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:** [sase-1t3.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.10.1.md)

# Slim agent list rows and bound retained snapshots

## Scope and inherited decisions

This child epic implements the `slim-list-rows` section of
`plan:202611/tui_storm_trust_heap_recovery.md`, assigned as `sase-1t3.10`. Read that
artifact and the current phase notes with audited commands before work. The phase was in
progress with no prior implementation notes at planning time. The inspected sase
revision was `487825956a`; the opened sase-core revision was `d2010865`. The compact
contract has not landed in those trees. Check newer history and reuse any landed work
before implementing.

The parent decisions are final: `hotfix_reinstall=yes`, `trust_ack_compat=cutover`, and
`tui_perf_memory=no`. This child epic changes no memory notes. It does not repeat the
completed storm or trust work. Global installs and restarting the live TUI remain
human-only actions.

Four sequential medium phases separate the cross-language contract, ordinary-row
conversion, workflow cutover, and ownership validation. This avoids making one coder
both redesign the wire and migrate every consumer and lifetime in the same turn.
Intermediate commits keep existing consumers working; the production switch happens only
after all TUI row kinds have compact constructors.

## Evidence and code anchors

The completed prerequisite `sase-1t3.9` records the post-storm census. At approximately
two hours, `ARTIFACT_SNAPSHOT_CACHE` held 8/8 entries and 812 records; the process had
378,739 dicts, 684,723 lists, 277,559 paths, 2,816 Agents, RSS 3.65 GB, and gen-2 pause
p95 2.03 s. These are whole-process figures, not bytes attributable solely to this
phase. The wire watchlist uses the obsolete defining-module name:
`sase.core.agent_scan_wire.AgentArtifactRecordWire` reports zero, while the actual
`sase.core.agent_scan_wire_records.AgentArtifactRecordWire` appeared with 1,217
transient objects. Use the actual class/module when measuring.

Current implementation:

- In sase-core, `crates/sase_core/src/agent_scan/index/query.rs` builds a full
  `AgentArtifactScanWire`. `project_record_for_list` clears linked repos and only
  removes `_raw` and `_data` from outputs; it preserves nested marker trees. The
  existing query also owns source reconciliation, candidate selection, windowing, clan
  context, and completeness claims. Preserve that machinery.
- `crates/sase_core_py/src/agent_scan/mod.rs` exposes the index query and serializes its
  full-shaped result into Python objects.
- In sase, `src/sase/core/agent_scan_wire_conversion.py::_record_from_dict` constructs
  nested `AgentArtifactRecordWire` objects even for `record_shape=list`.
  `agent_scan_index_queries.py` and `agent_scan_facade.py` are the binding seams.
- `src/sase/ace/tui/models/_agent_loader_artifact_index.py` requests list projection but
  caches those full-shaped Python snapshots. Its sibling
  `_agent_loader_artifact_cache.py` has `max_entries=8`, index/WAL signature checks,
  exact query keys, and a same-key lock-busy stale-read path.
- `_agent_loader_sources.py`, `_agent_loader_tiered.py`, and
  `_loaders/{_done_snapshot_loaders,_running_loaders,_workflow_snapshot_loaders}.py`
  turn snapshots into mutable Agents. `_meta_enrichment_wire.py` applies substantial
  metadata used by status overrides, action availability, and filters.
- `models/_projected_record.py` already loads one artifact directory for detail. Today
  it restores only step output and linked repos. The pump-free scheduling is in
  `actions/agents/_display_detail_render.py`. Full payloads applied directly to Agents
  can remain reachable after selection moves.
- The recent per-load memos in `models/_loader_memos.py`, the project identity load
  scope, and lazy instruction-bundle files must survive the migration.

The census also names `AgentCatalogLinkFacets` and Rich `FIFOCache`. The former's
current owner is the archive/catalog path, not this artifact snapshot cache. Track their
before/after counts, but do not invent a causal link or expand this child epic into an
archive-catalog or Rich-cache rewrite. Unresolved independent owners become
`PROPOSED FOLLOW-UP:` notes on the assigned phase for the outer land agent.

## Contract and invariants

Use an additive compact contract, leaving full-record APIs available for explicit detail
and non-TUI callers. Suggested names are `AgentArtifactListRecordWire` and
`AgentArtifactListSnapshotWire`; use the repository's naming conventions. The new
snapshot carries existing stats, window, completeness, and clan-context metadata. No
SQLite schema migration is required merely to change the returned representation.

List records contain the facts needed for list paint, filters, grouping, identity,
runtime aggregation, status overrides, deduplication, and row-local actions. Inventory
actual field reads before defining the wire. Cover at least:

- Project/workflow/artifact paths, timestamp, source owner/machine, names and aliases,
  PID/process identity, workspace ownership, and marker presence versus parsed state.
- Status/outcome and finalizer/retry/hijack/monitor/gate/plan/wait indicators; queue and
  waiting summaries; times and runtime intervals consumed by the list.
- Session/member/parent identity, tribe/clan generation and summaries, workflow and
  prompt-step identity/order/type, hidden flags, and action capability facts.
- Small query/row summaries, model/provider fields, change indicators, output-variable
  facts that filters actually inspect, and authoritative handles for deferred content.

Prefer scalar fields and small typed sequences. Workflow steps may use a sequence of
compact typed step rows; they must not carry full workflow states or arbitrary outputs.
Preserve missing/null distinctions and source precedence. Detail-only outputs,
tracebacks, large prompts, linked-repository structures, media/file lists, and
instruction manifests are deferred where they are not needed for list behavior. Do not
strip data needed by a filter and then hydrate every row to recover it: add the minimal
shared summary in Rust or retain the established indexed query path.

Python converts compact values directly into Agents. Reconstructing `AgentMetaWire`,
`DoneMarkerWire`, full workflow trees, or generic nested proxy dictionaries defeats the
change. Shared normalization and domain decisions belong in Rust; Python keeps
presentation and local integration glue. A temporary full Rust record during an existing
index read is acceptable; full records must not cross the new list binding or remain in
its response. Reuse selection logic rather than running a second query.

After a load returns, keep only Agents, load diagnostics, and at most one immutable
compact cached generation. No raw binding dict, `AgentArtifactScanWire`, or full record
tree is retained by the roster, a completed worker closure, or a cache. Do not share
cached mutable Agents between generations.

All keystroke and paint paths remain free of disk I/O and parsing. Keep highlight
immediate, details debounced, refreshes coalesced, navigation gates honored, and
incremental `patch_row` behavior intact. Re-check selected identity and current row
version after every await before applying detail results.

## Phase slim-core-contract

Open sase-core with `/sase_repo` and read its `AGENTS.md`. Work in its printed path.

1. Inventory field consumers in the Python loaders, live query corpus, grouping, status
   overrides, and row-local action facts. Record a concise field-to-consumer map and
   explicit deferred-field list as this phase's bead note so downstream coders do not
   independently choose incompatible projections.
2. Add the compact Rust wire types and a pure projection from existing record facts.
   Keep source precedence explicit, including inconsistent and absent markers.
3. Expose compact index queries using the existing selection/revalidation pipeline. Add
   compact responses for source scans and targeted directory scans by projecting before
   PyO3 serialization. These are needed for cold/missing-index fallback and delta loads;
   do not leave a hidden full-Python decode path there.
4. Add and register the PyO3 functions in the agent-scan domain, with round-trip tests.
   Keep existing full APIs and their defaults compatible. If a shared schema number must
   change, update all required copies and fixtures; prefer a distinct additive wire that
   avoids an unrelated database rebuild.
5. Test compact/full parity for row facts, identity/order, window limits, candidate
   filters, hidden rows, clan context, missing markers, and completeness metadata.
   Include large detail payloads and assert the compact serialized payload excludes
   them. Cover workflow steps and session/gate/monitor members in the contract tests.

Pass the core gate through `sase tool run check` from that checkout. Let host
finalization commit the core and binding changes. Record the resulting core revision or
enough host evidence for the next phase to resolve it. No production Python consumer may
call the new bindings before its source pin contains this commit.

## Phase scalar-row-adapters

1. Resolve the landed core contract commit and move `sase-core-revision.txt` forward to
   a revision containing it, following `docs/contribute/rust-backend.md` and
   `just ratchet-core-revision`. Preserve any newer pin moved by another phase. Do not
   edit the release-owned PyPI dependency pin or versions.
2. Add frozen/slotted Python compact-wire records and thin conversion/facade methods.
   Preserve bounded index-lock behavior and error signaling. Decode compact payloads
   without calling `_record_from_dict` or building full marker dataclasses.
3. Implement direct compact constructors for ordinary done and running/home rows, plus
   the metadata adapter they share. Preserve local RUNNING-claim merging,
   process-identity handling, hidden/no-op rules, finalizer and status overrides,
   retries, plan associations, waits, imported owners, and workspace-retention facts.
   Reuse existing presentation helpers and per-load memo scopes.
4. Test against the current full-path fixtures and compare all list/filter/action facts,
   allowing only explicitly deferred detail fields to differ. Add a test that makes the
   full-record Python decoder fail if the compact constructor uses it.

Keep live TUI acquisition on the old API until the next phase adds the remaining
consumers. Tests exercise the new adapters. If integration-only public symbols need
temporary lint exemptions, follow `symvision.md` and key them to this child epic or the
still-open consuming phase, never a phase about to close. The next phase must remove the
exemptions. Run sase's required checks with the matching local core build.

## Phase workflow-list-cutover

1. Construct workflow parent rows and prompt-step children directly from the compact
   contract. Preserve appears-as-agent and anonymous workflows, nested step parents,
   workflow child identity, session/gate/monitor member suppression, represented clan
   context, and deduplication precedence with ordinary rows.
2. Switch TUI acquisition and source dispatch to the compact APIs for cached Tier 1,
   revalidated loads, authoritative full history, missing/stale-index bounded fallback,
   explicit targeted deltas, and any live-plan loader using the same path. Keep all
   window/truncation/completeness/load-state semantics and fallback bounds. Do not
   change the CLI/mobile full API contract incidentally; test shared consumers such as
   `agent/listing_snapshot.py` if signatures or shared helpers change.
3. Extend `_projected_record.py` to restore every newly deferred detail field, using the
   existing exact-directory full-record lookup. Preserve workflow-step matching, output
   merge behavior, missing/deleted record handling, and retryable lock-busy behavior.
   Never hydrate rows simply to render/filter/group them.
4. Ensure selection hydration stays off the pump, coalesces duplicate requests for the
   same current row, and ignores stale selection or replacement-row completions. Keep
   explicit export/dismiss operations that need full data on their worker paths.
5. Remove temporary integration exemptions and unreachable migration helpers. Retain
   full APIs with legitimate callers. Cover end-to-end query, grouping, selection,
   `patch_row`, bounded-to-complete merge, and load-memo parity in tests.

Relevant suites include `tests/test_agent_loader_agents_list_projection.py`,
`test_agent_loader_query_window*.py`, `test_agent_loader_incomplete_history*.py`, the
workflow/status-override loader suites, `tests/ace/tui/models/test_projected_record.py`,
`test_agent_loader_memos.py`, and the incremental AgentList tests. Run meaningful
focused suites plus the standard gate, not a new exhaustive verification recipe.

## Phase bounded-retention

1. Replace the eight-entry artifact snapshot cache with one retained compact generation,
   including across changes in filter, root, window, and scan options. Keep exact keys
   and index/WAL signature checks: a same-key lock-busy read may return the one stale
   generation; a different key may not. Preserve mutation refusal, cache diagnostics,
   and revalidate/full-history bypass behavior. A superseded in-flight load must not
   republish an obsolete generation as current.
2. Audit escaping references after Agent construction and release obsolete raw/wire
   values. Update cache diagnostics/census accessors to describe the new representation.
   Correct the old wire-class watchlist entry if still present; do not duplicate a
   correction already landed by another phase.
3. Prevent selected-detail payloads from accumulating across navigation. Use a bounded
   selected-detail owner or release only the deferred fields on deselection; retain the
   compact facts required for list/filter/action behavior. A replacement roster,
   canceled hydration, tab change, or delayed worker must not leave full records or
   previous-generation snapshots reachable. Full records are transient while their
   needed detail values are applied; never retain the full record itself.
4. Add behavioral ownership tests using weak references where available, collectable
   sentinels, and retained payload size/count assertions. Cycle at least twenty
   filter/window/generation changes and selections with large outputs. Assert one cache
   entry, bounded selected detail, collectible old snapshots, no full Python wire decode
   on list paths, and unchanged list facts after detail eviction.
5. Capture an identical fixture workload before/after using the parent/base revision and
   completed tree. Record row count, binding payload bytes, Python allocations or
   retained object counts, loader index/decode times, and cache statistics. Use the
   actual defining-module class for wire counts. A warm compact list load should
   allocate zero full Python artifact records; detail loads should scale with selected
   identities rather than archive size. Demonstrate that retained graphs do not grow
   with the number of refreshes.
6. Run a separate-instance real-data smoke/census long enough for warm loading and
   repeated selection/refresh, recording startup, RSS, allocated blocks, gen-2 pauses if
   sampled, and j/k responsiveness. Attribute changes conservatively. The outer
   `sase-1t3.14` owns the full one-hour integrated soak; do not repeat its entire job or
   require this phase alone to meet the outer epic's total RSS target.

Use `/sase_monitor` for long builds, verification, and soak waits, preparing all
fixtures and commands before handing off. Never attach to or restart the user's live
TUI. Record measurement method, revisions, environment, and results on this phase and
make them available to `sase-1t3.10`'s land agent.

## Verification, follow-ups, and landing

Read `ace.md`, `tui.md`, `tui_perf.md`, and `lint_and_test.md` through audited memory
reads before implementation. Each changed checkout passes its `sase tool run check`
gate. Core targeted runs use its `just` wrappers, never bare cargo. Prepare a local core
build using the documented workflow; do not use a global install to test. Use existing
visual tests for parity and read screenshot guidance before capture. Unexpected rendered
differences are regressions to investigate, not automatic golden updates. Do not run
`just check-full` without explicit authorization naming it.

Phase workers close only their own assigned child phase. Before closing, run
`sase bead epic-symbols <their-phase-id>` and resolve or re-key every remaining
exemption to a genuinely open consumer. They never close `sase-1t3.10`, this child epic,
or `sase-1t3`. No worker creates follow-up task beads; use `PROPOSED FOLLOW-UP:` notes
on the assigned phase. A failure proven identical on the clean base is recorded with its
evidence and existing tracking ID where available; it does not leave a completed phase
stranded open.

The child epic's host-assigned land agent verifies the combined result and pin
containment, consolidates the measurements, and checks
`sase bead epic-symbols sase-1t3.10`. Once this child epic's phases and its own
completion obligations are settled, that land agent completes the delegated original
phase with `sase bead close sase-1t3.10 --note "<verified behavior and evidence>"`. This
is the user's explicit child-epic delegation for this original phase, not permission for
a child phase worker to close an ancestor. Leave the outer epic `sase-1t3` and every
ancestor plan open for their own land agents. If descendant closure ordering is
enforced, settle the child epic first through its authorized land lifecycle; never
force-close descendants or manually change statuses.
