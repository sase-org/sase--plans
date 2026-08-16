---
tier: tale
title: Adaptive Launch Control description height
goal:
  Launch Control displays every wrapped model-pool member while keeping its modal
  context, footer, navigation, and border fully visible across supported viewport sizes.
size: small
proposed_by: bbugyi200.athena.033
create_time: 2026-08-15 20:25:59
status: wip
---

# Adaptive Launch Control description height

## Goal

Make the Launch Control description strip show every member of a model pool or fallback
chain, including when the member list wraps beyond two terminal rows, while keeping the
modal contained, navigable, and visually balanced at both preferred and narrow viewport
widths.

## Problem and root cause

The reported screenshot at `.sase/artifacts/home/tmp/screenshots/20260815_201803.png`
shows a highlighted built-in size alias whose pool summary wraps past the visible
description area. The renderer is already producing the complete member list in
`src/sase/ace/tui/modals/models_panel_rendering.py`; the data is not missing.

The clipping comes from `src/sase/ace/tui/styles.tcss`:

- `#models-panel-description` has a fixed border-box `height: 4`.
- Its top border and top padding consume two rows, leaving exactly two content rows.
- A description plus a sufficiently long pool/fallback summary needs three or more
  content rows, so Textual lays out the extra wrapped line outside the visible strip.
- `#models-panel-container`'s documented 39-row budget assumes that fixed four-row
  strip, so making the strip adaptive also requires preserving the title/footer and
  reallocating constrained height deliberately.

The existing fixed height intentionally prevents layout movement while navigating. This
change keeps four rows as the minimum so ordinary one- and two-line descriptions remain
stable, but permits the strip and modal to grow when visibility actually requires it.
Under viewport pressure, the scrollable model list should yield rows; the non-scrollable
contextual description and key footer should never be the clipped elements.

## Design

### Content-responsive description strip

Update the Launch Control rules in `src/sase/ace/tui/styles.tcss` so
`#models-panel-description` uses intrinsic height with a four-row minimum rather than a
four-row fixed height. Preserve the existing separator, padding, muted italic text, and
110-column preferred modal width. Rich's existing wrapping and per-member styles remain
authoritative: do not truncate the summary, impose an arbitrary member-count limit, add
a second rendering path, or make the non-focusable description strip independently
scrollable.

The common case must remain visually unchanged: content needing at most two rows still
gets the current two-row content area. A longer pool/fallback may add only the rows its
wrapped content needs. It is acceptable for the centered modal to resize for such a row;
complete, readable context takes priority over the old zero-layout-shift rule.

### Reliable constrained-height behavior

Revise the adjacent container/list sizing rules and their budget comments together so
the title, adaptive description, and two-line key footer retain their intrinsic heights.
When their combined preferred height reaches the modal/viewport cap, reduce the
`OptionList` viewport and rely on its existing scrolling behavior rather than clipping
the description or footer or allowing the modal border to leave the screen. Keep the
list's current 22-row maximum when space is available, and retain a useful list viewport
at narrow terminal sizes.

Keep this entirely in the Textual presentation layer. Highlight changes already update
the strip from in-memory `AliasView` data, so no new callbacks, disk reads, timers, or
render-path work are needed.

## Implementation

1. In `src/sase/ace/tui/styles.tcss`, replace the description's fixed height with
   content-responsive sizing plus the existing four-row minimum. Adjust the
   `#models-panel-container` / `#models-panel-list` height allocation as needed to make
   the list the flexible, scrollable region at the height cap. Rewrite the stale
   fixed-budget comments to document the new invariants and border-box accounting.
2. Extend `tests/test_models_panel_navigation.py` with a production-style geometry
   regression using a realistic four-member selector whose description and pool line
   wrap to at least three content rows at the panel's preferred width. Assert that:
   - the final selector member is present and the description's laid-out content area is
     tall enough to display every wrapped row;
   - the four-row minimum remains in effect for a normal short description;
   - the footer and modal border stay inside the viewport;
   - the option list remains usable/scrollable when it yields height to the expanded
     strip;
   - highlighting another row still updates the context correctly after the resize.
3. Add a narrow-viewport geometry case (80 columns by 40 rows) for the same long pool,
   where wrapping is more aggressive. Verify full description visibility, a contained
   modal, a visible footer, and a nonzero usable list viewport.
4. Add a dedicated long-pool fixture and PNG snapshot in the existing Models-panel
   visual suite (`tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py` and its
   fixture module). The golden should visibly include the final pool member, the full
   separator/footer treatment, and the complete modal border. Keep the existing
   two-member pool golden unchanged unless the reviewed pixel diff demonstrates an
   unavoidable intentional layout correction.

## Verification

After approval and before running repository commands, run `just install` for the
ephemeral workspace. Then:

1. Run the focused Models-panel navigation/geometry tests.
2. Run the dedicated Models-panel PNG snapshot case, inspect the generated actual,
   expected, and diff artifacts, and accept only the new intentional golden with
   `--sase-update-visual-snapshots`.
3. Run `just test-visual` to ensure the rest of the Launch Control and ACE snapshots do
   not regress.
4. Run the required `just check`. If its scoped selector escalates or reports unusual
   selection, follow project guidance and run `just check-full` through `/sase_monitor`
   with a `--next` continuation.

## Acceptance criteria

- Every model in the reported long pool is visible in Launch Control; none is silently
  clipped after wrapping.
- Short descriptions retain the current spacing and visual rhythm.
- Long descriptions expand only as needed, while the model list becomes the flexible
  scrolling region when total height is constrained.
- The title, description, footer, and full modal border remain visible at 120x40 and
  80x40 test sizes.
- Navigation and highlight updates remain synchronous, in-memory, and responsive.
- Focused tests, the complete visual snapshot suite, and `just check` pass.

## Out of scope

- Changing model-pool parsing, ordering, availability, selection, or resolution.
- Changing the textual content or color semantics of pool/fallback members.
- Adding scrolling or new keybindings to the description strip.
- Editing SASE memory files or generated instruction shims.
