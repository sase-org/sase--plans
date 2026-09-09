---
tier: epic
title: Apostrophe entry jump on every Admin Center tab
goal: "Pressing the apostrophe key inside any SASE Admin Center working tab enters
  adaptive hint-jump mode and moves that tab's selection, using one shared
  implementation with the Logs tab's existing back-stack semantics.

  "
phases:
  - id: shared
    title: Shared pane entry-jump mixin and Logs migration
    depends_on: []
    size: medium
    description: "shared: add the reusable pane entry-jump mixin that owns hint
      allocation, the pending-prefix state machine, and the bounded back stack, then
      migrate the Logs pane onto it with no behavior change.

      "
  - id: tasks
    title: Tasks tab jump
    depends_on:
      - shared
    size: small
    description: "tasks: wire the Tasks pane task list onto the shared mixin."
  - id: xprompts
    title: XPrompts tab jump
    depends_on:
      - shared
    size: small
    description: "xprompts: wire the XPrompt browser's non-header rows onto the shared
      mixin and reserve the apostrophe in the filter-first browser input while the
      filter is empty.

      "
  - id: projects
    title: Projects tab jump across all three sub-tabs
    depends_on:
      - shared
    size: medium
    description: "projects: wire the projects list and the shared repo/workspace
      inventory pane base onto the shared mixin so each sub-tab jumps within its own
      rows.

      "
  - id: updates
    title: Updates tab jump for Plugins and Agent CLIs
    depends_on:
      - shared
    size: medium
    description: "updates: wire the Updates pane's active-sub-tab option list onto the
      shared mixin, skipping group headers and no-opping on the list-free Core sub-tab.

      "
  - id: config
    title: Config tab jump over visible tree rows
    depends_on:
      - shared
    size: medium
    description: "config: wire the config field tree onto the shared mixin by decorating
      node labels in place, without rebuilding the tree or disturbing fold state.

      "
  - id: statistics
    title: Statistics tab jump to numbered views
    depends_on:
      - shared
    size: small
    description: "statistics: make the apostrophe arm the existing numbered-view
      selection so the already visible strip numbers act as the tab's jump hints.

      "
  - id: docs
    title: Documentation and full-suite verification
    depends_on:
      - tasks
      - xprompts
      - projects
      - updates
      - config
      - statistics
    size: small
    description:
      "docs: document the Admin Center-wide apostrophe jump in the ACE guide and run the
      exhaustive verification lane over the combined tree."
proposed_by: bbugyi200.athena.uo
status: done
bead_id: sase-gv
create_time: 2026-09-09 19:49:41
---

- **PROMPT:**
  [prompts/202608/admin_center_apostrophe_jump.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/admin_center_apostrophe_jump.md)
- **BEAD:**
  [sase-gv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gv/README.md)

# Plan: Apostrophe entry jump on every Admin Center tab

## Problem

The `'` (apostrophe) entry-jump key works on the SASE Admin Center's **Logs** tab and
nowhere else. `LogsPane` is the only Admin Center pane that binds `apostrophe` — a
repo-wide search for the key across `src/sase/ace/tui/modals/` finds it only in
`logs_pane.py` (plus two unrelated modals, `notification_modal.py` and
`model_picker_modal.py`).

The Admin Center has seven working tabs, defined in
`src/sase/ace/tui/modals/config_center_catalog.py`:

| #   | Tab        | Pane class           | Primary module            |
| --- | ---------- | -------------------- | ------------------------- |
| 1   | Config     | `ConfigPane`         | `config_pane_widget.py`   |
| 2   | Logs       | `LogsPane`           | `logs_pane.py`            |
| 3   | Projects   | `ProjectsPane`       | `projects_pane.py`        |
| 4   | Statistics | `StatisticsPane`     | `statistics_pane.py`      |
| 5   | Tasks      | `TasksPane`          | `tasks_pane.py`           |
| 6   | Updates    | `PluginsBrowserPane` | `plugins_browser_pane.py` |
| 7   | XPrompts   | `XPromptBrowserPane` | `xprompt_browser_pane.py` |

Six of the seven do nothing when `'` is pressed. Three of them also host sub-tabs whose
rows live in separate widgets (`RepoInventoryPane` / `WorkspaceInventoryPane` under
Projects; the plugins and agent-CLI lists under Updates), so "all tabs" really means
nine browsable surfaces.

The Admin Center home landing page is deliberately **out of scope**: it is not a working
tab, and its seven rows already have dedicated `1`-`7` keys plus mouse clicks.

## Goal

`'` behaves the same way on every Admin Center working tab: it paints adaptive hints
over that tab's selectable entries, a hint character moves the selection there, `'`
again returns to the previous position, and `Esc` cancels. Exactly one implementation of
that state machine exists in the tree when this epic lands.

## Shared behavior contract

Every tab must reproduce the semantics `LogsPane` already implements (see
`src/sase/ace/tui/modals/logs_pane.py:349-432`). This contract is the acceptance
criterion for each per-tab phase:

1. **Enter.** `'` allocates hints for the tab's current jump targets in visual order via
   `build_jump_hint_maps` (`src/sase/ace/tui/actions/navigation/jump_hints.py`), marks
   jump mode active, repaints the rows with a `[hint] ` prefix, and switches the pane's
   hint line to its JUMP variant.
2. **Hint alphabet.** Hints come from `build_jump_hint_maps` unchanged: one character
   from `0-9a-zA-Z` for up to 62 targets, two characters beyond that, truncated at `ZZ`.
   Hints are case-sensitive, so keys are normalized with
   `normalize_jump_key(event.key, event.character)` before matching.
3. **Complete / pend / cancel.** A key is matched with `match_jump_hint`. `PENDING`
   keeps jump mode open and stores the prefix; `COMPLETE` performs the jump; `INVALID`
   exits jump mode. `escape` exits jump mode without moving.
4. **Back stack.** A second `'` while jump mode is active pops the most recent
   still-valid index off a bounded back stack and jumps there without pushing; with an
   empty stack it jumps to the first hint target. A hint-driven jump pushes the pre-jump
   index only when the index actually changes. The stack is capped at 10 entries, oldest
   dropped.
5. **Stale data.** When a reload changes the row identities, hints _and_ the back stack
   are cleared. When identities are unchanged but a hint now points outside the row
   range, only the hints are cleared.
6. **Empty surface.** With zero jump targets, `'` is a silent no-op — jump mode is never
   entered.
7. **Text inputs win.** While a filter or value input owns focus, `'` remains an
   ordinary printable character. The one exception is the filter-first XPrompts browser,
   covered in its own phase.

## Design: one mixin, per-tab adapters

Copying ~110 lines of jump state machine into six more panes is not acceptable. This
epic adds one shared mixin and gives each pane a small adapter.

### New module `src/sase/ace/tui/modals/pane_entry_jump.py`

The mixin owns everything in the contract above that is not pane-specific. Suggested
surface (the `shared` phase owns the final naming):

- `PaneEntryJumpMixin` — the host-facing API:
  - `jump_mode_active` — read-only property used by renderers and hint lines.
  - `jump_hint_for(index) -> str | None` — the hint to prefix onto row `index`, or
    `None`.
  - `action_jump_to_entry()` — the action bound to `apostrophe`.
  - `handle_jump_key(key) -> bool` — returns `True` when the key was consumed by jump
    mode.
  - `exit_jump_mode()` / `clear_jump_hints()` — teardown, with and without a repaint.
  - `invalidate_jump_hints(*, identities_changed, target_count)` — the rule 5 helper
    panes call from their reload paths.
- Host hooks each pane implements (all pane-private, so they keep a `_jump_` prefix):
  - `_jump_target_count() -> int`
  - `_jump_current_index() -> int | None`
  - `_jump_select_index(index) -> None` — move the selection through the pane's
    _existing_ selection path, so bookmarks, `ProgrammaticSelectionGuard`, detail loads,
    and debouncers keep working.
  - `_jump_repaint() -> None` — rebuild the rows and refresh the hint line.

Jump state is held in a small module-private dataclass (`mode_active`, `pending_prefix`,
`hint_to_index`, `index_to_hint`, `back_stack`) created lazily through a private
accessor, so the mixin needs no `__init__` and does not perturb the panes' Textual MRO.
Do **not** use mutable class attributes for the maps.

Hint decoration reuses the existing `apply_jump_hint_prefix` helper in
`src/sase/ace/tui/modals/saved_agent_group_revival_rendering.py:73`; re-export it from
the new module rather than writing a second prefix renderer.

### Wiring pattern per pane

Each pane phase does the same four things:

1. Add `("apostrophe", "jump_to_entry", "Jump")` to the pane's `BINDINGS`.
2. Add an `on_key` branch that gives `handle_jump_key` first refusal, then treats a bare
   `'` as `action_jump_to_entry()`, following `logs_pane.py:142-165`. Panes that already
   define `on_key` extend it; panes that do not gain one. Where the pane already has a
   bespoke key handler with an early return for a focused `Input`
   (`statistics_pane.py:276-282`), keep that guard first.
3. Prefix row labels with `jump_hint_for(index)` inside the pane's existing option/label
   builder.
4. Add `': jump` to the pane's normal hint line and a
   `JUMP ' <back|first>  <esc> cancel` variant, matching `LogsPane._hints`.

### Index space

Hint indices are **positions in the pane's logical row list**, not raw `OptionList`
indices, because several lists interleave disabled group headers. Panes with headers
(XPrompts, Updates) must map logical index to option index before assigning
`highlighted`, and must not allocate hints to header rows. `XPromptBrowserPane` already
distinguishes them with the `__header__` id prefix, and
`PluginsBrowserControlsMixin._is_item` already does the same check for Updates.

### Per-tab jump targets

| Tab        | Targets                                                         | Selection effect                                             |
| ---------- | --------------------------------------------------------------- | ------------------------------------------------------------ |
| Config     | visible tree rows (sections and leaves) in render order         | move the tree cursor, update the detail panel                |
| Logs       | log source rows                                                 | highlight the source and load its tail (already implemented) |
| Projects   | active sub-tab's rows: filtered projects, repos, or workspaces  | highlight the row and update the detail panel                |
| Statistics | the seven numbered views                                        | switch to that view                                          |
| Tasks      | task rows                                                       | highlight the task and show its output                       |
| Updates    | active sub-tab's item rows (Plugins, Agent CLIs); Core has none | highlight the row and render its detail                      |
| XPrompts   | non-header xprompt rows                                         | highlight the row and update the preview                     |

### Two deliberate design decisions

**Statistics does not get row hints.** The pane renders Rich tables with no row cursor,
so hinting its body would produce hints that cannot be selected. Its real navigable
entries are the seven views, and the view strip already paints `1`-`7` next to each name
(`PanelTabStrip(show_numbers=True)`). On this tab `'` therefore arms the existing
numbered-view selection — the same state `select_view` (`0` by default) arms — and the
already-visible strip numbers act as the hints. Rendering a second, zero-based hint
alphabet on top of the numbers would be actively confusing.

**Updates → Core is a no-op.** The Core sub-tab has no list at all
(`plugins_browser_layout.py:98-103`); it is a banner plus a version panel. Under rule 6
that is a silent no-op, which is the same thing Logs already does with zero sources.
Hinting the three sub-tabs instead was considered and rejected: `[` / `]` already cycle
them, and a sub-tab jump would mean something different from a row jump on every other
surface.

### Non-goals

- `Ctrl+O` / `Ctrl+Shift+O` (`jump_to_entry_fast` / `jump_to_entry_forward`) stay
  unimplemented in the Admin Center. Logs does not have them today and the request is
  specifically about `'`.
- The `` ` `` cross-tab Jump All modal is untouched.
- Making the Admin Center panes read `jump_to_entry` from the keymap registry instead of
  hard-coding `"apostrophe"` is out of scope. Logs, `notification_modal.py`, and
  `model_picker_modal.py` all hard-code it today; changing that convention is a separate
  change and would widen this epic's blast radius for no user-visible gain.
- The Admin Center home landing page, as stated above.

## Verification for every phase

Every phase runs `just install` first (workspaces are ephemeral and may be stale), then
`just check` before finishing. Phases that change a pane's rendered hint line must also
run `just test-visual` and refresh only their own tab's goldens with
`--sase-update-visual-snapshots`; goldens live in `tests/ace/tui/visual/snapshots/png/`.
Hint lines are already near the 120-column snapshot width, so confirm the new `': jump`
text does not truncate the rest of the line — shorten an existing segment if it does.

New tests follow the existing Logs jump tests in
`tests/ace/tui/test_logs_pane.py:408-560`: drive the real modal with
`pilot.press("apostrophe")` and assert on jump state, the resulting selection, and the
back stack.

---

## Phase `shared`: Shared pane entry-jump mixin and Logs migration

Create `src/sase/ace/tui/modals/pane_entry_jump.py` implementing `PaneEntryJumpMixin`
exactly as described under **Design** above, covering every numbered item of the
**Shared behavior contract**.

Then migrate `LogsPane` (`src/sase/ace/tui/modals/logs_pane.py`) onto it:

- Delete `_log_jump_mode_active`, `_log_jump_pending_prefix`, `_log_jump_hint_to_index`,
  `_log_jump_index_to_hint`, `_log_jump_back_stack`, `_clear_log_jump_hints`,
  `_exit_log_jump_mode`, `_handle_log_jump_key`, `_jump_to_source_index`,
  `_log_jump_hints_are_valid`, and the body of `action_jump_to_entry`.
- Implement the four `_jump_*` host hooks in terms of the pane's existing helpers:
  `_jump_target_count` from `len(self._source_options)`, `_jump_current_index` from
  `_highlighted_index()`, `_jump_select_index` from
  `_rebuild_options(selected_index=...)` plus `_start_load(...)` when the index changed,
  and `_jump_repaint` from `_rebuild_options` plus `_update_hints`.
- `_render_source_option_label` asks the mixin for the hint via `jump_hint_for` and
  decorates with the shared `apply_jump_hint_prefix`.
- Keep the reload invalidation in `_apply_load_result` (`logs_pane.py:201-212`),
  expressed through the mixin's `invalidate_jump_hints` helper: source ids changed ⇒
  hints and back stack cleared; ids unchanged but a hint out of range ⇒ hints only.
- `_hints()` keeps both its current strings verbatim.

Update `tests/ace/tui/test_logs_pane.py` to the mixin's attribute names. All six
existing Logs jump tests must still pass with unchanged assertions about _behavior_ —
only the attribute names they read may change. Treat any behavior difference as a bug in
the migration, not as a test to relax.

Add focused unit tests for the mixin itself (hint allocation width, pending-prefix
matching, back stack cap of 10, no push when the index is unchanged, both invalidation
rules, and the zero-target no-op) in a new `tests/ace/tui/test_pane_entry_jump.py`.

If Symvision flags the new module's private/public split, resolve it by reading
`sase/memory/symvision.md` through the `/sase_memory_read` skill rather than by adding
pragmas.

**Done when:** the mixin exists with unit tests, `LogsPane` contains no bespoke jump
state machine, and the Logs jump behavior is byte-for-byte what it is today.

---

## Phase `tasks`: Tasks tab jump

`TasksPane` (`src/sase/ace/tui/modals/tasks_pane.py`) shows a `TaskList` of merged
in-memory and durable task rows built by `TasksPaneSelectionMixin._create_options`
(`src/sase/ace/tui/modals/tasks_pane_selection.py:130-135`). Rows are 1:1 with
`self._tasks`, so the logical index space is simply `range(len(self._tasks))`.

- Add `PaneEntryJumpMixin` to the `TasksPane` bases and
  `("apostrophe", "jump_to_entry", "Jump")` to `BINDINGS`. `TasksPane` has no `on_key`
  today; add one that follows the Logs pattern.
- `_jump_select_index` must go through the existing selection path so the output panel,
  the `ProgrammaticSelectionGuard`, and the `TasksSessionState` bookmark all stay
  correct — reuse the helpers around `tasks_pane_selection.py:291-315` rather than
  assigning `highlighted` directly.
- Decorate labels in `_create_options` via `jump_hint_for`.
- Tasks refreshes itself on a 0.25s interval (`tasks_pane.py:108`) and reloads from the
  durable store. Apply rule 5 on every refresh that rebuilds rows, so a background
  refresh never leaves a hint pointing at a dismissed task.
- `_hints()` (`tasks_pane.py:165-169`) gains `': jump` and a JUMP variant. It is
  currently a `@staticmethod`; it must become an instance method to read jump state.

Tests go in `tests/ace/tui/test_tasks_pane.py` (helpers in `_tasks_pane_helpers.py`):
entering jump mode paints hints, a hint selects the right task and shows its output, `'`
`'` returns, `Esc` cancels without closing the modal, and a refresh that removes the
hinted task clears jump mode instead of selecting the wrong row.

Refresh `config_center_tasks` goldens.

**Done when:** `'` jumps between task rows and a live task refresh cannot strand a stale
hint.

---

## Phase `xprompts`: XPrompts tab jump

`XPromptBrowserPane` (`src/sase/ace/tui/modals/xprompt_browser_pane.py`) is the one
**filter-first** tab: `focus_default` focuses `BrowserFilterInput`, not the list
(`xprompt_browser_pane.py:145-150`). Its `OptionList` mixes disabled `__header__` group
rows with `item__<name>` rows.

- Allocate hints only over the flat item rows from `_get_flat_items()`; map a logical
  index to its option index with the existing `_option_index_for_item` before assigning
  `highlighted`.
- Decorate item labels in `_create_options` / `create_browser_options`
  (`src/sase/ace/tui/modals/xprompt_browser_options.py`). Header rows are never
  decorated.
- `_jump_select_index` routes through `_restore_highlight_and_preview` / the existing
  highlight path so the preview panel, the bookmark, and the selection guard stay in
  sync.
- Filtering rebuilds the list in `on_input_changed`; apply rule 5 there.
- **Filter-first reservation.** `BrowserFilterInput.on_key`
  (`src/sase/ace/tui/modals/xprompt_browser_filter_input.py:38-58`) already reserves
  digit keys for the Admin Center's numbered tabs _while the filter is empty_, letting
  them become ordinary text once the filter has content. Extend exactly that pattern to
  `apostrophe`: with an empty filter, stop the event and call the pane's
  `action_jump_to_entry()`; with a non-empty filter, let it be typed. Update the class
  docstring to describe the new reservation.
- `browser_hint_text` (`src/sase/ace/tui/modals/xprompt_browser_helpers.py`) gains
  `': jump` and a JUMP variant.

Tests: extend `tests/ace/tui/test_xprompt_browser_load_keymap.py` or add a sibling —
hints skip group headers; a hint selects the right xprompt and updates the preview; `'`
with an empty filter enters jump mode; `'` with `bug` already typed appends a literal
apostrophe and does not jump.

**Done when:** `'` jumps from the filter-first XPrompts tab without breaking apostrophes
in filter text.

---

## Phase `projects`: Projects tab jump across all three sub-tabs

`ProjectsPane` (`src/sase/ace/tui/modals/projects_pane.py`) hosts three sub-tabs in a
`ContentSwitcher`: its own projects `OptionList`, plus `RepoInventoryPane` and
`WorkspaceInventoryPane`, which both derive from `_InventoryPaneBase`
(`src/sase/ace/tui/modals/project_inventory_panes.py:101`).

Do the wiring in **two places**, not three:

1. `ProjectsPane` + `ProjectListControllerMixin` for the projects sub-tab. Targets are
   `self._filtered_records`; decorate in `_create_options`
   (`src/sase/ace/tui/modals/project_list_controller.py:76-86`) and select through
   `_refresh_options(preferred_project=...)` so marks, the detail debouncer, and the
   selection guard are preserved. Note the empty-state placeholder row
   (`Option(..., id="empty")`) is not a jump target.
2. `_InventoryPaneBase` once, so Repos and Workspaces both inherit the behavior.

Because the inventory panes are descendants of `ProjectsPane`, a key pressed while a
repo or workspace list has focus reaches the inventory pane's binding first and is
stopped there. Belt and braces: add `"jump_to_entry"` to
`ProjectsPane._PROJECT_ONLY_ACTIONS` (`projects_pane.py:122-140`) so the parent's
binding is disabled via `check_action` whenever the active sub-tab is not `projects`.

`_ProjectsFilterInput` and `_InventoryFilterInput` are **not** changed: both panes are
list-first, so `'` stays ordinary filter text while an input has focus, per contract
rule 7.

Hint lines: `_hints_text()` for projects and the inventory panes' own hint lines each
gain `': jump` plus a JUMP variant. Apply rule 5 wherever records reload —
`action_reload_projects`, the inventory worker completion in `on_worker_state_changed`,
and filter changes.

Tests in `tests/ace/tui/test_projects_pane.py` plus the repo/workspace inventory tests:
each sub-tab jumps within its own rows; switching sub-tabs while jump mode is active
leaves no stale hints; a reload after a project is deleted clears jump mode.

Refresh `config_center_projects`, `config_center_repos`, and `config_center_workspaces`
goldens.

**Done when:** `'` jumps correctly on all three Projects sub-tabs and never crosses
between them.

---

## Phase `updates`: Updates tab jump for Plugins and Agent CLIs

`PluginsBrowserPane` (`src/sase/ace/tui/modals/plugins_browser_pane.py`) is a single
pane hosting three sub-tabs. `PluginsBrowserControlsMixin._active_option_list`
(`src/sase/ace/tui/modals/plugins_browser_controls.py:189-197`) already returns
`#plugins-list`, `#agent-clis-list`, or `None` for Core — that is the whole sub-tab
dispatch the jump adapter needs.

- Jump targets are the active list's **item** rows, filtered with the existing
  `_is_item` header check (`plugins_browser_controls.py:205-211`). With
  `_active_option_list()` returning `None` (Core), `_jump_target_count()` returns `0`
  and `'` is a no-op.
- Decorate plugin rows in the plugins option builder and agent-CLI rows in the agent-CLI
  list builder (`src/sase/ace/tui/modals/plugins_browser_rendering.py` and
  `plugins_browser_agent_clis.py`); leave `_HEADER_PREFIX` rows alone.
- `_jump_select_index` assigns `highlighted` through the same path `action_next_option`
  uses, so the detail debouncer, `_detail_name` dedup guard, and the two selection
  guards behave normally.
- Add `"jump_to_entry"` to the `browse_only` set in
  `PluginsBrowserLayoutMixin.check_action`
  (`src/sase/ace/tui/modals/plugins_browser_layout.py:182-192`) so the binding is
  disabled on Core rather than silently doing nothing.
- Sub-tab switches (`_switch_to_subtab`), filter changes (`on_input_changed`), and
  catalog reloads all clear jump state per rule 5.
- `PluginsFilterInput` is unchanged: `'` stays filter text while it has focus.
- Hint lines: `_hints()` and `_agent_cli_hints()` gain `': jump` plus JUMP variants.
  `_core_hints()` is left alone — Core has no jump.

Tests alongside `tests/ace/tui/test_plugins_browser_pane.py` and
`test_plugins_browser_pane_agent_clis.py`: hints skip group headers on both sub-tabs, a
hint selects the right row and renders its detail, `'` on Core does nothing and does not
enter jump mode, and switching sub-tabs clears hints.

Refresh `config_center_plugins` and `config_center_agent_clis` goldens.

**Done when:** `'` jumps on Plugins and Agent CLIs, and is an inert no-op on Core.

---

## Phase `config`: Config tab jump over visible tree rows

`ConfigPane` (`src/sase/ace/tui/modals/config_pane_widget.py`) is the only tab backed by
a Textual `Tree` rather than an `OptionList`. It already tracks every visible row in
`_node_by_path` (insertion-ordered, so it matches render order) and can move the cursor
with `ConfigPaneNavigationMixin._move_cursor`.

- Jump targets are the visible rows in `_node_by_path` order — both section nodes and
  leaves, which is what `j`/`k` already traverse.
- **Do not call `_rebuild_tree` to paint hints.** `_rebuild_tree` re-adds every section
  with `expand=True` (`config_pane_widget.py:220`) and would destroy the user's fold
  state. Instead decorate in place: iterate `_node_by_path` and `node.set_label(...)`,
  wrapping the label from `render_row_label(view, path)` in `apply_jump_hint_prefix`.
  Exiting jump mode restores the plain labels the same way.
- `_jump_select_index` calls `_move_cursor(tree, node)`, which already updates the
  detail panel and records the bookmark through `_update_detail`.
- Apply rule 5 in `_rebuild_tree`, `action_toggle_modified`, `action_refresh`,
  `_do_jump`, and the filter's `on_input_changed` — anything that changes the visible
  row set.
- `ConfigFilterInput` is unchanged: `'` stays filter/jump-path text while it has focus.
- `_hints()` (`config_pane_widget.py:330-336`) gains `': jump` and a JUMP variant. That
  line already advertises `:: jump` for the dotted-path jump; word the new entry so the
  two are distinguishable — for example `': hint jump` alongside the existing
  `:: path jump`.

Tests in `tests/ace/tui/test_config_pane_widget_navigation.py`: hints cover the visible
rows in render order; a hint moves the cursor and repaints the detail; entering and
leaving jump mode preserves collapsed sections; a filter change while jump mode is
active clears the hints.

Refresh `config_center_config` goldens.

**Done when:** `'` jumps between config tree rows without disturbing fold state, and the
two jump keys read as distinct in the hint line.

---

## Phase `statistics`: Statistics tab jump to numbered views

`StatisticsPane` (`src/sase/ace/tui/modals/statistics_pane.py`) has no row cursor. Per
the design decision above, `'` arms the pane's existing numbered-view selection.

- Add `("apostrophe", "jump_to_entry", "Jump")` to the pane's keymap-derived bindings
  (`build_statistics_bindings`) or to `BINDINGS`, whichever keeps the pane's
  configurable-keymap story intact — this pane builds its bindings from
  `StatisticsPaneKeymaps`, so do not bypass that.
- `action_jump_to_entry` sets `self._pending_view_select = True` and refreshes the hint
  line, exactly as the `select_view` key already does.
- Extend `on_key` (`statistics_pane.py:276-308`) so `apostrophe` also cancels an armed
  selection, the way `select_view` does at line 286. Keep the existing early return
  while `#statistics-custom-range` has focus, so `'` remains typable in the custom-range
  input.
- The armed-state hint line already exists; make sure it reads sensibly when armed by
  `'` — the seven strip numbers are the hints, and `Esc` or any non-digit cancels.
- This phase does **not** add `PaneEntryJumpMixin` to `StatisticsPane`. Say so in a
  short comment on `action_jump_to_entry` so a future reader does not "fix" the
  inconsistency without context.

Tests in `tests/ace/tui/test_statistics_view_number_select.py`: `'` then `3` selects
Projects; `'` then `Esc` cancels; `'` while the custom-range input has focus types an
apostrophe and does not arm.

Refresh `config_center_statistics` goldens only if the hint line changed.

**Done when:** `'` on Statistics arms view selection identically to `select_view`.

---

## Phase `docs`: Documentation and full-suite verification

Update `docs/ace.md`:

- In the Admin Center material under **Global Keybindings** (around line 2036), state
  that `'` is an Admin Center-wide entry-jump key and that hints follow the shared
  adaptive alphabet already described at lines 112-116.
- Add the key to the per-tab sections that document keybindings — **Projects Tab**,
  **Statistics Tab**, **XPrompt Browser** → Keybindings, **Tasks Tab** → Keybindings,
  and **Updates Tab** — naming each tab's jump targets from the table in this plan.
- Document the two design decisions explicitly: Statistics jumps between its seven
  numbered views, and the Updates Core sub-tab has no jump targets.
- Document the XPrompts filter reservation next to the existing empty-filter digit
  reservation.

The `?` help modal is **not** expected to change: its binding tables are scoped to the
ChangeSpecs, Agents, and Axe tabs and mention the Admin Center only as
`Admin Center: 1-7 jump, # back`
(`src/sase/ace/tui/modals/help_modal/changespecs_bindings.py:280`,
`axe_bindings.py:151`). Confirm that is still accurate and leave it alone if so.

Then verify the combined tree:

```bash
just install
just check-full
just test-visual
```

`just check-full` is required here rather than `just check`: this epic touches nine pane
surfaces across seven modules, which is exactly the broadening case the repo's two-speed
verification rule calls out.

Finally, walk the real TUI once (`sase ace`, then `#`) and press `'` on each of the
seven tabs and on the Projects and Updates sub-tabs, confirming the contract's enter /
hint / back / cancel behavior end to end.

**Done when:** the guide documents the key on every tab, the full suite and visual suite
are green, and a manual pass over all nine surfaces matches the contract.
