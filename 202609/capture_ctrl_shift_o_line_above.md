---
tier: tale
title: Add Ctrl-Shift-O to insert a blank line above the current capture line
goal:
  "Bob Mac Capture's main draft editor handles Ctrl-Shift-O as the inverse of its
  existing native Ctrl-O open-line behavior: it inserts a blank physical line
  immediately above the caret's current line, moves the caret onto that new blank line,
  preserves the draft's line-ending convention, dismisses completion like other text
  edits, and leaves Ctrl-O plus every prompt and picker context unchanged."
size: small
proposed_by: bbugyi200.athena.02e
create_time: 2026-09-09 20:00:36
status: wip
---

<!-- sase:links:start -->

## Links

| Relation | Artifact                                         | Why                                                                                                               |
| -------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| related  | [plan:202609/capture_ctrl_u_previous_line.md][1] | Establishes native NSTextView edit, line-terminator, completion-dismissal, and edge-case test conventions.        |
| related  | [plan:202609/capture_line_edge_cycling.md][2]    | Establishes physical-line, exact-modifier, modal-isolation, and caret-scrolling conventions for editor shortcuts. |

[1]:
  https://github.com/bobs-org/bob-cli--plans/blob/main/202609/capture_ctrl_u_previous_line.md
[2]:
  https://github.com/bobs-org/bob-cli--plans/blob/main/202609/capture_line_edge_cycling.md

<!-- sase:links:end -->

# Plan: Add Ctrl-Shift-O to insert a blank line above the current capture line

## 1. Scope and repository

All implementation changes belong in the linked **`bob-mac-capture`** repository. The
coding agent must open it first and use only the path printed by:

```bash
sase repo open bob-mac-capture -r "Implement Ctrl-Shift-O line-above insertion in the capture editor"
```

No change is needed in `bob-cli`: the capture grammar, subprocess JSON contracts, and
vault mutation are unaffected. This is an AppKit editor-keymap feature.

Expected files:

| File                                                  | Responsibility                                                                  |
| ----------------------------------------------------- | ------------------------------------------------------------------------------- |
| `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` | Recognize exact Ctrl-Shift-O in the main editor and introduce its command.      |
| `Sources/BobMacCapture/CapturePanelController.swift`  | Resolve and apply the physical-line insertion through the backing `NSTextView`. |
| `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`   | Cover routing, pure edit resolution, live text-view behavior, and isolation.    |
| `README.md`                                           | Document the shortcut, its caret behavior, completion behavior, and scope.      |

Do not change any source before this plan is approved.

## 2. Current behavior verified during planning

- The app does not explicitly route Ctrl-O. `CaptureKeyCommandRouter` has no `o` key
  code (ANSI hardware key code `31`) and returns `nil`, after which
  `CapturePanelController`'s local key monitor returns the event to AppKit. The existing
  Ctrl-O behavior named in the request is therefore the backing `NSTextView`'s native
  open-line behavior, not code that should be rewritten or replaced.
- The main-editor switch is reached only after early routing for the canceled-draft
  stash picker, Add block ID prompt, and Name Pomodoro prompt. Keeping the new case in
  that editor switch preserves native text-field behavior in both prompts and prevents
  the editor edit from leaking into the picker.
- Every custom mutation of the draft first resolves an editable backing `NSTextView`,
  dismisses completion, and edits with `NSTextView.insertText` or an AppKit command.
  Directly assigning `textView.string` would bypass the established undo, input-method,
  accessibility, and SwiftUI synchronization path.
- Existing line-aware operations use UTF-16 `NSRange` values and
  `NSString.getLineStart`, so "line" consistently means a physical newline-delimited
  line rather than a visual soft-wrapped fragment. `CaptureBulletNewlineEditResolver`
  also preserves an existing CRLF or CR convention instead of always adding LF.
- Ctrl-J/Ctrl-K carry a sticky goal column in `verticalMovementGoal`; ordinary edits
  invalidate it indirectly when their resulting selection moves. A line-above edit can
  leave the caret at the same numeric UTF-16 offset, so this new path must explicitly
  clear that cached goal after a successful insertion.
- The macOS app and `BobMacCaptureTests` targets exist only under `#if os(macOS)` in
  `Package.swift`. A Linux worker can parse their Swift source but cannot type-check or
  run those AppKit tests; macOS CI remains the authoritative gate.

## 3. Behavior decisions

These are implementation decisions rather than open questions.

1. **Preserve Ctrl-O exactly as it is.** Route only ANSI `o` with the exact modifier set
   `[.control, .shift]`. Plain Ctrl-O must continue to return `nil` from the app router
   and reach AppKit. Plain O, Shift-O, Command-O, Ctrl-Option-O, Ctrl-Command-O, and any
   Ctrl-Shift-O combination that also contains Option or Command remain unrouted. This
   follows the exact-modifier convention of the neighboring Ctrl bindings and avoids
   stealing system/menu commands.

2. **Operate on physical lines.** The current line is the one containing a collapsed
   caret according to `NSString.getLineStart`. Soft wrapping does not create a line for
   this command. This matches Ctrl-I, Ctrl-U, Ctrl-A/Ctrl-E, Ctrl-J/Ctrl-K, and bullet
   indentation, and keeps the edit independent of view layout.

3. **Insert above without changing existing content.** Insert exactly one preferred line
   terminator at the current physical line's `lineStart`. This moves the original line
   down intact and creates a blank line in its former position. Do not copy bullet
   indentation, markers, or leading whitespace: Ctrl-I owns indentation-aware bullet
   creation, while this shortcut is a literal blank-line operation.

4. **Put the caret on the new blank line.** After insertion, collapse the selection to
   the original `lineStart`, before the inserted terminator. The user can type on the
   newly opened line immediately. Repeated Ctrl-Shift-O presses add further blank lines
   above and keep the caret on the newest one. Scroll that range into view.

5. **Require a collapsed, in-bounds caret.** A non-empty or invalid selection makes the
   resolver decline without changing text, selection, completion, or the cached
   vertical-movement goal; the event falls through to AppKit. A multi-line selection has
   no unambiguous "current line" in the existing `NSRange` representation, and this rule
   avoids silently discarding or collapsing selected text.

6. **Preserve the draft's line-ending convention.** Use the same preference as Ctrl-I:
   CRLF if any CRLF exists, otherwise CR if any CR exists, otherwise LF. Factor this
   small policy into a shared file-private helper used by both resolvers rather than
   cloning it. This is a behavior-preserving refactor for Ctrl-I and prevents the two
   line-insertion commands from drifting.

7. **Treat it as a normal text edit.** Resolve first; only on success dismiss
   completion, call `NSTextView.insertText(_:replacementRange:)`, set and reveal the
   resulting selection, and clear `verticalMovementGoal`. This preserves native
   undo/edit notifications and avoids dismissing completion when no editable draft
   responder is available.

8. **Limit the binding to the main draft editor.** Do not add it to
   `isolatedPromptCommand` or `stashPickerCommand`. Add block ID and Name Pomodoro keep
   native single-line text-field handling, and the stash picker must never mutate the
   hidden draft. Existing early returns enforce this, but tests must make it explicit.

9. **Do not add an IME-only exception or Caps Lock normalization here.** Sibling edit
   handlers rely on AppKit's edit path and exact modifier equality. Any broader policy
   change should cover all capture-editor shortcuts together, not only this one.

## 4. Implementation

### 4.1 Route the new command

In `CaptureKeyCommandRouter.swift`:

1. Add a command such as `.insertLineAbove` beside the other editor-editing cases.
2. Add `static let o: UInt16 = 31` to the private `KeyCode` namespace.
3. In the main editor switch only, map `KeyCode.o` to `.insertLineAbove` when
   `modifiers == [.control, .shift]`; return `nil` for every other modifier set.
4. Leave all modal routing functions unchanged. In particular, do not add a plain Ctrl-O
   command: its current AppKit fallthrough is part of the regression contract.

### 4.2 Resolve a deterministic line-above edit

In `CapturePanelController.swift`, add a small equatable edit value containing:

- `replacementRange`: a zero-length range at the current physical line start;
- `replacementText`: the preferred line terminator; and
- `resultingSelection`: a collapsed range at that same line start.

Add a pure resolver near `CaptureBulletNewlineEditResolver`. It accepts the draft string
and selected UTF-16 range, rejects a non-collapsed or out-of-bounds selection, obtains
the physical `lineStart` with `NSString.getLineStart`, and returns the edit above. It
must accept the empty draft and every valid document edge: those are real empty physical
lines, not failure cases.

Extract `CaptureBulletNewlineEditResolver`'s existing private preferred-terminator logic
to a shared file-private helper and call it from both the existing Ctrl-I resolver and
the new resolver. Do not otherwise alter Ctrl-I resolution.

### 4.3 Apply through the native text view

Add `insertLineAboveInEditableTextView(firstResponder:model:)` beside the existing
newline helpers:

1. Resolve `editableTextView(firstResponder)` and the pure line-above edit before any
   side effects; return `false` if either declines.
2. Dismiss completion.
3. Apply the edit with
   `textView.insertText(edit.replacementText, replacementRange: edit.replacementRange)`.
4. Set `edit.resultingSelection` and call `scrollRangeToVisible` for it.
5. Return `true` so the local event monitor consumes the keystroke.

Wire `.insertLineAbove` into the exhaustive `perform(_:)` switch. Because the sticky
Ctrl-J/Ctrl-K column belongs to the controller rather than the static helper, clear
`verticalMovementGoal` only when the helper reports success, then return that result.
Update the comment above `editableTextView(_:)` so its inventory includes Ctrl-Shift-O.

## 5. Tests

All tests belong in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, using the
existing `keyEvent(keyCode:modifiers:characters:)` factory and `NSTextView` test style.

### 5.1 Router contract

Add a focused router test that proves:

- key code `31` with exactly `[.control, .shift]` returns `.insertLineAbove`, both with
  and without completion visible;
- plain Ctrl-O returns `nil`, pinning the existing native fallthrough;
- unmodified O, Shift-O, Command-O, Ctrl-Option-O, Ctrl-Command-O, Ctrl-Shift-Option-O,
  and Ctrl-Shift-Command-O all return `nil`;
- Ctrl-Shift-O does not return the edit command in task-ID prompt, Pomodoro-name prompt,
  or stash-picker contexts. Use the realistic Ctrl-O control character `"\u{0F}"` for
  the modal assertions so `printableModalCommand` is exercised rather than bypassed by
  an empty `characters` value.

### 5.2 Pure resolver contract

Test exact edit ranges, terminators, resulting selections, and applied strings. Include
at least:

- a caret in the middle of the second line of `"one\ntwo\nthree"`: insert at UTF-16
  offset `4`, producing `"one\n\ntwo\nthree"` with the caret at `4`;
- a caret on the first line: insert at `0`, proving line-above works at the document
  start;
- a caret at the end of the last non-empty line: insert at that line's start, not at the
  caret;
- the empty draft: insert one LF at `0`, with the caret still at `0`;
- a trailing empty line (`"a\n"`, caret `2`) and a blank middle line (`"a\n\nb"`, caret
  `2`), proving empty physical lines are valid targets;
- CRLF and CR-only drafts, proving the full existing terminator is inserted and ranges
  remain UTF-16-correct;
- a line preceded by an emoji or other surrogate pair, proving offsets are not treated
  as Swift character indices;
- a non-collapsed selection, a negative range, and an out-of-bounds range, all of which
  return `nil`.

Also retain or extend the existing Ctrl-I CRLF/CR coverage so extracting the shared
terminator policy is demonstrably behavior-preserving.

### 5.3 Live `NSTextView` contract

Add `@MainActor` tests that prove the apply helper:

- inserts a blank line above a middle line, keeps all existing text intact, moves the
  caret onto the new blank line, and dismisses an open completion;
- can be invoked repeatedly, adding one line per invocation while leaving the caret on
  the newest blank line;
- works at the first line and in an empty draft;
- returns `false` and changes nothing for a non-editable text view, an unrelated
  responder, no responder, and a non-collapsed selection; completion must remain visible
  on every declined path.

Where practical, enable an undo manager on the test text view and assert one undo
restores the pre-insertion text. If AppKit does not expose stable undo behavior for a
standalone test view, cover undo in the manual macOS check instead of weakening the
existing `insertText` implementation pattern.

Add a controller-level regression for `verticalMovementGoal`: establish a carried column
with Ctrl-J/Ctrl-K, perform a successful Ctrl-Shift-O insertion whose selected UTF-16
offset can remain numerically unchanged, then verify the next vertical movement starts
from the blank line's actual column zero rather than the stale carried column.

## 6. Documentation

Update the README's main Keyboard table with a `Ctrl-Shift-O` row:

- in the editor, it inserts a blank physical line immediately above the current line and
  moves the caret onto it;
- while completion is visible, it performs the same edit and closes completion;
- in Add block ID and Name Pomodoro, it keeps native text-field behavior.

In the nearby editor-behavior prose, state that Ctrl-O remains AppKit's native open-line
binding and Ctrl-Shift-O is the app-owned line-above counterpart. Explain that the new
shortcut inserts only a line terminator (no automatic bullet or indentation), uses
physical lines, and works through the native text view so undo and editor
synchronization remain intact. Do not add it to the separate stash-picker key table.

## 7. Validation and verification

From the opened `bob-mac-capture` checkout:

1. On any host, parse all macOS source and test files to catch Swift syntax errors:

   ```bash
   swiftc -parse Sources/BobMacCapture/*.swift Tests/BobMacCaptureTests/*.swift
   ```

2. Run formatting lint and resolve every new error in changed files. Existing unrelated
   warnings are not license to introduce another:

   ```bash
   swift-format lint --recursive Package.swift Sources Tests
   ```

3. On Linux, keep the cross-platform target green with `swift build && swift test`, but
   report accurately that this does not compile or execute `BobMacCapture` or
   `BobMacCaptureTests`.
4. On macOS 26, run `just all`; the authoritative CI job must pass its format, build,
   complete Swift test, bundle, signature/plist, launch, and reinstall gates.
5. Manually in an installed development build, compare both shortcuts in a multiline
   draft: confirm Ctrl-O is unchanged; Ctrl-Shift-O opens a blank line above from the
   first, middle, last, and already-empty lines; typing begins on that new line; undo
   restores the draft; completion closes; and the key does not mutate the draft while a
   prompt or stash picker is active.

## 8. Out of scope

- Reimplementing, remapping, or otherwise claiming plain Ctrl-O.
- Visual/soft-wrapped line semantics or automatic indentation/bullet continuation.
- Defining line-above behavior for non-collapsed selections.
- Changing Ctrl-Shift-O inside the Add block ID field, Name Pomodoro field, or stash
  picker.
- General modifier normalization for Caps Lock or general IME-policy changes.
- Any bob-cli capture grammar, completion protocol, preview, or vault mutation change.

## 9. Done when

1. Exact Ctrl-Shift-O inserts one blank physical line above the collapsed caret's line,
   places the caret on that new line, and is undoable as a native text edit.
2. Plain Ctrl-O and all non-target modifier combinations retain their current AppKit
   behavior.
3. Completion dismissal, prompt/picker isolation, line-ending preservation, Unicode
   offsets, empty/edge lines, declined selections/responders, repeated insertion, and
   sticky vertical-column invalidation are covered by tests.
4. README documentation matches the shipped editor and modal behavior.
5. Local syntax/style/cross-platform checks pass, and the macOS 26 CI job passes the
   complete app test and packaging workflow.
