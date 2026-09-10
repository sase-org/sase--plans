---
tier: tale
title: Close the Bob Mac Capture panel after a successful capture
goal:
  Pressing Return (or Command-Return) in the capture panel hides the panel as soon as
  the capture succeeds, while a failed capture keeps the panel, its draft, and its error
  on screen, and the next hotkey press always opens a clean panel.
size: small
proposed_by: bbugyi200.athena.00v
create_time: 2026-09-09 19:59:51
status: wip
---

# Close the Bob Mac Capture panel after a successful capture

All work in this plan happens in the `bob-mac-capture` linked repository. Open it with
the `/sase_repo` skill first (`sase repo open bob-mac-capture -r "<reason>"`) and use
the path it prints as the only path for reads and writes. No `bob-cli` source change is
required: the capture CLI contract is unchanged.

## Objective

Make the capture pop-up disappear once the user submits a capture, so re-capturing means
pressing the global hotkey again rather than dismissing a panel that has already done
its job. Preserve every existing failure affordance: a capture that fails must still
leave the panel open with the complete draft and an actionable error.

## Diagnosed root cause

Nothing in the submit path ever hides the panel.

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` correctly maps Return to
  `.submit` and Command-Return to `.submitAndOpen`.
- `Sources/BobMacCapture/CapturePanelController.swift` handles those two commands by
  calling `model.submit(openAfterCapture:)` and swallowing the event. It orders the
  panel out (`panel?.orderOut(nil)`) only from the `.escape` branch.
- `Sources/BobMacCapture/CapturePanelModel.swift` finishes a capture in
  `completeSubmit(requestID:response:openAfterCapture:)`. On success it clears the
  draft, records `lastSuccess`, sets `statusText`, bumps `successAnnouncementTick`,
  posts the success notification, and optionally opens the target in Obsidian — but it
  has no way to reach the window, so the panel simply stays on screen showing the
  emptied editor and a "Captured → …" summary line.

The model is deliberately window-agnostic (it is unit-tested without a window), so the
fix is a controller-supplied dismissal hook invoked from the success branch only,
mirroring the existing `targetOpener: (URL) -> Void` injection point.

A second, consequential gap: `lastSuccess` is never cleared, and `destinationSummary` is
derived from it. Today that stale "Captured → …" line is harmless because the user
watches it appear in the panel they are still looking at. Once the panel closes on
success, that same line becomes the first thing rendered on the _next_ hotkey press,
describing a capture that already completed. Reopening must therefore start clean.

## Implementation

1. `Sources/BobMacCapture/CapturePanelModel.swift` — add a dismissal hook alongside the
   existing injected `targetOpener`:

   ```swift
   var panelDismisser: () -> Void = {}
   ```

   Call it exactly once, as the last statement of the `.success` branch of
   `completeSubmit` (after `notificationService?.notifyCaptureSuccess(...)` and after
   the `openAfterCapture` `targetOpener(url)` call, so the Obsidian hand-off is already
   issued when the panel goes away). Do **not** call it from the `.failure` branch, from
   `failSubmit(requestID:error:)`, or from the early "Bob is not resolved" guard in
   `submit(openAfterCapture:)`: every failure path must keep the panel visible so the
   retained draft, the error text, and the Retry/Copy Diagnostic buttons stay reachable.

2. `Sources/BobMacCapture/CapturePanelModel.swift` — add a presentation reset used when
   the panel is (re)shown:

   ```swift
   func prepareForPresentation() { ... }
   ```

   When `hasDraft` is `false`, clear the leftovers of the previous session:
   `lastSuccess`, `previewResult`, `errorMessage`, `parseDiagnostics`,
   `completionResponse`, `selectedCompletionIndex`, `pendingDiscardConfirmation`,
   `previewState = .idle`, and `statusText = ""`. When `hasDraft` is `true`, return
   without touching anything: an Escape-retained or failure-retained draft must reopen
   exactly as the user left it, including its error message and diagnostics. Reuse the
   existing private draft/analysis reset helpers where they fit rather than duplicating
   assignments; do not clear `attributedDraft` itself in this method.

3. `Sources/BobMacCapture/CapturePanelController.swift` — own the window side:
   - Extract the existing `panel?.orderOut(nil)` into a single private `hidePanel()`
     method that emits `CaptureSignpost.event("panel-dismiss")` and then orders the
     panel out. Use it from the `.escape` branch that currently calls `orderOut`
     directly, so both dismissal routes share one implementation and the existing
     panel-ordering signpost story covers dismissal too (metadata only — no draft text,
     consistent with the repository's privacy contract).
   - Wire the model hook in `init(model:)`, after `super.init()`:
     `model.panelDismisser = { [weak self] in self?.hidePanel() }`. `[weak self]` is
     required: the controller holds the model strongly, so a strong capture would create
     a retain cycle.
   - Keep using `orderOut(nil)`. Do not switch to `close()` or `performClose(_:)`:
     `performClose(_:)` routes through `windowShouldClose(_:)` → `model.requestClose()`
     and would trigger the discard-confirmation path, and `orderOut` is what preserves
     the pre-warmed panel and its `NSHostingView` for the next hotkey press, which the
     README documents as a launch-latency property.
   - Call `model.prepareForPresentation()` at the top of `show()`, before
     `panel.center()` and `makeKeyAndOrderFront(nil)`, so no stale frame is ever
     rendered.

4. `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` — extend the existing
   fake-`bob` based suite (`Tests/Fixtures/fake-bob`, driven by `FAKE_BOB_STDOUT`,
   `FAKE_BOB_EXIT`, and `FAKE_BOB_RECORD_PATH`). Each test injects a counting
   `model.panelDismisser = { dismissCount += 1 }`:
   - A successful `submit(openAfterCapture: false)` dismisses exactly once and still
     clears the draft.
   - A successful `submit(openAfterCapture: true)` dismisses exactly once and still
     opens the Obsidian URL through `targetOpener`.
   - A `bob`-reported failure (`FAKE_BOB_STDOUT` with `"ok":false` plus
     `FAKE_BOB_EXIT=1`) does not dismiss and still retains the draft.
   - A transport failure (malformed `FAKE_BOB_STDOUT`) does not dismiss.
   - `submit` with no resolved process client does not dismiss.
   - `prepareForPresentation()` after a successful capture clears `lastSuccess` and
     `destinationSummary`.
   - `prepareForPresentation()` with a retained draft and an `errorMessage` preserves
     the draft, the error, and `pendingDiscardConfirmation`.

   Do **not** add an `NSPanel` visibility assertion for the controller wiring (for
   example ordering a real panel front and asserting `isVisible` flips). That depends on
   a window-server session and on building the SwiftUI hosting view, which is exactly
   the kind of flakiness the current suite avoids by testing
   `CapturePanelController.makePanel()` as a pure factory. The one-line wiring is
   covered by review plus the manual checks below.

5. `README.md` — document the new contract:
   - Keyboard table: Return is "Capture, then close the panel"; Command-Return is
     "Capture, open the target in Obsidian, then close the panel". The Escape row is
     unchanged.
   - Runtime Contract: add a bullet stating that a successful capture hides the panel,
     that a failed capture keeps the panel open with its draft and error, and that
     reopening after a success starts from a clean panel.
   - Troubleshooting: in the "Capture fails but the draft disappears" bullet, note that
     failures deliberately keep the panel on screen, so a panel that vanished means the
     capture landed (and the success notification names the route).

## Considered and rejected

- **Hiding the panel optimistically on the keypress instead of on success.** It would
  feel marginally faster, but it would hide the error and the retained draft behind a
  second hotkey press and would undercut the README's promise that a failed capture
  never loses the draft in front of the user. `bob` is a native binary, so waiting for
  the actual result costs a few tens of milliseconds, and the panel already shows
  "Capturing…" during that window.
- **A Settings toggle for auto-close.** Unconditional behavior is what was asked for; a
  preference adds a persisted key, a Settings row, and a second code path for no current
  benefit.
- **Publishing a `dismissRequestTick` counter observed by the controller through
  Combine.** Equivalent behavior with more indirection; the closure matches the
  `targetOpener` pattern already established in this model.

## Risks and accepted trade-offs

- The VoiceOver announcement driven by `successAnnouncementTick` (which focuses the
  status line in `CapturePanelView`) is cut short when the panel goes away. No
  accessibility code is removed, and the success `UNUserNotificationCenter` banner —
  which names the route — remains the announced confirmation. Accepted.
- Pressing Escape twice during an in-flight submit hides the panel before the result
  arrives; the success path then calls `orderOut` on an already-hidden panel, which is a
  harmless no-op.
- Reopening the panel after `orderOut` becomes the common path rather than a rare one,
  so keyboard focus landing back in the editor on the second and third open must be
  checked manually (step 3 of Validation).

## Validation

1. On the macOS 26 host with the repository's Apple toolchain selected, run
   `just format-lint`, `just build`, and `just test` from the `bob-mac-capture`
   checkout; all three must pass, including the new `CapturePanelModelTests` cases.
2. Build and install the bundle (`just bundle` then `just install`) so notification
   delivery and the hotkey are exercised against the signed bundle rather than a raw
   `.build` binary.
3. Manual checks with the installed app:
   - Hotkey → type a valid capture → Return: the panel disappears, the "Captured"
     notification names the route, and the note receives the task.
   - Press the hotkey again: the panel reopens empty, with no "Captured → …" summary and
     no stale status text, and typing goes straight into the editor without clicking.
   - Hotkey → type a capture → Command-Return: the panel disappears and Obsidian opens
     the target note.
   - Force a failure (for example a route that does not resolve): the panel stays open,
     the draft is intact, the error and the Retry/Copy Diagnostic buttons are shown, and
     Retry after fixing the draft then closes the panel.
   - Escape with a nonempty draft still shows "Draft retained" and keeps the panel; a
     second Escape hides it; the next hotkey press restores that draft unchanged.
4. If the implementing agent has no macOS 26 host available, it must say so explicitly
   and report steps 1–3 as not run rather than as passed: the `BobMacCapture` target
   depends on AppKit, SwiftUI, and the macOS 26 SDK and cannot be built or tested on the
   Linux development host, where only `CaptureCore` compiles.
