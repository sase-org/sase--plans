---
tier: tale
title: Restore bob-mac-capture GitHub Actions CI
goal:
  The macOS 26 SwiftPM workflow builds, tests, bundles, and verifies bob-mac-capture
  successfully.
size: small
proposed_by: bbugyi200.athena.006
create_time: 2026-09-09 20:00:25
status: wip
---

# Restore bob-mac-capture GitHub Actions CI

## Context and root cause

`actstat` identifies the current failure as GitHub Actions run `31765412374`, job
`macOS 26 SwiftPM`, step `Build`, for `master` commit `88cc781`. The Swift compiler
rejects `captureLivePreview` because it asks the generic schema-versioned decoder to
decode `CaptureCommandResponse`, which intentionally does not conform to
`SchemaVersioned`: `bob capture --format json` emits an `ok`-discriminated response
without a `schema_version` field. The immediately preceding run (`31763705729`) failed
for the same reason, so this is a deterministic source defect rather than a flaky runner
failure. Older actor-isolation and `OSStatus` failures occurred on earlier commits and
are not present in the current build log.

The nearby general `capture` method already establishes the intended contract by routing
schema-less capture JSON through `decodeCaptureResult`. The existing
`testLivePreviewAlwaysUsesNoClipAndPrioritySeed` test also exercises a schema-less
successful live-preview response and its required arguments/environment.

## Implementation

1. Update `BobProcessClient.captureLivePreview` to use `decodeCaptureResult` instead of
   the `decode(_:expectedSchema:...)` helper. Preserve the `preview` process lane,
   `BOB_PRIORITY_ROLL_SEED` override, dry-run/no-clipboard argument precondition, and
   existing capture response semantics. Do not add synthetic schema-version conformance
   to `CaptureCommandResponse`, because that would contradict the real `bob capture`
   JSON contract.
2. Keep the change narrowly scoped. Use the existing live-preview process-client test as
   the regression test; strengthen it only if the current fixture does not prove that
   schema-less capture JSON is accepted through the live-preview path.

## Verification

1. Review the focused diff and run `git diff --check`.
2. On a macOS 26/Swift 6-capable environment, mirror the CI sequence until all stages
   pass: Swift formatting lint, `swift build`, `swift test`, ad-hoc bundle assembly,
   plist lint, bundle identifier assertion, and strict code-signature verification.
3. Confirm specifically that the live-preview test passes and that the compiler no
   longer reports a `SchemaVersioned` requirement for `CaptureCommandResponse`.
4. Recheck the resulting GitHub Actions run when the change is published; if a later
   stage previously hidden by the build failure surfaces, diagnose it from its exact log
   and iterate within this same focused repair until the full `macOS 26 SwiftPM` job is
   green.

The current workspace host is Linux and has no Swift toolchain, so macOS-only build,
test, bundle, plist, and codesign verification cannot be claimed locally. Any handoff
must distinguish the locally completed diff checks from verification performed by the
macOS GitHub Actions runner.
