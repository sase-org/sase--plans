---
tier: tale
title: Fix Bob Mac Capture Settings presentation
goal:
  Bob Mac Capture reliably opens and reuses its Settings window from the Bob status menu
  through the supported macOS 26 SwiftUI scene API.
size: medium
proposed_by: bbugyi200.athena.00m
create_time: 2026-09-09 20:00:05
status: wip
---

# Fix Bob Mac Capture Settings presentation

## Context and diagnosis

Implement this work in the `bobs-org/bob-mac-capture` repository, opened through
`sase repo open gh:bobs-org/bob-mac-capture`. Begin from the latest `origin/master` and
retain the existing capture-panel cancellation fix at `ef64f0a`.

Bob Mac Capture is an `LSUIElement` menu-bar app. Its SwiftUI `App` declaration owns a
`Settings` scene, while `AppDelegate` builds the visible status item with AppKit. The
status menu's Settings item currently calls:

```swift
NSApp.sendAction(Selector(("showSettingsWindow:")), to: nil, from: nil)
```

This is the root cause of the silent failure. `showSettingsWindow:` is a string-built,
unsupported responder-chain convention rather than a public AppKit or SwiftUI settings
presentation API. `sendAction` returns `false` when no responder handles the selector,
but the app discards that result, so clicking Settings does nothing and reports no
error. The latest upstream commit does not change this path.

The app targets macOS 26, where Apple's supported bridge for an AppKit-owned menu to a
SwiftUI scene is `NSHostingSceneRepresentation`: register the retained representation
with `NSApplication`, then invoke `settingsScene.environment.openSettings()`.

## Goal and acceptance criteria

- Clicking Bob -> Settings opens the app's existing settings UI and brings it to the
  foreground from a cold menu-bar-only launch.
- Clicking Settings while the window already exists brings that same window forward; it
  does not create duplicate settings windows.
- Closing the settings window and selecting Settings again reopens it.
- The settings view continues to use the one shared `AppSettings` and
  `NotificationService` instances owned by the application delegate.
- The app remains an accessory `LSUIElement` app, and Capture, Recheck Bob, Quit, the
  global hotkey, and the dynamic warning label/tooltip continue to behave as before.
- No stringly typed settings selector or ignored responder-chain result remains.

## Implementation

1. Make the existing AppKit delegate the application entry point and explicit owner of
   the SwiftUI settings scene. Retire the thin `BobMacCaptureApp` SwiftUI lifecycle
   wrapper so there is exactly one entry point and exactly one Settings scene.
2. Add a retained `NSHostingSceneRepresentation` containing `SettingsView`, wired to the
   delegate's existing `settings` and `notificationService` objects. Register it with
   `NSApplication.addSceneRepresentation` during `applicationWillFinishLaunching`,
   before the status menu can request presentation.
3. Capture the representation's supported `environment.openSettings()` action behind a
   small injectable/internal presentation seam. Change the Settings menu action to
   invoke that action and activate the accessory app so the settings window becomes
   key/frontmost. Keep the representation alive for the full app lifetime and use a weak
   capture where needed to avoid a delegate/action retain cycle.
4. Preserve all existing launch work in `applicationDidFinishLaunching` and all teardown
   work in `applicationWillTerminate`; the lifecycle migration must not reorder
   process-client setup, panel prewarming, hotkey registration, target watching, or
   cancellation.

## Regression coverage

- Add a focused `@MainActor` test using the presentation seam to prove that invoking the
  delegate's Settings menu action calls the registered settings-opening action exactly
  once. This prevents a regression back to an unhandled responder selector without
  requiring a GUI session in the unit test.
- Keep the existing Info.plist test proving `LSUIElement` and bundle identity intact,
  and run the complete existing test suite to cover the unchanged capture/status
  behavior.

## Validation

Run the repository's macOS 26 checks against the latest upstream base:

```sh
just format-lint
just build
just test
just bundle
plutil -lint Resources/Info.plist
codesign --verify --deep --strict ".build/bundle/Bob Mac Capture.app"
```

Then perform a real macOS 26 smoke test using the bundled or installed app:

1. Quit any prior Bob Mac Capture process and launch the new bundle.
2. Open Bob -> Settings and verify the window is visible and frontmost.
3. Select Settings again while it is open and verify the existing window comes forward
   without a duplicate; close it and verify a subsequent selection reopens it.
4. Verify the displayed values still update from the delegate-owned settings and
   notification service.
5. Verify Capture, Recheck Bob, Quit, the global hotkey, and the warning status-item
   appearance still work, and confirm the app still has no Dock icon.

If an interactive GUI session is unavailable to the implementing agent, record the
manual smoke test as the only outstanding validation rather than treating compilation or
the unit seam as proof that window ordering works.
