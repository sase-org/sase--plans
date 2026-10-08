---
tier: tale
title: Updates tab — drop the scope sub-tabs and keep one All view
goal:
  The Admin Center Updates tab shows one merged All inventory with no
  Outdated/Installed/Available sub-tabs and no [ / ] scope cycling.
size: medium
proposed_by: bbugyi200.apollo.5x
create_time: 2026-10-08 13:56:08
status: wip
---

# Updates tab: drop the scope sub-tabs and keep one "All" view

## Goal

The SASE Admin Center **Updates** tab (`PluginsBrowserPane`) currently has a four-way
scope strip (**Outdated / Installed / Available / All**, default **Installed**), cycled
with `]` / `[`. The user wants a single view. Remove the scope sub-tabs entirely. The
tab always shows what the **All** sub-tab shows today: every SASE core package, plugin,
and registered agent CLI, grouped into the existing **SASE**, **Plugins · Built-in**,
**Plugins · Community**, and **Agent CLIs** sections. Inside each section, rows with an
update sort first, then by label, exactly as `select_rows` already does.

## Background: why `[` / `]` breaks on "Available"

This is context only. The fix is to delete the feature, not to patch it.

`PluginsBrowserLayoutMixin._set_scope` (in
`src/sase/ace/tui/modals/plugins_browser_layout.py`) rebuilds the list, then calls
`_sync_state_visibility()`. That sets `#updates-list` to `display = False` whenever the
scope has no item rows (for example, "Everything is installed." on Available). It then
calls `focus_default()`, which tries to focus that hidden `OptionList`. Textual cannot
focus a widget that is not displayed, so focus leaves the pane. The pane-local `[` / `]`
bindings are no longer in the binding chain, and the `ConfigCenterModal` screen stops
the chain, so the keys do nothing. Without scopes, this path no longer exists.

## Non-goals

- No feature flag. The scopes are session-only UI state with no external callers to
  migrate, and the user explicitly asked for removal (see `sase_flags` policy: flags are
  for old branches that must stay reachable).
- Do not touch the unrelated `sase.ace.update_scope.UpdateScope` enum
  (EVERYTHING/SASE/PROVIDERS). It is used by the comprehensive-update preview and
  execution code. Only the Updates-pane `UpdateScope` **Literal** in
  `plugins_browser_rows.py` goes away.
- Do not touch the `H` agent-CLI history toggle (`toggle_history_scope`). It is a
  different "scope" and stays.
- Do not change the mark semantics, mark-all (`*`) section rules, filter matching, jump
  (`'`), header digest, or row sort order beyond what removing the scope dimension
  implies. `*` keeps marking "every visible same-section row", and "visible" now means
  "matches the filter".
- Do not add a replacement count line where the strip was. The header digest already
  summarizes update counts.

## Source changes

### `src/sase/ace/tui/modals/plugins_browser_rows.py`

- Delete the `UpdateScope` Literal, `SCOPE_ORDER`, `SCOPE_LABELS`, `_row_in_scope`, and
  `scope_counts`.
- Change `select_rows(rows, *, needle)` to drop the `scope` keyword and the
  `_row_in_scope` filter. Keep the section grouping, empty-section omission, and
  `(not update_available, label.casefold())` sort unchanged. Update its docstring.

### `src/sase/ace/tui/modals/plugins_browser_layout.py`

- Stop composing the `PanelTabStrip(... id="updates-scopes")`. The header `Static` is
  followed directly by the filter input.
- Delete `_scope_tabs`, `_refresh_scope_strip`, `_on_scope_clicked` (the
  `@on(PanelTabStrip.TabClicked)` handler), `_set_scope`, `_cycle_scope`,
  `action_cycle_scope`, and `action_cycle_scope_reverse`.
- Remove the `_scope` / `_session_state` TYPE_CHECKING attribute declarations that are
  no longer used, plus now-unused imports (`PanelTab`, `PanelTabStrip`, `on`, `cast`,
  and the `plugins_browser_rows` scope imports). Update the module and class docstrings,
  which mention "scope navigation" and "scope strip".

### `src/sase/ace/tui/modals/plugins_browser_pane.py`

- Remove the two `BINDINGS` entries `("right_square_bracket", "cycle_scope", ...)` and
  `("left_square_bracket", "cycle_scope_reverse", ...)`.
- Remove `self._scope: UpdateScope = self._session_state.scope` and the `UpdateScope`
  import.

### `src/sase/ace/tui/modals/plugins_browser_rendering.py`

- `_rebuild_groups` calls `select_rows(self._rows, needle=...)` with no scope.
- `_render_all` no longer calls `_refresh_scope_strip()`. Remove its TYPE_CHECKING stub
  if one exists.
- Drop the `_scope: UpdateScope` TYPE_CHECKING declaration and the `UpdateScope` import.

### `src/sase/ace/tui/modals/plugins_browser_status.py`

- `_status_message`: delete the per-scope empty branches ("Nothing needs an update." /
  "Nothing is installed." / "Everything is installed."). With no scope, the list is
  empty only when there are no rows (handled earlier as "No updates found.") or when a
  filter matches nothing.
- `_no_match_message`: collapse to the generic `"Nothing matches the current filter."`.
  There is no other scope to point at, so drop the "N matches in X ([ / ] to switch
  scope)" suggestion and its loop. If the method becomes a one-liner, inline it.
- `_hints`: remove the `_SCOPE_NAV_HINT` part.
- Drop the `_scope` TYPE_CHECKING declaration and the `SCOPE_LABELS`, `SCOPE_ORDER`,
  `UpdateScope`, and `select_rows` imports (whatever becomes unused) along with the
  `_SCOPE_NAV_HINT` import.

### `src/sase/ace/tui/modals/plugins_browser_constants.py`

- Delete `_SCOPE_NAV_HINT` and its comment.

### `src/sase/ace/tui/modals/plugins_browser_input.py`

- Remove the `left_square_bracket` / `right_square_bracket` interception from
  `PluginsFilterInput.on_key`. While the filter has focus, `[` and `]` become ordinary
  characters typed into the filter. Keep the `escape` handling. Update the class
  docstring, which says "Brackets cycle the pane-local scopes even while the filter owns
  focus".

### `src/sase/ace/tui/modals/config_center_session.py`

- Delete the `scope: UpdateScope = "installed"` field from `UpdatesSessionState` and its
  TYPE_CHECKING `UpdateScope` import. If `Literal` becomes unused there, drop it from
  the `typing` import too. Admin Center persistence does not serialize this field
  (verify with a grep of `_admin_center_persistence.py`), so no migration is needed.

### `src/sase/ace/tui/styles.tcss`

- Delete the `PluginsBrowserPane #updates-scopes { ... }` rule. It spans two lines in
  the "Updates tab: one merged inventory" block.

### Help popup: `src/sase/ace/tui/modals/help_modal/binding_common.py`

`ADMIN_CENTER_UPDATES_SECTION` is already stale. Remove these two entries:
`("Core / Plugins / Agent CLIs", "Three update sub-tabs")` and
`("] / [", "Next / previous sub-tab")`. Keep every other entry. Respect the help-box
width rules in `src/sase/ace/CLAUDE.md`: descriptions stay at 32 characters or fewer.

After the edits, run
`rg -n "cycle_scope|_set_scope|SCOPE_ORDER|SCOPE_LABELS|scope_counts|_row_in_scope|_SCOPE_NAV_HINT|updates-scopes|_refresh_scope_strip|_scope_tabs" src tests`.
It must return nothing that refers to the Updates pane. `memory_pane_*` and
`config_transaction.py` have their own unrelated `cycle_scope` methods; leave those
alone.

## Docs

- `docs/ace.md` "Updates Tab" section (around the "A scope strip cycled with `]` / `[`"
  sentence): replace the sentence with one saying the list always shows every row
  (installed or not), with updatable rows first in each section. Also rewrite the later
  sentence "A filter that matches nothing in the current scope names the scopes holding
  matches, with `[` / `]` to switch." It should say a filter that matches nothing shows
  `Nothing matches the current filter.`
- `docs/configuration.md` "Updates tab" section:
  - Rewrite the "Use `]` / `[` to cycle a scope strip ... reads
    `Everything is installed.`" sentences the same way.
  - Delete the `]` / `[` row from the keymap table.
  - Rewrite the `/` row: drop "a filter matching nothing in the current scope names the
    scopes with matches".
  - Run `just fmt` so the table re-aligns.
- `docs/agent_providers.md` (around "one-scope-away **Available** installs"): reword so
  installs are described as being right in the same list, for example "inline installs
  for missing CLIs with `*` mark-all". Drop "one-scope-away" and **Available**.
- Run `rg -n -i "scope strip|Outdated / Installed|one-scope-away|switch scope" docs` and
  fix any remaining hit about the Updates tab.

## Tests

Use `git mv` for the renames so history follows.

### Helpers

- `tests/ace/tui/_plugins_browser_pane_helpers.py::_open_plugins_pane`: drop the `scope`
  parameter and the `pane._set_scope(scope)` call, plus the `UpdateScope` import.
- `tests/ace/tui/visual/_ace_config_center_modal_helpers.py::_open_plugins_modal`: drop
  the `scope` parameter and the `pane._set_scope(scope)` call.
- Remove every call-site `scope=` argument to these helpers.

### `tests/ace/tui/test_plugins_browser_rows_scopes.py` → `git mv` to `tests/ace/tui/test_plugins_browser_rows_select.py`

- Delete the scope-only tests:
  - `test_row_in_scope_outdated_installed_and_all`
  - `test_scope_counts_ignore_filter_and_count_each_row_once`
  - `test_scope_order_cycles_outdated_installed_available_all`
  - `test_available_scope_holds_only_not_installed_rows`
  - `test_scope_counts_available_against_all_and_installed`
  - `test_select_rows_available_scope_lists_missing_clis`
- Keep the rest, updating `select_rows(..., scope="all", ...)` calls to the new
  signature.
- Add one test: `select_rows` returns installed **and** not-installed rows (for example
  a not-installed plugin, a missing agent CLI, and an installed plugin with an update).
  Within a section, the updatable row sorts before the others.

### `tests/ace/tui/test_plugins_browser_pane_scopes.py` → `git mv` to `tests/ace/tui/test_plugins_browser_pane_inventory.py`

- Delete these tests:
  - `test_scope_membership_includes_manual_cli_error_and_excludes_available`
  - `test_scope_strip_counts_and_tab_click_select_scope`
  - `test_available_scope_lists_only_not_installed_rows`
  - `test_scope_cycle_walks_outdated_installed_available_all`
  - `test_empty_available_scope_reads_everything_installed`
  - `test_filter_miss_names_scopes_with_matches`
- Replace them with:
  - A pane test that the list holds both installed and not-installed rows, for example
    `plugin:nvim` (not installed), `plugin:github`, and `cli:codex`, and that
    `#updates-scopes` is not mounted (`not pane.query("#updates-scopes")`).
  - A pilot test of the original bug. Open the Updates tab with the list focused, press
    `right_square_bracket` and then `left_square_bracket`. The visible row ids and the
    highlighted row are unchanged, focus is still on `#updates-list`, and the modal is
    still on the `updates` tab.
- Adapt the remaining tests:
  - `test_cursor_survives_refresh_filter_and_scope_by_identity`: rename it to drop
    "scope", remove the `_set_scope` steps, and keep the refresh and filter identity
    assertions.
  - `test_install_mark_survives_scope_switch_away_from_available_rows`: rewrite it so
    the mark survives a filter that hides the row and then clears, or delete it if
    `tests/ace/tui/test_plugins_browser_pane_marks.py` already covers that.
  - `test_filter_miss_everywhere_keeps_generic_message`: drop the
    `_set_scope("installed")` line.
  - `test_empty_identity_open_lands_on_first_outdated_row`: the default view is now All,
    so confirm the expected first-updatable row still holds, and adjust the expected id
    if the SASE/Plugins ordering changes it.

### `tests/ace/tui/test_plugins_browser_pane_agent_clis.py`

- Delete `test_updates_scopes_cycle_and_gate_row_actions`, but keep its
  `check_action("update_agent_clis")` assertion in a surviving test or a small new one.
- Delete `test_updates_scope_cycling_handles_brackets_from_core_and_lists`. The new
  pilot test above replaces it.
- `test_agent_cli_session_restores_scope_and_row_by_identity`: rename to
  `..._restores_row_by_identity`, and drop `state.updates.scope = "all"` and the
  `pane._scope` assertions.
- `test_updates_scope_hints_share_projects_wording`: rename, for example
  `test_updates_hints_omit_scope_navigation`, and assert `"[ ] scope" not in hints` and
  `"a sync agents" in hints`.
- `test_update_sase_action_remains_pane_wide`: drop the `SCOPE_ORDER` loop and just
  assert one `action_update_sase()` call starts the preview.
- Remove the `SCOPE_ORDER` import.

### `tests/ace/tui/test_plugins_browser_pane_loading.py`

- `test_plugins_session_restores_plugin_by_identity`: drop `state.updates.scope`, the
  `scope="all"` helper argument, and the `state.updates.scope == "all"` assertion.
- `test_updates_filter_forwards_brackets_and_tab_switches_main_tab`: rename, for example
  `test_updates_filter_types_brackets_and_tab_switches_main_tab`. With the filter
  focused, pressing `left_square_bracket` now types `[` into the filter
  (`filter_input.value == "["`) and the tab stays `updates`. Then clear the value or
  press `escape`, and keep the `tab` main-tab assertions.

### `tests/ace/tui/test_plugins_browser_pane_marks.py`

- The test around line 59 that uses `_set_scope("installed")` to hide a marked row
  should hide it with the filter (`_apply_updates_filter`-style helper) instead, keeping
  its "marked but not visible still installs" intent.
- The mark-all test around line 347 uses the Installed scope to hide qwen/muse. Express
  the same "mark-all only marks visible rows" intent with a filter, or restructure it so
  `*` on `cli:claude` marks only same-section `mark_update` rows. In the All view,
  qwen/muse are visible but carry `install`, not `mark_update`, so they are not targets
  anyway. Update the comment to match.

### Other test files

- `tests/ace/tui/test_plugins_browser_pane_all_current.py`: drop the `SCOPE_ORDER` loop
  and assert the banner once.
- `tests/ace/tui/test_plugins_browser_pane_jump.py` (around line 185): `_set_scope`
  served only to trigger a row rebuild that clears jump state. Use a filter change or
  `action_refresh()` instead, or delete it if
  `test_updates_filter_change_clears_jump_hints` already covers it.

### Visual PNG snapshots

- Delete these scope-only snapshot tests and `git rm` their goldens under
  `tests/ace/tui/visual/snapshots/png/`:
  - `test_config_center_updates_available_scope_png_snapshot`
    (`config_center_updates_available_scope_120x40.png`) in
    `tests/ace/tui/visual/test_ace_png_snapshots_config_center_agent_cli_install.py`
  - `test_config_center_updates_outdated_scope_cli_only_png_snapshot`,
    `test_config_center_updates_outdated_scope_plugin_only_png_snapshot`, and
    `test_config_center_updates_outdated_scope_all_current_png_snapshot`
    (`config_center_updates_outdated_scope_{cli_only,plugin_only,all_current}_120x40.png`)
    in `tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugins.py`

  The All view is already covered by `config_center_updates_all_current`,
  `config_center_updates_digest`, `config_center_plugins_tab`, and friends.

- Every remaining Updates golden changes. The two strip rows go away, so the content
  moves up. The footer hint line loses `[ ] scope`. Goldens that opened the tab without
  `_open_plugins_modal` (and so used the old **Installed** default) now also show
  not-installed rows. Regenerate them with `just fix-tui-screenshots` through
  `/sase_monitor`, with a generous timeout. A targeted
  `-- tests/ace/tui/visual -k "config_center_updates or config_center_plugins or config_center_agent_cli"`
  run is fine, plus any help-modal goldens if the Admin Center Updates help section is
  visible in one.
- Inspect the report. Every update should be "strip gone, rows shifted up, `[ ] scope`
  hint gone, not-installed rows now present". Any other difference is a bug.

## Verification

1. Run the `rg` sweep from the source section.
2. Run `just fmt`, then `sase tool run check`. Read `lint_and_test` first: it covers
   symvision for removed symbols, mypy for removed TYPE_CHECKING attributes, and ruff
   for unused imports.
3. Run the targeted PNG regeneration above and inspect its report.
4. Optional live check with `sase screenshot`: open the Admin Center (`#`), press `8`,
   and confirm there is no scope strip and that not-installed rows appear alongside
   installed ones. Press `]` / `[` and confirm nothing changes.
