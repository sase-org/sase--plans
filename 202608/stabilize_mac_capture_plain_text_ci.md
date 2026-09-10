---
tier: tale
title: Stabilize bob-mac-capture plain-text paste CI coverage
goal:
  Restore bob-mac-capture CI with deterministic coverage of rich-only pasteboard
  rejection while preserving plain-text paste behavior.
size: small
proposed_by: bbugyi200.athena.021
create_time: 2026-09-09 20:00:30
status: wip
---

# Stabilize bob-mac-capture plain-text paste CI coverage

## Objective

Restore the `bobs-org/bob-mac-capture` GitHub Actions workflow while preserving the
capture editor's contract: Command-V consumes an explicitly available nonempty
plain-text flavor, normalizes its newlines, and otherwise leaves the responder untouched
so AppKit can decide whether a native fallback is possible.

## Diagnosis

`actstat --repo bobs-org/bob-mac-capture -n 5 --format json` identifies CI run
`31835093819`, job `macOS 26 SwiftPM`, step `Test`, as the current failure on commit
`9880af5`. The failed log reports five assertions, all cascading from
`testPlainTextDeclinesRichOnlyPasteboardAndLeavesTextViewUntouched`:

- the scratch `NSPasteboard` unexpectedly reports `.string` after HTML and RTF are
  written;
- `string(forType: .string)` returns AppKit's synthesized `"Rich capture text"`;
- `PlainTextPaste` consequently accepts and inserts that value.

The preceding attempted fix assumed `NSPasteboard.types` would distinguish an explicitly
advertised string flavor from a synthesized one on every macOS 26 environment. The CI
runner disproves that assumption: the test fixture itself is OS-behavior-dependent, so
it cannot reliably model a rich-only source or verify the guard. The rest of the
161-test suite passes, as do formatting and build steps.

## Implementation

1. In `Sources/BobMacCapture/PlainTextPaste.swift`, introduce the smallest internal
   pasteboard-reading abstraction needed by the plain-text path: reported pasteboard
   types and string lookup by type. Make `NSPasteboard` conform and route
   `plainText(from:)` and `insert(into:from:willInsert:)` through that abstraction
   without changing application call sites or the plain-text/newline/insertion behavior.
2. In `Tests/BobMacCaptureTests/PlainTextPasteTests.swift`, use a deterministic test
   reader for the rich-only case. Model rich types without `.string` while allowing the
   fake accessor to return a convertible rich-text string, and assert that the guard
   declines it before insertion. Keep the real scratch-pasteboard tests that cover a
   genuine `.string` flavor, text insertion, formatting preservation, and responder
   rejection.
3. Update comments and test names only where necessary so they describe reported type
   availability rather than claiming a universal AppKit relationship between `types` and
   synthesized string conversion.

## Validation

Run validation from the linked `bob-mac-capture` checkout, using the same repository
scripts as GitHub Actions:

1. `./Scripts/xcode-swift.sh test --filter PlainTextPasteTests`
2. `./Scripts/xcode-swift.sh format lint --recursive Package.swift Sources Tests`
3. `./Scripts/xcode-swift.sh build`
4. `./Scripts/xcode-swift.sh test`
5. `./Scripts/bundle.sh --identity "-"`

If local execution cannot provide macOS 26/AppKit, record that environmental limitation
and rely on the deterministic unit boundary for the regression; do not weaken the
production contract or skip the full commands on an available macOS host. Re-run the
focused test after any formatting or implementation adjustment, inspect the final diff,
and confirm only the intended production and test files changed.

## Acceptance criteria

- A reported `.string` flavor is read, normalized, and inserted exactly as before.
- A reader without `.string` is rejected even if requesting `.string` could synthesize a
  value; the text view and `willInsert` callback remain untouched.
- Regression coverage no longer depends on which implicit types a particular macOS 26
  pasteboard server adds to an HTML/RTF test fixture.
- Formatting, build, focused tests, the full SwiftPM test suite, and bundling pass in
  the CI-compatible macOS environment.
