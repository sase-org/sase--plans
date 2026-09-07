---
tier: tale
title: Notify when an install-triggered restart completes
goal:
  A running-copy just install produces one polished, actionable macOS completion
  notification from the successfully relaunched app without changing install
  reliability.
size: medium
proposed_by: bbugyi200.athena.01k.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-xz.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xz.1/README.md)
- **COMMITS:**
  - [7ca1654](https://github.com/sase-org/sase/commit/7ca1654a2175b3e042b862f9bacb20f04535d2bc)
    — feat(pager): add source-language facade over the rust contract

# Plan: Notify when an install-triggered restart completes

All implementation work belongs in the linked `bob-mac-capture` repository. Open it with
`/sase_repo` before editing and use the path returned by that command. The `bob-cli`
capture protocol and source tree are not involved.

## Objective and user experience

When `just install [target] [identity]` replaces a Bob Mac Capture bundle that was
already running, have the newly launched replacement post exactly one native macOS
notification after `applicationDidFinishLaunching` completes its normal setup. The
notification is confirmation that the replacement process itself reached a usable launch
point, rather than merely confirmation that the install helper handed a request to
LaunchServices.

Use concise, static copy that remains truthful for an upgrade, same-version reinstall,
or rollback:

- title: `Install complete`
- body: `Bob Mac Capture restarted successfully.`
- sound: the default notification sound
- action: `Capture`

Assign a dedicated install-restart category. Clicking the notification body or choosing
`Capture` should bring up the capture panel; dismissing it should do nothing. Keep the
content free of install paths, signing identities, versions, capture text, and other
dynamic or private data. Let native Notification Center supply layout and the app
identity rather than adding a custom attachment or bespoke window.

This is a best-effort user notification, not part of installation correctness. Respect
the app's existing notification authorization: do not prompt during install or startup,
and do not fail, delay, or retry `just install` when authorization is absent/denied or
Notification Center rejects the request. Existing authorization guidance and the
signing-identity caveat continue to apply.

## Implementation

### 1. Carry a launch-scoped install-restart signal through the helper

Define one narrowly scoped, Foundation-only launch argument contract in `CaptureCore`
(for example `BobMacCaptureLaunchContext.installRestartArgument`) so the main app and
the separate `BobMacCaptureInstallHelper` executable cannot silently drift to different
string literals. Its parser should inspect explicit argument arrays, match the reserved
argument exactly, and remain independently unit-testable. Make the helper target depend
on `CaptureCore`; do not move process discovery, termination, launching, AppKit state,
or notification behavior into the shared module.

Change `InstallRelauncher`'s injected open operation so it receives both the bundle path
and the application arguments. Its production implementation should continue invoking
`/usr/bin/open` with discrete `Process.arguments`, adding `--args` followed by the
shared install-restart argument. Do this only in the existing `restart` path, after
every captured live PID has accepted graceful termination and exited. Preserve the
existing bounded open retries, no-op behavior for empty/exited PID snapshots, exact
bundle/PID selection, no-`open -n` rule, and shell-free path handling.

Do not add a marker file, `UserDefaults` key, environment variable, Apple Event, or
distributed notification. A process argument is one-shot, cannot survive a failed launch
and cause a misleading banner later, requires no cleanup/expiry race, and is delivered
to the replacement process that will own the notification. `Scripts/install.sh` should
retain its current orchestration, stdout contract, and exit semantics; it need not know
the private launch argument.

### 2. Post the notification only from the replacement app

At the end of `AppDelegate.applicationDidFinishLaunching`, after status-item, Bob
client, panel, hotkey, watcher, diagnostics, and existing `launch-complete` setup, parse
`ProcessInfo.processInfo.arguments`. When and only when the exact install-restart signal
is present, emit a metadata-only signpost such as
`install-restart-notification-requested` and ask `NotificationService` to post the
install-complete content once. Do not remove or rewrite `ProcessInfo.arguments`, and do
not notify for an ordinary Finder/LaunchServices launch, `swift run`, direct binary
launch, the menu-bar Restart action, or an install that found the selected copy stopped.

Add a dedicated `NotificationService` entry point and a pure static content builder for
the install-complete notification. Reuse the existing UUID request IDs, async
scheduling, foreground `.banner`/`.sound`/`.list` presentation, and intentionally
non-fatal error handling. Do not request authorization from this path and do not report
a success notification before the replacement reaches `applicationDidFinishLaunching`.

### 3. Make the completion notification useful when clicked

Register an install-restart notification category alongside the existing single- and
multi-capture categories, with one foreground `Capture` action. Extend the pure response
routing policy so both the default body click and that explicit action for the
install-restart category produce a `showCapture` route, while dismissal and mismatched
category/action combinations remain no-ops.

Inject a MainActor capture-panel callback into `NotificationService` alongside its
existing URL opener, and have its delegate execute the computed route: open Obsidian
targets for existing capture notifications or call back into `AppDelegate` to show the
capture panel for the install-complete notification. Preserve existing default-click,
`Open Note`, and `Open Notes` behavior and avoid activating/showing UI when a
notification is merely delivered.

### 4. Cover the cross-process contract and notification UX with tests

Extend the current test targets rather than creating a UI-test bundle:

- In `CaptureCoreTests`, verify the install-restart argument is recognized exactly and
  unrelated, prefix/suffix, empty, and ordinary launch arguments do not opt in.
- In `BobMacCaptureInstallHelperTests`, assert the helper opens the selected path once,
  after every outgoing PID exits, with `--args` and the shared signal as distinct
  arguments. Preserve and adapt all refusal, timeout, retry, PID-reuse, empty-snapshot,
  and quoted-path tests; a no-op restart must never construct an open request.
- In `BobMacCaptureTests`, assert the exact title/body, default sound, dedicated
  category, empty privacy-sensitive metadata, category registration, and response
  routes. Cover body click and `Capture`, dismissal/mismatches, the injected panel
  callback, and regression cases for existing singular/plural Obsidian routes.

Keep live `UNUserNotificationCenter` objects out of unit tests, following the existing
pure-builder/routing structure. The launch-argument parser plus the macOS integration
gate should cover app lifecycle wiring without trying to construct a full app delegate
launch inside XCTest.

### 5. Extend installer integration coverage and documentation

In `.github/workflows/ci.yml`, extend the existing bounded **Install and reinstall**
gate. During its running-copy reinstall case, require a fresh
`install-restart-notification-requested` signpost from the replacement process in
addition to the existing old-PID exit, new-PID, singleton, and `launch-complete`
assertions. Keep the fresh/stopped install cases proving that no app launches at all. Do
not try to grant notification permission or assert banner pixels in CI; the signpost
proves that LaunchServices delivered the one-shot signal and the replacement requested
the notification, while unit tests prove the content and routing. Retain bounded polling
and EXIT cleanup.

Update `README.md`'s Notifications and **Updating, Reinstalling, and Rollback** sections
to describe the one-shot completion banner, its `Capture` interaction, and its scope: it
appears only after an install-triggered restart and only when macOS notification
permission allows it. State that denied/not-yet-requested authorization is silent and
non-fatal, an install never prompts for permission, a stopped install does not launch or
notify, and menu-bar Restart does not masquerade as an install completion.

## Validation

On macOS, run:

1. `bash -n Scripts/bundle.sh Scripts/install.sh Scripts/xcode-swift.sh`.
2. `just format-lint`, `just build`, and `just test`, including the shared
   launch-context, helper, content/category, and response-routing coverage.
3. `just bundle` and verify the plist, bundle identifier, and deep signature; confirm
   `BobMacCaptureInstallHelper` and no new marker/resource file are embedded in the app.
4. Run the CI temporary-`HOME` sequence: stopped install with no process, ordinary
   launch with no install-notification signpost, running reinstall with PID turnover,
   `launch-complete`, and one fresh install-notification-request signpost, then stopped
   reinstall with no process.
5. With notifications authorized for a consistently signed installed bundle, keep the
   app running and run `just install`. Confirm one native banner with the exact copy and
   sound appears only after the new PID launches; body click and `Capture` each open the
   capture panel; dismissal has no side effect; the hotkey and normal capture
   notifications still work.
6. Repeat with the selected app stopped and with `Bob -> Restart Bob Mac Capture`;
   neither should show the install-complete notification. Exercise denied and
   not-determined authorization and confirm there is no permission prompt and
   installation/relaunch still succeeds. Confirm an induced restart/open failure never
   produces a false success notification.
7. Confirm the `macos-26` GitHub Actions workflow passes, especially the installer gate,
   launch smoke test, app/helper tests, and signing checks.

On a non-macOS planning/inspection host, limit local checks to portable shell syntax and
repository review; AppKit, UserNotifications, `/usr/bin/open --args`, notification UI,
and the macOS integration gate must be reported as macOS/CI validation rather than
claimed locally.

## Acceptance criteria and scope

- A running-copy `just install` causes the replacement app to request exactly one
  install-complete notification after its normal launch setup, with the approved static
  copy, default sound, and native presentation.
- The launch signal is one-shot and process-scoped: no stale state can produce a later
  false notification, and fresh/stopped installs, ordinary launches, raw builds, and
  menu-bar restarts do not request it.
- The notification's body click and `Capture` action open the capture panel; dismissal
  does nothing; existing capture-notification actions keep their current behavior.
- Missing, denied, or reset notification authorization never triggers a prompt and never
  changes installer output, status, rollback, restart ordering, or app startup success.
- Restart refusal, timeout, or launch failure cannot claim success. Successful installer
  stdout remains exactly the installed path, and helper/open diagnostics remain on
  stderr.
- No persistent handoff marker, force termination, broad process matching, custom
  notification window/attachment, new entitlement, bundle identifier/signing change,
  version bump, launch-at-login change, `bob-cli` change, or menu-restart behavior
  change is in scope.
