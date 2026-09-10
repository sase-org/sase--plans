---
tier: tale
title: Restore Bob Mac Capture launch after the AppKit entry-point migration
goal:
  The installed Bob Mac Capture app launches, shows its "Bob" menu-bar item, registers
  its hotkey, and macOS CI proves the app actually launches on every future change.
size: medium
proposed_by: bbugyi200.athena.00o.f1
create_time: 2026-09-09 20:00:25
status: wip
---

# Restore Bob Mac Capture launch after the AppKit entry-point migration

## Symptom

The installed app no longer comes up: no `Bob` item appears in the macOS menu bar, the
capture hotkey does nothing, and Settings cannot be opened. There is no crash dialog.

## Root cause (confirmed)

Commit `a3e620b` ("fix: present settings through AppKit-hosted scene") replaced the
app's entry point. It deleted `Sources/BobMacCapture/BobMacCaptureApp.swift`, which
contained:

```swift
@main
struct BobMacCaptureApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    ...
}
```

and moved `@main` onto `AppDelegate` itself. Under the previous SwiftUI app lifecycle,
`@NSApplicationDelegateAdaptor` was what instantiated `AppDelegate`, retained it, and
assigned it to `NSApplication.delegate`.

`@main` on an `NSObject` class that conforms to `NSApplicationDelegate` resolves to
AppKit's default `main()`, which calls `NSApplicationMain`. `NSApplicationMain` does not
construct the delegate itself — it obtains the delegate from the main storyboard/nib
named by `NSMainStoryboardFile` or `NSMainNibFile`. `Resources/Info.plist` declares
neither; its only class key is `NSPrincipalClass = NSApplication`. With no nib to load,
nothing ever instantiates `AppDelegate`, so `NSApp.delegate` stays `nil`.

Consequently `applicationWillFinishLaunching(_:)` and
`applicationDidFinishLaunching(_:)` are never called, so `configureStatusItem()` never
runs (no `Bob` item), the hotkey is never registered, the capture panel is never
prewarmed, and the settings scene is never added. The process launches and runs an empty
AppKit event loop, which is why the failure is silent and produces no crash report.

Supporting evidence:

- `Resources/Info.plist` contains no `NSMainNibFile` / `NSMainStoryboardFile` key, and
  the repository contains no `.xib`, `.storyboard`, or nib resource.
- `grep -rn "mainMenu\|MenuBarExtra\|NSMenu(" Sources Tests README.md` matches only the
  status-item `NSMenu()` in `AppDelegate.configureStatusItem()`. Nothing constructs an
  application entry point, a nib, or a main menu.
- macOS CI is green on master tip `f59ab74` (run `31799467198`): formatting,
  `swift build`, `swift test`, bundle, plist lint, codesign verification, and
  install/reinstall all pass. No CI step ever launches the app, which is exactly why a
  launch-breaking change shipped green.

## What is not the cause

Do not re-investigate these; they were checked and cleared:

- **The Hammerspoon cutover (`chezmoi` commit `95369559`)**: it removed only the retired
  capture feature. The Pomodoro menu-bar runtime and screenshot bindings remain intact
  in `home/dot_hammerspoon/init.lua`, and `chezmoi` contains no Bob Mac Capture launch,
  LaunchAgent, or login-item wiring at all (the app is mentioned only in `README.md`).
  Hammerspoon never launched the app.
- **`df7df60` (production hotkey default)**: `AppSettings.init` only reads
  `UserDefaults`, and `registerHotKey()` runs _after_ `configureStatusItem()`; a
  registration failure is caught and reported through `diagnosticStatus` rather than
  aborting launch.
- **`f59ab74` (lazy `notificationService`)**: it only moves `NotificationService()`
  construction into `applicationWillFinishLaunching`, a method that never runs in the
  broken build.

## Second defect from the same migration

The same entry-point change also left `NSApp.mainMenu` `nil`, because neither a nib nor
the SwiftUI app lifecycle supplies one any more. That silently removes the standard Edit
key equivalents (Command-X/C/V/A/Z) that the capture panel's text editor relies on, and
removes Command-Q. Fix it here: it is caused by the same commit, it is invisible until a
user tries to paste into a capture, and it cannot be observed until the app launches
again.

## Implementation

Work in `bobs-org/bob-mac-capture`, opened with
`sase repo open bob-mac-capture -r "<specific reason>"`.

1. **Add an explicit entry point that assigns the delegate.** Create
   `Sources/BobMacCapture/BobMacCaptureMain.swift`:
   - `@main enum BobMacCaptureMain` with a `@MainActor static func main()`.
   - Inside `main()`: reference `NSApplication.shared` **first**, then construct
     `AppDelegate()`, assign it to `NSApplication.shared.delegate`, and then call
     `NSApplication.shared.run()`.
   - `NSApplication.delegate` is a weak reference, so the instance must also be held by
     a strong static stored property (or an equivalent explicit lifetime extension) for
     the life of the process. Do not rely on the local binding alone.
   - Use `run()` rather than `NSApplicationMain(...)`, so the delegate assigned above is
     the one that receives `applicationWillFinishLaunching` and
     `applicationDidFinishLaunching`. `run()` calls `finishLaunching()`, which posts
     both.
   - Remove the `@main` attribute from `AppDelegate` and leave the rest of the class as
     is.

   Do **not** introduce a top-level-code `Sources/BobMacCapture/main.swift`. The
   `BobMacCaptureTests` target depends on the `BobMacCapture` executable target, and an
   `@main` enum keeps `AppDelegate` an ordinary constructible class, which
   `testOpenSettingsMenuActionUsesRegisteredSettingsPresenterOnce` requires. If SwiftPM
   nonetheless rejects the `@main` enum in an executable target that a test target
   imports, the fallback is a `main.swift` performing the same three steps; record which
   form was used and why.

2. **Install a main menu before registering the settings scene.** Add a testable pure
   builder (for example `static func makeMainMenu() -> NSMenu`) and assign
   `NSApp.mainMenu` at the top of `applicationWillFinishLaunching(_:)`, before
   `NSApplication.shared.addSceneRepresentation(representation)`.
   - Application menu: Hide, Hide Others, Show All, and Quit (`q`), using the standard
     `NSApplication` selectors.
   - Edit menu: Undo, Redo, Cut, Copy, Paste, and Select All, using the standard
     first-responder selectors (`undo:`, `redo:`, `cut:`, `copy:`, `paste:`,
     `selectAll:`) and their standard key equivalents, so the capture editor regains
     normal editing shortcuts.
   - Keep the app an `LSUIElement` accessory app; do not change the activation policy
     and do not add a Window or File menu.

3. **Make launch completion observable.** Emit
   `CaptureSignpost.event("launch-complete")` as the last statement of
   `applicationDidFinishLaunching(_:)`. This is metadata-only instrumentation consistent
   with the existing signposts and is what the new CI gate asserts on.

4. **Add a launch smoke gate to macOS CI.** In `.github/workflows/ci.yml`, after the
   existing bundle/verify steps, add a step that actually launches the bundled app on
   the `macos-26` runner and fails if the delegate never runs:
   - Launch the signed bundle through LaunchServices (`open`), poll until the process is
     running, and assert it is still alive after a short settle period.
   - Assert the app reached the end of `applicationDidFinishLaunching` by requiring the
     `launch-complete` signpost in
     `log show --predicate 'subsystem == "org.bobs.bob-mac-capture"'` for the run
     window.
   - Quit the app at the end of the step, and on failure upload any new
     `~/Library/Logs/DiagnosticReports/` entries plus the filtered log as an artifact,
     so a future launch failure is diagnosable from Linux.
   - Verify this gate is a real gate: confirm it fails against the current broken entry
     point (for example by temporarily reverting step 1 locally in the branch, observing
     the red run, then restoring the fix) and passes with the fix. A launch gate that
     cannot fail is worthless, and this is the one check that would have caught this
     bug.
   - If the runner's GUI session makes `open` or `log show` unreliable, fall back in
     this order and document the choice in the workflow: (a) execute
     `"<bundle>/Contents/MacOS/BobMacCapture"` directly and keep the log assertion; (b)
     have the app write a marker file at the end of `applicationDidFinishLaunching` only
     when an environment variable naming that path is set, and assert on the marker.
     Never weaken the gate to a mere "the binary exists" check.

5. **Tests.**
   - Add a `BobMacCaptureTests` case over `makeMainMenu()` asserting the Edit menu
     exposes the expected selectors and key equivalents and that a Quit item exists.
     This is the regression guard for step 2.
   - Keep every existing test passing, including the settings-presenter test that
     constructs `AppDelegate()` directly.

6. **README.** Add a short troubleshooting entry for "the `Bob` menu-bar item does not
   appear" that records the owner-side diagnostics (`pgrep -fl BobMacCapture`,
   `log show --last 5m --predicate 'process == "BobMacCapture"'`, and checking
   `~/Library/Logs/DiagnosticReports/`) and states the structural requirement this bug
   came from: the app has no nib, so the entry point must assign
   `NSApplication.shared.delegate` explicitly and must supply its own main menu.

## Verification

1. Review the complete diff against this plan and run `git diff --check`.
2. This workspace host is Linux with no Swift toolchain, AppKit, or `swift-format`. Do
   not claim local formatting, build, test, bundle, or codesign results; the handoff
   must separate Linux-side diff checks from macOS runner results.
3. Require a green `macOS 26 SwiftPM` run that includes the new launch gate, and confirm
   from that run's log that the `launch-complete` assertion actually executed rather
   than being skipped.
4. Owner-assisted checks on the target Mac after reinstalling with `just install`:
   - The `Bob` item appears in the menu bar after launch and after logout/login.
   - Control-Shift-Command-I opens the capture panel.
   - The menu-bar Settings item opens the Settings window.
   - Command-A, Command-C, and Command-V work inside the capture editor, and Command-Q
     quits the app.
5. If the menu-bar item is still missing after the fix, collect the diagnostics from the
   README entry added in step 6 before making further changes; the app now runs its
   delegate, so any remaining failure will produce a real log or crash report rather
   than silence.

## Out of scope

- Reverting `a3e620b`'s settings-scene approach. `NSHostingSceneRepresentation` plus
  `addSceneRepresentation` in `applicationWillFinishLaunching` is the documented macOS
  26 pattern and correctly fixes opening Settings; only the entry point was wrong.
- Migrating the status item to SwiftUI `MenuBarExtra`.
- Any `chezmoi` or Hammerspoon change. The cutover was verified unrelated to this
  failure.
- The remaining owner-assisted Mac smoke tests already recorded as a proposed follow-up
  on bead `bob-cli-j.7`; this plan only adds the launch checks above.

## Rollback

If the new entry point regresses, the fallback is restoring the previous SwiftUI entry
point (`@main struct BobMacCaptureApp: App` with
`@NSApplicationDelegateAdaptor(AppDelegate.self)` and the `Settings` scene in its body),
which is known to launch correctly but reintroduces the macOS 26 Settings-window bug
that `a3e620b` fixed. Prefer fixing forward.
