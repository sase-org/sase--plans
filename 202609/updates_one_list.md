---
tier: tale
size: medium
title: "Updates tab phase `list`: one list, domain sections, and the scope filter"
goal:
  "The Admin Center Updates tab renders one master/detail surface over the merged
  `UpdateRow` model: domain sections in a single `OptionList`, a cycled Outdated /
  Installed / All scope filter on the existing `]` / `[` keys, kind-dispatched detail,
  one selection guard, one bookmark, one filter that searches all three domains, and one
  hint line. The Core / Plugins / Agent CLIs sub-tabs, their `ContentSwitcher`, and the
  second OptionList/detail pair are gone, and no `_active_subtab` reference remains
  under `src/`."
proposed_by: bbugyi200.apollo.sase-w0.2
bead: sase-w0.2
create_time: 2026-09-03 14:22:11
status: wip
---

- **PARENT:** [202609/unified_updates_tab_1.md](unified_updates_tab_1.md)
- **BEAD:**
  [sase-w0.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w0/sase-w0.2.md)

# Plan: One list, domain sections, and the scope filter

This is phase `list` of epic `sase-w0`, bead `sase-w0.2`.

This is the structural phase of the epic `plan:202609/unified_updates_tab_1.md` ("One
Updates tab"). Phase `rows` (`sase-w0.1`, landed as `f67169ea7`) already added the pure
`UpdateRow` model, capability derivation, and the per-load row build on the worker
thread. **Read the epic plan file first** — especially the `list` phase section and the
"Cross-cutting constraints every phase must respect" section — then implement this plan.

Two later phases depend on this one and must not be implemented here:

- `header` (`sase-w0.3`) owns the digest header, the promoted all-current banner,
  `PluginsLoadResult.core_error`, and the truthfulness invariant.
- `marks` (`sase-w0.4`) owns collapsing `_marked_install` + `_marked_agent_clis` into
  one row-key mark set, the global `Esc` clear, and the marked-work aggregate line.

This phase keeps **two** mark sets and only re-points them at the merged list.

## Required reading before you start

```bash
sase memory read tui_perf.md lint_and_test.md symvision.md \
  -r "Implementing sase-w0.2: merging the Updates-tab sub-tabs into one list"
```

Rule 12 (guard programmatic `OptionList` highlights, clear the guard synchronously —
never through `call_later`) is the highest-risk line in this phase. Rule 7 (debounce the
detail, never the highlight) and rule 4 (re-read the live scope and highlight when a
worker result lands) are the other two that this phase can break.

---

## 1. `src/sase/ace/tui/modals/plugins_browser_rows.py`

Add the scope model and the grouping projection to the existing row module. Everything
here stays pure and widget-free.

```python
UpdateScope = Literal["outdated", "installed", "all"]
SCOPE_ORDER: tuple[UpdateScope, ...] = ("outdated", "installed", "all")
SCOPE_LABELS: dict[UpdateScope, str] = {
    "outdated": "Outdated",
    "installed": "Installed",
    "all": "All",
}
```

**Section metadata** (replaces `_BUILTIN_GROUP` / `_COMMUNITY_GROUP` in
`plugins_browser_constants.py`), in render order:

| section             | header text                 | style                     |
| ------------------- | --------------------------- | ------------------------- |
| `sase`              | `── SASE ──`                | `bold dim`                |
| `plugins-builtin`   | `── Plugins · Built-in ──`  | `bold dim`                |
| `plugins-community` | `── Plugins · Community ──` | `bold {_COMMUNITY_STYLE}` |
| `agent-clis`        | `── Agent CLIs ──`          | `bold dim`                |

`_COMMUNITY_STYLE` comes from `sase.plugins.render_common` (the same import
`plugins_browser_rendering.py` uses today). Keep the exact `── {group} ──` shape so
`_HEADER_PREFIX` options and `_is_item` are unchanged.

**New public functions** (both gain non-test consumers in this phase, so they must be
public for symvision):

- `row_in_scope(row: UpdateRow, scope: UpdateScope) -> bool`
  - `outdated` → `row.update_available or row.error is not None`
  - `installed` → `row.installed`
  - `all` → `True`
- `scope_counts(rows: Sequence[UpdateRow]) -> dict[UpdateScope, int]` — one pass, used
  for the scope strip's live counts. Counts ignore the filter.
- `select_rows(rows, *, scope, needle) -> list[tuple[str, str, list[UpdateRow]]]` —
  returns exactly the `_grouped` shape the pane already uses
  (`(header text, header style, rows)`), with empty sections omitted. `needle` is
  already `.strip().casefold()`-ed by the caller and matched with
  `needle in row.haystack`. Within a section, sort by
  `(not row.update_available, row.label.casefold())`.

**Symvision fallout from the merge (do not skip):** the merged `_row_text` paints
`row.version_label`, so `plugin_version_label` and `agent_cli_version_label` lose their
cross-file consumers. Rename them to `_plugin_version_label` and
`_agent_cli_version_label` (both stay used in-file by the row builders) and drop the
now-dead imports in `plugins_browser_rendering.py` and `plugins_browser_agent_clis.py`.
`dev_state_label` stays public — `_core_note_cell` still imports it. `build_plugin_row`
stays public — `plugins_browser_latest.py` still imports it.

---

## 2. `src/sase/ace/tui/modals/plugins_browser_constants.py`

- `_DETAIL_PLACEHOLDER = "Select an entry to view its details."`
- delete `_SUBTAB_NAV_HINT`; add `_SCOPE_NAV_HINT = "[ ] scope"`
- delete `_BUILTIN_GROUP` / `_COMMUNITY_GROUP` (moved to the rows module)
- delete `_ITEM_PREFIX`; add `_ROW_PREFIX = "updates-row__"`
- keep `_HEADER_PREFIX` unchanged

Also delete the module-local `_ITEM_PREFIX = "agent-cli__"` in
`plugins_browser_agent_clis.py` and `plugins_browser_agent_clis_actions.py`, and the
module-local `_DETAIL_PLACEHOLDER` in `plugins_browser_agent_clis.py`.

---

## 3. `src/sase/ace/tui/modals/config_center_session.py`

- Delete the `UpdatesSubTab` literal and its `__all__` entry (grep confirms no consumer
  outside the Updates-pane modules and their tests).
- `UpdatesSessionState` becomes:

  ```python
  @dataclass
  class UpdatesSessionState:
      """Session-only cursor state for the Updates pane."""

      scope: UpdateScope = "installed"
      rows: SelectionBookmark = field(default_factory=SelectionBookmark)
      agent_cli_history_all: bool = False
  ```

  `plugins` and `agent_clis` are deleted. Import `UpdateScope` from
  `.plugins_browser_rows` under `if TYPE_CHECKING:` only — the module has
  `from __future__ import annotations`, so the annotation stays a string and the session
  module keeps its current (cheap) import graph.

Session state is process-local; there is no durable migration.

---

## 4. `src/sase/ace/tui/modals/plugins_browser_layout.py`

Delete `_SUBTAB_ORDER`, `_SUBTAB_WIDGET_IDS`, `_SUBTABS`, `_switch_to_subtab`,
`_cycle_subtab`, `action_cycle_subtab`, `action_cycle_subtab_reverse`, and
`_core_hints`.

### `compose`

```
yield Static(self._header_renderable(), id="updates-header", markup=False)
yield PanelTabStrip(self._scope_tabs(), self._scope,
                    uppercase_active=True, id="updates-scopes")
yield pane_module._PluginsFilterInput(
    placeholder="/ filter components, plugins, agent CLIs…",
    id="updates-filter-input")
with Horizontal(id="updates-panels"):
    with Vertical(id="updates-list-panel"):
        yield Static(self._status_message(), id="updates-status", markup=False)
        yield pane_module._PluginList(*self._create_options(), id="updates-list")
    with Vertical(id="updates-detail-panel"):
        with VerticalScroll(id="updates-detail-scroll"):
            yield Static(_DETAIL_PLACEHOLDER, id="updates-detail", markup=False)
            yield Static("", id="updates-history", markup=False)
yield Static(self._hints(), id="updates-hints", markup=False)
```

### Scope strip and cycling

```python
def _scope_tabs(self) -> tuple[PanelTab, ...]:
    counts = scope_counts(self._rows)
    return tuple(
        PanelTab(scope, f"{SCOPE_LABELS[scope]} {counts[scope]}", "#AF87FF")
        for scope in SCOPE_ORDER
    )

def _refresh_scope_strip(self) -> None:      # tolerate an unmounted strip
    self.query_one("#updates-scopes", PanelTabStrip).set_tabs(
        self._scope_tabs(), active_tab=self._scope)
```

- `@on(PanelTabStrip.TabClicked)` → `_set_scope(cast(UpdateScope, event.tab_id))` when
  the id is in `SCOPE_ORDER`.
- `action_cycle_scope` / `action_cycle_scope_reverse` step `+1` / `-1` through
  `SCOPE_ORDER`.
- `_set_scope(scope)`:

  ```python
  if scope == self._scope: return
  self.reset_jump_state(repaint=True)     # hints/back stack drop before rows change
  self._scope = scope
  self._session_state.scope = scope
  if self._detail_debouncer is not None:
      self._detail_debouncer.cancel()
  self._refresh_scope_strip()
  self._rebuild_groups()
  self._rebuild_options()
  self._sync_state_visibility()
  self._render_detail_now(force=True)
  self.focus_default()
  ```

  A scope switch is a deliberate full rebuild (perf rule 6): the default `installed`
  scope is ~13 rows.

### `check_action`

Drop `plugin_only`, `browse_only`, and every `_active_subtab` conjunct:

```python
if action == "update_sase":        return self._can_update_sase()
if action == "update_agent_clis":  return not self._loading and self._agent_cli_plan_worker is None
if action == "sync_agents":        return callable(getattr(self.app, "action_sync_agents", None))
if action == "switch_mode":        return self._can_switch_mode()
row_capability = {
    "install": "install", "toggle_install_mark": "install",
    "uninstall": "uninstall", "update": "update",
    "toggle_history_scope": "history",
}
if action in row_capability:
    row = self._highlighted_row()
    return row is not None and row_capability[action] in row.capabilities
if action in _ROW_NAV_ACTIONS:     return self._has_item_rows()
return super().check_action(action, parameters)
```

`_ROW_NAV_ACTIONS = {"next_option", "prev_option", "jump_to_entry", "toggle_mark", "scroll_detail_down", "scroll_detail_up", "scroll_to_top", "scroll_to_bottom"}`.
`_has_item_rows()` is `any(rows for _, _, rows in self._grouped)`. `focus_filter`,
`toggle_verbose`, `refresh`, `toggle_offline`, `cycle_scope`, and `cycle_scope_reverse`
fall through to `super()` and are pane-wide.

`on_mount` calls `_sync_state_visibility()` then `_sync_header()` (renamed from
`_sync_current_banner`) as it does today. `action_sync_agents` is unchanged.

---

## 5. `src/sase/ace/tui/modals/plugins_browser_controls.py`

- `_option_list()` → `#updates-list`; **delete** `_active_option_list()` and re-point
  `action_next_option`, `action_prev_option`, `focus_default`, and the jump mixin at
  `_option_list()`.
- `_detail_scroll()` always returns `#updates-detail-scroll` (no branch).
- `_detail_widget()` → `#updates-detail`.
- `action_focus_filter` / `_set_filter_value` / `on_input_changed` /
  `on_input_submitted` → `#updates-filter-input`.
- `action_toggle_verbose` repaints `#updates-hints`.
- Drop the `_active_subtab` entry from the `TYPE_CHECKING` block.

`_is_item`, the debounced filter path (`_schedule_filter_apply` / `_apply_filter`), and
`cancel_input` are otherwise unchanged.

---

## 6. `src/sase/ace/tui/modals/plugins_browser_input.py`

`PluginsFilterInput.on_key` forwards `[` / `]` to `pane.action_cycle_scope_reverse()` /
`pane.action_cycle_scope()`. Update the class docstring: brackets cycle the pane-local
**scopes**.

---

## 7. `src/sase/ace/tui/modals/plugins_browser_rendering.py` — the core of the phase

`_grouped` changes element type from `PluginCatalogEntry` to `UpdateRow` everywhere.

### Rebuild path

- `_rebuild_groups()` →
  `self._grouped = select_rows(self._rows, scope=self._scope, needle=self._filter_text.strip().casefold())`.
  **Delete** `_matches`, `_refresh_plugin_haystacks`, and `_plugin_haystack` (the
  per-row haystack is built once per load by the row model), plus the
  `_plugin_haystacks` field and the `_refresh_plugin_haystacks()` call in
  `plugins_browser_workers.py`.
- `_flat_plugin_entries()` → `_flat_rows() -> list[UpdateRow]`.
- `_create_options()` emits a disabled
  `Option(header_text, id=_HEADER_PREFIX+section_key, disabled=True)` per section
  followed by `Option(self._row_label(row_index, row), id=_ROW_PREFIX + row.key)`.
- `_plugin_row_label` → `_row_label(row_index, row)` (jump-hint prefix logic unchanged).
- `_row_text(row)` replaces `_row_text(entry)` **and** `_agent_cli_row(status)`:

  ```
  mark cell   "[✓] " when marked else "    "
              plugin  -> row.key.removeprefix("plugin:") in self._marked_install
              agent-cli -> row.key.removeprefix("cli:") in self._marked_agent_clis
              core    -> never marked
  glyph       _INSTALLED_GLYPH/green when row.installed else _AVAILABLE_GLYPH/dim
  label       row.label, style f"bold {row.accent}"
  version     "  " + row.version_label (dim) when non-empty
  source      agent-cli only: "  [" + row.source.replace("_", " ") + "]" (bold dim)
  update      "  " + _UPDATE_GLYPH (bold cyan) when row.update_available
  verbose     plugin only and self._verbose: "  ★{stars}" and "  {updated_at}" (dim)
              read from row.payload
  ```

  Delete `_agent_cli_row`, `_install_method_label`, and `_plugin_display_label` (the row
  model already carries `label`).

- `_rebuild_options()` — the single rebuild path and the **only** place `highlighted` is
  assigned programmatically:

  ```python
  option_list = self._option_list()
  if option_list is None: return
  preferred = self._restore_key or self._session_state.rows.identity
  self._restore_key = None
  rows = self._flat_rows()
  self._selection_guard.clear()
  option_list.clear_options()
  options = self._create_options()
  option_list.add_options(options)
  self._rebuild_row_identity_maps(options, rows)
  selected_key: str | None = None
  if rows:
      if preferred is None and self._session_state.rows.row is None:
          row_index = next((i for i, r in enumerate(rows) if r.update_available), 0)
      else:
          row_index = restore_selection_by_identity(
              rows, prior_identity=preferred,
              prior_visual_row=self._session_state.rows.row,
              identity_fn=lambda row: row.key)
      selected_key = rows[row_index].key
      option_index = self._row_option_index.get(selected_key)
      ...  # guard.prepare(selected_key, row_index) immediately before the assignment
      option_list.highlighted = option_index
  else:
      option_list.highlighted = None
  self._record_bookmark(selected_key)
  self._update_static("#updates-hints", self._hints())
  ```

  `prepare()` is called immediately before assigning `highlighted`, and the guard is
  cleared **synchronously** inside `should_ignore` — never through `call_later`,
  `call_after_refresh`, or a timer (perf rule 12).

- `_rebuild_plugin_identity_maps` → `_rebuild_row_identity_maps`, producing
  `_row_option_index: dict[str, int]` (key → option index, `_ROW_PREFIX` stripped) and
  `_row_logical_row: dict[str, int]` (key → flat item index). Delete
  `_plugin_option_index`, `_plugin_logical_row`, `_plugin_entry_by_name`,
  `_option_index_for_plugin`, `_logical_row_for_plugin`, `_entry_by_name`,
  `_highlight_named`, and `_skip_to_first_item` (the last two are already dead).
- `_record_plugin_bookmark` → `_record_bookmark(key)` writing
  `self._session_state.rows`. Keep today's "only clear the bookmark when the data is
  authoritative and unfiltered" behavior:
  `if key is None: if self._rows and not self._filter_text.strip(): record(None, None)`.

### Selection, highlight, detail

- `_highlighted_row() -> UpdateRow | None` moves here from `plugins_browser_workers.py`
  (delete the workers copy) and resolves the one list's highlighted option id: strip
  `_ROW_PREFIX`, reject `_HEADER_PREFIX`, return `self._rows_by_key.get(key)`. Delete
  `_highlighted_plugin_row` and re-point its callers in `plugins_browser_status.py` at
  `_highlighted_row`.
- `_current_entry()` → `row.payload if row and row.kind == "plugin" else None` (keeps
  `plugins_browser_install/uninstall/update/incoming/latest` unchanged).
  `_highlighted_name()` unchanged in meaning.
- `on_option_list_option_highlighted` keeps one branch for `#updates-list`: resolve the
  id to a key, compare against the live highlight, consult
  `self._selection_guard.should_ignore(...)`, then `self._record_bookmark(key)`,
  `self._update_static("#updates-hints", self._hints())` (immediate — cheap), and
  `self._schedule_detail()` (debounced — perf rule 7). Delete
  `_on_agent_cli_highlighted`.
- `_render_detail_now(*, force=False)` dedups on `self._detail_key` (was `_detail_name`
  / `_agent_cli_detail_name`, both deleted) and dispatches on `row.kind`:

  | kind        | `#updates-detail`                                                                         | `#updates-history`                                          |
  | ----------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
  | `None`      | `_DETAIL_PLACEHOLDER`                                                                     | `display = False`                                           |
  | `core`      | `_core_detail_panel(row.payload)`                                                         | `display = False`                                           |
  | `plugin`    | `_ensure_plugin_incoming_commits` + `_ensure_plugin_latest` + `_detail_renderable(entry)` | `display = False`                                           |
  | `agent-cli` | `_agent_cli_detail_panel(status)`                                                         | `display = True` + `_render_agent_cli_history(force=force)` |

  Delete `_render_agent_cli_detail`; `action_toggle_history_scope` calls
  `_render_agent_cli_history(force=True)` directly. `_render_agent_cli_history`
  retargets `#updates-history` and reads the highlighted row's payload instead of
  `_current_agent_cli()`'s old list.

- `_core_detail_panel(package)` replaces `_core_versions_panel` /
  `_core_versions_table`: a per-package `Panel` titled `package.name`, border `#AF87FF`,
  built from a `Table.grid` of `_core_glyph` / `_core_version_cell` / `_core_note_cell`
  for **that** package, then `_mode_line()`, then that package's incoming-commits
  section (`_core_incoming_sections` collapses to
  `_core_incoming_section(package) -> RenderableType | None`), then the
  `NotUvToolInstall` warning line when applicable. The `u` call-to-action is dropped
  here — it already lives in the hint line.

### Marks (still two sets in this phase)

- `_refresh_install_mark_row(name)` and `_refresh_agent_cli_mark_row(name)` both resolve
  `self._row_option_index[f"plugin:{name}"]` / `[f"cli:{name}"]` and
  `replace_option_prompt_at_index(index, self._row_text(row))`.
- `_advance_install_mark_selection` / `_advance_agent_cli_mark_selection` walk the
  merged list and skip rows of the other kind (match on `install` / `mark_update` in
  `row.capabilities`).
- `_clear_install_marks` / `_clear_agent_cli_marks` repaint `#updates-hints`.
- `_prune_stale_marked_install` intersects with **`self._rows`**, not `self._grouped`:
  `{row.key.removeprefix("plugin:") for row in self._rows if "install" in row.capabilities}`.
  This matters: the default `installed` scope hides exactly the not-installed rows that
  install marks live on, so pruning against the visible groups would wipe every install
  mark on reload.

### `_render_all`

```python
self.reset_jump_state()
self._rebuild_groups()
self._prune_stale_marked_install()
self._prune_agent_cli_marks()
self._refresh_scope_strip()
self._sync_header()
self._update_static("#updates-hints", self._hints())
self._rebuild_options()
self._sync_state_visibility()
self._render_detail_now(force=True)
```

Delete `_render_agent_clis` and `_repaint_agent_cli_options`.

---

## 8. `src/sase/ace/tui/modals/plugins_browser_status.py`

- `_sync_current_banner` → `_sync_header()`: always-visible `#updates-header` showing
  `self._all_current_banner()` when `self._all_up_to_date()` else
  `self._summary_text()`. No `display` toggling. (The `header` phase replaces the
  else-branch with the digest.)
- `_sync_state_visibility` targets `#updates-status` / `#updates-list` and decouples the
  two widgets so a failed source stays visible above a populated list:

  ```python
  message = self._status_message()
  status.update(message)
  status.display = bool(message)
  option_list.display = self._has_item_rows()
  ```

- `_status_message()` becomes row/scope aware:

  ```
  loading                          -> "Loading updates…"
  self._error is not None          -> f"Could not load plugins:\n{self._error}"
  agent_cli_error and no rows      -> f"Could not load agent CLIs:\n{...}"
  not self._rows                   -> "No updates found."
  no visible rows and a filter     -> "Nothing matches the current filter."
  no visible rows, scope outdated  -> "Nothing needs an update."
  no visible rows, scope installed -> "Nothing is installed."
  no visible rows, scope all       -> "No updates found."
  otherwise                        -> ""
  ```

- `_hints()` — one line for the whole tab; both squeeze workarounds
  (`plugins_browser_agent_clis.py:256-258`, `plugins_browser_status.py:246-248`) and
  their comments are deleted, so `u core+plugins` and `A update CLIs` both get their
  verbs back:

  ```
  jump mode -> "JUMP ' {back|first} · esc cancel"    (unchanged)
  i install [(n)]            when marks or the row can install
  I/space mark               when the row can install
  space mark                 when the row can mark_update
  u update core + plugins    when _can_update_sase()
  A update CLIs
  a sync agents
  m switch                   when _can_switch_mode()
  U upd ↑                    when the row can update
  x rm                       when the row can uninstall
  H history                  when the row is an agent CLI
  r reload · ctrl+d/u scroll · o{ off|(on) } · v{ verb|(on) } · / filter · ' jump
  _SCOPE_NAV_HINT · Tab/Shift+Tab tab
  "{n} marked" + "esc clear"  when either mark set is non-empty, else "esc"
  ```

  The marked count is `len(self._marked_install) + len(self._marked_agent_clis)`.

- `_can_install_highlighted` / `_can_mark_highlighted` / `_can_update_highlighted` /
  `_can_uninstall_highlighted` read `self._highlighted_row()`. `_can_update_sase` /
  `_can_switch_mode` / `_all_up_to_date` / `_sase_up_to_date` / `_all_current_banner` /
  `_summary_text` / `_summary_line` / `_summary_hint` / `_plural` are unchanged.

---

## 9. `src/sase/ace/tui/modals/plugins_browser_agent_clis.py` and `..._actions.py`

Delete (all superseded by the merged list): `_render_agent_clis`,
`_record_agent_cli_bookmark`, `_repaint_agent_cli_options`, `_agent_cli_row_label`,
`_agent_cli_row`, `_install_method_label`, `_agent_cli_summary`,
`_agent_cli_status_message`, `_sync_agent_cli_visibility`, `_agent_cli_hints`,
`_agent_cli_option_list`, `_agent_cli_by_name`, `_highlight_agent_cli`,
`_on_agent_cli_highlighted`, `_render_agent_cli_detail`, and the module-local
`_ITEM_PREFIX` / `_DETAIL_PLACEHOLDER`.

Keep: `_agent_cli_color`, `_agent_cli_detail_panel`, `_render_agent_cli_history`,
`action_toggle_history_scope`, `_agent_cli_update_entry`, `_make_agent_cli_update_plan`,
and the whole plan/confirm/execute path.

`_current_agent_cli()` becomes
`row.payload if (row := self._highlighted_row()) and row.kind == "agent-cli" else None`.

In `plugins_browser_agent_clis_actions.py`:

- `action_toggle_mark()` dispatches on the highlighted row's kind: `plugin` →
  `self.action_toggle_install_mark()`; `agent-cli` → `self._toggle_agent_cli_mark()`;
  `core`/`None` → the existing "Select an installable plugin or an updatable agent CLI
  to mark." warning toast.
- `action_clear_marks_or_close()` loses its sub-tab conjuncts and keeps two branches in
  this phase (the single global clear is the `marks` phase's job): agent-CLI marks
  first, then install marks, then close.
- `action_update_agent_clis()` drops `self._active_subtab == "agent-clis"` from its
  `names` expression — with no sub-tab there is nothing left to condition on, so marked
  CLIs are consumed from wherever they were set.
- `_refresh_agent_cli_mark_row` / `_advance_agent_cli_mark_selection` use the merged
  list (see §7).
- `_can_mark_agent_cli(status)` and `_prune_agent_cli_marks` are unchanged (they already
  read `_rows_by_key`).
- Delete `_active_subtab` from the `TYPE_CHECKING` block.

---

## 10. `src/sase/ace/tui/modals/plugins_browser_jump.py`

- Rewrite the module docstring: the pane hosts **one** list over every domain; the
  logical row list is that list's item rows (disabled section headers are never jump
  targets). The "Core hosts no list, so `'` is a silent no-op" behavior it documents is
  exactly what this phase removes.
- `_jump_item_option_indices`, `_jump_current_index`, `_jump_select_index` use
  `_option_list()`.
- `_jump_repaint()` loses its branch: one `self._rebuild_options()` call, which repaints
  its own hints.
- Drop the `_active_subtab` / `_agent_cli_hints` / `_repaint_agent_cli_options` entries
  from the `TYPE_CHECKING` block.

---

## 11. `src/sase/ace/tui/modals/plugins_browser_workers.py`

- `_start_load` repaints the merged widgets only: `#updates-header` (via
  `_sync_header()`), `#updates-hints`, plus the existing `_sync_state_visibility()`.
  Delete the `#plugins-summary`, `#updates-core-hints`, `#agent-clis-summary`,
  `#agent-clis-hints`, and `#sase-core-versions` updates.
- `self._restore_name` → `self._restore_key`, seeded from
  `(row := self._highlighted_row()) and row.key` or `self._session_state.rows.identity`
  (perf rule 4: the live highlight is re-read here, and `_rebuild_options` re-reads the
  live scope when the result lands).
- Delete `self._refresh_plugin_haystacks()` and the `_highlighted_row` definition (moved
  to the rendering mixin).
- Keep the row-building `dataclasses.replace(result, rows=...)` task body exactly as
  phase `rows` left it.

---

## 12. `src/sase/ace/tui/modals/plugins_browser_pane.py`

- Module docstring: one merged inventory, not three sub-tabs.
- Imports: drop `UpdatesSubTab`, `_SUBTAB_ORDER`, `_SUBTABS`, `_SUBTAB_WIDGET_IDS`, and
  `PanelTab`/`PanelTabStrip` if `just check` reports them unused.
- `BINDINGS`: `("right_square_bracket", "cycle_scope", "Next Scope")` and
  `("left_square_bracket", "cycle_scope_reverse", "Previous Scope")`. Every other
  binding keeps its key, action name, and target.
- `__init__`: add `self._scope: UpdateScope = self._session_state.scope`,
  `self._selection_guard = ProgrammaticSelectionGuard()`,
  `self._restore_key: str | None = self._session_state.rows.identity`,
  `self._detail_key: str | None = None`, `self._row_option_index: dict[str, int] = {}`,
  `self._row_logical_row: dict[str, int] = {}`, and retype
  `self._grouped: list[tuple[str, str, list[UpdateRow]]]`. Remove `_active_subtab`,
  `_plugin_selection_guard`, `_agent_cli_selection_guard`, `_restore_name`,
  `_detail_name`, `_agent_cli_detail_name`, `_plugin_haystacks`, `_plugin_option_index`,
  `_plugin_logical_row`, and `_plugin_entry_by_name`. Keep `_marked_install` and
  `_marked_agent_clis` (the `marks` phase merges them).

---

## 13. `src/sase/ace/tui/styles.tcss`

Replace the whole `/* Updates tab: Core / Plugins / Agent CLIs */` block (currently
lines ~8536-8674) with one master/detail pair:

```
/* Updates tab: one merged inventory */
PluginsBrowserPane #updates-scopes  { height: 1; text-align: center; text-wrap: nowrap;
                                      text-overflow: clip; margin-bottom: 1; }
PluginsBrowserPane #updates-header  { width: 100%; height: auto; text-style: bold; }
PluginsBrowserPane #updates-filter-input { width: 100%; margin-bottom: 1; }
PluginsBrowserPane #updates-panels  { width: 100%; height: 1fr; }
PluginsBrowserPane #updates-list-panel { width: 58; height: 100%; }
PluginsBrowserPane #updates-status  { width: 100%; height: auto; color: $text-muted; }
PluginsBrowserPane OptionList       { height: 1fr; border: solid $secondary; }
PluginsBrowserPane #updates-detail-panel  { width: 1fr; height: 100%; margin-left: 1; }
PluginsBrowserPane #updates-detail-scroll { height: 1fr; }
PluginsBrowserPane #updates-detail  { width: 100%; height: auto; color: $text; }
PluginsBrowserPane #updates-history { width: 100%; height: auto; color: $text; margin-top: 1; }
PluginsBrowserPane #updates-hints   { height: auto; color: $text-muted; text-align: center; }
```

`58` is the wider of today's two list panels, so `v1.2.3 → v1.2.4  [npm]` rows still
fit. `#updates-hints` is `height: auto` because the `marks` phase adds a second line.

---

## 14. Tests

### Shared helpers (these two edits unblock most of the suite)

- `tests/ace/tui/_plugins_browser_pane_helpers.py`:
  `_open_plugins_pane(page, *, session_state=None, scope="all")` replaces
  `pane._switch_to_subtab("plugins")` with `pane._set_scope(scope)`. `"all"` is the
  helper default so existing plugin-centric tests still see not-installed catalog rows.
  `_option_labels` reads `#updates-list`.
- `tests/ace/tui/visual/_ace_config_center_modal_helpers.py`: `_open_plugins_modal` does
  the same.

### Existing Updates-pane tests

Rewrite every `_switch_to_subtab` / `action_cycle_subtab` /
`state.updates.active_subtab` / `pane._active_subtab` call site as a scope selection, or
drop it where the merged list already shows the row:

- `test_plugins_browser_pane_agent_clis.py:58-72` (sub-tab cycling) becomes a
  **scope**-cycling test over `SCOPE_ORDER`; `:85-97` seeds `state.updates.scope`;
  `:184` (`_agent_cli_summary`) is deleted or repointed at the merged row labels;
  `:207-231` (bracket cycling through sub-tabs) becomes bracket cycling through scopes;
  `:376-379` records `pane._scope`.
- `test_plugins_browser_pane_jump.py`: the three sub-tab cases collapse — add the epic's
  "`'` jumps across all three kinds in one hint space" test and drop the "Core is a
  silent no-op" assertion at `:206-216`.
- `test_plugins_browser_pane_loading.py:66-77` seeds/asserts `state.updates.scope` and
  `state.updates.rows`; `:300-304` becomes a scope switch; `:323` expects
  `"Nothing matches the current filter."`; `:336` (empty catalog) asserts the merged
  list still shows the core and agent-CLI rows and no `plugin:` row instead of
  `"No SASE plugins found."`; `:348` (error state) keeps asserting
  `"gh not found" in pane._status_message()` and that `#updates-status` is displayed
  **while `#updates-list` stays visible**.
- `test_plugins_browser_pane_agent_clis_history.py:336-346` becomes scope selections
  plus a highlight move onto / off an agent-CLI row (the history panel is now gated by
  the highlighted row's kind, not a sub-tab).
- `test_plugins_browser_pane_all_current.py`: `.display is True/False` on
  `#updates-current-banner` becomes a `"You're all up to date"` substring check over the
  rendered `#updates-header`, in all three scopes.
- `test_admin_center_selection_resume.py`: the `plugins` and `agent-clis` cases collapse
  into one `_ResumeCase("updates", "6")` using `#updates-list`, `pane._detail_key`, and
  the `updates-row__` prefix.
- Every remaining `#plugins-*` / `#agent-clis-*` selector in `tests/ace/tui/` moves to
  its merged id.

### New tests

Add `tests/ace/tui/test_plugins_browser_pane_scopes.py` (widget-level) plus pure
`select_rows` cases in `tests/ace/tui/test_plugins_browser_rows.py`:

1. Every core package, plugin, and registered CLI appears exactly once per scope, under
   the right section header, sections in the fixed order.
2. Scope membership: a manual-only CLI with a newer version lands in `Outdated`; a row
   whose only defect is a probe error lands in `Outdated`; a not-installed plugin is
   absent from `Installed` and present in `All`.
3. Outdated-first ordering inside a section, then case-insensitive label order.
4. One filter needle matches a core package by its distribution name, a plugin by its
   topic, and a CLI by its binary — in one query.
5. `'` allocates hints across all three kinds in one space.
6. The cursor survives a refresh, a filter, and a scope switch by identity, and an
   empty-identity open lands on the first outdated row.
7. `check_action` offers `x` / `U` / `H` / `I` on the right rows and refuses them on the
   others (including every capability withdrawn under `NotUvToolInstall`).
8. An install mark set in `All` survives a switch to `Installed` and back (the
   `_prune_stale_marked_install`-against-`_rows` regression).
9. The scope strip's labels carry live counts and the strip's `TabClicked` selects a
   scope directly.

### Bench

`tests/ace/tui/bench_plugins_catalog_scale.py`:

- `_open_plugins_pane(page)` already selects the `All` scope via the helper default, so
  the n=2000 guard still measures the full catalog; assert `pane._scope == "all"` after
  open so a helper-default change cannot silently shrink the bench.
- `#plugins-filter-input` → `#updates-filter-input`.
- Before `_measure_install_mark`, move the highlight onto the first row with `"install"`
  in `row.capabilities` (the merged list now opens on a core row), so the mark toggle
  measures the same work it does today.
- `_matched_row_count` and the `scale_filter_match_count` assertion are unchanged:
  neither core row (`sase`, `sase-core-rs`) nor the default empty agent-CLI set matches
  `FILTER_KEYSTROKE = "q"`. Verify this empirically at n=10 before assuming it.
- `TARGET_P95_MS = 16.0` at every size, including n=2000, must still hold. Do **not**
  rewrite `tests/perf/baselines/plugin_catalog_scale_baseline.json` — refreshing the
  recorded baseline is the `docs` phase's job.
  `tests/perf/bench_plugin_catalog_scale.py` is a non-TUI fetch/enrichment bench and
  needs no change.

### PNG snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugins.py`: rewrite the
seven `_switch_to_subtab` calls as scope selections plus (where the test needs a
specific row) an explicit highlight move by row key, then rebaseline the goldens this
phase invalidates with `--sase-update-visual-snapshots`. Inspect
`.pytest_cache/sase-visual/` on any unexpected diff. Waits on `pane._detail_name` /
`pane._agent_cli_detail_name` become waits on `pane._detail_key` (`"plugin:github"`,
`"cli:claude"`). Do **not** add the new scenarios the epic lists — those belong to the
`docs` phase.

---

## 15. Verification

```bash
just install     # ephemeral workspace clones may have drifted deps
just check       # hand to /sase_monitor if it runs long
just test-visual
pytest -s -m slow tests/ace/tui/bench_plugins_catalog_scale.py
```

Do not run `just check-full` inline; it is monitor-only and belongs to the epic's land
agent.

**Acceptance for this phase**

- `just check` and `just test-visual` green.
- The bench's `filter_keystroke` and `j_press` p95 stay under 16 ms at every size,
  including n=2000 in the `All` scope.
- `rg -n "_active_subtab|_switch_to_subtab|cycle_subtab|UpdatesSubTab" src/` returns
  nothing under the Updates pane, and
  `rg -n "plugins-list|agent-clis-|plugins-summary|plugins-detail|plugins-hints|updates-core-hints|updates-current-banner|sase-core-versions|updates-subtab" src/`
  returns nothing.
- Every binding listed in `PluginsBrowserPane.BINDINGS` still resolves to a live action,
  and `j` `k` `g` `G` `Ctrl+D` `Ctrl+U` `'` work on the merged list, including for SASE
  core rows where `'` is a documented no-op today.

## Out of scope for this phase

- The digest header, `PluginsLoadResult.core_error`, and the all-current truthfulness
  invariant (`header`, `sase-w0.3`).
- Merging the two mark sets, the global `Esc` clear, and the marked-work aggregate hint
  line (`marks`, `sase-w0.4`).
- Documentation (`docs/ace.md`, `docs/configuration.md`, `docs/plugins.md`,
  `docs/agent_providers.md`), the new PNG scenarios, and the recorded scale-bench
  baseline refresh (`docs`, `sase-w0.5`).
- Anything in the epic's own "Out of scope" list (the verb collapse, rerouting `u`, an
  adaptive narrow layout, the `query_profile` filter dialect, the badge-click
  inversion).
