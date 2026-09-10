---
tier: tale
title: Add a Tab-expanded em-dash snippet to Bob Mac Capture
goal:
  Plain Tab expands an immediately preceding `--` to `—` without regressing completion,
  bullet indentation, modal key handling, or native focus traversal.
size: small
proposed_by: bbugyi200.athena.0dw
create_time: 2026-09-09 20:00:01
status: wip
---

# Add a Tab-expanded em-dash snippet to Bob Mac Capture

## Outcome

Teach the native Bob Mac Capture editor one local typing snippet: when a collapsed caret
sits immediately after `--`, pressing plain Tab replaces those two ASCII hyphens with
the single Unicode em dash `—` and leaves the caret immediately after it. Keep Bob CLI
authoritative for capture grammar, parsing, completion, preview, and writes; this is an
AppKit editor assist in the linked `bob-mac-capture` repository only.

## Current behavior and constraints

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` routes plain Tab to completion
  acceptance when completion is visible and otherwise to continuation-bullet
  indentation. Shift-Tab routes to outdent. Stash-picker and Add-block-ID modal states
  claim their own Tab behavior before the normal editor path.
- `Sources/BobMacCapture/CapturePanelController.swift` performs direct editor commands
  against the first-responder `NSTextView`. Bullet indentation already uses a pure
  `NSString`/`NSRange` resolver and `NSTextView.insertText` so AppKit retains ownership
  of UTF-16 selection coordinates, undo, input methods, accessibility, and ordinary
  text-change callbacks.
- If the current Tab action cannot indent, the controller returns `false`, allowing
  AppKit's normal focus traversal. The new snippet must preserve that fallback.
- No capture-snippet facility currently exists in Bob Mac Capture, Bob CLI, or the
  retired Hammerspoon capture implementation. Do not route this local punctuation
  substitution through `bob capture-complete` or broaden Bob's capture grammar.

## Behavioral contract

Use this precedence for plain Tab:

1. Existing stash-picker and Add-block-ID modal handling remains unchanged.
2. A visible Bob completion remains first and accepts the selected completion exactly as
   it does today.
3. Otherwise, if the editable capture text view has a collapsed selection and the two
   UTF-16 code units immediately before the caret are exactly `--`, replace only that
   trigger with `—`, preserve all text before and after it, collapse the caret after the
   replacement, and consume Tab.
4. If no snippet matches, retain the current continuation-bullet indentation attempt.
5. If neither a snippet nor indentation applies, return the event to AppKit for normal
   focus traversal.

The trigger is caret-adjacent rather than whole-line or whitespace-delimited, so it can
be used in prose and before existing suffix text. A non-collapsed selection never
expands a snippet; it continues through the existing indentation/fallthrough behavior.
Shift-Tab, modified Tab combinations, completion acceptance, and modal Tab handling do
not gain snippet behavior.

## Implementation

### 1. Add a deterministic snippet resolver

In the `BobMacCapture` executable target, add a small editor-snippet abstraction near
the other native text-edit helpers (a focused source file is preferred over adding more
unrelated responsibilities to the panel controller):

- Represent the supported trigger/replacement pair as `--` -> `—` in one explicit
  snippet catalog so matching and the list of user-facing snippets have a single code
  source.
- Define an equatable edit value containing the replacement range, replacement text, and
  resulting collapsed selection.
- Implement a pure resolver over `NSString` plus `NSRange`. Validate the selection
  bounds, require a collapsed caret, inspect only the characters immediately preceding
  the caret, and return `nil` without mutation when no trigger matches.
- Express ranges in AppKit's UTF-16 coordinate system. Replacing the two-character
  trigger with one em dash must move the caret back by one UTF-16 code unit while
  preserving any Unicode prefix and any suffix after the caret.

Keep this deliberately local and data-driven, but do not add user settings, persistence,
template placeholders, automatic expansion while typing, or Bob CLI wire-format changes
for a one-entry snippet catalog.

### 2. Integrate the resolver into plain-Tab handling

Update `CaptureKeyCommandRouter.swift` and `CapturePanelController.swift` so the command
name and implementation describe plain Tab's ordered editor assists rather than
pretending it can only indent:

- Preserve the router's existing completion-visible branch and every higher-priority
  modal branch.
- For ordinary plain Tab, perform snippet expansion against the editable first responder
  first. Apply a resolved edit with `NSTextView.insertText`, explicitly restore the
  resolver's selection, and dismiss stale completion state only after an edit is
  actually accepted.
- When expansion declines, immediately try the existing bullet-indentation helper with
  `.increase`; when indentation also declines, return `false` so the monitored key event
  reaches AppKit unchanged.
- Leave Shift-Tab wired directly to `.decrease`, and leave Command/Option/Control Tab
  combinations untouched.

Do not mutate `CapturePanelModel.plainDraft` directly. Native text editing must continue
to drive the existing SwiftUI/AppKit update path so highlighting, rewrite analysis,
preview, and future completion requests see the expanded draft normally.

### 3. Cover matching, precedence, and native editing with tests

Extend `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` (or add a focused test file
under the same test target if that keeps the new abstraction clearer) with:

- Pure resolver cases for a draft containing only `--`, a trigger after ordinary and
  non-ASCII prefix text, a trigger before suffix text with the caret between trigger and
  suffix, and correct replacement/caret ranges.
- Negative cases for a non-collapsed selection, insufficient/mismatched preceding text,
  invalid selection bounds, and a caret that is not immediately after the trigger.
- Native `NSTextView` application showing that the edit replaces only `--`, preserves
  surrounding content, places the caret after `—`, consumes the action, and dismisses
  stale completion state only on success.
- Fallback coverage showing that a nonmatching Tab still indents an eligible
  continuation bullet, while a nonmatching/non-bullet editor state declines without
  changing text or selection so normal focus traversal remains possible.
- Key-router assertions that plain Tab without visible completion selects the combined
  snippet/indent action, visible completion still selects acceptance, Shift-Tab still
  selects outdent, modal contexts still consume/route Tab as before, and unsupported
  modifier combinations still return `nil`.

Prefer assertions on the pure edit and final text/selection over synthesizing a full
window key event; the existing router tests already cover key-code dispatch separately.

### 4. Document the user-visible shortcut

Update `README.md`'s Keyboard table and nearby native-editor-assist explanation to state
that plain Tab first expands an immediately preceding `--` to `—`, otherwise performs
the existing continuation-bullet indentation, and otherwise uses normal focus traversal.
Explicitly preserve the completion-visible and Add-block-ID columns so the documented
precedence is unambiguous.

## Non-goals

- No changes to the primary `bob-cli` repository, capture grammar, JSON interfaces, or
  completion candidate protocol.
- No configurable or persisted snippet collection, settings UI, placeholder/tab-stop
  engine, background expansion, smart punctuation beyond `--`, or global macOS text
  replacement.
- No change to Shift-Tab outdent semantics, completion selection, task-ID prompts,
  stash-picker modality, or bullet indentation depth rules.

## Verification

From the linked `bob-mac-capture` repository:

1. Run focused `BobMacCaptureTests` while iterating on the resolver, native application,
   and key-routing cases.
2. Run `just format-lint` to enforce the repository's Swift formatting rules.
3. Run `just build` and `just test` to compile both targets and execute the full SwiftPM
   suite.
4. Run `just bundle` (or `just all`) to exercise the same application packaging path as
   CI. Verify the existing signing/plist checks when available in the macOS toolchain.

If the implementation environment lacks macOS 26 / Apple Swift 6 tooling, record that
platform limitation precisely and rely on the macOS CI job for the unavailable build,
test, and bundle gates; do not weaken or rewrite the XCTest coverage.
