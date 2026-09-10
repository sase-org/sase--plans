---
tier: tale
title: Keep capture-panel controls visible while the window autosizes
goal:
  Make the compact Bob Mac Capture panel size from measured content without ever
  clipping the editor, status text, or primary action buttons, including on its first
  presentation and when the screen-height clamp activates.
size: medium
proposed_by: bbugyi200.athena.00w.f0.f0
create_time: 2026-09-09 19:59:52
status: wip
---

# Keep capture-panel controls visible while the window autosizes

## Goal

Preserve the compact, content-sized popup introduced by
`ffd3786 feat(capture): size the panel window to its content`, while fixing the layout
failure shown in `20260814_100746.png`: the one-line editor is visible, but the bottom
edge of the window cuts through the footer so only the tops of its status text and
buttons can be seen.

The editor, the status row, and the primary Preview/Capture/discard controls must be
inside the visible window at every ordinary panel size. Completion, destination,
preview, and error content must never be silently cropped or covered by those controls.
When the panel cannot show every auxiliary section because the screen-height cap binds,
the auxiliary region must scroll while the editor and primary controls stay put.

This is a `medium` tale: the work is substantial but bounded to the linked
`bob-mac-capture` application and can be implemented and verified coherently by one
agent. It does not change `bob-cli`, capture grammar, subprocess arguments, or JSON
contracts.

## Where the work happens

All implementation changes belong in the linked `bob-mac-capture` repository. Before
reading or editing it, open it through the `sase_repo` skill and use only the path that
command prints:

```sh
sase repo open bob-mac-capture -r "Keep autosized capture-panel controls visible"
```

Paths below are relative to that checkout.

## Evidence and root cause

- The supplied screenshot shows the fresh panel at roughly its compact fallback height.
  The editor fits, but the footer begins below the bottom edge; the status text and
  Preview/Capture buttons are clipped.
- `CapturePanelLayout.panelCompactContentHeight` is an estimate made from
  `footerEstimatedHeight = 28`, not a measurement of the rendered SwiftUI footer. The
  estimate is therefore allowed to be shorter than the actual compact layout.
- `CapturePanelView` wraps the **entire** editor/auxiliary/footer stack in one vertical
  `ScrollView`. A scroll view can legitimately accept a viewport shorter than its
  document, so it gives SwiftUI permission to put the footer below the visible frame.
  This is useful as overflow behavior but cannot itself enforce that primary controls
  stay visible.
- The controller learns the natural height only through a change-based
  `.onGeometryChange` callback. Prewarming can lay out the hosting view before the panel
  is ready to apply the result. Later, `show()` calls `layoutSubtreeIfNeeded()`, but a
  second layout with the same geometry is not a geometry _change_, so it is not a
  reliable replay mechanism. The window can consequently remain at the too-small
  fallback height.
- Once `applyIdealContentHeight(_:)` accepts a target, it pins both `contentMinSize` and
  `contentMaxSize` to that height. A bad initial target therefore persists rather than
  being corrected by AppKit's normal fitting behavior.
- The pure frame math in `CapturePanelWindowSizer` is otherwise sound: preserving the
  top edge, pixel rounding, screen clamping, and width-only user resizing should remain.

The fix must address both failure layers. Merely increasing the hard-coded compact
height would hide this exact screenshot but regress with font metrics, localized button
labels, conditional discard actions, or future footer content. Merely retrying the
measurement would still let the whole-root scroll view move the footer offscreen when
the height clamp binds.

## Design

### 1. Partition the panel into persistent and overflow regions

Refactor `CapturePanelView` into three explicit vertical regions:

1. A persistent top region containing the title-bar drag inset and
   `AutosizingCaptureEditor`.
2. An optional, vertically scrollable auxiliary region containing the completion list,
   destination summary, error message and its Retry/Copy Diagnostic controls, and the
   non-idle preview.
3. A persistent bottom region containing the status text and the primary
   discard/Preview/Capture controls.

Extract the current footer `HStack` into a small `CapturePanelFooter` view so the same
rendered object can be measured and assigned high vertical layout priority. The editor
and footer must use fixed/intrinsic vertical sizing and must sit **outside** the
auxiliary `ScrollView`; the scroll view is the only region allowed to absorb compression
when the screen-height cap binds. Do not use an overlay for the footer: overlays can
cover completion or preview content and recreate the problem in a less obvious form.

Render the auxiliary scroll view, and the spacing around it, only when at least one
auxiliary section exists. A fresh empty popup therefore remains a compact editor plus
footer with no empty middle viewport. Preserve the current section ordering and
completion/preview limits. Keep `.scrollBounceBehavior(.basedOnSize)` so the auxiliary
region behaves like ordinary content while it fits and becomes scrollable only when it
must.

The error block's conditional buttons remain next to their error message in the
auxiliary region. Under normal sizing the window grows to show the entire block. If the
screen clamp binds, use `ScrollViewReader`/stable section IDs (or the equivalent native
SwiftUI mechanism) to reveal a newly appearing error block so its message and actions
are immediately reachable instead of materializing below the viewport. Completion
navigation must continue to scroll the selected row into view.

### 2. Measure rendered regions; do not infer footer height

Replace the scalar `onIdealContentHeightChange` contract with an equatable, internal
value such as `CapturePanelContentMetrics` containing:

- `idealContentHeight`: the complete natural height of the persistent top, auxiliary
  content (when present), persistent footer, exact conditional spacing, and top/bottom
  padding.
- `minimumVisibleContentHeight`: the height that must be reserved for the persistent
  editor/footer regions and their required padding/spacing before any height is offered
  to the auxiliary scroll viewport.

Measure the actual laid-out top/editor region, auxiliary document, and footer with
`onGeometryChange` or preference keys, then combine them through a pure helper such as
`CapturePanelContentHeightPolicy`. The helper owns the spacing/padding arithmetic and is
unit-testable without a live window. It must count spacing only for sections that are
actually present and round consistently at the display scale.

The footer measurement must come from the rendered footer, not a constant. Remove
`footerEstimatedHeight`. Keep a conservative fallback content height only for the short
interval before the first valid metrics arrive; name it as a fallback, not an ideal
measurement, and make it large enough for the one-line editor plus the standard footer
at the supported font/control size. The fallback must remain far below the old 420-point
window and is a safety net, not the steady-state source of truth.

Do not publish geometry through `CapturePanelModel`: geometry is a view/controller
concern, and feeding it through `@Published` would make the model invalidate the view
that produced the measurement.

### 3. Make first measurement delivery replayable

In `CapturePanelController`, replace the one-shot application path with a small
receive/cache/apply handshake:

1. The hosting-view callback always stores the latest valid
   `CapturePanelContentMetrics`, even if the panel is not yet fully installed or a frame
   update is temporarily reentrant.
2. Assign/store the created panel before a hosting-view layout can report metrics, then
   install the root view and force an initial layout during prewarm.
3. After the content view is installed, explicitly apply the cached metrics if present.
   A valid report received too early is therefore deferred, not dropped.
4. On every `show()`, set the existing recenter intent, lay out the current model state,
   and explicitly replay the cached metrics. Do not depend on `.onGeometryChange`
   emitting an unchanged value a second time.
5. If no valid report exists yet, use the conservative compact fallback. Once a report
   arrives, replace the fallback target immediately. The panel must never momentarily
   use the 96-point degenerate floor as a user-visible compact size.

Keep the existing finite-value checks, half-point anti-oscillation threshold,
`isApplyingContentHeight` guard, unanimated frame changes, and first-application
recentering. Cache the latest report separately from the last height actually applied;
those are different states and conflating them would reintroduce the lost-measurement
bug.

### 4. Clamp without sacrificing the persistent controls

Update `CapturePanelWindowSizer` to resolve a report rather than an unqualified ideal
height, or add an explicit `minimumVisibleContentHeight` argument. Its target policy is:

- Start from the measured ideal height.
- Apply the existing global degenerate floor, pixel rounding, configured maximum, and
  screen margins.
- When the screen has enough room, never choose a target below the measured persistent
  minimum. A stale global floor such as 96 points must not override the actual
  editor/footer requirement.
- When the ideal height is above the available limit, give the remaining height to the
  auxiliary scroll viewport; the persistent top and bottom regions remain laid out at
  their intrinsic heights.
- Handle the mathematically impossible case where the screen itself is shorter than the
  persistent minimum deterministically: use the available screen height, keep the window
  on screen, and allow the outermost emergency overflow path to scroll. This is only a
  last-resort accessibility/screen constraint, not the ordinary layout.

Continue to preserve the window's top edge after presentation, keep it inside the
visible screen frame, and keep height content-owned while width remains user-resizable.
Width changes must trigger fresh region measurements because status wrapping, editor
wrapping, and control layout can legitimately change the ideal and persistent heights.

### 5. Preserve interaction and accessibility behavior

Keep all existing editor behavior: attributed highlighting, selection, undo, the
one-through-six-line growth policy, internal editor scrolling after six lines, and
Control-J/Shift-Return/Option-Return newline insertion. The completion menu must remain
below the editor, must not cover typed text or preview content, and must retain keyboard
selection scrolling.

Give the extracted footer and auxiliary scroll region stable accessibility grouping and
labels where necessary, but do not change the current button names or shortcuts. Verify
that VoiceOver focus does not jump when a height report is replayed and that the status
announcement focus hooks continue to operate from the extracted footer.

## Implementation steps

1. In `Sources/BobMacCapture/CapturePanelView.swift`, replace `footerEstimatedHeight`
   with a conservative pre-measurement fallback; add the pure content-height composition
   type(s) and the `CapturePanelContentMetrics` callback contract.
2. Split the root view into the persistent editor, optional scrollable auxiliary
   content, and extracted persistent footer. Measure their rendered heights and report
   composed metrics. Preserve conditional preview rendering and the current visual
   materials, padding, spacing, and completion behavior except where the region split
   requires an explicit background or separator for a clean stationary footer.
3. In `Sources/BobMacCapture/CapturePanelController.swift`, cache metrics before
   applying them, make panel installation order safe for early callbacks, force prewarm
   layout, and replay the latest metrics during panel creation and every `show()`.
4. In `Sources/BobMacCapture/CapturePanelWindowSizer.swift`, incorporate the measured
   persistent minimum into clamping while retaining the existing pixel rounding,
   maximum-height, screen-margin, and top-edge anchoring behavior.
5. Update `README.md`'s runtime contract to say that editor and primary actions are
   persistent, auxiliary content is the overflow region, and first presentation uses
   rendered metrics rather than a footer estimate.

## Automated verification

Extend `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` without weakening the
existing autosize, key-routing, or editor-height tests:

- Content composition: an empty state totals the measured editor, measured footer,
  required padding, and exactly one inter-region spacing; adding/removing auxiliary
  content adds/removes both its measured height and only the spacing it needs.
- Persistent minimum: changing the measured footer height (representing wrapping, larger
  controls, or future buttons) changes `minimumVisibleContentHeight` and the compact
  ideal; no test may depend on the old `28`-point estimate.
- Sizer policy: on a normal screen, an ideal below the persistent minimum resolves to
  that minimum; an oversized ideal clamps to the screen/maximum while leaving enough
  height for the persistent regions; an impossibly short screen takes the documented
  emergency path and remains on screen.
- Early report replay: deliver valid metrics before/during hosting-view installation,
  finish creating the panel, and assert that the measured target is eventually applied
  rather than lost.
- Show replay: applying a report, perturbing or resetting the panel to the fallback, and
  showing it again reapplies the unchanged cached report even though no new geometry
  change occurs.
- Idempotence/reentrancy: repeated equivalent reports do not move the frame; a newer
  report received while a resize is being applied remains cached and is not discarded.
- Existing frame anchoring and width-only resize tests continue to pass with the richer
  metrics contract.

Where practical on macOS, add a hosting-view layout regression test for the empty model
at the initial width: after layout, the reported compact ideal must be at least the
reported persistent minimum and larger than the fallback shown in the failing path.
Avoid assertions against private SwiftUI subview class names; the pure composition and
controller replay tests are the durable contract.

Run the repository's supported checks:

```sh
just format-lint
just build
just test
```

If the implementation host cannot select an Apple developer directory, report the exact
blocker and run only safe partial checks (such as `git diff --check` and Swift parse
checks) without claiming that the macOS suite passed.

## macOS interaction and visual verification

Verification on macOS 26 is required because this is ultimately an AppKit/SwiftUI layout
defect:

1. Install/restart a development build, open a fresh popup, and compare with
   `20260814_100746.png`. Confirm the complete Ready label and Preview/Capture buttons
   are visible on the first frame, not after a delayed resize, while the popup remains
   dramatically shorter than the former 420-point panel.
2. Prewarm the app, wait before opening, and open/close/reopen the unchanged empty panel
   several times. Confirm an unchanged geometry report is replayed and the footer never
   returns to the clipped fallback state.
3. Type through soft wraps and insert/delete hard newlines. Confirm the editor and
   window grow/shrink in lockstep, the footer stays visible, and the top edge/caret stay
   fixed.
4. Trigger and dismiss completion, preview loading/success/failure, destination summary,
   and capture error states. Confirm the window grows to show them when space permits;
   nothing is covered by the fixed footer; and a newly appearing error plus its actions
   is revealed if the auxiliary viewport must scroll.
5. Trigger discard confirmation and verify Ready/status text, Discard, Keep Draft,
   Preview, and Capture all remain fully inside the panel. Resize the width through its
   supported range and repeat so wrapped status text and alternate controls are measured
   rather than clipped.
6. Put the panel near the bottom of the display and exercise a maximum six-line editor,
   five completion rows, preview, and error. Confirm the panel stays on screen, editor
   and primary footer remain stationary, and only the auxiliary middle region scrolls.
7. Reopen a retained draft and then reopen after successful capture. Confirm both the
   retained large state and reset compact state use current measured metrics on their
   first visible frame.
8. Repeat the compact, error, and width-reflow checks with VoiceOver, Reduce Motion, and
   a second display scale if available. Confirm controls remain focusable and fully
   visible, announcements still fire, and there is no oscillation or animation flicker.

## Acceptance criteria

- The fresh panel's first visible frame contains the entire one-line editor,
  Ready/status text, and Preview/Capture buttons; the screenshot's clipped-footer state
  cannot recur merely because the first geometry callback was early or unchanged.
- The editor and primary footer are persistent regions. Completion, summary, error, and
  preview content never render behind them; when the height cap binds, auxiliary content
  scrolls instead of pushing primary controls outside the window.
- Window height is based on current rendered region measurements, including actual
  footer height and conditional spacing, not a 28-point footer estimate.
- Prewarm, first show, repeated show, retained-draft reopen, clearing, and width reflow
  all replay or refresh measurements reliably without feedback loops or jitter.
- Top-edge anchoring, on-screen clamping, width-only user resizing, editor autosizing,
  completion navigation, newline shortcuts, accessibility focus, and submission behavior
  remain intact.
- The compact panel remains substantially smaller than 420 points when empty, `bob-cli`
  is unchanged, the README describes the final layout contract, and all supported checks
  pass on macOS.
