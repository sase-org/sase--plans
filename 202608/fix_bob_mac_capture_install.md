---
tier: tale
title: Fix Bob Mac Capture installer path capture
goal:
  Make just install robust to Swift build output while preserving verified atomic
  installation.
size: small
proposed_by: bbugyi200.athena.00c.f0
create_time: 2026-09-09 20:00:04
status: wip
---

# Fix Bob Mac Capture installer path capture

## Objective

Make `just install` reliably install the signed `Bob Mac Capture.app` bundle when
SwiftPM emits build progress on standard output, while preserving the installer's
signature checks, bundle-identifier check, atomic replacement, and rollback behavior.

## Repository and confirmed root cause

The implementation target is the separate `bobs-org/bob-mac-capture` repository. Open it
through `sase repo open gh:bobs-org/bob-mac-capture` before reading or editing it.

`just install` delegates to `Scripts/install.sh`. That script currently runs
`Scripts/bundle.sh` in command substitution and assigns all captured standard output to
`staged_app`. The bundler's `swift build` command writes progress and compiler warnings
to standard output, then `bundle.sh` prints the staged app path. Consequently,
`staged_app` becomes a multiline string beginning with `Building for production...`
rather than one filesystem path. `cp -R "${staged_app}" ...` treats the whole string as
one source pathname and macOS rejects it with `File name too long`. The Swift build,
link, and code-sign steps have already succeeded; the reported Swift concurrency
warnings are unrelated to this failure.

## Implementation

1. In `Scripts/install.sh`, remove the installer's dependence on parsing the bundler's
   human-readable/build output. Invoke `Scripts/bundle.sh` normally for its side effect,
   keep its diagnostics visible, and derive `staged_app` from the already explicit
   `bundle_root` plus `Bob Mac Capture.app`. Preserve a clean stdout contract for
   `install.sh` by directing nested bundler output to stderr if necessary, leaving the
   successfully installed path as the installer's sole stdout result. Let the existing
   `set -e` behavior stop installation if bundling fails, before the installed app is
   touched.
2. Extend `.github/workflows/ci.yml` with an installer integration check on the macOS
   runner. Give the script a temporary `HOME`, pass the literal `~/Applications` target
   used by the default `just install` recipe, and assert that it installs exactly
   `Bob Mac Capture.app` beneath that temporary home. Capture and check the installer's
   stdout so future build chatter cannot silently become either a source path or part of
   the result contract. Verify the installed bundle's signature, plist, and bundle
   identifier, and clean up the temporary home.
3. Keep the change scoped to packaging/install behavior. Do not alter Swift source to
   address the non-fatal Swift 6 warnings as part of this fix.

## Validation

- Run `bash -n Scripts/bundle.sh Scripts/install.sh`.
- Run the normal Swift checks: formatting lint, `swift build`, and `swift test`.
- On macOS, run the new temporary-`HOME` install integration check with the ad-hoc
  identity `-`; confirm the command exits successfully, stdout is exactly the final
  install path, the app exists, `plutil -lint` passes,
  `codesign --verify --deep --strict` passes, and `CFBundleIdentifier` is
  `org.bobs.bob-mac-capture`.
- Re-run installation over the first temporary install to exercise the existing
  backup/swap path and revalidate the final bundle, ensuring the path-handling fix does
  not weaken replacement or rollback mechanics.

## Acceptance criteria

- `just install` no longer passes Swift build output to `cp` as a pathname and succeeds
  for the default `~/Applications` target with ad-hoc signing.
- A failed bundle build still exits before changing an existing installation.
- Successful installation emits only the installed app path on stdout while build and
  signing diagnostics remain visible on stderr.
- CI exercises the real installer, including a reinstall, and verifies the resulting
  bundle identity and signature.
- No Swift concurrency-warning cleanup or unrelated application behavior is included.
