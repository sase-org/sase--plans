---
tier: tale
title: Autosize the Bob Mac Capture draft editor
goal:
  The capture popup presents a compact, content-sized multiline editor with reliable
  newline shortcuts and unobstructed completion and preview content.
size: medium
proposed_by: bbugyi200.athena.00w
create_time: 2026-09-09 19:59:42
status: wip
---

# Autosize the Bob Mac Capture draft editor

## Goal

Make the `bob-mac-capture` popup feel like a compact capture surface: the draft editor
opens at exactly one visual line, grows and shrinks with hard newlines and soft
wrapping, and remains usable for longer drafts without crowding the completion menu,
live preview, errors, or footer actions. Add an explicit Control-J newline shortcut
while preserving the existing attributed-text highlighting, selection, completion,
preview, accessibility, and capture contracts.

This is a `medium` tale because the work is substantial but bounded to one linked macOS
application and can be implemented and verified coherently by one agent. There is no
`bob-cli` grammar or JSON-contract change: `bob-cli` already accepts embedded newlines
and normalizes them as whitespace at parse/capture time.

## Current behavior and constraints

- `Sources/BobMacCapture/CapturePanelView.swift` gives the attributed `TextEditor` a
  hard-coded `minHeight` of 190 points, so an empty or short draft consumes several
  lines of space.
- The completion list is now an in-flow sibling below the editor (an earlier overlay was
  deliberately removed because it obscured text), but it can consume up to 220 points.
  That placement must remain non-overlapping, and the live preview must retain priority
  when vertical space is tight.
- `CapturePanelController.makePanel()` creates a 760-by-360 panel while the SwiftUI root
  declares a 620-by-420 minimum. The implementation must reconcile those competing size
  assumptions so the layout has a real, testable minimum rather than relying on hosting
  view compression.
- The editor uses the macOS 26 `TextEditor` binding to `AttributedString` and
  `AttributedTextSelection`. Replacing it with a plain vertical `TextField` would lose
  grammar highlighting; replacing it with a custom `NSTextView` would unnecessarily put
  selection, undo, input-method composition, and attributed updates at risk.
- Shift-Return and Option-Return are documented as newline shortcuts, but the key router
  currently lets completion acceptance win whenever the menu is visible. Control-J is
  not represented in the app's explicit command map even though the Cocoa text system
  commonly provides Emacs-style control bindings. The app should own and test the
  requested shortcut rather than depend on a user-remappable system default.

## Design

### Editor sizing and appearance

Keep the existing attributed `TextEditor` and wrap it in a private, reusable
`AutosizingCaptureEditor` view in `CapturePanelView.swift`. Measure a hidden,
non-accessible `Text` sizing peer constrained to the editor's actual usable text width,
using the same monospaced body font and line spacing. The measurement string must
contain a harmless trailing zero-width sizing character so both an empty draft and a
draft ending in `\n` reserve the correct final line. Re-measure when either draft
contents or available width changes, so explicit newlines, soft wraps, deletion,
completion acceptance, retained draft restoration, and panel resizing all produce the
right height.

Centralize the policy in an internal `CapturePanelLayout`/`CaptureEditorHeightPolicy`
helper rather than scattering magic numbers:

- Minimum: one rendered line plus the editor's established comfortable text insets.
- Natural growth: one line-fragment step at a time, based on measured rendered height,
  not newline counting (long wrapped lines therefore behave correctly).
- Maximum: six visual lines. After that, keep the card height fixed and let the native
  `TextEditor` scroll, ensuring long drafts and the caret remain accessible without
  displacing the rest of the popup.
- Round measured heights to display pixels and publish a new height only when it
  actually changes, avoiding SwiftUI geometry feedback loops and per-keystroke jitter.
- Apply only a short, restrained ease-out transition when the measured line count
  changes; disable it when Reduce Motion is enabled. Do not animate ordinary edits
  within the same line count.

Preserve the current 10-point content padding, material fill, 8-point corner radius,
disabled state, accessibility label, attributed binding, selection binding, and existing
text-change callback. Move the empty-state prompt into a true non-layout-affecting
overlay and align it to the editor's first baseline/text inset so the one-line card
neither grows for the placeholder nor looks vertically off-center.

### Completion and preview layout

Keep `CompletionList` as a normal `VStack` sibling directly below the editor—never put
it back inside an editor overlay. Bound it to approximately three fully visible rows and
retain its vertical `ScrollView`, selected-row auto-scrolling, material, shadow, pointer
acceptance, and accessibility semantics. The reduced visible-row budget is intentional:
the user can still traverse every candidate, while the live preview stays on screen as
the editor grows.

Give `PreviewPane` a fixed vertical ideal size/high layout priority and give the
completion scroll view the flexible vertical budget. Align the root frame to the
top-leading edge. In `CapturePanelController.makePanel()`, make the initial content
height and `contentMinSize` agree with the SwiftUI 620-by-420 minimum while preserving
the current 760-point initial width and resizable panel style. This establishes the
invariant that at the smallest supported panel size a six-line editor, bounded
completion viewport, preview, and footer remain separate; if optional diagnostics need
extra room, the completion viewport yields/scrolls before editor text or preview is
clipped.

### Explicit newline routing

Update `CaptureKeyCommandRouter` with the macOS J key code and route an exact Control-J
chord to `.insertNewline` whether or not completion is visible. For Return/Enter,
evaluate the existing Shift-Return and Option-Return newline chords before completion
acceptance, so the README's current promise remains true while a menu is open. Preserve
plain Return, Command-Return, Tab, completion navigation, and Escape behavior for their
existing modifier combinations.

Make `.insertNewline` an explicit AppKit responder action in `CapturePanelController`:
when the first responder is the capture text view, dismiss the now-stale completion,
send the standard `insertNewline:` editing command to that responder, and consume the
original event. Do not splice `model.plainDraft` directly; responder-based insertion is
what preserves the current selection, undo registration, input-method behavior, and the
normal binding callback that triggers remeasurement, parse, completion, and preview. If
the editor is not the first responder, pass the event through rather than inserting text
into an unrelated control.

## Implementation steps

1. In the linked `bob-mac-capture` repository, add the sizing policy and private
   autosizing editor composition to `Sources/BobMacCapture/CapturePanelView.swift`;
   replace the 190-point minimum editor with the measured/clamped height and correct the
   prompt overlay alignment.
2. In the same view file, cap the completion viewport to the three-row budget, retain it
   in normal layout flow, and set vertical sizing/layout priority so the preview and
   footer never become overlay targets or get clipped before the completion list
   scrolls.
3. In `Sources/BobMacCapture/CapturePanelController.swift`, reconcile the panel's
   initial and minimum content sizes and add responder-based newline insertion with
   stale completion dismissal.
4. In `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, add explicit Control-J
   routing and make newline modifier precedence independent of completion visibility
   without altering unrelated shortcuts.
5. Update the keyboard and editor-behavior sections of `README.md` to document
   Control-J, one-through-six-line growth, internal scrolling for longer drafts, and the
   fact that multiline editing is still whitespace-normalized by `bob-cli` at capture
   time.

## Automated verification

Extend `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` (or add a focused
capture-panel layout test file if clearer) with regression coverage for:

- Control-J mapping to `.insertNewline` with completion both hidden and visible.
- Shift-Return and Option-Return continuing to map to `.insertNewline` while completion
  is visible, while plain Return still accepts completion and unrelated Control chords
  keep their existing behavior.
- The editor height policy returning exactly its one-line minimum for empty/short
  content, growing for larger rendered measurements, shrinking again, and clamping at
  six lines.
- The panel's initial content size satisfying its declared minimum while retaining the
  nonactivating, floating, resizable style already under test.
- Any factored responder helper inserting through an `NSTextView` first responder and
  declining to act for an unrelated responder, if that helper can be tested without
  reaching into SwiftUI internals.

Run the repository's supported macOS checks through its Xcode-resolved wrapper:

```sh
just format-lint
just build
just test
```

## macOS interaction and visual verification

Because the final behavior depends on SwiftUI/AppKit text layout, verify a bundled or
development build on macOS 26 in addition to unit tests:

1. Open a fresh panel and confirm the empty editor is one line tall, the prompt baseline
   is centered, focus/caret are immediate, and preview/footer remain visible.
2. Type within one line, cross a soft-wrap boundary, insert several hard newlines with
   Control-J (including at the beginning, middle, end, and after a trailing newline),
   and delete back to one line. Confirm smooth one-line-step growth/shrinkage with no
   flicker.
3. Reach seven or more visual lines and confirm the editor stops at six, scrolls
   natively, and keeps the insertion point visible. Resize the panel narrower/wider and
   confirm soft wrapping reflows the height within the same cap.
4. Trigger completion with the editor at one line and near its cap. Confirm the list is
   below—not over—the text, shows/scrolls about three rows, keyboard selection remains
   visible, and the live preview is readable throughout. Press Control-J, Shift-Return,
   and Option-Return while completion is open and confirm each inserts at the caret and
   dismisses the stale list; plain Return must still accept the selected completion.
5. Confirm completion acceptance, parse highlighting, undo/redo, selection replacement,
   paste of multiline text, a retained draft after Escape/failure, and successful
   capture reset all preserve caret/content and recompute height correctly.
6. Repeat the line-growth check with Reduce Motion and with VoiceOver enabled; verify no
   unwanted animation, duplicate prompt announcement, focus loss, or inaccessible
   completion/preview content. Exercise an input method/composed-text entry once to
   catch responder-command regressions.

## Acceptance criteria

- A fresh or successfully reset capture editor occupies exactly one visual line; it
  grows and shrinks with rendered content up to six lines, then scrolls internally.
- Control-J reliably inserts `\n` at the current selection and triggers normal height,
  parse, completion, and preview updates. Shift-Return and Option-Return do the same
  even while completion is visible.
- Completion never overlays editor text or the preview, remains fully keyboard
  navigable, and scrolls its selected row into view within its bounded viewport.
- Preview, footer actions, highlighting, accessibility, selection, undo, retained
  drafts, and capture submission retain their existing behavior.
- `bob-cli` source and capture contracts remain unchanged, documentation describes the
  final interaction accurately, and formatting, build, and test commands pass on macOS.
