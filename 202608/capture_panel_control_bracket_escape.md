---
tier: tale
title: Recognize Control-[ as Escape in the Bob Mac Capture panel
goal:
  Control-[ follows the capture panel's existing non-destructive Escape behavior in
  every panel state.
size: small
proposed_by: bbugyi200.athena.00w.f0.f0.w0.w0.w0.f0
create_time: 2026-09-09 19:59:52
status: wip
---

# Recognize Control-[ as Escape in the Bob Mac Capture panel

## Context

Implement this work in the linked `bobs-org/bob-mac-capture` repository. Open it first
and use only the path printed by SASE for repository reads and writes:

```sh
sase repo open bob-mac-capture -r "Implement Control-[ as an Escape alias in the capture panel"
```

Base the implementation on the latest `origin/master`
(`984852c feat!: make capture panel close retain drafts by default` at planning time).
This is a macOS presentation and key-routing change only. It does not require a
`bob-cli` change and must not alter the versioned `bob capture`, `capture-parse`, or
`capture-complete` subprocess/JSON contracts.

The relevant path is already centralized:

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` converts an AppKit `NSEvent`
  into a semantic `CaptureKeyCommand`. Physical Escape currently maps to `.escape`.
- `Sources/BobMacCapture/CapturePanelController.swift` applies `.escape`: when a
  completion list is visible it dismisses that list; otherwise it closes the panel via
  `closeRetainingDraft()`. Both branches consume the event.
- `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` already tests physical Escape at
  the router and tests both controller-level `.escape` outcomes.
- `README.md` documents the public keyboard contract.

The macOS virtual key code for the ANSI left-bracket key is `33`. The router already
uses virtual key codes for every capture shortcut, so the new chord should follow that
existing representation instead of inspecting `characters` (which may already be the
Escape control character for Control-[). Routing this chord to the existing `.escape`
command is important: it makes the alias inherit completion-first behavior, retained
draft close behavior, event consumption, and any future Escape changes without adding a
second controller path.

## Goal and acceptance criteria

- Pressing Control-[ while the capture panel is key has the same semantic behavior as
  pressing Escape.
- With no completion visible, one Control-[ closes the panel and retains a nonempty
  draft. Reopening shows the retained draft and the existing `Draft retained` status.
- With a completion visible, the first Control-[ dismisses only the completion and
  leaves the panel open; a subsequent Control-[ closes the panel and retains the draft.
- The chord is consumed when handled and does not insert a bracket or Escape character
  into the editor.
- Only plain Control-[ is added. Bare `[`, Shift-[, Command-[, Option-[,
  Control-Shift-[, and Control-Command-[ continue through normal AppKit/editor handling
  and do not close the panel through this route.
- Physical Escape keeps its current behavior, including completion-first dismissal.
- Control-C remains the destructive discard-and-close shortcut; Control-[ never discards
  or clears the draft.
- Return variants, Control-J, Tab, arrows, Control-N, and Control-P remain unchanged.
- The README presents Escape and Control-[ as equivalent keys for both ordinary editor
  and completion-visible states.

## Design decisions and scope

1. **Reuse `.escape`; do not add a new command.** Add a router alias only. No change is
   needed in `CaptureKeyCommand`, `CapturePanelController`, `CapturePanelModel`, or the
   panel view because the existing `.escape` command already implements the complete
   requested behavior.
2. **Match modifiers strictly.** Return `.escape` only when the normalized
   device-independent modifier flags equal `.control`, matching the router's existing
   strict Control-C and Control-J rules. Extra Shift or Command modifiers therefore
   cannot close the panel accidentally. This intentionally preserves the router's
   existing treatment of additional device-independent flags such as Caps Lock; global
   modifier normalization would affect other shortcuts and is outside this change.
3. **Use the router's virtual-key convention.** Name the new key-code constant clearly
   (for example, `leftBracket`) and assign `UInt16 = 33`. This is the same physical-key
   strategy used by the existing letter and navigation shortcuts. Supporting arbitrary
   keyboard-layout character translation would require a broader input-policy design and
   is not part of this focused alias.
4. **Avoid redundant state tests.** Existing controller tests already prove that
   `.escape` retains a draft when closing and dismisses completion first. New tests
   should prove that Control-[ reaches that exact command in both visibility states and
   that nearby modifier combinations are not captured. Do not duplicate model tests or
   introduce test-only controller access solely for this alias.

## Implementation

1. Update `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`:
   - Add `static let leftBracket: UInt16 = 33` to the private `KeyCode` namespace.
   - Add a `case KeyCode.leftBracket` switch arm that returns `.escape` only when
     `modifiers == .control`; otherwise return `nil` so AppKit can process the event.
   - Keep the physical `KeyCode.escape` arm and all other shortcut arms unchanged. The
     new result must not depend on `completionVisible`; the controller decides what
     `.escape` means from current model state.

2. Extend `testKeyRouterMatchesCaptureShortcuts` in
   `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` (or use a focused sibling test if
   that reads more clearly), using the existing `keyEvent(keyCode:modifiers:)` helper:
   - Assert key code `33` plus `.control` maps to `.escape` with completion hidden.
   - Assert the same event maps to `.escape` with `completionVisible: true`.
   - Assert key code `33` with no modifiers, `.shift`, `.command`, `.option`,
     `[.control, .shift]`, and `[.control, .command]` returns `nil`.
   - Retain the existing physical-Escape and Control-C assertions. Together with the
     existing `testEscapeClosesOnceAndRetainsNonemptyDraft` and
     `testEscapeDismissesCompletionBeforeClosingPanel`, these assertions cover the
     end-to-end semantic path without changing controller tests.

3. Update the keyboard table in `README.md`:
   - Rename the `Escape` row to `Escape / Ctrl-[` (using the README's existing `Ctrl-`
     spelling) while preserving both descriptions: close and retain in the editor, close
     completion when completion is visible.
   - Leave the distinct `Control-C` discard row intact so retained close and destructive
     discard remain unmistakable.

## Validation

Before running toolchain checks, inspect the final diff and verify that only the router,
its tests, and the README changed. Run:

```sh
git diff --check
just format-lint
just build
just test
```

The Swift package imports AppKit and SwiftUI and requires macOS 26 with Apple's
toolchain. If the implementation host cannot execute those commands, record the exact
toolchain limitation and use the repository's `macos-26` GitHub Actions workflow as the
automated lint/build/test/bundle/launch gate; do not describe unrun Swift checks as
passing.

On macOS, smoke test a bundled installation (`just bundle && just install`, followed by
`Bob -> Restart Bob Mac Capture`):

1. Open the panel, type a nonempty draft, press Control-[: the panel closes. Reopen it
   and confirm the draft is unchanged and marked `Draft retained`.
2. Trigger a completion, press Control-[: only the completion closes. Press Control-[
   again: the panel closes. Reopen and confirm the draft remains.
3. Type a bare `[` and exercise Shift-[, Command-[, Option-[, Control-Shift-[, and
   Control-Command-[: none should invoke the new close alias; normal editor/system
   behavior should remain available.
4. Confirm physical Escape still follows the same completion-first then retained-close
   behavior.
5. Confirm Control-C still discards and closes, demonstrating that Control-[ is a
   non-destructive Escape alias rather than an abort alias.

## Handoff

Keep the change within `bob-mac-capture`; no coordination commit in `bob-cli` is
expected. If the current branch has moved beyond the planning-time commit, re-read the
router, controller Escape handling, router tests, and README table before editing and
preserve equivalent behavior rather than relying on stale line numbers.
