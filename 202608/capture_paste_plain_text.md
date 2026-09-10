---
tier: tale
title: Strip formatting on paste in the Bob Mac Capture panel
goal:
  Pasting rich content into the capture editor inserts plain text immediately, with no
  multi-second stall and no formatting carried over from the source app.
size: medium
proposed_by: bbugyi200.athena.01g
create_time: 2026-09-09 19:59:54
status: wip
---

# Plan: Strip Formatting On Paste In The Bob Mac Capture Panel

## Repository

All source changes land in the **`bob-mac-capture` linked repo**. Open it first:

```bash
sase repo open bob-mac-capture -r "Implement plain-text paste in the capture panel"
```

Use the path that command prints as the only path for reads and writes. Every file path
below is relative to that checkout root.

## Symptom

Pasting content copied from Google Keep (Chrome) into the capture panel's editor stalls
for roughly 1–3 seconds before the text lands, and the pasted text arrives carrying the
source's formatting.

## Diagnosis

### What is already established

1. **The editor is the rich-text editor.**
   `Sources/BobMacCapture/CapturePanelView.swift:370` binds
   `TextEditor(text: $model.attributedDraft, selection: $selection)` — the macOS 26
   `AttributedString` editor. The adjacent `.textEditorStyle(.plain)` is visual chrome
   (no border/background); it does **not** make the editor plain-text.

2. **Nothing in the app intercepts paste.** `CaptureKeyCommandRouter` has no `v` case,
   and the Edit menu's Paste item (`Sources/BobMacCapture/AppDelegate.swift:141`) sends
   the stock `paste:` down the responder chain. So Cmd-V is AppKit's native rich paste
   on the backing `NSTextView`, which reads the _richest_ flavor the pasteboard offers
   (`com.apple.webarchive` / `public.rtf` / `public.html`) in preference to
   `public.utf8-plain-text`.

3. **The preserved formatting is itself the proof that the rich flavor is read.** A
   plain-text read cannot preserve formatting. So the rich-import path is definitely
   live.

4. **The competing explanation is quantitatively eliminated.** The other candidate was
   the post-edit analysis cascade: every text change runs
   `CapturePanelModel.scheduleAnalysis`, which spawns `bob capture-parse`, then
   `bob capture --dry-run --no-clip`, and sometimes `bob capture-complete`. Measured
   against the live vault on athena:

   | Command                                                  | Wall clock                                 |
   | -------------------------------------------------------- | ------------------------------------------ |
   | `bob capture-parse --format json -- "..."`               | 3–4 ms                                     |
   | `bob capture --dry-run --no-clip --format json -- "..."` | 3 ms warm, 24 ms cold                      |
   | `bob capture-complete --cursor 5 --format json -- "..."` | 3 ms                                       |
   | `bob capture-targets --format json`                      | 224 ms (cached; **not** on the paste path) |

   `bob-cli` is a Rust binary, so process start is ~3 ms, not interpreter-startup slow.
   The whole post-paste cascade is ~10 ms on top of the 50 ms debounce — two to three
   orders of magnitude short of the reported 1–3 s. This explanation is dead.

### Conclusion

The suspicion is correct in mechanism: the app ingests the rich pasteboard flavor, and
that ingest is the only step on the paste path with a plausible multi-second cost.
Chrome publishes `public.html` (and frequently `com.apple.webarchive`) for a Keep copy,
and AppKit imports both through the WebKit-backed HTML reader, synchronously on the main
thread — the well-known hundreds-of-milliseconds-to-seconds path. RTF import, by
contrast, is milliseconds.

A secondary amplifier rides along and is worth naming because the same fix removes it: a
rich paste yields an `AttributedString` with many attribute runs, which then feeds
SwiftUI text measurement (`AutosizingCaptureEditor.sizingText`), the editor's own
rendering, and `CapturePanelModel.applyHighlighting`'s whole-string `transform` on the
main actor. A plain-text paste collapses that to a single run.

The fix is also robust to the one hypothesis that cannot be fully eliminated from Linux
— that some of the cost is Chrome _lazily rendering_ a promised rich flavor on demand (a
synchronous IPC round trip, made worse by App Nap in the source app). Reading only
`public.utf8-plain-text`, which Chrome writes eagerly, avoids that too.

## Step 1 — Confirm on the Mac before writing code (~2 minutes, no build)

This is a gate. Run it on the Mac, record both results, and only then implement.

1. Copy a note from Google Keep, then dump the pasteboard flavors:

   ```bash
   osascript -e 'clipboard info'
   ```

   Expect `«class HTML»`, `«class RTF »`, and/or a webarchive entry alongside `string`,
   with the rich entries far larger than the plain one. Record the exact list and sizes.

2. **A/B control.** With the same Keep content still on the clipboard, strip it to plain
   text in place and paste the _identical characters_ again:

   ```bash
   pbpaste | pbcopy   # rewrites the clipboard as public.utf8-plain-text only
   ```

   Then Cmd-V into the capture panel.
   - **Plain-text paste is instant, original was 1–3 s** → root cause confirmed.
     Character count and the downstream `bob` analysis are exonerated by construction,
     because they are identical across the two pastes. Proceed.
   - **Plain-text paste is also slow** → the root cause is _not_ flavor conversion.
     Stop, report the measurement, and do not ship the rest of this plan as a fix; it
     would still be a correct change but not the cure. Re-plan from the new evidence.

Record both results in the commit message so the diagnosis is durable.

## Step 2 — Implement plain-text paste

### Design decision: retarget the Edit menu item, do not add a key-router case

The main menu is what delivers Cmd-V to the capture panel today — the existing
regression test `testMainMenuExposesStandardEditSelectorsAndQuit` says so explicitly
("without an explicit main menu, the capture editor silently loses Cmd-X/C/V/A/Z"). The
panel's local `NSEvent` monitor has no `v` case, so it already passes Cmd-V through
untouched.

Therefore retargeting the Edit menu's Paste item is **sufficient and single-path**. It
also covers the Settings window's text fields for free, and it sidesteps any question
about whether a local event monitor or a menu key equivalent sees Cmd-V first. Do
**not** add a `.pasteAsPlainText` case to `CaptureKeyCommandRouter`: it would be a
redundant second path to keep in sync, and quite possibly dead code.

### 2a. New file: `Sources/BobMacCapture/PlainTextPaste.swift`

An enum namespace with a testable seam:

```swift
enum PlainTextPaste {
    /// The clipboard's plain-text flavor, newline-normalized. `nil` when the pasteboard
    /// offers no plain text, so the caller can fall back to AppKit's native paste.
    static func plainText(from pasteboard: NSPasteboard) -> String?

    /// Pure: CRLF and lone CR to LF.
    static func normalized(_ raw: String) -> String

    /// Inserts the pasteboard's plain text into `responder` when it is an editable
    /// `NSTextView`. Returns `false` — inserting nothing — when the responder is not an
    /// editable text view or the pasteboard has no plain text.
    @discardableResult
    static func insert(
        into responder: NSResponder?,
        from pasteboard: NSPasteboard,
        willInsert: () -> Void = {}
    ) -> Bool
}
```

Requirements:

- `plainText(from:)` reads `NSPasteboard.PasteboardType.string` only. If that is absent
  it returns `nil`. It must **never** fall back to reading `public.html` or
  `public.rtf`, which would insert markup source as literal text. Declining lets the
  caller fall through to native paste, which preserves today's behavior for exotic
  pasteboards (image-only, file promises) at the cost of the slow path only in those
  rare cases.
- `normalized(_:)` maps `\r\n` → `\n` and lone `\r` → `\n`. This is not cosmetic: the
  capture grammar is physical-line-aware (see bob-cli commit "make capture text
  physical-line-aware"), so a CR-carrying paste would be parsed wrong.
- `insert(into:from:willInsert:)` resolves the responder to an editable `NSTextView`,
  calls `willInsert()`, then
  `textView.insertText(text, replacementRange: textView.selectedRange())`. Use
  `insertText(_:replacementRange:)` — the same call the existing Ctrl-J and Backspace
  helpers use — so undo coalescing, IME, and accessibility stay native and the model's
  `onChange` text observation fires exactly as it does for typing.
- Treat an empty plain-text flavor as "decline" (return `false`).
- Emit `CaptureSignpost.event("paste-plain-text")` on a successful insert. Event name
  only, no payload — the README's Privacy contract forbids ever putting draft text in a
  signpost.

Reuse the responder→`NSTextView` resolution rather than duplicating it: widen
`CapturePanelController.editableTextView(_:)` from `private static` to internal `static`
and call it from `PlainTextPaste`.

### 2b. `Sources/BobMacCapture/AppDelegate.swift`

- In `makeEditMenu()`, change the Paste item's action from `Selector(("paste:"))` to
  `#selector(AppDelegate.pastePlainText(_:))`, keeping `keyEquivalent: "v"` and
  `target: nil`. Target-nil resolution walks the key window's responder chain and ends
  at `NSApp`'s delegate, so the delegate receives it; the `NSTextView` does not respond
  to this selector and will not intercept it. Update the existing comment above
  `makeEditMenu()`, which currently explains that _every_ item is a raw first-responder
  selector — that is no longer true of Paste, and the comment should say why.

- Add the handler:

  ```swift
  @objc func pastePlainText(_ sender: Any?) {
      let inserted = PlainTextPaste.insert(
          into: NSApp.keyWindow?.firstResponder,
          from: .general,
          willInsert: { [weak self] in self?.panelModel?.dismissCompletion() }
      )
      guard !inserted else { return }
      NSApp.sendAction(Selector(("paste:")), to: nil, from: sender)
  }
  ```

  Dismissing completion before the insert mirrors the existing Ctrl-J and Backspace
  helpers. `panelModel` is optional and nil before `applicationDidFinishLaunching`; the
  optional chain above handles that, and a Settings-window paste harmlessly dismisses a
  completion list that is not on screen.

- Add `validateMenuItem(_:)` to `AppDelegate` so the Paste item still greys out when
  there is nothing to paste — the behavior the stock `paste:` gave for free. It must
  return `true` for every selector _except_ `pastePlainText(_:)`, which is enabled only
  when the key window's first responder is an editable `NSTextView` **and** the general
  pasteboard offers plain text. Getting the default wrong here would disable unrelated
  menu items, so write it as an explicit allow-everything-else default.

### 2c. Deliberately not done here

Do **not** reach for `AttributedTextFormattingDefinition` as the fix. It constrains
which attributes may survive an edit, but it runs _after_ the expensive HTML import has
already happened — it is a correctness tool, not a latency fix, and adding it here would
obscure which change actually bought the speedup.

Drag-and-drop into the editor still takes AppKit's native rich path. Leave it alone in
this change; it is listed under Follow-ups.

## Step 3 — Tests

All in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` unless the file grows
unwieldy, in which case a new `PlainTextPasteTests.swift` in the same target is fine.
Match the existing style: `@MainActor` test methods, headless `NSTextView`, a comment on
each test naming the regression it guards.

1. **`normalized(_:)` purity tests** — CRLF, lone CR, mixed CR/CRLF/LF, an LF-only
   string (unchanged), and the empty string.

2. **Flavor selection.** Build a scratch pasteboard with `NSPasteboard.withUniqueName()`
   (and `releaseGlobally()` in teardown — never touch `NSPasteboard.general` in tests).
   Write `.html`, `.rtf`, and `.string` flavors where the rich flavors carry markup and
   the plain flavor carries known text. Assert `plainText(from:)` returns exactly the
   plain text and never markup source.

3. **Declines a rich-only pasteboard.** Same scratch pasteboard with `.html`/`.rtf` but
   no `.string`: assert `plainText(from:)` is `nil` and `insert(...)` returns `false`
   and leaves the text view untouched — this is the fall-through-to-native-paste
   contract.

4. **Insertion behavior.** Editable headless `NSTextView` with a known string and a
   collapsed selection mid-string, plus a scratch pasteboard carrying rich _and_ plain
   flavors. Assert `insert(...)` returns `true`, `textView.string` is the plain text
   spliced at the selection, and `willInsert` ran. Then assert the paste introduced
   **no** formatting: no `.link` attribute anywhere, and `.font` and `.foregroundColor`
   uniform across the whole string (a single effective run) rather than differing over
   the inserted range. Do not assert against a hardcoded font — assert _uniformity_,
   since `NSTextView` supplies its own default typing attributes.

5. **Declines unrelated and non-editable responders.** `nil` responder, an `NSButton`,
   and a non-editable `NSTextView` each return `false` — mirroring
   `testInsertNewlineDeclinesUnrelatedResponder`.

6. **Update `testMainMenuExposesStandardEditSelectorsAndQuit`.** Its `editSelectors`
   expectation becomes
   `["undo:", "redo:", nil, "cut:", "copy:", "pastePlainText:", "selectAll:"]`. Leave
   the `keyEquivalent` expectations unchanged. Extend the test's existing comment to say
   the deviation from the stock `paste:` is deliberate, and why — otherwise the next
   reader will "fix" it back.

7. **Menu validation.** Assert `validateMenuItem(_:)` returns `true` for a menu item
   carrying some other selector, so the allow-everything-else default is pinned.

## Step 4 — Documentation

`README.md`:

- **Keyboard section**: add a row for Cmd-V — inserts the clipboard's plain text; source
  formatting is discarded deliberately. State both reasons: the capture grammar is plain
  text, and reading a rich flavor forced a synchronous WebKit HTML import that cost
  seconds per paste.
- **Troubleshooting section**: add a "pasting feels slow" entry pointing at the Step 1
  procedure (`osascript -e 'clipboard info'` and the `pbpaste | pbcopy` A/B), so the
  next person can re-run the same discriminator.
- **Diagnostics and Signposts section**: add `paste-plain-text` to the list of emitted
  events.

## Step 5 — Verification

**The implementing agent runs on Linux and cannot build or run the `BobMacCapture`
target.** AppKit and SwiftUI are macOS-only; only `CaptureCore` compiles on Linux. Do
not run `just build` / `just test` locally, and do not report their failure as a
regression — that is expected and is not evidence about this change.

Verification is therefore:

1. `swift-format lint --recursive Package.swift Sources Tests` if a `swift-format` is
   available locally. Formatting only; skip without ceremony if unavailable.
2. Push and let the `macos-26` GitHub Actions workflow run lint, build, test, bundle,
   signature verification, and the launch smoke test. **CI is the build gate for this
   change.** Do not declare the work done on a red or unrun CI.
3. Manual confirmation on the Mac, after `just bundle && just install`:
   - Copy from Google Keep, Cmd-V into the panel: text lands immediately and carries no
     formatting.
   - Cmd-Z undoes the paste as a single action.
   - Edit → Paste from the menu behaves identically to Cmd-V.
   - Settings → executable override field still pastes.
   - Paste while a completion list is visible: the list dismisses, and analysis re-runs
     on the new text.
   - Re-run the Step 1 A/B and record the before/after latency in the commit message.

## Out Of Scope / Follow-ups

File these with `/sase_new_task` if they still apply after implementation:

- **Non-breaking space in pasted text.** Web copies routinely contain U+00A0. Because it
  is not an ASCII space, it can silently break marker parsing (`@route`, `s:<N>`,
  `p:<N>`) — the capture would look right and route wrong. This change deliberately does
  not normalize it; decide separately whether pasted (or all) input should fold NBSP to
  a plain space, since that is a capture-grammar decision spanning bob-cli and
  bob-mac-capture.
- **Drag-and-drop into the editor** still uses AppKit's native rich path and keeps both
  the formatting and the latency.
- **Very large pastes** still trigger a full `bob capture-parse` + dry-run cascade with
  no size ceiling. Cheap today (~10 ms), but unbounded in principle.
