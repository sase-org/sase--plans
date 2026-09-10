---
tier: tale
title:
  Move Ctrl-J bullet insertion to Ctrl-I and add Ctrl-J/Ctrl-K column-keeping caret
  movement
goal:
  In the Bob Mac Capture editor, Ctrl-I inserts the indentation-aware `- ` row that
  Ctrl-J used to insert, and Ctrl-J / Ctrl-K move the caret to the next / previous
  physical line, keeping the current column when the target line is long enough and
  clamping to that line's end when it is not.
size: medium
proposed_by: bbugyi200.athena.02t
create_time: 2026-09-09 20:00:37
status: wip
---

# Plan: Move Ctrl-J bullet insertion to Ctrl-I and add Ctrl-J/Ctrl-K column-keeping caret movement

## 1. Where this work happens

All changes land in the **`bob-mac-capture`** linked repo. Open it first with your
`/sase_repo` skill and use only the path that command prints:

```bash
sase repo open bob-mac-capture -r "Remap Ctrl-J bullet insertion to Ctrl-I and add Ctrl-J/Ctrl-K caret movement"
```

No change is needed in `bob-cli`. This is purely an editor keymap change in the macOS
app: `grep -rn 'Ctrl-J\|ctrl+j\|Control-J' --include=*.md --include=*.py` over `bob-cli`
returns nothing, and no `bob` subprocess or JSON contract is involved.

Files touched:

| File                                                  | Change                                                                                                                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` | Two new key codes, two new `CaptureKeyCommand` cases, `insertBulletNewline` moves from key code 38 to 34                                                      |
| `Sources/BobMacCapture/CapturePanelController.swift`  | New direction enum + result struct, new pure resolver, new apply helper with sticky-column state, two `perform(_:)` cases, Ctrl-J -> Ctrl-I doc-comment fixes |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`   | Updated Ctrl-I router tests, new Ctrl-J/Ctrl-K router tests, new resolver table tests, new live-`NSTextView` tests                                            |
| `README.md`                                           | Ctrl-J row becomes Ctrl-I, two new Keyboard rows, prose updates                                                                                               |

## 2. Current behavior (verified, not assumed)

Read at `871eaed` (`feat: restart Bob Mac Capture after a successful reinstall`).

- `CaptureKeyCommandRouter.KeyCode` declares `j: UInt16 = 38`, and the editor switch in
  `command(for:context:)` has
  `case KeyCode.j: return modifiers == .control ? .insertBulletNewline : nil`. That is
  the only routing of Ctrl-J today.
- `CaptureKeyCommand.insertBulletNewline` is dispatched by
  `CapturePanelController.perform(_:)` to the static
  `insertBulletNewlineInEditableTextView(firstResponder:model:)`, which delegates the
  edit to the pure `CaptureBulletNewlineEditResolver.resolve(in:selectedRange:)`. **None
  of that changes.** Only the key code that reaches it changes. The command case keeps
  its behavior-based name.
- Key codes are Apple virtual key codes: `i = 34`, `j = 38`, `k = 40`. The file already
  encodes `a = 0`, `s = 1`, `c = 8`, `e = 14`, `u = 32`, `leftBracket = 33`, `p = 35`,
  `n = 45`, all of which match the same table, so these three are consistent with it.
- Ctrl-I is **not** routed anywhere today. The app's global hotkey is
  Control-Shift-Command-I (`HotKeyManager.swift`), which is a different modifier set and
  a Carbon global hotkey, not an editor key event; the `modifiers == .control` equality
  test in the router already excludes it. No `NSMenuItem` in `AppDelegate.swift` uses a
  Control key equivalent.
- Ctrl-K is **not** routed today, so it currently falls through the local key monitor to
  AppKit's standard key bindings, where `^k` is `deleteToEndOfParagraph:`. Today Ctrl-K
  in the capture editor **deletes the rest of the line.** See decision 5; this is the
  one behavior this change removes.
- Ctrl-J falling through would reach AppKit with `event.characters == "\n"`. It never
  does today because the router claims it.
- The editor is a SwiftUI `TextEditor` (`AutosizingCaptureEditor` in
  `CapturePanelView.swift`) bound to `model.attributedDraft` / `model.editorSelection`,
  but every caret-aware keymap reaches the backing `NSTextView` through
  `CapturePanelController.editableTextView(_:)` on the panel's first responder and sets
  the caret with `textView.setSelectedRange(_:)`. `moveLineEdge` (Ctrl-A / Ctrl-E) is
  the closest sibling; follow it.
- Caret movement already feeds the model: `AutosizingCaptureEditor` calls
  `model.editorSelectionDidChange(cursorUTF8Offset:)` on every selection change. Nothing
  extra is needed to keep parse / preview / completion in sync after a caret move.
- Every line-aware helper here (`CaptureBulletNewlineEditResolver.resolve`,
  `bulletIndentationEdit`, `emptyBulletRowDeletionRange`, `lineEdgeCyclingLocation`,
  `previousLineDeletionRange`) uses `NSString.getLineStart(_:end:contentsEnd:for:)`,
  i.e. **physical** lines, and the README describes them that way.
- `perform(_:)`'s switch is exhaustive with no `default:`, so adding a
  `CaptureKeyCommand` case makes the compiler point at every site that must be updated.

## 3. Design decisions

These are decisions, not open questions. Each records the alternative so review can flip
one cheaply.

1. **Physical lines, not visual (soft-wrapped) lines.** "Next line" means the next
   paragraph as `NSString.getLineStart` defines it, and "column" means the UTF-16 offset
   from that paragraph's start. This matches Ctrl-A/Ctrl-E, Ctrl-U, Ctrl-I, and Tab
   indentation, and keeps the resolver a pure function with no layout-manager
   dependency. _Consequence to accept knowingly:_ the editor soft-wraps (it grows
   through six visual lines, then scrolls), so on a wrapped paragraph one Ctrl-J jumps
   the whole paragraph rather than one visual row. _Alternative:_ delegate to
   `textView.doCommand(by: #selector(NSResponder.moveDown(_:)))` and let AppKit's own
   sticky x-position do the work, which is visual-line aware and about five lines of
   code. Rejected for consistency with every sibling binding, because it is untestable
   as a pure function, and because the sticky x-position is AppKit-internal state that
   SwiftUI's `AttributedTextSelection` round-trip may reset between presses. If review
   prefers visual lines, section 4.2 collapses to two `doCommand(by:)` calls and
   sections 5.2/5.3 are deleted.

2. **A sticky goal column, invalidated by caret identity.** Consecutive Ctrl-J/Ctrl-K
   presses carry the column from the first press, so descending through a short line and
   continuing onto a long one restores the original column — vim's `curswant`, which is
   what "maintaining the current column position if possible" means for a `j`/`k`
   binding. The controller stores `(column, caret)` and honors the carried column only
   when the text view's current selection still equals the exact `caret` it last set.
   Any edit, click, arrow key, Ctrl-A/Ctrl-E, or completion acceptance moves the
   selection and invalidates the pairing for free. _Alternative:_ hook every other input
   path to clear the goal explicitly; rejected because typing and mouse clicks never
   pass through `perform(_:)`, so that approach would be incomplete by construction.

3. **Stop at the document edges; do not wrap around.** Ctrl-K on the first line and
   Ctrl-J on the last line leave the caret where it is. This matches decision 2 of the
   Ctrl-A/Ctrl-E plan: wrap-around in this app is reserved for list selection
   (completion rows, stash rows), where it is cheap and reversible.

4. **Consume Ctrl-J and Ctrl-K unconditionally once the draft's text view holds focus —
   including when the move declines.** This is the one place this plan deliberately
   diverges from `moveLineEdge`, which returns `false` at the document edges so AppKit's
   harmless native paragraph movement runs. Here, returning `false` for Ctrl-K would
   hand the event to AppKit's `deleteToEndOfParagraph:` and **silently delete the rest
   of the line at exactly the moment the user hit the top of the draft.** Ctrl-J is
   consumed the same way for symmetry. `moveVertically` still returns `false` when the
   first responder is not the editable text view (focus on a button, a prompt field, the
   panel itself), which is safe because there is no draft text there to destroy.

5. **Accept that Ctrl-K stops deleting to end of paragraph.** That AppKit default is
   what Ctrl-K does in the capture editor today, and this change removes it. The user
   asked for `k` = up, which is incompatible with keeping it. Ctrl-U (delete to
   beginning of line) is unaffected, and the Emacs kill/yank pair was never wired up in
   this app anyway. Note it in the README so the loss is discoverable. _Alternative:_
   pick a different key for "up"; rejected, the request named Ctrl-K explicitly.

6. **Exactly `.control`, no other modifier**, for all three bindings. Use
   `modifiers == .control`, matching Ctrl-I's predecessor and Ctrl-A/Ctrl-E/Ctrl-U/
   Ctrl-S/Ctrl-C. In particular Control-Shift-Command-I (key code 34 with three
   modifiers) must stay unrouted so the global hotkey is never shadowed, and
   Ctrl-Shift-J/K must stay AppKit's. (Consequence, inherited from every sibling and
   intentionally not fixed here: with Caps Lock engaged `modifiers` is
   `[.control, .capsLock]` and none of these bindings fire.)

7. **Do not dismiss completion.** Ctrl-I keeps the `model.dismissCompletion()` call it
   inherits from Ctrl-J because it mutates text. Ctrl-J/Ctrl-K must not call it: they
   only move the caret, and `editorSelectionDidChange` re-anchors the open completion
   list at the new caret, exactly as Ctrl-A/Ctrl-E do today. Deliberate consequence:
   with the completion list open, Ctrl-J moves the _caret_, not the completion
   selection; Ctrl-N/Ctrl-P and the arrow keys remain the completion-list bindings.

8. **A non-collapsed selection collapses toward the direction of travel and then
   moves.** Ctrl-J anchors at `NSMaxRange(selectedRange)`, Ctrl-K at
   `selectedRange.location`. This diverges from `lineEdgeCyclingLocation`, which
   declines on a non-collapsed selection and lets AppKit collapse it — an option
   decision 4 has taken away, since falling through is exactly what must not happen. The
   chosen behavior is what every editor does and leaves no dead keystroke.

9. **Snap a clamped target back to a composed-character boundary.**
   `targetStart + column` can land inside a surrogate pair or a combining sequence (an
   emoji in the draft is enough). Snap with
   `NSString.rangeOfComposedCharacterSequence(at:)` so the resolver is deterministic
   rather than relying on `NSTextView` to fix it up.

10. **Prompts and the stash picker are untouched.** `command(for:context:)` returns
    early into `taskIDPromptCommand` / `pomodoroNamePromptCommand` /
    `stashPickerCommand` before reaching the editor switch, so none of the three
    bindings can leak into those modes; Ctrl-I/J/K keep whatever native behavior they
    have there, exactly as Ctrl-J does today. Lock this in with regression assertions
    (section 5.1) rather than leaving it implicit. In particular, Ctrl-J/Ctrl-K are
    deliberately **not** added to the stash picker as row navigation; Ctrl-N/Ctrl-P and
    the arrows already own that, and this request is about the editor caret.

## 4. Implementation

### 4.1 `CaptureKeyCommandRouter.swift`

Add two cases to `CaptureKeyCommand`, next to the other caret-movement commands (after
`moveToEndOfLineOrNextLine`):

```swift
case moveToNextLineKeepingColumn
case moveToPreviousLineKeepingColumn
```

Add two entries to the private `KeyCode` enum, keeping it in its existing loose
alphabetical-by-letter grouping — `i` before `j`, and `k` after `j`:

```swift
static let i: UInt16 = 34
static let k: UInt16 = 40
```

`static let j: UInt16 = 38` stays; only what it routes to changes.

In the editor switch inside `command(for:context:)`, replace the single existing
`case KeyCode.j:` with three cases, keeping the Ctrl family together:

```swift
case KeyCode.i:
    return modifiers == .control ? .insertBulletNewline : nil
case KeyCode.j:
    return modifiers == .control ? .moveToNextLineKeepingColumn : nil
case KeyCode.k:
    return modifiers == .control ? .moveToPreviousLineKeepingColumn : nil
```

Make no change to `isolatedPromptCommand`, `stashPickerCommand`, or
`printableModalCommand` (decision 10).

### 4.2 `CapturePanelController.swift`

Add the direction enum and result struct near the top of the file, beside
`CaptureLineEdge`:

```swift
/// Which adjacent physical line Ctrl-J / Ctrl-K targets.
enum CaptureVerticalDirection {
    case next
    case previous
}

/// Where a Ctrl-J / Ctrl-K move lands, plus the goal column to carry into the next
/// consecutive vertical move so a short line in between does not lose the column.
struct CaptureVerticalMove: Equatable {
    var location: Int
    var goalColumn: Int
}
```

Add the pure resolver as a `nonisolated static` member of `CapturePanelController`,
alongside `lineEdgeCyclingLocation`. Mark it `nonisolated` for the same reason its
siblings are (commit `fc1c16b`): the type is `@MainActor` and pure resolvers must stay
callable off the main actor.

```swift
/// Ctrl-J / Ctrl-K target. Returns the new collapsed caret location and the goal
/// column to carry forward, or `nil` when there is no adjacent physical line -- the
/// last line for `.next`, the first line for `.previous` -- or when the selection is
/// out of bounds. Unlike `lineEdgeCyclingLocation`, `nil` does **not** mean "fall
/// through to AppKit": see `moveVertically(_:firstResponder:)`.
///
/// A non-collapsed selection collapses toward the direction of travel first, so
/// `.next` measures its column from the selection's end and `.previous` from its
/// start.
///
/// The column is the UTF-16 offset from the physical line's start, and it is clamped
/// to the target line's `contentsEnd` so the caret never lands on or past a line
/// terminator. A clamped target that would split a surrogate pair or a combining
/// sequence snaps back to that sequence's start.
///
/// "Line" means a physical line as `NSString.getLineStart(_:end:contentsEnd:for:)`
/// defines it, matching Ctrl-I, Ctrl-U, and Ctrl-A/Ctrl-E. Working from `lineStart` /
/// `contentsEnd` / `lineEnd` rather than from raw offsets keeps this correct for CRLF
/// terminators and for a draft that ends in a newline.
nonisolated static func verticalMovementTarget(
    _ direction: CaptureVerticalDirection,
    in text: NSString,
    selectedRange: NSRange,
    goalColumn: Int?
) -> CaptureVerticalMove? {
    guard selectedRange.location >= 0,
          selectedRange.length >= 0,
          selectedRange.location + selectedRange.length <= text.length
    else {
        return nil
    }

    let anchor = direction == .next ? NSMaxRange(selectedRange) : selectedRange.location

    var lineStart = 0
    var lineEnd = 0
    var contentsEnd = 0
    text.getLineStart(
        &lineStart,
        end: &lineEnd,
        contentsEnd: &contentsEnd,
        for: NSRange(location: anchor, length: 0)
    )

    let column = goalColumn ?? (anchor - lineStart)

    var targetStart = 0
    var targetContentsEnd = 0
    switch direction {
    case .previous:
        // A previous line exists exactly when this one does not start the draft.
        guard lineStart > 0 else {
            return nil
        }
        text.getLineStart(
            &targetStart,
            end: nil,
            contentsEnd: &targetContentsEnd,
            for: NSRange(location: lineStart - 1, length: 0)
        )
    case .next:
        // A next line exists exactly when this one carries a terminator; the draft's
        // final line has `contentsEnd == lineEnd`.
        guard contentsEnd < lineEnd else {
            return nil
        }
        targetStart = lineEnd
        if targetStart >= text.length {
            // A draft ending in a newline has an empty final line whose start,
            // contents end, and end all equal the length.
            targetContentsEnd = targetStart
        } else {
            text.getLineStart(
                nil,
                end: nil,
                contentsEnd: &targetContentsEnd,
                for: NSRange(location: targetStart, length: 0)
            )
        }
    }

    var location = min(targetStart + column, targetContentsEnd)
    if location > targetStart, location < targetContentsEnd {
        let composed = text.rangeOfComposedCharacterSequence(at: location)
        if composed.location < location {
            location = composed.location
        }
    }
    return CaptureVerticalMove(location: location, goalColumn: column)
}
```

Add the sticky-column state as a stored property next to the controller's other private
state (`localMonitor` and friends):

```swift
/// Goal column for consecutive Ctrl-J/Ctrl-K moves, paired with the exact collapsed
/// caret this controller last left behind. Any other edit, click, or caret move
/// changes the text view's selection away from `caret`, so the pairing invalidates
/// itself without this controller having to observe every other input path.
private var verticalMovementGoal: (column: Int, caret: NSRange)?
```

Add the apply helper next to `moveLineEdge`. Unlike its siblings this is an **instance**
method, because the goal column is per-controller state; it still takes `firstResponder`
so tests can drive it with a bare `NSTextView`. It deliberately takes no
`CapturePanelModel` (decision 7).

```swift
/// Ctrl-J / Ctrl-K: move the caret to the next / previous physical line, keeping the
/// column across consecutive presses. Returns `true` whenever the draft's text view
/// holds focus, **including** when the move declines at the first or last line:
/// letting Ctrl-K fall through to AppKit would run `deleteToEndOfParagraph:` and
/// delete the rest of the line. Returns `false` only when there is no editable text
/// view to move in, where falling through is harmless. Like Ctrl-A/Ctrl-E this never
/// dismisses completion: a caret move is not an edit, and `editorSelectionDidChange`
/// re-anchors the completion list at the new caret on its own.
func moveVertically(
    _ direction: CaptureVerticalDirection,
    firstResponder: NSResponder?
) -> Bool {
    guard let textView = Self.editableTextView(firstResponder) else {
        verticalMovementGoal = nil
        return false
    }

    let selection = textView.selectedRange()
    let carriedColumn = verticalMovementGoal.flatMap {
        $0.caret == selection ? $0.column : nil
    }

    guard let move = Self.verticalMovementTarget(
        direction,
        in: textView.string as NSString,
        selectedRange: selection,
        goalColumn: carriedColumn
    ) else {
        // The caret did not move, so any carried goal stays valid and a press back the
        // other way still restores the column.
        return true
    }

    let target = NSRange(location: move.location, length: 0)
    textView.setSelectedRange(target)
    textView.scrollRangeToVisible(target)
    verticalMovementGoal = (column: move.goalColumn, caret: target)
    return true
}
```

`scrollRangeToVisible` matters: the editor scrolls internally past six visual lines, and
a bare `setSelectedRange` does not bring the caret back into view.

Add the two cases to `perform(_:)`, next to the existing `moveToBeginningOfLine...` /
`moveToEndOfLine...` cases. Note there is no `Self.` prefix — `perform(_:)` and
`moveVertically` are both instance members:

```swift
case .moveToNextLineKeepingColumn:
    return moveVertically(.next, firstResponder: panel?.firstResponder)
case .moveToPreviousLineKeepingColumn:
    return moveVertically(.previous, firstResponder: panel?.firstResponder)
```

### 4.3 Rename Ctrl-J in doc comments

`insertBulletNewline` is now reached by Ctrl-I, so update every comment that names the
key. These are the exact sites at `871eaed`; re-grep with
`grep -rn 'Ctrl-J' Sources Tests` and confirm zero hits when done:

| File                                                 | Line | Current text to fix                                              |
| ---------------------------------------------------- | ---- | ---------------------------------------------------------------- |
| `Sources/BobMacCapture/CapturePanelController.swift` | 365  | `/// Ctrl-J: resolve a deterministic native text edit...`        |
| `Sources/BobMacCapture/CapturePanelController.swift` | 555  | `...matching Ctrl-U, Ctrl-J, and Tab bullet indentation.`        |
| `Sources/BobMacCapture/CapturePanelController.swift` | 633  | `...matching Ctrl-J, Ctrl-A/Ctrl-E, and Tab bullet indentation.` |
| `Sources/BobMacCapture/CapturePanelController.swift` | 951  | `// Ctrl-J, Ctrl-U, the placeholder-row Backspace, and Tab...`   |
| `Sources/BobMacCapture/CapturePanelController.swift` | 963  | `...the empty-bullet placeholder `- ` that Ctrl-J inserts.`      |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`  | 897  | `// exactly where the caret sits right after Ctrl-J inserted...` |

While editing line 951, extend that comment's list to mention Ctrl-J/Ctrl-K only if you
keep it accurate — those two move the caret rather than editing, so the simplest correct
edit is to swap `Ctrl-J` for `Ctrl-I` and leave the list otherwise alone.

## 5. Tests

All new tests go in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, which already
holds the router and `NSTextView` helper tests and has the private
`keyEvent(keyCode:modifiers:characters:)` factory.

### 5.1 Router coverage

**Update the existing Ctrl-J assertions first.** These four sites currently assert key
code 38 maps to `.insertBulletNewline` and will otherwise fail:

- `testKeyRouterMatchesCaptureShortcuts()` lines 502, 559 and the `[.control, .shift]`
  nil assertion at line 560 — change key code `38` to `34`.
- `testKeyRouterMatchesControlJAsBulletNewlineOnlyWithControlModifier()` (line 755) —
  rename to `testKeyRouterMatchesControlIAsBulletNewlineOnlyWithControlModifier()` and
  change every `38` to `34`. Add one assertion that key code `34` with
  `[.control, .shift, .command]` is `nil`, so the Control-Shift-Command-I global hotkey
  can never be shadowed (decision 6). Add one assertion that key code `38` with
  `.control` is **no longer** `.insertBulletNewline`.

Add `testKeyRouterMapsControlVerticalCaretMovement()`:

- `keyCode: 38, modifiers: .control` -> `.moveToNextLineKeepingColumn`
- `keyCode: 40, modifiers: .control` -> `.moveToPreviousLineKeepingColumn`
- The same two with `completionVisible: true` -> the same commands (decision 7).
- `nil` for each of: `keyCode: 38` and `keyCode: 40` unmodified; each with `.shift`,
  `.command`, `.option`, `[.control, .shift]`, `[.control, .option]`, and
  `[.control, .command]`.
- With `CaptureKeyRoutingContext(taskIDPromptVisible: true)` and again with
  `pomodoroNamePromptVisible: true`: `nil` for key codes 34, 38, and 40 with `.control`
  (decision 10).
- With `CaptureKeyRoutingContext(stashPickerVisible: true, stashEntryCount: 2)`: `nil`
  for the same three. Pass realistic control characters — `characters: "\u{09}"` for
  Ctrl-I, `"\u{0A}"` for Ctrl-J, `"\u{0B}"` for Ctrl-K — so the assertion actually
  exercises `printableModalCommand`'s `CharacterSet.controlCharacters` guard rather than
  passing only because the default `characters` is empty. Ctrl-I is the one that matters
  here: its character is a literal Tab.

### 5.2 Resolver coverage

Add `testVerticalMovementTargetKeepsColumnAcrossPhysicalLines()` calling
`CapturePanelController.verticalMovementTarget(_:in:selectedRange:goalColumn:)`
directly, in the style of the existing `lineEdgeCyclingLocation` tests.

Every expectation below was produced by compiling and running the exact resolver body
from section 4.2 against **both** swift-corelibs-Foundation (Swift 6.3.3, Linux) and
**Apple's Foundation** (Apple Swift 6.3.2, macOS 26.5, arm64) while this plan was
written. Both platforms produced identical results for all 30 cases. Treat a
disagreement as a transcription mistake in your port, not as a wrong expectation.

`"alpha\nhi\nbravo"` (length 14; line starts 0/6/9, contents ends 5/8/14, line ends
6/9/14) — this is the ragged draft that proves the goal column:

| Selection | Direction   | `goalColumn` | Expected             |
| --------- | ----------- | ------------ | -------------------- |
| `(4, 0)`  | `.next`     | `nil`        | `(loc: 8, goal: 4)`  |
| `(8, 0)`  | `.next`     | `4`          | `(loc: 13, goal: 4)` |
| `(13, 0)` | `.next`     | `4`          | `nil`                |
| `(13, 0)` | `.previous` | `4`          | `(loc: 8, goal: 4)`  |
| `(8, 0)`  | `.previous` | `4`          | `(loc: 4, goal: 4)`  |
| `(4, 0)`  | `.previous` | `4`          | `nil`                |
| `(8, 0)`  | `.next`     | `nil`        | `(loc: 11, goal: 2)` |

Rows 1-2 are the feature: column 4 of `alpha` clamps to the end of the two-character
`hi`, then re-expands to column 4 of `bravo`. Rows 4-5 are the mirror. Row 7 is the
control: **without** a carried goal, the same press from the same caret lands at column
2, which is what makes the sticky column observable.

`"one\ntwo\nthree"` (length 13; line starts 0/4/8, contents ends 3/7/13):

| Selection | Direction   | `goalColumn` | Expected             |
| --------- | ----------- | ------------ | -------------------- |
| `(1, 0)`  | `.next`     | `nil`        | `(loc: 5, goal: 1)`  |
| `(9, 0)`  | `.previous` | `nil`        | `(loc: 5, goal: 1)`  |
| `(4, 3)`  | `.next`     | `nil`        | `(loc: 11, goal: 3)` |
| `(4, 3)`  | `.previous` | `nil`        | `(loc: 0, goal: 0)`  |
| `(99, 0)` | `.next`     | `nil`        | `nil`                |
| `(99, 0)` | `.previous` | `nil`        | `nil`                |

Rows 3-4 are decision 8: the selection `"two"` collapses to its end for `.next`
(column 3) and to its start for `.previous` (column 0).

Add `testVerticalMovementTargetHandlesEdgeCaseDrafts()` for the shapes that break naive
offset arithmetic:

| Draft              | Selection | Direction   | `goalColumn` | Expected             |
| ------------------ | --------- | ----------- | ------------ | -------------------- |
| `""`               | `(0, 0)`  | `.next`     | `nil`        | `nil`                |
| `""`               | `(0, 0)`  | `.previous` | `nil`        | `nil`                |
| `"a\n"`            | `(1, 0)`  | `.next`     | `nil`        | `(loc: 2, goal: 1)`  |
| `"a\n"`            | `(2, 0)`  | `.next`     | `1`          | `nil`                |
| `"a\n"`            | `(2, 0)`  | `.previous` | `1`          | `(loc: 1, goal: 1)`  |
| `"a\n\nb"`         | `(1, 0)`  | `.next`     | `nil`        | `(loc: 2, goal: 1)`  |
| `"a\n\nb"`         | `(2, 0)`  | `.next`     | `1`          | `(loc: 4, goal: 1)`  |
| `"a\n\nb"`         | `(4, 0)`  | `.previous` | `1`          | `(loc: 2, goal: 1)`  |
| `"a\r\nb"`         | `(1, 0)`  | `.next`     | `nil`        | `(loc: 4, goal: 1)`  |
| `"a\r\nb"`         | `(4, 0)`  | `.previous` | `1`          | `(loc: 1, goal: 1)`  |
| `"ab\n\u{1F600}x"` | `(1, 0)`  | `.next`     | `nil`        | `(loc: 3, goal: 1)`  |
| `"ab\n\u{1F600}x"` | `(2, 0)`  | `.next`     | `nil`        | `(loc: 5, goal: 2)`  |
| `"- alpha\n  - b"` | `(6, 0)`  | `.next`     | `nil`        | `(loc: 13, goal: 6)` |
| `"- alpha\n  - b"` | `(13, 0)` | `.previous` | `nil`        | `(loc: 5, goal: 5)`  |

Notes on the non-obvious rows:

- `"a\n"` rows 3-5: the trailing newline creates an empty final line at offset 2, which
  `.next` can reach and `.previous` can leave.
- `"a\n\nb"` rows 6-8: a blank middle line clamps column 1 to offset 2 and the carried
  goal restores it on the line below.
- `"a\r\nb"` (length 4): CRLF is one terminator, so `.next` from offset 1 steps over
  both characters to offset 4, and `.previous` from 4 comes back to 1. Raw
  `location + 1` arithmetic would land inside the terminator.
- `"ab\n\u{1F600}x"` (length 6): the emoji is two UTF-16 units at offsets 3-4. Column 1
  would clamp to offset 4, the middle of the surrogate pair; decision 9 snaps it back
  to 3. Column 2 lands cleanly on 5.
- `"- alpha\n  - b"` rows 13-14 prove there is no smart-home behavior: the column is a
  raw offset from the line start, so the two-space indent shifts what column 6 means.

### 5.3 Live text-view coverage

Add `@MainActor func testMoveVerticallyKeepsColumnInEditableTextView()`, modeled on the
existing `testMoveLineEdgeCyclesCaretInEditableTextView()`. `moveVertically` is an
instance method, so build a controller the way the other `@MainActor` controller tests
do
(`let model = CapturePanelModel(); let controller = CapturePanelController(model: model)`).
`NSTextView(frame: .zero)` is editable by default, which `editableTextView(_:)`
requires.

- `textView.string = "alpha\nhi\nbravo"`, caret at `4`.
  - `controller.moveVertically(.next, firstResponder: textView)` returns `true`,
    `selectedRange() == NSRange(8, 0)`.
  - Again: returns `true`, `NSRange(13, 0)`. **This is the regression test for the whole
    feature** — without the sticky column it would be `NSRange(11, 0)`.
  - Again: returns `true` (decision 4 — consumed, not `false`) and the caret stays at
    `NSRange(13, 0)`.
  - `.previous` twice: `NSRange(8, 0)` then `NSRange(4, 0)`, both `true`.
  - `.previous` once more: `true`, caret unchanged at `NSRange(4, 0)`.
- Goal invalidation, same view and controller: from caret `4`, one `.next` lands at `8`;
  then `textView.setSelectedRange(NSRange(location: 7, length: 0))` to simulate the user
  clicking or typing elsewhere; then `.next` must land at `NSRange(10, 0)` (column 1 of
  `bravo`), **not** `NSRange(13, 0)`. This is the test for decision 2.
- Two independent controllers must not share a goal: assert a fresh
  `CapturePanelController` moving in the same text view from caret `8` lands at
  `NSRange(11, 0)`.
- `controller.moveVertically(.next, firstResponder: nil)` returns `false`.
- A non-editable text view is refused: build
  `let noneditable = NSTextView(); noneditable.isEditable = false` (the existing tests
  at line 1501 already use this shape) and assert `false`.

## 6. Documentation

In `README.md`:

1. In the "Keyboard" table (columns `In the editor` / `While completion is visible` /
   `While Add block ID is open` / `While Name Pomodoro is open`), change the existing
   `Ctrl-J` row's key cell to `Ctrl-I`. Leave all four of its description cells exactly
   as they are — the behavior did not change.

2. Add two rows immediately after the `Ctrl-E` row, so the caret-movement bindings stay
   together:

   ```
   | Ctrl-J | Move the caret to the next physical line, keeping the current column when that line is long enough and clamping to its end when it is not; stops on the last line | Same move, leaving completion open and re-anchored at the new caret | Native text-field behavior | Native text-field behavior |
   | Ctrl-K | Move the caret to the previous physical line, keeping the current column when that line is long enough and clamping to its end when it is not; stops on the first line | Same move, leaving completion open and re-anchored at the new caret | Native text-field behavior | Native text-field behavior |
   ```

3. In the editing prose paragraph that begins "Ctrl-J starts the next canonical `- `
   row...", replace all four occurrences of `Ctrl-J` with `Ctrl-I`. That paragraph later
   says "All five shortcuts act on the native text view directly" — the set it counts is
   Ctrl-I (renamed), Tab, Shift-Tab, Ctrl-U, and Backspace, so the count stays five.
   Re-read the sentence after editing and confirm it still reads correctly rather than
   adjusting the number reflexively.

4. Append a paragraph after the existing Ctrl-A/Ctrl-E paragraph, in the same voice:

   > Ctrl-J and Ctrl-K move the caret to the next and previous physical line, keeping
   > the column it started from: passing through a shorter line clamps the caret to that
   > line's end, and the next press in the same direction restores the original column
   > on the first line long enough to hold it. Any other keystroke, click, or edit
   > resets that remembered column. They stop at the last and first line rather than
   > wrapping around, and like Ctrl-A and Ctrl-E they only move the caret, so they never
   > touch the draft's text, undo history, or an in-flight IME composition. Because
   > Ctrl-K now belongs to the editor, it no longer falls through to AppKit's
   > delete-to-end-of-paragraph binding; Ctrl-U remains the line-deleting shortcut.

Do not add rows to the separate "Key while stash is open" table (decision 10).

## 7. Validation

Be honest about what can and cannot run. This package's app target is macOS-only
(`Package.swift` compiles `BobMacCapture` and `BobMacCaptureTests` only under
`#if os(macOS)`), so on the Linux host **`swift build` compiles `CaptureCore` and
nothing this plan changes.** Verified while writing this plan: a clean `swift build` on
athena reports `Build complete!` after compiling only the eight `CaptureCore` files.

**On the Linux host, from the opened `bob-mac-capture` checkout:**

```bash
export PATH="$HOME/.local/share/swiftly/bin:$PATH"

# 1. Syntax gate. Parses the macOS-only sources and tests without resolving AppKit or
#    SwiftUI. Exits 0 today; it must still exit 0 after your edits.
swiftc -parse Sources/BobMacCapture/*.swift Tests/BobMacCaptureTests/*.swift

# 2. Cross-platform regression check. This change does not touch CaptureCore, so this
#    must stay green.
swift build && swift test
```

Do **not** treat `swift-format lint` on the Linux host as a gate. Verified while writing
this plan: the Linux swift-format 6.3.3 emits thousands of Indentation/LineLength
warnings on unmodified files (2733 on `BobMacCaptureTests.swift` alone) and the macOS
toolchain emits the same class of warnings, and in both cases the command still **exits
0**. CI's lint step has no `--strict`, so warnings never fail it. Only an `error:` line
in a file you changed is a real failure.

**Type-check the app target on the tailnet Mac.** This is the highest-value check
available before CI, and it was confirmed working while this plan was written:

```bash
# `mac` is Kelly's MacBook Pro, macOS 26.5, Command Line Tools only (no full Xcode).
# It is offline unless powered on with the lid open, so treat this as best-effort.
ssh mac 'cd ~/projects/github/bbugyi200/bob-mac-capture && git fetch && git checkout <your-branch> && ./Scripts/xcode-swift.sh build'
```

`Scripts/xcode-swift.sh` explicitly accepts Command Line Tools for Xcode 26+, and that
host resolves macOS SDK 26.5 with Apple Swift 6.3.2. An incremental
`./Scripts/xcode-swift.sh build` there completes in about 30 seconds and **does**
compile `Sources/BobMacCapture`, so it catches every compile error in section 4.

**What that Mac cannot do:** `./Scripts/xcode-swift.sh test` fails there with
`no such module 'XCTest'`, because Command Line Tools do not ship XCTest. Your section 5
tests cannot be compiled or run on it. Do not report the suite as passing on that basis.

**The real gate** is `.github/workflows/ci.yml`, the `macOS 26 SwiftPM` job (full
Xcode), which runs `swift-format lint`, `build`, `test`, `bundle`, plist/signature
verification, a launch smoke test, and an install/reinstall check on every pull request.
State plainly in your handoff which checks ran where, and link the CI run.

**Manual confirmation (only if a Mac with the app installed is available; otherwise say
you skipped it).** Open the capture panel and type the three-line draft `alpha` / `hi` /
`bravo`. Put the caret after `alph` and press Ctrl-J twice: it must land after `brav`,
not after `br`. Press Ctrl-K twice to get back. Hold Ctrl-J past the last line and
confirm nothing is deleted and the caret stops. Press Ctrl-I mid-draft and confirm it
inserts the `- ` row Ctrl-J used to. Press Ctrl-J with the completion list open and
confirm the list stays up and re-anchors at the new caret rather than changing its
selection. That last check also confirms the SwiftUI `AttributedTextSelection` binding
observed the programmatic `setSelectedRange` — if completion goes stale or targets the
old caret, the binding did not sync, and the fix is to route the caret through the
model's selection instead of the text view.

## 8. Out of scope

- No wrap-around at the document edges (decision 3).
- No visual-line (soft-wrap-aware) movement (decision 1).
- No Ctrl-J/Ctrl-K row navigation in the stash picker or the completion list; Ctrl-N/
  Ctrl-P and the arrow keys keep that job (decisions 7 and 10).
- No change to Ctrl-I/Ctrl-J/Ctrl-K inside the Add block ID and Name Pomodoro fields;
  they keep native `NSTextField` behavior, exactly as Ctrl-J does today. That means
  Ctrl-K still deletes to end of field in those prompts. Leaving the editor and the
  prompts inconsistent is deliberate: it is the same split Ctrl-A/Ctrl-E/Ctrl-U already
  have.
- No replacement for the `deleteToEndOfParagraph:` behavior Ctrl-K loses in the editor
  (decision 5).
- No shift-selecting variants (Ctrl-Shift-J/K to extend a selection downward/upward).
- No user-configurable keymap.
- No `hasMarkedText()` IME guard. No sibling handler has one, and adding a divergent
  policy here is not this change's job.
- No fix for the Caps Lock interaction in decision 6; it is pre-existing across every
  Ctrl binding in this router and should be fixed for all of them at once or not at all.

## 9. Done when

1. Ctrl-I inserts the indentation-aware `- ` row that Ctrl-J used to insert, with
   identical behavior, and Ctrl-J no longer does.
2. Ctrl-J and Ctrl-K move the caret to the next and previous physical line, keeping the
   column across consecutive presses, clamping on short lines, and stopping at the last
   and first line.
3. Ctrl-K never deletes text: pressing it on the first line consumes the key and leaves
   the draft unchanged.
4. Control-Shift-Command-I still opens the capture panel and is not routed as Ctrl-I.
5. The router, resolver, and live-`NSTextView` tests from section 5 exist and pass in
   CI, including the updated Ctrl-I assertions at the four former Ctrl-J sites.
6. `grep -rn 'Ctrl-J' Sources Tests` returns only the new Ctrl-J caret-movement
   references, with no stale bullet-insertion ones.
7. `swiftc -parse` over the macOS sources and tests exits 0, and
   `swift build && swift test` for `CaptureCore` stays green on the Linux host.
8. The macOS 26 CI job is green on the PR.
9. The README Keyboard table and editing prose describe Ctrl-I, Ctrl-J, and Ctrl-K,
   including that the vertical moves stop rather than wrap and that Ctrl-K no longer
   deletes.
