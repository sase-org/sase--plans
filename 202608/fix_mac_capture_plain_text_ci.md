---
tier: tale
title: Fix bob-mac-capture plain-text paste CI failure
goal:
  GitHub Actions passes while Command-V reads only explicitly advertised plain text and
  preserves rich-only fallback.
size: small
proposed_by: bbugyi200.athena.01s
create_time: 2026-09-09 20:00:06
status: wip
---

# Fix bob-mac-capture plain-text paste CI failure

## Objective

Restore the `bob-mac-capture` GitHub Actions `macOS 26 SwiftPM` job while preserving the
intended Command-V behavior: use a pasteboard's explicitly advertised plain-text flavor
without triggering AppKit conversion of HTML or RTF, and leave rich-only pasteboards to
the existing native-paste fallback.

## Diagnosed root cause

GitHub Actions run `31824335272` fails only in the `Test` step. The failing regression
test creates an `NSPasteboard` containing HTML and RTF but no `.string` type. On macOS
26, `NSPasteboard.string(forType: .string)` synthesizes `"Rich capture text"` from those
rich flavors even though `.string` is absent from the pasteboard's advertised types.
`PlainTextPaste.plainText(from:)` currently calls that converting accessor directly, so
`PlainTextPaste.insert` consumes the paste, invokes its insertion callback, and mutates
the text view. That violates the helper's plain-text-only contract and produces the four
assertions reported by `PlainTextPasteTests`.

## Implementation

1. Open the linked `bob-mac-capture` repository through `sase repo open` and confirm
   that the checkout still points at the failing change and has no unrelated local
   edits.
2. Update `Sources/BobMacCapture/PlainTextPaste.swift` so `plainText(from:)` first
   verifies that the pasteboard explicitly advertises `.string` in `pasteboard.types`,
   then reads and normalizes that value. Do not ask AppKit for `.string` when only rich
   types are advertised, because that request performs the unwanted rich-to-plain
   conversion.
3. Keep the existing behavior for an advertised nonempty `.string`, newline
   normalization, responder validation, insertion callbacks, and native fallback. If
   needed for clarity, minimally strengthen the existing rich-only regression test to
   assert the distinction between advertised types and AppKit's synthesized string;
   avoid unrelated paste or menu changes.

## Validation

1. Run the repository's Swift formatting lint over `Package.swift`, `Sources`, and
   `Tests` using the same command path as CI.
2. On macOS 26, run the focused `PlainTextPasteTests` suite and confirm the rich-only
   case returns `nil`, does not invoke `willInsert`, and does not mutate the text view,
   while rich-plus-plain insertion still succeeds.
3. Run the full CI-equivalent local sequence: SwiftPM build, all tests, bundle creation,
   plist/signature verification, and the repository's remaining smoke/install checks
   where the local environment supports them.
4. Re-run the relevant GitHub Actions workflow after the fix is published, use `actstat`
   plus the Actions job log to verify `macOS 26 SwiftPM` passes end to end, and report
   any validation that cannot be performed from a non-macOS checkout.

## Scope and risk

The change is intentionally limited to the pasteboard type gate and its regression
coverage. The main risk is accidentally rejecting a genuine plain-text paste or changing
fallback behavior; the paired rich-plus-plain and rich-only tests cover those two
boundaries. No bob-cli capture grammar or cross-repository contract change is required.
