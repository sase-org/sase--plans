---
tier: tale
size: small
title: Make bob-mac-capture use the Xcode Swift toolchain
goal:
  "`just test` reliably finds XCTest even when another Swift installation shadows Xcode
  on PATH."
proposed_by: bbugyi200.athena.00r
create_time: 2026-09-09 20:00:05
status: wip
---

# Make bob-mac-capture use the Xcode Swift toolchain

## Objective

Make the repository-controlled build, test, and bundle commands consistently use the
SwiftPM toolchain from the selected full Xcode 26 installation, so `just test` can
import `XCTest` regardless of an unrelated `swift` earlier on `PATH`. Preserve the
current test framework and application behavior.

## Diagnosed root cause

The failure is environmental tool selection rather than a missing package dependency or
an invalid test source:

- The failing checkout is current `master` at `f59ab74`, and GitHub Actions run
  `31799467198` passes on that exact SHA. Its macOS 26 runner uses Xcode 26.6 / Apple
  Swift 6.3.3 and successfully executes all 57 XCTest tests with the same `swift test`
  operation.
- Every reported local diagnostic stops at `import XCTest`; no test body or
  `@testable import` is reached. On macOS, XCTest is supplied by Xcode's platform
  developer frameworks, so adding XCTest to `Package.swift` or rewriting the tests would
  treat the symptom rather than the toolchain mismatch.
- `README.md` requires "SwiftPM from the selected Xcode toolchain," but `justfile` and
  `Scripts/bundle.sh` invoke bare `swift`. Those calls trust shell `PATH`, so a
  Swift.org / Swiftly installation or another non-Xcode driver can compile ordinary
  targets while lacking Xcode's XCTest module. A full-Xcode selection problem can
  produce the same symptom when the active developer directory points at standalone
  Command Line Tools.
- Apple's `xcrun` contract exists specifically to resolve developer tools from the
  active Xcode developer directory. Supplying both the macOS SDK and the default Xcode
  toolchain also avoids ambient `SDKROOT`, `TOOLCHAINS`, and `PATH` selecting an
  incompatible combination.

Before editing, capture `command -v swift`, `swift --version`,
`/usr/bin/xcode-select --print-path`, `/usr/bin/xcrun --sdk macosx --find swift`, and
`/usr/bin/xcrun --sdk macosx --show-sdk-version` on the affected Mac. This confirms
which of the two local selection paths triggered the mismatch and gives a before/after
record; it does not change the solution or expand scope.

## Implementation

1. Add a small executable helper under `Scripts/` that runs Swift through
   `/usr/bin/xcrun` with the `macosx` SDK and the Xcode default toolchain, forwarding
   every argument and exit status unchanged. Honor `DEVELOPER_DIR` / the active
   `xcode-select` choice so nonstandard Xcode application names remain supported, but do
   not honor a `PATH`-shadowing Swift or ambient alternate `TOOLCHAINS` selection.
2. Have the helper fail before compilation with one concise, actionable diagnostic when
   a full Xcode 26+ developer directory, its macOS 26+ SDK, or its Swift executable
   cannot be resolved. Include the selected developer directory and the command needed
   to select an installed Xcode; do not silently fall back to standalone Command Line
   Tools or another Swift installation.
3. Update `justfile` so the Xcode-resolved helper backs `build`, `test`, and the
   `swift format` fallback. Keep an explicitly installed `swift-format` usable for the
   existing fast path, and preserve recipe names, arguments, and default behavior.
4. Update `Scripts/bundle.sh` to use the same helper for both its product build and
   `--show-bin-path` lookup. Resolve the helper relative to the script/package root so
   bundling works from any current directory. Leave bundle assembly, signing, output,
   and install behavior unchanged.
5. Exercise the helper in `.github/workflows/ci.yml` for toolchain reporting, build, and
   test so CI validates the same entry point developers use. Keep the existing SDK
   version assertion and all later bundle/install checks.
6. Tighten the README requirement/troubleshooting text: show the commands that compare
   bare `swift` with Xcode-resolved Swift, explain that XCTest import failures indicate
   the wrong developer toolchain, and document selecting Xcode through `xcode-select` or
   one-command `DEVELOPER_DIR` without prescribing a test-framework migration.

## Validation

Perform static checks first, then run the macOS-only checks with the selected Xcode 26+
toolchain. Iterate on any directly related failure and rerun the full sequence:

1. Run `bash -n` on the new helper and `Scripts/bundle.sh`, `just --list`,
   `git diff --check`, and the repository's Swift formatting lint.
2. Confirm the diagnostic path without changing system selection by setting
   `DEVELOPER_DIR=/Library/Developer/CommandLineTools` for one helper `--version`
   invocation. It must fail early and name the full-Xcode selection remedy instead of
   reaching an `import XCTest` compiler error.
3. Put a temporary failing executable named `swift` first on `PATH`, then run
   `just build` and `just test`. Confirm the fake executable is never invoked, the
   production targets build, and all 57 existing XCTest tests pass. Remove the temporary
   directory afterward.
4. Run `just bundle` and verify `.build/bundle/Bob Mac Capture.app` with `plutil -lint`,
   `codesign --verify --deep --strict`, and a bundle-identifier check for
   `org.bobs.bob-mac-capture`. This proves the second formerly bare-Swift path uses the
   same toolchain and preserves packaging behavior.
5. Run `just all` once without the shadowed `PATH` to validate the documented local
   workflow end to end.
6. If the fix is later published through an explicitly authorized workflow, confirm a
   fresh `macOS 26 SwiftPM` GitHub Actions run is green. Otherwise, report remote CI as
   pending publication rather than claiming it passed.

## Completion criteria

- `just test` resolves Swift through the selected full Xcode 26+ toolchain and all 57
  existing XCTest tests pass even when bare `swift` on `PATH` is incompatible.
- Missing or incorrectly selected Xcode fails early with an actionable toolchain message
  rather than `no such module 'XCTest'`.
- Build, format fallback, bundle, and CI use the same repository-owned toolchain entry
  point; bundling and signing behavior are unchanged.
- No XCTest dependency is added to `Package.swift`, no tests are migrated solely to mask
  the environment error, and application source behavior is unchanged.
