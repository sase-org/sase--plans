---
tier: tale
title: Label the ACE header usage cluster with a dim "usage:" prefix
goal:
  The ACE header's usage-window badges start with a dim `usage:` label styled like the
  status-row `<type>:` labels. When space is tight, the label is dropped before any
  window is hidden.
size: small
proposed_by: bbugyi200.athena.0rv
create_time: 2026-09-25 06:46:24
status: wip
---

# Add a dim `usage:` label to the ACE header usage cluster

## Goal

The ACE header's top row shows the provider usage-window badges at the right edge (for
example `🎭 62% 3d4h · fable 100% 1d8h  🤖 45% 2d5h`). Nothing says what these badges
are. The rows under the header already label their groups as `<type>: <body>`, with only
the label dimmed. Examples are the top bar's `inbox: ⚑1 ✉18` (`TopBarGroup` in
`src/sase/ace/tui/widgets/top_bar_group.py`) and the Agents status row's
`load: 5/8 · model: opus@high · project: +sase` (`AgentLoadIndicator`,
`LaunchContextBar`). The usage cluster should follow the same rule and render as:

```
usage: 🎭 62% 3d4h · fable 100% 1d8h  🤖 45% 2d5h
```

## Current design (for context)

- `src/sase/ace/tui/widgets/usage_header.py`: `UsageHeader` docks
  `ProviderUsageIndicator` (`#provider-usage-indicator`) on the right. It measures a
  symmetric cell budget and passes it in with `set_usage_budget`.
- `src/sase/ace/tui/widgets/provider_usage_indicator.py`:
  `ProviderUsageIndicator._build_content` calls
  `build_usage_indicator_segment(groups, budget=..., dark=...)`. `_content_width` is
  `content.cell_len`. It drives `click_within_rendered_extent`, so any cell that renders
  text opens Providers · Usage when clicked.
- `src/sase/ace/tui/widgets/_provider_usage_indicator.py`:
  `build_usage_indicator_segment` does all of the budget packing:
  1. the full segment (every window) if it fits;
  2. otherwise the longest complete-window prefix plus `  +N`;
  3. otherwise `_fallback_segment`: `usage N`, `N`, `N`, `…`.

  `_render_visible_windows` emits a leading `_OUTER_PADDING` (`" "`, gap style), the
  provider groups (badge surface), an optional `+N` disclosure, and a trailing
  `_OUTER_PADDING`. `_prefix_cell_widths` mirrors that width math.

## Design decisions

1. **Label text and style match the status rows exactly.** The label is the literal
   `"usage: "` as one Rich span with style `"dim"`. There is no explicit color or
   background, the same as `TopBarGroup._composed_text` (`f"{GROUP_LABEL}: "`,
   `style="dim"`) and `AgentLoadIndicator`'s `_LOAD_LABEL = "load: "`. It takes the
   header's own foreground and background, so it reads like the dim labels on the rows
   below in both dark and light themes. Do not add the label to the badge-surface or
   gap-surface palette.
2. **The label replaces the leading outer padding.** When the label is shown, the
   segment is `"usage: "` followed directly by the first provider icon. The label's
   trailing space separates it from the first badge, like the space in `load: ` or
   `inbox: `. The leading `_OUTER_PADDING` gap cell is left out in this case. The
   trailing `_OUTER_PADDING` stays as it is. So the labeled full segment is exactly
   `cell_len("usage: ") - cell_len(_OUTER_PADDING)` (= 6) cells wider than the unlabeled
   full segment.
3. **Data beats the label (the same full → compact rule the top bar uses).**
   `choose_top_bar_density` drops labels before it hides values. The packing ladder
   becomes:
   1. **labeled** full segment, when `budget is None` or it fits;
   2. otherwise the existing **unlabeled** ladder with no changes: unlabeled full,
      prefix plus `+N`, then `usage N` / `N` / `…`.

   The label never appears together with a `+N` overflow or a fallback rung. The
   `usage N` fallback already names itself, so there is no `usage: usage 3`.

4. **The label is part of the click and hover target.** `_content_width` already follows
   `content.cell_len`, so a click on `usage:` opens Providers · Usage and does not
   expand the header. `TopBarGroup` behaves the same way: "the label is part of the
   widget, so hovering or clicking it behaves like the value". The tooltip text does not
   change.
5. **Presentation-only change.** This is Textual/Rich rendering, so under the Rust core
   boundary it stays in this repo. There are no config, keymap, or `default_config.yml`
   changes.

## Implementation steps

### 1. `src/sase/ace/tui/widgets/_provider_usage_indicator.py`

- Add a module constant `_USAGE_LABEL = "usage: "` next to `_OUTER_PADDING` and the
  others. Add `_USAGE_LABEL_STYLE = "dim"` too, with a short comment that it matches the
  status-row `<type>: ` micro-labels.
- Give `_render_visible_windows` a keyword-only `labeled: bool` parameter. When it is
  `True`, append `_USAGE_LABEL` with `_USAGE_LABEL_STYLE` in place of the leading
  `_OUTER_PADDING` gap span. The rest stays the same.
- In `build_usage_indicator_segment`:
  - Render the labeled full segment first
    (`visible_count=total, hidden_count=0, labeled=True`). Return it when
    `budget is None` or `labeled.cell_len <= budget`.
  - Otherwise continue with the current logic unchanged, using `labeled=False` for the
    unlabeled full check and every prefix/overflow render. `_prefix_cell_widths` and
    `_overflow_width` keep describing the unlabeled layout.
  - Update the docstring. It should say the function returns the labeled full cluster
    when it fits, and otherwise the richest unlabeled complete-window prefix.
- Update the module docstring or a nearby comment so the label rule is written down: the
  label is dropped first, and it never appears with `+N` or a fallback.

### 2. `src/sase/ace/tui/widgets/provider_usage_indicator.py`

- No logic change should be needed. Check that `_build_content` still returns `Text("")`
  for empty groups, so no label is shown without windows. Check that `_content_width`
  now includes the label cells.
- Optional: in the `_build_tooltip` notation text, mention that the dim `usage:` label
  is dropped first when space is tight. Keep it to one short clause, or leave the
  tooltip alone.

### 3. Docs: `docs/ace.md`

In the header usage-indicator section (around the paragraph that begins "Provider groups
always render in provider-name order" and the "Clicking the usage cluster" sentence):

- Say that the cluster starts with a dim `usage:` label, in the same style as the
  `<type>: <body>` labels on the top bar and status rows.
- Say that when the labeled cluster does not fit, the label is dropped first, before any
  window is hidden behind `+N`.
- Add the `usage:` label to the list of clickable parts in the "Clicking the usage
  cluster — including …" sentence.

Run `just fmt` afterwards so the Markdown wrapping is normalized.

### 4. Unit tests

Update the existing tests that assumed the unlabeled full segment was the `budget=None`
/ "fits" result. Mainly:

- `tests/test_provider_usage_indicator_presentation_layout.py`. Several tests build
  `full = build_usage_indicator_segment(groups)` and then probe `full.cell_len - 1`,
  expecting an overflow. With the label, `full` is now the labeled segment, and
  `full.cell_len - 1` yields the unlabeled full segment. Derive the unlabeled width
  explicitly, for example with a small test helper that returns
  `build_usage_indicator_segment(groups, budget=labeled.cell_len - 1)`, which is the
  unlabeled full segment. Keep each test's original intent (overflow, fallback ladder,
  monotonic widths, no dangling `·`). Literal expectations like `" 🎭 62% 3d4h "` become
  `"usage: 🎭 62% 3d4h "` where the labeled form is expected.
- `tests/test_provider_usage_indicator_presentation.py` and
  `tests/test_provider_usage_indicator_presentation_style.py`: update exact `.plain`
  expectations and any span-offset or style lookups that assumed the segment starts with
  the gap-styled `" "`.
- `tests/llm_provider/test_agy_usage_indicator_default.py`,
  `test_claude_fable_usage_identity.py`, `test_claude_fable_usage_indicator_default.py`,
  `test_muse_usage_indicator_default.py`, `test_usage_peek.py`: fix any exact-plain
  assertions.
- `tests/test_provider_usage_indicator_widget.py` and
  `tests/ace/tui/test_usage_header.py`: these mostly use `in` / `== ""` checks. Fix any
  that break.

Add new focused tests in `tests/test_provider_usage_indicator_presentation_layout.py`,
or split them into a new module if that file would grow too large for the `toobig` gate:

- With `budget=None` and with a budget equal to the labeled width, the segment starts
  with `usage: `, followed directly by the first provider icon (no extra gap cell). The
  `usage: ` span's style is exactly `"dim"`, with no color or background.
- With `budget = labeled.cell_len - 1`, the result is the unlabeled full segment (same
  plain text as before this change, starting with `" "`) and contains no `usage:`.
- For every budget from 0 to the labeled width, `usage:` appears only when every window
  is visible. It never appears together with `+N` or a fallback rung, and the result
  never exceeds the budget. Extend the existing every-budget sweep or add a parallel
  one.
- Empty groups still render `""`, so there is no bare label.
- Widget/header click test (in `tests/ace/tui/test_usage_header.py`, following the
  existing click tests there): a click on the `usage:` label cells opens Providers ·
  Usage and does not toggle the tall header.

### 5. PNG visual goldens

The header usage cluster appears in the goldens produced by:

- `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`
- `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py`

(`top_bar_usage_*`, `top_bar_agy_usage_badges_*`, `top_bar_compact_usage_badges_*`,
`top_bar_disable_pill_usage_*`, `header_usage_long_title_80x24`, …). Regenerate them
with the screenshot tool, scoped to those files. Read the `tui` / `tui_screenshot`
reference memory first.

```bash
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py
```

This is a known-long command, so run it through `/sase_monitor` if it could exceed the
turn's synchronous limit. Then check that there are no other goldens with drift that
show a populated usage cluster. Grep the visual fixtures for usage-projection
monkeypatches, such as `_provider_usage_indicator_fixtures` imports or
`cached_usage_indicator_projection`, and include any extra files found in the command
above. View a few of the regenerated PNGs, including a dark and a light theme one and a
narrow (60/80-column) one. Confirm three things: `usage:` is dim like `inbox:` on the
row below; wide headers show the label and narrow headers drop it before hiding windows;
and the title stays centered.

## Verification

- `sase tool run check` must pass (lint gates plus scoped tests).
- The scoped `just fix-tui-screenshots` run above finishes with status `clean` or
  `applied` (not `partial`), and the regenerated goldens have been checked by eye.
- Do not run `just check-full` unless explicitly told to.

## Out of scope

- Changing the badge palette, window ordering, or the tooltip's per-window facts.
- Making the header usage cluster a `TopBarGroup`, or moving it off the header row.
