---
tier: tale
title: Keep the sticky agent header visible in the File/LLM Calls-only layout
goal:
  The Agents-tab sticky identity header panel stays visible above the detail column in
  every layout, including the File-only and LLM Calls-only layouts, and `d` toggles it
  there.
size: small
proposed_by: bbugyi200.athena.0pp
create_time: 2026-09-22 18:40:25
status: wip
---

# Plan: Keep the sticky agent header visible in the File/LLM Calls-only layout

## Context

Epic `sase-16k` added the sticky, collapsible identity header panel (`AgentHeaderPanel`,
`#agent-header-panel`) at the top of the Agents-tab detail column. By design it was
shown only while the metadata panel is visible. It is hidden in the
`DetailLayoutMode.SECONDARY_ONLY` layout (Agent view picker `p` then `]`: "File only" /
"LLM Calls only"), so the metadata panel "takes back the space". The user now wants the
header **always visible** whenever the selected node has an identity header. That
includes the layout where only the LLM Calls (tools) panel is shown. For consistency,
the file-only variant of the same layout gets it too: it is one layout mode, and "always
visible" should not depend on which secondary panel is selected.

This work is presentation-only Textual code and stays in this repo; `sase-core` is not
touched. No feature flag is needed. The change extends a feature that shipped today; it
does not deprecate a path that callers must keep reaching.

### How it works today

- `AgentDetail.compose()` (`src/sase/ace/tui/widgets/agent_detail.py`) yields, inside
  `#agent-detail-layout` (a `Vertical`): `#agent-header-panel` (first), then
  `#agent-prompt-scroll`, `#agent-search-scroll`, `#agent-search-command`,
  `#agent-file-scroll`, and `#agent-llm-calls-scroll`. The header is already above every
  secondary scroll in DOM order, so no compose change is needed.
- `AgentDetail._sync_header_visibility()` shows the panel only when
  `self.is_metadata_visible() and panel.has_identity`. That condition is the only thing
  that hides it in `SECONDARY_ONLY`. `AgentDetailPanelMixin._hide_metadata_scrolls()`
  and `_show_active_metadata_scroll()`
  (`src/sase/ace/tui/widgets/_agent_detail_panels.py`) and every publish path already
  call the sync, so visibility tracks layout changes.
- The prompt panel keeps rendering its document while it is hidden.
  `_update_display_impl` always calls `prompt_panel.update_display(agent)` or the
  workflow worker. So the identity sink (`_on_identity_header`) keeps publishing the
  current node's `IdentityHeader` in `SECONDARY_ONLY`: `has_identity` is already correct
  there. `show_empty()` and clan documents publish `None`, which keeps the panel hidden
  for "No agent selected" and clan rows.
- `header_toggle_available()` returns `has_identity and not hidden`, and the `d` action,
  the palette availability (`commands/context.py` `_header_toggle_available`), and
  `check_app_action` all flow through it. Once the panel is visible in `SECONDARY_ONLY`,
  `d` works there too with no keymap change.
- CSS (`src/sase/ace/tui/styles.tcss`): `#agent-header-panel` is
  `height: auto; max-height: 50%`. `#agent-prompt-scroll.expanded` and
  `#agent-search-scroll.expanded` were already moved to `height: 1fr` by `sase-16k`, so
  they share the column with the header. But
  `#agent-file-scroll.expanded, #agent-llm-calls-scroll.expanded` are still
  `height: 100%`, and `.expanded` is applied to a secondary scroll only in
  `SECONDARY_ONLY`. With the header visible, a 100%-height secondary scroll would
  overflow the column and clip its bottom border and last lines.

## Changes

1. **Visibility rule** (`src/sase/ace/tui/widgets/agent_detail.py`,
   `_sync_header_visibility`): drop the `is_metadata_visible()` term. The panel is
   visible iff `panel.has_identity`. Update the docstring (for example "Hide the header
   unless the current document published an identity") and keep the defensive
   `try/except` shape. Leave all existing call sites in place. They still matter because
   the identity can change (publish, clan, empty), and keeping them is cheap and
   idempotent. Do not change `is_metadata_visible()`. `,/` metadata search
   (`actions/agents/_metadata_search.py`) and the view-picker tests rely on its current
   meaning.
2. **CSS** (`src/sase/ace/tui/styles.tcss`): change
   `#agent-file-scroll.expanded, #agent-llm-calls-scroll.expanded` from `height: 100%`
   to `height: 1fr`, matching the prompt and search `.expanded` rules. The secondary
   panel then fills whatever the auto-height header leaves, collapsed (2 content rows
   plus border) or expanded (capped at 50%). Leave the `.layout-*` split rules alone.
   They are already fr-based.
3. **Mixin stub docs**: in `_agent_detail_panels.py`, the `DetailLayoutMode` comment
   `SECONDARY_ONLY = "secondary_only"  # Metadata 0% / File/LLM Calls 100%` can stay.
   Optionally append "(below the sticky header)" if that reads naturally. No logic
   changes in the mixin.
4. **Docs** (`docs/ace.md`):
   - In the "Agents Tab Metadata Panel" → **Header panel** bullet, replace "whenever the
     metadata panel is shown" with wording saying it stays at the top of the detail
     column in every layout, including the file-only and LLM Calls-only layouts, where
     it sits above the secondary panel. Remove "and the file/LLM Calls-only layout" from
     the hidden-for list (it stays hidden for clan rows and "No agent selected").
   - In the `d` row of the "Keybindings: Agents Tab" table ("sticky identity header
     above the metadata panel"), say "above the detail panels" (or similar) so the text
     is not metadata-specific. Keep the table column widths aligned (the repo formats
     Markdown tables).
   - In the Agent view picker paragraph ("`]` shows only the selected File or LLM Calls
     panel"), add that the sticky header panel stays above it.
   - `docs/configuration.md` ("available only on the Agents tab while the header panel
     is shown") stays accurate; leave it.
5. **Tests**:
   - `tests/ace/tui/widgets/test_agent_header_panel.py`: replace
     `test_secondary_only_hides_header_and_split_keeps_it` with a test that asserts the
     header stays visible and toggleable in `SECONDARY_ONLY` for **both** secondary
     modes:
     - File: set `detail._has_file_content = True`, then
       `set_detail_layout(DetailLayoutMode.SECONDARY_ONLY)`.
     - LLM Calls: set `detail._panel_mode = DetailPanelMode.LLM_CALLS` and
       `detail._has_llm_calls_content = True`, then apply the layout. Use the existing
       private-attribute style with `# noqa: SLF001`.

     In each case assert `not panel.has_class("hidden")`,
     `detail.header_toggle_available() is True`, and
     `detail.is_metadata_visible() is False`. After `await pilot.pause()`, also assert
     that the visible secondary scroll sits below the header and inside the detail
     column: `secondary.region.y >= panel.region.bottom` and
     `secondary.region.bottom <= detail.region.bottom`. That guards the CSS `1fr`
     change. Then call `detail.toggle_header_expanded()`, pause, and assert the header
     is expanded (`"Name:"` in `_header_text(panel)`) while the secondary scroll still
     fits inside the column. Keep the existing split-layout (`METADATA_LARGER`)
     assertion.

   - Keep `test_clan_selection_hides_header` and `test_empty_state_hides_header`
     unchanged. They must still pass: hidden when there is no identity. If cheap, add a
     one-line check that `show_empty()` while in `SECONDARY_ONLY` still hides the
     header.
   - `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_layout.py`
     `test_agents_file_only_layout_png_snapshot`: extend the final `wait_for_state` so
     it also waits until `#agent-header-panel` has no `hidden` class. The golden must
     show the header above the full-height file panel. The fixture file text ("The
     metadata panel should not be visible in this snapshot.") stays true.
   - Search `tests/` for any other assertion that the header or the metadata panel is
     hidden in `SECONDARY_ONLY`, and adjust only the header-related ones. The
     `test_agent_view_picker.py` `is_metadata_visible()` assertions stay as-is.

## Visual verification

Read `sase/memory/tui_screenshot.md` (via `sase memory read`) before capturing.

- Capture a live screenshot with a `--keep` session in each layout on an agent that has
  LLM Calls: `p` `t` then `]` (LLM Calls only), and `p` `f` then `]` (file only). Toggle
  `d` in each. Inspect the PNGs: the header's border and title are intact above the
  secondary panel, the secondary panel's bottom border is not clipped, and the expanded
  header caps at half the column and scrolls rather than pushing the secondary panel
  off-screen.
- Run the full `just fix-tui-screenshots` through `/sase_monitor`. Expect
  `agents_file_only_layout_120x40` to change (the header now appears above the file
  panel). Inspect every created, removed, or updated golden. Any change outside
  `SECONDARY_ONLY` layouts means the CSS change leaked, and it must be fixed rather than
  accepted. In the other layouts the secondary scrolls never carry `.expanded`, so they
  should be byte-identical.

## Verification

- `sase tool run check` (the `just check` recipe) must pass.
- The targeted tests pass: `tests/ace/tui/widgets/test_agent_header_panel.py`,
  `tests/ace/tui/test_agent_view_picker.py`,
  `tests/ace/tui/modals/test_agent_view_modal.py`,
  `tests/ace/tui/test_agents_zoom_panel_action.py`,
  `tests/test_command_availability_agents_actions.py`, and the updated PNG snapshot
  test.
