---
tier: tale
title: Restart Bob Mac Capture automatically after a successful reinstall
goal:
  A successful just install restarts the exact installed app it replaced when that app
  was running, while stopped installs remain stopped and installer safety contracts
  remain intact.
size: medium
proposed_by: bbugyi200.athena.01k
create_time: 2026-09-09 20:00:40
status: wip
---

# Plan: Restart Bob Mac Capture automatically after a successful reinstall

All implementation work belongs in the linked `bob-mac-capture` repository. Open it with
`/sase_repo` before editing and use the path returned by that command. The `bob-cli`
capture protocol and this repository's source files are not involved.

## Objective

Make `just install [target] [identity]` restart Bob Mac Capture when the copy at the
selected install target was already running on the invoking Mac. A fresh install, an
update while the app is stopped, a raw `swift run BobMacCapture` process, and a copy
running from the other supported install target must remain stopped/untouched.

The observable successful-update sequence is:

1. Build and fully verify the staged bundle without disturbing the running app.
2. Record the running instance(s) whose bundle path is exactly the selected install
   path.
3. Perform the existing atomic swap, rollback-on-install-failure, and post-install
   signature verification.
4. If a matching instance is still running, ask it to terminate normally, wait for every
   outgoing PID to exit, and then open the newly installed bundle exactly once.
5. Return only after the restart has been initiated successfully. Continue emitting only
   the installed path on stdout on success.

This deliberately follows the existing menu-bar `AppRelauncher` invariants: launching
before the old PID exits would let LaunchServices reactivate the old process and would
leave the Carbon hotkey owned by the wrong instance; `open -n` would instead create an
overlap and a hotkey conflict. The installer therefore needs explicit process discovery,
graceful termination, bounded exit waiting, and launch-after-exit behavior rather than
an unconditional `open` or a broad `pkill`.

## Implementation

### 1. Add a testable macOS install-relaunch helper

Add a small macOS-only Swift executable target, for example
`BobMacCaptureInstallHelper`, in `Package.swift`, with its source under a
correspondingly named directory in `Sources/`. The helper is a repository-side
installation tool, not a nested executable in `Bob Mac Capture.app`; it therefore does
not change the app's bundle layout, signature, identifier, entitlements, or runtime
menu.

Give the helper two narrowly scoped operations that communicate with
`Scripts/install.sh` through explicit argv and machine-readable stdout:

- **Discover before replacement:** enumerate
  `NSRunningApplication.runningApplications(withBundleIdentifier: "org.bobs.bob-mac-capture")`,
  normalize each non-nil `bundleURL`, and emit only the decimal PIDs whose launch-time
  bundle path equals the normalized selected `install_path`. Perform this snapshot after
  the staged bundle and helper have built successfully but before `install_path` is
  renamed. This timing matters because a running application's reported bundle URL can
  follow the old bundle to the temporary backup path during the swap.
- **Restart after replacement:** accept the captured PIDs and the installed bundle path,
  resolve each still-live PID back to an `NSRunningApplication`, and defensively confirm
  its bundle identifier before acting. If none remain, return success without launching
  anything; this respects a user who quit during the narrow swap window. Otherwise call
  `terminate()` on every still-live captured instance, wait until all of them are gone,
  and only then invoke `/usr/bin/open` for the installed bundle once. Bound the exit
  wait to roughly the same ten seconds as `AppRelauncher`, retry `open` a small bounded
  number of times as the menu restart does, and return a nonzero status with a useful
  stderr diagnostic on a refused termination, timeout, or launch failure. Never use
  `forceTerminate()`, `open -n`, process-name-only selection, or shell-interpolated
  bundle paths.

Keep process enumeration, termination/waiting, sleeping, and launching behind small
injectable closures/protocols so the policy can be unit tested without killing or
opening a real application. Parse arguments strictly, reject malformed/non-numeric PIDs,
keep paths as discrete argv values, and reserve stdout for discovery results so the
installer can capture it without contaminating its own output contract.

Do not try to drive the existing menu item through UI scripting or Apple Events: those
paths introduce Accessibility/Automation permissions and cannot reliably distinguish the
installed copy being replaced. The existing in-process `AppRelauncher` remains the right
implementation for a user-initiated restart; the install helper mirrors its lifecycle
ordering from outside the outgoing process and uses `NSRunningApplication.terminate()`
so `applicationWillTerminate` still releases the hotkey, watcher, and active Bob
subprocess cleanly.

### 2. Orchestrate restart from `Scripts/install.sh`

After `Scripts/bundle.sh` succeeds, build the helper with the same
`Scripts/xcode-swift.sh`/release toolchain and resolve its binary from SwiftPM's
reported binary directory. Keep all nested Swift build output on stderr, as today. A
helper-build or discovery failure must occur before the installed bundle is touched.

Immediately before the current backup/rename transaction, call the helper's discovery
operation and retain its PID list in shell state. Preserve the exact existing target
validation, staged signature/bundle-ID checks, same-directory temporary and backup
paths, swap, restoration traps, and post-install signature verification.

Only after the new installed copy passes post-install verification and the old backup
has been removed should `install.sh` invoke the helper's restart operation. Pass the
captured PIDs and `install_path` as distinct arguments. With an empty PID set, skip the
restart operation (or let it perform its documented no-op) so installing never starts an
app that was not running. With captured PIDs that have all disappeared, likewise leave
the app stopped.

Treat restart failure as an incomplete `just install`: print a clear stderr message and
exit nonzero, but do **not** roll back a bundle that has already passed all installation
checks. The diagnostic must say that the new bundle is installed at `install_path` even
though the running-process handoff failed, so the user can start it or use the existing
menu restart if an old instance remains. On a complete success, preserve the established
contract that stdout contains exactly one line, the final installed path; helper build,
discovery, termination, wait, and launch diagnostics belong on stderr.

### 3. Cover discovery and lifecycle ordering with tests

Add a macOS-only test target for the helper (or place tests in an existing macOS test
target if that produces a cleaner SwiftPM graph). Use fake running-application records
and injected operations to cover at least:

- discovery selects the exact normalized target bundle path and bundle identifier while
  ignoring the other supported install location, a same-named raw/debug executable,
  missing bundle URLs, and unrelated applications;
- an empty snapshot and a snapshot whose processes exited before handoff are no-ops and
  never call terminate or open;
- one or multiple matching live PIDs are all asked to terminate, and the installed
  bundle is opened exactly once only after all are observed terminated;
- a PID that now belongs to a different bundle identifier is ignored rather than killed,
  guarding the capture-to-restart race;
- termination refusal, exit timeout, and exhausted `open` retries fail without forcing a
  process or launching early;
- PID and argument parsing rejects malformed input, and a path containing spaces or
  quotes remains an uninterpolated argument.

Retain the existing `AppRelauncherTests`: they continue to guard menu-bar restart. Add
only narrowly useful shared constants/helpers between the two implementations; do not
move app lifecycle behavior into the Foundation-only `CaptureCore` target.

### 4. Extend the macOS installer integration gate

Expand `.github/workflows/ci.yml`'s existing **Install and reinstall** step rather than
adding a disconnected mock-only check:

1. Install into the temporary `HOME` while no app is running and verify the app is not
   auto-launched, stdout is exactly the expected path, and the plist, bundle identifier,
   and signature remain valid.
2. Launch that installed bundle through LaunchServices, wait for its exact
   `BobMacCapture` PID and `launch-complete` signpost, then reinstall it. Assert the old
   PID exits, a different PID appears, exactly one app instance remains, and a new
   `launch-complete` event proves the replacement completed application startup.
3. Terminate the app, wait for its PID to disappear, install once more, and assert it
   stays stopped. This is the regression guard against turning install into
   unconditional launch.

Use bounded polling and an EXIT cleanup trap so a failed assertion cannot leave the app
or helper running on the hosted Mac. When matching processes in this integration check,
also tie the process back to the temporary installed bundle so an unrelated runner
process cannot satisfy the test.

### 5. Update user-facing installation documentation

Revise `README.md`'s Development and **Updating, Reinstalling, and Rollback** sections
to make `just install` the complete update operation: it restarts the selected installed
copy only when that copy was running, otherwise it leaves the app stopped. Remove the
now-obsolete instruction to choose `Bob -> Restart Bob Mac Capture` after every install,
while retaining the menu item as the documented manual restart mechanism.

Document that automatic restart has the same unsent-draft behavior as Quit/manual
Restart, that an app launched from the other install target or from `swift run` is not
touched, and that a restart-stage error can leave a verified new bundle installed even
though the command returns nonzero. Keep the signing-identity, launch-at-login,
notification authorization, rollback, and uninstall guidance otherwise unchanged.

## Validation

On macOS, run:

1. `bash -n Scripts/bundle.sh Scripts/install.sh Scripts/xcode-swift.sh`.
2. `just format-lint`, `just build`, and `just test`, including the helper unit tests
   and all existing `AppRelauncherTests`.
3. `just bundle` followed by signature, plist, and bundle-identifier verification to
   confirm the helper target did not enter or alter the application bundle.
4. Exercise the CI temporary-`HOME` sequence locally when practical: stopped install,
   launch, running reinstall with PID turnover and one healthy replacement, then stopped
   reinstall with no launch. Repeat with a target path whose parent contains spaces if
   the helper tests do not already exercise that boundary end to end.
5. Run `just install ~/Applications` against the real app in both states. When running,
   verify the command returns only after the old PID has exited, a single new PID has
   launched, the production hotkey opens the capture panel, and Diagnostics does not
   show a hotkey conflict. When stopped, verify no process appears.
6. Confirm the `macos-26` GitHub Actions workflow passes, especially launch smoke,
   bundle signing, and all three installer states.

On a non-macOS planning/inspection host, limit local checks to portable shell syntax and
repository review; AppKit helper compilation and restart behavior must be reported as
macOS/CI validation rather than claimed from Linux.

## Acceptance criteria and scope

- A successful `just install` restarts exactly the selected installed Bob Mac Capture
  instance if it is running and produces a different live PID with no overlap/hotkey
  conflict.
- If that exact installed copy is not running, `just install` does not launch it and
  does not disturb a raw build or a copy at the other supported target.
- No termination or relaunch is attempted until the new bundle has passed post-install
  verification; pre-swap failures retain the old running process and existing rollback
  behavior.
- Automatic restart uses normal application termination, bounded waiting, and
  launch-after-exit ordering. It never broad-kills by executable name, force-kills,
  launches before the old process exits, or uses `open -n`.
- Successful installer stdout remains exactly the installed path. Restart-stage failures
  are nonzero and explicit without rolling back an already verified installation.
- The menu-bar Restart action and its tests remain functional and unchanged in user
  behavior; no confirmation dialog, draft persistence feature, version bump, signing or
  entitlement change, launch-at-login change, or `bob-cli` protocol/source change is in
  scope.
