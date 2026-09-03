---
tier: tale
title: One list, domain sections, and the scope filter
goal:
  "The Admin Center Updates tab becomes one master/detail surface: one scope strip on ]
  / [, one filter, one OptionList with domain section headers, one selection guard, one
  bookmark, one hint line, and a kind-dispatched detail panel, with no `_active_subtab`
  reference left anywhere in `src/`."
size: medium
proposed_by: bbugyi200.apollo.sase-w0.2
bead: sase-w0.2
create_time: 2026-09-03 13:51:04
status: wip
---

- **PARENT:** [202609/unified_updates_tab_1.md](unified_updates_tab_1.md)
- **BEAD:**
  [sase-w0.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w0/sase-w0.2.md)

# Plan: One list, domain sections, and the scope filter (epic `sase-w0`, phase `list`)

Phase bead: `sase-w0.2`. Parent epic: `sase-w0` (`plan:202609/unified_updates_tab_1.md`,
phase id `list`). Read the epic plan's `Phase list` section and its
`Cross-cutting constraints` section before starting — this plan is the
implementation-level expansion of them and deliberately does not restate their
rationale.

Phase `rows` (`sase-w0.1`, commit `f67169ea7`) already landed
`src/sase/ace/tui/modals/plugins_browser_rows.py`: the pure `UpdateRow` model,
capability derivation, the moved version-label helpers, `build_update_rows`, and the
`PluginsLoadResult.rows` plumbing. Every visible pixel is still today's three sub-tabs.
This phase spends that groundwork: it deletes the sub-tabs and lands the whole visible
merge.

## Goal

The Admin Center **Updates** tab becomes **one** master/detail surface: one scope strip
on `]` / `[`, one filter, one `OptionList` with domain section headers, one selection
guard, one bookmark, one hint line, and a kind-dispatched detail panel. No
`_active_subtab` reference survives anywhere in `src/`.

## Non-goals (owned by later phases — do not do these)

- **`sase-w0.3` (`header`)** owns the digest header, the promoted per-source failure
  lines, the offline/cache-age/agents-sync chips, `PluginsLoadResult.core_error`, and
  the `_sase_up_to_date` truthfulness fix. This phase only creates the single
  `#updates-header` widget and keeps today's two states in it (all-current banner or
  `_summary_text()`).
- **`sase-w0.4` (`marks`)** owns collapsing `_marked_install` + `_marked_agent_clis`
  into one `_marked` set, the global `Esc` clear, `A` consuming CLI marks from anywhere,
  and the marked-work aggregate line. This phase keeps **two disjoint mark sets** and
  keeps `A` / `Esc` scoped, substituting _the highlighted row's kind_ for the deleted
  `_active_subtab` conjunct. Do not "fix" problem 4 here; `sase-w0.4`'s regression test
  needs it still broken.
- **`sase-w0.5` (`docs`)** owns `docs/ace.md`, `docs/configuration.md`,
  `docs/plugins.md`, `docs/agent_providers.md`, the new snapshot scenarios, and the
  recorded scale-bench baseline refresh. **Do not edit any file under `docs/` in this
  phase**, and do not run the bench's `--write-baseline` path.

---

## Step 1 — Scopes, sections, and selection in `plugins_browser_rows.py`

Add to `src/sase/ace/tui/modals/plugins_browser_rows.py` (keep everything already
there):

```python
UpdateScope = Literal["outdated", "installed", "all"]

SCOPE_ORDER: tuple[UpdateScope, ...] = ("outdated", "installed", "all")
SCOPE_LABELS: dict[UpdateScope, str] = {
    "outdated": "Outdated", "installed": "Installed", "all": "All",
}
```

**Section metadata**, in display order, as one module-level tuple mapping
`UpdateRowSection` → `(header title, header style)`:

| section             | title                 | style                     |
| ------------------- | --------------------- | ------------------------- |
| `sase`              | `SASE`                | `bold #AF87FF`            |
| `plugins-builtin`   | `Plugins · Built-in`  | `bold dim`                |
| `plugins-community` | `Plugins · Community` | `bold {_COMMUNITY_STYLE}` |
| `agent-clis`        | `Agent CLIs`          | `bold #87D7FF`            |

`_COMMUNITY_STYLE` comes from `sase.plugins.render_common` (already imported by the
rendering mixin; import it in `plugins_browser_rows.py` too). The built-in / community
styles are exactly today's `_rebuild_groups` styles.

**Public functions to add:**

- `row_in_scope(row: UpdateRow, scope: UpdateScope) -> bool`
  - `outdated` → `row.update_available or row.error is not None`
  - `installed` → `row.installed`
  - `all` → `True`
- `scope_counts(rows) -> dict[UpdateScope, int]` — **one** pass over the tuple
  incrementing three counters. Do not call `row_in_scope` three times per row in the
  caller.
- `select_rows(rows, *, scope, needle) -> list[tuple[str, str, list[UpdateRow]]]` —
  returns exactly the `(header title, header style, rows)` shape `_grouped` already
  uses, sections in the order above, **empty sections omitted**. `needle` is already
  casefolded and stripped by the caller; when non-empty a row matches iff
  `needle in row.haystack`.
- `plugin_row_key(name) -> str` (`f"plugin:{name}"`), `agent_cli_row_key(name) -> str`
  (`f"cli:{name}"`), `row_name(row) -> str` (`row.key.split(":", 1)[1]`). Build
  `_build_core_row` / `build_plugin_row` / `_build_agent_cli_row` keys through these so
  the key format lives in one place.

**Sort once per load, not per rebuild.** `build_update_rows` must return rows already in
display order: grouped by section in the order above and, within a section, sorted by
`(not row.update_available, row.label.casefold())`. `select_rows` then only filters and
buckets — an `O(n)` pass, which is what keeps the n=2000 filter keystroke inside the 16
ms budget. Sorting on the load worker also satisfies perf rules 1 and 2.

`build_plugin_row` is called again by `_apply_plugin_latest` to patch one row in place;
that path must keep the row's existing position (see Step 8), so a lazy latest
completion can never reorder or reclassify a row under the cursor (perf rule 6 /
`sase-w0` cross-cutting constraint 5).

Extend `tests/ace/tui/test_plugins_browser_rows.py` (pure, widget-free) with: scope
membership per kind including a manual-only CLI (`update_available=True`, `manual`, no
`mark_update`) landing in `outdated`; a row whose only defect is `error is not None`
landing in `outdated`; `scope_counts` agreeing with three `row_in_scope` sweeps;
`select_rows` omitting empty sections, ordering sections, and placing outdated rows
first inside a section; one needle matching a core package, a plugin, and a CLI by their
own haystack fields; and `plugin_row_key` / `row_name` round-tripping.

## Step 2 — Session state (`config_center_session.py`)

```python
@dataclass
class UpdatesSessionState:
    """Session-only cursor state for the merged Updates pane."""

    scope: UpdateScope = "installed"
    rows: SelectionBookmark = field(default_factory=SelectionBookmark)
    agent_cli_history_all: bool = False
```

Import `UpdateScope` from `.plugins_browser_rows` (no cycle: that module imports nothing
from `config_center_session`). Delete the `UpdatesSubTab` alias and its `__all__` entry.
Session state is process-local, so there is no migration.

The two bookmarks (`plugins`, `agent_clis`) become the one `rows` bookmark. Its
identities are **row keys** (`plugin:github`, `cli:claude`, `core:sase`), never bare
names.

## Step 3 — Constants (`plugins_browser_constants.py`)

- `_DETAIL_PLACEHOLDER = "Select a row to view its details."`
- Delete `_SUBTAB_NAV_HINT`; add `_SCOPE_NAV_HINT = "[ ] scope"`.
- Delete `_ITEM_PREFIX`; add `_ROW_PREFIX = "updates-row__"`.
- Delete `_BUILTIN_GROUP` / `_COMMUNITY_GROUP` (the section table in
  `plugins_browser_rows.py` replaces them).
- Keep `_HEADER_PREFIX` unchanged — `_is_item`, the jump mixin, and `action_next_option`
  / `action_prev_option` keep working untouched.

Also delete the module-local `_ITEM_PREFIX = "agent-cli__"` in
`plugins_browser_agent_clis.py` and `plugins_browser_agent_clis_actions.py`.

Item option ids become `_ROW_PREFIX + row.key`; header option ids stay
`_HEADER_PREFIX + <section title>`.

## Step 4 — Layout (`plugins_browser_layout.py`)

Delete `_SUBTAB_ORDER`, `_SUBTAB_WIDGET_IDS`, `_SUBTABS`, `_switch_to_subtab`,
`_cycle_subtab`, `action_cycle_subtab`, `action_cycle_subtab_reverse`, `_core_hints`,
`_on_subtab_clicked`, and the `ContentSwitcher` import.

`compose` becomes one surface:

```
Static                        id="updates-header"           (banner or summary)
PanelTabStrip                 id="updates-scopes"           (3 scope tabs, uppercase_active=True)
_PluginsFilterInput           id="updates-filter-input"     placeholder "/ filter components, plugins, agent CLIs…"
Horizontal                    id="updates-panels"
  Vertical                    id="updates-list-panel"
    Static                    id="updates-status"
    _PluginList               id="updates-list"
  Vertical                    id="updates-detail-panel"
    VerticalScroll            id="updates-detail-scroll"
      Static                  id="updates-detail"
      Static                  id="updates-history"
Static                        id="updates-hints"
```

`#updates-history` is created with `display = False`; only an agent-CLI row turns it on.

Add `_scope_tabs() -> tuple[PanelTab, ...]` building one `PanelTab` per `SCOPE_ORDER`
entry with id = the scope, label = `f"{SCOPE_LABELS[scope]} {count}"` from
`scope_counts(self._rows)`, accent `#AF87FF`; and `_sync_scope_strip()` calling
`set_tabs(self._scope_tabs(), active_tab=self._scope)` inside a `try/except` (the strip
may not be mounted yet).

Scope navigation:

```python
@on(PanelTabStrip.TabClicked)
def _on_scope_clicked(self, event) -> None: ...   # event.stop(); dispatch to _switch_to_scope

def _switch_to_scope(self, scope: UpdateScope) -> None:
    if scope == self._scope:
        return
    self.reset_jump_state()          # repaint=False: the rebuild below repaints
    self._scope = scope
    self._session_state.scope = scope
    if self._detail_debouncer is not None:
        self._detail_debouncer.cancel()
    self._rebuild_groups()
    self._rebuild_options()
    self._sync_state_visibility()
    self._sync_scope_strip()
    self._render_detail_now(force=True)
    self.focus_default()

def _cycle_scope(self, step: int) -> None: ...
def action_cycle_scope(self) -> None: ...
def action_cycle_scope_reverse(self) -> None: ...
```

A scope switch is a deliberate full rebuild through the **one** `_rebuild_options` path
(epic constraint: perf rule 6 permits it; the default scope is ~13 rows).

`check_action` becomes:

```python
if action == "update_sase":            return self._can_update_sase()
if action == "update_agent_clis":      return not self._loading and self._agent_cli_plan_worker is None
if action == "sync_agents":            return callable(getattr(self.app, "action_sync_agents", None))

row = self._highlighted_row()
row_capability = {
    "install": "install", "toggle_install_mark": "install",
    "uninstall": "uninstall", "update": "update",
    "toggle_history_scope": "history",
}
if action in row_capability:
    return row is not None and row_capability[action] in row.capabilities
if action == "toggle_mark":
    # Core rows carry no mark verb at all; plugin / CLI rows keep today's
    # "select an installable plugin…" toast for the non-markable case.
    return row is not None and row.kind in ("plugin", "agent-cli")
if action in {"next_option", "prev_option", "jump_to_entry"}:
    return bool(self._grouped)
return super().check_action(action, parameters)
```

`focus_filter`, `toggle_verbose`, `switch_mode`, `refresh`, `toggle_offline`, and the
four scroll actions lose every gate and fall through to `super()` — they are pane-wide
now. Delete the `plugin_only` and `browse_only` sets.

`focus_default()` (in `plugins_browser_controls.py`) focuses `#updates-list` when it
exists, else the pane.

`on_mount` calls `_sync_header()` (renamed from `_sync_current_banner`) and
`_sync_scope_strip()`.

## Step 5 — Bindings, pane fields (`plugins_browser_pane.py`)

Bindings: `right_square_bracket` → `cycle_scope` ("Next scope"), `left_square_bracket` →
`cycle_scope_reverse` ("Previous scope"). **Every other binding keeps its key, its
action name, and its target.**

Field changes in `__init__`:

- `self._active_subtab` → `self._scope: UpdateScope = self._session_state.scope`
- `self._plugin_selection_guard` + `self._agent_cli_selection_guard` →
  `self._selection_guard = ProgrammaticSelectionGuard()`
- `self._restore_name` →
  `self._restore_key: str | None = self._session_state.rows.identity`
- `self._detail_name` + `self._agent_cli_detail_name` → `self._detail_key: str | None`
- `self._grouped: list[tuple[str, str, list[UpdateRow]]]`
- `self._plugin_option_index` / `_plugin_logical_row` / `_plugin_entry_by_name` →
  `self._row_option_index: dict[str, int]` and `self._row_logical_row: dict[str, int]`
  (both keyed by row key). `_plugin_entry_by_name` is deleted outright: `_rows_by_key`
  already holds every row and its payload.
- Delete `self._plugin_haystacks`.
- Keep `_marked_install`, `_marked_agent_clis`, `_rows`, `_rows_by_key`,
  `_agent_cli_history_key` as they are.

Drop the `_SUBTAB_ORDER` / `_SUBTABS` / `_SUBTAB_WIDGET_IDS` and `UpdatesSubTab`
imports; drop the now-unused `PanelTab` / `PanelTabStrip` re-import if nothing else
needs it. Update the module docstring (it still says "pane-local Core / Plugins / Agent
CLIs sub-tabs").

## Step 6 — Core detail, extracted (`plugins_browser_core_detail.py`, new)

`plugins_browser_rendering.py` is already 640 lines and `toobig` warns at 700 for
`src/`, so move the core-package rendering out into a new module with **public**
functions (real non-test consumer = the rendering mixin, so symvision is satisfied):

```python
def core_glyph(package) -> Text            # moved verbatim from _core_glyph
def core_version_cell(package) -> Text     # moved verbatim from _core_version_cell
def core_note_cell(package) -> Text        # moved verbatim from _core_note_cell (uses dev_state_label)
def install_mode_line(mode, dev_root, *, blocked) -> Text | None   # moved from _mode_line
def build_core_detail_panel(
    package, *, incoming, incoming_loading, mode_line, blocked, update_cta
) -> Panel
```

`build_core_detail_panel` renders **one** package: a one-row `Table.grid` of
`core_glyph / name / core_version_cell / core_note_cell`, then the mode line, then that
package's `build_incoming_commits_renderable` section when incoming commits exist or are
loading, then either the `NotUvToolInstall` warning line or the `u  run \`sase
update\``call-to-action — i.e. exactly today's`_core_versions_panel`body narrowed to one package. Title`"SASE
Core"`, border `#AF87FF`, unchanged.

Delete `_core_versions_panel`, `_core_versions_table`, `_core_glyph`,
`_core_version_cell`, `_core_note_cell`, `_mode_line`, and `_core_incoming_sections`
from the rendering mixin, and add a thin pane method:

```python
def _core_detail_panel(self, package: CorePackageVersion) -> Panel: ...
```

that gathers `self._core_incoming_commits.get(package.name)`, the loading flag
(`self._loading and incoming is None`, gated on `self._incoming_commits_enabled`),
`install_mode_line(...)`, `isinstance(self._uv_tool, NotUvToolInstall)`, and
`self._can_update_sase()`.

This is a strict improvement over today: both packages shared one cramped panel; each
now gets the full detail column.

## Step 7 — Rendering, selection, and detail (`plugins_browser_rendering.py`)

**`_render_all`:**

```python
self.reset_jump_state()
self._rebuild_groups()
self._prune_stale_marked_install()
self._prune_agent_cli_marks()
self._sync_header()
self._rebuild_options()
self._sync_state_visibility()
self._render_detail_now(force=True)
```

(`_render_agent_clis` is gone; `_sync_scope_strip` is called from `_rebuild_options`.)

**`_rebuild_groups`:**
`self._grouped = select_rows(self._rows, scope=self._scope, needle=self._filter_text.strip().casefold())`.
Delete `_matches`, `_refresh_plugin_haystacks`, and the static `_plugin_haystack`;
delete the `_refresh_plugin_haystacks()` call in `plugins_browser_workers.py`.

**`_rebuild_options` is the single rebuild path and the only place `highlighted` is
assigned programmatically.** This is the rule-12 trap named in the epic plan.

```python
def _rebuild_options(self) -> None:
    option_list = self._option_list()
    if option_list is None:
        return
    preferred = self._restore_key or self._session_state.rows.identity
    prior_row = self._session_state.rows.row
    self._restore_key = None
    rows = self._flat_rows()
    selected_key: str | None = None
    self._selection_guard.clear()
    option_list.clear_options()
    options = self._create_options()
    option_list.add_options(options)
    self._rebuild_identity_maps(options, rows)
    if rows:
        if preferred is None and prior_row is None:
            logical = _first_outdated_index(rows)      # else 0
        else:
            logical = restore_selection_by_identity(
                rows, prior_identity=preferred, prior_visual_row=prior_row,
                identity_fn=lambda row: row.key,
            )
        selected_key = rows[logical].key
        index = self._row_option_index.get(selected_key)
        if index is None:            # defensive: fall back to the first row
            logical, selected_key = 0, rows[0].key
            index = self._row_option_index.get(selected_key)
        if index is not None:
            self._selection_guard.prepare(selected_key, logical)
            option_list.highlighted = index
    else:
        option_list.highlighted = None
    self._record_bookmark(selected_key)
    self._update_static("#updates-hints", self._hints())
    self._sync_scope_strip()
```

`prepare()` is called **immediately before** the assignment. The guard is cleared
**synchronously** inside `should_ignore` — never via `call_later`, `call_after_refresh`,
or a timer.

Landing rule: with no prior identity **and** no prior visual row, land on the first row
with `update_available or error is not None`, else on the first row. With a prior
bookmark, `restore_selection_by_identity` decides (identity, then nearest visual row,
then 0) exactly as today.

**Helpers** (all replacing their `_plugin_*` predecessors):

- `_flat_rows() -> list[UpdateRow]`
- `_rebuild_identity_maps(options, rows)` → `_row_option_index` (option id minus
  `_ROW_PREFIX` → option index, headers skipped) and `_row_logical_row` (row key →
  logical index)
- `_logical_row_for_key(key)`, `_record_bookmark(key)`:
  ```python
  def _record_bookmark(self, key: str | None) -> None:
      if key is None:
          if self._updates_loaded_once and not self._filter_text.strip():
              self._session_state.rows.record(None, None)
          return
      self._session_state.rows.record(key, self._logical_row_for_key(key))
  ```
  (The agent-CLI bookmark's provisional `display()` path is dropped; `record()` is gated
  on `_updates_loaded_once` instead. `SelectionBookmark.display` keeps its other callers
  in `procs_pane_selection.py` and `project_inventory_pane_base.py`, so it does not
  become dead.)
- `_highlighted_row() -> UpdateRow | None` — resolve `#updates-list`'s highlighted
  option id to `self._rows_by_key`. **Move it here from `plugins_browser_workers.py`**
  and delete `_highlighted_plugin_row`, `_highlighted_name`, and `_highlight_named`.
- `_highlighted_key() -> str | None`
- `_current_entry()` → the highlighted row's payload when `row.kind == "plugin"`, else
  `None`. This one method is what keeps the install / update / uninstall / incoming /
  latest mixins working with no changes to their bodies.
- `_entry_by_name(name)` → `self._rows_by_key.get(plugin_row_key(name))`'s payload.

**`_create_options` / `_row_text`.** Headers stay disabled options carrying
`_HEADER_PREFIX` ids and read `── {title} ──` in the section's style. Item ids are
`_ROW_PREFIX + row.key`. `_plugin_row_label` becomes `_row_label(logical_row, row)`,
still applying `apply_jump_hint_prefix` over the **logical** index.

`_row_text(row)` renders one uniform row shape, dispatching only where the kinds
genuinely differ:

```
"[✓] " | "    "                         # marked (plugin/CLI only; core always blank)
"●" green / "○" dim                     # row.installed
" " + label                             # bold; agent-CLI rows use `bold {row.accent}`
"  " + row.version_label (dim)          # omitted when empty
"  [install method]" (bold dim)         # agent-CLI rows only
"  ↑" (bold cyan)                       # row.update_available
"  ★{stars}  {updated_at}" (dim)        # plugin rows only, and only when _verbose
```

The plugin and agent-CLI branches must reproduce today's `_row_text` / `_agent_cli_row`
byte-for-byte so the PNG diff is confined to the layout change. Core rows are new: they
use the same shape with `row.version_label` (which `_core_version_label` already
produces).

`_row_marked(row)` is the one place the two mark sets are consulted:
`row.kind == "plugin" and row_name(row) in self._marked_install` or
`row.kind == "agent-cli" and row_name(row) in self._marked_agent_clis`.

**Highlight handler** — one branch, no list dispatch:

```python
def on_option_list_option_highlighted(self, event) -> None:
    if event.option_list.id != "updates-list" or event.option is None or event.option.id is None:
        return
    identity = str(event.option.id).removeprefix(_ROW_PREFIX)
    current_identity = self._highlighted_key()
    current_row = self._logical_row_for_key(current_identity) if current_identity else None
    if (current_identity is None or current_row is None or identity != current_identity
            or self._selection_guard.should_ignore(
                identity, current_row,
                current_identity=current_identity, current_row=current_row)):
        return
    self._record_bookmark(current_identity)
    self._update_static("#updates-hints", self._hints())   # cheap, immediate
    self._schedule_detail()                                 # 150 ms debouncer
```

The highlight itself is never debounced; only the detail rebuild is (perf rule 7).

**Detail, dispatched not rewritten:**

```python
def _render_detail_now(self, *, force: bool = False) -> None:
    row = self._highlighted_row()
    key = row.key if row is not None else None
    if not force and key == self._detail_key:
        if row is not None and row.kind == "agent-cli":
            self._render_agent_cli_history()   # scope may have flipped under H
        return
    self._detail_key = key
    self._update_detail(row)

def _update_detail(self, row: UpdateRow | None) -> None:
    detail = self._detail_widget()            # "#updates-detail"
    history = ...                             # "#updates-history"
    history.display = row is not None and row.kind == "agent-cli"
    if row is None:
        detail.update(_DETAIL_PLACEHOLDER); return
    if row.kind == "plugin":
        entry = row.payload
        self._ensure_plugin_incoming_commits(entry)
        self._ensure_plugin_latest(entry)
        detail.update(self._detail_renderable(entry))      # unchanged
    elif row.kind == "agent-cli":
        detail.update(self._agent_cli_detail_panel(row.payload))   # unchanged
        self._render_agent_cli_history(force=True)
    else:
        detail.update(self._core_detail_panel(row.payload))
```

Reuse `build_detail_panel` / `build_community_warning_panel` / `_agent_cli_detail_panel`
/ `build_agent_cli_history_panel` **unchanged**. Narrow the `row.payload` union with
`isinstance` (mypy) rather than `cast`.

`_detail_scroll()` in `plugins_browser_controls.py` stops branching and always returns
`#updates-detail-scroll`. `_detail_widget()` returns `#updates-detail`.

**Marks in place:** `_refresh_install_mark_row(name)` and
`_refresh_agent_cli_mark_row(name)` both resolve `self._row_option_index.get(<row key>)`
and call `replace_option_prompt_at_index(index, self._row_text(row))`; both return early
when the row is filtered out of the current scope. `_advance_install_mark_selection` and
`_advance_agent_cli_mark_selection` both walk `#updates-list`'s item rows for the next
row carrying the same capability (`install` / `mark_update`).

**Pruning must read `self._rows`, not `self._grouped`** — `_grouped` is scope- and
filter-limited, so pruning against it would silently drop marks the user set in another
scope:

```python
def _prune_stale_marked_install(self) -> None:
    if not self._marked_install:
        return
    self._marked_install &= {
        row_name(row) for row in self._rows
        if row.kind == "plugin" and "install" in row.capabilities
    }
```

`_prune_agent_cli_marks` keeps its `_can_mark_agent_cli` sweep over
`self._agent_cli_statuses` (already row-backed) and moves into `_render_all`.

## Step 8 — Lazy latest completion (`plugins_browser_latest.py`)

`_apply_plugin_latest` keeps patching the catalog, `_rows`, and `_rows_by_key`, and must
additionally patch the row **inside `_grouped` at its existing position** (replacing
`updated if entry.name == name else entry` over the entry lists with the same
substitution over row lists, matched on `row.key`). It must **not** call
`_rebuild_groups` or `_rebuild_options`, must not re-sort, and must not re-derive scope
membership — one row is patched in place through `_refresh_install_mark_row`, which
already goes through `replace_option_prompt_at_index`. Drop the `_plugin_entry_by_name`
write.

## Step 9 — Status, hints, header (`plugins_browser_status.py`)

- `_sync_current_banner` → `_sync_header`: always-visible `#updates-header` Static that
  renders `self._all_current_banner()` when `self._all_up_to_date()`, else
  `self._summary_text()`. Add `_header_renderable() -> RenderableType` returning the
  same choice so tests can assert on it without reaching into the widget. **Do not**
  change `_all_up_to_date` / `_sase_up_to_date` — that is `sase-w0.3`.
- `_sync_state_visibility` targets `#updates-status` / `#updates-list`, and keeps
  today's rule `show_status = self._error is not None or not has_rows`. Add a short
  comment noting `sase-w0.3` promotes per-source failures into the header and un-hides
  the list.
- `_status_message`:
  ```python
  if self._loading:            return "Loading updates…"
  if self._error is not None:  return f"Could not load plugins:\n{self._error}"
  if not self._rows:           return "No updates were found."
  if self._filter_text.strip():return "No rows match the current filter."
  return {"outdated": "Nothing needs an update.",
          "installed": "Nothing is installed.",
          "all": "No updates were found."}[self._scope]
  ```
- One `_hints()` replaces `_hints` / `_core_hints` / `_agent_cli_hints`. Jump mode keeps
  its `JUMP ' {action} · esc cancel` line. Otherwise, in order: `i install [(n)]` (or
  `i install` when installable), `I/space mark`, `u update core + plugins`,
  `A update CLIs`, `a sync agents`, `m switch`, `U upd ↑`, `x rm`, `r reload`,
  `ctrl+d/u scroll`, `o off|(on)`, `v verb|(on)`, `/ filter`, `' jump`,
  `_SCOPE_NAV_HINT`, `Tab/Shift+Tab tab`, and `{n} marked` + `esc clear` or `esc`.
  Delete both squeeze comments (`plugins_browser_agent_clis.py:256`,
  `plugins_browser_status.py:246`): three lines became one, so `u core+plugins` returns
  to this line and `A` regains its verb.
- `_can_install_highlighted` / `_can_mark_highlighted` / `_can_update_highlighted` /
  `_can_uninstall_highlighted` read `self._highlighted_row()`. Delete
  `_can_install_entry` and its `TYPE_CHECKING` declarations;
  `action_toggle_install_mark` gates on the highlighted row's `install` capability
  instead.

## Step 10 — Agent-CLI mixin cleanup (`plugins_browser_agent_clis.py`)

Delete: `_render_agent_clis`, `_record_agent_cli_bookmark`,
`_repaint_agent_cli_options`, `_agent_cli_row_label`, `_agent_cli_summary`,
`_agent_cli_status_message`, `_sync_agent_cli_visibility`, `_agent_cli_hints`,
`_agent_cli_option_list`, `_agent_cli_by_name`, `_highlighted_agent_cli_name`,
`_highlight_agent_cli`, `_on_agent_cli_highlighted`, `_render_agent_cli_detail`, and the
module-local `_ITEM_PREFIX` / `_DETAIL_PLACEHOLDER` / `_ACCENT`.

Keep and re-point:

- `_agent_cli_row(status)` — becomes the agent-CLI branch of `_row_text` (call it from
  the rendering mixin; `_install_method_label` and `_agent_cli_color` stay with it).
- `_current_agent_cli()` — now `self._highlighted_row()` narrowed to an `AgentCliStatus`
  payload.
- `_render_agent_cli_history(*, force=False)` — reads `_current_agent_cli()`, writes
  `#updates-history`, keeps the `_agent_cli_history_key` dedup.
- `action_toggle_history_scope` and `_agent_cli_detail_panel` — unchanged bodies.

Also delete the now-unused `_agent_cli_summary` / `_core_hints` / `_core_versions_panel`
static updates in `plugins_browser_workers.py::_start_load`; that method now refreshes
`#updates-header` and `#updates-hints` only, and captures
`self._restore_key = self._highlighted_key() or self._session_state.rows.identity`.

## Step 11 — Mark actions bridged off the row, not the sub-tab (`plugins_browser_agent_clis_actions.py`, `plugins_browser_install.py`)

Every `self._active_subtab == ...` conjunct becomes the **highlighted row's kind**. Add
a one-line comment at each site saying `sase-w0.4` replaces the bridge with one mark
set; do not widen the behavior here.

- `action_toggle_mark`: `row.kind == "agent-cli"` → `_toggle_agent_cli_mark()`,
  otherwise → `action_toggle_install_mark()`.
- `action_update_agent_clis`: consume `self._marked_agent_clis` only when the
  highlighted row is an agent CLI; otherwise target every updatable installed CLI, as
  today.
- `action_clear_marks_or_close`: try the highlighted row's kind first, then the other
  kind, then close — preserving today's "clears one disjoint half" semantics.
- `action_toggle_install_mark` / `action_clear_install_marks_or_close` /
  `_clear_install_marks` / `_clear_agent_cli_marks` update `#updates-hints`.

## Step 12 — Jump adapter and filter input

`plugins_browser_jump.py`: `_active_option_list()` → `_option_list()`; `_jump_repaint`
loses its branch and is one `self._rebuild_options()` call (which repaints its own hint
line). Rewrite the module docstring — the "Core hosts no list, so `'` is a silent no-op"
behavior it documents is exactly what this phase removes.

`plugins_browser_input.py`: brackets call `action_cycle_scope_reverse` /
`action_cycle_scope`; update the class docstring ("Brackets cycle the pane-local
scopes…").

`plugins_browser_controls.py`: `action_focus_filter`, `on_input_changed`,
`on_input_submitted`, `_set_filter_value` target `#updates-filter-input`;
`action_toggle_verbose` updates `#updates-hints`; delete `_active_option_list` and point
`action_next_option` / `action_prev_option` / `focus_default` at `_option_list()`.

## Step 13 — CSS (`src/sase/ace/tui/styles.tcss`, the block at ~8536-8674)

Replace the whole `/* Updates tab: Core / Plugins / Agent CLIs */` block with one
master/detail pair:

- `#updates-scopes` keeps today's `#updates-subtabs` rule verbatim
  (`height: 1; text-align: center; text-wrap: nowrap; text-overflow: clip; margin-bottom: 1;`).
- `#updates-header`: `width: 100%; height: auto; text-style: bold;`
- `#updates-filter-input`: `width: 100%; margin-bottom: 1;`
- `#updates-panels`: `width: 100%; height: 1fr;`
- `#updates-list-panel`: `width: 58;` (the wider of today's two, so
  `v1.2.3 → v1.2.4  [npm]` rows still fit) `height: 100%;`
- `#updates-status`, `#updates-detail-panel`, `#updates-detail-scroll`,
  `#updates-detail`, `#updates-history`: today's `#plugins-*` / `#agent-clis-history`
  rules with the new ids.
- `#updates-hints`:
  `height: auto; color: $text-muted; text-align: center; text-wrap: nowrap; text-overflow: clip;`
  — `auto` because `sase-w0.4` adds a second line; `nowrap`/`clip` so today's
  already-wide single line keeps clipping instead of wrapping and stealing a list row.
- `PluginsBrowserPane OptionList { height: 1fr; border: solid $secondary; }` is
  unchanged.

Delete every `#updates-subtab-*`, `#plugins-*`, `#agent-clis-*`, `#sase-core-versions`,
`#updates-current-banner`, and `#updates-core-hints` rule.

## Step 14 — Tests

**`tests/ace/tui/_plugins_browser_pane_helpers.py`**

- `_open_plugins_pane(page, *, session_state=None, scope="all")`: drop
  `pane._switch_to_subtab("plugins")`, call `pane._switch_to_scope(scope)` instead
  (default `"all"` so the existing suite still sees not-installed plugin rows).
- `_option_labels` reads `#updates-list`.
- `_highlight(pane, name)` matches `updates-row__plugin:<name>`; add
  `_highlight_row(pane, key)` for core / CLI rows and `_row_ids(pane)`.
- Add `_header_text(pane)` = `_render(pane._header_renderable())`.

**`tests/ace/tui/visual/_ace_config_center_modal_helpers.py`**: `_open_plugins_modal`
selects the `all` scope instead of `_switch_to_subtab("plugins")`.

These two edits unblock most of the suite.

**Mechanical renames across the Updates-pane tests** (`#plugins-list` → `#updates-list`,
`#plugins-hints` → `#updates-hints`, `#plugins-status` → `#updates-status`,
`#plugins-detail-scroll` → `#updates-detail-scroll`, `#plugins-filter-input` →
`#updates-filter-input`, `#agent-clis-list` → `#updates-list`, `#agent-clis-history` →
`#updates-history`, `plugin__x` → `updates-row__plugin:x`, `agent-cli__x` →
`updates-row__cli:x`, `pane._detail_name == "x"` → `pane._detail_key == "plugin:x"`,
`pane._agent_cli_detail_name == "x"` → `pane._detail_key == "cli:x"`,
`pane._flat_plugin_entries()` → `pane._flat_rows()`). Files:
`test_plugins_browser_pane_jump.py`, `..._loading.py`, `..._detail.py`,
`..._install.py`, `..._agent_clis.py`, `..._agent_clis_history.py`,
`..._all_current.py`, `test_admin_center_selection_resume.py`,
`test_plugins_catalog_scale_fixture.py`,
`visual/test_ace_png_snapshots_config_center_plugins.py`,
`visual/test_ace_png_snapshots_config_center_plugin_actions.py`.

**Tests that change meaning, not just ids:**

- `test_plugins_browser_pane_agent_clis.py::test_updates_subtabs_cycle_and_gate_plugin_actions`
  and `::test_updates_subtabs_handle_brackets_from_core_and_lists` become **scope**
  cycling tests: `]` / `[` walk `outdated → installed → all` and back, the strip's
  active tab follows, focus stays on `#updates-list`, and `check_action` follows the
  highlighted row rather than a sub-tab.
- `::test_agent_cli_session_restores_subtab_and_row_by_identity` → restores
  `state.updates.scope` and the `rows` bookmark identity `cli:codex`.
- `::test_updates_subtab_hints_share_projects_wording` → the one `_hints()` line
  contains `[ ] scope`, `a sync agents`, `u update core + plugins`, and `A update CLIs`.
- `::test_update_sase_action_remains_pane_wide` → iterate the three **scopes**.
- `test_plugins_browser_pane_jump.py::test_updates_core_subtab_apostrophe_is_an_inert_no_op`
  → **inverted**: `'` now jumps to core rows too; `check_action("jump_to_entry", ())` is
  `True` and a hint selects `core:sase`.
- `::test_updates_subtab_switch_clears_jump_hints` → a **scope** switch clears them.
- `::test_updates_plugins_apostrophe_paints_hints_skipping_headers` → 6 targets in the
  `all` scope (2 core + 4 plugins), hints `[0]`…`[5]`, section headers still unhinted.
- `test_plugins_browser_pane_all_current.py` → assert on `_header_text(pane)` /
  `pane._all_up_to_date()` instead of `#updates-current-banner`'s `display`, and on
  `pane._core_detail_panel(package)` instead of `pane._core_versions_panel()`.
- `test_plugins_browser_pane_loading.py::test_plugins_pane_empty_catalog` → an empty
  catalog still lists the SASE section; assert no plugin rows and that the list stays
  visible. `::test_plugins_pane_error_state` keeps today's status-placeholder rule. The
  three `_core_versions_panel` tests → `_core_detail_panel(...)`.
  `::test_updates_filter_forwards_brackets_and_tab_switches_main_tab` asserts a scope
  change instead of a sub-tab change.
- `test_admin_center_selection_resume.py`: the `plugins` and `agent-clis` cases collapse
  into **one** `_ResumeCase("updates", "6", ("right_square_bracket",))` (`]` moves
  `installed → all`); `_surface_selection` reads `#updates-list`, `_detail_selection`
  reads `pane._detail_key`, and the prefix map entry becomes
  `"updates": "updates-row__"`.
- `test_plugins_browser_pane_agent_clis_history.py`'s `H`-scope test drives scope
  selection plus a highlight onto `cli:claude` instead of sub-tab switching, and asserts
  `check_action("toggle_history_scope", ())` is `False` while a plugin row is
  highlighted.

**New file `tests/ace/tui/test_plugins_browser_pane_scopes.py`:**

1. Every core package, plugin, and registered CLI appears exactly once, in the right
   section, in each scope.
2. Scope membership: a manual-only CLI and an error-only row are both in `Outdated`;
   `Installed` holds only installed rows; `All` is the union.
3. Landing: a fresh pane with an outdated row lands on the **first outdated** row; with
   none, on the first row; a session bookmark wins over both.
4. One filter needle matches a core package, a plugin, and a CLI by their own fields in
   one search.
5. `'` jumps across all three kinds in one hint space.
6. The cursor survives a refresh, a filter, and a scope switch **by identity** (and
   `_selection_guard` leaves no stale intent behind).
7. `check_action` offers `x` / `U` / `H` / `I` / `space` on exactly the right rows and
   refuses them on the others (notably: no `H` on a plugin row, no `U`/`x` on a core or
   CLI row, no `space` on a core row).
8. Scope-strip labels carry live counts and the active scope follows `]` / `[`.
9. A mark set on a plugin row survives a scope switch to a scope where the row is
   hidden, and is not pruned by the reload sweep.

**Bench `tests/ace/tui/bench_plugins_catalog_scale.py`:** open at the **`All`** scope
(the helper's new default) so n=2000 still measures the full catalog; move
`#plugins-filter-input` → `#updates-filter-input`; `_matched_row_count` still sums
`pane._grouped` (now `UpdateRow` lists) and must still equal
`scale_filter_match_count(n)` — the scale catalog's `q`-prefixed names cannot match
`sase` / `sase-core`, so the two new core rows do not perturb it. Add an explicit
"highlight the first row with the `install` capability" step immediately before
`_measure_install_mark`, because the merged list's landing row is now a core row when
nothing is outdated. `TARGET_P95_MS = 16.0` at every size, **including n=2000**, is
unchanged and must hold. Update the module docstring (it says "Updates > Plugins
sub-tab").

**PNG suite `tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugins.py`:**
replace the seven `_switch_to_subtab` calls with scope selections and explicit
highlights (`config_center_plugins_long_description` in particular must highlight
`plugin:megasync`, because nothing in that fixture is outdated and the pane now lands on
a core row). Do **not** add new scenarios — that is `sase-w0.5`. Rebaseline the goldens
this phase invalidates with `just test-visual -- --sase-update-visual-snapshots`, then
**inspect every regenerated PNG** in `.pytest_cache/sase-visual/` before committing: a
golden that silently lost the detail panel or the section headers is the failure mode
this phase is most likely to hide.

## Verification

Per `sase/memory/lint_and_test.md`:

```bash
just install                 # ephemeral workspace clones may have drifted deps
just check                   # inline; hand to /sase_monitor if it runs long
just test-visual             # this phase changes rendered output
```

Then, explicitly:

```bash
rg -n "_active_subtab|_switch_to_subtab|action_cycle_subtab|UpdatesSubTab|_SUBTAB" src/sase/ace/tui/modals/plugins_browser_*.py src/sase/ace/tui/modals/config_center_session.py
rg -n "plugins-list|plugins-hints|plugins-detail|plugins-status|plugins-summary|plugins-filter-input|agent-clis-|sase-core-versions|updates-current-banner|updates-core-hints|updates-subtab" src/
just _lint-symvision
just _lint-toobig
pytest -s -m slow tests/ace/tui/bench_plugins_catalog_scale.py
```

The two `rg` sweeps must come back empty for the Updates pane (`config_hub_pane.py`,
`projects_pane.py`, and `project_inventory_types.py` keep their own unrelated
`_active_subtab` / `action_cycle_subtab`; do not touch them). The bench must print
`filter_keystroke` and `j_press` p95 under 16 ms at every size including n=2000; record
the numbers in the bead close note but **do not** write the baseline file (`sase-w0.5`
owns that).

`just check-full` is **not** run here — it is the epic's landing gate and runs only
through `/sase_monitor` before `sase-w0`'s combined tree lands.

## Acceptance

- Opening Updates shows one list with `── SASE ──`, `── Plugins · Built-in ──`,
  `── Plugins · Community ──`, and `── Agent CLIs ──` sections under a cycled Outdated /
  Installed / All strip, with the cursor already on the first problem.
- `]` / `[` cycle scopes; `/` searches all three domains; `'` jumps across all three
  kinds; `j` / `k` / `g` / `G` / `Ctrl+D` / `Ctrl+U` work everywhere, including where
  `'` is a documented no-op today.
- `i` / `I` / `Space` / `x` / `U` / `m` / `u` / `A` / `H` / `a` / `r` / `o` / `v` /
  `Esc` keep their keys and their targets; only _when_ they are offered changes, from
  the active sub-tab to the highlighted row's capability set.
- The highlight survives a refresh, a filter change, and a scope switch by identity; a
  lazy latest completion patches one row and never reclassifies or reorders it.
- `just check` and `just test-visual` are green, no `_active_subtab` reference remains
  in `src/`, and the n=2000 `All`-scope bench holds the 16 ms p95 budget.

## Risks and how this plan defuses them

1. **Rule 12 (the highest-risk line of the merge).** Two `ProgrammaticSelectionGuard`s
   become one, and `_rebuild_options` becomes the only programmatic assigner of
   `highlighted`. `prepare()` immediately precedes the assignment; `should_ignore`
   clears synchronously. New test 6 covers it directly.
2. **Pruning against the visible list.** Both prune sweeps now read `self._rows`, so a
   scope or filter can never silently delete a mark. New test 9 covers it.
3. **Per-keystroke sorting.** Rows are sorted once per load on the worker thread, so
   `select_rows` stays a single `O(n)` filtering pass and the n=2000 filter budget is
   unaffected.
4. **Silent PNG damage.** Every regenerated golden is inspected before commit, and the
   plugin/agent-CLI row text is reproduced byte-for-byte so the diff is confined to the
   layout change.
5. **Phase bleed.** The three explicit non-goals above, plus a comment at each
   sub-tab-conjunct bridge site naming `sase-w0.4`, keep `sase-w0.3` and `sase-w0.4`
   (both in progress) landing on a tree they still recognize.
