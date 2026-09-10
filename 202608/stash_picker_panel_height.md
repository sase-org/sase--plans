---
tier: tale
title: Keep every stash-picker action visible while the panel autosizes
goal:
  The Control-S stash picker makes the capture panel tall enough to show its rows and
  bottom Delete All action, while retaining safe scrolling on constrained screens.
size: medium
proposed_by: bbugyi200.athena.02g.f0.f0
create_time: 2026-09-09 20:00:31
status: wip
---

# Keep every stash-picker action visible while the panel autosizes

## Goal

Fix Bob Mac Capture's Control-S stash picker so opening it from the compact capture
panel grows the window to the picker's complete intended height. The bottom **Shift-D
Delete All** action must be visible without scrolling on an ordinarily sized display,
including with a one-entry stash, and must remain outside the draft-row scroll viewport.
Preserve the panel's compact closed state, top-anchored content-owned resizing, the
five-row stash viewport cap, and the existing screen-height fallback.

This is a focused layout and sizing correction in the linked `bob-mac-capture`
repository. It does not change uppercase-D routing, clear/restore semantics, stash
capacity or persistence, capture grammar, or any subprocess/JSON contract.

## Evidence and root cause

- The supplied screenshot shows a one-entry stash after the destructive action was moved
  below the row list: the draft row occupies the visible auxiliary region and the bottom
  Delete All action lies below the footer boundary.
- `CapturePanelView` starts from a compact fallback height and gives the auxiliary
  region the remaining space between a fixed editor and footer. The generic auxiliary
  region is itself a vertical `ScrollView`.
- `CanceledDraftStashPicker` now contains a second vertical `ScrollView` for rows with
  only a `maxHeight`, followed by the Delete All button. During the first constrained
  layout pass, that flexible nested scroll view/card can report the already-compressed
  auxiliary viewport instead of its complete intrinsic height.
- `CapturePanelContentHeightPolicy` then treats that compressed measurement as the ideal
  auxiliary height and excludes all auxiliary content from
  `minimumVisibleContentHeight`. The controller therefore has no independent signal that
  the panel must grow enough for the bottom action, producing a stable undersized layout
  instead of converging on the full picker height.

## Product and layout contract

- Opening a nonempty stash from the compact panel must immediately resize the panel so
  the editor, the intended visible stash rows, **Shift-D Delete All**, and the footer
  actions are all visible together whenever the screen has enough room.
- Size the draft-row viewport to the actual number of entries up to the existing
  five-row cap. A one-entry stash should grow only enough for one row plus the action; a
  stash with five or more entries should show the five-row viewport, with later rows
  reachable through the existing inner scrolling and selection-follow behavior.
- Keep Delete All after and outside the draft-row `ScrollView`. It must not scroll with
  the rows and must receive nonzero layout space before the row viewport is allowed to
  compress.
- Break the circular dependency between the old panel height and the picker's reported
  ideal height. Picker sizing must come from its intrinsic row/action/card layout (or an
  equivalent explicit, testable policy), not from the currently available auxiliary
  viewport.
- Treat the stash picker's non-scrolling action chrome as required visible auxiliary
  content in the panel's minimum-height calculation. On a physically short screen where
  the full ideal height cannot fit, retain the existing screen clamp and auxiliary
  overflow behavior: the editor, footer, and destructive action stay available, while
  the row viewport is the portion that shrinks and scrolls.
- Dismissing or clearing the picker must shrink the panel back to the correct compact
  height. Reopening it, changing the stash count, and changing panel width/display scale
  must recompute rather than replay a stale picker measurement.
- Preserve all current keyboard, pointer, accessibility, privacy, and visual semantics;
  this fix must not make lowercase `d` destructive or put draft payloads in diagnostics.

## Implementation

1. Refine the auxiliary sizing contract in
   `Sources/BobMacCapture/CapturePanelView.swift`.
   - Represent auxiliary sizing with both an ideal height and a required visible height,
     or extend `CapturePanelContentHeightPolicy.metrics` with the equivalent explicit
     minimum-auxiliary input.
   - Continue treating ordinary completion/preview/error content as overflow-capable,
     but include the stash picker's non-scrolling action/card chrome in
     `minimumVisibleContentHeight` so `CapturePanelWindowSizer` cannot preserve the old
     compact height when the modal picker opens.
   - Keep all height inputs finite, nonnegative, pixel-rounded, and derived from shared
     layout constants or live geometry; do not add a screenshot-specific window-height
     constant.

2. Make `CanceledDraftStashPicker` expose a stable intrinsic size independent of the
   outer auxiliary viewport.
   - Give the inner draft-row viewport an explicit content-derived height for
     `min(stashCount, stashVisibleRows)` rows, including the list's padding and row
     spacing, while retaining the existing maximum and scroll-to-selection behavior.
   - Align any row/action/card constants with the actual SwiftUI modifiers, or measure
     the non-flexible pieces directly, so the sizing policy cannot omit the Delete All
     row or silently drift when its label/padding changes.
   - Give the bottom Delete All action fixed-size/layout priority relative to the row
     viewport. Under a height clamp, compress the row viewport rather than collapsing or
     pushing the action below an outer scroll boundary.
   - Ensure the full picker/card measurement is what reaches the panel metrics callback
     on first presentation and whenever entry count, width, or display scale changes.

3. Preserve the window controller and sizer invariants in
   `Sources/BobMacCapture/CapturePanelController.swift` and
   `Sources/BobMacCapture/CapturePanelWindowSizer.swift`, changing them only if the
   refined metrics contract requires it.
   - Continue caching the newest metrics, replaying them on presentation, resizing
     without animation, preserving the panel's top edge, and clamping to the screen's
     visible frame.
   - Keep the existing hard maximum for ideal overflow content, but allow the required
     visible portion of the stash picker to participate in the same persistent-minimum
     rule already used for the editor and footer.
   - Avoid timers, delayed resize guesses, or imperative one-off height bumps; the
     measured/pure policy must remain the single source of truth.

4. Extend `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` with focused regression
   coverage.
   - Test the picker/auxiliary height policy for one entry, exactly five entries, and
     more than five entries. Assert that the bottom action/card chrome is included,
     one-entry sizing is compact, and the row contribution caps at five.
   - Test that stash auxiliary metrics increase both ideal content height and the
     required visible content height, while the no-auxiliary compact case and ordinary
     overflow auxiliary case retain their current results.
   - Exercise `CapturePanelWindowSizer` with the refined stash metrics to prove a normal
     display receives the full target, the persistent minimum can win over the general
     maximum when appropriate, and an impossibly short display still clamps safely.
   - Add or extend a controller/presentation regression that starts from the compact
     fallback, applies one-entry stash metrics, and verifies the panel grows to the
     computed content height, remains top-anchored, is idempotent on repeated metrics,
     and can shrink after picker dismissal.
   - Keep existing uppercase-D, model clear, restore-navigation, footer visibility, and
     general autosizing tests unchanged as regression coverage.

5. Update `README.md` only where needed to make the runtime sizing promise explicit: the
   stash picker grows the panel to expose its fixed bottom Delete All action and up to
   five visible rows, with row scrolling or auxiliary overflow reserved for longer
   stashes or genuinely screen-constrained layouts.

## Validation

From the linked `bob-mac-capture` checkout, run:

1. `just format-lint`
2. `just build`
3. `just test`

Then manually exercise a signed/bundled debug build on macOS:

- Begin with the compact empty panel, open a one-entry stash with Control-S, and confirm
  the panel grows immediately enough to show the complete row, the bottom **Shift-D
  Delete All** action, status/footer text, and all footer buttons without auxiliary
  scrolling.
- Repeat with two, five, and 36 entries. Confirm the panel grows with the visible row
  count only through five, Delete All remains fixed below the row viewport, keyboard
  selection scrolls only the rows, and every visible option is clickable and exposed to
  VoiceOver.
- Dismiss with Escape and Control-S, restore a row, and clear with Shift-D and by click;
  confirm each transition recomputes/shrinks the panel correctly and preserves the
  existing model and keyboard behavior.
- Resize the panel width and test increased text size/display scale so reflow produces a
  fresh safe height rather than clipping the action or replaying stale metrics.
- Test on the smallest available display/visible-frame configuration. Confirm the panel
  stays on-screen, the editor/footer/Delete All action remain available, and only the
  draft-row viewport or generic auxiliary overflow scrolls when all ideal content
  physically cannot fit.

## Non-goals

- Do not alter uppercase-D versus lowercase-d routing, confirmation behavior, stash
  ordering/capacity, row accelerators, restore/pop semantics, or session-only storage.
- Do not remove the five-row cap, make all 36 drafts simultaneously visible, or bypass
  the existing screen clamp.
- Do not redesign the panel, move Delete All back above the rows, add a separate window,
  or introduce user-resizable panel height.
