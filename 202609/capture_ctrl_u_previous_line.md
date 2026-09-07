---
tier: tale
title: Extend Ctrl-U to delete the previous line when there is nothing to delete
goal:
  In the Bob Mac Capture editor, Ctrl-U keeps deleting from the caret to the beginning
  of the current physical line, and when the caret already sits at that line's start (so
  the ordinary deletion would remove nothing) it instead deletes the entire previous
  physical line including its terminator, stopping on the first line.
size: small
proposed_by: bbugyi200.athena.01e.f1
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-xy.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.1/README.md)
  - [bbugyi200.athena.sase-xy.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.2/README.md)
  - [bbugyi200.athena.sase-xy.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.3/README.md)
  - [bbugyi200.athena.sase-xy.4.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xy.4.1/README.md)
- **COMMITS:**
  - [4bb47fc](https://github.com/sase-org/sase/commit/4bb47fc987541c33386ee508024a956c44cbb79d)
    — feat(pager): resolve links against ordered workspace anchors
  - [f6501e3](https://github.com/sase-org/sase/commit/f6501e308fbf83d544501c724e201c54762361b6)
    — feat(pager): include :line suffixes in scanned file-path spans
  - [a0fcc5a](https://github.com/sase-org/sase/commit/a0fcc5ade1600a815f1f250dcc15d95e67060aaf)
    — feat(pager): thread link context through entry points
  - [5144564](https://github.com/sase-org/sase/commit/51445642c37303762ef7bb51be7c49c680c19ee4)
    — feat(pager): resolve dead ends in one background pass

# Plan: Delete the previous line with Ctrl-U when the current deletion would be empty

## 1. Where this work happens

All changes land in the **`bob-mac-capture`** linked repo. Open it first with your
`/sase_repo` skill and use only the path that command prints:

```bash
sase repo open bob-mac-capture -r "Implement Ctrl-U previous-line deletion in the capture editor"
```

No change is needed in `bob-cli` itself; this is purely an editor keymap change in the
macOS app. This is the direct sequel to commit `f2c1ed8` ("feat(capture): cycle
Ctrl-A/Ctrl-E across physical lines"), and it deliberately mirrors that change's
structure, naming, and test layout.

Files touched:

| File                                                  | Change                                                                                  |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` | Rename one enum case (no new key codes, no new routing logic)                           |
| `Sources/BobMacCapture/CapturePanelController.swift`  | New pure `previousLineDeletionRange` resolver plus one new branch in the Ctrl-U handler |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`   | New resolver tests and live-`NSTextView` tests; four renamed existing references        |
| `README.md`                                           | Reword the `Ctrl-U` keyboard-table row and the editing prose sentence                   |

## 2. Current behavior (verified, not assumed)

- `CaptureKeyCommandRouter` already routes Ctrl-U:
  `case KeyCode.u: return modifiers == .control ? .deleteToBeginningOfLine : nil`
  (`KeyCode.u = 32`). **No new key code or routing rule is needed** -- unlike the
  Ctrl-A/Ctrl-E work, which had to claim two previously unrouted keys.
- `CapturePanelController.deleteToBeginningOfLineInEditableTextView(firstResponder:model:)`
  is the whole handler today. It resolves the editable text view, calls
  `model.dismissCompletion()`, then hands the edit to AppKit:
  `textView.doCommand(by: Selector(("deleteToBeginningOfLine:")))`. It returns `false`
  only when there is no editable text view, and `true` in every other case -- including
  the case where AppKit deletes nothing.
- Consequently, with the caret already at column zero, Ctrl-U consumes the key and does
  nothing at all, no matter how many times it is pressed. Closing that gap is the entire
  point of this plan.
- The `.deleteToBeginningOfLine` command has exactly seven references across the repo (3
  in sources, 4 in tests); `deleteToBeginningOfLineInEditableTextView` has 6 (2 in
  sources, 4 in tests). Both counts were taken with `grep -rn deleteToBeginningOfLine`.
- Programmatic deletions in this file are applied with
  `textView.insertText("", replacementRange:)`, not by mutating `textView.string`.
  `deleteEmptyBulletRowInEditableTextView` does exactly that, and its tests
  (`testDeleteEmptyBulletRowRemovesFinalPlaceholderRowAndItsNewline` and siblings)
  assert that AppKit leaves the caret at `replacementRange.location` afterwards -- so
  the new branch does **not** need its own `setSelectedRange` call.
- Every line-aware helper here (`CaptureBulletNewlineEditResolver.resolve`,
  `bulletIndentationEdit`, `emptyBulletRowDeletionRange`, `lineEdgeCyclingLocation`)
  uses `NSString.getLineStart(_:end:contentsEnd:for:)`, i.e. **physical** lines, and the
  README already describes Ctrl-U in those terms.
- Caret and text changes already feed the model: `AutosizingCaptureEditor` calls
  `model.editorSelectionDidChange(cursorUTF8Offset:)` and the text-change hook on every
  edit, so nothing extra is needed to keep parse/preview in sync after the deletion.

## 3. Design decisions

These are decisions, not open questions. Each records the alternative so review can flip
one cheaply.

1. **The trigger is "the caret is at the physical line start", not "the current line is
   empty".** The request named the empty-line case; this is its strict superset, and the
   two coincide in the flow that motivates the feature (press Ctrl-U at the end of a
   line to clear it, press again to remove the line above). The superset is the right
   rule for three reasons: it is precisely the set where today's Ctrl-U silently does
   nothing, it is the exact analogue of the Ctrl-A trigger shipped in `f2c1ed8` ("the
   primitive already has nothing to do here"), and the narrow rule creates a dead end --
   after Ctrl-U on `one\ntw|o` the caret sits at column zero of the non-empty line `o`,
   where the narrow rule would refuse forever and break the "repeated presses walk up"
   promise. The cost is that Ctrl-U at column zero of a _non-empty_ line now deletes the
   line above rather than doing nothing; that is a deliberate, undoable trade.
   _Alternative:_ require `lineStart == contentsEnd` (a genuinely empty current line) by
   also fetching `contentsEnd` in the resolver and adding one guard. That is a two-line
   change plus adjusting the "caret at start of a non-empty line" cases in section 5.1
   from a range to `nil`.

2. **Delete the previous line _including_ its terminator:
   `[previousLineStart, lineStart)`.** The caret lands at `previousLineStart`, which is
   column zero of the current line in its new position, so a second press chains
   straight into the line above that. _Alternative:_ delete only the terminator, joining
   the two lines. That is what Backspace already does, so it would make Ctrl-U
   redundant.

3. **Stop at the first line; do not wrap around.** With the caret at offset 0 the
   resolver declines and Ctrl-U keeps consuming the key with no edit, exactly as today.
   This matches decision 2 of the Ctrl-A/Ctrl-E plan; wrap-around in this app is
   reserved for list selection, and silently deleting the _last_ line of a draft because
   the caret was at the top would be a destructive surprise.

4. **Keep AppKit's native `deleteToBeginningOfLine:` for every case the new branch does
   not claim.** The new resolver only ever returns a range for a collapsed caret at a
   physical line start with a line above it; everything else -- ordinary mid-line
   deletion, non-collapsed selections, the first line -- falls through to the existing
   `doCommand(by:)` call, so today's behavior is bit-for-bit preserved, including
   AppKit's own line-boundary handling, undo naming, and any kill-buffer interaction.
   This is a deliberate divergence from decision 3 of the Ctrl-A/Ctrl-E plan, which took
   over both branches: a caret _move_ is trivially reimplementable, whereas the ordinary
   Ctrl-U branch is a text mutation, and this file's stated rule (see the
   `insertNewline` and Ctrl-U doc comments) is that text mutation stays AppKit's. _Known
   limitation inherited from this choice:_ if the caret sits at the start of a
   soft-wrapped visual row that is not a physical line start, the resolver declines and
   AppKit's own line-based deletion may itself be a no-op, so Ctrl-U still appears inert
   there. That is exactly today's behavior, not a new defect, and the new Ctrl-A cannot
   put the caret in that state (it moves to _physical_ line starts). _Alternative:_ own
   both branches with a single resolver that returns `[lineStart, caret)` when
   non-empty. It makes the whole binding pure and testable and closes the wrapped-row
   dead end, at the cost of replacing a working native edit path.

5. **Collapsed carets only.** With a non-empty selection the resolver declines and the
   native path handles it as it does today. This composes correctly: the first press
   collapses/deletes via AppKit, a later press engages previous-line deletion.

6. **Keep `model.dismissCompletion()` unconditional.** Ctrl-U dismisses completion today
   on every invocation, and the new branch is a text mutation just like the old one, so
   it must dismiss too. Do not move the call inside either branch. (This is the opposite
   of the Ctrl-A/Ctrl-E rule, which deliberately preserved completion because a caret
   move is not an edit.)

7. **Rename the routed command to `.deleteToBeginningOfLineOrPreviousLine`.** The routed
   command vocabulary is the app's shared name for "what this key does", and the sibling
   commands added in `f2c1ed8` are `moveToBeginningOfLineOrPreviousLine` /
   `moveToEndOfLineOrNextLine`. The compiler finds every one of the seven references.
   Keep the handler named `deleteToBeginningOfLineInEditableTextView`: it is still "the
   Ctrl-U handler", and renaming it would churn four existing test call sites plus the
   two existing `testDeleteToBeginningOfLine*` test names for no clarity gain.
   _Alternative:_ skip the rename entirely if review prefers the smallest possible diff.

8. **No router change beyond the rename, and no new modifier surface.** Ctrl-U is
   already `modifiers == .control` only, and the prompts and stash picker already return
   early before the editor switch, so Ctrl-U cannot leak into them. Lock that in with
   regression assertions (section 5.3) rather than leaving it implicit.

## 4. Implementation

### 4.1 `CaptureKeyCommandRouter.swift`

Rename the enum case and its single use. Nothing else in this file changes:

```swift
case deleteToBeginningOfLineOrPreviousLine   // was: deleteToBeginningOfLine
```

```swift
case KeyCode.u:
    return modifiers == .control ? .deleteToBeginningOfLineOrPreviousLine : nil
```

### 4.2 `CapturePanelController.swift`

Add the pure resolver as a `nonisolated static` member, immediately after
`lineEdgeCyclingLocation` so the two line-walking helpers sit together. `nonisolated`
matches its siblings (the type is `@MainActor`, and these resolvers must stay callable
off the main actor; see commit `fc1c16b`).

```swift
/// Ctrl-U fallback: the range of the previous physical line, including its terminator,
/// for a collapsed caret that already sits at the start of its own physical line --
/// the state where deleting to the beginning of the line would remove nothing.
///
/// Returns `nil` whenever the ordinary deletion still has work to do or there is no
/// line above, so the caller falls through to AppKit's native
/// `deleteToBeginningOfLine:`: a non-collapsed selection, an out-of-bounds selection, a
/// caret past column zero, or a caret on the draft's first line.
///
/// "Line" means a physical line as `NSString.getLineStart(_:end:contentsEnd:for:)`
/// defines it, matching Ctrl-J, Ctrl-A/Ctrl-E, and Tab bullet indentation. Taking the
/// previous line as `[previousLineStart, lineStart)` rather than as raw offset
/// arithmetic keeps this correct for CRLF terminators, for blank lines, and for a draft
/// that ends in a newline.
nonisolated static func previousLineDeletionRange(
    in text: NSString,
    selectedRange: NSRange
) -> NSRange? {
    guard selectedRange.length == 0,
          selectedRange.location > 0,
          selectedRange.location <= text.length
    else {
        return nil
    }

    var lineStart = 0
    text.getLineStart(
        &lineStart,
        end: nil,
        contentsEnd: nil,
        for: NSRange(location: selectedRange.location, length: 0)
    )
    // Anything left before the caret on this line is the ordinary Ctrl-U deletion.
    guard selectedRange.location == lineStart else {
        return nil
    }

    // `location > 0` already guarantees `lineStart > 0`, so a previous line exists.
    var previousLineStart = 0
    text.getLineStart(
        &previousLineStart,
        end: nil,
        contentsEnd: nil,
        for: NSRange(location: lineStart - 1, length: 0)
    )
    return NSRange(location: previousLineStart, length: lineStart - previousLineStart)
}
```

Then extend the existing handler. Keep its name, its `model.dismissCompletion()` call,
and its native fallback; only the new `if let` is added, and the doc comment is
rewritten to describe both branches:

```swift
/// Ctrl-U: delete from the caret to the beginning of the current physical line. When
/// the caret is already at that line's start -- where the ordinary deletion would
/// remove nothing -- delete the whole previous physical line instead, so repeated
/// presses walk up the draft line by line and stop on the first line. The ordinary
/// branch stays AppKit's native deletion so line boundaries, undo, IME, and
/// accessibility remain owned by the text system.
static func deleteToBeginningOfLineInEditableTextView(
    firstResponder: NSResponder?,
    model: CapturePanelModel
) -> Bool {
    guard let textView = editableTextView(firstResponder) else {
        return false
    }

    model.dismissCompletion()

    if let deletionRange = previousLineDeletionRange(
        in: textView.string as NSString,
        selectedRange: textView.selectedRange()
    ) {
        textView.insertText("", replacementRange: deletionRange)
        textView.scrollRangeToVisible(NSRange(location: deletionRange.location, length: 0))
        return true
    }

    textView.doCommand(by: Selector(("deleteToBeginningOfLine:")))
    return true
}
```

`scrollRangeToVisible` mirrors `moveLineEdge`: the editor scrolls internally past six
visual lines (README "Keyboard" section), and this branch is the one that can move the
caret to a line that was previously above the visible region. It is a no-op when the
caret is already visible.

Finally, update the single `perform(_:)` case label to the renamed command
(`case .deleteToBeginningOfLineOrPreviousLine:`); the switch is exhaustive with no
`default`, so the compiler points at it.

## 5. Tests

All new tests go in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, which already
holds the router tests, the `NSTextView` helper tests, and the private
`keyEvent(keyCode:modifiers:characters:)` factory.

### 5.1 Resolver coverage

Add `testPreviousLineDeletionRangeTargetsTheLineAboveAColumnZeroCaret()` calling
`CapturePanelController.previousLineDeletionRange(in:selectedRange:)` directly, in the
style of the existing `bulletIndentationEdit` and `lineEdgeCyclingLocation` tests.

**Every expectation in this section was compiled and executed against
swift-corelibs-Foundation's `NSString` while this plan was written: all 20 cases below
produce exactly the stated results, and the chaining walk-through in section 5.2 was
executed too. Treat a disagreement as a transcription mistake in your port, not as a
wrong expectation.**

`"one\ntwo\nthree"` (length 13; line starts 0/4/8):

| Caret         | Expected                    | Resulting draft |
| ------------- | --------------------------- | --------------- |
| 6             | `nil` (ordinary deletion)   | --              |
| 4             | `NSRange(0, 4)`             | `"two\nthree"`  |
| 8             | `NSRange(4, 4)`             | `"one\nthree"`  |
| 0             | `nil` (first line, no wrap) | --              |
| 13            | `nil` (ordinary deletion)   | --              |
| `(4, len 3)`  | `nil` (non-collapsed)       | --              |
| `(0, len 5)`  | `nil` (non-collapsed)       | --              |
| `(99, len 0)` | `nil` (out of bounds)       | --              |

Add `testPreviousLineDeletionRangeHandlesEdgeCaseDrafts()` for the shapes that break
naive offset arithmetic:

- Empty draft `""`, caret `0`: `nil`.
- Trailing newline `"one\n"` (length 4), caret `4`: `NSRange(0, 4)`, leaving `""`. This
  is the literal "current line is empty" case from the feature request.
- Blank middle line `"a\n\nb"` (length 4): caret `2` (on the blank line) ->
  `NSRange(0, 2)`, leaving `"\nb"`; caret `3` (start of `b`) -> `NSRange(2, 1)`, which
  deletes the blank line itself and leaves `"a\nb"`.
- CRLF `"a\r\nb"` (length 4): caret `3` -> `NSRange(0, 3)`, leaving `"b"` (both
  terminator characters go); caret `1` -> `nil`.
- Indented bullet `"- a\n  - b"` (length 9): caret `4` -> `NSRange(0, 4)`, leaving
  `"  - b"`; caret `6` (after the two-space indent) -> `nil`, proving column zero rather
  than "first non-whitespace" -- the indent is the ordinary branch's business.
- Astral plane `"🧪\nb"` (length 4 in UTF-16): caret `3` -> `NSRange(0, 3)`, leaving
  `"b"`; caret `2` (between the surrogate pair's line and the newline) -> `nil`.
- Consecutive blank lines `"\n\n"` (length 2), caret `2` -> `NSRange(1, 1)`.
- Blank first line `"\nb"` (length 2), caret `1` -> `NSRange(0, 1)`, leaving `"b"`.

### 5.2 Live text-view coverage

Model these on the existing `testDeleteToBeginningOfLine*` tests: `NSTextView()` with
`isEditable = true`, a `CapturePanelModel` whose `completionResponse` is
`sampleCompletionResponse()`.

Add `@MainActor func testDeleteToBeginningOfLineRemovesPreviousLineFromColumnZero()`:

- `textView.string = "one\ntwo\n"`, caret at `8` (the empty final line).
- First call returns `true`; string is `"one\n"`, `selectedRange() == NSRange(4, 0)`,
  and `model.completionResponse` is `nil`.
- Second call returns `true`; string is `""`, caret `NSRange(0, 0)`.
- Third call returns `true` (Ctrl-U always consumes the key) and leaves the string `""`
  with the caret at `NSRange(0, 0)` -- the first-line stop.

That test is the regression test for the actual feature request: repeated presses walk
up line by line and then stop.

Add `@MainActor func testDeleteToBeginningOfLineKeepsNativeBehaviorForMidLineCarets()`:

- `textView.string = "one\ntwo"`, caret at `6` (between `w` and `o`): returns `true`,
  string becomes `"one\no"`, caret `NSRange(4, 0)`. This pins decision 4 -- the ordinary
  branch is untouched and still native.
- Re-running from that state (caret `4`, non-empty line `o`) returns `true` and leaves
  `"o"` with caret `NSRange(0, 0)`. This pins decision 1: the trigger is column zero,
  not an empty line.

Leave the existing `testDeleteToBeginningOfLineRemovesOnlyCurrentPhysicalLinePrefix` and
`testDeleteToBeginningOfLineDeclinesNoneditableUnrelatedAndMissingResponders` unchanged
except for the enum rename -- they are the proof that this change did not disturb the
native path or the responder guards.

### 5.3 Router coverage

The router's Ctrl-U behavior does not change, so this is rename plus regression only:

- Update the four existing `.deleteToBeginningOfLine` references (lines ~503, ~555,
  ~784, ~787) to `.deleteToBeginningOfLineOrPreviousLine`.
- In `testKeyRouterMatchesCaptureShortcuts`, add `nil` assertions for `keyCode: 32`
  unmodified, `modifiers: .command`, `[.control, .shift]`, `[.control, .option]`, and
  `[.control, .command]`, matching the shape already used for key codes 33 and 8.
- Add `nil` assertions for `keyCode: 32, modifiers: .control` under
  `CaptureKeyRoutingContext(taskIDPromptVisible: true)` and again with
  `pomodoroNamePromptVisible: true`, and under
  `CaptureKeyRoutingContext(stashPickerVisible: true, stashEntryCount: 2)`. For the
  stash-picker assertion pass `characters: "\u{15}"` (Ctrl-U's control character) so it
  actually exercises `printableModalCommand`'s `CharacterSet.controlCharacters` guard
  rather than passing only because the default `characters` is empty.

## 6. Documentation

In `README.md`, replace the `Ctrl-U` row of the "Keyboard" table (line ~240, the table
whose columns are `In the editor` / `While completion is visible` /
`While Add block ID is open` / `While Name Pomodoro is open`) with:

```
| Ctrl-U | Delete from the caret to the beginning of the current physical line; when the caret is already at that start, delete the previous line instead, stopping on the first line | Same deletion, closing completion | Native text-field behavior | Native text-field behavior |
```

In the editing prose paragraph (line ~371), replace the sentence

> Ctrl-U deletes from the caret to the beginning of the current physical line.

with

> Ctrl-U deletes from the caret to the beginning of the current physical line, and when
> the caret is already at that start -- where there is nothing left to delete -- it
> removes the previous line entirely, so repeated presses walk up the draft line by line
> and stop on the first line.

Leave the surrounding "All five shortcuts act on the native text view directly" sentence
alone: the count is unchanged (Ctrl-U was already one of the five), and the claim stays
true because both branches go through the native text view.

## 7. Validation

Run from the path `sase repo open bob-mac-capture` printed:

1. `swiftc -parse` over the macOS sources and tests, as the previous change did -- the
   `BobMacCapture` target itself cannot be type-checked or run on Linux (AppKit).
2. `swift-format lint` on the touched files; the repo has pre-existing Indentation and
   LineLength warnings, so the bar is "no new `error:` lines", not a clean run.
3. `swift build && swift test` for the `CaptureCore` target. Two failures in
   `BobProcessClientTests` (`testCancellationTerminatesProcess`,
   `testRunTerminatesAndThrowsTimedOutWhenProcessOutlivesTheTimeout`) are pre-existing
   process-kill flakes unrelated to this change; anything else failing is yours.
4. Optionally re-run the section 5.1 expectations as a standalone Linux Foundation
   script (`swift` on a scratch file importing only `Foundation`) -- the resolver has no
   AppKit dependency, so it is directly executable there and this is the cheapest way to
   catch a transcription slip before CI.
5. The real gate is the macOS 26 SwiftPM CI job, which is the only place
   `BobMacCaptureTests` actually runs. Say so plainly in the handoff rather than
   implying a local pass.
