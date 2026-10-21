---
tier: tale
title: Roomier custom gate popups
goal:
  Preview-less custom gate popups grow with the terminal so their context is easy to
  read, and their footer is never clipped.
size: medium
decisions:
  compact_width_cap:
    ask: How wide should a preview-less custom gate get on a wide terminal?
    choices:
      cap_120: Up to 120 columns, twice today's 60 on a ~264-column screen
      cap_100: Up to 100 columns; shorter lines but more wrapping and scrolling
      cap_140:
        Up to 140 columns; least wrapping, but long prose lines on very wide screens
    default: cap_120
    why: Doubles the readable line length but stays under half of a very wide screen
    answer: cap_120
  preview_rail:
    ask:
      Should custom and sudo gates with a preview also widen their left Context/Decision
      rail on wide terminals?
    choices:
      grow:
        Rail takes 35% of the body (42-72 columns); 120-column goldens stay unchanged
      keep: Rail stays fixed at 42 columns; only preview-less gates change
    default: grow
    why:
      Preview gates have the same cramped 42-column notes column; one CSS rule fixes it
    answer: grow
proposed_by: bbugyi200.athena.12u
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-21 08:17:35
status: wip
---

# Plan: Roomier custom gate popups

## Problem

The popup in the screenshot is a **preview-less custom notification gate**:
`CustomGateModal` (`src/sase/ace/tui/modals/custom_gate_modal.py`) composed with the
`gate-review-shell--compact` class because `CustomGateModalData.preview_text is None`.
Two fixed cell counts in `src/sase/ace/tui/styles.tcss` size it. Neither has changed
since commit `0fa8b643ef` ("feat(tui): add responsive gate review workbench"):

```tcss
.gate-review-shell--compact {
    width: 60;            /* fixed 60 columns, whatever the terminal width */
    max-width: 96%;
    height: auto;
    max-height: 96%;
}

.gate-review-shell--compact > .gate-review-actions--compact {
    width: 100%;
    height: auto;
    max-height: 30;       /* fixed 30 rows, whatever the terminal height */
}
```

On the user's terminal (about 264x78 cells), the popup takes about 23% of the width.
Context prose wraps at about 56 characters. The key-hint footer wraps onto three lines.
The 30-row review scroll shows a scrollbar while 40+ rows of the screen sit unused.

A scratch Textual probe (using `StyledGateTestApp`-style loading of the real stylesheet)
also found a **latent clipping bug** in today's layout. The shell is `height: auto` with
`max-height: 96%`, and its review scroll is `height: auto`. Textual has no flex-shrink,
so when the content is long, the clamped shell clips its children instead of shrinking
the scroll. At 100x30 or 80x24 with long notes, the footer lands entirely below the
shell's bottom border. At 80x24 that happens even with one short note. The probe
measured `footer_y=35..38` against `shell_bottom=29` at 100x30. Increasing the height
cap without fixing this would make the bug worse, so the plan fixes both.

## Approach

### 1. Width: scale with the terminal (CSS)

Replace the fixed compact width with a percentage, clamped by the existing horizontal
breakpoints that `CustomGateModal.HORIZONTAL_BREAKPOINTS` already applies
(`-gate-review-narrow` below 100 columns, `-gate-review-wide` at 100 or more):

```tcss
.gate-review-shell--compact {
    width: 75%;
    min-width: 60;
    max-width: 96%;
    height: auto;
    max-height: 96%;
}

CustomGateModal.-gate-review-wide .gate-review-shell--compact {
    max-width: 120;   /* compact_width_cap: cap_100 / cap_120 / cap_140 */
}
```

The `compact_width_cap` decision chooses that `max-width` value: 120 for `cap_120`, 100
for `cap_100`, or 140 for `cap_140`. Name it in a short CSS comment, and make the width
tests and the new large golden below expect whichever cap was approved. With any cap,
the 120x40 width is 90.

Textual 8 applies `max-width` after `min-width` (`Widget._get_box_model`). The narrow
case therefore stays safe: at 80 columns the popup is 60 wide (75% = 60), and on
terminals under 63 columns the 96% cap wins over `min-width: 60`. The width grows
smoothly with no jump at the breakpoint: 74 at 99 columns, 75 at 100, 90 at 120, then
the cap from 160 columns up (120 with the default). The probe confirmed these widths:
264->120, 160->120, 120->90, 100->75, 80->60, 60->57.

Do not change `.gate-review-shell` (the 98%x98% workbench used by preview gates,
`PlanApprovalModal`, and `SudoRequestModal`).

### 2. Height: fit the review scroll to the room that is actually left (Python + CSS fallback)

CSS alone cannot express "remaining height after auto-height siblings, inside an
auto-height parent". Inside an auto parent, fr and % children resolve against the
container while Textual computes the auto height, so the shell would always fill the
screen. A single `vh` value either wastes rows on tall screens or clips the footer on
short ones: the probe showed `70vh` clipping at 100x30. So size the scroll in Python,
which the probe validated at every size and across live resizes:

- In `CustomGateModal`, add a resize hook (follow the existing pattern in
  `init_plan_modal.py`: `on_resize` -> `self.call_after_refresh(...)`). It calls a new
  private method, e.g. `_fit_compact_review_scroll()`, that returns immediately when
  `self._data.preview_text is not None`.
- The method computes the scroll's max height from the measured layout, so a wrapped
  header or footer, the password warning, and future chrome are all accounted for:
  - `shell_max` = the shell's resolved CSS max height:
    `int(shell.styles.max_height.resolve(self.size, self.app.size))`, or
    `self.size.height` when unset. Reading it from CSS keeps 96% in one place.
  - `top` = `scroll.region.y - shell.region.y` (top border, header, and margins).
  - `bottom` =
    `footer.region.bottom - scroll.region.bottom + shell.styles.gutter.bottom` (footer
    margin, footer border and lines, bottom border). Footer and scroll regions are still
    computed when they are clipped, so this works when overflowing too.
  - `scroll.styles.max_height = max(_COMPACT_SCROLL_MIN_ROWS, shell_max - top - bottom)`
    with a small named floor (about 6 rows).
- The header and footer heights depend only on width, so one extra layout pass settles
  everything without a feedback loop. Gate actions and draft banners render inside the
  scroll, and the footer text is static after compose, so only resizes need a refit.
- Replace the CSS `max-height: 30` with a scalable pre-fit fallback such as
  `max-height: 70vh`. It only matters for the first frame before the refit runs.
- The shell stays `height: auto`, so short gates stay short and only long content grows
  the popup. In the probe, a short gate at 264x78 stayed 26 rows tall. An 8-note gate
  grew to 49 rows with no scrollbar. At 120x40 the scroll keeps today's 30 rows with the
  footer visible. At 80x24 and 60x18 the footer stays inside the border and the scroll
  shrinks to fit.

Keep the method small. `custom_gate_modal.py` is 389 lines today and must stay well
under the 500-line `toobig` limit. Do not add these hooks to other modals.

> [!decision] preview_rail = grow In `styles.tcss`, give the preview-bearing rail of the
> two generic gates a proportional width:
>
> ```tcss
> CustomGateModal .gate-review-body > .gate-review-actions,
> SudoRequestModal .gate-review-body > .gate-review-actions {
>     width: 35%;
>     min-width: 42;
>     max-width: 72;
> }
> ```
>
> Leave the base `.gate-review-body > .gate-review-actions` rule (dock, `width: 42`,
> height, margin) in place for `PlanApprovalModal`, whose own `width: 44` and
> `--decisions` overrides must not change. Add `min-width: 0; max-width: 100%;` to the
> shared
> `PlanApprovalModal.-gate-review-narrow ..., CustomGateModal.-gate-review-narrow ...`
> rule. Otherwise the new 72-column cap would stop the narrow, bottom-docked custom-gate
> rail from spanning the full width. This is a no-op for `PlanApprovalModal`, which sets
> no min/max on that rail. `SudoRequestModal` is not in the narrow rule, so below 100
> columns its rail resolves to its 42-column minimum, as it does today. At 120 columns
> the body is about 113 cells, 35% is under 42, and the rail stays 42, so every existing
> 120x40 preview golden is unchanged.

> [!decision] preview_rail = keep Leave the rail rules alone. Only the compact shell
> (sections 1 and 2) changes.

Update the comment above `PlanApprovalModal .gate-review-body > .gate-review-actions`
("generic gates stay at 42") so it is still true under the chosen branch.

## Tests

Add focused layout tests, for example in a new
`tests/ace/tui/test_custom_gate_modal_sizing.py` (the existing presentation test file is
197 lines, so either location is fine). Use the existing `StyledGateTestApp` and
`data()`/`option()` helpers from `tests/ace/tui/_custom_gate_modal_helpers.py` so the
real stylesheet applies. Build a "long notes" fixture with several long prose notes
(about 300 characters each) plus a long shell command note. Cover:

- **Width scaling** (preview-less): the `#custom-gate-container` width is the cap (120
  by default) at 264x78 and 160x50, 90 at 120x40, 75 at 100x30, 60 at 80x24, and at most
  96% of the screen at 60x18.
- **Footer never clipped**: with long notes at 80x24, 100x30, 120x40, and 160x50,
  `#custom-gate-footer`'s region bottom is at or above the shell's bottom border
  (`shell.region.bottom - shell.styles.gutter.bottom`). Also test the short-note gate at
  80x24, which clips today.
- **Grows on tall screens**: with long notes at 160x50, `#custom-gate-review-scroll` is
  taller than the old 30-row cap.
- **Short gates stay compact**: a one-note gate at 264x78 keeps a shell well under the
  screen height (it is `height: auto`, not stretched).
- **Refit on resize**: open at 160x50, `await pilot.resize_terminal(80, 24)`, pause, and
  assert the footer is still inside the border. Resize back and assert the scroll grows
  again.
- **Preview gates untouched by the fit**: a preview-bearing gate gets no inline
  `max_height` from the new method.
- Under `preview_rail = grow`: at 200x56 the rail is wider than 42 and at most 72; at
  120x40 it is 42; at 90x40 (narrow) the bottom-docked rail spans the body width. Add
  one `SudoRequestModal` width assertion if a styled harness for it is easy to reuse.

Keep the existing assertions in `test_custom_gate_modal_presentation.py` passing (class
names `gate-review-shell--compact` / `gate-review-actions--compact` do not change).

## Visual goldens

Read the `tui_screenshot` memory note (through `/sase_memory_read`) and follow its
Golden Maintenance contract.

- Expected golden updates: `custom_gate_no_preview_120x40` and
  `custom_gate_password_warning_120x40`, the two preview-less custom gate goldens in
  `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`. Their shells go from 60
  to 90 columns wide.
- Add one new preview-less golden at a large size, for example
  `custom_gate_no_preview_long_200x56`, using long notes. It locks in the "grows with
  the terminal" behavior and the visible footer. Extend that file's `_data()` helper
  with an optional `notes` override instead of duplicating it. That file is 400 lines,
  so stay under 500.
- All other custom gate, sudo request, and plan gate goldens must be unchanged. That
  includes the 90x40, 200x56, and 262x72 plan gate goldens, which exercise the narrow
  rule this plan touches.
- Run the targeted update (e.g.
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py tests/ace/tui/visual/test_ace_png_snapshots_sudo_request.py tests/ace/tui/visual/test_ace_png_snapshots_plan_gate.py tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py`)
  through `/sase_monitor`. Inspect the retained report and every changed or created PNG
  before accepting it. Prove host independence with two targeted `--check` runs, one
  with `TZ=UTC` exported, and neither may report an `unstable` node.

## Verification

1. `sase tool run check` passes (lint gates plus scoped tests, including the new sizing
   tests).
2. The targeted visual check passes with the regenerated and new goldens, and nothing
   else changed.
3. Look at the regenerated `custom_gate_no_preview_120x40` and the new large golden. The
   popup should be visibly wider, the footer should fit on fewer lines, and long context
   should no longer show a needless scrollbar on the large size.
