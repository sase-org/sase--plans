---
tier: tale
title: Scroll expanded Agents sticky header with Ctrl+D/U
goal:
  Ctrl+D/U scrolls the visible expanded Agents header when it overflows, while existing
  deck and other-tab scrolling remains intact otherwise.
size: medium
proposed_by: bbugyi200.apollo.1r
create_time: 2026-09-25 19:49:39
status: wip
---

# Route Agents half-page scrolling to an overflowing expanded header

## Goal

On the Agents tab, the configured `scroll_detail_down` / `scroll_detail_up` keys (Ctrl+D
/ Ctrl+U by default) scroll the sticky agent identity header whenever that header is
visibly expanded and has vertical overflow. This takes priority over the focused deck
even when the header is already at its top or bottom. When the header is hidden,
collapsed, or fits without scrolling, keep the current focused-deck behavior. Preserve
the existing Artifacts and Services behavior.

## Current behavior and relevant code

- `AgentHeaderPanel` in `src/sase/ace/tui/widgets/agent_header_panel.py` is a
  `VerticalScroll`. `AgentDetail` mounts it above `DeckArea`, and
  `src/sase/ace/tui/styles.tcss` caps its height at 50% of the detail column. It
  currently receives no Ctrl+D/U action.
- `AgentHeaderPanel.is_expanded` reports the toggle state, but `show_identity()` also
  paints the expanded content when the identity has hints. The routing condition must
  reflect what is actually shown, including this forced-expanded case.
- `BasicNavigationMixin.action_scroll_detail_down/up` in
  `src/sase/ace/tui/actions/navigation/_basic.py` always uses
  `AgentDetail.effective_detail_scroll_id()` on Agents, which resolves to the focused
  deck's active scroll; those branches also release the Main deck bottom pin.
- Committed inline metadata search intercepts configured scroll keys in
  `src/sase/ace/tui/actions/agents/_metadata_search.py` and scrolls its deck overlay.
  Typing search consumes keys before app bindings. Both paths need header priority while
  an overflowing expanded header is visible.

## Implementation

1. Add a small, read-only eligibility check and half-page scrolling method at the
   `AgentDetail` / `AgentHeaderPanel` presentation boundary. Check a mounted, visible
   identity header; the effective displayed expansion state (`_expanded` or the
   identity's `has_hints`); and live Textual layout overflow (`max_scroll_y > 0`), not
   the estimated `rendered_row_count`. Scroll by half the header's _own_ visible content
   height with `animate=False`, using at least one row for a tiny viewport. Return
   whether the header claimed the key. Return true at either scroll boundary so the deck
   never moves on a second keystroke. Keep the check inexpensive and free of rendering
   or I/O.
2. In the Agents branches of `action_scroll_detail_down/up`, try that header method
   before choosing the focused deck. If it claims the key, return without releasing the
   deck's bottom pin. Otherwise preserve the existing deck target, half-page amount, and
   pin behavior. Leave other tabs and Ctrl+F/B or g/G routing as they are.
3. Give the same header priority to the registry-backed `scroll_detail_down/up` keys in
   inline metadata search, before its committed overlay scroll and before typing search
   consumes the key. If the header is ineligible, preserve each search mode's current
   key behavior. Do not change the keymap names or defaults in `bindings.py`,
   `app_keymaps.py`, or `default_config.yml`.

## Verification

- Add mounted Textual/Pilot regression coverage near
  `tests/ace/tui/widgets/test_agent_header_panel.py` for a genuinely overflowing
  expanded header (`max_scroll_y > 0`): Ctrl+D moves its offset down and Ctrl+U moves it
  back while the focused deck offset and its Main bottom pin remain unchanged. Include
  the top/bottom boundary behavior, a header shown expanded because it has hints, and a
  non-Main focused deck or split layout.
- Cover the fallback when the header is collapsed, expanded but fits
  (`max_scroll_y == 0`), or hidden: Ctrl+D/U still targets the focused deck. Confirm a
  short viewport cannot produce a zero-row header step.
- Extend `tests/ace/tui/test_agent_metadata_search.py` for the committed and typing
  search key paths: overflowing expanded header takes priority; when the header cannot
  scroll, existing overlay/typing handling remains. Use configured keymap values in the
  test rather than depending only on hard-coded defaults where practical.
- Run focused tests for the changed behavior, then `sase tool run check` (the
  repository's default whole-repo lint and diff-scoped test gate). Do not run
  `check-full` without an explicit instruction.

## Completion criteria

Ctrl+D/U consistently scrolls the visible expanded header on Agents only when its
laid-out content overflows, regardless of focused deck or search mode; at its limits the
deck stays still. All other cases retain their present target and behavior, and the
focused tests plus `sase tool run check` pass.
