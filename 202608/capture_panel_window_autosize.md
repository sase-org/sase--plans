---
tier: tale
title: Size the Bob Mac Capture panel window to its content
goal:
  The capture popup window opens compact and its height tracks the SwiftUI content
  height as the editor, completion list, errors, and preview appear and disappear.
size: medium
proposed_by: bbugyi200.athena.00w.f0
create_time: 2026-09-09 19:59:54
status: wip
---

# Size the Bob Mac Capture panel window to its content

## Goal

The previous change (`434c753 feat: autosize capture draft editor`) shrank the draft
editor to one line but left the panel window itself pinned at a 760-by-420 content rect,
so the popup did not get smaller — it just gained a large empty gap between the one-line
editor and the footer. Finish the intent of that work: make the _window_ the thing that
autosizes.

A freshly opened popup must be a compact, Spotlight-like bar — a one-line editor plus
the footer actions and nothing else — and the window height must then follow the SwiftUI
content's natural height as the editor grows, the completion list appears, the live
preview arrives, and errors show or clear. Growth and shrink must be anchored, clamped
to the screen, and free of layout feedback loops.

This is a `medium` tale: substantial but bounded work in one linked macOS application,
implementable and verifiable by a single agent. There is no `bob-cli` change — no
grammar, JSON contract, or subprocess interface is touched.

## Where the work happens

All source changes are in the linked `bob-mac-capture` repository. Open it with the
`sase_repo` skill first and use only the path that command prints:

```sh
sase repo open bob-mac-capture -r "Implement content-sized capture panel window"
```

Paths below are relative to that checkout.

## Current behavior and constraints

- `CapturePanelLayout.panelInitialContentSize` is `760x420` and
  `panelMinimumContentSize` is `620x420`. `CapturePanelController.makePanel()` uses the
  former for the content rect and the latter for `contentMinSize`, and
  `CapturePanelView` re-declares the same 420-point minimum on its root `.frame`. Three
  places therefore assert a tall window that no longer matches the content.
- `CapturePanelView`'s root is a `VStack` with `.padding(18)` and 12-point spacing. Its
  sections are already individually bounded: editor at most six visual lines
  (`CaptureEditorHeightPolicy`), completion viewport at most three rows, destination
  summary `lineLimit(2)`, preview a fixed 92-point ideal, footer a single control row.
  Only the idle preview placeholder is unconditional dead space in the empty state.
- The panel is a prewarmed, non-activating, floating, resizable `NSPanel` whose
  `contentView` is a plain `NSHostingView`. `show()` calls `panel.center()` on every
  presentation. `CapturePanelController` is already the `NSWindowDelegate`, which is the
  natural home for resize policy.
- The style mask is `[.nonactivatingPanel, .titled, .fullSizeContentView, .resizable]`
  with no `.closable`/`.miniaturizable`, so there are no traffic-light buttons to
  collide with content — but the titled window still owns a drag region across the top
  of the frame. At the current 420-point height that region overlapped the editor's
  first line unnoticed; at a ~130-point compact height it must be treated as a
  deliberate drag strip instead of stolen editor clicks.
- Unit tests run only on macOS (the repo's `just` targets go through
  `Scripts/xcode-swift.sh`). Any new sizing logic must therefore be factored into pure,
  directly callable helpers rather than being observable only through live SwiftUI
  layout, matching how `CaptureEditorHeightPolicy` and `insertNewlineInEditableTextView`
  are already tested.

## Design

### 1. One source of truth: the content's ideal height

SwiftUI measures; AppKit obeys. Wrap `CapturePanelView`'s existing root `VStack` in a
vertical `ScrollView` and report the _content's_ laid-out height to the controller:

```swift
ScrollView(.vertical) {
    content
        .frame(maxWidth: .infinity, alignment: .topLeading)
        .onGeometryChange(for: CGFloat.self) { $0.size.height } action: { onIdealContentHeightChange($0) }
}
.scrollBounceBehavior(.basedOnSize)
```

The `ScrollView` earns its place twice over:

- A vertical scroll view proposes an unconstrained height to its content, so the
  measured height is the content's _ideal_ height and is independent of the window's
  current height. That is what makes the loop convergent: resizing the window cannot
  change the number we measure, so a resize never triggers another resize.
- It is the graceful-overflow path. If the clamp in step 2 ever binds (very small
  display, very large dynamic type, unusually long error text), the popup scrolls
  instead of clipping the Capture button off the bottom. With
  `.scrollBounceBehavior(.basedOnSize)` it is inert whenever the content fits, and
  because scroll-wheel events hit-test to the innermost view first, the six-line-capped
  editor keeps its own wheel scrolling.

`CapturePanelView` gains one stored property,
`var onIdealContentHeightChange: (CGFloat) -> Void = { _ in }`, supplied by the
controller when it builds the hosting view. Do not route this through
`CapturePanelModel`: an `@Published` height would republish into the view that produced
it. Keep the root `.frame(minWidth:)` and drop the 420-point `minHeight` entirely.

Set `sizingOptions = []` on the `NSHostingView` so the hosting view never imposes its
own intrinsic-size constraints on the window; the sizer below is the only authority on
window height.

### 2. Height policy: a pure, testable sizer

Add `Sources/BobMacCapture/CapturePanelWindowSizer.swift` with a value type that owns
all window geometry math and touches no live window:

- `contentHeight(forIdealContentHeight:availableScreenHeight:) -> CGFloat` clamps the
  measured ideal into `[panelMinimumContentHeight, panelMaximumContentHeight]`, further
  clamps to `availableScreenHeight - 2 * panelScreenMargin` when a screen height is
  known, and rounds up to a display pixel.
- `frame(forCurrentFrame:contentHeight:chromeHeight:visibleFrame:) -> NSRect` converts a
  content height to a frame rect that **preserves the window's top edge and its width**,
  then nudges the result back inside `visibleFrame` if growing downward would cross the
  bottom edge.

Anchoring the top edge is the intuitive choice and matches Spotlight, Alfred, and
Raycast: the editor sits at the top of the popup, so the caret and every character
already typed stay put while new lines appear below. Anchoring the center would slide
the text the user is looking at half a line on every wrap.

Constants move into `CapturePanelLayout` and replace the two `CGSize` constants:

- `rootPadding = 18`, `titlebarDragInset = 28`, `sectionSpacing = 12` — used by the view
  instead of today's literals so the compact estimate cannot drift out of sync.
- `panelInitialContentWidth = 760`, `panelMinimumContentWidth = 620` (unchanged widths).
- `panelCompactContentHeight` — a computed first-frame estimate of the empty state
  (`titlebarDragInset + editor one-line height + sectionSpacing + footer estimate + rootPadding`),
  used for `panelInitialContentSize` so the prewarmed panel is already approximately
  right before its first measurement.
- `panelMinimumContentHeight = 96` — a hard floor that only guards against a degenerate
  measurement, deliberately below the real compact height so a correct measurement is
  never inflated.
- `panelMaximumContentHeight = 720` — above the worst realistic composition (six-line
  editor plus completion viewport plus summary plus error block plus preview plus
  footer), so the clamp is a safety net rather than a routine constraint.
- `panelScreenMargin = 24`.

### 3. Applying the height, without feedback loops

`CapturePanelController` gains `applyIdealContentHeight(_:)` (internal, so tests can
call it) plus `appliedContentHeight` and `pendingRecenter` state:

1. Ignore non-finite measurements and anything at or below one point; an unlaid-out
   hosting view must never collapse the panel.
2. Resolve the target content height through the sizer, using
   `panel.screen?.visibleFrame ?? NSScreen.main?.visibleFrame`.
3. Return early unless the target differs from `appliedContentHeight` by at least half a
   point, or a recenter is pending. Combined with the ideal-height measurement, this is
   the second guard against oscillation.
4. Pin the height before moving the frame: set `contentMinSize` to
   `(panelMinimumContentWidth, target)` and `contentMaxSize` to
   `(.greatestFiniteMagnitude, target)`. Order matters — AppKit constrains `setFrame`
   against these, so a stale pin would fight the new frame.
5. Apply: when `pendingRecenter` is set, `setContentSize` then `panel.center()` and
   clear the flag; otherwise `setFrame(sizer.frame(...), display: true)`.

Guard the whole body with an `isApplyingContentHeight` reentrancy flag.

Resizing is **not** animated. `NSWindow.setFrame(display:animate:)` runs a blocking ~200
ms animation that cannot keep up with per-keystroke growth, and any window animation
would desynchronize from the SwiftUI content inside it and show a clipped or floating
footer mid-flight. For the same reason, remove the editor's internal
`easeOut(duration: 0.12)` height transition and its `accessibilityReduceMotion` handling
from `AutosizingCaptureEditor` so the editor card and the window change size in the same
layout pass. Instant, lockstep resizing is what makes this feel crisp rather than
rubbery; if animation is ever revisited, the window and its content must animate
together.

### 4. Height is content-owned; width stays the user's

The window keeps `.resizable`, but a vertical drag would fight the content on the next
keystroke, so make it inert and leave horizontal resizing fully functional (it changes
soft wrapping, which legitimately changes the content height). Implement
`windowWillResize(_:to:)` on the existing delegate to return the proposed width together
with the frame height required by `appliedContentHeight`. That single method makes live
width drags reflow the editor and follow the content height automatically, and makes
vertical drags a no-op, with the `contentMinSize`/`contentMaxSize` pins as backup.

### 5. Presentation and recentering

`show()` sets `pendingRecenter = true`, calls `contentView?.layoutSubtreeIfNeeded()` so
the current draft's height is measured and applied _before_ the window is centered and
ordered front, then proceeds as it does today. If the measurement instead lands after
presentation, the pending flag makes the first application recenter exactly once;
subsequent applications keep the top edge fixed. A fresh popup therefore appears
centered and compact, a retained draft (from Escape or a failed capture) reopens
centered at its own natural size, and typing afterwards only grows downward.

### 6. Make the compact state genuinely compact

Two content changes matter for the empty state:

- Render `PreviewPane` only when `model.previewState != .idle`. The idle placeholder is
  a 92-point card that says "Preview" and shows nothing; keeping it would put a hundred
  points of empty chrome under a one-line editor and undercut the whole change. The pane
  appears on the first keystroke (`.loading`) and collapses again when the draft is
  cleared, so it does not flicker mid-draft.
- Use `titlebarDragInset` for the root's top padding instead of `rootPadding`, so the
  editor's first line starts below the titled window's drag region. At the old height
  that overlap was invisible; at compact height it would mean clicking the top of the
  editor drags the window. The resulting strip doubles as an obvious place to grab and
  move the panel.

With the fixed 420-point window gone, the three-row completion viewport that was chosen
to protect the preview inside that window is no longer necessary: raise
`completionVisibleRows` from 3 to 5. The window now grows to fit the list instead of the
list competing with the preview, and the worst-case total stays under
`panelMaximumContentHeight`. Leave the editor's six-line cap alone — that limit was a
deliberate decision in the previous plan and is not what this change is about.

## Implementation steps

1. Replace the `panelInitialContentSize`/`panelMinimumContentSize` constants in
   `Sources/BobMacCapture/CapturePanelView.swift` with the layout constants from design
   step 2, keeping `panelInitialContentSize`/`panelMinimumContentSize` only if their
   values are recomputed from the new width/height constants.
2. Add `Sources/BobMacCapture/CapturePanelWindowSizer.swift` with the clamping and
   frame-math functions, pure and free of any live `NSWindow` access.
3. In `CapturePanelView`, wrap the root stack in the measuring `ScrollView`, add the
   `onIdealContentHeightChange` property, drop the 420-point `minHeight`, apply the root
   padding/spacing constants, and make `PreviewPane` conditional on a non-idle preview
   state.
4. In `AutosizingCaptureEditor`, remove the height animation transaction and the
   now-unused `accessibilityReduceMotion` environment read; keep the measurement,
   clamping, prompt overlay, attributed binding, selection, and accessibility behavior
   exactly as they are.
5. Raise `completionVisibleRows` to 5.
6. In `Sources/BobMacCapture/CapturePanelController.swift`: build the hosting view with
   a weak-self height callback and `sizingOptions = []`; add `appliedContentHeight`,
   `pendingRecenter`, `isApplyingContentHeight`, and `applyIdealContentHeight(_:)`; add
   `windowWillResize(_:to:)`; and update `makePanel()` and `show()` per design steps 3
   through 5.
7. Update `README.md` so the panel section describes a content-sized popup: compact
   empty state, height that follows content and is not user-draggable, width that is,
   and internal scrolling as the small-screen fallback.

## Automated verification

Extend `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`:

- Update `testPanelHasStableNonActivatingStyleInInitializer` for the new sizing: the
  style mask, floating, and collection-behavior assertions stay; the initial content
  height must equal the compact constant and be far below the old 420 points; the
  content minimum width must still be honored.
- Sizer clamping: an ideal below the floor returns the floor, an ideal above the ceiling
  returns the ceiling, an ideal in between is returned rounded up to a pixel, and a
  small `availableScreenHeight` clamps below the ceiling with margins applied.
- Sizer frame math: growing and shrinking both keep `maxY`, `origin.x`, and `width`
  unchanged, and a window positioned near the bottom of `visibleFrame` that grows is
  pushed up so the resulting frame stays entirely inside `visibleFrame`.
- `applyIdealContentHeight(_:)` on a real panel: a larger ideal grows the content height
  and keeps the top edge; applying the same ideal twice leaves the frame untouched (the
  anti-oscillation guard); and zero, negative, and non-finite measurements leave the
  frame untouched.
- `windowWillResize(_:to:)` returns the pinned frame height for a taller proposal while
  passing the proposed width through for a wider one.
- Leave the existing key-router, editor-height-policy, completion, and newline-responder
  tests passing unchanged.

Run the repository's supported checks through its Xcode-resolved wrapper:

```sh
just format-lint
just build
just test
```

If this host cannot select an Apple developer directory, report the exact blocker
verbatim instead of implying the checks passed.

## macOS interaction and visual verification

Because the outcome is a window that resizes itself, verify a development or bundled
build on macOS 26 in addition to unit tests:

1. Open a fresh popup and confirm it is a compact bar — one-line editor plus footer, no
   empty preview card, no dead space — centered like before and focused for typing.
2. Type across a soft wrap, insert newlines with Control-J, Shift-Return, and
   Option-Return, then delete back down. Confirm the window grows and shrinks one line
   at a time, the top edge never moves, the editor card and window stay in lockstep with
   no clipped footer or flicker, and the caret stays where it was.
3. Keep typing past six visual lines and confirm the window stops growing for the
   editor, the editor scrolls internally, and the caret stays visible.
4. Trigger completion and confirm the window grows to show the list below the editor
   without covering the preview, that the list scrolls its selection into view, and that
   dismissing it shrinks the window back.
5. Watch the preview appear on the first keystroke and collapse when the draft is
   cleared; force a capture failure and confirm the error block and its buttons grow the
   window and are never clipped.
6. Drag the window's right edge to change width and confirm the height follows the
   reflowed wrapping during the live drag; try to drag the bottom edge and confirm the
   height does not change; drag the top strip and confirm the window moves and that
   clicking the first editor line still places the caret.
7. Move the panel near the bottom of the screen and grow the draft; confirm it slides up
   to stay fully on screen instead of running off the bottom.
8. Escape with a draft and reopen: the popup returns centered at the size its retained
   content needs. Capture successfully, reopen, and confirm a compact empty popup again.
9. Repeat the growth check with Reduce Motion and with VoiceOver enabled, and on a
   second display with a different scale factor if one is available. Confirm no unwanted
   animation, no announcement or focus regressions, and correct sizing at both scales.

## Acceptance criteria

- A fresh capture popup is a compact window sized to a one-line editor plus footer, with
  no empty preview placeholder and no dead vertical space.
- The window height tracks the content: editor growth and shrink, completion appearing
  and dismissing, preview arriving, and errors showing and clearing all resize the
  window, anchored at its top edge and clamped inside the screen's visible frame.
- Height changes are stable — no oscillation, no per-keystroke jitter, and no resize
  triggered by a resize — and the editor and window change size in the same pass.
- Vertical dragging cannot change the height; horizontal dragging still works and
  reflows the content, with internal scrolling as the fallback if the screen clamp
  binds.
- Editor highlighting, selection, undo, completion, newline shortcuts, accessibility,
  retained drafts, and capture submission behave exactly as before.
- `bob-cli` is unchanged, `README.md` describes the final behavior, and the repository's
  format-lint, build, and test commands pass on macOS.
