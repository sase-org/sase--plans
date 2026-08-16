---
tier: epic
title: Relations, reveal, and grouping as Artifacts contract features
goal: Hierarchy, family, and link relations are declared contract facts rendered by
  one host-owned relation panel on every Artifacts pane; reveal is a reversible lens
  that never destroys a composed query; and every pane's grouping runs on the one
  shared GroupFoldRegistry with collapsed banners as first-class navigation targets.
  A sidecar that names a property gets jumpers and grouping without shipping code.
phases:
- id: vocabulary
  title: Relations and grouping become declared contract facts
  depends_on: []
  size: medium
  description: 'vocabulary: replace the PaneRelationDecl/PaneGroupingDecl placeholders
    with real records, derive RELATIONS and GROUPING from named rules, add the additive
    ref.relations and ref.grouping declaration blocks, and surface both in the pane
    explainer.'
- id: index
  title: The host-owned relation index and its built-in sources
  depends_on:
  - vocabulary
  size: large
  description: 'index: build one immutable per-snapshot RelationIndex over the three
    primitives with derived inverses, cycle detection, and dangling diagnostics, then
    supply it from built-in sources for all five panes plus a generic declared-property
    source.'
- id: panel
  title: One host-owned relation panel and generalized jumpers
  depends_on:
  - index
  size: large
  description: 'panel: replace AncestorsChildrenPanel with a shell-owned RelationPanel,
    generalize the ancestor/child/sibling key modes off Patch onto any pane with the
    RELATIONS capability, and route cross-pane edges through the shared entry request.'
- id: reveal
  title: Reveal as a reversible lens
  depends_on:
  - panel
  size: medium
  description: 'reveal: keep the query rewrite as the mechanism but wrap it in a lens
    record with a visible return affordance, generalize the hard-coded Patch rewrite
    terms onto the contract, and make dangling targets diagnostics instead of pane
    invalidations.'
- id: grouping
  title: Every pane's grouping on the shared fold registry
  depends_on:
  - vocabulary
  size: large
  description: 'grouping: give every pane a GroupFoldRegistry, convert the four panes
    that render non-selectable headers into foldable banners that are first-class
    navigation and jump targets, and migrate Beads off its inverted expanded-epic
    set.'
- id: conform
  title: Conformance, docs, and the relation performance gate
  depends_on:
  - reveal
  - grouping
  size: medium
  description: 'conform: extend the conformance harness and the synthetic third-party
    fixture with relation and grouping checks, document the relation and banner grammar,
    and hold the navigation performance gate on every converted pane.'
proposed_by: bbugyi200.athena.sase-m6.7
parent_bead: sase-m6.7
create_time: 2026-08-16 02:53:05
status: wip
bead_id: sase-m6.7.1
---

- **PROMPT:** [prompts/202608/artifacts_relations_and_grouping.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_relations_and_grouping.md)
- **PARENT:** [202608/artifacts_pane_contract.md](artifacts_pane_contract.md)
- **BEAD:** [sase-m6.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.7.1.md)

# Plan: Relations, reveal, and grouping as Artifacts contract features

This is the child epic for phase `relations` of epic `sase-m6`
(`plan:202608/artifacts_pane_contract.md`). Everything the parent plan says about that
phase is inherited verbatim; this plan decides how it is cut into work that distinct
agents can complete.

## Why this shape

Patch's jumpers are the richest feature in the Artifacts tab and the least obviously
generalizable, and the parent epic's own words are the reason this cannot be one pass:
"when a target is outside the current result set, navigation **rewrites the query** — so
the jumper and the query language are one feature." Three separate things have to be
true before a jumper can generalize, and each is a different kind of work:

1. a pane must be able to **say** what its relations and groups are (declaration),
2. the host must be able to **compute** them once per snapshot without provider code
   (indexing),
3. the host must be able to **render and drive** them identically everywhere (panel,
   reveal, banners).

Splitting on those seams is what keeps the hot-path risk contained. The parent epic
names the relation panel as "the specific performance risk in this epic," and the
mitigation — "preserve the prebuilt-index discipline rather than rebuilding per
selection" — is only checkable if the index is a separate, immutable, testable artifact
built before any widget consumes it. `index` therefore lands before `panel`, and `panel`
is not allowed to compute an edge.

Grouping is severable from all of that: it needs the contract vocabulary and nothing
else. `GroupFoldRegistry` "needs a third consumer, not a design," so `grouping` depends
only on `vocabulary` and runs beside the `index → panel → reveal` chain.

```text
vocabulary ── index ── panel ── reveal ──┐
     └──────── grouping ────────────────┴── conform
```

## Grounding

Verified in this workspace at `172b1a1a0`. Line numbers are from that tree.

| Fact                                                   | Evidence                                                                                                                                                                                                |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The relation/grouping contract fields are placeholders | `PaneRelationDecl`, `PaneGroupingDecl`, `PaneStatusCounter` (`_artifact_tab_model.py:160-181`)                                                                                                          |
| Nothing populates them                                 | `_assemble_contract` hard-codes `relations=()`, `grouping=PaneGroupingDecl()` (`_artifact_tab_contract.py:371-372`)                                                                                     |
| `RELATIONS`/`GROUPING` are reserved, not derived       | both bound to `_rule_later_phase` (`_artifact_tab_contract_rules.py:474-475`)                                                                                                                           |
| They have no host actions, and conformance knows it    | `CAPABILITY_HOST_ACTIONS[...] = ()` (`_artifact_tab_actions.py:59-62`); special-cased in `harness.py`                                                                                                   |
| Patch's relation widget is Patch-shaped                | `AncestorsChildrenPanel`, 612 lines, typed on `Patch`/`PatchGraphIndex` (`widgets/ancestors_children_panel.py`)                                                                                         |
| Its driver is tab-gated, not capability-gated          | `_is_patch_tree_tab()` compares `current_tab` against three literals (`actions/navigation/_tree.py:11-16`, 357 lines total)                                                                             |
| Reveal already exists, unnamed and unadvertised        | `_change_query_for_navigation` writes hard-coded `ancestor:`/`sibling:` tokens and pushes onto the literal `"patches"` history stack, with no lens record and no return affordance (`_tree.py:293-357`) |
| The relation oracle is frozen but Patch-only           | `test_relation_goldens.py` asserts `cases["cross_kind_edges"] == []`                                                                                                                                    |
| The fold registry is generic and under-used            | `GroupFoldRegistry` is 101 lines, keyed on `tuple[str, ...]` (`models/group_fold.py`); consumers are Agents and Patches only                                                                            |
| Beads folds with its own, inverted mechanism           | `_expanded_epics: set[tuple[str, str]]` — an _expanded_ set (`beads_navigation.py:84`, `beads_options.py:43`)                                                                                           |
| Three panes render headers that cannot fold at all     | `Option(..., disabled=True)`: `files_list.py:63-70`, `plans_list.py:259-272`, `commits_timeline.py:167-175`                                                                                             |
| Provider-declared blocks are already an accepted shape | `ref.capabilities` validates in Python and degrades visibly (`_artifact_tab_contract_provider.py:26-82`)                                                                                                |
| A new `ref` key must be registered on both paths       | `_KNOWN_REF_CONFIG_KEYS` (`sidecar_ref_config.py:66-80`) — `capabilities` is listed; an unlisted key fails inline `ref:`                                                                                |
| Relation graph work is Python-owned today              | `graph_index_facade.build_patch_graph_index` dispatches straight to the Python builder                                                                                                                  |
| The typed cross-pane target already exists             | `ArtifactEntryTarget(pane_id, parts)` with a token round-trip (`widgets/artifacts/entry_navigation.py:31-89`)                                                                                           |
| Cross-pane routing already exists                      | `_request_artifacts_entry` switches pane and requests the target (`actions/artifacts_navigation.py:116-126`)                                                                                            |
| Relations data exists and is unreachable               | Beads `phases_by_epic`/`plan_links` (`beads_data_models.py:108,111`), Files `LogicalFile.versions`, `VcsCommitWire.parent_ids`, `bead_plan_links.BeadPlanLink`                                          |

## Decisions a phase worker must not silently revert

**1. The relation index is Python-owned host logic, not Rust — for now.** The parent
epic's own constraint is that the Rust provider-spec wire stays at schema version 1
because CI installs the published floor core, and bead `sase-lm` is a live instance of
that failure. `graph_index_facade` is the existing precedent: patch graph indexing is
explicitly Python-owned and dispatches to the Python builder. Put `RelationIndex` in
`src/sase/core/` so a CLI or future frontend can import it without Textual, shape the
entry point like `graph_index_facade` so a Rust backend can be dropped in later, and do
not open a `sase-core` change inside this epic.

**2. Family is not hierarchy.** Patch "siblings" are a normalized base name including
reverted variants (`strip_reverted_suffix`), not graph siblings. Collapsing family into
a parent pointer loses Patch semantics. Three primitives stay three.

**3. This epic ships no keymap renames.** `y`, `R`, `s`, `L` and `o` belong to
`sase-m6.9`, which is blocked on this phase precisely so it can be deferred or reverted
independently. Link jumps therefore land behind each pane's _existing_ binding routed
through the contract (Beads' open-plan, Plans' open-bead), not behind a unified `L`. The
reveal return affordance introduces **no new binding** at all — it reuses `prev_query`
(`^`), which `sase-m6.6` already namespaces per pane. If a phase worker finds themselves
editing `default_config.yml`'s Artifacts keymap block, they have taken `sase-m6.9`'s
work.

**4. Providers name properties; they never name UI.** A relation declaration whose
`source` is not a declared `ref.properties` key is a validation error that degrades the
pane visibly — not a silently dropped edge. Providers never supply a traversal callback,
a key, a colour, or a widget. Anything needing a real traversal (Beads' dependency
graph) is a built-in source gated by capability.

**5. Cross-pane targets exist in the edge from day one**, even where the first UI only
opens them. `ArtifactEntryTarget` already carries `pane_id`; an edge that stores a bare
name instead is the retrofit the parent epic paid to avoid.

**6. Nothing computes an edge on the keystroke path.** `update_relationships_from_index`
exists so 100 selections do not rebuild the graph 100 times. The `RelationIndex` is
built once per snapshot on the loader thread and is immutable; the panel reads it.

## Inherited non-negotiable constraints

These come from the parent epic and hold in every phase here.

- Providers declare facts, never UI. No callbacks, widgets, colours, keybindings,
  command strings, or Python entry points in sidecar config, ever.
- No provider code runs during render, navigation, completion, or query evaluation.
- The Rust provider-spec wire stays at schema version 1. `ref.relations` and
  `ref.grouping` are Python-side, additive, and optional.
- A broken provider degrades visibly, never silently. One malformed relation block must
  not remove a tab, and must never remove another tab.
- Read `sase/memory/tui_perf.md` before implementing any phase. Hold `SASE_TUI_PERF=1`
  navigation p95 under 16 ms on every converted pane, measured after each conversion.

## Seam ownership

`grouping` runs in parallel with the `index → panel → reveal` chain, and both touch
navigation. To keep the land merge clean, ownership is assigned rather than negotiated:

- `panel` owns `actions/navigation/_tree.py`, `widgets/ancestors_children_panel.py`,
  `actions/patch/_display.py`, and the relation entries in `_artifact_tab_actions.py`.
- `grouping` owns `models/group_fold.py`, `actions/patch/_grouping_nav.py`,
  `actions/navigation/_fold.py`, the four list builders, and the grouping entries in
  `_artifact_tab_actions.py`.
- `actions/navigation/jump_hints.py` and `actions/artifacts_navigation.py` are shared.
  `grouping` owns the `EntryJumpAnchor` widening for banner anchors; `panel` adds no new
  anchor type and reuses `ArtifactEntryTarget`. Neither phase reformats a function the
  other owns.

## Phases

### vocabulary — Relations and grouping become declared contract facts

Give the contract a real vocabulary before anything computes or renders against it.

_Records._ Replace the `PaneRelationDecl` placeholder with a frozen record carrying
`name`, `kind` (a closed `RelationKind`: `HIERARCHY`, `FAMILY`, `LINK`), `label`,
`source` (a declared property name for providers, or a host source id for built-ins),
`target_pane` (a pane id, or `None` for same-pane), `inverse` (the derived inverse's
name, or `None`), and the `directed`/`transitive` flags the three primitives differ on.
Replace `PaneGroupingDecl`'s single `keys` tuple with an ordered tuple of grouping modes
(`id`, `label`, `keys`) plus a `default_mode`. Both `_presentation_digest` and
`_presentation_digest_for` read those fields today and must be updated together, or two
digests for the same contract will disagree.

_Derivation._ Retire `_rule_later_phase` for these two capabilities and write named
rules beside the existing ones: `relations_from_declared_edges` (ON when at least one
relation declaration survives validation) and `grouping_from_declared_modes` (ON when at
least one mode is declared). Both stay OFF on a degraded pane through the existing
`degraded_safe_host` rule, and both remain suppressible with a reason string through
`ref.capabilities.suppress`, which already works generically.

_Declaration._ Add `ref.relations` and `ref.grouping` as additive, optional Python-side
blocks at `DOCUMENT_REF_PROVIDER_SPEC_SCHEMA_VERSION`, validated in
`_artifact_tab_contract_provider.py` in the same shape as
`extract_provider_suppressions` — returning `(value, error, code)` with codes
`invalid_ref_relations` and `invalid_ref_grouping` so a malformed block produces a
named, degraded tab rather than a missing feature. **Add both keys to
`_KNOWN_REF_CONFIG_KEYS` in `sidecar_ref_config.py`** or an inline `ref:` declaration
will fail validation while a `use:`-provided plugin spec silently succeeds; the two
paths already diverge, and `capabilities` is the worked example. Validation rejects a
relation whose `source` is not in `ref.properties`, a `target_pane` that is not a
configured pane id, and any key outside the closed set.

_Built-ins._ Add `relations` and `grouping` fact fields to `_BuiltinAdapter` and fill
them for Stitches, Patches, Beads, Files, and the Plan adapter. This phase declares
facts only — no source computes an edge and no widget reads one yet.

_Explainer._ Fold both into `explanation_payload()` and extend `sase artifact pane show`
(text and `--json`) to print every relation with its kind, source, target pane, and
derived inverse, and every grouping mode with its keys. Relation and grouping entries
gain fields, so bump `PANE_SHOW_SCHEMA_VERSION` to 2 and update any golden that pins the
payload.

Exit condition: `sase artifact pane show patches` explains why `RELATIONS` is ON and
names the three Patch relations; `sase artifact pane show ref:notes` explains why it is
OFF and names the rule.

### index — The host-owned relation index and its built-in sources

Compute every edge once per snapshot, off the event loop, with no provider code.

_The index._ Add `src/sase/core/artifact_relations.py`: a `RelationEdge` (kind, relation
name, label, `source: ArtifactEntryTarget`, `target: ArtifactEntryTarget`), an immutable
`RelationIndex` with `edges_for(target)` returning the three primitives already grouped,
and a `RelationDiagnostic` list. The module imports no Textual widget so a CLI or a
future frontend can use it. Derive inverses (a declared `parent` edge yields the
`children` edge set), detect cycles during construction rather than during traversal,
and record an edge whose target is named but absent from the snapshot as a **dangling**
edge with a diagnostic — never drop it and never raise.

_The sources._ A `RelationSource` ABC turns one already-loaded pane snapshot into a
`RelationIndex`. It is pure: no I/O, no globbing, no provider code, no Git.

- **Patch** wraps the existing `PatchGraphIndex`. `_find_ancestors`, `_find_siblings`,
  and `get_children` semantics move here unchanged, including cycle-break-on-visited and
  the "parent not in list" terminal case. `hide_reverted` is a presentation filter
  applied by the panel, not a property of the index — today it is baked into the
  traversal, and baking it in would make one index per display option.
- **Beads** emits hierarchy epic→phase from `phases_by_epic` and link edges for `deps`
  and the resolved plan link from `plan_links`.
- **Files** emits a family over `LogicalFile.versions`.
- **Stitches** emits hierarchy commit→parent from `VcsCommitWire.parent_ids` and a
  cross-pane link commit→patch targeting `pane_id="patches"`.
- **Documents/Plans** emits hierarchy proposal→active→archive and the cross-pane
  plan↔bead link from `BeadPlanLink`.
- **Generic provider** emits edges from `ref.relations` over declared properties, plus
  one host-owned filename family rule: a bundle laid out as `<name>/<name>__a.md`,
  `<name>__b.md`, `<name>.md` is a family whose parent is the unsuffixed document. That
  rule is what makes the research sidecar's swarm bundles the first non-Patch family on
  a real third-party provider with no sidecar change, no frontmatter, and no schema
  bump.

_Wiring._ Each pane builds its index in the same worker pass that builds its rows, keyed
by the pane's existing snapshot generation, and hands it to the pane as one immutable
value. Add `tui_trace` spans mirroring
`widget.ancestors_children.update_relationships_from_index` so a regression is
attributable.

_Oracle._ Extend `goldens/relations/cases.json` and `test_relation_goldens.py` from a
Patch-only probe onto `RelationIndex` directly, add per-pane cases for each built-in
source, and replace the `cross_kind_edges == []` assertion with the real
Stitches→Patches and Plans↔Beads edges. Keep every existing Patch case byte-identical:
this phase must not change a single Patch relation answer.

Exit condition: every pane can be asked for its relations without a widget, and the
Patch golden set is unchanged.

### panel — One host-owned relation panel and generalized jumpers

The host owns the UI; providers supply only edges.

_The panel._ Add `widgets/artifacts/relation_panel.py`: a `RelationPanel` that renders
sections per relation kind using the contract's declared labels, on the contract's
accent, in the shared shell's grammar (`docs/artifacts_pane_visual_grammar.md`). Key
assignment, the multi-key buffer, hint rendering, hidden counts, and the empty fallback
are all host-owned. Preserve Patch's key grammar exactly — `<`, `<<`, `<a`; `>`, `>>`,
`>a`, `>2a`, `.`; `~`, `~~`, `~a` — because it is the owner's primary surface and the
goldens pin it. Dangling edges render as visibly dangling links with their diagnostic;
they never invalidate the pane.

Delete `AncestorsChildrenPanel` rather than aliasing it. A compatibility alias keeps 612
lines of Patch-typed traversal alive behind the new name, and the two tests that
subclass it (`test_changespec_detail_only_refresh.py`, `test_changespec_graph_index.py`)
are better retargeted at the panel's real seam. Update the lazy entry in
`widgets/__init__.py`/`.pyi` and the `_w_ancestors_children` attribute name with it.

_The driver._ Generalize `TreeNavigationMixin`: `_is_patch_tree_tab()` becomes a
`contract.has(PaneCapability.RELATIONS)` check, `_tree_patches()` becomes the pane's
relation index, and `_navigate_to_patch_compat` becomes a target-based
`_navigate_to_relation_target` operating on `ArtifactEntryTarget`. Keep the
`changespec`/`_navigate_to_changespec` legacy aliases working. A target on another pane
routes through `_request_artifacts_entry`; a target on this pane that is visible is
selected directly; a target on this pane that is filtered out hands off to `reveal`,
which lands next — until then, report it as dangling.

_Surfaces._ Populate `CAPABILITY_HOST_ACTIONS[RELATIONS]` with `start_ancestor_mode`,
`start_child_mode`, `start_sibling_mode`, and the link-jump action, and shrink the
conformance harness's four-capability special case accordingly. Relation hints reach the
footer through the pane's `conditional_footer_entries()`; the help modal's per-pane
relation section is generated from the contract's declarations rather than hand-written
(observe the 57-character box rules in `src/sase/ace/CLAUDE.md`).

_Performance._ Measure `SASE_TUI_PERF=1` navigation p95 on Patches and on one converted
provider pane before and after, and record both numbers on the phase bead. A panel that
rebuilds an index per selection fails this phase regardless of test results.

Exit condition: Beads epic→phase and Stitches commit→parent are reachable with `<`/`>`,
and no relation code branches on a pane id.

### reveal — Reveal as a reversible lens

Jumping through a graph must not destroy a composed query.

Keep the query rewrite as the underlying mechanism and give it an identity. A
`RelationReveal` record captures the origin query's source text, its canonical form, the
profile digest that parsed it, the origin selection target, and the relation that caused
the jump. The shell renders a chip in the identity/scope lane — built by a new
`shell.build_reveal_chip`, pure like every other shell renderer — that names the
relation and the way back. Returning restores the source text, the canonical form, and
the origin selection.

**No new keybinding.** `sase-m6.6` already namespaces query history per pane
(`query_history.json` as `{pane_id: {prev, next}}`), so the rewrite pushes the origin
record onto that stack and `prev_query` (`^`) is the way back; the chip advertises it.
This keeps `sase-m6.9` free to make every keymap decision. If the per-pane history stack
turns out not to be wired for a given pane, add the named action and leave it `unbound`
in `default_config.yml` — do not choose a key here.

Generalize the rewrite itself. `_change_query_for_navigation` hard-codes `ancestor:` and
`sibling:` — Patch's tokens. Drive it from the relation declaration plus the pane's
compiled query profile: a relation maps to a profile field, and the rewrite is built
through the profile's canonicalizer so it round-trips. A pane whose profile has no field
for that relation does not write an invalid query; it selects the target if visible and
reports it as dangling otherwise.

The lens clears when the user edits the query themselves or when the profile digest
changes, so a stale "return" can never restore a query that no longer parses.

Exit condition: revealing a filtered-out ancestor from a composed Patch query and
returning restores the exact composed query and the original selection.

### grouping — Every pane's grouping on the shared fold registry

`GroupFoldRegistry` is already generic and already keyed on `tuple[str, ...]`. It needs
consumers, not a design.

_Registry per pane._ Move the Patches-only `_patch_group_fold_registry` into a per-pane
map held by the app, keyed by pane id, keeping the old attribute as a property alias so
`actions/changespec/_grouping_nav.py`'s legacy accessors keep working. Call
`clear_unknown()` once per refresh on every pane, as Patches already does, so a group
whose last member disappeared cannot re-apply a stale collapse.

_Banners._ Add one shared banner row model and renderer in the shell so every pane's
group header looks the same. A collapsed banner is a first-class navigation stop and
jump target — this is the behavior Patches has and the other four panes do not.
`grouping` owns widening `EntryJumpAnchor` for the shared banner anchor (subsuming
`PatchBannerJumpAnchor`), per the seam-ownership rule above.

_Conversions._

- **Beads** is the one with a real semantic trap: `_expanded_epics` is an _expanded_
  set, so epics default to collapsed, while `GroupFoldRegistry` stores the _collapsed_
  set and groups default to expanded. Migrating naively flips every Beads epic open on
  first paint. Preserve today's behavior by seeding the collapsed set from the loaded
  epics on each snapshot before applying user state.
  `test_ace_png_snapshots_artifacts_beads.py` waits on `pane._expanded_epics` and must
  move with it.
- **Files** date separators (`file_group_header`, `disabled=True`) become foldable
  banners.
- **Plans/Documents** `_section_option` headers become foldable banners.
- **Stitches** day headings (`build_timeline_day`) become foldable banners.
- **Patches** keeps its behavior exactly and moves onto the shared registry accessor.

_Declared grouping._ A provider's `ref.grouping` modes render through the same registry.
Providers name declared properties; the host owns banner text, keys, order, and colour.

_Surfaces._ Populate `CAPABILITY_HOST_ACTIONS[GROUPING]` with the fold/expand/collapse
and grouping-cycle actions and shrink the harness special case for `GROUPING`.

_Performance._ Banners are emitted in the same pass that builds rows — no second O(n)
sweep, and nothing that walks the group tree per keystroke. `build_patch_tree` is
rebuilt several times per fold action today; do not extend that pattern to four more
panes without measuring.

Exit condition: `h`/`l` fold a Files date group and a Stitches day, collapsed banners
are `j`/`k` stops on every pane, and Beads still opens with its epics collapsed.

### conform — Conformance, docs, and the relation performance gate

Close the child epic by making the two new capabilities enforceable rather than
documented.

Extend `tests/ace/tui/artifacts_contract/harness.py` with checks parametrized over every
resolved sub-tab: every declared relation resolves to a reachable `ArtifactEntryTarget`
or a recorded diagnostic; every declared grouping mode produces banners that are
navigable and foldable; every relation and grouping action the contract names is
registered. Extend the synthetic `notes` fixture
(`tests/ace/tui/artifacts_contract/fixtures/notes/`) with a `ref.relations` declaration
and a second, suffixed document so a third-party provider — not a built-in — is the
conformance case for both the declared-property edge and the filename family. The
fixture stays data-only: the existing `test_synthetic_fixture_is_declarative_data_only`
assertion is the guard.

The harness's four-capability exemption shrinks to `STATUS_COUNTERS` and `SHELL`.
`SHELL` still resolves through `_rule_later_phase` even though `sase-m6.5` landed
(`_artifact_tab_contract_rules.py:477`); that is out of this epic's scope — record it on
the phase bead as `PROPOSED FOLLOW-UP:` for `sase-m6.10`, do not fix it here.

Document the grammar: extend `docs/artifacts_pane_visual_grammar.md` with the relation
panel's layout slot and the banner row's treatment, and write the three relation
primitives, the declaration blocks, and the reveal lens into the contract documentation
that `sase-m6.10` completes.

Hold the gate: `SASE_TUI_PERF=1` navigation p95 under 16 ms on every converted pane,
captured per `docs/perf_runbook.md`, with the numbers recorded on the bead.

Expect Artifacts PNG goldens to move — the relation panel and banner rows both change
pixels. Inspect `.pytest_cache/sase-visual/` artifacts and review each diff. Retired
task `sase-lo` warns that blanket `--sase-update-visual-snapshots` silently absorbs
unrelated drift; regenerate only goldens whose sole difference you have actually looked
at.

## Verification

Every phase runs `just install` first — workspaces are ephemeral and dependencies drift.

`vocabulary` is narrow enough for `just check` inline. `index`, `panel`, `reveal`,
`grouping`, and `conform` all touch `tests/ace/tui/test_artifacts_*` broadly, so
`just check`'s scoped lane will escalate: run `just check-full` through `/sase_monitor`
(`sase monitor start --command 'just check-full' …`) with a `--next` action so the
follow-up agent acts on the result. It routinely outruns a single agent turn and must
never be run inline.

Measure `SASE_TUI_PERF=1` navigation p95 after each pane conversion, not only in
`conform` — a regression found at the end is a regression with five phases of suspects.

## Risks

- **The relation panel is the hot-path risk**, and the parent epic says so explicitly.
  The mitigation is structural: `index` produces an immutable per-snapshot value before
  `panel` exists, so "rebuilds per selection" is a reviewable property rather than a
  benchmark result.
- **The Beads fold polarity inversion is a silent behavior change.** Expanded-set to
  collapsed-set is the one migration in this epic that looks correct and renders wrong.
  It gets an explicit seeding step and a visual golden.
- **`grouping` runs parallel to the `panel` chain and both touch navigation.** Mitigated
  by the seam-ownership section; if the owner prefers a clean serial merge, `grouping`
  can simply be run after `reveal` without any plan change.
- **Reveal depends on `sase-m6.6`'s per-pane history actually being wired.** If a pane's
  history stack is missing, the fallback is a named-but-unbound action, not a key choice
  — key choices belong to `sase-m6.9`.
- **Providers gaining relations enlarges the plugin blast radius.** A malformed
  `ref.relations` block degrades that one tab visibly and never removes another.

## Rejected alternatives

**Collapse family into hierarchy.** One primitive, less code — and it loses Patch
semantics, because Patch siblings are a normalized base-name family including reverted
variants, not graph siblings. The parent epic rules this out by name.

**Let providers supply a traversal callback.** Maximum expressiveness for a sidecar
author; forfeits the guarantee that no provider code runs during navigation, which is
the property the whole contract exists to buy.

**Model `RelationIndex` in Rust now.** Correct destination eventually, and wrong inside
this epic: it needs a `sase-core` release plus a floor ratchet before CI can see it, and
bead `sase-lm` is the live instance of that failure mode. `graph_index_facade` is the
existing Python-owned precedent for exactly this kind of graph work.

**Give Beads its own fold registry.** Least disruptive; leaves two fold mechanisms and
guarantees the next grouping feature is written twice. The registry needs a third
consumer, not a design.

**Bind a new "return to query" key now.** Reads better than a chip pointing at `^`, and
it would take a keymap decision that `sase-m6.9` was deliberately severed to own.

**Ship relations and grouping as one phase.** They share the contract vocabulary and
nothing else; fusing them serializes two independent large efforts behind one agent and
puts the epic's stated hot-path risk in the same tree as four list-builder rewrites.

## Follow-up, explicitly out of scope

- A Rust `RelationIndex` behind the `graph_index_facade`-shaped entry point.
- Folding `ref.relations` and `ref.grouping` into the `ref.pane` block — that is
  `sase-m6.8`'s payload, and it should keep accepting both spellings during migration.
- `PaneCapability.STATUS_COUNTERS`, and the `SHELL` capability's stale
  `later_phase_reserved` verdict — both belong to `sase-m6.10`.
- The Agents tab's own reveal (`actions/navigation/_agent_reveal.py`). Agents is outside
  the Artifacts contract; converting it is its own change.
