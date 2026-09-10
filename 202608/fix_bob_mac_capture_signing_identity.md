---
tier: tale
title: Make bob-mac-capture signing failures actionable
goal:
  Local installation uses a real available signing identity or gives precise, tested
  recovery guidance.
size: small
proposed_by: bbugyi200.athena.006.f0
create_time: 2026-09-09 20:00:05
status: wip
---

# Make bob-mac-capture signing failures actionable

## Objective

Make local `bob-mac-capture` installation fail with precise recovery guidance when the
requested code-signing identity is unavailable, and correct the installation examples so
users supply a real keychain identity instead of copying the documentation placeholder.

## Diagnosis

The release build completed and the first fatal line came from `codesign`:

```text
Apple Development: Name (bob-mac-capture): no identity found
```

`just` correctly forwards its quoted `identity` argument through `Scripts/install.sh` to
`Scripts/bundle.sh`, and `bundle.sh` passes it unchanged to `codesign --sign`. The
literal value in the failing invocation is derived from the README template
`Apple Development: Name (TEAMID)`: `Name` is still a placeholder, and `bob-mac-capture`
is an application/repository name rather than an Apple Developer team identifier.
Consequently, the selector does not match a usable identity in the Mac's keychains. The
available output does not establish whether the Mac has another valid identity; that
must be determined with `security find-identity -p codesigning -v`.

The Swift concurrency diagnostics are warnings under the current language mode. They did
not fail the build and are not the cause of this installation failure. The duplicate
failure occurs because the documented `just install` path invokes `bundle.sh` itself, so
running `just bundle` first performs the same unsuccessful signing work twice.

## Implementation

1. In `Scripts/bundle.sh`, route both signing invocations through one small helper that
   preserves the original `codesign` output and nonzero status. When `codesign` reports
   `no identity found` for a non-ad-hoc identity, append focused guidance that:
   - echoes the rejected selector safely;
   - tells the user to run `security find-identity -p codesigning -v`;
   - explains that they must pass an identity name shown by that command or its SHA-1;
   - points users with no valid identity to Xcode's certificate management; and
   - gives `--identity "-"` as the explicit local ad-hoc alternative.

   Do not auto-select a certificate, silently fall back to ad-hoc signing, or pre-reject
   identities by parsing `security` output. `codesign` is the authority for identity
   resolution and can search identity stores that `security find-identity` does not
   enumerate. For other signing errors, retain the original failure without misleading
   “identity missing” advice.

2. In `README.md`, replace the copy-pasteable placeholder workflow with an explicit
   identity-discovery step and clearly marked example output. Explain that the name/team
   values come from the signing certificate—not from the app or repository name—and that
   a SHA-1 is preferable when names are duplicated. Separate the bundle-only example
   from installation, state that `just install` builds its own staged bundle, and
   document the ad-hoc `just install ~/Applications` fallback plus its permission/trust
   tradeoff. Add a troubleshooting entry for `no identity found`, including the “no
   valid identities” certificate-creation case.

3. In `.github/workflows/ci.yml`, add a macOS shell regression check that invokes the
   bundle script with a deliberately nonexistent named identity, asserts a nonzero exit,
   and verifies that the output retains the native `no identity found` message and adds
   the discovery and ad-hoc recovery commands. Keep the existing ad-hoc bundle and
   signature-verification steps as the success-path coverage.

## Validation

1. Run `bash -n Scripts/bundle.sh Scripts/install.sh` and the repository formatting
   lint.
2. On macOS, exercise the new negative-path check with a guaranteed-missing identity and
   confirm it fails without installing or replacing an existing app, preserves the
   original `codesign` diagnostic, and prints the exact recovery guidance.
3. Run `security find-identity -p codesigning -v` on the affected Mac:
   - if it lists a valid identity, rerun
     `just install ~/Applications "<exact name or SHA-1>"` and verify the installed app
     with `codesign --verify --deep --strict` and `codesign -dvv`;
   - if it lists no valid identities, create/download an Apple Development certificate
     with its private key in Xcode and repeat, or deliberately use
     `just install ~/Applications` for an ad-hoc local install.
4. Run `swift build`, `swift test`, and `./Scripts/bundle.sh --identity "-"`; then lint
   the generated plist, verify the ad-hoc bundle signature, and confirm its bundle
   identifier remains `org.bobs.bob-mac-capture`.
5. Run the complete GitHub Actions workflow on macOS 26 to cover both the new failure
   diagnostic and the existing successful ad-hoc bundle path.

## Out of scope

Do not change Swift concurrency annotations or otherwise address the warnings shown
before the successful link. Do not change the default ad-hoc identity, certificate
selection policy, bundle identifier, install target restrictions, or atomic replacement
behavior.
