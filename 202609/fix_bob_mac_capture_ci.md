---
tier: tale
title:
  Fix the red bob-mac-capture CI by removing implicit self capture in
  InstallRelauncherTests
goal:
  The bob-mac-capture `CI` workflow on `master` is green again, end to end, including
  the install/reinstall and signpost steps that have never successfully executed.
size: medium
proposed_by: bbugyi200.athena.065
create_time: 2026-09-09 20:00:38
status: wip
---

# Plan: Fix the red bob-mac-capture CI

## Summary

`bobs-org/bob-mac-capture` CI has failed on every push since `871eaed` (5 consecutive
red runs on `master`, most recently run `34154292497` on `e5d7306`). Every run dies at
the same place: job `macOS 26 SwiftPM`, step `Test` (`./Scripts/xcode-swift.sh test`),
exit code 1.

The failure is a **compile error, not a test assertion failure**. Fix it by moving four
test fixtures in one file out of the test class and into file scope.

## Root cause

`Tests/BobMacCaptureInstallHelperTests/InstallRelauncherTests.swift` references the test
class's own instance members from inside closure literals:

- the stored property `applicationsPath` (declared at line 7), and
- the instance method `record(pid:path:bundleIdentifier:)` (declared at line 345)

Those closures are arguments to `InstallRelauncher`'s memberwise initializer.
`InstallRelauncher` (`Sources/BobMacCaptureInstallHelper/InstallRelauncher.swift`) is a
struct whose `runningApplications`, `applicationForPID`, `terminate`, `isTerminated`,
`sleep`, and `open` parameters are **stored properties**, which makes the closures
escaping. Swift requires implicit-`self` references inside an escaping closure to be
written explicitly, so the compiler emits 13 errors of the form:

```
InstallRelauncherTests.swift:80:21: error: call to method 'record' in closure requires
  explicit use of 'self' to make capture semantics explicit
InstallRelauncherTests.swift:145:38: error: reference to property 'applicationsPath' in
  closure requires explicit use of 'self' to make capture semantics explicit
```

Affected lines in the current `master` (`e5d7306`) revision: **80, 145, 163, 187 (×2),
210 (×2), 236 (×2), 282 (×2), 299**. `swift build` is unaffected and the CI `Build` step
passes, because `swift build` does not compile test targets.

## Why nobody caught it locally

`BobMacCaptureInstallHelper` and its test target are macOS-only: `Package.swift` gates
them behind `#if os(macOS)`, and `InstallRelauncher.swift` imports AppKit. On the Linux
dev host `swift build` compiles only `CaptureCore` (verified: `Build complete!` with
just the 14 `CaptureCore` jobs), and `Scripts/xcode-swift.sh` refuses to run at all
without Apple developer tools. GitHub Actions is therefore the **only** gate that ever
type-checks this file, and five commits landed on `master` without anyone watching it.

## Secondary exposure the implementer must expect

The compile abort happens early in the test build. The CI log shows
`[7/24] Compiling BobMacCaptureInstallHelperTests InstallRelauncherTests.swift` failing,
then `[11/24] Emitting module BobMacCaptureTests`, then `error: fatalError` — the
per-file body type-checking jobs for `BobMacCaptureTests` never ran, and no test ever
executed.

Consequences:

1. Roughly 900 lines of test code added in `871eaed`, `78315d9`, `3f28a06`, `976a835`,
   and `e5d7306` (`BobMacCaptureTests.swift`, `NotificationServiceTests.swift`,
   `BobMacCaptureLaunchContextTests.swift`, `InstallHelperArgumentsTests.swift`) have
   **never been type-checked or run on macOS**.
2. The CI steps added in `871eaed` (+159 lines: `Install and reinstall`) and `3f28a06`
   (+30 lines: the `install-restart-notification-requested` signpost assertions) run
   _after_ `Test` and have therefore **never executed once**.

So the first genuinely green `Test` step is likely to expose further failures. Treat
this plan as "get CI green", not "land one edit". The last known-green commit is
`a6d6f4c` (run `34131259735`) — useful as a baseline when bisecting behaviour.

## Implementation

### 1. Open the repository

`bob-mac-capture` is a linked repo, not this checkout. Use the `/sase_repo` skill:

```bash
sase repo open bob-mac-capture -r "Fix the failing macOS CI Test step"
```

Use the printed path as the only path for all reads and writes below. All file paths in
this plan are relative to that path.

### 2. Apply the fix

In `Tests/BobMacCaptureInstallHelperTests/InstallRelauncherTests.swift`, move the
fixtures out of the class so no closure captures `self` at all:

- Delete the three instance stored properties from the class body (lines 7-9):
  `applicationsPath`, `homeApplicationsPath`, `rawExecutablePath`.
- Delete the trailing `private func record(...)` instance method (lines 345-355).
- Re-declare all four at **file scope** (after the imports, before the class), as
  `private let` constants and a `private func`. A file-scope `private` declaration is
  visible throughout the file, so **every one of the existing call sites stays
  byte-identical** — the diff is a pure move.

The resulting file-scope declarations:

```swift
private let applicationsPath = "/Applications/Bob Mac Capture.app"
private let homeApplicationsPath = "/Users/test/Applications/Bob Mac Capture.app"
private let rawExecutablePath = "/Users/test/bob-mac-capture/.build/debug/BobMacCapture"

private func record(
    pid: pid_t,
    path: String,
    bundleIdentifier: String = InstallRelauncher.bundleIdentifier
) -> RunningApplicationRecord {
    RunningApplicationRecord(
        processIdentifier: pid,
        bundleIdentifier: bundleIdentifier,
        bundleURL: URL(fileURLWithPath: path)
    )
}
```

Both `InstallRelauncher` and `RunningApplicationRecord` are `internal` in the helper
module and reachable through the file's existing `@testable import`, so `private`
file-scope declarations may name them.

**Why this shape rather than sprinkling `self.`:** these fixtures are pure constants and
a pure factory — they read nothing from the instance, so they do not belong on it.
Moving them removes the error class outright, so a future test that adds another closure
cannot reintroduce it, and it matches the sibling
`Tests/BobMacCaptureTests/AppRelauncherTests.swift`, which tests the same
struct-of-closures shape with no instance state whatsoever.

**Acceptable fallback** if the move runs into trouble: prefix the 13 references with
`self.` instead. That also compiles, and
`Tests/BobMacCaptureTests/CapturePanelModelTests.swift` already uses explicit
`self.analysisSettled(...)` inside escaping closures, so it is consistent with the
codebase. Prefer the move; use this only if the move does not work out.

### 3. Do not attempt to verify the fix by compiling locally

The target cannot build on this Linux host — that is the whole reason the bug shipped.
Do not burn time on `swift build`, `swift test`, or `./Scripts/xcode-swift.sh`; the
first two silently skip the macOS targets and the third exits 69 with a "no Apple
developer tools" message. Local `swift build` is still worth one run _only_ to confirm
`CaptureCore` was not disturbed.

Static checks that are worth doing before pushing:

- Keep every line at or below 100 characters (swift-format's default `lineLength`; the
  repo has no `.swift-format` config). The CI `Lint Swift formatting` step runs
  `swift-format lint --recursive Package.swift Sources Tests` **before** `Test`, so a
  formatting slip turns one red step into a different red step.
- Re-read the edited file end to end and confirm no remaining bare reference to
  `applicationsPath`, `homeApplicationsPath`, `rawExecutablePath`, or `record(` resolves
  to something that no longer exists.

### 4. Commit and push

Commit through the `/sase_git_commit` skill (the only sanctioned path for git commits).
Push to `master`, matching how every other commit in this repo landed.

### 5. Watch CI and iterate until green — this is the real work

```bash
actstat
```

or watch the specific run:

```bash
gh run watch --repo bobs-org/bob-mac-capture <run-id>
gh run view --repo bobs-org/bob-mac-capture --job <job-id> --log
```

For long waits use the `/sase_monitor` skill rather than blocking the turn.

Then handle what surfaces, in workflow order:

- **`Lint Swift formatting` fails** — run the same `swift-format lint` invocation
  mentally against the diff; fix and push.
- **`Test` still fails to compile** — expect this to be in `BobMacCaptureTests`, whose
  bodies have never been type-checked (see "Secondary exposure"). Read the errors from
  the log; they will be ordinary Swift diagnostics.
- **`Test` compiles but assertions fail** — the new key-router / notification /
  launch-context tests from the last four commits are running for the first time. Judge
  each failure on its merits: a genuinely wrong assertion in the new test gets fixed in
  the test; a real product bug gets fixed in `Sources/`. Do not weaken or delete an
  assertion just to get green — if a failure looks like a real product defect that is
  larger than this plan, fix it if it is bounded, and otherwise report it rather than
  silencing the test.
- **`Bundle` / `Verify plist and signature` / `Launch smoke test` /
  `Install and reinstall` fail** — these run for the first time since `871eaed` and
  `3f28a06` added them. The `Install and reinstall` step is the least-exercised: it
  installs into a temp `HOME`, launches through LaunchServices, reinstalls, and requires
  PID turnover plus both a `launch-complete` and an
  `install-restart-notification-requested` signpost. On failure, pull the
  `launch-smoke-test-diagnostics` artifact the workflow uploads and read the filtered
  unified-log output before changing anything.

Repeat until a full run is green. Report the final green run URL.

## Definition of done

- `Tests/BobMacCaptureInstallHelperTests/InstallRelauncherTests.swift` compiles with no
  implicit-`self`-in-escaping-closure errors.
- A `CI` run on `bobs-org/bob-mac-capture` `master` completes **successfully** — every
  step, including `Install and reinstall`, which has never passed.
- No test assertion was weakened or removed to reach green.
- The final green run URL is reported.

## Non-goals

- Do not restructure `InstallRelauncher`, the install helper, or the CI workflow beyond
  what a specific failure demands.
- Do not address the deprecated-`init(contentsOf:)` warnings in
  `Tests/CaptureCoreTests/BobProcessClientTests.swift`; they are warnings, not errors,
  and predate this breakage.
- Do not try to make the macOS-only targets buildable on Linux.

## Follow-up worth filing separately (not part of this plan)

The structural gap is that macOS-only targets are verified _only_ by GitHub Actions, so
a red `master` can persist unnoticed for five commits. Worth a separate task bead (via
`/sase_new_task`): a durable convention that agents touching `bob-mac-capture` must
watch CI to completion after pushing, since no local gate exists.
