---
tier: tale
title: Every Artifacts pane on the shared fold registry
goal:
  Every declared Artifacts grouping mode renders and navigates through one host-owned
  fold and banner system without regressing Patch or Beads behavior.
size: medium
proposed_by: bbugyi200.athena.sase-m6.7.1.5
bead: sase-m6.7.1.5
create_time: 2026-08-16 03:24:30
status: wip
---

- **PARENT:**
  [202608/artifacts_relations_and_grouping.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifacts_relations_and_grouping.md)
- **BEAD:**
  [sase-m6.7.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.7.1.5.md)

# Plan: Every Artifacts pane on the shared fold registry

## Goal

Make the declared `GROUPING` capability operational for every Artifacts pane. Every pane
must own fold state through `GroupFoldRegistry`, emit the same host-rendered banner row,
treat collapsed banners as stable `j`/`k` and jump-hint targets, expose the existing
fold and grouping-cycle actions through the contract, preserve Patches behavior, and
preserve Beads' default-collapsed epic tree.

## Constraints and current seams

- Keep grouping state and rendering host-owned. Provider declarations supply only mode
  ids, labels, and property keys; they do not supply widgets, callbacks, colors, keys,
  or executable grouping logic.
- Do not change `src/sase/default_config.yml`; keymap unification belongs to the later
  keymap phase. Reuse the existing `h`/`l`, `H`/`L`, and grouping-cycle actions and the
  pane-specific Beads aliases.
- Keep Patch grouping answers and legacy changespec accessors compatible. Move the
  active Patch registry behind the shared per-pane registry ownership without changing
  `build_patch_tree` semantics.
- Build grouped rows once per option refresh. Navigation and jump handling consume the
  resulting row/anchor indexes and must not rebuild or rescan a group tree per
  keystroke.
- Call `GroupFoldRegistry.clear_unknown()` exactly once for each pane snapshot refresh,
  after the full set of group keys for that refresh is known.
- Preserve Beads' polarity: existing epics start collapsed. Seed newly loaded epic group
  keys into the collapsed registry while retaining user changes for already-known keys,
  so refreshing does not reopen or reclose groups unexpectedly.
- Keep grouping work inside the ownership assigned by the epic design: fold models and
  routing, list builders, shared banner anchors, and grouping capability actions. Do not
  modify relation-panel ownership files or introduce keymap renames.

## Implementation

1. Introduce the shared Artifacts grouping model and shell renderer.
   - Add a frozen banner/row record carrying pane id, grouping mode, stable group key,
     level, label, member count, collapsed state, and the member entry targets needed
     for focus restoration.
   - Add pure grouping helpers that bucket already-loaded rows from a declared
     `PaneGroupingModeDecl`, build banners and visible rows in one pass, and return the
     known group keys plus stable navigation/jump indexes.
   - Add the single shell banner renderer using the pane contract's accent and the
     shared fold glyph/count/hint grammar. Expanded banners remain non-selectable;
     collapsed banners are selectable and carry a stable banner anchor.
   - Widen `EntryJumpAnchor` from the Patch-only banner variant to the shared Artifacts
     banner anchor while retaining legacy Patch/changespec token compatibility.

2. Centralize per-pane fold and grouping-mode state and route host actions through it.
   - Initialize one per-pane, per-mode `GroupFoldRegistry` map and active-mode/focused-
     banner state keyed by pane id. Keep `_patch_group_fold_registry` and changespec
     properties as compatibility aliases onto the Patches entry.
   - Add shared helpers for current registry lookup, snapshot reconciliation,
     expand/collapse at focus, layer-wise expand/collapse-all, mode cycling, focus
     re-anchoring, and jump target generation.
   - Route the existing fold and grouping-cycle actions to the active Artifacts pane
     when its contract has `PaneCapability.GROUPING`; keep Agents and AXE behavior
     unchanged. Permit those actions on non-Patch Artifacts panes without adding or
     renaming bindings.
   - Populate `CAPABILITY_HOST_ACTIONS[GROUPING]` with the actual registered fold,
     collapse, expand, and cycle actions, and remove `GROUPING` from the conformance
     harness's later-phase exemption.

3. Convert each built-in pane without changing its entry identity or detail behavior.
   - Patches: use the shared registry accessor and shared banner anchor/renderer while
     preserving its three grouping modes, nested family/date banners, navigation order,
     legacy aliases, and fold-layer semantics.
   - Beads: replace `_expanded_epics` with collapsed registry state; make epic groups
     and section banners participate in the shared banner/navigation model; preserve
     default-collapsed epics, filter-driven phase reveal, selected-target restoration,
     and the existing Beads `h`/`l` actions.
   - Files: group the already-loaded logical rows according to the active declared mode
     (source, kind, or project), replacing disabled date separators with shared foldable
     banners and retaining newest-first row ordering inside groups.
   - Plans/Documents: group proposal/active/archive/provider document rows by the active
     contract mode, including arbitrary provider-declared property keys, replacing the
     hard-coded disabled section options with shared foldable banners while preserving
     counts, filters, pending-entry reveal, and stable targets.
   - Stitches: replace disabled timeline day headings with shared banners, support the
     declared date/repository/author modes from the loaded commit entries, and preserve
     selected commit, marks, hints, and guarded OptionList highlight updates.

4. Make banner focus and jump behavior uniform.
   - Extend each pane navigator's cached visible-order model to include collapsed banner
     targets. `j`/`k` moves through row and banner stops without selecting a hidden
     member or scheduling a detail update for banner-only focus.
   - Generate entry hints for visible entry targets and collapsed banner anchors in one
     render-order sequence; restore back/forward jump anchors to the exact pane, mode,
     and group key, degrading to the closest visible member when a group disappeared.
   - On collapse, move focus to the deepest visible collapsed banner that contains the
     selected row. On expansion, move to the first visible child banner or member. On
     filter, refresh, or mode changes, clear stale banner focus and stale fold keys.

5. Cover behavior, compatibility, and the hot path.
   - Add model and renderer tests for registry reconciliation, declared-property
     grouping, stable shared banner anchors, collapsed/expanded rendering, and nested
     layer operations.
   - Add focused pane tests for Files and Stitches `h`/`l`, Plans and provider-declared
     grouping, Beads default-collapsed polarity and refresh persistence, Patch behavior
     parity, `j`/`k` banner stops, jump/back-forward restoration, capability action
     registration, and no pane-id-specific provider path.
   - Update the Beads PNG fixture to wait on registry state, run the affected visual
     snapshot tests, inspect every diff artifact, and accept only intentional banner
     changes.
   - Add or extend navigation performance coverage so all five converted panes exercise
     repeated `j`/`k` over a prebuilt visible-stop cache. Record `SASE_TUI_PERF=1` p95
     for every pane and require each result to remain below 16 ms.

## Verification

1. Run `just install` before repository checks.
2. Run focused model, pane, contract-conformance, navigation, jump-history, and visual
   tests during implementation; inspect PNG actual/expected/diff artifacts before
   updating an intentional golden.
3. Capture and record `SASE_TUI_PERF=1` navigation p95 for Patches, Beads, Files,
   Plans/Documents, and Stitches, each below 16 ms.
4. Run `just check` after edits. Because this phase touches broad Artifacts TUI tests,
   run `just check-full` through `/sase_monitor` with a continuation action and resolve
   all failures before closing the phase.
5. Recheck that the worktree contains no keymap-default changes and that Patch legacy
   aliases still pass. Close only `sase-m6.7.1.5` with a note listing the focused tests,
   full check, visual review, and measured per-pane p95 results.
