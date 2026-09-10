---
tier: tale
title: Restore bob-mac-capture GitHub Actions CI
goal:
  The macOS 26 SwiftPM workflow passes end to end after focused test compatibility
  repairs.
size: small
proposed_by: bbugyi200.athena.00f
create_time: 2026-09-09 19:59:46
status: wip
---

# Restore bob-mac-capture GitHub Actions CI

## Objective

Make the `bobs-org/bob-mac-capture` `macOS 26 SwiftPM` job pass without changing
application behavior, then exercise every command in `.github/workflows/ci.yml`,
including the bundle and signature checks that the failing Test step currently masks.

## Diagnosed root cause

`actstat` identifies GitHub Actions run `31793863709` on `master` at
`70589ca524ab5f9780672850ffbceff763698096`. The job builds successfully and fails while
compiling `swift test`. The triggering production commit only changes live preview
response decoding; the failing test constructs predate it and CI has never completed
successfully in the repository's recorded run history.

The current compiler errors are confined to test code:

- `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` calls the `@MainActor`-isolated
  `CapturePanelController.makePanel()` from a nonisolated test method.
- The same file passes Carbon's `eventHotKeyExistsErr`, imported as `Int`, where the
  fake registrar and `HotKeyRegistrationError` require `OSStatus` (`Int32`). The
  `.development` inference diagnostic is a cascade from the ill-typed setup.
- `Tests/BobMacCaptureTests/NotificationServiceTests.swift` tries to construct
  `UNAuthorizationStatus.ephemeral`, a case explicitly unavailable on macOS.
- Notification tests also access main-actor-isolated `NotificationService` static
  members from nonisolated methods. These are warnings in Swift 5 mode but should be
  aligned with the actor boundary while the tests are being repaired.

The checkout must be updated to the diagnosed `origin/master` before editing. The
current execution host is Linux and has no Swift or Xcode toolchain, so authoritative
build, test, bundle, plist, and signing validation requires a macOS 26 environment.

## Implementation

1. Open `gh:bobs-org/bob-mac-capture` through `sase repo open`, confirm the worktree is
   clean, fetch `origin`, and update the local branch to the diagnosed `origin/master`
   without discarding unrelated user changes. Stop and report if the target has moved
   incompatibly or the worktree is no longer clean.
2. In `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, run the panel-construction
   test on `MainActor`. Give the Carbon conflict code an explicit `OSStatus` value and
   reuse that typed value for both the fake registrar and expected error. Preserve the
   assertions and production hot-key behavior; qualify `HotKeyConfiguration.development`
   only if a non-cascading inference error remains.
3. In `Tests/BobMacCaptureTests/NotificationServiceTests.swift`, remove the impossible
   macOS construction of `.ephemeral` while retaining coverage for every macOS-available
   authorization status. Mark methods that exercise `NotificationService`'s
   actor-isolated surface as `@MainActor` so the test contract matches the production
   isolation boundary. Do not broaden this repair into unrelated deprecation or
   Sendable-warning cleanup.
4. Review the focused diff and verify it contains only the test compatibility changes
   needed for the diagnosed errors.

## Validation

On macOS 26 with the repository's configured Swift/Xcode toolchain, reproduce the CI
steps in order and fix any directly exposed regression before considering the repair
complete:

1. Run the workflow's Swift formatting lint over `Package.swift`, `Sources`, and
   `Tests`.
2. Run `swift build`.
3. Run `swift test` and confirm all test targets compile and all tests pass, with none
   of the diagnosed actor-isolation, `OSStatus`, inference, or unavailable-case errors.
4. Run `./Scripts/bundle.sh --identity "-"`.
5. Run `plutil -lint` and `codesign --verify --deep --strict` against
   `.build/bundle/Bob Mac Capture.app`, and confirm its bundle identifier is
   `org.bobs.bob-mac-capture`.
6. If the change is published through an explicitly authorized workflow, inspect the
   resulting GitHub Actions run and use `actstat` to confirm the `macOS 26 SwiftPM` job
   is green. Otherwise, report that remote CI remains pending publication rather than
   claiming it passed.

## Completion criteria

- The three current test compilation root causes are corrected without application
  behavior changes.
- The full macOS 26 workflow sequence passes in an environment capable of running it,
  including the two steps previously skipped after `swift test` failed.
- Any remote status claim is backed by a new GitHub Actions run for the repaired SHA.
