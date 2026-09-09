---
tier: tale
title: Allow matching Apple Command Line Tools for bob-mac-capture
goal:
  Repository build, test, and bundle commands work with compatible Command Line Tools
  26+ without trusting a shadowing Swift executable or requiring full Xcode.
size: small
proposed_by: bbugyi200.athena.00r.f0
create_time: 2026-09-09 19:53:28
status: wip
---

# Allow matching Apple Command Line Tools for bob-mac-capture

## Objective

Make `just build`, `just test`, and `just bundle` use the selected Apple Swift toolchain
without requiring the full Xcode application when a compatible Command Line Tools for
Xcode 26+ installation is selected. Continue to isolate repository commands from a
Swift.org/Swiftly executable that shadows Apple's `swift` on `PATH`, and retain the
existing XCTest test suite.

## Diagnosed root cause

The new failure is produced by `Scripts/xcode-swift.sh` itself, before it invokes Swift.
The helper accepts only developer-directory strings ending in `/Contents/Developer`, so
it unconditionally rejects Apple's standard standalone developer directory,
`/Library/Developer/CommandLineTools`. Its accompanying claim that standalone Command
Line Tools omit XCTest and other required frameworks is too broad.

Apple's current Command Line Tools documentation states that the standalone package
contains the same macOS SDK and toolchain binaries shipped with Xcode; the relevant
distinction is that Xcode-only commands such as `xcodebuild` and `xctrace` are absent.
This repository uses SwiftPM plus macOS SDK frameworks and does not call either of those
Xcode-only commands. Therefore the helper should judge the selected developer tools by
their capabilities and SDK version, not by whether their directory lives inside an
`.app` bundle. A full Xcode 26+ installation remains a valid option, but should not be a
prerequisite when Command Line Tools 26+ can resolve and run the package tests.

Reference:
<https://developer.apple.com/documentation/xcode/installing-the-command-line-tools>

## Implementation

1. Update `Scripts/xcode-swift.sh` to remove the `/Contents/Developer`-suffix gate.
   Continue to resolve the developer directory from `DEVELOPER_DIR` or
   `/usr/bin/xcode-select --print-path`, clear ambient `TOOLCHAINS` and `SDKROOT`, and
   invoke Swift only through `/usr/bin/xcrun --sdk macosx --toolchain default` so a
   shadowing executable on `PATH` cannot recur.
2. Validate the selected tools by behavior: require an existing developer directory, a
   resolvable macOS SDK with major version 26 or newer, and a resolvable Apple `swift`
   executable. Accept both a full Xcode developer directory and
   `/Library/Developer/CommandLineTools` when those checks pass. Preserve forwarded
   arguments and the underlying Swift exit status.
3. Make failures specific to the selected tool source. For an absent, stale, or
   incompatible standalone package, direct the user to update/install Command Line Tools
   26+ or select a compatible full Xcode installation. For an incompatible Xcode
   selection, direct the user to select Xcode 26+. Do not tell users to install Xcode
   merely because the standard Command Line Tools path is selected.
4. Update the helper's usage text and `README.md` Requirements/Troubleshooting guidance
   to describe both supported Apple tool sources. Correct the XCTest explanation: the
   original failure comes from an incompatible or PATH-shadowing Swift toolchain; the
   remedy is to use a matching Apple toolchain with macOS SDK 26+, not necessarily the
   full Xcode app.

## Validation

1. Run `bash -n Scripts/xcode-swift.sh Scripts/bundle.sh Scripts/install.sh`,
   `just --list`, and `git diff --check` on the final tree.
2. On a macOS 26 host with Command Line Tools 26+ installed, select them explicitly for
   the process with `DEVELOPER_DIR=/Library/Developer/CommandLineTools`; verify
   `./Scripts/xcode-swift.sh --version`, `just build`, and `just test` succeed.
3. Repeat the Command Line Tools build and test with a deliberately failing fake `swift`
   first on `PATH`; verify no fake invocation occurs, proving the original shadowing
   issue remains fixed.
4. Run `just bundle` under the Command Line Tools selection and retain the existing
   `plutil -lint`, bundle-identifier, executable, and
   `codesign --verify --deep --strict` checks so accepting Command Line Tools does not
   weaken packaging.
5. If Xcode 26+ is also installed, run `just test` once with its `/Contents/Developer`
   selected to verify that existing full-Xcode support remains intact. If it is
   unavailable, report this optional compatibility check as not run, not as passed.
6. Exercise the failure path with an invalid developer directory and, where available, a
   pre-26 Apple toolchain; verify the helper exits before invoking Swift and gives a
   source-appropriate update/selection remedy.
7. Run `just all` with the compatible Command Line Tools selection as the final local
   end-to-end check, then confirm the existing macOS CI workflow passes after the change
   is published.

## Scope constraints

- Do not migrate tests from XCTest to Swift Testing or add a package dependency for
  XCTest.
- Do not trust or fall back to a bare `swift` from `PATH`.
- Do not lower the package's macOS 26 deployment target or the helper's macOS SDK 26
  minimum.
- Do not add an `xcodebuild`-based project or make the GUI Xcode application mandatory.
