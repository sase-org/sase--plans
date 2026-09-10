---
tier: tale
title: Add a Restart action to the Bob Mac Capture menu-bar menu
goal:
  The `Bob` menu-bar menu offers "Restart Bob Mac Capture", which quits the running
  process and relaunches the installed app bundle so a freshly installed build takes
  over, and which refuses to terminate at all when a relaunch could not succeed.
size: medium
proposed_by: bbugyi200.athena.00x
create_time: 2026-09-09 19:59:47
status: wip
---

# Add a Restart action to the Bob Mac Capture menu-bar menu

All work in this plan happens in the `bob-mac-capture` linked repository. Open it with
the `/sase_repo` skill first (`sase repo open bob-mac-capture -r "<reason>"`) and use
the path it prints as the only path for reads and writes. No `bob-cli` source change is
required: the capture CLI contract is untouched.

## Objective

Add a `Restart Bob Mac Capture` item to the `NSStatusItem` menu built in
`Sources/BobMacCapture/AppDelegate.swift`. Choosing it must terminate the running
process and bring the app back from its installed bundle, so that `just bundle` +
`just install` can be completed from the menu bar instead of by quitting and hunting for
the app in Finder.

The load-bearing safety property: the app must never terminate unless the relaunch is
credible. A Restart that quits an `LSUIElement` app without bringing it back leaves the
user with no menu-bar item, no global hotkey, and no obvious way to notice what
happened.

## Design constraints

These are the reasons the obvious implementations are wrong; the implementation below
follows from them.

1. **Quit-then-launch, not launch-then-quit.** If a new instance starts while the old
   one is alive, the incoming instance calls `RegisterEventHotKey` while the outgoing
   one still owns Control-Shift-Command-I. `HotKeyManager` reports
   `eventHotKeyExistsErr`, so the restarted app comes up with a dead hotkey and Settings
   shows "Hotkey conflict" — the exact failure documented in README's "Hotkey Conflicts"
   section. Two `Bob` status items would also appear.
2. **The relaunch must wait for the old PID to exit.** LaunchServices deduplicates by
   bundle identifier, so a plain `/usr/bin/open <path>` issued while the old instance is
   still running merely activates it and no restart happens. `open -n` would force a
   second instance and reintroduce constraint 1. Waiting for exit is therefore
   mandatory, not a refinement.
3. **The waiter cannot live inside the app,** because the process doing the waiting is
   the process that is exiting. It must be a detached child that outlives its parent; a
   child spawned with `Process` survives its parent's exit and is reparented to
   `launchd`. Async AppKit APIs such as
   `NSWorkspace.shared.openApplication(at:configuration:)` are not usable here: their
   completion handlers may never run in a process that is calling `NSApp.terminate` in
   the same turn.
4. **Use `Bundle.main.bundleURL`, captured at launch.** It is derived from the main
   executable path recorded when the process started and does not follow file-system
   moves. `Scripts/install.sh` swaps the new bundle into the _same_ install path
   (`mv install → backup`, then `mv staged → install`), so the launch-time path already
   names the newly installed build — this is precisely what makes Restart the
   update-completion step. `NSRunningApplication.current.bundleURL` must not be used: it
   can follow the old bundle to the backup path the installer then deletes.
5. **Terminate through `NSApp.terminate(nil)`.** That is what runs the existing
   `applicationWillTerminate`, which calls `hotKeyManager?.invalidate()`,
   `vaultWatcher?.invalidate()`, and `processClient?.cancelActiveProcess()`. Combined
   with the wait-for-exit helper, this guarantees the Carbon hotkey is released before
   the new instance tries to register it.
6. **Refuse rather than quit when relaunch cannot work.** Two detectable cases: the
   process is not running from an `.app` bundle at all (`swift run`, a raw `.build`
   binary — `Bundle.main.bundleURL.pathExtension != "app"`), or the bundle no longer
   exists on disk. In both, stay running and report; do not terminate.
7. **No entitlement work is needed.** `Scripts/bundle.sh` signs with `--options runtime`
   and no entitlements file, so the app uses the hardened runtime without App Sandbox
   and may spawn `/bin/sh`. Adding App Sandbox would break this design.
8. **Restart discards an unsent draft, exactly as Quit already does.** No confirmation
   dialog: drafts are deliberately never persisted, an accessory app would have to
   activate itself to show an alert, and the existing Quit item sets the precedent for
   this trade-off.

## Implementation

### 1. `Sources/BobMacCapture/AppRelauncher.swift` (new)

Add the relaunch logic to the `BobMacCapture` target as an injectable, unit-testable
`@MainActor struct AppRelauncher`:

```swift
var bundleURL: URL                          // default: Bundle.main.bundleURL
var processIdentifier: Int32                // default: ProcessInfo.processInfo.processIdentifier
var fileExists: (String) -> Bool            // default: FileManager.default.fileExists(atPath:)
var spawn: (String, [String]) throws -> Void
var terminate: () -> Void                   // default: { NSApp.terminate(nil) }
```

The default `spawn` builds a `Process`, sets `executableURL`/`arguments`, points
`standardOutput` and `standardError` at `FileHandle.nullDevice`, and calls `try run()`.
It must **not** call `waitUntilExit()` — the child has to outlive this process.

Add `enum AppRelaunchError: Error, Equatable` with `notBundled(path: String)`,
`bundleMissing(path: String)`, and `spawnFailed(String)`, each carrying a short
human-readable message suitable for Diagnostics and a notification body.

Expose the argv construction as a pure static function so it can be asserted directly:

```swift
static func relaunchArguments(processIdentifier: Int32, bundlePath: String) -> [String]
```

returning
`["-c", script, "bob-mac-capture-relaunch", String(processIdentifier), bundlePath]`,
where `script` reads the PID and path from `$1`/`$2`:

```sh
n=0
while [ "$n" -lt 100 ] && kill -0 "$1" 2>/dev/null; do
  /bin/sleep 0.1
  n=$((n + 1))
done
i=0
while [ "$i" -lt 3 ]; do
  /usr/bin/open "$2" && exit 0
  /bin/sleep 1
  i=$((i + 1))
done
exit 1
```

Three details in that script are deliberate and should be preserved with comments:

- The PID and bundle path are passed as **positional parameters**, never interpolated
  into the script text. `Bob Mac Capture.app` contains spaces, and this removes shell
  quoting from the correctness argument entirely.
- The wait loop is **bounded** at roughly ten seconds so a cancelled termination cannot
  leave an immortal poller behind. If the app is somehow still alive when the loop gives
  up, `open` just activates the existing instance, which is harmless.
- The `open` retry loop covers a Restart clicked during the brief window in which
  `Scripts/install.sh` has renamed the old bundle away and not yet moved the new one in.

`func restart() throws` then: verify `bundleURL.pathExtension == "app"`, verify
`fileExists(bundleURL.path)`, `try spawn("/bin/sh", Self.relaunchArguments(...))`, and
only then call `terminate()`. Every failure throws before anything terminates.

### 2. `Sources/BobMacCapture/AppDelegate.swift`

- Extract the status-item menu into `static func makeStatusMenu() -> NSMenu`, mirroring
  the existing `makeMainMenu()`/`makeAppMenu()` factories, and have
  `configureStatusItem()` call it. Leave every item's `target` nil so actions keep
  routing down the responder chain to the app delegate exactly as they do today.
- Insert
  `NSMenuItem(title: "Restart Bob Mac Capture", action: #selector(restartApp), keyEquivalent: "")`
  immediately after the existing separator and immediately before the Quit item,
  grouping it with the other lifecycle action. Give it no key equivalent: `q` and `,` on
  Quit and Settings mirror standard macOS shortcuts, and there is no standard shortcut
  for restart.
- Add an injectable `var relauncher = AppRelauncher()` property (non-private, mirroring
  the existing `settingsPresentation` test seam that
  `testOpenSettingsMenuActionUsesRegisteredSettingsPresenterOnce` already replaces).
- Add `@objc private func restartApp()`, which emits
  `CaptureSignpost.event("restart-requested")`, calls `try relauncher.restart()`, and on
  a thrown error sets `settings.diagnosticStatus = "Restart failed: <reason>"` and posts
  the restart-failure notification from step 3.

### 3. `Sources/BobMacCapture/NotificationService.swift`

Add
`nonisolated static func restartFailureContent(message: String) -> UNMutableNotificationContent`
with the title `"Restart failed"`, plus `func notifyRestartFailure(message:)`, mirroring
the existing `failureContent`/`notifyCaptureFailure` pair. The status menu closes the
moment the item is clicked, so a refusal that leaves the app running needs to be visible
without opening Settings; reusing `failureContent` would mislabel it "Capture failed".

### 4. Tests

In `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, add
`testStatusMenuOffersRestartBeforeQuit`: assert the full title, action, and
key-equivalent sequence of `AppDelegate.makeStatusMenu()`, including that the separator
sits directly above Restart and that Restart precedes `Quit Bob Mac Capture`. This is
the status-menu analogue of the existing
`testMainMenuExposesStandardEditSelectorsAndQuit` guard.

Add `Tests/BobMacCaptureTests/AppRelauncherTests.swift` with fakes recording call order:

- `testRestartSpawnsTheWaiterBeforeTerminating` — spawn is recorded before terminate,
  the launch path is `/bin/sh`, `args[0] == "-c"`, and the PID string and bundle path
  appear as discrete argv elements.
- `testRestartRefusesWhenNotRunningFromAnAppBundle` — a `.build/debug/BobMacCapture`
  path throws `.notBundled`; neither spawn nor terminate is called.
- `testRestartRefusesWhenTheInstalledBundleIsMissing` — an `.app` path with `fileExists`
  returning false throws `.bundleMissing`; terminate is never called.
- `testRestartDoesNotTerminateWhenTheHelperFailsToSpawn` — a throwing spawn propagates
  and terminate is never called. This is the regression guard for the safety property.
- `testRelaunchArgumentsNeverInterpolateThePathIntoTheScript` — a bundle path containing
  a space and a double quote survives as its own argv element and does not appear
  anywhere inside the script string.

Add `testRestartFailureContentIsLabelledDistinctlyFromCaptureFailure` to
`Tests/BobMacCaptureTests/NotificationServiceTests.swift`.

### 5. `README.md`

- Document the status-menu items, including Restart, in the Runtime Contract section.
- In "Updating, Reinstalling, and Rollback", make Restart the documented final step
  after `just install`: the already-running process keeps executing the old code until
  it is restarted or the user logs in again, and `Bob → Restart Bob Mac Capture` is how
  that handoff happens.
- State that Restart discards an unsent draft exactly as Quit does, and that it refuses
  to quit (reporting instead) when the app is not running from an installed `.app`
  bundle.
- Add a Troubleshooting entry for "Restart reports 'Restart failed'", covering the
  unbundled-run and missing-bundle causes and pointing at Settings → Diagnostics.

## Validation

Local checks on the Linux host, where the AppKit target cannot be compiled at all —
report these results honestly rather than claiming a build that did not happen:

1. `swift-format lint --recursive Package.swift Sources Tests` parses every file and
   exits non-zero on a syntax error. The pre-existing `[Indentation]` **warnings** are
   expected (the repository uses 4-space indentation while swift-format defaults to 2)
   and lint still exits 0 — do not reformat the repository to silence them.
2. `swift build --target CaptureCore` and `swift build --target CaptureCoreTests` still
   succeed, guarding against accidentally pulling AppKit into the Foundation-only
   target.
3. `swift build --target BobMacCapture` fails with `no such module 'AppKit'` on Linux.
   This is expected and is not evidence of a defect in the change; do not restructure
   the new code to make it build on Linux.

Full verification therefore happens on macOS and in CI:

4. On a macOS 26 host: `just format-lint`, `just build`, `just test`, `just bundle`, and
   `just install ~/Applications`.
5. The GitHub Actions `macos-26` workflow must be green end to end, including the launch
   smoke test and the install/reinstall step.
6. Manual verification on the Mac, after installing the built bundle: a. Click
   `Bob → Restart Bob Mac Capture`. The menu-bar item disappears and returns within a
   couple of seconds, exactly one `Bob` item is present afterwards, and Settings →
   Diagnostics reports "Hotkey registered: …", not "Hotkey conflict". b. Press
   Control-Shift-Command-I after the restart and confirm the capture panel opens,
   proving the Carbon hotkey was released and re-registered. c. `pgrep -x BobMacCapture`
   before and after shows a different PID and exactly one process. d. End-to-end update
   path: make a visible change, `just bundle` + `just install`, click Restart, and
   confirm the new behavior is live without any manual quit or relaunch. e. Refusal
   path: run the app unbundled (`swift run BobMacCapture`), trigger Restart, and confirm
   the app keeps running and surfaces the failure instead of quitting. f.
   `log show --last 2m --predicate 'subsystem == "org.bobs.bob-mac-capture"'` shows
   `restart-requested` followed by a fresh `launch-complete`.

## Scope constraints

- Do not add Restart to the application's main-menu App menu, and do not give it a
  global keyboard shortcut.
- Do not add a confirmation dialog; match Quit's existing draft-discarding behavior.
- Do not use `open -n`, and do not launch the new instance before the old process exits.
- Do not use `NSRunningApplication.current.bundleURL` to locate the bundle.
- Do not place relaunch logic in `CaptureCore`; that target is the Foundation-only `bob`
  process layer and app lifecycle does not belong there.
- Do not add App Sandbox entitlements or otherwise change the signing and
  hardened-runtime configuration in `Scripts/bundle.sh`.
- Do not persist the capture draft to disk so it can survive a restart.
- Do not bump `CFBundleShortVersionString`/`CFBundleVersion` or add a version indicator
  to the menu. A build-stamped version display would make it easy to confirm which build
  is running after a restart, but it needs build-time stamping and belongs in its own
  task bead.
- Do not add a CI assertion that exercises the restart action end to end; the existing
  launch smoke test remains the only launch gate.
- Make no changes in `bob-cli` itself.
