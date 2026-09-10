---
tier: tale
title: Let the capture panel grow to the full height of the screen
goal:
  The bob-mac-capture panel grows with the draft up to the screen's visible frame minus
  margins, instead of stopping at the fixed 6-line editor cap and the fixed 720pt panel
  ceiling.
size: medium
proposed_by: bbugyi200.athena.07m
create_time: 2026-09-09 19:59:53
status: wip
---

# Let the capture panel grow to the full height of the screen

## Repository

All changes land in the **`bob-mac-capture`** linked repo, not in `bob-cli`. Open it
first and use the printed path as the only path for reads and writes:

```bash
sase repo open bob-mac-capture -r "Remove the capture panel's fixed height caps"
```

Every path below is relative to that repo root.

## Symptom

The `bob-mac-capture` panel is supposed to grow with the draft in the capture input box,
up to the height of the screen. In practice it stops growing at a fixed height well
below the screen: the editor starts scrolling internally and the top of the draft is
clipped out of view even though roughly two thirds of the display is empty.

Evidence (user screenshot `~/tmp/screenshots/20260819_084339.png`, 2874x1696 on a 2x
display, i.e. a 1437x848 pt screen): the editor's rounded material box measures ~149 pt
tall and the whole panel measures ~353 pt tall, on a screen with ~800 pt of usable
visible frame. The draft's first line is scrolled out of the editor viewport.

## Root cause

There are three stacked ceilings. Only the first one is binding today; the second binds
as soon as the first is lifted; the third is the structural gap that has to be closed
for a screen-height panel to actually lay out correctly.

### 1. `CaptureEditorHeightPolicy` hard-caps the editor at six nominal lines (binding)

`Sources/BobMacCapture/CapturePanelView.swift`

```swift
static let editorLineHeight: CGFloat = 22
static let editorMaximumVisibleLines = 6
static let editorContentPadding: CGFloat = 10
```

```swift
var maximumHeight: CGFloat {
    roundedToPixel(lineHeight * CGFloat(maximumVisibleLines) + verticalPadding)
}

func resolvedHeight(forMeasuredTextHeight measuredTextHeight: CGFloat) -> CGFloat {
    let measuredHeight = max(measuredTextHeight, lineHeight)
    let unclampedHeight = measuredHeight + verticalPadding
    return roundedToPixel(min(max(unclampedHeight, minimumHeight), maximumHeight))
}
```

`maximumHeight` evaluates to `22 * 6 + 20 = 152` pt — a compile-time constant with no
relationship to the screen. `AutosizingCaptureEditor` measures the real text height with
a hidden sizing `Text`, feeds it through `resolvedHeight(forMeasuredTextHeight:)`, and
pins the editor with `.frame(height: editorHeight)`. Once the measured text exceeds 132
pt the editor stops growing and the inner `TextEditor` scrolls. The panel then stops
growing too, because `CapturePanelContentHeightPolicy.metrics` derives the panel's ideal
content height from that already-clamped editor height.

152 pt matches the ~149 pt measured in the screenshot. **This is the cap the user is
hitting.**

Note also that the `22` pt `editorLineHeight` constant is a stale proxy: the editor
renders `.system(.body, design: .monospaced)`, whose line height is ~16 pt at the
default text size, which is why the screenshot shows ~9 rows inside a nominally "6 line"
box. The cap should not be expressed in nominal lines at all.

### 2. `CapturePanelWindowSizer` applies a fixed 720 pt ceiling (binds next)

`Sources/BobMacCapture/CapturePanelWindowSizer.swift`

```swift
var maximumContentHeight: CGFloat = CapturePanelLayout.panelMaximumContentHeight  // 720
...
var clamped = min(max(metrics.idealContentHeight, persistentMinimum), maximumContentHeight)
```

The screen clamp (`availableScreenHeight - 2 * screenMargin`) is applied _after_ and _in
addition to_ this constant, so `min(720, screenLimit)` wins. On the user's ~800 pt
visible frame the screen limit is ~752 pt, so 720 would cap the panel ~32 pt short even
after root cause 1 is fixed; on a taller external display the gap is much larger.

### 3. No screen budget reaches the SwiftUI layer, so the window clamp cannot degrade gracefully

The screen limit lives entirely in `CapturePanelController` / `CapturePanelWindowSizer`.
`CapturePanelView` has no knowledge of it. That is fine today only because the 152 pt
editor cap guarantees the content is always small. Remove that cap without plumbing a
budget and the layout breaks in two ways:

- `CapturePanelContentHeightPolicy.metrics` folds the full editor height into
  `minimumVisibleContentHeight`. When that exceeds the screen limit,
  `CapturePanelWindowSizer.contentHeight` takes the `screenLimit < persistentMinimum`
  branch and returns `screenLimit`. The window is then smaller than the content SwiftUI
  laid out, so content is clipped rather than scrolled.
- Inside `CapturePanelView.content`, the editor carries `.layoutPriority(2)` and
  `.fixedSize(horizontal: false, vertical: true)` while `auxiliaryScrollRegion` carries
  `.layoutPriority(0)`. An unbounded editor therefore squeezes the auxiliary region
  (completion list, destination summary, live preview, error) to zero before it yields
  any space itself — regressing the documented contract in `README.md` that the middle
  region is the part that scrolls when the clamp binds.

So the editor's maximum height must become a **budget derived from the available screen
height**, not a constant: the screen limit minus the panel's persistent non-editor
chrome minus a reserve for the auxiliary region.

## Design

Push the screen height budget one level down, from the controller into the view, and let
the editor size itself against it.

- `CapturePanelController` already knows `panel.screen?.visibleFrame`. Publish that
  available screen height to `CapturePanelModel` so `CapturePanelView` can read it.
- `CapturePanelView` converts the screen height into an editor height budget using the
  layout constants it already owns plus the measured footer and auxiliary heights, and
  hands that budget to `CaptureEditorHeightPolicy` as an explicit `maximumHeight`.
- `CapturePanelWindowSizer`'s fixed ceiling becomes a fallback used only when the screen
  height is unknown, so on a real screen the screen limit is the sole ceiling.
- Everything stays in the existing pure-value-type style: the budget math is a
  `struct`/`enum` with no `NSWindow` access, so it is unit-testable exactly like
  `CapturePanelWindowSizer` and `CapturePanelContentHeightPolicy` are today.

Priority order when the budget binds, preserving the README contract: footer and editor
stay visible; the editor stops growing at its budget and scrolls internally; the
auxiliary region keeps at least its reserved minimum and scrolls.

## Implementation steps

### Step 1 — Publish the available screen height from the controller

`Sources/BobMacCapture/CapturePanelController.swift`,
`Sources/BobMacCapture/CapturePanelModel.swift`

1. Add an `@Published var availableScreenHeight: CGFloat?` (or equivalently named
   property) to `CapturePanelModel`, defaulting to `nil`.
2. In `CapturePanelController`, add a private `updateAvailableScreenHeight()` that reads
   `panel.screen?.visibleFrame.height ?? NSScreen.main?.visibleFrame.height` and assigns
   it to the model only when the value actually changes (guard against redundant
   `objectWillChange` churn, which would re-trigger a metrics report and resize).
3. Call it from:
   - `makePanelIfNeeded()`, immediately after the hosting view is installed and before
     `applyLatestContentMetricsIfPossible()`, so the first layout already has a budget;
   - `replayLatestContentMetricsForPresentation()`, before the layout/apply, so a panel
     re-shown on a different display re-budgets;
   - `applyContentMetrics(_:force:)`, where `visibleFrame` is already resolved — reuse
     that same value rather than reading the screen twice;
   - a new `windowDidChangeScreen(_:)` `NSWindowDelegate` callback, so dragging the
     panel between displays re-budgets.
4. Confirm no feedback loop: the budget depends only on the screen, never on the applied
   window height, so `model → view → metrics → controller → model` cannot oscillate. The
   existing `isApplyingContentHeight` / `metricsArrivedDuringApplication` reentrancy
   guard in `applyContentMetrics` still covers metrics that arrive mid-resize.

### Step 2 — Make the editor's maximum height a budget, not a constant

`Sources/BobMacCapture/CapturePanelView.swift`

1. Change `CaptureEditorHeightPolicy` so its ceiling is injected:
   - Replace `maximumVisibleLines: Int` with an explicit `maximumHeight: CGFloat?`
     (`nil` = unbounded), or keep a stored `maximumHeight` with the old six-line value
     as the default only for the no-budget case. Either way `resolvedHeight(...)` must
     clamp to the injected ceiling and must never clamp below `minimumHeight`.
   - `visibleLineCount(forMeasuredTextHeight:)` is used only by tests today
     (`grep -rn "visibleLineCount"` to confirm before touching it). Either delete it or
     re-derive it from the injected ceiling; do not leave it reporting a stale 6.
2. Add a small pure type — e.g. `CaptureEditorHeightBudget` — next to the other layout
   policy structs that answers: _given the available screen height, the measured footer
   height, and the auxiliary state, how tall may the editor be?_ Recommended formula:

   ```
   screenLimit    = availableScreenHeight - 2 * CapturePanelLayout.panelScreenMargin
   nonEditorChrome = titlebarDragInset
                   + sectionSpacing * (auxiliary == nil ? 1 : 2)
                   + footerHeight
                   + rootPadding
   auxiliaryReserve = auxiliary == nil ? 0 : min(auxiliaryIdealHeight, auxiliaryFloor)
   editorMaximum   = max(minimumEditorHeight, screenLimit - nonEditorChrome - auxiliaryReserve)
   ```

   Keep the `nonEditorChrome` expression in sync with
   `CapturePanelContentHeightPolicy.metrics` — the two must agree on the spacing count
   or the panel will over- or under-shoot the screen limit by one `sectionSpacing`.
   Prefer factoring the shared `persistentHeight` arithmetic into one place used by
   both.

3. Add `CapturePanelLayout.auxiliaryReservedHeight` (a new constant; ~88 pt is a
   reasonable starting point — roughly a two-line destination summary plus spacing).
   Bound it by the auxiliary region's own ideal height so a short auxiliary never
   over-reserves, and by the stash picker's existing `minimumVisibleHeight` when the
   stash picker is the auxiliary content — that path already has a correct floor and
   must keep it.
4. `AutosizingCaptureEditor` currently builds its own
   `CaptureEditorHeightPolicy(displayScale:)`. It must instead receive the budgeted
   policy (or the budget) from `CapturePanelView`, which is the view that knows the
   measured footer height, the auxiliary state, and `model.availableScreenHeight`.
5. When `model.availableScreenHeight` is `nil` (headless unit tests, no screen), fall
   back to the existing behavior rather than unbounded growth: use
   `CapturePanelLayout.panelMaximumContentHeight` as the ceiling. Do not fall back to
   the six-line constant.
6. Ensure the editor height recomputes when the budget changes: add
   `.onChange(of: model.availableScreenHeight)` and
   `.onChange(of: measuredFooterHeight)` handling so a screen change re-resolves the
   editor height and re-reports metrics.

### Step 3 — Stop the sizer from imposing an artificial ceiling on a real screen

`Sources/BobMacCapture/CapturePanelWindowSizer.swift`

1. Change `maximumContentHeight` to `CGFloat?` defaulting to `nil` (no artificial cap),
   and apply it only when non-`nil`. Keep the property so the existing
   `maximumContentHeight: 500` unit tests remain expressible.
2. In `CapturePanelController.applyContentMetrics`, construct the sizer with
   `maximumContentHeight: visibleFrame == nil ? CapturePanelLayout.panelMaximumContentHeight : nil`
   so the 720 pt constant survives strictly as the no-screen fallback. Update the
   constant's doc comment to say so; `panelMaximumContentHeight` is no longer a
   steady-state ceiling.
3. Fix the sub-pixel rounding leak in `contentHeight(...)`: it ends with
   `roundedToPixel(clamped)`, which rounds `.up` and can therefore return a value a
   fraction of a point _above_ `screenLimit`. Round the screen limit down to the pixel
   grid before clamping, or round the final result down when the screen limit binds.
4. `applyContentMetrics` calls `panel.center()` on the `pendingRecenter` path.
   `NSWindow.center()` biases toward the upper third and can push a screen-height panel
   partly off-screen. After centering, run the resulting frame through
   `sizer.frame(forCurrentFrame:contentHeight:chromeHeight:visibleFrame:)` (or clamp it
   into `visibleFrame` directly) so a full-height panel is always fully on-screen.

### Step 4 — Tests

`Tests/BobMacCaptureTests/BobMacCaptureTests.swift`

1. `testEditorHeightPolicyUsesOneLineMinimumAndSixLineCap` (line ~910) asserts the exact
   behavior being removed. Rewrite it as a budget test: one-line minimum preserved,
   growth tracks measured text, clamped at the injected `maximumHeight`, never clamped
   below `minimumHeight`, pixel rounding preserved at `displayScale: 2`.
2. New tests for the budget type:
   - a tall screen yields an editor maximum far above the old 152 pt;
   - the budget shrinks by exactly `sectionSpacing + auxiliaryReserve` when auxiliary
     content is present;
   - a short screen (e.g. 400 pt available) still returns at least the one-line minimum
     and never a negative or sub-minimum budget;
   - `availableScreenHeight == nil` falls back to `panelMaximumContentHeight`.
3. New sizer tests:
   - with `maximumContentHeight: nil` and a tall `availableScreenHeight`, a very large
     ideal height resolves to exactly `availableScreenHeight - 2 * screenMargin` (and
     never a hair above it, covering the rounding fix);
   - the existing `maximumContentHeight: 500` tests still pass unchanged in behavior.
4. New end-to-end policy test: compose `CaptureEditorHeightBudget` →
   `CaptureEditorHeightPolicy` → `CapturePanelContentHeightPolicy.metrics` →
   `CapturePanelWindowSizer.contentHeight` for a very long draft on a tall screen, and
   assert the resolved content height equals the screen limit — i.e. the editor consumes
   the whole budget and the window is exactly screen-limited, with no clipping headroom
   lost.
5. Keep `testReceiveContentMetricsGrowsAndKeepsTopEdge`,
   `testReceiveContentMetricsIsIdempotentForARepeatedMeasurement`, and
   `testReceiveStashContentMetricsGrowsFromCompactAndShrinksAfterDismissal` green; they
   pin the top-edge anchoring, idempotence, and stash-picker behavior that must not
   regress.

### Step 5 — Documentation

`README.md`, panel bullet at lines ~84-100.

Update the description so it states the real contract: the editor grows with the draft
up to a screen-derived budget rather than a fixed line count; the window's ceiling is
the screen's visible frame minus margins; when the budget binds the editor scrolls
internally, the auxiliary region keeps its reserved minimum and scrolls, and the footer
actions always stay visible.

## Verification

**These targets are macOS-only.** `Package.swift` declares `platforms: [.macOS("26.0")]`
and `Scripts/xcode-swift.sh` resolves Swift through `/usr/bin/xcrun` against a macOS 26+
SDK, so `just test` / `just build` cannot run on a Linux host — they fail at toolchain
resolution before compiling anything. Run the automated checks on the Mac:

```bash
just format-lint
just build
just test
```

If the implementing agent is running on Linux, it must say so explicitly in its final
report and hand the build/test/manual verification to the user rather than claiming the
change is verified.

Manual check on the Mac after `just install`:

1. Open the panel with the hotkey and confirm a fresh popup is still the compact
   one-line Spotlight-style bar (no regression in the empty state).
2. Paste or type a draft far longer than the screen. The panel must grow smoothly past
   the old ~350 pt total height, stay anchored at its top edge, and stop exactly at the
   screen's visible frame minus the 24 pt margins — not before.
3. With that long draft, trigger the live preview (`Preview`) and a route completion.
   Both must remain visible in the scrollable auxiliary region; the footer buttons must
   stay on-screen and clickable.
4. Drag the panel to a display of a different height and confirm it re-budgets to the
   new screen instead of keeping the old ceiling.
5. Open the stash picker (Ctrl-S) with a long draft and confirm the fixed **Shift-D
   Delete All** action is still preserved ahead of row height.

## Acceptance criteria

- No fixed constant limits the editor or the panel to a height below the screen when a
  screen is known. `editorMaximumVisibleLines` is gone (or no longer participates in the
  ceiling), and `panelMaximumContentHeight` is reachable only when the screen height is
  unknown.
- With a draft long enough to overflow, the panel's content height equals
  `visibleFrame.height - 2 * panelScreenMargin`, exactly, with no sub-pixel overshoot.
- The auxiliary region (completion, destination, preview, error) and the footer remain
  visible at every panel height.
- The panel stays fully inside `visibleFrame` on every path, including the recenter
  path.
- `just format-lint`, `just build`, and `just test` all pass on macOS.

## Out of scope

- Making the panel height user-draggable. `windowWillResize` deliberately pins the
  height to `appliedContentHeight` and `README.md` documents height as content-owned;
  leave that contract alone.
- Changing the editor font, `editorLineHeight` as a _minimum_-height input, or any
  capture grammar / `bob` subprocess contract.
- Any change in `bob-cli` itself. This is a pure `bob-mac-capture` presentation fix; the
  capture JSON interface is untouched.

## Risks

- **Layout-priority regression.** Removing the editor's ceiling changes how SwiftUI
  distributes space between the editor (`layoutPriority(2)`) and the auxiliary region
  (`layoutPriority(0)`). If the auxiliary reserve is miscomputed, the preview pane will
  silently collapse to zero height instead of scrolling. Manual verification step 3 is
  the guard for this — do not skip it.
- **Resize feedback loop.** The metrics pipeline is
  `view measure → onContentMetricsChange → controller → setFrame → view re-measure`. The
  budget must depend only on the _screen_, never on the applied window height, or the
  panel will oscillate. Assign `availableScreenHeight` only on an actual change.
- **Spacing-count drift.** `CaptureEditorHeightBudget` and
  `CapturePanelContentHeightPolicy.metrics` must agree on
  `sectionSpacing * spacingCount`. If they disagree the panel will consistently miss the
  screen limit by one spacing unit; the Step 4.4 end-to-end test is what catches this.
