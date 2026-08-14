---
tier: tale
title: One typed entry target on every Artifacts pane
goal:
  Every Artifacts pane uses pane-aware, serializable row identity for navigation, marks,
  jump history, deep links, and Patch reference copy.
size: medium
proposed_by: bbugyi200.athena.sase-m6.3
bead: sase-m6.3
create_time: 2026-08-14 18:02:10
status: done
---

- **PROMPT:**
  [prompts/202608/artifact_entry_identity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_entry_identity.md)
- **PARENT:** [202608/artifacts_pane_contract.md](artifacts_pane_contract.md)
- **BEAD:**
  [sase-m6.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.3.md)

# One typed entry target on every Artifacts pane

## Goal

Replace the Artifacts tab's convention-based tuple identities and Patch-only index state
with one immutable `ArtifactEntryTarget` value that names both its pane and its row,
round-trips through a canonical string token, and is implemented by every live Artifacts
pane. Patch must become a normal entry navigator without changing its query, grouping,
detail, or mutation behavior. Marks and Patch jump history must survive row reordering
and refreshes, Files and Stitches deep links must stop silently degrading, and the Patch
pane must expose its already-supported `@patch:` reference in copy mode.

This is Python/TUI identity and adapter work. It does not change the Rust core, provider
schema, query syntax, visual grammar, or the later `ArtifactsPaneContract` capability
layer.

## Design

### Typed identity and canonical token

- Replace the tuple alias in `src/sase/ace/tui/widgets/artifacts/entry_navigation.py`
  with a frozen, slotted `ArtifactEntryTarget` dataclass containing `pane_id` and an
  immutable `parts` tuple.
- Give it `to_token()` and `from_token()` methods using one documented, versioned,
  delimiter-safe encoding. Validate malformed tokens, empty pane ids, and non-string
  components so the output can safely become a JSON persistence key in the later query
  phase.
- Keep a narrow legacy-tuple construction path for staged migration and tests. Map the
  old leading kinds (`commit`, `bead`, `file`, `patch`, and document ref kinds) to their
  owning pane, but make all production row-target helpers construct explicit typed
  targets in this phase.
- Add focused unit coverage for equality/hash behavior, legacy conversion, arbitrary
  delimiter and Unicode content, deterministic token output, valid round-trips, and
  invalid tokens.

### One abstract navigator, including Patch

- Convert `ArtifactEntryNavigator` from a structural `Protocol` to an `ABC` with the
  complete navigation, deferred-request, jump-hint, mark, and conditional-footer
  surface. Make concrete Artifacts panes inherit it so omissions fail at construction or
  type checking instead of being papered over at call sites.
- Update the row helpers with explicit pane identities: `patches/(project, name)`,
  `stitches/(repo, full_sha)`, `beads/(project, row_kind, bead_id)`,
  `files/(logical_id)`, and `ref:<kind>/(project, row_kind, document_identity)`. Add
  public `patch_row_target()` and `commit_row_target()` helpers and replace Stitches'
  private inline tuple construction.
- Implement the missing `request_entry_target()` and `conditional_footer_entries()`
  methods on Files and Stitches. Deferred requests must be consumed when the next
  matching row model arrives; footer entries may be empty but must be explicit.
- Implement the navigator on `ArtifactsPatchesPane` by adapting the existing app-owned
  Patch list and grouping state. Selection resolves a stable target against the current
  row model, focuses the list synchronously, clears banner focus when selecting a row,
  and uses existing refresh/detail paths. Jump hints and marks are translated only at
  the render boundary; no I/O or data-scaled work is added to key handlers.
- Remove the Patch rejection in `ArtifactsView.entry_navigator()`. Replace the two
  `getattr` fallbacks in `artifacts_navigation.py` with direct ABC calls, and derive a
  cross-pane request's destination from the target's `pane_id` so pane identity is no
  longer duplicated by convention at deep-link call sites.

### Stable marks and jump anchors

- Include `patches` in the app-owned `_artifacts_marked_targets` map and route every
  Artifacts `action_toggle_mark` / clear operation through the navigator. Preserve the
  Patch mark action's selection behavior while storing membership by
  `ArtifactEntryTarget`, not by list index.
- Change Patch display, row patching, footer counts, bulk status changes, and bulk agent
  launches to resolve marked targets against the current Patch rows. Remove the
  `marked_indices` reactive state and stop clearing Patch marks during ordinary reloads;
  stale targets simply do not resolve until their row is present again.
- Update command-context mark counts and the `AcePage` / `AcePageGroup` state and
  isolation snapshots to use the active pane's stable targets. Update focused harnesses
  and annotations that still model Patch marks as indices.
- Replace Patch row entries in `EntryJumpAnchor` with `ArtifactEntryTarget` while
  retaining the existing typed collapsed-banner anchor. Snapshot the selected Patch via
  `patch_row_target`, validate it against the current row model, and resolve it back to
  the current index on restore. Keep AXE's unrelated index navigation isolated from the
  Artifacts anchor type. Update back/forward history tests to prove Patch anchors still
  restore the same row after reorder, discard removed rows, and preserve banner history.

### Patch artifact-reference copy target

- Add a `reference` target to the Patch copy-mode registry and default keymaps, dispatch
  it through the existing nonblocking clipboard delivery path, and emit `@patch:<name>`
  for the selected Patch (or visible marked Patches in visual order where marked-copy
  behavior applies).
- Update the generated/default copy-key model, copy palette dispatch coverage, command
  availability expectations, footer/help copy-mode documentation, and focused copy
  tests. Do not add resolver work to the UI thread; `@patch:` is already a live built-in
  artifact kind.

## Compatibility and invariants

- Existing target tuple callers remain accepted only at the dataclass boundary; all
  checked-in production helpers emit typed targets.
- Pane identity is part of equality and serialization, so identical parts in different
  panes never collide and cross-pane requests cannot select the wrong adapter.
- Target creation, lookup, mark toggling, and jump restoration are pure in-memory work.
  Build lookup maps when a snapshot/list is replaced and keep navigation synchronous to
  preserve the ACE p95-under-16-ms requirement.
- Marks survive refresh and re-sort. Row actions that need concrete Patches operate only
  on targets currently resolvable in visual order and handle stale targets without
  indexing errors.
- No pane-specific `getattr` fallback remains for navigator methods, and the Patch pane
  is returned by `ArtifactsView.entry_navigator()` like every other live pane.

## Verification

1. Add/run focused tests for target token and legacy construction, every row-target
   helper's pane identity, ABC completeness, Files/Stitches deferred deep links, Patch
   navigator selection, cross-pane plan/bead requests, stable Patch marks across
   reorder/refresh, stable Patch back/forward anchors, marked bulk operations, command
   context, and the Patch `@patch:` copy target/help/footer registry.
2. Run the existing Artifacts contract/conformance fixtures and the broad navigation,
   marking, copy, command-catalog, and Patch display suites touched by the migration.
3. Run `just install`, then `just check`. Because this phase broadly changes
   `tests/ace/tui/test_artifacts_*`, run `just check-full` through `/sase_monitor` with
   a `--next` action, as required by the epic design, and act on its terminal result.
4. Re-run `git diff --check`, inspect the final diff for leftover production tuple
   targets, `marked_indices`, navigator `getattr` fallbacks, and Patch index anchors,
   then re-run the smallest affected tests after any cleanup.
