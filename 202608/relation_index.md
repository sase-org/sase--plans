---
tier: tale
title: The host-owned relation index and its built-in sources
goal:
  Every Artifacts pane can be asked for its relations without a widget, because one
  immutable RelationIndex per snapshot computes hierarchy, family, and link edges from
  the contract's declarations with derived inverses, cycle detection, and dangling
  diagnostics — and the Patch relation goldens are unchanged.
size: medium
proposed_by: bbugyi200.athena.sase-m6.7.1.2
bead: sase-m6.7.1.2
create_time: 2026-08-16 03:34:48
status: wip
---

- **PARENT:**
  [202608/artifacts_relations_and_grouping.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifacts_relations_and_grouping.md)
- **BEAD:**
  [sase-m6.7.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.7.1.2.md)

# Plan: The host-owned relation index and its built-in sources

Implements phase `index` (bead `sase-m6.7.1.2`) of epic `sase-m6.7.1`
(`plan:202608/artifacts_relations_and_grouping.md`). The `vocabulary` phase
(`sase-m6.7.1.1`, landed) gave the Artifacts contract real `PaneRelationDecl` /
`PaneGroupingDecl` records and filled them for every built-in pane. Nothing computes an
edge yet. This phase computes every edge once per snapshot, off the event loop, with no
provider code — and no widget reads the result (that is `panel`, `sase-m6.7.1.3`).

Read `sase/memory/tui_perf.md` before starting
(`sase memory read tui_perf.md -r "..."`). Run `just install` first — workspaces are
ephemeral and dependencies drift.

## Grounding

Verified in a clean checkout at `2abe188aa`. Line numbers are from that tree.

| Fact                                                                         | Evidence                                                                                                                                                                                                                     |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Relation/grouping records are real and populated for all five panes          | `_artifact_tab_model.py:126-186`; `_artifact_tab_contract_adapters.py:67-474`                                                                                                                                                |
| `ArtifactEntryTarget` is pure data but lives in a Textual module             | `widgets/artifacts/entry_navigation.py:31-89` imports `textual.widgets.OptionList` and `rich.text` at line 10-12                                                                                                             |
| `_artifact_tab_model` is deliberately Textual-free                           | module docstring, `_artifact_tab_model.py:1-6`                                                                                                                                                                               |
| Patch relation semantics live in the widget today                            | `_find_ancestors` / `_find_siblings` (`widgets/ancestors_children_panel.py:165-255`)                                                                                                                                         |
| `PatchGraphIndex` already holds every Patch primitive                        | `name_map`, `children_by_parent`, `siblings_by_base_name` (`models/patch_graph_index.py:22-42`)                                                                                                                              |
| The Patch oracle probes the widget, not an index                             | `_RelationshipProbe(AncestorsChildrenPanel)` (`tests/ace/tui/artifacts_contract/test_relation_goldens.py:16-26`)                                                                                                             |
| Row identities differ per pane                                               | `patch_entry.py:10-15`, `beads_list.py:44-48`, `files_list.py:37`, `commits_timeline.py:26-32`, `plans_list.py:43-56`                                                                                                        |
| Beads hierarchy data is `Issue.parent_id`; `phases_by_epic` is built from it | `beads_data.py:112-129`; `bead/model.py:184`                                                                                                                                                                                 |
| Beads dependency + plan link data is already on the snapshot                 | `Issue.dependencies` (`bead/model.py:206`), `plan_links` holds a **resolved absolute path** (`beads_data_sources.py:102-126`)                                                                                                |
| Files versions never leave their row                                         | `FileVersion.logical_id == LogicalFile.logical_id` always (`files_data.py:222-242`, `330-333`)                                                                                                                               |
| Plans de-duplicates a path across lifecycle stages                           | `archive` drops anything in `active_by_path` (`plans_data.py:220-224`)                                                                                                                                                       |
| Plans↔Beads link data is already on the snapshot                             | `PlansSnapshot.bead_plan_links` (`plans_data_models.py:81`), `BeadPlanLink.bead_type` (`bead_plan_links.py:13-27`)                                                                                                           |
| Commit parents and SASE footer tags are already available                    | `VcsCommitWire.parent_ids` (`core/vcs_log_wire.py:93`); `SASE_PATCH` is an emitted footer tag                                                                                                                                |
| `AggregatedCommitWire.repo` is a display label, not a project key            | `core/vcs_log_wire.py:106-116`; `Patch.project_name` is the project **directory key** (`ace/patch/models/patch.py:164-173`)                                                                                                  |
| A display ref can be mapped back to a project key                            | `ProjectRefDisplaySnapshot.project_key_for_ref` (`project_display_names.py:188`), loaded by `load_project_ref_display_snapshot`                                                                                              |
| Provider documents arrive as archive rows with frontmatter                   | `plans_data.py:183-206`; `Plan.frontmatter` / `relpath` / `path` (`plan_search/model.py:15-38`)                                                                                                                              |
| Every pane already has exactly one worker pass that builds its rows          | `_prepare_patch_load_from_disk` (`actions/patch/_loading.py:166-177`), `_build_snapshot` on the three snapshot panes, `_collect_payload` (`commits_collection.py:189-209`)                                                   |
| Each pane's worker result is already a frozen carrier dataclass              | `_PreparedPatchLoad`, `_FilesSnapshotResult` (`files_pane.py:51-55`), `_BeadsSnapshotResult` (`beads_pane.py:42`), `_PlansSnapshotResult` (`plans_pane.py:55-59`), `CommitCollectionPayload` (`commits_collection.py:61-65`) |
| Stitches already caches a per-result index keyed by `id(result)`             | `_authoritative_query_indexes` (`commits_collection.py:308-341`)                                                                                                                                                             |
| Python-owned graph work has an established `sase.core` shape                 | `core/graph_index_facade.py:1-28`                                                                                                                                                                                            |

## Decisions

**D1. `ArtifactEntryTarget` moves to a Textual-free module; every import path keeps
working.** The epic requires `src/sase/core/artifact_relations.py` to import no Textual
widget, and requires edges to carry `ArtifactEntryTarget` from day one. Those are only
compatible if the target record itself is Textual-free. Move the dataclass (and
`_TOKEN_VERSION`, `_TOKEN_DELIMITER`, `_LEGACY_KIND_TO_PANE_ID`) verbatim into
`src/sase/core/artifact_entry_target.py`, and have `entry_navigation.py` import and
re-export it, keeping it in that module's `__all__`. All ~205 existing
`from .entry_navigation import ArtifactEntryTarget` call sites are untouched. Do not
change the token encoding or `from_legacy` behavior.

**D2. Dangling is a same-pane verdict only.** A `RelationIndex` is built from exactly
one pane's snapshot, so it cannot know whether another pane holds a row. An edge whose
`target.pane_id != index.pane_id` is never marked dangling and never produces a
diagnostic. Same-pane edges are checked against the source's declared node universe.

**D3. Each source declares its own node universe.** `known_targets()` is the set of
identities that source's snapshot contains — not the pane widget's `entry_targets()`,
which the index must not depend on. For Files this includes version-level targets (see
D6).

**D4. One direction per relation pair; the index derives the other, except where the
source already owns both orders.** For each declared relation with `inverse` set, if no
edge was emitted under the inverse's name, the index materializes the reversed edge set
under that name (kind/label inherited from the inverse's declaration when it is also
declared, otherwise from the forward declaration). A relation with `directed=False` gets
its symmetric reverse materialized under the _same_ name. Patches deliberately emits
both `ancestors` and `children` directly, because `PatchGraphIndex.children_by_parent`
is the authoritative child ordering and today's goldens pin it; Beads emits only
`parent` and lets `children` derive, which is the epic plan's worked example.

**D5. Stitches declares `patches`, not `plans`.** `vocabulary` gave Stitches a `plans`
link to `ref:plan` sourced from a non-existent `stitch_plan_links`; the epic plan's
`index` section mandates "a cross-pane link commit→patch targeting `pane_id="patches"`",
and the oracle requirement names "the real Stitches→Patches ... edges". A commit's
`SASE_PLAN` tag holds a plan _reference_, and resolving it to the path the Plans pane
uses as row identity needs per-project plan-root I/O that the commits worker does not
have — so a `plans` edge could only be emitted with a knowingly wrong identity. Replace
the Stitches `plans` declaration with `patches` (`kind=LINK`,
`source="stitch_patch_tag"`, `target_pane="patches"`, `inverse="stitches"`,
`directed=True`, `transitive=False`) so the declaration and the source agree. Record the
dropped commit→plan edge as a `PROPOSED FOLLOW-UP`.

**D6. Files' family is row→version, and versions join the node universe.** `LogicalFile`
is the selectable row; its versions are the family the `versions` declaration names, and
they never belong to another row. Emitting version↔version edges would make
`edges_for(<row>)` empty — useless to `panel`. Emit `row → (logical_id, version_id)`
edges; `directed=False` gives the reverse for free (D4). Include those version targets
in `known_targets()` so they are not spuriously dangling.

**D7. Provider document panes gain one host-owned `bundle` family declaration.** The
filename family rule must work "with no sidecar change", but the
`relations_from_declared_edges` rule only turns `RELATIONS` ON when a declaration
survives — so a host-owned rule that emits edges for an undeclared relation would be a
contract lie. In `compile_provider_contract`, for a non-degraded pane whose
`kind != "plan"`, append one host-owned
`PaneRelationDecl(name="bundle", kind=FAMILY, label="Bundle", source="document_filename_family", target_pane=None, inverse=None, directed=False, transitive=False)`
to the provider's declared relations. This flips `RELATIONS` ON for `ref:notes`, which
the `vocabulary` phase's exit condition observed as OFF; that is an intended, documented
consequence of this phase and the tests that pin it move with it.

**D8. Plans hierarchy is the lifecycle chain, and it is sparse by construction.** Emit
`children` edges proposal→active, proposal→archive, and active→archive when one resolved
plan path appears in more than one stage; `parent` derives (D4). `plans_data` drops an
archive entry whose path is already active, so in the common case a plan is in exactly
one stage and this relation is empty. That is correct, not broken. Do not substitute the
plan document's `- **PARENT:**` body link for it in this phase — record it as a
`PROPOSED FOLLOW-UP`.

**D9. `hide_reverted` never enters the index.** It is a presentation filter the panel
applies. Today it is baked into `_find_ancestors` / `_find_siblings`; the moved logic
takes no such parameter. `AncestorsChildrenPanel` keeps its current behavior untouched —
this phase does not modify it, and `panel` deletes it.

**D10. Nothing computes an edge on a keystroke path.** Every index is built inside the
pane's existing worker pass and handed to the pane as one immutable value. No lazy
rebuild-on-selection, no index construction in a render or navigation handler.

## Work

### 1. `src/sase/core/artifact_entry_target.py` (new)

Move `ArtifactEntryTarget`, `_TOKEN_VERSION`, `_TOKEN_DELIMITER`, and
`_LEGACY_KIND_TO_PANE_ID` out of `widgets/artifacts/entry_navigation.py` verbatim. In
`entry_navigation.py`, replace them with
`from sase.core.artifact_entry_target import ArtifactEntryTarget` and keep the name in
`__all__` and in `widgets/artifacts/__init__.py` / `__init__.pyi` exactly as it is
today. No behavior change; `tests/ace/tui/test_artifacts_entry_navigation.py`-style
tests (find them by grepping for `from_token`) must pass unmodified.

### 2. `src/sase/core/artifact_relations.py` (new)

Textual-free. May import `sase.core.artifact_entry_target` and
`sase.ace.tui._artifact_tab_model` (for `PaneRelationDecl` / `RelationKind`) — the
latter is Textual-free by contract, and `core/graph_index_facade.py` is the precedent
for a `sase.core` module reaching into Python-owned host logic.

Records (all `@dataclass(frozen=True, slots=True)`):

- `RelationEdge`: `kind: RelationKind`, `relation: str`, `label: str`,
  `source: ArtifactEntryTarget`, `target: ArtifactEntryTarget`,
  `dangling: bool = False`, `derived: bool = False`.
- `RelationDiagnostic`: `code: str`, `relation: str`, `message: str`,
  `target: ArtifactEntryTarget | None = None`. Codes: `dangling_target`,
  `relation_cycle`, `undeclared_relation`.
- `RelationEdges`: `hierarchy`, `family`, `link` — three `tuple[RelationEdge, ...]`
  grouped by `RelationKind`, plus `def __bool__`.
- `RelationIndex`: `pane_id: str`, `relations: tuple[PaneRelationDecl, ...]`,
  `edges: tuple[RelationEdge, ...]`, `diagnostics: tuple[RelationDiagnostic, ...]`, and
  a private `Mapping[ArtifactEntryTarget, RelationEdges]` built once at construction.
  API: `edges_for(target) -> RelationEdges` (empty triple for an unknown target),
  `edges_for_relation(target, name) -> tuple[RelationEdge, ...]`, and
  `chain(target, name) -> tuple[RelationEdge, ...]`.

`chain()` is the transitive walk `panel` needs and the Patch oracle asserts against:
follow the first edge of `name` from `target`, break when the next hop was already
visited, and stop after a dangling edge. This reproduces `_find_ancestors`'
cycle-break-on-visited and "parent not in list" terminal case exactly.

Construction — `build_relation_index(*, pane_id, relations, edges, known_targets)`:

1. Drop an edge naming an undeclared relation and record `undeclared_relation`. Never
   raise.
2. Mark `dangling=True` and record one `dangling_target` diagnostic per distinct
   (relation, target) whose `target.pane_id == pane_id` and which is not in
   `known_targets` (D2).
3. Derive inverse / symmetric edges per D4, flagged `derived=True`, appended in forward
   edge order so output is deterministic.
4. Detect cycles at construction, not traversal: for each declared relation with
   `directed=True and transitive=True`, walk it and record one `relation_cycle`
   diagnostic per distinct cycle (name the entry target in the message). Keep the edges.

`RelationSource` ABC in the same module: abstract `pane_id`, `relations()`,
`known_targets()`, `raw_edges()`, plus a concrete `build_index()` that calls
`build_relation_index(...)`. It is pure by contract: no I/O, no globbing, no provider
code, no Git. State that in the class docstring.

Keep this module under ~400 lines; split the derivation/diagnostics helpers into
`src/sase/core/artifact_relations_build.py` if it grows past that (`toobig` gates `src`
at 700/850/1000).

### 3. `src/sase/ace/tui/relations/` (new package) — the built-in sources

Textual-free host logic, one module per source, each exposing one
`build_*_relation_index(...) -> RelationIndex` and a private `RelationSource` subclass.
`__init__.py` re-exports only the six builders.

- `patches.py` — `build_patches_relation_index(patches, graph_index, *, contract)`. Node
  universe: `patch_row_target(p)` for every patch. `ancestors`: one edge per patch with
  a `parent`, targeting the found parent's _actual-case_ name and its own
  `project_name`; when the parent is absent from `name_map`, target
  `(patch.project_name, parent_name_as_written)` — it becomes dangling, which is the
  terminal "parent not in list" case. `children`: one edge per entry of
  `graph_index.get_children(name)`, preserving that list's order (D4). `siblings`: one
  edge per `graph_index.get_siblings_of(patch)`, preserving the `__<N>` ordering the
  index already applies. No `hide_reverted`, no status filtering (D9).
- `beads.py` — `build_beads_relation_index(snapshot, *, contract)`. Node universe: every
  `(project, "task"|"epic"|"phase", id)` from `snapshot.tasks` / `.epics` /
  `.phases_by_epic`. `parent`: from `Issue.parent_id` (`children` derives, D4).
  `dependencies`: from `Issue.dependencies[].depends_on_id` (`dependents` derives with a
  synthesized label). `plans`: from `snapshot.plan_links[(project, bead_id)]`, targeting
  `("ref:plan", (project, "active" if the bead is not CLOSED else "archive", path))` —
  that mirrors how `plans_data` places a path (`BeadPlanLink.live`). Cross-pane, so
  never dangling (D2).
- `files.py` — `build_files_relation_index(snapshot, *, contract)`. Per D6.
- `stitches.py` —
  `build_stitches_relation_index(entries, *, contract, project_keys_by_repo)`.
  `parents`: one edge per `commit.parent_ids` entry, targeting `(repo, parent_full_id)`
  (`children` derives, D4). `patches`: parse the commit message's `SASE_PATCH` footer
  tag (`sase.vcs_log.tags.commit_tag_view` already strips and returns SASE tags — reuse
  it, do not re-implement footer parsing) and emit one cross-pane edge to
  `("patches", (project_key, patch_name))`, where `project_key` comes from the injected
  mapping and falls back to the repo label. The mapping keeps the source pure; the pane
  builds it (step 4).
- `documents.py` — `build_documents_relation_index(snapshot, *, contract)` for the
  built-in `ref:plan` pane. Node universe: proposal, active, and archive row targets as
  `plan_row_target` builds them. `children`: the lifecycle chain per D8. `beads`: one
  edge per `snapshot.bead_plan_links` entry whose path matches a row, targeting
  `("beads", (link.project, kind, link.bead_id))` with kind mapped from `link.bead_type`
  (`PLAN`→`epic`, `PHASE`→`phase`, `TASK`→`task`).
- `provider.py` — `build_provider_relation_index(snapshot, *, contract)` for
  `ref:<kind>` panes. Node universe: the archive rows the provider pane renders. For
  each declared relation whose `source` is a `ref.properties` key, read that key from
  `match.plan.frontmatter`; a same-pane value resolves against `relpath`, then `path`,
  then stem (case-insensitive) and is dangling when unmatched; a `target_pane` value
  emits `ArtifactEntryTarget(target_pane, (value,))` unchanged. Then apply the
  host-owned filename family under the `bundle` relation (D7): group documents by the
  stem with a trailing `__<suffix>` stripped, and emit `parent ↔ member` edges where the
  parent is the document whose stem equals the base; when no unsuffixed document exists,
  connect the suffixed members to each other. `directed=False` supplies the reverse
  (D4).

### 4. Contract declarations touched by this phase

In `_artifact_tab_contract_adapters.py`, replace the Stitches `plans` relation with
`patches` per D5. In `_artifact_tab_contract.py`'s `compile_provider_contract`, append
the host-owned `bundle` relation per D7 (define the record next to the other host-owned
adapter constants, not inline). Update the contract-compiler assertions in
`tests/ace/tui/artifacts_contract/test_contract_compiler.py` and any `pane show`
expectation in `tests/main/test_artifact_pane.py` that names the Stitches `plans`
relation or asserts `ref:notes` has no relations. Do **not** bump
`PANE_SHOW_SCHEMA_VERSION` — the payload shape is unchanged.

### 5. Wiring — one index per snapshot, on the loader thread

Each pane builds its index inside the worker pass that already builds its rows, wrapped
in a `tui_trace` span named `relations.index.<pane_id>` with a `count=` counter
(mirroring `widget.ancestors_children.update_relationships_from_index`). Each pane
exposes `relation_index() -> RelationIndex | None`.

- **Patches** — `actions/patch/_loading.py`: add `relation_index` to
  `_PreparedPatchLoad`, build it in `_prepare_patch_load_from_disk` (worker), and
  install it in `_apply_reloaded_patches`. `_apply_patches` (the synchronous
  `_load_patches` path, which already builds the query index inline) builds it there.
  Store as `self._patch_relation_index` / `self._patch_relation_index_for_id` keyed on
  `id(_all_patches)`, exactly like `_get_patch_graph_index`, and initialize both in
  `actions/_state_init_navigation.py` beside `_patch_graph_index`. Do not touch
  `actions/patch/_display.py` — the epic assigns that file to `panel`.
- **Beads / Files / Documents** — add a `relation_index` field to
  `_BeadsSnapshotResult`, `_FilesSnapshotResult`, and `_PlansSnapshotResult`; populate
  it in each pane's `_build_snapshot`; assign it to `self._relation_index` in
  `_apply_snapshot`. The Documents pane picks `build_documents_relation_index` when
  `contract.facts.is_plan_adapter` and `build_provider_relation_index` otherwise. Do not
  add fields to `BeadsSnapshot` / `FilesSnapshot` / `PlansSnapshot` — those are shared
  data models that the parallel `grouping` phase also reads, and the per-load carrier is
  the right home.
- **Stitches** — `commits_collection.py`: add `relation_index` to
  `CommitCollectionPayload`, build it in `_collect_payload` (resolve
  `project_keys_by_repo` there via
  `load_project_ref_display_snapshot().project_key_for_ref`, which is cached), thread it
  through `_apply_result` into a `_relation_indexes_by_result: dict[int, RelationIndex]`
  cache maintained beside `_authoritative_query_indexes` in
  `_remember_authoritative_result` (evict on the same bound), and expose
  `relation_index()` off the current result.

A pane whose contract lacks `PaneCapability.RELATIONS` stores `None` and builds nothing.

### 6. Oracle

- `tests/ace/tui/artifacts_contract/goldens/relations/cases.json`: keep the four
  existing Patch cases (`parent_chain`, `cycle`, `missing_parent`, `family`)
  byte-identical. Add a `panes` object with one case per built-in source (beads, files,
  stitches, documents, provider) holding minimal literal snapshot inputs and expected
  edges as `{relation, kind, source_parts, target_parts, target_pane, dangling}`
  records. Replace `"cross_kind_edges": []` with the real expected Stitches→Patches and
  Plans↔Beads edges.
- `tests/ace/tui/artifacts_contract/test_relation_goldens.py`: delete
  `_RelationshipProbe` and assert against `RelationIndex` directly — `ancestors` from
  `index.chain(target, "ancestors")`, `children` from
  `edges_for_relation(target, "children")`, `siblings` from
  `edges_for_relation(target, "siblings")`, each mapped back to names. The four Patch
  answers must be unchanged; this is the phase's hard invariant.

### 7. New tests

- `tests/core/test_artifact_relations.py`: inverse derivation, symmetric
  `directed=False` materialization, `undeclared_relation` drop, same-pane dangling with
  diagnostic, cross-pane never dangling, cycle diagnostic without edge loss, `chain()`
  stopping on a visited hop and on a dangling hop, and that a built index is not mutated
  by later calls.
- `tests/core/test_artifact_entry_target.py` (or extend the existing entry-navigation
  test): assert `sase.core.artifact_relations` and `sase.core.artifact_entry_target`
  import with `textual` absent from `sys.modules` in a subprocess. This is the "usable
  from a CLI" guarantee, and it is the check that fails first if someone re-couples the
  module.
- `tests/ace/tui/test_artifacts_relation_sources.py`: one focused test per source over
  hand-built snapshots, plus the provider filename-family case (`bundle/bundle__a.md`,
  `bundle/bundle__b.md`, `bundle.md`) and a provider declared-property edge and its
  dangling counterpart.
- `tests/ace/tui/test_artifacts_relation_wiring.py`: for each of the five panes, assert
  the worker pass returns a carrier holding a `RelationIndex` for the pane's id, and
  that the pane exposes it after apply. Keep these light — they are wiring assertions,
  not pane behavior tests.

### 8. Symvision

Everything this phase adds has a non-test consumer through the wiring in step 5 except
`RelationIndex.chain`, `edges_for_relation`, and any helper only `panel` will call.
Methods are not module-level defs, so prefer keeping them as methods. If
`just _lint-symvision` reports a genuinely unused public def that `panel` will consume,
add `--epic-symbol 'sase-m6.7.1(<symbol>)'` to the Symvision invocation in the
`Justfile` — that bead is the in-progress epic — rather than making it private.

## Verification

1. `just install` (mandatory: ephemeral workspace).
2. `just fmt`.
3. Focused first:
   `pytest tests/core/test_artifact_relations.py tests/ace/tui/artifacts_contract/test_relation_goldens.py tests/ace/tui/artifacts_contract/test_contract_compiler.py tests/ace/tui/test_artifacts_relation_sources.py tests/ace/tui/test_artifacts_relation_wiring.py tests/main/test_artifact_pane.py`.
4. This change touches `tests/ace/tui/test_artifacts_*` broadly, so `just check`'s
   scoped lane will escalate. Run `just check-full` through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …`) with a `--next` action so the
   follow-up agent acts on the result. Never run it inline.
5. Performance: the epic's gate is per-conversion, and this phase converts no widget —
   but it does add work to five loader passes. Capture `SASE_TUI_PERF=1` navigation p95
   on Patches and on one snapshot pane per `docs/perf_runbook.md`, confirm it stays
   under 16 ms, and record both numbers on bead `sase-m6.7.1.2`. Also confirm by reading
   the diff that no index is constructed outside a worker pass or a load path — a
   benchmark cannot prove that and a reviewer can.
6. Expect no PNG golden movement: no widget reads an index this phase. If Artifacts
   goldens do move, stop and find out why before regenerating anything (retired task
   `sase-lo`: blanket `--sase-update-visual-snapshots` absorbs unrelated drift).

## Risks

- **The Patch goldens are the tripwire.** Moving `_find_ancestors` / `_find_siblings`
  out of the widget is the one place this phase can silently change an answer. The four
  existing cases must stay byte-identical, and the `cycle` and `missing_parent` cases
  are the two that a naive rewrite gets wrong (break-before-append, and
  append-then-break).
- **`ArtifactEntryTarget` has ~205 call sites.** The move must be a pure relocation plus
  a re-export; any change to the token encoding breaks persisted marks and jump state.
- **Stitches' snapshot machinery is unlike the other four.** It caches by `id(result)`
  rather than holding one snapshot, so the relation index must follow the same eviction
  path or it leaks per collection.
- **`grouping` (`sase-m6.7.1.5`) runs in parallel and also touches pane files.** This
  phase stays out of `models/group_fold.py`, the four list builders,
  `actions/navigation/_fold.py`, and `actions/patch/_grouping_nav.py`, and adds no
  `EntryJumpAnchor` type. Adding fields to the per-load carrier dataclasses instead of
  the shared snapshot models is deliberate seam hygiene.
- **`panel` (`sase-m6.7.1.3`) consumes this API next.** Keep `edges_for` returning the
  three primitives already grouped and keep dangling edges in the result — `panel`
  renders them as visibly dangling links.

## Out of scope

- Any widget reading an index, any keybinding, any keymap or `default_config.yml` edit,
  and any change to `AncestorsChildrenPanel`, `actions/navigation/_tree.py`, or
  `actions/patch/_display.py` — all `panel`'s.
- `CAPABILITY_HOST_ACTIONS[RELATIONS]` and the conformance-harness exemption — `panel`
  and `conform`.
- The reveal lens and the query rewrite — `reveal`.
- A Rust `RelationIndex`; the epic forbids a `sase-core` change here.

## Proposed follow-ups to record on the bead

Record these with `sase bead note sase-m6.7.1.2 'PROPOSED FOLLOW-UP: …'` while
implementing; the epic's land agent triages them.

- Stitches commit→plan edges from the `SASE_PLAN` footer tag, once a worker-side
  reference→path resolver is available to the commits pane (dropped in D5).
- Plans hierarchy from a plan document's `- **PARENT:**` body link, which is a real
  document graph the lifecycle chain does not capture (D8).
- A name-only Patch target resolver so a commit→patch jump does not depend on mapping a
  repo label to a project key (D5/step 3).
- Beads→Patches link edges from `Issue.changespec_name`, which is real data with no
  declared relation today.
