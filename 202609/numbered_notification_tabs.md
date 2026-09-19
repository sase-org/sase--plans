---
tier: tale
title: Numbered notification tabs with direct digit navigation
goal:
  Make notification tabs visibly numbered and instantly reachable with 1-9 and 0 while
  preserving responsive layout, modal safety state, and jump-mode behavior.
size: medium
proposed_by: bbugyi200.athena.0nh
create_time: 2026-09-18 21:38:31
status: wip
---

# Plan: Numbered notification tabs with direct digit navigation

## Design contract

Treat the notification strip's existing priority-sorted tab order as the only source of
truth for both presentation and navigation. Assign the first ten current positions the
shortcut sequence `1`, `2`, …, `9`, `0`; `0` means the tenth tab, matching the Admin
Center's established one-based numbered-tab convention. Tabs after the tenth remain
unnumbered and reachable with `[` / `]` or the mouse rather than advertising a shortcut
that cannot be entered with one key. An unavailable digit is a consumed no-op, so it
cannot leak through the modal to an application-level digit binding.

The mapping is deliberately positional, not persisted by tab identity. Notification tabs
can appear, disappear, or reorder after dismiss, mute, snooze, mark-read, or a priority
configuration change; each keypress must resolve against the same freshly classified
order that the strip renders. Route a successful digit jump through the existing
`_switch_notification_tag_tab()` path so row marks, pending confirmations, the `+1`
cursor, highlighting, and detail content receive exactly the same safe reset as bracket
and mouse navigation. Pressing the digit for the already-active tab is a no-op.

Bare digits select tabs only in the modal's normal state. Apostrophe row-jump mode keeps
first refusal over every hint character, including one- and two-character digit hints,
through the modal's existing `on_key()` interception. Thus `2` selects tab two, while
`'` then `2` selects the row carrying jump hint `2` without changing tabs.

Render each assigned shortcut as a compact, unboxed digit immediately before the tab
icon, following the visual language of the reusable numbered `PanelTabStrip`: inactive
digits are muted, while the active digit takes the tab's resolved accent so number,
icon, and selection read as one unit. Keep the digit inside the tab's cell-accurate
mouse range. Preserve the current fit-first behavior with three intentional tiers:

1. Full: shortcut, icon, label, count, and optional priority mark for every tab.
2. Compact: shortcut, icon, count/mark for inactive tabs, with the active label
   retained.
3. Micro: tightly separated shortcut/icon/count cells when compact still does not fit.

Use the same restrained box-drawing separator style as the other panel tab strips, with
tighter separators at compact and micro widths. Shortcut digits must never be shed by
reflow; they are the visible affordance for the new behavior. Existing unique-icon,
count, active-label, priority-mark, and terminal-cell-width guarantees remain intact.

## Implementation

1. In `src/sase/ace/tui/modals/notification_modal_constants.py`, define one immutable
   ordered shortcut sequence for `1` through `9`, then `0`, and use it anywhere the
   binding or renderer needs the positional contract. Update all notification footer
   variants to advertise direct digit selection and retain `[` / `]` as cycling, using
   concise wording that still survives the modal's already-dense hint line.

2. In `src/sase/ace/tui/modals/notification_modal.py`, generate ten modal-local, hidden
   digit bindings from that sequence and add a parameterized action that resolves the
   requested current tab position before delegating to `_switch_notification_tag_tab()`.
   Keep the operation synchronous, read-only except for existing in-memory UI state, and
   free of new I/O or event-loop work. Do not add these bindings to
   `src/sase/default_config.yml`: they are fixed positional labels rendered directly on
   dynamic tabs, like the Admin Center's numbered navigation, not user-rebindable
   semantic actions whose displayed digits could safely diverge.

3. In `src/sase/ace/tui/modals/notification_modal_tags.py`, incorporate shortcuts into
   `NotificationTagStrip` rendering from the shared sequence. Refactor the current
   full/compact boolean into an explicit fit ladder if needed so the widest fitting
   representation is selected without dropping shortcut digits. Style active and
   inactive shortcut spans deliberately, keep number/icon/count/priority-mark fragments
   within each tab's click range, and continue measuring ranges and overflow with
   `rich.cells.cell_len` so wide emoji do not shift later tabs. Resizes at an unchanged
   width must remain a no-op.

4. Update `docs/notifications.md` in both the modal-keybinding table and Tabs and
   Ordering description. Document the exact `1`–`9`, `0` mapping, the ten-tab limit,
   fallback navigation for later tabs, dynamic renumbering after the tab set changes,
   responsive rendering, and apostrophe jump-mode precedence. Avoid changing Rust core
   classification or ordering: this feature is presentation-only TUI behavior and must
   continue consuming the core-provided ordered tab list.

5. Extend focused regression coverage:
   - `tests/test_notification_modal_action_bindings.py`: assert all ten digit bindings,
     their ordered action arguments, hidden/modal-local behavior, and updated footer
     hints.
   - `tests/test_notification_modal_mark_and_tabs.py`: cover first, middle, ninth, and
     tenth selection; unavailable digits; pressing the active digit; state clearing via
     the shared switch path; and fresh positional resolution after a mutation changes
     the tab set/order.
   - `tests/test_notification_modal_jump.py`: add pilot-level proof that a bare digit
     switches tabs while apostrophe mode continues to consume the same digit (including
     the existing two-character `10` hint path) as row navigation.
   - `tests/test_notification_modal_tag_strip.py`: assert full, compact, and micro text
     and styles; `0` on the tenth tab; no false number on an eleventh tab; fit
     selection; active-tab clarity; unchanged-width resize behavior; and click ranges
     that include shortcut digits while remaining correct across wide icons and priority
     marks.
   - Keep existing tab-order/routing tests authoritative; the new feature must not alter
     ownership, priority sorting, labels, or counts.

6. Refresh the affected deterministic notification-modal PNG goldens under
   `tests/ace/tui/visual/snapshots/png/`, using the existing multi-tab Beads fixture as
   the primary aesthetic review. Strengthen its SVG assertions so the shortcut sequence
   is present in canonical order and every tab remains visible in the 120×40 compact
   layout. Add a narrower deterministic visual case only if the focused renderer tests
   cannot exercise the micro tier faithfully. Inspect the generated visual report and
   every changed golden rather than accepting snapshot rewrites mechanically.

## Acceptance criteria

- Every populated notification tab strip visibly numbers its first ten tabs as `1`–`9`,
  `0`, using the same current order as mouse and bracket navigation.
- Pressing one of those digits immediately activates the matching tab; missing positions
  do nothing, and tabs beyond ten remain reachable without misleading numeric labels.
- Dismiss/mute/snooze/read mutations cannot leave a stale digit-to-tab mapping, and
  direct jumps preserve the existing tab-switch safety resets and first-row highlight.
- Apostrophe jump hints, including numeric and multi-key hints, still win while jump
  mode is active.
- Full and narrow strips remain legible, active state is visually obvious, all rendered
  shortcut cells are clickable, and wide glyphs cannot corrupt hit testing.
- The change introduces no disk access, subprocess work, timers, or awaits on the
  keystroke/render path and does not change Rust core tab classification.

## Verification

Run the focused non-visual suites while iterating:

```bash
pytest -q \
  tests/test_notification_modal_action_bindings.py \
  tests/test_notification_modal_mark_and_tabs.py \
  tests/test_notification_modal_jump.py \
  tests/test_notification_modal_tag_strip.py \
  tests/test_notification_modal_tab_order.py \
  tests/test_notification_modal_tab_routing.py
```

Then run `just fix`, followed by `just check` (using `/sase_monitor` if it becomes
long-running). Refresh the targeted notification visual suite with
`just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py`
and any other notification snapshot modules selected by the changed strip, inspect the
retained report plus every PNG diff, and finish with the corresponding
`just fix-tui-screenshots --check` selectors. If scoped verification broadens or reports
an unusual selection, use `/sase_monitor` for `just check-full` as required by the
project's two-speed verification policy.
