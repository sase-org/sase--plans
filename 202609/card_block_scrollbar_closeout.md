---
tier: tale
title: Fix the last card-block scrollbar defect and land sase-19x
goal: "A tall-to-short new-subject swap in a spread Main deck leaves both the scroller
  and its scrollbar thumb at the top, and the verified card-block epic is closed.

  "
size: small
bead_id: sase-19x
proposed_by: bbugyi200.athena.0t1
create_time: 2026-09-26 18:21:32
status: wip
---

- **BEAD:**
  [sase-19x](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19x/README.md)

# Plan: Card-block scrollbar closeout

## Context

Epic `sase-19x` has ten closed phases and the closed repair epic `sase-19x.11`. Its
landing notes #13-14 identify the sole remaining epic-owned defect: in the Agents tab, a
tall block-spread document for subject `s1` can be scrolled to the bottom; replacing it
with a short document for subject `s2` leaves `VerticalScroll.scroll_y == 0` but
`vertical_scrollbar.position` at the old offset. The deck remains in spread mode.
`sase bead epic-symbols sase-19x` currently reports no entries, and the earlier
`card_blocks` flag removal has landed. This is presentation-only TUI work; no Rust core
change or new feature flag is needed.

`MainDeckView._show_document_spread()` in `src/sase/ace/tui/widgets/decks/main_view.py`
currently calls `scroll.scroll_to(y=0, animate=False)` for a new subject. The existing
`_scroll_main_to_top()` helper in `main_view_blocks.py` uses an immediate reset and a
deferred `sync_scrollbar_position()` after layout; the block-spread to block-paged path
already uses it successfully. Reuse that helper for the new-subject spread path after
applying the new content. Keep same-subject reading position and the other landing paths
intact.

## Implementation and acceptance

1. Add a focused Textual pilot regression, using the existing deck test fixtures, that
   forces the **Main deck itself** to remain spread across two subjects. Show a
   12-block, ten-lines-per-block Reply at 120x40, scroll its `VerticalScroll` to a
   positive bottom offset, then show a one-line Reply for a different subject. After
   layout settles, assert spread mode, zero `max_scroll_y`, zero `scroll_y`, and zero
   `vertical_scrollbar.position`. The test must observe a positive thumb position before
   the swap so it catches the recorded defect. Run the new test once against the
   unmodified implementation and confirm it fails on the stale thumb. Do not use a
   deck-paged fixture: that exercises a different branch.
2. Change the new-subject branch in `_show_document_spread()` to use the established
   scroll-reset/sync helper. Make the regression pass without a synthetic user scroll,
   extra full redraw, or new event-loop work. Preserve the existing block-spread to
   block-paged scrollbar test and new-subject top-landing test.
3. Run the focused pilot tests, `just fix`, then the repository's normal
   `sase tool run check` gate. Do not run `just check-full`. If the gate fails, inspect
   its triage and establish whether any failure is caused by this change; resolve new
   failures before declaring the repair complete. Earlier landing notes reported an
   unrelated active `sase-1ab` Symvision failure; treat that as historical context, not
   proof about the current tree.

## Epic closeout

After the repair is committed, re-read `sase-19x`, verify all descendants are closed,
confirm no `--epic-symbol` entries remain, and compare the current tree against landing
note #12 and the original epic plan for any new card-block integration drift. Record the
regression and gate result in the close note, including the outcomes of the already
triaged follow-ups from note #12. The designated epic land agent must perform the final
`sase-19x` close. A tale coder without that role should leave the parent open and
arrange recovery of the existing land agent with `sase bead work sase-19x` once the fix
has landed; do not create a second repair epic, phase, or task for this known defect.
