---
tier: tale
title: Keep capture completions visible without obscuring the draft
goal:
  Completion suggestions remain usable without covering typed text or reopening after
  acceptance.
size: medium
proposed_by: bbugyi200.athena.00l
create_time: 2026-09-09 19:59:57
status: wip
---

# Keep capture completions visible without obscuring the draft

## Objective

Fix the macOS capture panel so a large completion result never covers the text being
edited and accepting a suggestion with Return closes the completion list instead of
immediately reopening it. Preserve the existing completion replacement semantics,
continuous parse/highlight and live-preview updates, keyboard navigation, mouse
acceptance, and the documented meaning of plain Return while a completion is visible.

## Root cause

- `CapturePanelView` places `CompletionList` in the same top-leading `ZStack` as the
  `TextEditor`. The list has no height bound or scrolling container, so a sufficiently
  large candidate set establishes a tall stack and then paints its opaque material over
  the editor. The screenshot's visible preview row beneath an otherwise hidden editor is
  the direct result of that overlay layout.
- `CapturePanelModel.acceptSelectedCompletion()` clears `completionResponse`, replaces
  the server-provided byte range, and immediately schedules the ordinary analysis path.
  That path is allowed to request completions again. At the accepted token's end,
  `shouldRequestCompletion` still considers the cursor inside the completion span, so a
  cached or server response repopulates `completionResponse` after the debounce. The
  current test uses no process client and asserts synchronously, before this
  asynchronous reopen can occur.

## Implementation

1. In `Sources/BobMacCapture/CapturePanelView.swift`, retain a small `ZStack` only for
   the editor and its empty-draft placeholder. Move `CompletionList` into normal
   vertical layout immediately below the editor so it reserves its own space and cannot
   cover the draft. Give the candidate region a sensible maximum height and make it
   scroll for large responses; keep the selected candidate visible as keyboard
   navigation changes the index. Preserve row details, click acceptance, selection
   styling, and accessibility containment.
2. In `Sources/BobMacCapture/CapturePanelModel.swift`, distinguish the programmatic
   draft replacement caused by accepting a completion from a genuine subsequent editor
   edit. Invalidate/cancel stale analysis, clear the visible response, and run parse,
   highlighting, and live preview for the accepted draft without issuing a completion
   request for that same programmatic edit. Ensure any delayed SwiftUI text-change
   callback cannot accidentally lift that suppression. Clear the suppression on the next
   actual text change so completion remains responsive as soon as the user types again.
   Route Return, Tab, and mouse acceptance through this shared behavior without
   weakening UTF-8 replacement-range validation.
3. Extend the fake `bob` fixture and `CapturePanelModel` tests with a parse/completion
   case that still qualifies for completion at the accepted token's inclusive end.
   Exercise the debounced asynchronous path to prove the accepted text is applied, the
   list stays dismissed after analysis settles, parse/live preview still refresh, and a
   subsequent user edit can show completions again. Retain the existing key-router
   assertions so plain Return continues to mean “accept completion” when the list is
   visible.

## Validation

1. Run `swift format lint --recursive Package.swift Sources Tests` (or
   `swift-format lint --recursive Package.swift Sources Tests` when that executable is
   the repository's available formatter).
2. Run `swift build` and `swift test`.
3. Bundle and launch the app using the repository's documented development workflow.
   Trigger a route completion with enough candidates to exceed the bounded list height,
   and verify the typed draft remains fully visible, the list scrolls,
   arrow/Ctrl-N/Ctrl-P navigation keeps the selected row visible, and mouse selection
   still works.
4. Press Return on a selected candidate and wait beyond the completion debounce. Verify
   the replacement is inserted exactly once, the completion list remains absent while
   highlighting and live preview update, and typing another completion-relevant
   character allows suggestions to appear again. Also smoke-test Escape dismissal and
   Tab acceptance for regressions.

## Acceptance criteria

- Completion candidates never visually overlap or intercept interaction with the capture
  editor, including when the response contains more rows than fit in the panel.
- Accepting a completion dismisses it durably for the accepted programmatic edit; stale,
  cached, or debounced work cannot immediately repopulate the list.
- Parse highlighting and live preview still refresh after acceptance, and completions
  are eligible again on the next real user edit.
- Server-provided UTF-8 replacement ranges, keyboard selection/navigation, mouse
  acceptance, and accessibility behavior remain intact.
- Formatting, build, tests, and the focused installed-panel smoke checks pass.
