---
tier: tale
title: Ctrl+D/U scrolls the expanded Agents jump panel (sticky footer) first
goal:
  On the Agents tab, Ctrl+D/U half-page scroll the sticky jump panel whenever it is
  expanded and overflowing, taking priority over an expanded overflowing sticky header,
  and otherwise keep today's header-then-focused-deck routing.
size: small
proposed_by: bbugyi200.athena.0tu
create_time: 2026-09-29 07:02:02
status: wip
---

# Plan: Ctrl+D/U scrolls the expanded, overflowing Agents jump panel (sticky footer) first

## Goal

On the Agents tab, the configured `scroll_detail_down` / `scroll_detail_up` keys
(default `ctrl+d` / `ctrl+u`) already scroll the sticky **header** panel
(`AgentHeaderPanel`, `#agent-header-panel`) instead of the focused deck panel when the
header is expanded and its content overflows. Extend the same behavior to the sticky
**footer** — the jump panel (`AgentJumpPanel`, `#agent-jump-panel`, toggled with `.`) —
and give the footer priority over the header.

Routing for one Ctrl+D/U keypress on the Agents tab becomes:

1. **Footer** — if the jump panel is eligible (see below), scroll it by half its visible
   height and stop. It claims the key even when it is already at its top/bottom
   boundary; it never falls through to the header or deck in that case (same boundary
   semantics the header already has).
2. **Header** — otherwise, if the header is eligible (existing
   `AgentHeaderPanel.is_header_scrollable()` rules, unchanged), scroll it and stop.
3. **Focused deck** — otherwise, the existing focused-deck half-page scroll, including
   `_release_focused_deck_bottom_pin()`.

A sticky-panel claim must not scroll the deck and must not release the focused deck's
bottom pin (the header path already returns before the pin release; keep that).

### Footer eligibility

`AgentJumpPanel` is eligible only when **all** of the following hold:

- it is mounted, does not have the `hidden` class, and `display` is truthy;
- it has numbered targets (`has_targets`);
- it is expanded (`self._expanded` is True) **and** not narrowed to a pending two-digit
  prefix (`self._pending_prefix is None`) — the narrowed candidate view is transient,
  not the expanded view;
- `max_scroll_y > 0` (content actually overflows the `max-height: 40%` box).

A collapsed jump panel never claims, even if its collapsed legend overflows on a tiny
terminal (mirrors the header, whose collapsed preview never claims).

Note (no action needed): while a jump digit is pending, the app's `on_key` already
cancels the prefix for any non-digit key before the binding runs
(`_handle_member_jump_key` in `src/sase/ace/tui/actions/navigation/_member_jump.py`).
That keypress then routes by the panel's already-laid-out size; do not add special
handling for re-layout timing.

## Current code (for orientation)

- `src/sase/ace/tui/widgets/agent_header_panel.py` —
  `AgentHeaderPanel.is_header_scrollable()` and
  `AgentHeaderPanel.scroll_header_half_page(direction)` implement header eligibility and
  the half-page claim (`step = max(1, scrollable_content_region.height // 2)`,
  `scroll_relative(y=±step, animate=False)`, return True when eligible even at a
  boundary).
- `src/sase/ace/tui/widgets/agent_jump_panel.py` — `AgentJumpPanel(VerticalScroll)` with
  `_expanded`, `_pending_prefix`, `has_targets`, `is_expanded`; no scroll-claim API yet.
- `src/sase/ace/tui/widgets/agent_detail.py` —
  `AgentDetail.try_scroll_expanded_header(direction)` delegates to the header panel.
- `src/sase/ace/tui/widgets/_agent_detail_jump.py` — `AgentDetailJumpMixin` (mixed into
  `AgentDetail`) owns jump-panel wiring: `_jump_panel_or_none()`,
  `toggle_jump_panel_expanded()`, `_reset_jump_panel_scroll()`, etc.
- Two call sites route Ctrl+D/U to the header today; both must switch to the combined
  footer-then-header routing:
  - `src/sase/ace/tui/actions/navigation/_basic.py` — `_try_scroll_agents_header()` used
    by `action_scroll_detail_down` / `action_scroll_detail_up` on the `agents` tab (also
    reached from `widgets/hint_input_bar.py`, which calls those app actions).
  - `src/sase/ace/tui/actions/agents/_metadata_search.py` —
    `_try_scroll_expanded_header_for_key(key)` called first in
    `_handle_agent_metadata_search_key` (inline `/` metadata search intercepts keys in
    `on_key` before bindings run).
- Layout: `#agent-jump-panel` in `src/sase/ace/tui/styles.tcss` is already
  `height: auto; max-height: 40%` with a stable scrollbar gutter, so no CSS change is
  needed.

## Implementation

### 1. Shared sticky-panel scroll helper (new module)

Create `src/sase/ace/tui/widgets/_sticky_panel_scroll.py` with two small public
functions so both sticky panels share one implementation of the generic checks and the
half-page claim:

- `sticky_panel_can_scroll(panel: Any) -> bool` — the generic part of eligibility:
  mounted, not `hidden` class, `display` truthy, `int(panel.max_scroll_y) > 0`, with the
  same defensive `try/except` style the header uses today (any exception → False).
- `scroll_sticky_panel_half_page(panel: Any, direction: int) -> bool` — computes
  `step = max(1, int(panel.scrollable_content_region.height) // 2)` (height falls back
  to 0 on error, so step is 1), calls
  `panel.scroll_relative(y=step if direction >= 0 else -step, animate=False)`, returns
  True, or False if `scroll_relative` raises. It does **not** check eligibility itself;
  callers gate it.

Refactor `AgentHeaderPanel` to use these helpers while keeping its public method names
and behavior unchanged (existing tests call `is_header_scrollable()` and
`scroll_header_half_page()` directly):

- `is_header_scrollable()` = header-specific checks (identity present; shown expanded =
  `self._expanded or identity.has_hints`) combined with `sticky_panel_can_scroll(self)`.
- `scroll_header_half_page(direction)` =
  `if not self.is_header_scrollable(): return False`, then
  `return scroll_sticky_panel_half_page(self, direction)`.

Keep the check order cheap-first but behavior-identical to today.

### 2. `AgentJumpPanel` scroll-claim API

In `src/sase/ace/tui/widgets/agent_jump_panel.py` add, mirroring the header:

- `is_jump_panel_scrollable(self) -> bool` — "Return whether Ctrl+D/U should scroll this
  jump panel instead of the header or a deck." True only when `has_targets`,
  `self._expanded`, `self._pending_prefix is None`, and `sticky_panel_can_scroll(self)`.
- `scroll_jump_panel_half_page(self, direction: int) -> bool` — returns False when not
  eligible; otherwise delegates to `scroll_sticky_panel_half_page(self, direction)`.
  Docstring should state that it claims the key at the boundaries too.

### 3. `AgentDetail` routing

- In `src/sase/ace/tui/widgets/_agent_detail_jump.py` (`AgentDetailJumpMixin`) add
  `try_scroll_expanded_jump_panel(self, direction: int) -> bool` using
  `_jump_panel_or_none()` and `scroll_jump_panel_half_page`, normalizing direction to
  `1 if direction >= 0 else -1` and swallowing exceptions → False (same shape as
  `try_scroll_expanded_header`).
- In `src/sase/ace/tui/widgets/agent_detail.py` add
  `try_scroll_expanded_sticky_panel(self, direction: int) -> bool` that returns
  `self.try_scroll_expanded_jump_panel(direction) or self.try_scroll_expanded_header(direction)`.
  Docstring: the jump panel (footer) wins over the header when both are eligible. Keep
  `try_scroll_expanded_header` as is (it is the header leg and existing tests call it).

### 4. Call sites

- `src/sase/ace/tui/actions/navigation/_basic.py`: rename `_try_scroll_agents_header` to
  `_try_scroll_agents_sticky_panel` (docstring: "Scroll an overflowing expanded Agents
  jump panel or header; True when claimed.") and have it call
  `try_scroll_expanded_sticky_panel` via the same `getattr` pattern. Update both uses in
  `action_scroll_detail_down` / `action_scroll_detail_up`. The early `return` must stay
  before `_release_focused_deck_bottom_pin()`.
- `src/sase/ace/tui/actions/agents/_metadata_search.py`: rename
  `_try_scroll_expanded_header_for_key` to `_try_scroll_expanded_sticky_panel_for_key`
  and call `try_scroll_expanded_sticky_panel` instead of `try_scroll_expanded_header`.
  Update its single caller in `_handle_agent_metadata_search_key`.
- `tests/ace/tui/test_agent_metadata_search.py`: the `_SearchScrollHost` test double
  defines a `_try_scroll_expanded_header_for_key` delegate to the mixin; rename it to
  `_try_scroll_expanded_sticky_panel_for_key` (delegating to the renamed mixin method)
  so the existing committed/typing search header-priority tests keep passing unchanged
  otherwise.
- `grep -rn "_try_scroll_agents_header\|_try_scroll_expanded_header_for_key"` across
  `src/` and `tests/` afterwards must return nothing.

### 5. Docs

`docs/ace.md`, Agents keybinding table row currently reading
`` `Ctrl+D` / `Ctrl+U` | Scroll focused deck panel down / up (half page) ``: change the
description to say the keys scroll the expanded jump panel when its content overflows,
else the expanded header panel when its content overflows, else the focused deck panel
(half page). Keep the table aligned (`just fmt` reformats Markdown). Leave the in-TUI
help modal text (`modals/help_modal/agents_bindings.py`) unchanged — it is width-bound
and did not mention the header case either. Do not edit `CHANGELOG.md` (release-please
owns it). No keymap/config change: no new keys or actions, so
`src/sase/default_config.yml` is untouched.

## Tests

Add a new focused module `tests/ace/tui/widgets/test_agent_jump_panel_scroll.py` (keep
it under ~500 lines). Model it on
`tests/ace/tui/widgets/test_agent_header_panel_scroll.py`: a
`_FooterScrollNavApp(BasicNavigationMixin, App[None])` with `current_tab = "agents"`
composing `AgentDetail(id="agent-detail-panel")`, driving
`app.action_scroll_detail_down()` / `app.action_scroll_detail_up()`. Reuse helpers from
`tests/ace/tui/widgets/_agent_jump_panel_helpers.py` (`_labeled_map`,
`_labeled_map_and_roster`, `_solo`, `_show_agent`, `_jump_panel`) and
`tests/ace/tui/widgets/_agent_header_panel_shared.py` (`LONG_XPROMPT`, `artifact_agent`,
`header_panel`, `show_agent_full`). Inject a jump map with
`detail._on_member_jump_map(jump_map, roster)  # noqa: SLF001` as the existing
jump-panel tests do, then `detail.toggle_jump_panel_expanded()` and
`await pilot.pause()`. Always assert the preconditions (`is_jump_panel_scrollable()`,
`max_scroll_y > 0`, and for priority tests `panel.is_header_scrollable()`) before
exercising keys.

Verified sizing facts from a prototype (use them to pick fixtures):

- At `size=(80, 24)`, an artifact agent with `LONG_XPROMPT`, header expanded, plus a
  20-entry `_labeled_map_and_roster(agent, [f"target-{i:02d}" for i in range(20)])`
  expanded footer ⇒ **both** overflow (header `max_scroll_y` ≈ 13, footer ≈ 18).
- At `size=(80, 24)`, a 2-entry (`["aa", "bb"]`) expanded roster does **not** overflow
  (`max_scroll_y == 0`).
- At `size=(80, 12)`, a **collapsed** 12-target `_labeled_map(...)` (no roster)
  overflows (`max_scroll_y == 1`) — use it for the "collapsed never claims" case.

Cases to cover:

1. **Expanded overflowing footer claims half-page scroll.** Pin the focused deck's main
   view to bottom (`detail.deck_area.panel(0).main_view.pin_to_bottom()` and
   `wait_for(pilot, lambda: main_view.is_bottom_pin_settled)` as the header test does),
   wrap the focused deck scroll's `scroll_relative` to record calls; Ctrl+D moves the
   footer by `max(1, footer_height // 2)`, Ctrl+U returns it to 0; deck `scroll_y`
   unchanged, no deck `scroll_relative` calls, bottom pin retained.
2. **Boundary claims without moving the deck.** Footer at `max_scroll_y` + Ctrl+D stays
   put and deck does not move; footer at 0 + Ctrl+U likewise.
3. **Fallbacks go to the focused deck** (header not overflowing in these cases, e.g.
   `_solo()` agent): collapsed-but-overflowing footer (80×12 case above) does not claim
   and the deck receives exactly one `scroll_relative`; expanded-but-fitting footer (2
   targets at 80×24) returns False from `detail.try_scroll_expanded_jump_panel(1)`;
   hidden footer (`detail.show_empty()` or a `None` jump map) is not scrollable; a
   narrowed footer (`panel.set_pending_prefix("1")` while expanded and overflowing)
   reports `is_jump_panel_scrollable() is False`.
4. **Footer wins over header.** With both expanded and overflowing (80×24 fixture
   above), Ctrl+D scrolls the footer and leaves the header `scroll_y` and deck
   untouched; Ctrl+U likewise. Then collapse the footer with
   `detail.toggle_jump_panel_expanded()` and verify Ctrl+D now scrolls the header
   (header `scroll_y` advances by `max(1, header_height // 2)`).
5. **Footer at a boundary still wins.** Both eligible, footer scrolled to
   `max_scroll_y`: Ctrl+D leaves footer at max and the header does not move.
6. **Header still claims when the footer is expanded but fits.** Header expanded and
   overflowing plus a 2-target expanded footer: Ctrl+D scrolls the header.
7. **Metadata-search routing.** Call
   `AgentMetadataSearchMixin._try_scroll_expanded_sticky_panel_for_key(host, "ctrl+d")`
   with a minimal host (e.g. a `SimpleNamespace` with
   `_keymap_registry=SimpleNamespace(app=SimpleNamespace(scroll_detail_down="ctrl+d", scroll_detail_up="ctrl+u"))`
   and `_agent_detail=lambda: detail`) over the both- overflowing fixture: returns True
   and scrolls the footer, not the header; an unrelated key (e.g. `"x"`) returns False.

If a late debounced publish ever clears an injected jump map during a test, re-inject
after `wait_for` settles rather than weakening assertions.

The existing suites must keep passing unchanged except the `_SearchScrollHost` rename:
`tests/ace/tui/widgets/test_agent_header_panel_scroll.py`,
`tests/ace/tui/widgets/test_agent_jump_panel_expansion.py`,
`tests/ace/tui/widgets/test_agent_jump_panel_prefix.py`,
`tests/ace/tui/test_agent_metadata_search.py`.

## Verification

1. Run the targeted suites first:
   `pytest tests/ace/tui/widgets/test_agent_jump_panel_scroll.py tests/ace/tui/widgets/test_agent_header_panel_scroll.py tests/ace/tui/widgets/test_agent_jump_panel_expansion.py tests/ace/tui/widgets/test_agent_jump_panel_prefix.py tests/ace/tui/test_agent_metadata_search.py`.
2. `just fmt`, then `sase tool run check` (per the repo's lint-and-test rules; do not
   run `just check-full`). Watch the symvision stage: the new helper functions in
   `_sticky_panel_scroll.py` must each have a non-test consumer (both panels import
   them).
3. No PNG golden changes are expected (nothing renders differently); do not run
   screenshot updates.

## Out of scope

- Mouse-wheel scrolling (unchanged; Textual already scrolls the hovered panel).
- Collapsed-state scrolling of either sticky panel.
- `g` / `G` (scroll to top/bottom) and full-page scroll keys — they keep targeting the
  focused deck.
- Moving any logic into `sase-core`: this is presentation-only Textual focus/scroll
  routing, which the Rust core boundary explicitly leaves in this repo.
