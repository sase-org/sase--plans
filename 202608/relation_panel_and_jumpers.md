---
tier: tale
title: One host-owned relation panel and generalized jumpers
goal:
  Every Artifacts pane with the RELATIONS capability renders one host-owned
  RelationPanel over the immutable per-snapshot RelationIndex, and the ancestor / child
  / sibling key modes are driven by that contract instead of a Patch tab literal — so
  Beads epic→phase and Stitches commit→parent are reachable with `<`/`>`, no relation
  code branches on a pane id, and Patch's key grammar and rendered rows are unchanged.
size: medium
proposed_by: bbugyi200.athena.sase-m6.7.1.3
bead: sase-m6.7.1.3
create_time: 2026-08-16 04:47:36
status: wip
---

- **PARENT:**
  [202608/artifacts_relations_and_grouping.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifacts_relations_and_grouping.md)
- **BEAD:**
  [sase-m6.7.1.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.7.1.3.md)

# Plan: One host-owned relation panel and generalized jumpers

Implements phase `panel` (bead `sase-m6.7.1.3`) of epic `sase-m6.7.1`
(`plan:202608/artifacts_relations_and_grouping.md`). `vocabulary` (`sase-m6.7.1.1`) gave
the contract real `PaneRelationDecl` records for all five panes; `index`
(`sase-m6.7.1.2`, landed at `708c25452`) built one immutable `RelationIndex` per
snapshot on every loader thread and deliberately let no widget read it. This phase is
the widget and the driver: the host renders the edges and drives the keys.

Read `sase/memory/tui_perf.md` before starting
(`sase memory read tui_perf.md -r "..."`). Run `just install` first — workspaces are
ephemeral and dependencies drift.

## Grounding

Verified in this workspace at `708c25452`. Line numbers are from that tree.

| Fact                                                                  | Evidence                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The relation widget is 613 lines typed on `Patch`/`PatchGraphIndex`   | `widgets/ancestors_children_panel.py:51-612`                                                                                                                                                                                                         |
| Its driver is tab-gated, not capability-gated                         | `_is_patch_tree_tab` compares `current_tab` to three literals (`actions/navigation/_tree.py:11-16`; file is 358 lines)                                                                                                                               |
| The key grammar lives in the driver, the key assignment in the widget | `_process_child_key` / `_is_valid_next_child_key` (`_tree.py:147-262`); `_assign_ancestor_keys` / `_build_subtree` (panel:289-479)                                                                                                                   |
| The query rewrite is Patch-token hard-coded                           | `ancestor:` / `sibling:` literals and the `"patches"` history stack (`_tree.py:293-357`)                                                                                                                                                             |
| Every pane already exposes a prebuilt index                           | `relation_index()` on `snapshot_pane.py:50-52`, `panes.py:142-148`, `actions/patch/_loading.py:619-626`                                                                                                                                              |
| The index already answers all three primitives                        | `edges_for`, `edges_for_relation`, `chain` (`core/artifact_relations.py:74-113`)                                                                                                                                                                     |
| `chain()` already reproduces Patch ancestor-walk semantics            | break on visited hop, append-then-stop on dangling (`core/artifact_relations.py:89-113`)                                                                                                                                                             |
| Every pane declares its relations in order, up-direction first        | patches `ancestors,children,siblings`; beads `parent,children,plans,dependencies`; stitches `parents,children,patches`; plan `parent,children,beads`; files `versions` (`_artifact_tab_contract_adapters.py:92-123,177-208,262-303,357-368,424-455`) |
| Provider panes get a host-owned `bundle` family                       | `PROVIDER_BUNDLE_RELATION` (`_artifact_tab_contract_adapters.py:40-49`)                                                                                                                                                                              |
| `RELATIONS` has no host actions and conformance knows it              | `CAPABILITY_HOST_ACTIONS[RELATIONS] = ()` (`_artifact_tab_actions.py:59`); four-capability exemption (`artifacts_contract/harness.py:75-82`)                                                                                                         |
| The typed cross-pane route already exists                             | `_request_artifacts_entry` switches pane and requests the target (`actions/artifacts_navigation.py:113-125`)                                                                                                                                         |
| The non-PR jump-back stack is per pane and already generic            | `self._artifacts_jump_history[pane_key] = origin` (`actions/artifacts_navigation.py:309-311`)                                                                                                                                                        |
| The Patch jump-back stack is a different mechanism                    | `_push_patch_to_history` snapshots a Patch row anchor for any `_PATCH_TABS` tab (`actions/navigation/_entry_jump_generic.py:97-118,168-180`)                                                                                                         |
| Every pane's navigator contract is one ABC                            | `ArtifactEntryNavigator` (`widgets/artifacts/entry_navigation.py:30-69`)                                                                                                                                                                             |
| Every pane already has one debounced detail path to hook              | `_apply_detail_panel_update` (`actions/patch/_display.py:218-260`), `_on_option_highlighted` (`beads_pane.py:265-273`, `files_pane.py:297-301`, `plans_pane.py:317-325`), `_sync_timeline_selection` (`commits_detail.py:64-76`)                     |
| Every pane has one list column the panel can sit in                   | `#list-container` (`panes.py:51-56`), `#beads-list-panel` (`beads_pane.py:93-97`), `#files-list-panel` (`files_pane.py:116-121`), `#plans-list-panel` (`plans_pane.py:131-137`), `#stitches-list-container` (`commits_pane.py:101-105`)              |
| Duplicate ids across sibling pane subtrees are already the norm       | two mounted document panes both own `#plans-list` (`plans_pane.py:338-339`)                                                                                                                                                                          |
| Link jumps already exist behind each pane's own binding               | `action_beads_open_plan` (`actions/_artifacts_beads_browse.py:54-76`), `action_plans_open_bead` (`actions/artifacts_plans.py:85-103`); both bound to `L` (`bindings.py:144,163`)                                                                     |
| Those two actions recompute the target the index already emits        | bead→plan kind/path (`_artifacts_beads_browse.py:64-73`) vs. the `plans` edge; plan→bead kind/id (`artifacts_plans.py:96-102`) vs. the `beads` edge                                                                                                  |
| Conditional footer entries are the shared per-pane hint path          | `beads_navigation.py:204-234`, `plans_navigation.py:171-175`, `files_navigation.py:170`, `commits_detail.py:96-97`                                                                                                                                   |
| Help sections are already generated per compiled contract             | `_document_contract_sections` (`modals/help_modal/patches_artifact_bindings.py:194-268`); boxes are 57 wide / 50 content (`binding_common.py:10-11`)                                                                                                 |
| One help row is pinned by a test                                      | `("< / > / ~", "Navigate to ancestor / child / sibling")` (`tests/test_keymaps_display_help.py:228`)                                                                                                                                                 |
| Two tests subclass the widget being deleted                           | `_FakePanel` (`tests/ace/tui/test_changespec_graph_index.py:59-79`), `_RecordingAncestors` via `_w_ancestors_children` (`tests/ace/tui/test_changespec_detail_only_refresh.py:182`)                                                                  |
| Two more import it directly                                           | `tests/test_ancestors_children_panel.py:7`, `tests/test_changespec_status_indicators.py:4`                                                                                                                                                           |
| `RelationEdges` is whitelisted as an unused epic symbol               | `--epic-symbol 'sase-m6.7.1(RelationEdges)'` (`Justfile:314`)                                                                                                                                                                                        |
| The Artifacts p95 gate is already a runnable bench                    | `tests/ace/tui/bench_artifacts_jk.py` (16 ms budget, every sub-tab); `tests/ace/tui/bench_tui_jk.py::test_bench_patches_jk`                                                                                                                          |

## Decisions

**D1. The host assigns key roles from declaration order and kind — never from a pane id
or a relation name.** Add a closed `RelationRole` (`ANCESTOR`, `DESCENDANT`, `FAMILY`,
`LINK`) and a pure `assign_relation_roles(relations) -> Mapping[str, RelationRole]`:

- the **first** declared `HIERARCHY` relation is `ANCESTOR` (`<`); if its `inverse`
  names another declared `HIERARCHY` relation, that one is `DESCENDANT` (`>`);
- every `FAMILY` relation is `FAMILY` (`~`), merged in declaration order;
- every `LINK` relation is `LINK` (no key mode — routed through each pane's existing
  link binding, D7);
- any further `HIERARCHY` pair beyond the first renders in the panel with no key mode.
  No pane declares one today; this is a defined outcome, not an error.

Every built-in pane and the provider `bundle` family already satisfy this ordering
(grounding). The rule is host-owned and documented; a provider names properties, never a
key. **Rejected:** adding a `direction` field to `PaneRelationDecl` — that record,
`_presentation_digest`, `explanation_payload`, and `PANE_SHOW_SCHEMA_VERSION` are
`vocabulary`'s, already shipped, and a second schema bump for a fact the host can derive
buys nothing.

**D2. Layout is Textual-free; the widget only paints.** Put the roles, key assignment,
traversal, and hidden-count logic in `src/sase/core/artifact_relation_layout.py` and
keep `widgets/artifacts/relation_panel.py` a renderer over the value it returns. This is
what makes Patch key-grammar parity testable without mounting a widget, and it keeps
both files far under `toobig`'s 700-line warn.

**D3. Patch behavior is preserved exactly, and that is this phase's hard invariant.**
The moved algorithms keep, verbatim: ancestor walk that traverses _through_ hidden
entries and counts them; descendant pre-order counter with the depth-0 single-leaf `>`
special case, the `>>`-then-`>a…>z` sequence, and the 27-entry cap that silently skips
the overflow; hidden descendants pruning their whole subtree and being counted only at
their own level; siblings filtered-and-counted; `<`/`<<`/`<a`, `>`/`>>`/`>a`/`>2a`/`.`,
`~`/`~~`/`~a`; the `  [k] name [S]` row shape, the box-drawing descendant tree, the
`(N hidden)` suffixes, the `ANCESTORS`/`CHILDREN`/`SIBLINGS` headers (now
`decl.label.upper()`), and `display = False` when there is nothing to show.
`tests/ace/tui/artifacts_contract/test_relation_goldens.py` must stay byte-identical.

**D4. Presentation facts come from the pane; the index stays presentation-free.**
`RelationEntryFact(label: str, status: str = "", hidden: bool = False)` and a new
**non-abstract** `ArtifactEntryNavigator.relation_entry_facts()` returning
`Mapping[ArtifactEntryTarget, RelationEntryFact]`, default `{}`. With no fact the host
falls back to `label = target.parts[-1]`, no status glyph, not hidden. `hide_reverted`
lives here — the Patches pane computes `hidden` from the app's effective hide-reverted
verdict, which is exactly where `index`'s D9 said it belongs. The status-glyph table
(`_get_simple_status_indicator`) moves into the panel module unchanged and renders
nothing for an empty status, so no non-Patch pane grows a Patch status letter.

**D5. A same-pane target that is filtered out hands off through a pane hook, not a pane
branch.** New non-abstract
`ArtifactEntryNavigator.reveal_entry_target(target, *, role) -> bool`, default `False` →
the host reports the target as dangling with a warning notification.
`ArtifactsPatchesPane` overrides it to call the existing `_change_query_for_navigation`,
so today's Patch rewrite is preserved to the token. `reveal` (`sase-m6.7.1.4`) replaces
that override with the lens; nothing else has to move for it.

**D6. Jump-back origin is a pane hook too.** New non-abstract
`ArtifactEntryNavigator.record_relation_origin(origin)`: the default writes
`app._artifacts_jump_history[pane_key] = origin` (the same thing the non-PR hint jump
already does); `ArtifactsPatchesPane` overrides it to call `_push_patch_to_history()`.
Without this the host would push a _Patch_ row anchor while the user is on Beads —
`_current_entry_jump_anchor` reads `app.patches` for any `_PATCH_TABS` tab, and
`current_tab` is `"artifacts"` for every pane.

**D7. No new keybinding, no keymap edit, no new action name.**
`CAPABILITY_HOST_ACTIONS[RELATIONS]` becomes
`("start_ancestor_mode", "start_child_mode", "start_sibling_mode", "beads_open_plan", "plans_open_bead")`
— the adapter-specific link-jump actions that already exist and are already bound, now
_resolving their target from the relation index_ instead of re-deriving it from the
snapshot. That satisfies the phase's "and the link-jump action" without taking
`sase-m6.9`'s `L`, and matches the existing `ENTRY_NAVIGATION` precedent of listing
adapter commands. `default_config.yml` is not touched.

**D8. `AncestorsChildrenPanel` is deleted, not aliased.** Delete the module, its lazy
export in `widgets/__init__.py`/`.pyi`, and rename `_w_ancestors_children` →
`_w_relation_panel`. The four tests that import or subclass it are retargeted at the
real seams (layout parity, panel render, no-rebuild-per-selection, detail-only refresh).

**D9. Nothing computes an edge on the keystroke path.** The panel is handed the pane's
already-built `RelationIndex` and never constructs one. Every panel refresh runs on the
pane's existing debounced detail path. The one data-scaled thing this phase adds per
selection is the Patch facts map, and it is cached on `(id(_all_patches), hidden_flag)`
exactly like `_get_patch_graph_index`.

**D10. Seam hygiene with the parallel `grouping` phase.** This phase does not touch
`models/group_fold.py`, `actions/patch/_grouping_nav.py`, `actions/navigation/_fold.py`,
the four list builders, or `jump_hints.py`, and adds **no** `EntryJumpAnchor` type. It
owns `actions/navigation/_tree.py`, `widgets/ancestors_children_panel.py`,
`actions/patch/_display.py`, and the `RELATIONS` entry in `_artifact_tab_actions.py`.

## Work

### 1. `src/sase/core/artifact_relation_layout.py` (new) — Textual-free layout

Records (`@dataclass(frozen=True, slots=True)`) plus one pure builder.

- `RelationRole(StrEnum)`: `ANCESTOR`, `DESCENDANT`, `FAMILY`, `LINK`.
- `assign_relation_roles(relations: tuple[PaneRelationDecl, ...]) -> dict[str, RelationRole]`
  per D1.
- `RelationEntryFact(label, status="", hidden=False)` per D4.
- `RelationRow(key: str, target: ArtifactEntryTarget, label: str, status: str, depth: int, dangling: bool, cross_pane: bool, children: tuple[RelationRow, ...] = ())`.
- `RelationSection(relation: str, label: str, kind: RelationKind, role: RelationRole, rows: tuple[RelationRow, ...], hidden_count: int)`.
- `RelationKeymap(ancestors: tuple[tuple[str, ArtifactEntryTarget], ...], children: tuple[tuple[str, ArtifactEntryTarget], ...], siblings: tuple[tuple[str, ArtifactEntryTarget], ...], links: tuple[tuple[str, ArtifactEntryTarget], ...])`
  — ordered, with `target_for(role, key)` and `role_for_relation` lookups. Ordered
  tuples, not dicts, because `action_start_ancestor_mode` takes "the first ancestor" and
  the current code relies on `dict` insertion order to mean that.
- `RelationView(sections, keymap, roles)` with `__bool__` = "any row or any hidden".
- `build_relation_view(*, index: RelationIndex, origin: ArtifactEntryTarget, relations, facts: Mapping[...] = {}) -> RelationView`.

`build_relation_view` walks, per role:

- **ANCESTOR** — `index.chain(origin, name)`; drop hops whose fact is `hidden` while
  continuing the walk, counting them; assign `<` for one, `<<`+`<a…<z` for many.
  `chain()` already gives break-on-visited and append-then-stop-on-dangling (D3).
- **DESCENDANT** — recursive `index.edges_for_relation(target, name)` in index order,
  with the exact counter/key/cap/leaf rules of `_build_subtree`; hidden children are
  dropped with their subtrees and counted at their own level.
- **FAMILY** — `index.edges_for_relation(origin, name)` in index order, hidden dropped
  and counted; `~` for one, `~~`+`~a…~z` for many.
- **LINK** — `index.edges_for_relation(origin, name)` in index order, no key mode.

An edge with `dangling=True` becomes a row flagged `dangling`; an edge whose
`target.pane_id != index.pane_id` becomes a row flagged `cross_pane`. Neither is ever
dropped and neither raises. Cycle detection already happened at construction; the walk
just terminates.

### 2. `src/sase/ace/tui/widgets/artifacts/relation_panel.py` (new) — the widget

- `RelationPanel(Static)` with
  `update_relations(*, index, origin, relations, facts, accent) -> RelationKeymap` and
  `clear()`. It calls `build_relation_view`, renders, and returns the keymap. It never
  builds an index (D9).
- Rendering matches D3 exactly for hierarchy/family sections. LINK sections render under
  the declared label with a `→ <pane label>` marker for a cross-pane row and no key. A
  dangling row renders its label dimmed with a trailing `(missing)` marker and the
  section keeps its key — dangling never invalidates the pane.
- Section headers use `decl.label.upper()` on the contract accent; rows keep today's
  `#FFAF00` key / `#00D7AF` label / status-glyph styling.
- `_get_simple_status_indicator` moves here verbatim, with an added empty-status
  early-out returning `("", "")`.
- A `tui_trace("widget.relation_panel.update_relations", count=...)` span replaces the
  two `widget.ancestors_children.*` spans; update `docs/perf_runbook.md`'s span list.
- Also in this module: `RelationPanelHostMixin` with `relation_panel()` (scoped
  `query_one` returning `None` when absent), `refresh_relation_panel()` (reads
  `relation_index()`, `selected_entry_target()`, `relation_entry_facts()`, updates the
  panel, and publishes the keymap onto the app **only when this pane is the active
  surface**), and `relation_footer_entries(keymap)` returning the conditional
  `(action, label)` pairs for the shared footer.

### 3. `src/sase/ace/tui/widgets/artifacts/entry_navigation.py` — three default hooks

Add to `ArtifactEntryNavigator`, all **non-abstract** so no pane breaks:
`relation_entry_facts()` (D4), `reveal_entry_target(target, *, role)` (D5),
`record_relation_origin(origin)` (D6). Document each as host-called-uniformly. Export
`RelationEntryFact` from `widgets/artifacts/__init__.py` only if a pane needs it (the
Patches pane does).

### 4. Mount and drive the panel on all five panes

Compose one `RelationPanel(id="<pane>-relation-panel")` at the bottom of each pane's
list column: `panes.py` (replacing `AncestorsChildrenPanel`), `beads_pane.py`,
`files_pane.py`, `plans_pane.py`, `commits_pane.py`. Mix `RelationPanelHostMixin` into
each pane class. Call `refresh_relation_panel()` from each pane's existing debounced
detail path — `_update_detail` (beads/files/plans), `_sync_timeline_selection`
(stitches) — and from `_apply_detail_panel_update` for Patches.

`ArtifactsPatchesPane` additionally overrides `relation_entry_facts()` (label = name,
status = `PatchGraphIndex.status_by_name`, `hidden` = effective hide-reverted ∧ terminal
status), `reveal_entry_target()`, and `record_relation_origin()`. Cache the facts map on
`(id(app._all_patches), hidden_flag)`.

`styles.tcss`: rename `#ancestors-children-panel` to a shared
`.artifacts-relation-panel` class rule with the identical
`height/max-height/padding/ border/background`, and give the Patches instance that class
so its pixels do not move.

### 5. `actions/patch/_display.py` — Patches drives the shared panel

Rename `_w_ancestors_children` → `_w_relation_panel` (here,
`actions/_state_init_navigation.py:71`, `actions/startup.py:111`,
`actions/_startup_mount.py:24,86-92`), retarget `_get_ancestors_children_widget` at
`#patches-relation-panel`/`RelationPanel`, and replace the
`update_relationships_from_index(...)` call with the pane's `refresh_relation_panel()`,
storing the returned `RelationKeymap` on `self._relation_keymap`. The empty branch
clears the panel and resets the keymap. `_get_patch_graph_index` stays — the facts map
uses it.

### 6. `actions/navigation/_tree.py` — the generalized driver

- `_is_patch_tree_tab()` / `_tree_patches()` are replaced by `_relation_navigator()` and
  `_relation_contract()`; the modes run when the contract has `PaneCapability.RELATIONS`
  and the keymap is non-empty. The only tab literals left are the legacy
  `"patches"`/`"changespecs"` aliases resolving to the `patches` contract — a tab shim,
  not a pane branch.
- `_ancestor_keys` / `_children_keys` / `_sibling_keys` are replaced by one
  `_relation_keymap: RelationKeymap` (initialized in
  `actions/_state_init_navigation.py`, declared in `actions/navigation/_types.py` and
  `actions/patch/_display.py`). `_child_key_buffer`, `_ancestor_mode_active`,
  `_child_mode_active`, `_sibling_mode_active`, and every `_process_*_key` /
  `_is_valid_next_child_key` body keep their current logic verbatim against the new
  lookup (D3).
- `_navigate_to_patch_compat` / `_navigate_to_patch` / `_find_in_current_list` are
  replaced by:

  ```text
  _navigate_to_relation_target(target, *, role):
      pane = active relation navigator; origin = pane.selected_entry_target()
      if origin is not None and origin != target: pane.record_relation_origin(origin)
      if target.pane_id != contract.id:            -> self._request_artifacts_entry(target)
      elif pane.select_entry_target(target):       -> done
      elif pane.reveal_entry_target(target, role=role): -> done
      else: notify("<label> is not in the current results", severity="warning")
  ```

- Keep `_navigate_to_patch(name, is_ancestor, is_sibling)` and `_navigate_to_changespec`
  as documented legacy shims that resolve a name against the Patches pane's index and
  call `_navigate_to_relation_target`, with a test that pins them.
  `_change_query_for_navigation` stays exactly as it is — it is now reached only through
  `ArtifactsPatchesPane.reveal_entry_target` (D5), and `reveal` generalizes it.
- If the file grows past ~450 lines, split target routing into
  `actions/navigation/_relation_nav.py` and keep `_tree.py` as the key modes.

`action_start_sibling_mode`'s Agents-tab delegation to
`_start_agent_neighbor_navigation` is untouched — Agents is outside the contract.

### 7. Link jumps route through the contract

`action_beads_open_plan` and `action_plans_open_bead` resolve their target from the
active pane's `RelationIndex` LINK edges for the selected row (first edge of the
declared link relation) and call `_navigate_to_relation_target`. Their "no link" warning
text and their `PLAN_OPEN_BEAD` availability gate are unchanged. The matching
`conditional_footer_entries` conditions in `beads_navigation.py` and
`plans_navigation.py` switch from the snapshot lookup to the index, and every non-PR
pane's `conditional_footer_entries()` gains the shared relation entries from
`relation_footer_entries(...)`. The Patches pane keeps returning `()` — it drives its
own footer directly, and its keys are advertised in the panel.

### 8. Surfaces: capability actions, conformance, help

- `_artifact_tab_actions.py`: populate `PaneCapability.RELATIONS` per D7.
- `tests/ace/tui/artifacts_contract/harness.py`: shrink the exemption to
  `{GROUPING, STATUS_COUNTERS, SHELL}` (`GROUPING` stays — it is the parallel phase).
- `modals/help_modal/patches_artifact_bindings.py`: add `_relation_rows(km, contract)`
  generating one row per pane section from `contract.relations` +
  `assign_relation_roles` — key column from the configured
  `start_ancestor_mode`/`start_child_mode`/`start_sibling_mode` display names, and the
  description from the declared labels, truncated to 32 characters per
  `src/sase/ace/CLAUDE.md`. Apply it to the built-in pane sections and the generated
  document sections. Leave `patches_bindings.py`'s "PR Navigation" row alone —
  `tests/test_keymaps_display_help.py:228` pins it and it is a different section.
- `Justfile`: drop `--epic-symbol 'sase-m6.7.1(RelationEdges)'` — this phase consumes
  it.

### 9. Tests

New:

- `tests/core/test_artifact_relation_layout.py` — role assignment (each built-in
  declaration set, a `FAMILY`-only pane, a hierarchy pair declared in reverse order, a
  `LINK`-only pane); ancestor keys `<`/`<<`/`<a`; descendant pre-order counter,
  single-leaf `>`, 27-entry cap; sibling keys `~`/`~~`/`~a`; hidden ancestors traversed
  through and counted; hidden descendants pruning subtrees; dangling and cross-pane row
  flags; empty view is falsey.
- `tests/ace/tui/test_artifacts_relation_panel.py` — rendered `Text` parity for the
  Patch case (headers, row shape, `(N hidden)`, tree branches), `display=False` when
  empty, dangling row marker, cross-pane link row, and that a 100-selection loop never
  calls `build_relation_index` (this replaces `test_changespec_graph_index.py`'s
  `_FakePanel` probe).
- `tests/ace/tui/test_artifacts_relation_navigation.py` — `<` on Beads selects the epic
  from a phase and `>` selects the phase from the epic; `<` on Stitches selects the
  commit's parent; a cross-pane link edge routes through `_request_artifacts_entry`; a
  filtered-out same-pane target reaches `reveal_entry_target` (Patches: the query
  rewrite fires, with the same token as today); a pane without `RELATIONS` is a no-op;
  `record_relation_origin` writes the Patch stack on Patches and
  `_artifacts_jump_history` elsewhere.
- `tests/ace/tui/test_artifacts_relation_surfaces.py` —
  `CAPABILITY_HOST_ACTIONS[RELATIONS]` is non-empty and registered; the harness
  exemption is exactly the three remaining capabilities; the generated help rows appear
  for a pane with `RELATIONS` ON and not for one with it OFF, and every generated
  description is ≤32 characters.

Retargeted / updated:

- `tests/test_ancestors_children_panel.py` → assert the same two family answers against
  `build_relation_view` (keep the expected values byte-identical).
- `tests/test_changespec_status_indicators.py` → import from the new module.
- `tests/ace/tui/test_changespec_graph_index.py` → keep the two `PatchGraphIndex`
  assertions, drop `_FakePanel`.
- `tests/ace/tui/test_changespec_detail_only_refresh.py` → `_RecordingRelationPanel`,
  `_w_relation_panel`, `_relation_keymap`.
- `tests/ace/tui/artifacts_contract/test_relation_goldens.py` — **unchanged**. If it
  moves, stop: a Patch relation answer changed.

**Parity procedure (do this before deleting the widget).** Write the new layout tests'
expected values by running the _current_ `AncestorsChildrenPanel` over each fixture and
recording its output, then assert those recorded values against `build_relation_view`.
Do not hand-derive them from the code — `cycle` and `missing_parent` are exactly the
cases a re-read gets wrong.

### 10. Docs

- `docs/artifacts_pane_visual_grammar.md`: add the relation panel as a named slot at the
  bottom of the content region's list column, with the section/row/hidden-count/dangling
  treatment and the D1 role rule stated as host-owned.
- `docs/perf_runbook.md`: replace the two `widget.ancestors_children.*` spans with
  `widget.relation_panel.update_relations`.

## Verification

1. `just install` (mandatory: ephemeral workspace).
2. `just fmt`.
3. Focused first:
   `pytest tests/core/test_artifact_relation_layout.py tests/ace/tui/test_artifacts_relation_panel.py tests/ace/tui/test_artifacts_relation_navigation.py tests/ace/tui/test_artifacts_relation_surfaces.py tests/ace/tui/artifacts_contract/ tests/ace/tui/test_changespec_detail_only_refresh.py tests/ace/tui/test_changespec_graph_index.py tests/test_ancestors_children_panel.py tests/test_changespec_status_indicators.py tests/test_keymaps_display_help.py tests/test_command_catalog_guards.py`.
4. This change touches `tests/ace/tui/test_artifacts_*` broadly, so `just check`'s
   scoped lane will escalate. Run `just check-full` through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …`) with a `--next` action so the
   follow-up agent acts on the result. Never run it inline.
5. **Performance gate** (this phase converts five panes, so it is the phase's own gate):
   capture before and after
   `pytest -s -m slow tests/ace/tui/bench_tui_jk.py::test_bench_patches_jk` and
   `pytest -s -m slow tests/ace/tui/bench_artifacts_jk.py`, confirm p95 < 16 ms on
   Patches and on every converted pane, and record the numbers on bead `sase-m6.7.1.3`.
   Also confirm by reading the diff that no `build_relation_index` /
   `build_*_relation_index` call exists outside a loader path — a bench cannot prove
   that and a reviewer can.
6. **PNG goldens will move** for the four non-PR panes (`artifacts_beads_populated`,
   `artifacts_files_populated`, `artifacts_files_nested_strip`,
   `artifacts_plans_*_populated`, `artifacts_stitches_*`, `artifacts_split_*`, and any
   Artifacts golden whose selected row has relations). Patch goldens should **not** move
   — if they do, the panel's rendering or CSS diverged, so fix that rather than
   regenerating. Run `just test-visual`, inspect every actual/expected/diff triple under
   `.pytest_cache/sase-visual/`, and regenerate only goldens whose sole difference you
   have actually looked at (retired task `sase-lo`: blanket
   `--sase-update-visual-snapshots` silently absorbs unrelated drift).
7. Exit condition, checked by hand in `sase ace`: on Beads, `<` from a phase selects its
   epic and `>` from an epic selects a phase; on Stitches, `<` selects the commit's
   parent; `L` on Beads still opens the linked plan and on Plans still opens the linked
   bead;
   `grep -rn '"patches"\|"beads"\|"stitches"' src/sase/ace/tui/actions/navigation/_tree.py src/sase/core/artifact_relation_layout.py src/sase/ace/tui/widgets/artifacts/relation_panel.py`
   returns only the documented legacy tab shim.

## Risks

- **Patch parity is the tripwire.** Five behaviors are easy to get subtly wrong:
  ancestors traversing _through_ hidden entries, hidden descendants pruning subtrees,
  the depth-0 single-leaf `>`, the pre-order counter spanning the whole tree, and the
  27-entry cap that increments before skipping. The parity procedure in step 9 exists
  because of this.
- **Five panes gain a widget in their list column.** That is the PNG churn, and it also
  shortens every list by a few rows. Check the `-split-narrow` 80x24 goldens
  specifically — that is where a panel can push the list to zero visible rows. If it
  does, cap the panel's height on the narrow split rather than hiding it.
- **`RelationKeymap` replaces three attributes read across the navigation mixins.** Grep
  for `_ancestor_keys`, `_children_keys`, `_sibling_keys` before and after; a missed
  reader silently degrades to "no relations".
- **The Beads/Plans link actions change their source of truth.** If the index's
  `plans`/`beads` edge disagrees with the old snapshot lookup for any row, `L`
  regresses. Test both directions against a snapshot that has a live and a closed bead.
- **`grouping` (`sase-m6.7.1.5`) runs in parallel and also touches pane files.** D10 is
  the contract; adding the panel to a pane's `compose` and calling one method from its
  detail path is the smallest possible footprint in the files both phases can see.
- **`reveal` (`sase-m6.7.1.4`) lands next and is already unblocked-in-flight.** Keep
  `_change_query_for_navigation` untouched and keep the hand-off a single named pane
  hook, so that phase replaces one override instead of rewriting the driver.

## Out of scope

- Any keymap or `default_config.yml` edit, any new action name, and any `y`/`R`/`s`/`L`
  /`o` rename — `sase-m6.9`.
- The `RelationReveal` lens record, the reveal chip, the generalized query rewrite, and
  making dangling targets non-invalidating at the query layer — `reveal`.
- `GROUPING`, `GroupFoldRegistry`, banner rows, and `EntryJumpAnchor` widening —
  `grouping`.
- Extending the conformance harness with relation checks, the synthetic `notes`
  fixture's `ref.relations` block, and the contract documentation — `conform`.
- `PaneCapability.STATUS_COUNTERS` and `SHELL`'s stale `later_phase_reserved` verdict —
  `sase-m6.10`.
- The Agents tab's own neighbor navigation and reveal — outside the Artifacts contract.

## Proposed follow-ups to record on the bead

Record these with `sase bead note sase-m6.7.1.3 'PROPOSED FOLLOW-UP: …'` while
implementing; the epic's land agent triages them.

- A pane-supplied status vocabulary for relation rows, so Beads/Files rows can carry a
  state glyph instead of the host's Patch-only status table (`STATUS_COUNTERS`,
  `sase-m6.10`).
- A second declared `HIERARCHY` pair per pane gaining its own key mode, once a pane
  declares one (D1 renders it today with no keys).
