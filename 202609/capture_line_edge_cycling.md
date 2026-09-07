---
tier: tale
title: Cycle Ctrl-A/Ctrl-E across physical lines in the capture editor
goal:
  In the Bob Mac Capture editor, Ctrl-A and Ctrl-E move to the beginning/end of the
  current physical line, and when the caret already sits on that edge they step to the
  beginning of the previous line / end of the next line, stopping at the first and last
  line.
size: medium
proposed_by: bbugyi200.athena.01e
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.toobig-4y.registry.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-4y.registry.0/README.md)
- **COMMITS:**
  - [07f44fc](https://github.com/sase-org/sase/commit/07f44fc905f491da7be8b286cca1dbc1d9f043cc)
    — refactor(agent-names): split registry facade modules

# Plan: Cycle Ctrl-A/Ctrl-E across physical lines in the capture editor

## 1. Where this work happens

All changes land in the **`bob-mac-capture`** linked repo. Open it first with your
`/sase_repo` skill and use only the path that command prints:

```bash
sase repo open bob-mac-capture -r "Implement Ctrl-A/Ctrl-E physical-line cycling in the capture editor"
```

No change is needed in `bob-cli` itself; this is purely an editor keymap change in the
macOS app.

Files touched:

| File                                                  | Change                                                                                       |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` | Two new key codes, two new `CaptureKeyCommand` cases, two new editor-context switch cases    |
| `Sources/BobMacCapture/CapturePanelController.swift`  | New `CaptureLineEdge` enum, new pure resolver, new apply helper, two new `perform(_:)` cases |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`   | New router tests, resolver tests, and one live-`NSTextView` apply test                       |
| `README.md`                                           | Two new rows in the Keyboard table plus a prose paragraph                                    |

## 2. Current behavior (verified, not assumed)

- `CaptureKeyCommandRouter` does **not** mention key codes `0` (`a`) or `14` (`e`)
  today. Both fall through `default: return nil`, so `CapturePanelController`'s
  `NSEvent.addLocalMonitorForEvents` monitor returns the event unhandled and AppKit's
  standard key bindings handle it: `^a` -> `moveToBeginningOfParagraph:`, `^e` ->
  `moveToEndOfParagraph:`. That is why "they already work" today.
- Because native paragraph movement is a no-op once the caret is already on the
  paragraph edge, pressing Ctrl-A twice currently does nothing the second time. Closing
  that gap is the entire point of this plan.
- The editor is a SwiftUI `TextEditor` (`AutosizingCaptureEditor` in
  `Sources/BobMacCapture/CapturePanelView.swift`), but every existing keymap that needs
  the caret reaches the backing `NSTextView` through
  `CapturePanelController.editableTextView(_:)` on the panel's first responder. Ctrl-J,
  Ctrl-U, Tab/Shift-Tab, and the placeholder Backspace all do this, and they set the
  caret with `textView.setSelectedRange(_:)`. Follow that same path.
- Caret movement already feeds the model: `AutosizingCaptureEditor` calls
  `model.editorSelectionDidChange(cursorUTF8Offset:)` on every selection change, which
  calls `scheduleAnalysis(cursorUTF8Offset:requestCompletion:)`. Nothing extra is needed
  to keep parse/preview/completion in sync after a caret move.
- Every line-aware helper in this codebase (`CaptureBulletNewlineEditResolver.resolve`,
  `bulletIndentationEdit`, `emptyBulletRowDeletionRange`) uses
  `NSString.getLineStart(_:end:contentsEnd:for:)`, i.e. **physical** lines. The README
  already describes Ctrl-U in those terms ("the current physical line").

## 3. Design decisions

These are decisions, not open questions. Each records the alternative so review can flip
one cheaply.

1. **Physical lines, not visual (soft-wrapped) lines.** A "line" here is a paragraph as
   `NSString.getLineStart` defines it. This matches Ctrl-U, Ctrl-J, and Tab indentation,
   and keeps the resolver a pure function with no layout-manager dependency.
   _Alternative:_ visual-line semantics via `NSLayoutManager.lineFragmentRange`;
   rejected because it is untestable as a pure function and inconsistent with every
   sibling keymap.

2. **Stop at the document edges; do not wrap around.** Ctrl-A on the first line at
   column zero leaves the caret where it is. Ctrl-E at the very end of the draft leaves
   the caret where it is. Wrap-around in this app is deliberately reserved for _list
   selection_ (completion rows, stash rows), where it is cheap and reversible; silently
   teleporting a text caret from the top of a draft to the bottom is a surprise the next
   keystroke turns into misplaced text. _Alternative:_ wrap Ctrl-A from the first line
   to the start of the last line and Ctrl-E from the last line to the end of the first.
   If review wants that, it is a two-line change in section 4.2 (replace the two
   `return nil` guards with the wrapped target) plus four test cases.

3. **The app owns the whole Ctrl-A/Ctrl-E behavior, including the ordinary
   move-to-edge.** The alternative -- intercept only when the caret is already on the
   edge and otherwise return `false` so AppKit handles the plain move -- is closer to
   repo idiom (`applyBulletIndentation` does exactly that), but it has a trap: if
   AppKit's `^a` ever lands the caret on a _visual_ line start that is not a physical
   line start, our edge detection would say "not on the edge", fall through, and native
   would no-op -- the user gets stuck and can never step to the previous line. Owning
   both branches makes the behavior deterministic and fully unit-testable. This is safe
   because the repo's "keep it native" rule (see the `insertNewline`/Ctrl-U comments) is
   about _text mutation_ -- undo coalescing, IME composition, accessibility edit
   notifications -- and none of those apply to a pure caret move.

4. **Collapsed carets only.** With a non-empty selection the resolver declines and the
   event falls through to AppKit, which collapses the selection to the paragraph edge
   exactly as it does today. This composes correctly: the first press collapses via
   AppKit, the second press engages line cycling.

5. **Exactly `.control`, no other modifier.** `Ctrl-Shift-A`/`Ctrl-Shift-E` (native
   selection extension), `Cmd-A` (the Select All menu item in `AppDelegate.swift`),
   `Cmd-E`, `Ctrl-Opt-A`, and `Ctrl-Cmd-A` must all stay unrouted. Use
   `modifiers == .control`, matching Ctrl-J/Ctrl-U/Ctrl-S/Ctrl-C. (Consequence,
   inherited from those siblings and intentionally not fixed here: with Caps Lock
   engaged `modifiers` is `[.control, .capsLock]` and the binding does not fire.)

6. **Do not dismiss completion.** Ctrl-J and Ctrl-U call `model.dismissCompletion()`
   because they mutate text. A caret move must not: today, native Ctrl-A with the
   completion list open re-anchors completion at the new caret through
   `editorSelectionDidChange`. Preserve that. Do not copy the `dismissCompletion()` line
   from the sibling handlers.

7. **Prompts and the stash picker are untouched.** `command(for:context:)` returns early
   into `taskIDPromptCommand` / `pomodoroNamePromptCommand` / `stashPickerCommand`
   before reaching the editor switch, so the two new cases cannot leak into those modes.
   The inline prompts are single-line `NSTextField`s and keep native Ctrl-A/Ctrl-E. Lock
   this in with regression assertions (section 5.1) rather than leaving it implicit.

## 4. Implementation

### 4.1 `CaptureKeyCommandRouter.swift`

Add two cases to `CaptureKeyCommand` (place them next to the other editing commands, for
example after `deleteToBeginningOfLine`):

```swift
case moveToBeginningOfLineOrPreviousLine
case moveToEndOfLineOrNextLine
```

Add two entries to the private `KeyCode` enum:

```swift
static let a: UInt16 = 0
static let e: UInt16 = 14
```

Add two cases to the editor switch inside `command(for:context:)` (put them next to
`case KeyCode.u:` so the Ctrl-family bindings stay together):

```swift
case KeyCode.a:
    return modifiers == .control ? .moveToBeginningOfLineOrPreviousLine : nil
case KeyCode.e:
    return modifiers == .control ? .moveToEndOfLineOrNextLine : nil
```

Make no change to `isolatedPromptCommand`, `stashPickerCommand`, or
`printableModalCommand`.

### 4.2 `CapturePanelController.swift`

Add the direction enum beside the existing `CaptureBulletIndentationDirection` near the
top of the file:

```swift
/// Which edge of a physical line Ctrl-A / Ctrl-E targets.
enum CaptureLineEdge {
    case beginning
    case end
}
```

Add the pure resolver as a `nonisolated static` member of `CapturePanelController`,
alongside `bulletIndentationEdit`. Mark it `nonisolated` for the same reason that helper
is (commit `fc1c16b`): the type is `@MainActor`, and the resolver must stay callable off
the main actor.

```swift
/// Ctrl-A / Ctrl-E target for a collapsed caret. Returns the new caret location, or
/// `nil` when the key should fall through to AppKit: a non-collapsed selection, an
/// out-of-bounds selection, or a caret already on the requested edge of the first
/// (`.beginning`) or last (`.end`) physical line, where there is no line to step to.
///
/// "Line" means a physical line as `NSString.getLineStart(_:end:contentsEnd:for:)`
/// defines it, matching Ctrl-U, Ctrl-J, and Tab bullet indentation. Working from
/// `lineStart` / `contentsEnd` / `lineEnd` rather than from raw offsets keeps this
/// correct for CRLF terminators and for a draft that ends in a newline.
nonisolated static func lineEdgeCyclingLocation(
    _ edge: CaptureLineEdge,
    in text: NSString,
    selectedRange: NSRange
) -> Int? {
    guard selectedRange.location >= 0,
          selectedRange.length == 0,
          selectedRange.location <= text.length
    else {
        return nil
    }

    var lineStart = 0
    var lineEnd = 0
    var contentsEnd = 0
    text.getLineStart(
        &lineStart,
        end: &lineEnd,
        contentsEnd: &contentsEnd,
        for: NSRange(location: selectedRange.location, length: 0)
    )

    switch edge {
    case .beginning:
        if selectedRange.location > lineStart {
            return lineStart
        }
        // A previous line exists exactly when this one does not start the draft.
        guard lineStart > 0 else {
            return nil
        }
        var previousLineStart = 0
        text.getLineStart(
            &previousLineStart,
            end: nil,
            contentsEnd: nil,
            for: NSRange(location: lineStart - 1, length: 0)
        )
        return previousLineStart
    case .end:
        if selectedRange.location < contentsEnd {
            return contentsEnd
        }
        // A next line exists exactly when this one carries a terminator; the draft's
        // final line has `contentsEnd == lineEnd`.
        guard contentsEnd < lineEnd else {
            return nil
        }
        let nextLineStart = lineEnd
        // A draft ending in a newline has an empty final line whose start, contents
        // end, and end all equal the length.
        guard nextLineStart < text.length else {
            return nextLineStart
        }
        var nextContentsEnd = 0
        text.getLineStart(
            nil,
            end: nil,
            contentsEnd: &nextContentsEnd,
            for: NSRange(location: nextLineStart, length: 0)
        )
        return nextContentsEnd
    }
}
```

Add the apply helper next to `deleteToBeginningOfLineInEditableTextView`. It
deliberately takes no `CapturePanelModel`: see decision 6.

```swift
/// Ctrl-A / Ctrl-E: move the caret to a physical-line edge, stepping to the adjacent
/// line when it is already there. Returns `false` without changing state whenever
/// `lineEdgeCyclingLocation` declines, so the key event falls through to AppKit's
/// native paragraph movement. This never dismisses completion: a caret move is not an
/// edit, and `editorSelectionDidChange` re-anchors the completion list at the new
/// caret on its own.
static func moveLineEdge(
    _ edge: CaptureLineEdge,
    firstResponder: NSResponder?
) -> Bool {
    guard let textView = editableTextView(firstResponder),
          let location = lineEdgeCyclingLocation(
            edge,
            in: textView.string as NSString,
            selectedRange: textView.selectedRange()
          )
    else {
        return false
    }

    let target = NSRange(location: location, length: 0)
    textView.setSelectedRange(target)
    textView.scrollRangeToVisible(target)
    return true
}
```

`scrollRangeToVisible` matters: the editor scrolls internally past six visual lines
(README "Keyboard" section), native paragraph movement scrolls the caret back into view
automatically, and a bare `setSelectedRange` does not.

Add the two cases to `perform(_:)`; its switch is exhaustive with no `default`, so the
compiler will point at it:

```swift
case .moveToBeginningOfLineOrPreviousLine:
    return Self.moveLineEdge(.beginning, firstResponder: panel?.firstResponder)
case .moveToEndOfLineOrNextLine:
    return Self.moveLineEdge(.end, firstResponder: panel?.firstResponder)
```

## 5. Tests

All new tests go in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, which already
holds the router and `NSTextView` helper tests and has the private
`keyEvent(keyCode:modifiers:characters:)` factory.

### 5.1 Router coverage

Add `testKeyRouterMapsControlLineEdgeMovement()`:

- `keyCode: 0, modifiers: .control` -> `.moveToBeginningOfLineOrPreviousLine`
- `keyCode: 14, modifiers: .control` -> `.moveToEndOfLineOrNextLine`
- The same two with `completionVisible: true` -> the same commands (decision 6: the
  binding must not change while completion is open).
- `nil` for each of: `keyCode: 0` unmodified; `keyCode: 14` unmodified;
  `keyCode: 0, modifiers: .command` (Select All must survive);
  `keyCode: 14, modifiers: .command`; `[.control, .shift]` on both (native selection
  extension); `[.control, .option]` on both; `[.control, .command]` on both.
- With `CaptureKeyRoutingContext(taskIDPromptVisible: true)` and again with
  `pomodoroNamePromptVisible: true`: `nil` for Ctrl-A and Ctrl-E.
- With `CaptureKeyRoutingContext(stashPickerVisible: true, stashEntryCount: 2)`: `nil`
  for Ctrl-A and Ctrl-E. Pass realistic control characters here --
  `characters: "\u{01}"` for Ctrl-A and `"\u{05}"` for Ctrl-E -- so the assertion
  actually exercises `printableModalCommand`'s `CharacterSet.controlCharacters` guard
  rather than passing only because the default `characters` is empty.

### 5.2 Resolver coverage

Add `testLineEdgeCyclingLocationStepsAcrossPhysicalLines()` calling
`CapturePanelController.lineEdgeCyclingLocation(_:in:selectedRange:)` directly, in the
style of the existing `bulletIndentationEdit` tests. Every expectation below is
hand-computed against `NSString` offsets; assert them exactly.

The resolver body in section 4.2 and every expectation in this section were compiled and
executed against swift-corelibs-Foundation's `NSString` while this plan was written: all
24 cases below produce exactly the stated results. Treat a disagreement as a
transcription mistake in your port, not as a wrong expectation.

`"one\ntwo\nthree"` (length 13; line starts 0/4/8, contents ends 3/7/13, line ends
4/8/13):

| Caret | Edge         | Expected                                           |
| ----- | ------------ | -------------------------------------------------- |
| 6     | `.beginning` | `4` (ordinary move to line start)                  |
| 4     | `.beginning` | `0` (already at line start -> previous line start) |
| 8     | `.beginning` | `4`                                                |
| 0     | `.beginning` | `nil` (first line, no wrap)                        |
| 5     | `.end`       | `7` (ordinary move to line end)                    |
| 7     | `.end`       | `13` (already at line end -> next line end)        |
| 3     | `.end`       | `7`                                                |
| 13    | `.end`       | `nil` (last line, no wrap)                         |

Add `testLineEdgeCyclingLocationHandlesEdgeCaseDrafts()` for the shapes that break naive
offset arithmetic:

- Empty draft `""`, caret `0`: `nil` for both edges.
- Trailing newline `"a\n"` (length 2): caret `1`, `.end` -> `2` (the empty final line);
  caret `2`, `.end` -> `nil`; caret `2`, `.beginning` -> `0`.
- Blank middle line `"a\n\nb"` (length 4): caret `2`, `.beginning` -> `0`; caret `2`,
  `.end` -> `4`; caret `1`, `.end` -> `2`.
- CRLF `"a\r\nb"` (length 4): caret `1`, `.end` -> `4` (steps over both terminator
  characters); caret `3`, `.beginning` -> `0`.
- Indented bullet `"- a\n  - b"` (length 9): caret `6`, `.beginning` -> `4`, proving
  column zero rather than "first non-whitespace" -- there is no smart-home behavior
  here; caret `4`, `.beginning` -> `0`.
- Declines: `NSRange(location: 4, length: 3)` -> `nil` for both edges (non-collapsed);
  `NSRange(location: 99, length: 0)` -> `nil` for both edges (out of bounds).

### 5.3 Live text-view coverage

Add `@MainActor func testMoveLineEdgeCyclesCaretInEditableTextView()`, modeled on the
existing `testEmptyBulletRowDeletionRange*` tests
(`let textView = NSTextView(frame: .zero)`; `NSTextView` is editable by default, which
`editableTextView(_:)` requires):

- `textView.string = "one\ntwo\nthree"`, caret at `6`.
  `moveLineEdge(.beginning, firstResponder: textView)` returns `true` and leaves
  `selectedRange() == NSRange(4, 0)`. Calling it again returns `true` and leaves
  `NSRange(0, 0)`. A third call returns `false` and leaves `NSRange(0, 0)` unchanged.
- Same view, caret at `5`: `.end` returns `true` -> `NSRange(7, 0)`, again `true` ->
  `NSRange(13, 0)`, a third time `false` with the caret still at `13`.
- `moveLineEdge(.beginning, firstResponder: nil)` returns `false`.

That first bullet is the regression test for the actual feature request: repeated
presses walk the caret up line by line and then stop.

## 6. Documentation

In `README.md`, in the "Keyboard" table (the one whose columns are `In the editor` /
`While completion is visible` / `While Add block ID is open` /
`While Name Pomodoro is open`), add two rows immediately after the `Ctrl-U` row:

```
| Ctrl-A | Move to the beginning of the current physical line; when the caret is already there, move to the beginning of the previous line, stopping on the first line | Same move, leaving completion open and re-anchored at the new caret | Native text-field behavior | Native text-field behavior |
| Ctrl-E | Move to the end of the current physical line; when the caret is already there, move to the end of the next line, stopping on the last line | Same move, leaving completion open and re-anchored at the new caret | Native text-field behavior | Native text-field behavior |
```

In the editing prose paragraph that currently ends with "Ctrl-U deletes from the caret
to the beginning of the current physical line. Backspace on an empty `- ` row removes it
in one action instead of requiring two ordinary backspaces. All five shortcuts act on
the native text view directly...", add a sentence for the new binding and fix the
now-stale count. Two things to get right:

- "All five shortcuts" enumerates the shortcuts that act on the native text view. Ctrl-A
  and Ctrl-E now do too, but they only move the caret -- they never edit -- so rather
  than bumping the number and implying they carry the same undo/IME story, split them
  out. Suggested wording to append to that paragraph:

  > Ctrl-A and Ctrl-E move the caret to the beginning and end of the current physical
  > line, and when the caret is already on that edge they step to the beginning of the
  > previous line or the end of the next one, so repeated presses walk the draft line by
  > line. They stop at the first and last line rather than wrapping around, and because
  > they only move the caret they never touch the draft's text, undo history, or an
  > in-flight IME composition.

- Leave the "All five shortcuts" sentence's count alone if you keep Ctrl-A/Ctrl-E out of
  that enumeration, as the wording above does. Re-read the paragraph after editing and
  confirm it still reads correctly.

Do not add rows to the separate "Key while stash is open" table: Ctrl-A and Ctrl-E are
deliberately not routed there.

## 7. Validation

Be honest about what can and cannot run. This package's app target is macOS-only
(`Package.swift` compiles `BobMacCapture` and `BobMacCaptureTests` only under
`#if os(macOS)`), and `Scripts/xcode-swift.sh` -- which `just build` / `just test` /
`just format-lint` all go through -- requires `xcode-select`. On a Linux host,
`just build` and `just test` cannot compile or run any of the code this plan changes.

**What to run locally (Linux host, from the opened `bob-mac-capture` checkout):**

```bash
export PATH="$HOME/.local/share/swiftly/bin:$PATH"

# 1. Syntax gate. Parses the macOS-only sources and tests without resolving AppKit or
#    SwiftUI, so it works on Linux. Exits 0 today; it must still exit 0 after your edits.
swiftc -parse Sources/BobMacCapture/*.swift Tests/BobMacCaptureTests/*.swift

# 2. Style gate. Exits 1 on a parse error and 0 with only style warnings, so treat any
#    `error:` line in your changed files as a hard failure and the pre-existing
#    Indentation/LineLength warnings as noise.
swift-format lint --recursive Package.swift Sources Tests

# 3. Cross-platform regression check. This change does not touch CaptureCore, so this
#    must stay green.
swift build && swift test
```

**What you cannot do locally:** type-check `BobMacCapture`, or run a single
`BobMacCaptureTests` case. Do not report the test suite as passing on that basis.

**The real gate** is `.github/workflows/ci.yml`, the `macOS 26 SwiftPM` job, which runs
`swift-format lint`, `build`, `test`, `bundle`, plist/signature verification, a launch
smoke test, and an install/reinstall check on every pull request. State plainly in your
handoff that the Swift compile and the `BobMacCaptureTests` suite were verified by CI,
not locally, and link the run.

**Manual confirmation (only if a macOS machine is available; otherwise say you skipped
it).** Build and install the app, open the capture panel, type a multi-line draft, then:
press Ctrl-A twice from mid-line and confirm the caret lands on the line start and then
on the previous line's start; hold Ctrl-A to the top and confirm it stops there; do the
mirror with Ctrl-E; and press Ctrl-A while the completion list is open and confirm the
list stays up and re-anchors. That last check also confirms the SwiftUI
`AttributedTextSelection` binding observed the programmatic `setSelectedRange` -- if
completion instead goes stale or targets the old caret, the binding did not sync and the
fix is to route the caret through the model's selection instead of the text view.

## 8. Out of scope

- No wrap-around at the document edges (decision 2).
- No smart-home ("first non-whitespace, then column zero") behavior; Ctrl-A goes to
  column zero in one step.
- No change to `Ctrl-Shift-A` / `Ctrl-Shift-E` selection extension, which stays
  AppKit's.
- No change to Ctrl-A/Ctrl-E inside the Add block ID and Name Pomodoro fields or the
  stash picker.
- No `hasMarkedText()` IME guard. No sibling handler has one, and adding a divergent
  policy here is not this change's job. If IME interaction turns out to be a problem,
  file it as a separate task bead.
- No fix for the Caps Lock interaction described in decision 5; it is pre-existing
  across every Ctrl binding in this router and should be fixed for all of them at once
  or not at all.

## 9. Done when

1. Ctrl-A and Ctrl-E move to the current physical line's beginning/end, and when already
   there step to the previous/next line, stopping at the first and last line.
2. The new router, resolver, and live-`NSTextView` tests from section 5 exist and pass
   in CI.
3. `swiftc -parse` over the macOS sources and tests exits 0, and
   `swift build && swift test` for `CaptureCore` stays green.
4. The macOS 26 CI job is green on the PR.
5. The README Keyboard table and editing prose describe the new behavior, including that
   it stops rather than wraps.
