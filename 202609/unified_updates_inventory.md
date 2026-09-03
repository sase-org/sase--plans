---
tier: tale
title: Implement the unified Updates inventory surface
goal:
  The Updates tab presents every SASE package, plugin, and agent CLI in one
  scope-filtered master/detail inventory while preserving existing actions and
  responsive navigation.
size: medium
proposed_by: bbugyi200.apollo.sase-w0.2
bead: sase-w0.2
create_time: 2026-09-03 12:26:10
status: wip
---

- **PARENT:** [202609/unified_updates_tab_1.md](unified_updates_tab_1.md)
- **BEAD:** sase-w0.2

# Implement the unified Updates inventory surface

## Objective

Complete phase `sase-w0.2` by replacing the Updates pane's Core / Plugins / Agent CLIs
sub-tabs with one domain-sectioned master/detail list. Preserve all existing action
names and keys while changing their availability from active-sub-tab checks to the
capabilities of the highlighted `UpdateRow`. Keep the digest/header and unified-mark
semantics owned by later epic phases; this phase supplies their shared widget and row
surface without implementing those phases early.

## Implementation

1. Extend the row projection with the presentation-only scope model and selector. Define
   `UpdateScope = Literal["outdated", "installed", "all"]`, the ordered scope tuple,
   section metadata, and `select_rows(rows, scope, needle)`. Filter against the
   precomputed `UpdateRow.haystack`, include row errors in `outdated`, omit empty
   sections, keep the SASE / built-in plugins / community plugins / agent-CLI order, and
   sort each section outdated-first then by casefolded label. Add a single
   `updates-row__<row.key>` identity prefix while retaining disabled header identities.

2. Replace pane-local sub-tab state with one scope and one bookmark. Change
   `UpdatesSessionState` from `active_subtab`, `plugins`, and `agent_clis` to
   `scope="installed"` and `rows`; retain `agent_cli_history_all`. Initialize one
   `ProgrammaticSelectionGuard`, one row-identity detail dedup key, and row-keyed lookup
   maps in `PluginsBrowserPane`. Rename bracket binding descriptions/actions to scope
   cycling without changing their physical keys, and remove all Updates-specific
   `UpdatesSubTab` exports and `_active_subtab` state.

3. Recompose `PluginsBrowserLayoutMixin` as one surface: a reusable `#updates-scopes`
   `PanelTabStrip` with live counts, `#updates-header`, `#updates-filter-input`, one
   `#updates-list`, one `#updates-detail`, conditional `#updates-history`, and one
   `#updates-hints`. Remove the `ContentSwitcher`, old sub-tab containers, and old
   duplicated widget IDs. A scope click or bracket cycle must reset jump state, cancel
   stale detail work, rebuild through the same selection path, persist the scope, and
   focus the unified list.

4. Consolidate grouping, options, and selection in `PluginsBrowserRenderingMixin`. Make
   `_rebuild_options` the only programmatic highlight assignment path: capture the
   bookmark, clear/rebuild options and O(1) row-key maps, use
   `restore_selection_by_identity`, prefer the first outdated row on first landing, call
   the guard's `prepare()` immediately before assigning `highlighted`, and consume its
   echo synchronously in the sole `OptionHighlighted` handler. Record one session
   bookmark and repaint cheap hints immediately; keep detail painting behind
   `DetailPanelDebouncer` and re-read the live highlighted row when the callback runs.

5. Dispatch the unified row and detail renderers by `UpdateRow.kind`. Reuse the plugin
   detail/incoming/latest code, the agent-CLI detail and history builders, and the core
   cells/mode/incoming helpers. Core detail becomes per-package; history is hidden and
   cleared for non-CLI rows. Adapt lazy plugin latest/incoming row patching to the
   unified IDs and maps without reclassifying scope or rebuilding the list. Consolidate
   current-entry/highlight helpers so existing mutation mixins still receive the same
   plugin or CLI payloads.

6. Collapse navigation, filter, jump, status, visibility, and action gates onto the
   unified surface. `/` searches every row kind through `row.haystack`; `'` allocates
   one cross-domain jump space; j/k and detail scrolling always use the single list and
   scroll container. Make pane-wide actions available pane-wide, gate row-scoped
   install/uninstall/update/history actions by the selected row's capabilities, and
   retain `Space` behavior for the current phase's still-separate install and CLI mark
   sets. Replace the three hint producers with one hint line and update all widget
   selectors and status/visibility paths.

7. Collapse the Updates CSS from three sub-tab layouts and two master/detail pairs to
   the unified IDs, keeping the list panel width at 58 and allowing the hints area to
   grow. Update pane helpers and focused tests to select scopes and query the unified
   widgets. Add row-selection and pane journeys covering exact-once membership across
   scopes, manual-only and error rows in Outdated, cross-domain filtering and jumping,
   identity restoration after refresh/filter/scope changes, and capability-correct
   action gates.

8. Repoint the TUI scale benchmark to open the `all` scope and preserve its 16 ms p95
   limit at n=2000. Update the existing Updates PNG scenarios to use scopes, regenerate
   only the intentionally changed goldens, and inspect visual diffs. Confirm no
   Updates-pane `_active_subtab`, retired sub-tab widget ID, or dual-list selector
   remains in `src/`.

## Verification

- Run focused row, pane, session-state, jump, action, detail, loading, history, and
  visual-fixture tests while iterating.
- Run `SASE_TUI_PERF=1` on `tests/ace/tui/bench_plugins_catalog_scale.py` and verify j/k
  p95 remains below 16 ms at every scale, including n=2000.
- Run the dedicated PNG suite with `just test-visual`; rebaseline intentional Updates
  images with `--sase-update-visual-snapshots`, then rerun for exact-pixel success.
- Run `just check` as the required repository verification lane.
- Before closing only `sase-w0.2`, run `sase bead epic-symbols sase-w0.2`, resolve or
  re-key every remaining entry, then close it with a note naming the successful checks.
