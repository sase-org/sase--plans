---
tier: tale
title: Collapsed agent panel jump hints
goal: 'Apostrophe jump mode renders and dispatches one-key hints for collapsed Agents-tab
  panel headers while preserving hidden-row, navigation-history, selective-refresh,
  and layout behavior.

  '
create_time: 2026-07-16 16:06:57
status: wip
prompt: 202607/prompts/collapsed_agent_panel_jump_hints.md
---

# Plan: Add apostrophe jump hints to collapsed agent panels

## Product context

The Agents tab can now collapse an entire tag panel into a title-only strip such as `▸ #chop · 2 [R1 W1]`. The
apostrophe jump mode already assigns top-to-bottom `[x]` hints to visible agent rows and collapsed _in-panel group_
banners, but `_jump_candidate_targets()` deliberately skips whole collapsed panels. That makes the compact panel header
visible and focusable with `J` / `K`, yet unreachable via the tab's one-key entry-jump workflow.

Extend the existing jump system so a collapsed whole-panel header is a first-class target. Pressing `'` should paint a
yellow `[x]` prefix directly in the collapsed border title, and pressing that character should focus the panel header
without exposing or acknowledging any hidden agent row. This is presentation and navigation state in the Python/Textual
frontend; it does not add shared domain behavior, so no change to the Rust core backend is required.

## Target behavior

- On the split Agents tab, apostrophe mode allocates hints in exact visual order across all selectable items: visible
  agent rows, collapsed in-panel group banners, and collapsed whole-panel headers. Because collapsed panels are
  currently stably rendered after expanded panels, their header hints naturally follow every visible target above them.
- A collapsed panel contributes one target for its header and none for its hidden rows or nested banners. An expanded
  panel's border title is not selectable and does not receive a hint.
- A hinted collapsed title renders as `[x] ▸ #tag · N [metrics]`, reusing the bold yellow hint style used by agent rows
  and group banners. The existing chevron, title styles, counts, focus border, and title-only height remain intact.
- Selecting a panel-header hint focuses that collapsed panel, clears any in-panel banner/attempt selection, and anchors
  `current_idx` to the panel's first backing rendered agent so the detail pane retains panel-local context. The hidden
  backing agent is not treated as the selected jump target and is not auto-acknowledged as read.
- Leaving a visible manually-unread agent for a collapsed header preserves the established departure semantics: arm that
  source row as appropriate, but do not acknowledge any agent inside the destination panel.
- Normal apostrophe history remains coherent. A collapsed panel header is recorded as an explicit panel anchor, so
  back-jump and forward-jump restore header focus; an anchor is discarded if its panel disappeared or is no longer
  collapsed. The no-history `''` fast path may select a panel header when it owns hint `1`, but it must not paint hints
  or update the jump footer.
- Escape, tab changes, completed selection, and other existing jump-mode exits remove the header hint and restore the
  collapsed title's normal width. The artifact-viewer navigation guard applies before both allocation and dispatch,
  exactly as it does for agent and banner targets.
- ChangeSpecs, AXE, merged Agents mode, `J` / `K`, `j` / `k`, `H` / `L`, persisted collapse state, and the underlying
  in-panel fold registries retain their current behavior.

## Design

### 1. Model a whole-panel target and anchor explicitly

Extend `src/sase/ace/tui/actions/navigation/jump_hints.py` with a discriminated whole-panel target such as
`("panel", panel_key)`, and include it in the Agents-tab jump-target and anchor unions. Use the unique `PanelKey`
(`str | None`) rather than a rendered index for this new target: the key is already the durable identity of a panel and
remains correct if the expanded/collapsed stable partition is rebuilt. Existing group-banner targets can keep their
current panel-index scope.

Add dedicated hint-to-panel and panel-to-hint maps alongside the existing agent and group-banner maps in:

- `src/sase/ace/tui/actions/_state_init.py`
- `src/sase/ace/tui/actions/navigation/_types.py`
- the tab-change and jump-mode teardown paths in `src/sase/ace/tui/app.py` and
  `src/sase/ace/tui/actions/navigation/_entry_jump_mode.py`

Keeping panel maps separate avoids weakening the banner-map types or feeding two-element panel targets into renderer
code that destructures three-element group banner targets.

### 2. Allocate the header in render order

Update `_jump_candidate_targets()` in `src/sase/ace/tui/actions/navigation/_entry_jump_mode.py` to emit exactly one
panel target when a key is in `_collapsed_panel_keys`, then skip that panel's tree walk. Expanded panels continue
through the existing `build_agent_tree()` walk, preserving the established ordering of visible agent and
collapsed-banner hints. Split the resulting generic hint map into agent, banner, and panel maps in
`_prepare_agents_jump_maps()`.

Do not add filesystem work, widget queries, or a new refresh path to target allocation. It should remain a read-only
walk over cached/in-memory panel and tree state.

### 3. Dispatch and preserve navigation history

Update `src/sase/ace/tui/actions/navigation/_entry_jump_dispatch.py` so panel targets participate in the same validity,
artifact-viewer guard, source-departure, history, and exit flow as the existing targets. Resolve the target's panel key
against the current `AgentPanelGroup`, focus that panel, clear `_current_group_key` and `current_attempt_number`, then
use the established panel-snap helper to anchor `current_idx` without acknowledging the hidden destination agent.

Teach `src/sase/ace/tui/actions/navigation/_entry_jump_agents.py` to:

- capture an explicit panel anchor whenever the currently focused panel is collapsed, before falling back to the current
  agent/banner forms;
- validate a panel anchor by stable key and require that the panel is still present and collapsed; and
- restore it by resolving the current rendered index, focusing the panel, clearing row/banner selection, and
  re-anchoring the backing detail context.

This keeps direct hint jumps, apostrophe back-jumps, and forward history symmetric, and prevents a formerly collapsed
header from silently restoring as an agent row after expansion.

### 4. Render the hint in the collapsed border title

Add an optional hint character to `agent_panel_border_title()` in
`src/sase/ace/tui/actions/agents/_display_panel_titles.py`. Prefix `[x] ` with the same `bold #FFFF00` style used by
agent rows and group banners, then render the existing collapsed chevron and summary unchanged. Callers that do not pass
a hint must produce byte-for-byte equivalent plain text and spans.

Thread the active inverse panel-hint map through the list-changed display path in
`src/sase/ace/tui/actions/agents/_display.py` and through both full and affected-panel refreshes in
`src/sase/ace/tui/actions/agents/_display_panel_refresh.py`. Only a collapsed panel looks up and supplies its own
`("panel", panel_key)` hint. The existing `_refresh_agents_jump_hint_display()` selective path should repaint all
mounted panels on mode entry/exit; do not introduce a full-list refresh or a parallel rendering implementation.

`AgentList.render_collapsed()` already computes its requested width from the current Rich `border_title`. Reusing it
after the hinted title is installed should temporarily add the four hint cells without clipping; exiting jump mode
should recompute the smaller normal request. Keep this width update and the fixed border-only height behavior covered
rather than special-casing layout math.

Update the relevant attribute declarations in `src/sase/ace/tui/actions/agents/_display.py` so static checking reflects
the new map. No key identifier, default binding, help text, command description, or `src/sase/default_config.yml` entry
changes: apostrophe remains “Jump to Entry”; the set of eligible entries is simply becoming complete.

## Testing strategy

Extend the focused jump tests, primarily `tests/ace/tui/test_jump_hints_for_folded_banners.py` and
`tests/ace/tui/test_agent_panel_collapse.py`, to cover:

- target order with expanded rows, collapsed group banners, and one or multiple collapsed whole panels; each collapsed
  panel consumes one hint while all of its hidden rows remain absent;
- stable panel-key forward/inverse maps, including the untagged `None` panel;
- dispatch to a non-focused collapsed panel, correct focused key and backing detail anchor, cleared group/attempt state,
  source unread-departure handling, and no destination unread acknowledgement;
- explicit panel anchors through back and forward history, no-history apostrophe and fast-jump behavior, plus stale
  anchors after expansion or panel removal;
- invalid hint and artifact-viewer guard behavior without focus mutation; and
- mode exit clearing the new maps as well as the existing maps.

Extend `tests/ace/tui/test_agent_panel_titles.py` and `tests/ace/tui/test_agent_panels_display.py` to assert the exact
hinted title text and yellow span, correct hint routing by panel key in full and affected refreshes, and removal on
exit. Include a width assertion (in the display tests or `tests/ace/tui/test_agent_left_panel_width.py`) showing that
the title request grows by the hint prefix while active and shrinks again without restoring hidden-row width.

Add a visual state to `tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py`: reuse the multi-panel
collapse fixture, enter apostrophe mode after `#chop` collapses, and assert the screen-order hint (for the current
fixture, `[3]`) appears in the focused `[3] ▸ #chop · 2 [R1 W1]` border title. Preserve the existing non-hint collapse
snapshot and add a dedicated jump-hint golden so both resting and transient states remain covered. Inspect generated
actual/expected/diff artifacts before accepting the new PNG.

## Risks and safeguards

- **Target/anchor ambiguity:** without a distinct panel target, existing code would misclassify the header as a group
  banner or hidden agent. Separate discriminants, maps, and stable-key validation keep the three selectable forms
  explicit.
- **Panel reorder and refresh races:** collapsed panels are repartitioned after refresh. Carry panel keys in new
  targets/anchors and resolve the current index only at dispatch/restore time; discard missing or expanded targets
  instead of focusing an index that now belongs to another panel.
- **Unread side effects:** the backing `current_idx` exists only to retain detail context. Test that header selection
  follows banner-like departure behavior and never calls destination acknowledgement.
- **Transient clipping or layout churn:** the hint makes the border title four cells wider. Let the existing collapsed
  render request that width on the selective repaint, and verify entry and exit visually and in unit tests.
- **Performance:** remain on `_refresh_affected_panel_widgets()` for apostrophe entry/exit, perform no I/O in allocation
  or render paths, and preserve the current display-patch trace behavior. This follows the audited TUI performance
  guidance and avoids a data-scaled full list rebuild on each mode transition.

## Validation and hand-off

The implementing agent must run `just install` first in its ephemeral workspace, then run focused unit tests while
iterating, `just test-visual` for the dedicated PNG suite, and `just check` before hand-off. Update PNG goldens only for
the intentional new jump-mode snapshot and inspect `.pytest_cache/sase-visual/` artifacts on any failure. Do not edit
memory files or generated provider instruction shims.
