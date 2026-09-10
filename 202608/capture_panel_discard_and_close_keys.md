---
tier: tale
title: Add Control-C discard and single-press Escape to the capture panel
goal:
  The Bob Mac Capture pop-up closes on a single Escape press while retaining the typed
  draft, and Control-C discards the draft and closes the panel in one press.
size: medium
proposed_by: bbugyi200.athena.00w.f0.f0.w0.w0.w0
create_time: 2026-09-09 19:59:53
status: wip
---

# Add Control-C discard and single-press Escape to the capture panel

## Context

Implement this work in the `bobs-org/bob-mac-capture` repository. Open it first:

```sh
sase repo open bob-mac-capture -r "Add Control-C discard and single-press Escape to the capture panel"
```

Use the printed path as the only path for reads and writes. Base the work on the latest
`origin/master` (`a20055e fix(capture): keep panel actions visible while autosizing` at
planning time). No `bob-cli` change is required: this is presentation and key routing
only, and it does not touch the versioned `bob capture` / `capture-parse` /
`capture-complete` subprocess or JSON contracts.

Three files own the current keyboard behavior:

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` maps an `NSEvent` plus a
  `completionVisible` flag to a `CaptureKeyCommand`.
- `Sources/BobMacCapture/CapturePanelController.swift` installs a local `.keyDown`
  monitor (`installKeyMonitorIfNeeded`, lines 231-284), routes each event through the
  router, and switches on the resulting command inside the monitor closure.
- `Sources/BobMacCapture/CapturePanelModel.swift` owns draft state, `discardDraft()`
  (line 241), `requestClose()` (line 231), and the `panelDismisser` closure the
  controller wires to `hidePanel()`.

### What Escape does today

`CapturePanelController.swift:262-272` handles `.escape` in three steps:

1. If a completion list is visible, dismiss the completion and consume the event.
2. Else if `pendingDiscardConfirmation` is already set, or the draft is empty, hide the
   panel.
3. Else call `model.requestClose()`, which sets `pendingDiscardConfirmation = true`,
   sets `statusText = "Draft retained"`, returns `false`, and leaves the panel open.

So with a nonempty draft the panel needs Escape twice: the first press only arms the
confirmation, which also reveals a **Discard** / **Keep Draft** button pair in the
footer (`CapturePanelView.swift:325-333`). That two-step gate is the sole purpose of
`pendingDiscardConfirmation`.

`README.md:131` already documents Escape as "Close completion, then close the panel
(retaining a nonempty draft)" — one press — so the implementation is what disagrees with
the documented contract, not the other way around.

### What is missing today

There is no key that discards a draft. `discardDraft()` is reachable only through the
footer **Discard** button, which itself only appears after Escape arms the confirmation.

## Goal and acceptance criteria

- Pressing Control-C (control as the only modifier) while the capture panel is key
  discards the current draft and closes the panel in one press, from every panel state:
  with or without a visible completion list, with or without an error showing, and with
  an empty draft (where it simply closes).
- Reopening the panel after Control-C starts from a clean slate: empty editor, no
  completion list, no live preview, no error, no destination summary, empty status, and
  a fresh `p:<N>` priority-roll seed.
- Pressing Escape once with a nonempty draft closes the panel and retains the draft.
  Reopening restores the draft — and any capture error attached to it — exactly as it
  was left, with `statusText` reading `Draft retained`.
- While a completion list is visible, Escape still dismisses only the completion and
  leaves the panel open; a second Escape then closes the panel.
- Escape with an empty draft closes the panel, as it does today.
- Command-C still copies the editor selection, and Control-Shift-C, Control-Command-C,
  and a bare `c` keystroke are unaffected — none of them discards a draft.
- Every other capture key keeps its behavior: Return, Command-Return,
  Shift/Option-Return, Control-J, Tab, Up/Down, and Control-N/Control-P.
- Closing the panel through its window close button also retains the draft instead of
  arming a confirmation.
- `pendingDiscardConfirmation` and the **Keep Draft** button are gone; the footer keeps
  a persistent **Discard** action with the same effect as Control-C.
- `README.md` documents both keys, and its footer-button prose no longer mentions **Keep
  Draft**.

## Design decisions

1. **Control-C is unconditional.** It is the abort key: it discards and closes even when
   a completion list is visible, rather than being absorbed by the completion the way
   Escape is. Escape remains the layered "back out one level" key. Consequence to
   accept: if Control-C is pressed during the brief window while a capture subprocess is
   still in flight, the draft is discarded anyway, so a capture that then fails surfaces
   its error with an empty editor. This is the literal requested behavior
   ("auto-discard"), the window is a few hundred milliseconds after Return, and no guard
   is added for it.
2. **Retire `pendingDiscardConfirmation` instead of leaving it stranded.** Once one
   Escape closes and retains, the confirmation state is unreachable from the keyboard
   and its **Keep Draft** button (bound to `.keyboardShortcut(.cancelAction)`, i.e.
   Escape, which the local key monitor already swallows) is dead UI. Closing the panel
   is now never destructive and needs no confirmation; discarding is always explicit.
3. **Keep a pointer-reachable discard.** Removing the confirmation pair would leave
   Control-C as the only way to clear a draft, so the footer gains a persistent
   **Discard** button that calls exactly the same model entry point. If the reviewer
   prefers a keyboard-only discard, dropping that button is a one-line change to
   `CapturePanelFooter`; nothing else in the plan depends on it.
4. **Strict modifier matching for Control-C**, `modifiers == .control`, mirroring the
   existing Control-J rule at `CaptureKeyCommandRouter.swift:50`. This keeps
   Command-Control-C and Shift-Control-C from destroying a draft by accident. It
   inherits the same known quirk as Control-J: with Caps Lock on, `.capsLock` is present
   in `deviceIndependentFlagsMask` and the shortcut does not match. Normalizing that
   across all shortcuts is deliberately out of scope here.
5. **Dismissal moves into the model.** `CapturePanelModel` already owns `panelDismisser`
   and calls it on a successful capture, and the existing model tests assert dismissal
   by counting calls to an injected closure. Both new actions follow that pattern
   instead of calling `hidePanel()` from the monitor closure, which no test can reach.

## Implementation

1. **Route the new key** in `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`:
   - Add `case discardAndClose` to `CaptureKeyCommand`.
   - Add `static let c: UInt16 = 8` to the private `KeyCode` enum.
   - Add a `case KeyCode.c:` arm returning `.discardAndClose` when
     `modifiers == .control`, and `nil` otherwise. Place it so the result does not
     depend on `completionVisible`.

2. **Replace the confirmation state machine** in
   `Sources/BobMacCapture/CapturePanelModel.swift`:
   - Delete the `pendingDiscardConfirmation` published property and its three
     assignments in `editorTextDidChange` (line 134), `resetAnalysisState` (line 263),
     and `completeSubmit` (line 358).
   - Replace `requestClose()` with a non-refusing close note, and add the two actions
     the key commands call:

     ```swift
     /// Records a close that keeps the draft. Closing the panel is never destructive;
     /// discarding is an explicit action (Control-C or the Discard button).
     func prepareForRetainedClose() {
         if hasDraft {
             statusText = "Draft retained"
         }
     }

     func closeRetainingDraft() {
         prepareForRetainedClose()
         panelDismisser()
     }

     func discardDraftAndClose() {
         discardDraft()
         panelDismisser()
     }
     ```

   - In `discardDraft()`, replace the bare `analysisTask?.cancel()` with the existing
     private `invalidateAnalysis()`. Control-C makes discard a common path, and bumping
     `analysisGeneration` is what stops an in-flight debounced parse/preview from
     writing `previewState` or `statusText` back onto a draft the user just discarded.
   - Leave `prepareForPresentation()` alone: its `guard !hasDraft` early return is
     exactly what makes an Escape-retained draft reopen untouched, and its reset branch
     is what gives a Control-C discard its clean slate.

3. **Handle the commands** in `Sources/BobMacCapture/CapturePanelController.swift`:
   - Extract the monitor's `switch` into a testable method, keeping every existing
     case's behavior byte for byte:

     ```swift
     /// Applies a routed key command. Returns `true` when the key event is consumed.
     @discardableResult
     func perform(_ command: CaptureKeyCommand) -> Bool
     ```

     `.insertNewline` returns `false` when the first responder is not an editable text
     view (so the event still propagates); every other case returns `true`.

   - Reduce the monitor closure body to routing plus
     `return self.perform(command) ? nil : event`.
   - Change `.escape` to: dismiss the completion and consume the event when
     `model.completionVisible`; otherwise call `model.closeRetainingDraft()`. No
     `pendingDiscardConfirmation` check, no second press.
   - Add `.discardAndClose` calling `model.discardDraftAndClose()`.
   - Change `windowShouldClose(_:)` to call `model.prepareForRetainedClose()` and return
     `true`.

4. **Update the footer** in `Sources/BobMacCapture/CapturePanelView.swift`
   (`CapturePanelFooter`, lines 325-333): delete the
   `if model.pendingDiscardConfirmation` block with its **Discard** / **Keep Draft**
   pair and its `.cancelAction` shortcut, and add a persistent **Discard** button before
   **Preview** that calls `model.discardDraftAndClose()`, is
   `.disabled(!model.hasDraft || model.isSubmitting)`, and carries
   `.help("Discards the draft and closes the panel (Control-C).")`.

5. **Document the keys** in `README.md`:
   - Keyboard table (line 123-131): change the Escape row's editor column to make the
     single press explicit — closes the panel, retaining a nonempty draft, with no
     confirmation step — and keep its completion column as "Close completion". Add a
     `Control-C` row reading "Discard the draft and close the panel" in both columns.
   - Line 133-134: replace "Capture/Preview/Discard/Keep Draft buttons" with
     "Capture/Preview/Discard buttons".
   - Line 115-119 ("Runtime Contract"): keep the retained-draft sentence accurate and
     note that Control-C or **Discard** is what clears a draft, so a close never
     destroys one.

## Regression coverage

Add to `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, using the existing private
`keyEvent(keyCode:modifiers:)` helper:

- Extend `testKeyRouterMatchesCaptureShortcuts` (or add a sibling test) asserting
  `keyCode: 8` with `.control` maps to `.discardAndClose` both with
  `completionVisible: true` and `false`, and that `keyCode: 8` with no modifier, with
  `.command`, with `[.control, .shift]`, and with `[.control, .command]` all map to
  `nil`.
- A `@MainActor` controller test: with a nonempty draft and an injected
  `model.panelDismisser` counter (assigned after the controller is constructed, so it
  replaces the controller's own dismisser), `controller.perform(.escape)` dismisses
  exactly once on the first call, leaves `plainDraft` unchanged, and sets `statusText`
  to `Draft retained`.
- A `@MainActor` controller test: with a completion response set on the model,
  `controller.perform(.escape)` dismisses zero times and clears `completionResponse`; a
  second `perform(.escape)` then dismisses exactly once.
- A `@MainActor` controller test: `controller.perform(.discardAndClose)` with a nonempty
  draft and a visible completion clears `plainDraft` and `completionResponse` and
  dismisses exactly once.

Add to `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`:

- `discardDraftAndClose()` clears the draft, `errorMessage`, `previewResult`,
  `parseDiagnostics`, and `previewState`, and calls `panelDismisser` exactly once.
- `closeRetainingDraft()` with an empty draft dismisses once and does not set
  `Draft retained`.
- Update the two existing tests that reference the removed property: drop the
  `XCTAssertFalse(model.pendingDiscardConfirmation)` assertion at line 98, and rename
  `testPrepareForPresentationWithRetainedDraftAndErrorPreservesDraftErrorAndDiscardConfirmation`
  (line 155) to drop its confirmation clause and its two confirmation statements while
  keeping the draft and error assertions.

## Validation

`swift build` and `swift test` for this package require macOS 26 with Apple's toolchain;
the sources import AppKit and SwiftUI, so they cannot be compiled on the Linux SASE
host. Do not report the change as verified from a Linux workspace.

On macOS, from the repository root:

```sh
just format-lint
just build
just test
```

Otherwise, push the branch and let the `macos-26` GitHub Actions workflow
(`.github/workflows/ci.yml`) run lint, build, test, bundle, and the launch smoke test,
and treat a green run as the automated gate.

Then smoke test the real panel from a bundled build (`just bundle && just install`, then
`Bob -> Restart Bob Mac Capture`):

1. Open the panel, type `test draft @Cash`, press Escape once: the panel closes
   immediately.
2. Press the capture hotkey again: the draft is back verbatim and the footer reads
   `Draft retained`.
3. Type `idea @ma` and wait for the completion list, press Escape: only the completion
   closes. Press Escape again: the panel closes with the draft retained.
4. Reopen, then press Control-C: the panel closes. Reopen again: the editor is empty
   with no completion, preview, error, destination summary, or leftover status.
5. Type a draft, wait for the completion list, press Control-C: the panel closes and the
   draft is gone on reopen.
6. Select text in the editor and press Command-C, then paste elsewhere: the copy works
   and the draft is untouched. Repeat with Control-Shift-C and confirm nothing happens.
7. Confirm Return captures, Command-Return captures and opens, Control-J and
   Shift-Return insert newlines, and Tab/arrows/Control-N/Control-P still drive the
   completion list.
8. With a nonempty draft, click the panel's window close button and reopen: the draft is
   retained.
9. Click the footer **Discard** button with a draft: the panel closes and the draft is
   gone on reopen.

If no interactive macOS session is available, record the manual smoke test as the one
outstanding validation rather than treating a green CI run as proof of panel behavior.

## Out of scope

- Any `bob-cli` change; no capture grammar, completion, preview, or JSON contract moves.
- Cancelling an in-flight `bob` subprocess when a draft is discarded
  (`BobProcessClient.cancelActiveProcess()` stays a quit-path concern).
- Normalizing `.capsLock` out of modifier comparisons for the existing shortcuts.
- Making the panel's global hotkey, the **Preview** action, or the status-menu items
  keyboard-configurable.
