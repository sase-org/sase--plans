---
tier: tale
title: Make the K bead preview card much larger
goal:
  Pressing K on a bead ID opens a card that fills most of the terminal, shows far more
  of the bead without scrolling, and keeps its footer visible at every size.
size: small
decisions:
  height_mode:
    ask:
      Should a short bead get a shorter card that grows with its content up to the large
      frame?
    choices:
      fixed:
        Every bead fills the large frame; short beads show empty rows above a pinned
        footer
      fit:
        Short beads stay short, long ones grow to the same max; adds a small Python
        resize hook
    default: fixed
    why: Matches the K file-preview panel's large baseline frame and stays CSS-only.
    answer: fixed
proposed_by: bbugyi200.athena.1df
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 10:19:56
status: wip
---

# Make the `K` bead preview card much larger

## Problem

Pressing `K` on a bead ID in the prompt input opens `BeadPreviewModal`. It inherited the
compact-card sizing of its siblings, the glossary and repo cards. In
`src/sase/ace/tui/styles.tcss`:

- `BeadPreviewModal > Container` uses
  `width: auto; max-width: 84; height: auto; max-height: 80%`.
- `BeadPreviewModal #bead-preview-scroll` is capped at `max-height: 20` cells. A comment
  explains why the cap is in cells: auto-height parents ignore `%`/`fr`/`vh` caps on
  their children before the first layout, which would clip the in-flow footer.

So on a large terminal (about 256x80) the card is about 84 columns by 32 rows. An epic's
description plus a 13-phase list does not fit in the 20-row body. Only phases 1-9 are
visible before you have to scroll, and most of the screen goes unused. The goal is a
card that is much larger in both directions and still readable.

## Outcome

- The card opens as a large frame: **90% of the terminal width (capped at 160 columns)
  and 90% of the terminal height**. The 160-column cap keeps description prose readable
  on very wide terminals.
- The scrollable body (status, description, related beads, property grid) gets all of
  the space between the pinned title and the footer, instead of a fixed 20 rows.
- The footer key hints stay visible at every terminal size, including small ones such as
  100x30.
- The dim separator between the related-beads block and the property grid spans the
  card's full width. Today it is a hard-coded 66-character string that would stop
  two-thirds of the way across the wider card.
- The reference docs stop calling the card "compact".

## Scope

- **In scope:** `BeadPreviewModal` only: its TCSS blocks, the separator in
  `_build_meta`, its tests, its PNG goldens, and its docs paragraph. This is
  presentation-only Textual code, so nothing crosses into `sase-core`.
- **Out of scope:**
  - The sibling `RepoPreviewModal` and `GlossaryPreviewModal` cards keep their compact
    84-column frames. The request is about the bead card, and their content is short.
  - The order of the `PHASES` / `CHILDREN` list. It comes from the Rust
    `bead_show_issue_detail` payload, and this plan does not change it.
  - `_PROGRESS_METER_CAP`. The chip-row meter keeps its 12-glyph cap.
  - Card content, key bindings, loading and navigation behavior, and every other modal.

## Changes

### 1. Frame and body sizing in `src/sase/ace/tui/styles.tcss`

Edit only the `/* ===== Bead Preview Modal Styling ===== */` section.

> [!decision] height_mode = fixed
>
> Give the container a definite size and let the body take the remaining rows. This is
> the same pattern `PreviewPanelModal` (the `K` file preview) uses:
>
> - `BeadPreviewModal > Container`: `width: 90%; max-width: 160; height: 90%;`. Keep
>   `border: thick $primary; background: $surface; padding: 1 2;` and
>   `overflow-y: hidden`. Remove `width: auto`, `height: auto`, and `max-height: 80%`.
> - `BeadPreviewModal #bead-preview-scroll`: replace `height: auto; max-height: 20;`
>   with `height: 1fr;`. Keep `scrollbar-gutter: stable`, the border, and the padding.
> - Replace the "Cell cap" comment above the scroll block with one line. It should say
>   that the container has a definite size, so the body uses `1fr` and the footer stays
>   pinned. The old auto-height clipping hazard no longer applies.
>
> The title (`height: auto`) and the footer (`height: auto`, with its margin, border,
> and padding) stay as they are. The vertical layout gives every remaining row to the
> `1fr` body.

> [!decision] height_mode = fit
>
> Keep an auto-height container. Compute the body's row cap in Python, in cells, so the
> auto-height clipping hazard in the existing comment never arises:
>
> - `BeadPreviewModal > Container`:
>   `width: 90%; max-width: 160; height: auto; max-height: 90%;`. Keep the border,
>   background, padding, and `overflow-y: hidden`.
> - `BeadPreviewModal #bead-preview-scroll`: keep `height: auto; max-height: 20;` as the
>   pre-mount fallback. Update the comment to say that Python widens the cap after
>   mount.
> - In `src/sase/ace/tui/modals/bead_preview_modal.py`, add `_sync_body_cap()`. It sets
>   `scroll.styles.max_height` to
>   `max(_MIN_BODY_ROWS, int(self.size.height * _FRAME_HEIGHT_FRACTION) - chrome)`.
>   - `_FRAME_HEIGHT_FRACTION = 0.9` mirrors the TCSS `max-height: 90%`. Say so in a
>     comment next to the constant.
>   - `_MIN_BODY_ROWS` is about 3.
>   - `chrome` adds up the non-body rows: the container's border and padding (4), the
>     body's own border (2), the title's `outer_size.height` (fallback 5 before layout,
>     which includes its bottom margin), and the footer's `outer_size.height` (fallback
>     4).
>   - Call it from `on_mount` through `call_after_refresh`, from a new screen
>     `on_resize` handler, and at the end of `_refresh_widgets`. The title height
>     changes with the breadcrumb and chip rows. Return early when the widgets are not
>     mounted (`NoMatches`).
>
> A short bead then gets a short card at the full width. A long bead grows until the
> container reaches 90% of the terminal height, then the body scrolls.

### 2. Full-width separator in `src/sase/ace/tui/modals/bead_preview_modal.py`

In `_build_meta`, replace `Text("-" * 66, style="dim")` with
`Rule(characters="─", style="dim")` (`from rich.rule import Rule`). Rich renders a
`Rule` at the full width of the `Static` that hosts it, so the separator spans the card
at any frame width. Leave everything else in `_build_meta` unchanged.

### 3. Layout regression tests

Add a focused test module, `tests/ace/tui/modals/test_bead_preview_modal_layout.py`.
`test_bead_preview_modal.py` is already 515 lines, so the new tests do not go there.

- Use the real stylesheet. Follow the `_StyledTestApp` pattern in
  `tests/test_notification_modal_scroll.py`: a tiny `App` subclass with
  `CSS_PATH = <repo>/src/sase/ace/tui/styles.tcss` that pushes the modal on mount. The
  existing `_BeadModalTestApp` in `test_bead_preview_modal.py` loads no TCSS, so it
  cannot see geometry.
- Build fixtures inline, so the module does not import another test module's private
  helpers:
  - a `BeadStoreRef` under `tmp_path`;
  - a short task bead with a one-line description;
  - a long epic with 14 phase children and a description of several paragraphs.
- Pass `preview=` so no loader runs.
- Each test resolves widgets through `#bead-preview-container`, `#bead-preview-scroll`,
  and `#bead-preview-footer`. Use `wait_for` from `sase.ace.testing` to wait until the
  footer's region has height before asserting.

Tests:

1. **Large frame on a big terminal:** the long epic at `size=(200, 60)`. The container
   `region` is exactly 160 columns wide (the max-width cap engages) and 54 rows tall
   (90% of 60). The footer region is non-empty and lies inside the container region. The
   body has `max_scroll_y > 0` or shows every phase. Assert whichever holds for the
   fixture, and choose the fixture so the assertion is stable.
2. **Footer survives a small terminal:** the long epic at `size=(100, 30)`. The
   container is 90x27. The footer region is non-empty, inside the container, and inside
   the screen. The body scrolls (`max_scroll_y > 0`).
3. **Short bead frame**, which depends on `height_mode`:
   - `fixed`: the short task at `size=(200, 60)` still gets the full 160x54 container,
     and the footer is at the bottom of the container.
   - `fit`: the short task's container is 160 wide and fewer than 54 rows tall. Add a
     resize test: mount the long epic at 200x60, call `pilot.resize_terminal(100, 30)`,
     and assert that the footer stays visible inside a 27-row container.
4. **Separator spans the width:** a unit test that builds
   `BeadPreviewModal(store, "sase-1aa", preview=<epic>)` without mounting, renders
   `_build_meta()` into
   `rich.console.Console(width=100, record=True, color_system=None)`, and asserts that
   one exported line is `"─" * 100`. `_build_meta` already tolerates a `None` palette
   before compose, so mounting is not needed.

Existing tests in `test_bead_preview_modal.py` and `test_bead_preview_render.py` should
pass unchanged. None of them assert sizes or the 66-dash string.

### 4. PNG visual snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_bead_preview.py` owns four 120x40 goldens
under `tests/ace/tui/visual/snapshots/png/`:

- `bead_preview_card_epic_dark_120x40`
- `bead_preview_card_epic_light_120x40`
- `bead_preview_card_phase_dark_120x40`
- `bead_preview_card_loading_dark_120x40`

All four change. At 120x40 the frame becomes 108x36, and the separator is full width.

Add one new snapshot that shows the large frame on a big terminal:

- Name it `bead_preview_card_long_epic_dark_200x60`, with the title
  `ACE bead preview card - long epic on a large terminal`.
- Use `AcePage(query='"visual"', size=(200, 60), patches=patches())`. The 160x50 agent
  session snapshot in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_session_panel_gate.py` shows
  the `size=` pattern.
- Use a `_visual_long_epic_preview()` fixture: a 14-phase epic with mixed statuses and
  sizes, a multi-paragraph description, a plan path, and notes. Reuse the module's
  pinned `_CREATED` / `_UPDATED` timestamps and `_visual_store()`.
- Wait for `PHASES` and for the last phase's title with `wait_for_svg_contains`, then
  `wait_for_visual_idle`.

Regenerate and review the goldens. Follow the TUI screenshot memory
(`sase memory read tui_screenshot.md`) and the lint-and-test memory:

- Run the targeted update
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_bead_preview.py`
  through `/sase_monitor`.
- Inspect the retained report and every created or updated PNG before finalizing. The
  frame must be visibly larger, the footer must be visible, and the separator must span
  the full width.
- Prove the goldens are independent of the host with the targeted
  `just fix-tui-screenshots --check -- <same selector>`, run twice, once with `TZ=UTC`
  exported. Neither run may report an `unstable` node.

### 5. Docs

In `docs/reference/tui/prompt-completion-panels.md`, `## Bead IDs` section, change the
sentence "`K` on a bead ID opens a compact bead card." to describe the large card.
Change only that sentence; the rest of the paragraph still holds.

- `fixed`: "`K` on a bead ID opens a large bead card that fills most of the terminal
  (90% of its height and 90% of its width, up to 160 columns)."
- `fit`: "`K` on a bead ID opens a wide bead card (90% of the terminal width, up to 160
  columns) that grows with its content up to 90% of the terminal height."

Run `just fix` afterwards (docs budgets and footprint catalog).

## Verification

- Run `just fix`, then `sase tool run check`. That is the agent `just check` lane. Do
  not run `just check-full`.
- Run the targeted visual update and the double check-mode run from Changes, step 4.
- Optional live confirmation: `sase screenshot` of a prompt that contains a bead ID,
  after pressing `Esc` then `K` on the ID. Live PNGs are not goldens.

## Acceptance criteria

- On a 200x60 terminal the bead card is 160x54. On 100x30 it is 90x27, and the footer
  key hints are visible.
- A long epic's description and phase list use the enlarged body. The body still scrolls
  with `j`/`k`, `Ctrl+D`/`Ctrl+U`, and `g`/`G` when content overflows.
- The separator above the property grid spans the card's width.
- The glossary and repo `K` cards are unchanged.
- The new layout tests and the updated and new PNG goldens pass. `sase tool run check`
  is green.
