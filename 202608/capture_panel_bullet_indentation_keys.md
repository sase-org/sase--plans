---
tier: tale
title: Add Tab and Shift-Tab bullet indentation to Bob Mac Capture
goal:
  Let users promote and demote the current authored capture bullet between Bob's
  column-zero and exact two-space levels without leaving the macOS capture panel.
size: small
proposed_by: bbugyi200.athena.022.f0.f0
create_time: 2026-09-09 19:59:52
status: wip
---

# Add Tab and Shift-Tab bullet indentation to Bob Mac Capture

## Scope and repository

Implement this tale entirely in the linked `bob-mac-capture` repository. Open it first
through the required audited path:

```sh
sase repo open bob-mac-capture -r "Implement Tab and Shift-Tab bullet indentation in the capture panel"
```

Use the printed checkout path for every read and write. The planning-time head is
`15a0e38 feat: decode nested capture parse depths`; re-read the named files if the
branch has advanced before implementation.

No `bob-cli` source or wire-contract change is required. Bob already owns and enforces
the bounded authored-child grammar:

- physical line 1 is the captured parent;
- later column-zero `-`/`*`/`+` rows are first-level authored children;
- the same rows prefixed by exactly two ASCII spaces are nested authored children;
- deeper, one-space, tab-indented, or otherwise malformed nonempty continuation rows are
  rejected; and
- `capture-parse` already reports the resulting depths while preview and capture render
  nested rows beneath the corresponding first-level authored row.

This feature is an editor convenience for changing those two raw source prefixes. It
must continue sending the resulting draft verbatim to Bob for parsing, diagnostics,
preview, completion, and capture rather than introducing a second Swift capture parser.

This is a small tale because the behavior is confined to the existing macOS key router,
the controller's native `NSTextView` edit helpers, focused AppKit tests, and keyboard
documentation. It does not change the model, view hierarchy, fake-bob fixture,
CaptureCore models, JSON schemas, or Rust implementation.

## Current behavior and diagnosed gap

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` knows virtual key code `48` as
  Tab. Its Tab arm currently returns `.acceptCompletion` whenever completion is visible
  and `nil` otherwise. Because it does not inspect modifiers, Shift-Tab also accepts a
  visible completion today. With no completion, both chords fall through to AppKit focus
  traversal.
- `Sources/BobMacCapture/CapturePanelController.swift` already centralizes direct
  backing-`NSTextView` edits for Shift/Option-Return, Ctrl-J, and the empty-row
  Backspace shortcut. Those helpers resolve only an editable first responder, dismiss
  completion on a successful edit path, and use native text-system commands or
  `insertText(_:replacementRange:)`, preserving SwiftUI binding updates, undo, IME, and
  accessibility.
- The newly supported nested draft syntax therefore has no keyboard route: users must
  type or remove the exact two leading spaces manually, despite the editor already
  intercepting neighboring structural keys.

## Required keyboard contract

Add two semantic commands, named consistently with the existing router (for example
`.increaseBulletIndentation` and `.decreaseBulletIndentation`), with these exact routing
rules:

1. **Plain Tab, completion hidden:** route to increase indentation.
2. **Plain Tab, completion visible:** preserve the existing completion contract and
   route to `.acceptCompletion`; accepting a route, wikilink, or other candidate takes
   precedence over indentation.
3. **Shift-Tab:** route to decrease indentation whether or not completion is visible.
   This deliberately replaces today's accidental Shift-Tab completion acceptance. A
   successful outdent dismisses the now-stale completion and lets the normal text-change
   analysis rebuild parse, preview, highlighting, and any applicable completion state.
4. **Every other Tab modifier combination** (Command, Option, Control, or combinations
   beyond Shift alone) returns `nil` and remains AppKit's. Use exact modifier equality,
   matching the router's Ctrl-J and Control-C safety policy; do not broaden this tale
   into global modifier normalization.
5. If an indentation command cannot edit the current responder or row, return `false`
   from the controller so the original event falls through. Thus Tab/Shift-Tab retain
   normal forward/reverse focus traversal outside an applicable bullet rather than
   becoming swallowed no-ops.

## Exact editing behavior

Operate on one physical continuation line at a time, located from the backing
`NSTextView`'s current UTF-16 `selectedRange()`:

- Accept a collapsed caret or a selection wholly contained in one physical line. A
  selection that includes a line delimiter or spans more than one physical line is out
  of scope and must decline unchanged; multi-row indentation can be designed separately
  without guessing at partial selections or mixed depths.
- Never transform physical line 1, even when its captured-parent text happens to begin
  with a Markdown marker. Only continuation rows participate in authored-child
  indentation.
- Recognize `-`, `*`, and `+` at the relevant prefix when the marker is at end of line
  (an interactive placeholder) or is followed by at least one ASCII space or tab. Do not
  modify prose, `-body`, one-space/deeper malformed rows, leading tabs, blank rows, or
  arbitrary Markdown-looking text.
- **Increase:** on an exact column-zero continuation bullet, insert exactly two ASCII
  spaces at that physical line's start. Decline if it already has two leading spaces;
  the panel must never create a third authored depth. The edit is intentionally a
  bounded source transformation, not a Swift owner-association parser: Bob's existing
  live parse remains authoritative for contextual errors such as indenting a nonempty
  first bullet with no preceding owner.
- **Decrease:** on an exact two-space continuation bullet, delete exactly those two
  leading ASCII spaces. Decline on a column-zero bullet; there is no authored level
  above it within a capture draft.
- Marker-only placeholder rows such as `-`, `- `, `  -`, and ` -` are eligible at their
  exact current depth. This makes the intended interactive flow work immediately after
  Ctrl-J creates `- `, before the user types the body.
- Preserve the marker character, separator run, body text, capture markers, Unicode,
  line ending, and every unrelated line byte-for-byte. Do not normalize `*`/`+` to `-`
  in the editor; Bob performs canonical rendering only at capture time.
- Transform selection endpoints through the two-character prefix edit so a caret stays
  at the same logical position in the bullet body and a same-line selection keeps the
  same logical text selected. On outdent, clamp an endpoint that was inside the removed
  prefix to the new line start. Use `NSString`/`NSRange` line APIs because
  `NSTextView.selectedRange()` is UTF-16-based; cover non-ASCII body text and LF/CRLF
  boundaries in tests.

Factor the line/range decision into a small pure or deterministic helper (for example,
an indentation-edit value carrying replacement range, replacement text, and resulting
selection) and keep the responder mutation in the controller. Apply the edit through
`NSTextView.insertText(_:replacementRange:)`, then restore the computed selection. Do
not assign `CapturePanelModel.plainDraft` directly or mutate `textStorage` behind the
text system: the native edit path is what supplies undo registration, accessibility, IME
correctness, and the `TextEditor` binding callback that already schedules Bob's
analysis.

Dismiss completion only after the helper has proved an edit is applicable and just
before applying it. A declined command must leave both the text and completion state
untouched so falling through to AppKit is genuine.

## Implementation

1. **Route the chords in `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`.**
   - Add the two indentation commands to `CaptureKeyCommand`.
   - Replace the broad Tab arm with the exact precedence/modifier matrix above.
   - Keep Return, Ctrl-J, Backspace, Escape/Ctrl-[, Control-C, arrow, and Ctrl-N/P
     behavior unchanged.

2. **Implement and apply the bounded edit in
   `Sources/BobMacCapture/CapturePanelController.swift`.**
   - Add an internal direction/edit representation and a deterministic resolver for a
     draft string plus `NSRange`, or equivalently a static resolver against a headless
     text view. Keep it narrow: identify the one physical line and the two supported
     source prefixes; do not inspect route markers, parse JSON, or infer capture output.
   - Add a static editable-responder helper in the style of
     `insertBulletNewlineInEditableTextView` and
     `deleteEmptyBulletRowInEditableTextView`. It returns `false` without state changes
     for noneditable/unrelated responders and inapplicable lines, and otherwise
     dismisses completion, applies one native prefix edit, restores the transformed
     selection, and returns `true`.
   - Extend `perform(_:)` with the new cases and return the helper's result so the local
     key monitor consumes only a real edit.
   - Update the nearby controller comment that currently names only Ctrl-J and Backspace
     as direct native draft edits.

3. **Add focused regression coverage in
   `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`.** Use the existing
   `keyEvent(keyCode:modifiers:)`, `sampleCompletionResponse()`, and headless editable
   `NSTextView` patterns.

   Router coverage must pin:
   - plain Tab with completion hidden -> increase;
   - plain Tab with completion visible -> accept completion;
   - Shift-Tab with completion hidden or visible -> decrease;
   - Command/Option/Control-Tab and extra-modifier Shift-Tab variants -> `nil`; and
   - every existing shortcut remains represented in the router matrix.

   Resolver/application coverage must pin:
   - `-`, `*`, and `+` first-level rows indent to exactly two spaces;
   - nested rows outdent to column zero;
   - empty placeholder rows work in both directions;
   - the caret before the marker, in the prefix, after the marker, and after Unicode
     body text is transformed predictably;
   - a same-line selection retains its logical selection, including the outdent clamp
     when it intersects the removed prefix;
   - LF and CRLF drafts change only the selected continuation row;
   - physical line 1, blank/prose/malformed rows, already nested rows on increase,
     already top-level rows on decrease, and multiline selections decline byte-for-byte;
   - noneditable, unrelated, and `nil` responders decline;
   - a successful edit dismisses a visible completion, while a declined edit preserves
     it; and
   - the resulting native edit is undoable back to the exact original string and
     selection (exercise the `NSTextView` undo manager if stable in the headless test;
     otherwise make this an explicit macOS smoke assertion and retain the native
     insertion API by review).

   No fake-bob or CaptureCore fixture expansion is needed: existing model/process tests
   already prove nested drafts remain one argv element and preview/capture use Bob's
   rendered hierarchy. These tests should stay focused on key routing and editor
   mutation.

4. **Update `README.md`.**
   - Replace the current Tab editor cell `(normal focus traversal)` with the applicable
     first-level-bullet indentation behavior while retaining “Accept the selected
     completion” in the completion column.
   - Add a Shift-Tab row documenting two-space outdent in both states, plus normal
     reverse focus traversal when the current row cannot be outdented.
   - Replace the prose that tells users to type two leading spaces manually with the new
     Ctrl-J -> Tab workflow, while keeping exact two-ASCII-space syntax explicit for
     pasted/hand-authored drafts.
   - State that Tab/Shift-Tab edit the native text view, preserve
     undo/IME/accessibility, stop at Bob's two supported levels, and leave Bob
     authoritative for validation and rendering.

## Validation

1. Inspect `git status --short` and the final diff in the linked repository. Only the
   router, controller, their focused tests, and README should change; run
   `git diff --check`.
2. On a macOS 26 host with Apple's selected toolchain, run from `bob-mac-capture`:

   ```sh
   just format-lint
   just build
   just test
   ```

   The AppKit/SwiftUI executable and `BobMacCaptureTests` cannot be compiled on the
   Linux SASE host. If the implementing host has the same limitation, run every
   available source/whitespace check, report the exact toolchain failure, and use the
   repository's `macos-26` GitHub Actions workflow as the automated
   format/build/test/bundle/launch gate. Do not describe unrun AppKit tests as passed.

3. On macOS, install a bundled build (`just bundle && just install`) and smoke test:
   - Draft `Parent\n- First\n- Second`, put the caret anywhere in `Second`, press Tab,
     and confirm the raw draft is `Parent\n- First\n  - Second`; live/explicit preview
     and a real capture render `Second` beneath `First` in the target note.
   - Press Shift-Tab on `Second`; it returns to column zero and preview/capture show it
     as a sibling again.
   - After Ctrl-J inserts `- ` following an existing first-level item, press Tab before
     typing; the placeholder becomes ` -` and the typed body remains nested.
   - Trigger route or wikilink completion on a bullet: plain Tab still accepts it.
     Trigger completion on a nested bullet and press Shift-Tab: the row outdents, the
     stale completion closes, and live analysis refreshes.
   - On the parent line, prose, a malformed row, the depth ceiling/floor, and a
     multiline selection, confirm the key falls through rather than changing or
     swallowing text.
   - Verify Command-Z/redo restore the indentation edit and caret/selection, and that
     keyboard focus traversal still works when the command declines.

## Acceptance criteria

- From the panel, plain Tab converts one applicable column-zero authored continuation
  bullet to the exact two-space nested form, and Shift-Tab converts it back without
  altering its marker, body, selection, line ending, or neighboring lines.
- The panel never creates unsupported deeper indentation; non-bullets, the captured
  parent, malformed rows, and multiline selections remain untouched and retain native
  focus behavior.
- Plain Tab still accepts a visible completion. Shift-Tab performs outdent rather than
  accepting completion, and completion is dismissed only when that edit succeeds.
- Native undo, IME, accessibility, SwiftUI draft binding, highlighting, diagnostics,
  live preview, and capture refresh through the existing text-system path.
- Bob remains the only capture grammar and rendering authority: the app changes only the
  documented raw two-space prefix, and the existing parse/preview/capture pipeline
  decides whether the resulting hierarchy is contextually valid.
- All available formatting, build, tests, whitespace checks, documentation review, and
  macOS smoke checks pass, with host-toolchain limitations reported accurately.
