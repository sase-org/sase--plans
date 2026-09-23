---
tier: epic
title: Services tab Service Procs and Scheduled Routines panels
goal: 'The Services tab sidebar renders as two titled, tribe-panel-style panels —
  "Service Procs" (daemon procs plus oneshots) and "Scheduled Routines" (routines with
  their jobs) — each with at-a-glance metadata in its title, and J/K jump to the
  first/last node of the next/previous panel.

  '
phases:
  - id: service-panels
    title: "Phase 1: Two-panel Services sidebar with titled panels"
    depends_on: []
    size: medium
    description: "service-panels: split the BgCmdList sidebar into statically composed
      Service Procs and Scheduled Routines panels over the unchanged global _axe_items
      index, with metadata titles, focus chrome, shared height allocation, width
      settling, scheduler-fold removal, docs, glossary strands, tests, and visual
      goldens.

      "
  - id: service-panel-jk
    title: "Phase 2: J / K panel jumps on the Services tab"
    depends_on:
      - service-panels
    size: small
    description:
      "service-panel-jk: add tab-scoped focus_next/prev_service_panel keymaps on J/K
      that select the first/last node of the adjacent non-empty panel with wrap, plus
      availability gating, palette/help/docs discoverability, tests, and goldens."
proposed_by: bbugyi200.athena.0q5--1
create_time: 2026-09-23 18:26:48
status: wip
---

- **PROMPT:**
  [prompts/202609/services_tab_panels.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/services_tab_panels.md)

# Services Tab: "Service Procs" and "Scheduled Routines" Panels

## Goal

Split the Services tab's single left sidebar into two stacked, titled panels, the same
way the Agents tab splits agents into tribe panels:

1. **Service Procs**: every service proc. That means the daemon rows (Scheduler,
   Telegram, and so on) plus the oneshot rows under the existing dim `── oneshots ──`
   divider. Per the glossary, a oneshot _is_ a service proc, so it belongs here.
2. **Scheduled Routines**: every routine (lumberjack) row, with its job (chop) rows
   nested under it.

Each panel gets a styled border title that carries at-a-glance metadata about what the
panel holds. A new pair of `J` / `K` keymaps jumps to the first or last node of the
panel below or above, exactly like `J` / `K` on the Agents tab.

## Current State (for orientation)

- `src/sase/ace/tui/_app_layout.py` composes one `BgCmdList` (`#bgcmd-list-panel`, an
  `OptionList`) inside `#bgcmd-list-container`.
- `_build_axe_items()` (`src/sase/ace/tui/actions/axe_display/_loader_items.py`) builds
  one flat `self._axe_items` list. It nests the routine/job rows under the `scheduler`
  `ServiceProcItem` behind the `"service:scheduler"` fold key. No key ever collapses
  that fold. When `_service_status is None`, it falls back to top-level routine rows.
  Oneshot `BgCmdItem` rows come last.
- `self.current_idx` indexes `_axe_items`. A great deal of code reads
  `_axe_items[current_idx]`: row actions, the command palette, link-follow and the link
  trail, the jump-all modal, entry-jump hints, and config actions. **The global flat
  list and global index must survive unchanged in meaning.**
- `_refresh_axe_display()` (`actions/axe_display/_render.py`) repaints everything and
  calls `bgcmd_list.update_list(...)`. j/k go through
  `_refresh_axe_display_debounced()`, which calls
  `bgcmd_list.update_highlight(current_idx)` and then debounces the dashboard.
- Width: `BgCmdList.WidthChanged` → `on_bg_cmd_list_width_changed`
  (`actions/_event_widgets.py`) clamps to `MIN_BGCMD_LIST_WIDTH` (35) and
  `MAX_BGCMD_LIST_WIDTH` (80), and reserves 40 cells for the dashboard.
- These already do what this plan wants on the Agents tab, so use them as the reference:
  - `actions/agents/_display_panel_titles.py`: the title grammar
    `{icon} {label} · {count} [chip] badges`.
  - `agent_count_chip.py`: the `[R2 F1]` chip grammar.
  - `actions/agents/_display_panel_layout.py`: `_apply_panel_heights`,
    `_settle_agent_list_container_width`, and `_focus_focused_panel_widget`.
  - `actions/agents/_panel_navigation.py`: `_change_focused_agent_panel`, the `J`/`K`
    implementation.
  - CSS in `styles.tcss`: `#agent-list-container AgentList`, `.-focused-panel`, and
    `.agent-panel-separated`.

## Design

### Layout and look

```
┌ ⚙ Service Procs · 8 [R4 F1 S1] ▷1 ✓1 ─┐   ← teal name; gold border when focused
│▌ [*] Scheduler                        │
│▌ [*] Telegram                         │
│▌ [!] Agents Sync  crash_loop · 3r     │
│▌ [-] Web  disabled here               │
│── oneshots ──                         │
│▷ #1 just check  running · 1m          │
│✓ #2 make docs  exit 0 · 4m ago        │
└───────────────────────────────────────┘
                                          ← one blank separator row (margin-top: 1)
┌ ◷ Scheduled Routines · 3 [R3] · 11 jobs ●1 ⚠1 ┐  ← gold name
│▌ [*] hooks  ⚠1  120c                  │
│  └─ [✓] rebase_stale                  │
│  └─ [●] refresh_prs                   │
│▌ [*] mentors  40c                     │
│  ...                                  │
└───────────────────────────────────────┘  ← last panel fills to the container bottom
```

- Service Procs sits on top, because it is the host's short, fairly fixed roster.
  Scheduled Routines sits below it. The two panels always keep this order.
- Both panels are `BgCmdList` instances composed statically. There are always exactly
  two, so the Agents tab's dynamic mount/retire machinery is not needed.
  - Service Procs: `BgCmdList(panel_key="service_procs", id="service-procs-panel")`.
  - Scheduled Routines:
    `BgCmdList(panel_key="scheduled_routines", id="scheduled-routines-panel")`.
  - The old `#bgcmd-list-panel` id is retired. The rename is deliberate: it forces every
    `query_one("#bgcmd-list-panel")` caller to decide which panel it means. Current
    callers are `_render.py`, `_startup_mount.py`, `_prompt_bar_mount.py`,
    `styles.tcss`, `_app_layout.py`, and the tests.
- Panel identity hues match the existing row taxonomy palette in `bgcmd_list.py`:
  - Service Procs: icon `⚙`, `bold #00D7AF` (the service teal). `⚙` already marks procs
    in Agents panel titles.
  - Scheduled Routines: icon `◷`, `bold #FFD700` (the routine gold). `◷` is in the
    bundled Fira Code, and it reads as "scheduled".
- Focus chrome copies the tribe panels:
  - The panel that holds the selection gets the `-focused-panel` class, which draws a
    `solid #FFD700` border. The other panel keeps `solid $primary` and shows **no** row
    highlight (`clear_highlight()`).
  - In the focused panel's title, the count and the chip brackets use `#FFD75F`
    (`_PANEL_SELECTED_CHROME_STYLE`). In the unfocused title they use `#AFAFAF`.
  - Services has no whole-panel focus, so there is no `❖` marker.
- CSS changes, replacing the `#bgcmd-list-panel` rule:
  - `#bgcmd-list-container BgCmdList { width: 100%; height: 1fr; border: solid $primary; padding: 0 1; }`
    (the height is overridden at runtime; see below).
  - `#scheduled-routines-panel { margin-top: 1; }`.
  - `#bgcmd-list-container BgCmdList.-focused-panel { border: solid #FFD700; }`.
  - Keep the existing `BgCmdList` rules: scrollbar gutter, nowrap, and highlighted-row
    styles.

### Panel titles (the metadata)

Build the titles as Rich `Text` with the same grammar as `agent_panel_border_title`:
`{icon} {Label} · {count} [chip] badges…`. Every number comes from caches the render
path already holds, so **title building never touches disk**. Zero counts are always
suppressed.

**Service Procs**: `⚙ Service Procs · {nodes} [R{n} W{n} F{n} S{n}] ▷{n} ✓{n} ✗{n}`,
then optional badges.

- `nodes` is the number of selectable nodes in the panel: daemon rows plus visible
  oneshot rows.
- The chip counts daemon rows by the existing shared severity vocabulary,
  `service_proc_severity()` in `_service_severity.py`, so it can never disagree with the
  row markers:
  - `R` = `ok` (running), style `bold #00D7AF`.
  - `W` = `warn` (unavailable, or exited cleanly while wanted), style `bold #FFAF5F`.
  - `F` = `fail` (crash_loop, backoff, unclean exit, or stopped while wanted), style
    `bold #FF5F5F`.
  - `S` = `muted` (intentionally stopped or disabled), style `#AFAFAF`.
  - Letters use the chrome style and counts use their metric style, exactly like
    `format_agent_count_chip`.
- After the chip come the oneshot badges. They reuse the row glyphs and styles already
  in `bgcmd_list.py` (`_ONESHOT_RUN_GLYPH`, `_ONESHOT_OK_GLYPH`, `_ONESHOT_FAIL_GLYPH`,
  and the `_oneshot_failed` logic): `▷` running, `✓` exit 0, `✗` failed or killed.
- Invariant, which must be unit-tested: chip counts + oneshot badge counts == `nodes`.
- Optional badges, appended in this order and separated by `·`:
  - `+{n} hidden` in `dim`, when the `.` toggle (`_axe_cmds_hidden`) hides `n` oneshots.
    Otherwise hidden rows would silently vanish.
  - `host {state}` in `service_host_style(state)`, when the snapshot's host state is not
    `running`.
  - `status unavailable` in `bold #FFAF5F`, when `_service_status is None`.

**Scheduled Routines**:
`◷ Scheduled Routines · {routines} [R{n} E{n} I{n}] · {jobs} jobs ●{n} !{n} ?{n} ⚠{n}`,
then an optional badge.

- `routines` is the number of configured routines. The chip splits them by their cached
  `LumberjackStatus.status`, matching the row marker:
  - `R` = `running`, style `bold #00D7AF`.
  - `E` = `error`, style `bold #FF5F5F`.
  - `I` = any other value, or no status (idle), style `#AFAFAF`.
- `jobs` counts **all** configured jobs across routines, from
  `_axe_lumberjack_chop_names`, including jobs hidden by a collapsed routine fold. The
  title describes the panel's content, not only its visible rows.
- Job badges come from each job's newest cached run in `_axe_chop_snapshots`, using the
  same glyphs and styles as the job row marker in `_format_chop_option`:
  - `●` `running`, style `bold green`.
  - `!` `failure` or `timeout`, style `bold red`.
  - `?` `missing_script`, style `bold yellow`.
- The overrun badge `⚠{n}` is the sum of `LumberjackSnapshot.overrun_chop_count`, in the
  same `bold #FFAF5F` as the per-routine overrun chip.
- Scheduler badge: routines are no longer drawn under the Scheduler row, so the title
  carries the scheduler's state. When `_service_status` contains a proc named
  `scheduler` that is not running, append ` · scheduler {state}`:
  - The text is `disabled` when `enablement.enabled` is false, `unavailable` when
    `available` is false, and otherwise `proc.state`.
  - Style it with `service_proc_style(...)`.
  - Show no badge when the status is unknown or no `scheduler` proc exists.
- Invariant, which must be unit-tested: the `R` + `E` + `I` counts == `routines`.

**Width**: a panel requests `max(content width, title.cell_len + 4)`, like
`AgentList.update_border_title`. The sidebar takes the max over both panels, still
clamped by the existing min, max, and terminal cap. When the terminal cap bites,
Textual's border-title ellipsis truncates the title. That is acceptable, and a narrow
golden covers it.

### Empty panels

A panel with no nodes renders **one disabled placeholder option**. It is not a node, so
it is never selectable and never counted.

- Service Procs placeholder: dim `No service procs`.
- Scheduled Routines placeholder: dim `No routines configured · {add key} to add`, where
  the key comes from `key_display_name(self._keymap_registry.app.add_axe_item)`.

The panel stays mounted at `placeholder + 2` border rows. Its title still renders, with
count `0`. This way the layout never jumps, and the empty state explains itself.

### Selection model (keep one global index)

- `_build_axe_items()` emits items in **visual order**:
  1. every `ServiceProcItem`, with no nested routines;
  2. then the oneshot `BgCmdItem` rows, unless `_axe_cmds_hidden` is set;
  3. then every routine `LumberjackItem`, each followed by its `ChopItem` rows when its
     `lumberjack:<name>` fold is expanded.

  The `service_status is None` fallback no longer changes the shape: routines always go
  in their own panel.

- Delete the `"service:scheduler"` fold concept everywhere:
  - its default-expand in `_build_axe_items`;
  - the pending-selection expand at `_loader_items.py` ~121-123;
  - the expand and rollback in `_follow_chop_link` / `_link_follow_targets.py` ~334-400;
  - the matching assertions in `tests/ace/tui/test_link_follow_entry_points.py`.

  Keep every `lumberjack:<name>` fold behavior unchanged.

- Add a pure, Textual-free panel index module, `actions/axe_display/_panels.py`:
  - `ServicesPanelKey = Literal["service_procs", "scheduled_routines"]`.
  - `SERVICES_PANEL_ORDER`.
  - `services_panel_key_for_item(item)`: `ServiceProcItem` and `BgCmdItem` map to
    `service_procs`; `LumberjackItem` and `ChopItem` map to `scheduled_routines`.
  - A frozen `ServicesPanelIndex` built from `_axe_items`. For each panel it holds
    `items`, `global_indices`, and `global_to_local`, plus `panel_for_global(idx)` and
    `local_idx_for(key, global_idx)`. Model it on `models/agent_panel_index.py`'s
    `_PanelSlice`.
  - `_build_axe_items()` rebuilds it and caches it as `self._axe_panel_index`.
    Initialize that in `actions/_state_init_late.py` and add it to the type stub in
    `_loader_state.py`.
- The focused panel is always _derived_: the panel of `current_idx`, or `service_procs`
  when `_axe_items` is empty. Cache the last-painted focused key as
  `self._axe_painted_panel_key`, so the highlight fast path can tell when focus crossed
  panels.
- Lowercase j/k keep their global wrap (`actions/navigation/_basic.py`), so j at the
  bottom of Service Procs continues into the first routine row, matching the Agents tab.
- Nothing about tab-switch restore (`_axe_last_idx` / `_axe_last_item_key`), identity
  restore, `toggle_hide_reverted`, `_switch_to_axe_view`, or the jump-all modal changes.
  They all speak global indices, which keep their meaning.

### Widget changes (`widgets/bgcmd_list.py`)

- `BgCmdList.__init__(..., panel_key: str = "service_procs")` stores `self.panel_key`.
- `SelectionChanged(index, panel_key)`: `index` stays panel-local.
- `update_list(...)`:
  - receives the panel's local slice, a local `current_idx` (use `-1` for no highlight,
    via `clear_highlight()`), and local `jump_hints`;
  - takes a new optional `empty_placeholder: Text | None`;
  - keeps the oneshot divider rule, now meaning "this panel has daemon rows and oneshot
    rows";
  - records a `rendered_line_count`: options plus divider lines, and 1 for a
    placeholder;
  - updates `_content_requested_width` and posts `WidthChanged` only when the combined
    requested width changes.
- `update_border_title(title: Text)`: set `border_title`, recompute the requested width,
  and post `WidthChanged` if it changed. Mirror `AgentList.update_border_title` and
  `_refresh_requested_width`.
- `clear_highlight()`: set `highlighted = None` inside the existing
  `_programmatic_update` guard, cleared synchronously in `finally` (tui_perf rule 12).
- `update_highlight(local_idx)` stays as it is.

### Render and refresh wiring (`actions/axe_display/_render.py`)

- Replace the single `bgcmd_list.update_list(...)` block in `_refresh_axe_display()`
  with `_paint_axe_panels()`. For each panel it:
  1. slices items and hints through `_axe_panel_index`;
  2. calls `update_list(...)` with the same cached status maps as today, so there is
     still no disk I/O;
  3. builds and sets the title;
  4. toggles `-focused-panel`.

  Then it applies heights (below) and records `_axe_painted_panel_key`.

- Replace `bgcmd_list.update_highlight(self.current_idx)` in
  `_refresh_axe_display_debounced()` with `_refresh_axe_panel_highlights()`:
  - **Same focused panel**: call `update_highlight(local)` on it. This is the j/k hot
    path and must stay O(1) widget work.
  - **Focus crossed panels**: `clear_highlight()` the old panel, highlight the new one,
    swap `-focused-panel`, and re-set both titles (only their focus chrome changed;
    build them from caches). Then move Textual focus (below).

  It never rebuilds options. The dashboard stays behind `_axe_detail_debouncer`
  (tui_perf rule 7).

- Textual keyboard focus follows the focused panel, so OptionList's own
  `up`/`down`/`home`/`end` act on the visible selection. It moves **only** when a
  `BgCmdList` already owns app focus (`self.focused`) and the hint input bar is not
  active. Background refreshes, such as a new oneshot landing through
  `_switch_to_axe_view`, therefore never steal focus from the prompt bar or a modal. Put
  this in a helper `_focus_axe_focused_panel(*, force: bool = False)`.
  - `force=True` skips the ownership check. The two existing
    `query_one("#bgcmd-list-panel").focus()` call sites use it:
    `actions/_startup_mount.py` ~200 and `actions/agent_workflow/_prompt_bar_mount.py`
    ~301.
- `on_bg_cmd_list_selection_changed` (`actions/_event_widgets.py`) maps
  `(event.panel_key, event.index)` to a global index through `_axe_panel_index`, then
  sets `current_idx`. This covers mouse clicks and OptionList-native keys in either
  panel.
- Width: `on_bg_cmd_list_width_changed` sizes the container from **both** panels'
  `_requested_width`, not from `event.width` alone.
  - Extract the clamp into a pure
    `services_sidebar_width(requested_widths, *, terminal_width)` in `_app_layout.py`,
    beside `agent_list_column_width`.
  - Also settle the width in the same frame at the end of `_paint_axe_panels()`, like
    `_settle_agent_list_container_width`, so rows and column change together.
- Heights: extract the Agents allocation math into a pure helper,
  `src/sase/ace/tui/util/panel_heights.py`:
  `allocate_panel_heights(content_rows, collapsed, container_height, *, filler_idx) -> list[PanelHeight] | None`.
  - `PanelHeight` is a frozen dataclass with `value: float` and
    `unit: Literal["cells", "fr"]`, plus a `to_scalar()` that builds the Textual
    `Scalar`.
  - It returns `None` when `container_height` is 0.
  - Move the math verbatim from `_apply_panel_heights`: natural = rows + 2 (2 when
    collapsed), one separator row per extra panel, the filler gets `1fr` when everything
    fits, per-panel minimums of `2 + min(rows, 2)`, the fix-smallest-first overflow
    pass, and `rows + 1` fraction weights.
  - Agents calls it with `filler_idx` = the first non-collapsed panel and keeps
    `option_count` as rows. **This must be a pure behavior-preserving refactor.** The
    Agents height tests (`test_agent_panels_display.py` and the
    `agents_overflowing_panel_full_height` golden) must pass untouched.
  - Services calls it from a new `_apply_axe_panel_heights()` with
    `content_rows = [panel.rendered_line_count, ...]`, `collapsed=[False, False]`, and
    `filler_idx=1`, so Scheduled Routines absorbs spare rows and the stack reaches the
    bottom.
  - When the container height is still 0 (first paint, or the tab is hidden), schedule
    one thin synchronous `call_after_refresh(self._reapply_axe_panel_heights)`.
  - `on_resize` (`_event_widgets.py`) also calls `_reapply_axe_panel_heights()`, which
    is cheap and never rebuilds options.

### Rust core boundary

The data here is presentation state only: panel partitioning, title chips, focus, and
layout, derived from the snapshots the TUI already caches. Another frontend has no need
to reproduce it, so nothing changes in `sase-core`.

### J / K (phase 2)

- New app keymaps sit next to the Agents pair in `src/sase/default_config.yml`, under
  `ace.keymaps.app`, with the comment `# Services tab side panels`:
  - `focus_next_service_panel: "J"`
  - `focus_prev_service_panel: "K"`
- Semantics mirror `_change_focused_agent_panel`:
  - `J` selects the **first** node of the next panel. `K` selects the **last** node of
    the previous panel.
  - Both wrap, and both skip panels with no nodes.
  - With two panels, from Service Procs: `J` goes to the first routine row, `K` to the
    last visible routine or job row. From Scheduled Routines: `J` goes to the first
    service row, `K` to the last service or oneshot row.
  - "Last" means the last _rendered_ node, so a collapsed routine contributes only its
    routine row.
  - It is a pure no-op when no other panel has nodes, or off the Services tab.
- On the way in, the action:
  - begins `_jk_perf` (`next_service_panel` / `prev_service_panel`);
  - calls `_record_jk_navigation()`;
  - pushes the jump-back origin with `_push_entry_jump_index_origin_if_changed`, so
    `ctrl+o` returns to where you were (the Agents tab also records an anchor);
  - then sets `current_idx`. The existing `watch_current_idx` →
    `_refresh_axe_display_debounced` path handles the highlight, focus crossing, and the
    debounced dashboard.
- Tab-scoped dispatch: `J`/`K` are currently bound only to the Agents actions, which are
  enabled on every tab and simply return off-tab. Gate each pair to its tab in
  `_app_action_availability.py`, so Textual falls through to the other binding for the
  same key. `tests/ace/tui/test_show_agent_run_log_keymap.py` shows this pattern for
  `V`.

## Phases

### Phase 1: Two-panel Services sidebar with titled panels

Implement everything in the Design section except `J`/`K`:

1. Add the pure panel index `actions/axe_display/_panels.py`. Include only what this
   phase uses; phase 2 adds the navigation helpers, so Symvision sees no unused symbols.
2. Add title stats and builders in `actions/axe_display/_panel_titles.py`:
   - frozen dataclasses `ServiceProcsPanelStats` / `ScheduledRoutinesPanelStats`;
   - pure `service_procs_panel_stats(...)` / `scheduled_routines_panel_stats(...)` over
     the cached state;
   - `service_procs_panel_title(stats, *, focused)` /
     `scheduled_routines_panel_title(stats, *, focused)`, returning Rich `Text`;
   - a small local `[L{n} …]` chip helper following `format_agent_count_chip`'s grammar.
     Do not bend the agent chip's fixed metric set.
3. Make the `BgCmdList` widget changes.
4. Change the layout and CSS, retiring `#bgcmd-list-panel`.
5. Reorder `_build_axe_items`, delete the `service:scheduler` fold, and cache
   `_axe_panel_index`.
6. Wire up `_paint_axe_panels`, `_refresh_axe_panel_highlights`,
   `_focus_axe_focused_panel`, the selection mapping, and the width.
7. Extract `util/panel_heights.py`; refactor Agents onto it and use it for Services.
8. In-app copy: update `widgets/axe_onboarding.py` (~134-135) to describe the two panels
   in place of "The Scheduler proc nests its routines". Leave `J`/`K` out; phase 2 adds
   it.
9. Docs:
   - `docs/ace.md`: "Keybindings: Services Tab" (~2932-2967), "Sidebar Row Taxonomy"
     (~2969-2988), "Dynamic Sidebar Width" (~3038-3045, where titles now count toward
     width), and the Navigation table (~3086-3099).
   - `docs/axe.md`: "Services Tab Views" (~1369-1377) and the scheduler-nesting prose
     (~1632-1652). Reword "select the top-level scheduler row before pressing `x`/`r`;
     those keys do nothing on its nested routines and jobs" to refer to routine and job
     rows in the Scheduled Routines panel.
10. Memory, which the user confirmed before this plan was proposed. Use
    `/sase_memory_write`, then run `sase memory init`:
    - Glossary strand `sase/memory/glossary/service-node.md` becomes: "A service node is
      one selectable row of the Services tab: a service-proc node or a oneshot node in
      the Service Procs panel, or a routine node and its job nodes in the Scheduled
      Routines panel. A service-proc node is stable across process restarts; panel
      titles, section dividers, and the host status line are chrome, not nodes."
    - Glossary strand `sase/memory/glossary/sase-node.md`: change "Grouping banners and
      tribe-panel titles are chrome, not nodes" to "Grouping banners and panel titles
      (Agents tribe panels, Services panels) are chrome, not nodes."

Tests:

- **Unit tests for the pure modules:**
  - panel partition, global↔local mapping, and the empty-list default;
  - both title builders: glyphs, styles, focus chrome, zero suppression, the
    hidden-oneshot, host, status-unavailable and scheduler badges, and the two sum
    invariants;
  - `allocate_panel_heights` (fits → filler `1fr`; overflow; below-minimum fractional
    fallback; `None` at height 0);
  - `services_sidebar_width`.
- **Widget tests** (extend `tests/test_bgcmd_list.py` and
  `tests/ace/tui/widgets/test_bgcmd_list_*`): local indices, `clear_highlight`, the
  placeholder (disabled, not counted, no `SelectionChanged`), `rendered_line_count` with
  the divider, title width affecting the requested width, and `panel_key` on
  `SelectionChanged`.
- **App-level tests** (extend `test_axe_selection_identity.py` and
  `test_axe_navigation.py`, or add `test_axe_panels.py`):
  - the item order is service → oneshots → routines, including when
    `_service_status is None`;
  - j at the last service row moves into Scheduled Routines and moves the
    `-focused-panel` class and highlight;
  - a click in the unfocused panel selects the right global row;
  - the in-panel j/k fast path never calls `update_list` or does disk I/O (reuse the
    `test_navigation_does_not_read_from_disk` pattern);
  - Textual focus follows only when a `BgCmdList` owns focus;
  - identity restore across the reorder;
  - hiding oneshots updates the `+N hidden` badge.
- **Updates to existing tests**: the fold assertions in
  `test_link_follow_entry_points.py`, the width tests in
  `test_bgcmd_left_panel_width.py`, and every test that queries `#bgcmd-list-panel`
  (`grep -rn "bgcmd-list-panel" src tests`).
- **Agents regression**: the existing panel-height tests pass unchanged.
- **Visual goldens** (`tests/ace/tui/visual/`):
  - Add a fixture that sets a realistic `ServiceStatusSnapshot`. The host is running.
    The procs are scheduler running, telegram running, agents_sync `crash_loop` with 3
    restarts, and a disabled proc. Add 2 oneshots (one running, one exit 0) and the
    existing routine/job fixture data. None of today's fixtures set `service_status`.
  - New goldens:
    - `services_panels_120x40`: selection on the Scheduler row.
    - `services_panels_routine_selected_120x40`: a job selected, so Scheduled Routines
      has the gold border.
    - `services_panels_empty_routines_120x40`: no routines, the scheduler stopped, the
      placeholder, and the scheduler badge.
    - `services_panels_narrow_70x36`: width cap and title truncation.
  - Then run `just fix-tui-screenshots` through `/sase_monitor` and inspect **every**
    creation and update group in the retained report. Every existing `axe_*`,
    `launch_context_bar_services_*`, `link_rail_axe_*`, and `help_guide_axe_*` golden is
    expected to change (new titles and the second panel). Nothing outside the Services
    tab should change except where the Agents height refactor was meant to be a no-op;
    **any Agents golden diff is a bug**.
- Verify with `sase tool run check`. Also capture a live `sase screenshot` of the
  Services tab (`-- -t services`) and look at it for polish: title spacing, focus
  border, and separator row.

No feature flag: this phase ships a complete, coherent layout. Lowercase j/k, the mouse,
entry-jump hints, and the jump-all modal all traverse both panels. Phase 2's `J`/`K` is
an additive keymap, not the missing half of a feature.

### Phase 2: J / K panel jumps on the Services tab

1. Keymap plumbing:
   - `focus_next_service_panel: "J"` / `focus_prev_service_panel: "K"` in
     `src/sase/default_config.yml` (next to the Agents pair, ~758-760).
   - Fields in `keymaps/app_keymaps.py`.
   - `_BINDING_META` entries in `keymaps/metadata.py` (`"Next Panel"` / `"Prev Panel"`,
     `show=False`).
   - Fallback `Binding`s in `bindings.py`.
   - Two `_CONTEXTUAL_APP_DUPLICATES` pairs in `keymaps/registry.py`
     (`focus_next_agent_panel`↔`focus_next_service_panel` and the prev pair), with a
     "Tab-disjoint" comment.
   - The schema needs no change, because `app` is `additionalProperties: string`.
2. Availability: in `_app_action_availability.py`, return `False` for
   `focus_next/prev_agent_panel` off the Agents tab and for
   `focus_next/prev_service_panel` off `SERVICES_TAB`.
3. Navigation helpers in `actions/axe_display/_panels.py`:
   - `ServicesPanelIndex.first_global(key)` / `last_global(key)`;
   - `adjacent_nonempty_panel(current_key, *, forward) -> ServicesPanelKey | None`, with
     wrap, which skips empty panels and never returns the current panel.
4. Action mixin `actions/axe_display/_panel_navigation.py` (`AxePanelNavigationMixin`),
   mixed into `AxeDisplayMixin` in `actions/axe_display/__init__.py`:
   - `action_focus_next_service_panel` / `action_focus_prev_service_panel` →
     `_change_focused_service_panel(forward=...)`, following the Design section's "J /
     K" semantics exactly (perf begin, nav-gate record, jump-origin push, set
     `current_idx`, and `call_after_refresh(jk_perf.mark_painted)`).
   - Keep the body synchronous and I/O-free.
5. Discoverability:
   - Command palette entries in `commands/_app_metadata_actions.py`: "Focus next
     Services panel" / "Focus previous Services panel", group `"Services"`, `AXE_ONLY`.
   - A `J / K` row "Jump into next / prev panel" in the Navigation section of
     `modals/help_modal/axe_bindings.py`, rendered with `d(a.focus_next_service_panel)`.
   - A `J`/`K` mention in `widgets/axe_onboarding.py`.
   - `J`/`K` added to `docs/ace.md` (the Services keybindings and Navigation table) and
     to any `docs/configuration.md` keymap listing that already lists the Agents pair.
   - Leave the footer alone. Agents does not advertise `J`/`K` there either.

Tests:

- **Unit tests** for `adjacent_nonempty_panel` and first/last: wrap, empty-panel skip,
  single non-empty panel → `None`.
- **Action tests**:
  - `J` from Service Procs → first routine node;
  - `K` from Service Procs → last rendered node of Scheduled Routines, including the
    collapsed-routine case;
  - `J` from Scheduled Routines wraps to the first service node;
  - `K` from Scheduled Routines → last service or oneshot node;
  - no-op when the other panel is empty (no history push, no nav-gate side effects
    beyond the record);
  - no-op off-tab;
  - `ctrl+o` returns to the origin row.
- **Keymap tests**: the registry maps `J` to both actions, and a user rebinding of one
  is not reverted by duplicate validation.
- **Availability tests**: each pair is disabled off its tab.
- **A real pilot test** that presses `J` and `K` on the Services tab **and** on the
  Agents tab through the actual bindings, and asserts each tab's action ran. No test
  presses `J` through a pilot today.
- **Visual goldens**: add `services_panels_after_J_120x40` (start on Scheduler, press
  `J`). Regenerate `help_guide_axe_120x40` through `/sase_monitor`
  `just fix-tui-screenshots`, and inspect the report.
- Verify with `sase tool run check`.
