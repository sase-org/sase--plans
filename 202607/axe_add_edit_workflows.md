---
tier: tale
title: AXE add and edit workflows
goal: Users can add and safely edit AXE lumberjacks and chops from ACE with non-blocking
  persistence, selection restoration, and runtime reconciliation.
bead: sase-8m.3
parent: sase/repos/plans/202607/axe_config_editor.md
create_time: 2026-07-22 14:00:19
status: done
---

- **PROMPT:** [202607/prompts/axe_add_edit_workflows.md](prompts/axe_add_edit_workflows.md)

# AXE add and edit workflows

## Goal

Complete bead `sase-8m.3` by joining the exact-key AXE config backend and shared schema editor to the live ACE AXE tab.
Users must be able to add lumberjacks and chops, edit cached lumberjack/base-chop identities (including disabled and
generated-instance cases), retain recorded-output editing on `E`, and reconcile successful config writes with AXE,
selection, chezmoi, and optional git follow-up without introducing synchronous work on navigation or render paths.

## Implementation

1. Extend the configurable ACE action surface and its documentation.
   - Add the `add_axe_item` app keymap with default `a` across `AppKeymaps`, binding metadata, bundled
     `default_config.yml`, command metadata, help keymaps, and AXE guide content.
   - Keep `e` as the existing contextual `edit_spec` action but reinterpret its AXE branch as config editing for a
     selected lumberjack or chop. Reuse the existing `E`/`edit_panel` action for recorded chop output, extending its AXE
     command scope, label, aliases, and handler instead of adding a conflicting duplicate binding.
   - Update command availability and the conditional footer so `a` remains help/palette-only as an AXE-wide action, `e`
     appears on editable AXE config rows, `E` appears only when the selected chop has recorded output, and manual `r`
     excludes disabled chops. Add guard tests that keep keymaps, command metadata, help, and footer behavior aligned.

2. Add focused, keyboard-friendly AXE add-flow modals and pure add metadata.
   - Implement a compact Add to AXE chooser whose option order follows the cached selection: new chop under the selected
     lumberjack is primary on lumberjack/chop rows, while new lumberjack is primary on empty/background rows. When a
     chop has no contextual parent, follow with a cached lumberjack picker.
   - Load exact AXE composition plus chop-script inventory in a thread worker. Present discovered executable, source,
     resolution/configured state and a Custom script choice without doing config, directory, subprocess, or git work in
     the action handler.
   - Add a new-entry identity form that pre-fills a stable chop name from discovered `sase_chop_*` executables, permits
     the name and script to be edited before preview, and rejects duplicate base identities live using the exact
     inventory rather than generated runtime names. Preserve arbitrary otherwise-valid key characters.
   - Extend the reusable AXE editor seed/model only as needed to represent a new entry with intentionally touched
     initial values, so a discovered script/default lumberjack interval produces sparse set operations while ordinary
     edits continue to emit only user-touched fields.

3. Build an AXE config-action adapter around the landed backend and editor contracts.
   - Add an AXE action mixin that snapshots only the stable selected `AxeItemKey` on the keystroke path, performs
     inventory/schema/script loading off-thread, resolves generated instances to their immutable base selector, and
     constructs `AxeEntryEditorSeed` values from exact effective fields, target-layer contributions, provenance, and
     writable scopes.
   - Translate `SchemaFieldOperation` values to `AxeFieldOperation`, call `plan_axe_entry_edit`, and project its generic
     plus strict AXE diagnostics, effective entity before/after, target diff, legacy-list promotion, and missing-script
     warning into `ConfigTransactionPreview`. Support conflict reload by rebuilding inventory/seed while retaining the
     modal draft.
   - Apply through `apply_axe_entry_edit`, run chezmoi propagation when the written source differs from the home target,
     and return the applied path plus the verified post-write AXE running state. Keep every planner, YAML/file access,
     chezmoi operation, script lookup, and process probe in existing transaction workers or pump-free background tasks.

4. Enrich the AXE collector/cache and preserve honest runtime behavior.
   - Carry enabled state, executable/config metadata, generated-instance/base identity, and bounded run history in
     `ChopSnapshot`; collect disabled chops as well as enabled/runtime-expanded chops in the existing off-thread full
     refresh.
   - Render disabled rows with a quiet disabled chip, map `e` on a generated instance to its base chop with an explicit
     all-instances warning, and use cached enabled/base metadata for footer, command, and manual-run guards. Retain
     recorded-run rendering and ensure j/k, footer computation, list rebuilding, and dashboard rendering remain free of
     disk or inventory reads.
   - Show an empty-AXE call to action using the configured `add_axe_item` key while leaving background-command rows and
     controls usable.

5. Reconcile successful writes without stealing newer UI state.
   - Store a pending target `AxeItemKey` together with the pre-refresh selection identity, expand the target parent, and
     consume it in the existing coalesced full-refresh rebuild only when the user is still on the compatible AXE
     selection. Drop or avoid applying stale intent after a tab/selection change; preserve the normal off-tab saved
     identity behavior.
   - Always schedule an immediate config refresh after a write. If restart was requested and the verified post-write
     state is still running, reuse the established daemon restart worker with an AXE-config-edit source and report
     success/failure as a write-then-runtime outcome. If AXE stopped meanwhile, do not start it. For Save only while
     running, explicitly state that the daemon keeps its old config until restarted.
   - Build any dirty-config commit offer off-thread, then reuse the shared confirmation and tracked commit/pull/push
     helpers. Commit follow-up and daemon failure must not hide that the config write itself succeeded.

6. Add regression and integration coverage, then verify the repository.
   - Cover add chooser context, parent/script/identity selection, exact duplicate checks, edit seed/provenance mapping,
     generated-instance warnings, missing executables, disabled rows/manual-run exclusion, `e`/`E` dispatch and
     availability, sparse plan/apply projection, conflict reload, commit offers, daemon
     stopped/restart/save-only/failure outcomes, and pending-selection generation guards.
   - Extend collector, footer/help/catalog, selection, and `AcePage` tests. Explicitly patch config inventory, script
     discovery, subprocess, and git readers to prove that action opening and j/k rendering invoke none synchronously.
   - Update the existing deterministic empty-AXE visual golden if the required call-to-action changes it, inspecting the
     generated actual/expected/diff artifacts before accepting the snapshot.
   - Run focused AXE/config/keymap/command tests while iterating. Because source files change, run `just install` and
     finish with `just check`; also run the dedicated AXE visual test(s) affected by the empty-state update.

## Acceptance criteria

- `a` is configurable and opens the correct contextual add flow from empty, lumberjack, chop, and background-command
  states; discovered and custom scripts can produce a validated new chop without manual YAML editing.
- `e` opens a sparse exact-key config edit for lumberjacks, base chops, disabled chops, and generated instances; `E`
  still opens recorded chop output and only appears when output exists.
- Config write, chezmoi propagation, refresh, optional commit/push, and AXE restart/save-only handling are non-blocking,
  truthful, and conflict-safe. A runtime reconciliation failure clearly says the config was saved.
- Disabled chops stay visible but cannot be run manually, and generated rows retain a stable mapping to the edited base.
- The post-write refresh selects the intended entry and expands its parent only if no newer navigation supersedes it.
- Targeted tests and the repository-required checks pass, with no synchronous config/inventory/subprocess/git access on
  AXE action-opening, navigation, or rendering paths.
