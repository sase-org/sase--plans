---
tier: tale
title: Add Control-U line-prefix deletion to Bob Mac Capture
goal:
  Control-U deletes from the capture editor caret to the current physical line start
  using native AppKit editing behavior.
size: small
proposed_by: bbugyi200.athena.02h
create_time: 2026-09-09 19:59:59
status: wip
---

# Add Control-U line-prefix deletion to Bob Mac Capture

## Goal

Add an editor-only `Control-U` keymap to the capture panel owned by the linked
`bob-mac-capture` repository. Pressing the exact shortcut must delete the text from the
caret back to the beginning of its current physical line while preserving the line
delimiter, every earlier line, and the text after the caret.

## Context and constraints

- The change belongs entirely to `bob-mac-capture`; `bob-cli` remains responsible for
  capture grammar and does not need a command-line or JSON contract change.
- The panel already translates `NSEvent` key-down events into `CaptureKeyCommand` values
  in `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, then lets
  `CapturePanelController` apply editor mutations through the first-responder
  `NSTextView`. Extend that path instead of editing the SwiftUI-bound draft directly.
- Match only the exact Control-U modifier combination. Plain U and variants containing
  Shift, Command, or Option must remain available to normal AppKit behavior.
- The shortcut must behave the same while completion is visible and dismiss completion
  after it reaches an editable text view, so the completion UI cannot describe text that
  was just removed.
- Preserve the stash picker's existing modal routing precedence: its context-specific
  branch runs before ordinary editor commands, so Control-U must not become a stash
  picker command.
- Use AppKit's native beginning-of-line deletion command on the editable first responder
  so line-boundary semantics, undo registration, input-method behavior, accessibility,
  and UTF-16/Unicode handling remain owned by the text system. If the first responder is
  absent, unrelated, or noneditable, decline the command and let the original event fall
  through.

## Implementation

1. Extend `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` with a dedicated capture
   command and the physical U key code. Route exact Control-U to that command from the
   ordinary panel/editor switch, independent of completion visibility, without changing
   the higher-priority stash-picker branch or any existing shortcut.
2. Extend `Sources/BobMacCapture/CapturePanelController.swift` with a small helper that
   resolves the editable `NSTextView`, dismisses active completion only after that
   resolution succeeds, invokes the native `deleteToBeginningOfLine:` responder command,
   and reports whether the event was consumed. Dispatch the new routed command through
   this helper and update the nearby native-editing commentary to include Control-U.
3. Expand `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` with regression coverage
   for routing and behavior:
   - exact Control-U routes both with and without completion, while plain U and modified
     variants do not;
   - the stash-picker routing context does not expose the ordinary editor shortcut;
   - deleting from the middle of a later line removes only that line's prefix, preserves
     the prior line, line delimiter, Unicode-safe suffix, and places the caret at the
     current line start;
   - a successful edit dismisses completion; and
   - noneditable, unrelated, and missing responders are declined without changing text
     or completion state.
4. Update the keyboard table and native-editing explanation in `README.md` so Control-U
   is discoverable and its beginning-of-physical-line behavior is unambiguous in both
   normal and completion-visible editor states.
5. From the `bob-mac-capture` repository, run `just format-lint`, `just build`, and
   `just test`. Fix any formatting, compiler, or test regressions before handing off the
   implementation.

## Acceptance criteria

- Exact Control-U in the capture editor deletes only the content between the caret and
  the beginning of the current physical line, leaving the newline and remaining suffix
  intact.
- The shortcut is consumed only for an editable text-view first responder, works while
  completion is visible, and dismisses stale completion state after handling the edit.
- Other U modifier combinations, noneditor responders, the stash picker, and every
  existing panel shortcut retain their prior behavior.
- The implementation uses the native AppKit text command rather than model-level string
  slicing, and the documented full lint/build/test verification passes.

## Non-goals

- Do not add a global hotkey, menu item, preference, or configurable keybinding system.
- Do not change Bob capture grammar, completion protocols, or vault mutation behavior.
- Do not define custom paragraph-wrapping semantics; the AppKit physical-line command is
  authoritative for the editor.
