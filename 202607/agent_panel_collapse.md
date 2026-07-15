---
tier: tale
goal: 'Repurpose the Agents-tab `H` / `L` keymaps so they collapse and expand an entire
  agent panel (tag bucket) instead of stepping every fold. A collapsed panel shrinks
  to a border-only strip that still shows its `#tag · N [S… R… W…]` count summary,
  stops contributing its (hidden) row widths to the agent-list column width, and stays
  selectable via `J` / `K` so it can be re-expanded.

  '
create_time: 2026-07-15 11:32:59
status: done
prompt: 202607/prompts/agent_panel_collapse.md
---

# Plan: Collapse / expand whole agent panels on the Agents tab

## Product context

On the `sase ace` Agents tab the left column stacks one bordered panel per tag bucket (`(untagged)`, `#chop`,
`#sase-64`, …). Each panel's border title is its `#tag · N [S… R… W…]` summary. Today `H` (`hooks_or_collapse_all`) and
`L` (`expand_all_folds`) try to step _every_ in-panel group/workflow fold one level at a time. That behavior is
confusing, brittle, and effectively unused.

Repurpose `H` / `L` on the Agents tab into a genuinely useful, intuitive gesture: **collapse or expand the whole panel
that owns the current selection.** A collapsed panel becomes a compact border-only strip — its count summary stays
visible — and it no longer forces the whole column wide just because it contains long agent rows. This lets the user
hide a noisy or finished tag bucket and reclaim horizontal space for the panels they care about, exactly as shown in the
reference screenshot where collapsing the widest panel (`#chop`) should narrow the entire agent-list column.

Scope note: this is a presentation/navigation change to Textual state and rendering in this repo. It does not touch
shared backend/domain behavior, so no `../sase-core` change is required (see `rust_core_backend_boundary`).

## Target behavior (specification)

Everything below is Agents-tab only. The AXE and ChangeSpecs branches of the same actions are unchanged, and the
existing Tools-panel detail-level routing that `H` / `L` perform first (`_route_tools_detail_level("min"/"max")`) stays
ahead of the panel logic.

- **`H` — collapse the focused panel.** Collapses the panel that contains the current selection (the focused panel
  always owns the current agent/banner). The panel drops to a border-only strip showing only its title
  (`#chop · 3 [S1 W2]`), prefixed with a collapsed chevron (`▸`). Selection stays on that panel — its border shows the
  focused highlight. No-op when: the Tools panel consumed the key; there is only one panel; panels are in merged ("All
  agents") mode; or the panel is already collapsed.
- **`L` — expand the focused collapsed panel.** Reverses `H` for the focused, collapsed panel and re-anchors selection
  to that panel's first selectable row (first visible agent, or first collapsed in-panel banner). No-op when the focused
  panel is not collapsed (after Tools routing).
- **`J` / `K` — cycle panel focus, including collapsed panels.** `J`/`K` (`focus_next/prev_agent_panel`) already move
  focus between panels; they must be able to land on a collapsed panel. Landing on a collapsed panel selects the panel
  itself (its border highlights); there are no in-panel rows to land on. From there `L` expands it, or `J`/`K` moves on.
- **`j` / `k` inside a collapsed focused panel** have nothing to step and are no-ops.
- **Width.** A collapsed panel must not contribute its hidden agent-row widths to the agent-list column width. It
  contributes only enough width to render its own title fully. Collapsing the widest panel therefore narrows the whole
  column to fit the remaining expanded panels (per the reference screenshot).
- **Height.** A collapsed panel occupies a fixed border-only height regardless of its position; freed vertical space
  goes to the expanded panels.
- **Persistence.** Collapsed state is in-memory session state (like existing fold state, which is not persisted to
  disk). It is pruned when a panel disappears and cleared when toggling split/merged grouping.

## Design overview

Introduce one new piece of app state: a set of collapsed panel keys, stored on the app alongside the existing panel/fold
state (mirrors how `_group_fold_registry` and `_fold_manager` are app-owned and survive refreshes). Panel collapse is
**orthogonal** to the existing in-panel group-fold registries and per-workflow `FoldStateManager`: a collapsed panel
simply renders zero rows, and its untouched in-panel fold state is restored verbatim when it is expanded again. Keep the
collapsed set on the app rather than on `AgentPanelGroup` to avoid threading it through the many `AgentPanelGroup`
construction sites.

Because each panel is its own `AgentList` (`OptionList`) widget mounted in `#agent-list-container`, and each widget
already auto-sizes width from its widest row and broadcasts a `WidthChanged` message that the container reduces to a
`max()`, the width behavior falls out naturally: a collapsed panel renders no rows and instead requests only a
title-sized width, so the container's `max()` drops to the remaining expanded panels. The border title already renders
on the top border line, so a border-only-height widget still shows the count summary.

Reuse the existing panel-set refresh path used by `action_toggle_agent_panel_grouping`
(`_invalidate_agent_panel_cache()` then `_refresh_agents_display(list_changed=True)`). `H`/`L` are discrete,
non-autorepeat actions, so a single full panel refresh is the right, established path — do not invent a new refresh code
path (TUI-perf rule 4). Make the memoized per-keystroke nav-stop structure collapse-aware so navigation stays correct
and O(1) under autorepeat (TUI-perf rules 7–8).

## Change areas (design-level, with anchors)

### 1. New state

- `src/sase/ace/tui/actions/_state_init.py` (near where `_panel_group` / `_agent_panels_grouped` are initialized): add
  `self._collapsed_panel_keys: set[PanelKey] = set()`.

### 2. Actions — repurpose `H` / `L`, add collapse/expand, remove dead fold-stepping

`src/sase/ace/tui/actions/agents/_folding.py`:

- In `action_hooks_or_collapse_all`, replace the `current_tab == "agents"` branch (currently
  `self._collapse_all_folds()`) with a call to a new `self._collapse_focused_panel()`. Keep the Tools routing and the
  AXE / ChangeSpecs branches intact.
- In `action_expand_all_folds`, replace the `current_tab == "agents"` branch (currently `self._expand_all_folds()`) with
  a new `self._expand_focused_panel()`. Keep Tools / AXE / ChangeSpecs branches intact.
- Add `_collapse_focused_panel()`:
  - No-op unless on the Agents tab, split mode, more than one panel, and the focused key is not already collapsed.
  - Add `focused_key` to `_collapsed_panel_keys`; clear `_current_group_key` and `current_attempt_number`; anchor
    `current_idx` to the first index of that panel (via `rendered_panel_slice`) so the detail panel keeps context.
  - `_invalidate_agent_panel_cache()` then `_refresh_agents_display(list_changed=True)`.
- Add `_expand_focused_panel()`:
  - No-op unless on the Agents tab and the focused key is currently collapsed.
  - Discard the key from `_collapsed_panel_keys`; recompute the panel's first selectable stop and set focus to it (agent
    index or in-panel banner key) — this relies on the nav-stop cache being collapse-aware (§4) so the freshly-expanded
    stops are returned.
  - `_invalidate_agent_panel_cache()` then `_refresh_agents_display(list_changed=True)`.
- Remove the now-unreachable Agents-tab fold-stepping helpers `_collapse_all_folds`, `_expand_all_folds`, and their
  private helpers if they become unused (`_get_focused_panel_workflow_keys`, and `_get_all_workflow_keys` if unused —
  verify with a repo grep). The AXE `_collapse_all_axe_folds` / `_expand_all_axe_folds` are separate and must stay.

### 3. Panel title — collapsed chevron affordance

`src/sase/ace/tui/actions/agents/_display_panel_titles.py`, `agent_panel_border_title(...)`: add an optional
`collapsed: bool = False` parameter that prepends a collapsed chevron glyph (`▸ `, matching the collapsed in-panel
banner glyph) so a collapsed panel reads as closed and re-openable, e.g. `▸ #chop · 3 [S1 W2]`. Default `False` keeps
all current call sites and existing title tests unchanged.

### 4. Navigation — make stops/visibility collapse-aware

`src/sase/ace/tui/actions/agents/_navigation_order.py`:

- `_panel_navigation_stops`: when the focused panel key is in `_collapsed_panel_keys`, return an empty stop list (the
  collapsed panel has no in-panel selectable rows). Add the focused-panel-collapsed signal to the `_nav_stops_cache`
  signature so the cache invalidates on collapse/expand even when `_panel_group` identity, focused index, fold version,
  mode, and merge flag are unchanged (required for `_expand_focused_panel` to read fresh stops before its refresh).
- `_visible_agent_panel_indices`: skip panels whose key is collapsed so cross-panel visible-row consumers (e.g. unread
  navigation) don't treat hidden rows as visible.
- `_agents_visible_order`: return empty when the focused panel is collapsed (defensive; keeps any focused-order consumer
  consistent).

`src/sase/ace/tui/actions/agents/_panel_navigation.py`, `_change_focused_agent_panel`: verify the existing empty-stops
`else` branch already lands focus correctly on a collapsed target panel (it calls `_snap_current_idx_to_focused_panel`,
anchoring `current_idx` to the panel's first agent). Ensure `_current_group_key` is cleared on that path. No structural
change expected beyond confirming this behavior.

### 5. Rendering & layout — collapsed panels render border-only and drop their width

`src/sase/ace/tui/widgets/agent_list.py` (+ helpers in `_agent_list_build.py`): add `AgentList.render_collapsed(...)`
that:

- Sets the programmatic-update guard, clears options, resets the per-row bookkeeping maps (`_row_entries`,
  `_banner_at_row`, `_row_render_ctx`, `_row_tier_styles`, `_row_by_agent_*`, `_banner_row_by_key`), and sets
  `_agents = []` so highlight/patch paths safely no-op.
- Marks a new `self._panel_collapsed = True` flag (initialize to `False` in `__init__`; cleared by the normal
  `build_list` path).
- Sets `_requested_width` to just the current `border_title` cell width plus a small constant pad (border corners +
  comfort) and posts `WidthChanged(_requested_width)`. This guarantees the title always renders in full while
  contributing far less than the panel's hidden rows would. (The container handler already `max()`es requests and clamps
  to `[_MIN_AGENT_LIST_WIDTH, _MAX_AGENT_LIST_WIDTH]`, so no change is required in `on_agent_list_width_changed`.)

`src/sase/ace/tui/actions/agents/_display_panel_refresh.py`:

- `_refresh_panel_widgets_impl` and `_refresh_affected_panel_widgets`: for each panel, branch on
  `key in _collapsed_panel_keys`. Collapsed → set the border title with `collapsed=True` (still computing
  `counts`/`len(panel_agents)` for the summary), call `widget.render_collapsed(...)` instead of
  `widget.update_list(...)`, and add a `-collapsed-panel` CSS class. Expanded → remove `-collapsed-panel` and take the
  normal `update_list` path.
- `_apply_panel_heights`: make it collapse-aware. Derive a per-widget collapsed flag from `_panel_group.panel_keys` +
  `_collapsed_panel_keys` (guarding the `len(panel_keys) == len(widgets)` case it already checks). Give every collapsed
  panel a fixed border-only height and exclude it from: the "first panel gets `1fr`" filler choice (pick the first
  non-collapsed panel as the filler instead), the untagged half-budget special case, the overflow-regime fraction
  weights, and the min-height budget (treat its min/natural as the fixed border-only height). Freed height flows to the
  expanded panels.
- `_refresh_focused_agent_panel_impl`: when the focused panel is collapsed, skip `update_highlight` (nothing to
  highlight) and just apply the `-focused-panel` class + widget focus.
- `_sync_panel_group`: after recomputing `panel_keys`, prune `_collapsed_panel_keys` down to keys still present, so a
  tag whose last agent was dismissed drops its collapsed state.

### 6. Merge/split toggle

`src/sase/ace/tui/actions/agents/_panel_navigation.py`, `action_toggle_agent_panel_grouping`: clear
`_collapsed_panel_keys` on toggle (alongside the existing `_current_group_key` reset) so split-mode tag collapses never
leak into the single merged "All agents" panel.

### 7. Entry-jump ("f") hints

`src/sase/ace/tui/actions/navigation/_entry_jump_mode.py`, `_jump_candidate_targets`: skip panels whose key is collapsed
so jump-hint letters aren't consumed by rows that aren't rendered.

### 8. Optional visual polish (validate via snapshot)

`src/sase/ace/tui/styles.tcss`: optionally add a subtle `AgentList.-collapsed-panel` treatment (e.g. a dimmer
non-focused border) so collapsed panels read as closed while a focused collapsed panel still shows the gold
`-focused-panel` border. Guard specificity so `-focused-panel` wins for a focused collapsed panel. Keep changes minimal
and confirm against PNG snapshots; the chevron + reduced height are the primary affordance, so this is polish, not core.

## Documentation & keymap sync (required)

Per `src/sase/ace/CLAUDE.md` (Help Popup Maintenance / Footer conventions) and the `gotchas` memory (Default Keymap
Config), keep docs in lockstep with behavior:

- `src/sase/ace/tui/bindings.py`: update the `H` and `L` binding descriptions to reflect panel collapse/expand.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`: update the folding section entry that currently reads
  "Expand/collapse one level (all; Tools: full/compact)" to describe collapse/expand of the focused agent panel; note
  near the `J`/`K` "Cycle focus across tag panels" line that collapsed panels are selectable.
- `src/sase/ace/tui/commands/_app_metadata.py`: update the command-palette descriptions for `hooks_or_collapse_all` and
  `expand_all_folds` so the Agents-tab meaning (collapse/expand panel) is discoverable while remaining accurate for the
  other tabs.
- `src/sase/ace/tui/keymaps/types.py`: adjust the generic display labels for those two actions only if needed to avoid
  describing removed behavior; do not rename the action identifiers (they are shared across tabs).
- Verify `src/sase/default_config.yml` — the `H`/`L` key bindings themselves are unchanged (no new keys/actions), so no
  config change is expected; confirm and adjust only if a label/description there references the old behavior.

## Testing strategy

Unit tests (fast, `just test`):

- Rewrite the Agents-tab capital-`H`/`L` fold-stepping tests in `tests/ace/tui/test_agent_fold_transitions.py`: remove
  the ones asserting the old "peel every fold one level" behavior (e.g. `test_capital_l_expands_every_group`,
  `test_capital_h_collapses_deepest_visible_group_and_workflow`, `test_capital_l_peels_one_level_per_press`,
  `test_capital_h_collapses_only_deepest_visible_group_level`, `test_capital_h_collapses_next_deepest_group_level`,
  `test_capital_h_peels_one_level_per_press`, `test_capital_h_does_not_step_invisible_workflows`). Keep
  `test_tools_panel_routes_fold_keys_to_detail_level` (Tools routing still precedes the panel logic) and the
  lowercase-`h`/`l` in-panel group-fold tests (unchanged behavior).
- Add a new test module (e.g. `tests/ace/tui/test_agent_panel_collapse.py`) covering: `H` adds the focused key to
  `_collapsed_panel_keys` and `L` removes it; the guards (single panel, merged mode, already collapsed/expanded, Tools
  routing wins); pruning on panel disappearance; and clearing on grouping toggle.
- Navigation: `_panel_navigation_stops` returns empty for a collapsed focused panel; `J` onto a collapsed panel focuses
  it (border highlight) and `L` re-expands and re-anchors; lowercase `j`/`k` no-op on a collapsed focused panel;
  `_jump_candidate_targets` omits collapsed panels. Extend/reuse `tests/ace/tui/test_agent_jk_navigation.py` /
  `test_agent_panel_first_selection.py` patterns.
- Width: extend `tests/ace/tui/test_agent_left_panel_width.py` to assert a collapsed panel requests only ~title width
  and that collapsing the widest panel drops the container `max()` to the remaining expanded panels.
- Title: extend `tests/ace/tui/test_agent_panel_titles.py` to assert `agent_panel_border_title(..., collapsed=True)`
  prepends the chevron and preserves the plain-text count summary.
- Height: a focused unit test that `_apply_panel_heights` assigns a collapsed panel the fixed border-only height and
  does not give it the `1fr` filler even at index 0.

Visual snapshot (`just test-visual`, goldens under `tests/ace/tui/visual/snapshots/png/`):

- Add a PNG snapshot (e.g. in `test_ace_png_snapshots_agents_interactions.py`) with a multi-panel Agents fixture where
  one wide panel is collapsed, asserting: the collapsed panel shows its `▸ #tag · N […]` title as a border-only strip,
  and the agent-list column is visibly narrower than with the panel expanded. Accept the new golden with
  `--sase-update-visual-snapshots` only after eyeballing it. This is the primary "beautiful + reliable" gate.

## Edge cases & reliability

- Collapsing the focused panel keeps selection on it (gold border); `L` or `J`/`K` are the exits. `j`/`k` intentionally
  do nothing inside it.
- `J`/`K` across several collapsed panels lands on each panel's header in turn (empty stops → the existing snap-to-panel
  path), so the user is never stranded.
- Only ≥2 panels in split mode can collapse; merged mode and single-panel layouts no-op so the user can never hide
  everything with no obvious way back.
- Removing all agents in a collapsed panel prunes it (and its collapsed state) on the next `_sync_panel_group`;
  killing/dismissing an agent inside a collapsed panel updates the header count on refresh while focus stays on the
  panel.
- Patch/remove fast paths (`patch_agent_row`, `try_remove_rows`) hit a collapsed widget with empty row maps, so they
  cleanly fall back to a full refresh that re-renders it collapsed — no stale rows.

## Risks

- **Layout math.** `_apply_panel_heights` has several regimes; the collapse-aware changes must not regress
  expanded-panel sizing. Mitigate with the focused height unit test plus the PNG snapshot.
- **Border-only title clipping.** Confirm the chosen collapsed height renders the full title on the top border; bump the
  fixed height by one row if a snapshot shows clipping.
- **Nav-stop cache staleness.** If the collapse signal is omitted from the `_nav_stops_cache` signature, `L` can read
  stale (empty) stops. The signature change in §4 is required, not optional.
- **Doc drift.** The Agents help popup, command palette, and footer must match the new behavior (`ace/CLAUDE.md`
  mandates help-popup sync).

## Validation before hand-off

The implementing agent must run `just install` (ephemeral workspaces may have stale deps) then `just check` (ruff +
mypy + fast tests) and `just test-visual` for the PNG suite, and must not modify memory or provider-instruction files
without explicit user permission.
