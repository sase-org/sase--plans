---
tier: tale
title: Focus the Add block ID field when @file+ selects a task without a block ID
goal:
  Accepting a "Needs block ID" task from `@route+` completion moves keyboard focus into
  the Add block ID field in the same interaction, so the user can type the ID
  immediately without clicking the field first.
size: medium
proposed_by: bbugyi200.athena.058
create_time: 2026-09-09 20:00:32
status: wip
---

# Add Block ID Prompt Must Take Keyboard Focus

## Symptom

In the Bob Mac Capture panel, typing `@file+`, selecting an Obsidian task from the
**Needs block ID** group, and pressing Return opens the inline **Add block ID** card —
but nothing in that card has keyboard focus. Typed characters go nowhere; the user must
click the `^` text field before the ID can be typed.

This is a regression against two shipped contracts:

- The approved epic `202608/file_plus_any_task.md` specifies "a fixed `^` prefix beside
  a **focused** monospaced text field" and requires the flow to be "keyboard-complete"
  with "predictable first-responder/focus restoration".
- The `bob-mac-capture` `README.md` "Keyboard" section states plainly: "Every capture
  action is reachable from the keyboard alone; the hotkey, editor, completion list,
  Stash/Capture/Preview/Discard buttons, and stash picker never require a pointer."

## Where To Work

The defect is entirely in the linked `bob-mac-capture` repository. No `bob-cli` change
is needed: `bob capture-complete --all-tasks` already returns the missing-ID candidate
with `requires_block_id`, and `bob capture-task-id` already performs the write. Before
reading or editing anything, open the repo and use only the printed path:

```bash
sase repo open bob-mac-capture -r "Fix Add block ID prompt keyboard focus"
```

## Diagnosis

### The model layer is already correct

`CapturePanelModel` publishes a monotonic focus request and does raise one for this
exact transition:

- `CapturePanelModel.swift` — `requestFocus(_:)` bumps `focusSequence` and publishes
  `focusRequest = CapturePanelFocusRequest(sequence:target:)`.
- `CapturePanelModel.swift` —
  `presentTaskIDPrompt(candidate:draftSnapshot:replacementRange:)` sets `taskIDPrompt`,
  sets `statusText = "Add block ID"`, and calls `requestFocus(.taskIDPromptBlockID)`.
- `CapturePanelModelTests.testMissingTaskCompletionOpensTaskIDPromptWithoutChangingDraft`
  already asserts `model.focusRequest.target == .taskIDPromptBlockID` and that the
  sequence advanced.

So the request is raised, and the existing tests pass. The break is downstream, in how
`CapturePanelView` turns that request into actual SwiftUI focus.

### The break: a one-shot `@FocusState` assignment made in the presenting transaction

`CapturePanelView` mirrors the request into `@FocusState` with a single assignment:

```swift
.onAppear { applyFocusRequest(model.focusRequest) }
.onChange(of: model.focusRequest) { _, request in applyFocusRequest(request) }
...
private func applyFocusRequest(_ request: CapturePanelFocusRequest) {
    focusedControl = request.target
}
```

`acceptSelectedCompletion()` mutates `taskIDPrompt` and `focusRequest` in one
synchronous call, so SwiftUI coalesces them into a single update. Two independent things
go wrong in that update, and either one alone produces the reported symptom:

1. **The target control does not exist while the request is being applied.**
   `TaskIDPromptCard` — and therefore the `TextField` carrying
   `.focused(focus, equals: .taskIDPromptBlockID)` — is only inserted by the body
   evaluation triggered by `model.taskIDPromptVisible` becoming true. During that body
   evaluation `focusedControl` is still `.editor`, so the newly inserted field has no
   reason to claim focus. `onChange` runs after that update, assigning a focus value for
   a control SwiftUI has only just installed.

2. **The capture editor resigns first responder in the same transaction.**
   `AutosizingCaptureEditor`'s `TextEditor` carries
   `.disabled(model.isSubmitting || model.taskIDPromptVisible)` together with
   `.focused(focus, equals: .editor)`. The moment the prompt appears, the currently
   focused editor becomes disabled, so AppKit resigns its `NSTextView` first responder
   and SwiftUI writes the focus binding back to `nil`. That write-back races the
   `onChange` assignment; when it lands last, focus ends up nowhere — which is exactly
   what the user sees.

The two mechanisms are consistent with the rest of the observed behavior:

- The initial `.editor` focus from `CapturePanelController.show()` works, because the
  editor already exists and nothing is being disabled.
- Re-focus requests raised _while the card is already open_ (validation failure, Bob
  error, `presentStashPicker` refusal) are believed to work, because the field is
  already installed and no disable transition is in flight.
- Only the presentation transition is broken, which is what the report describes.

### Why nothing else masks it

While the prompt is open, `CaptureKeyCommandRouter.taskIDPromptCommand(for:modifiers:)`
returns `nil` for ordinary printable keys, so the local monitor passes them through to
AppKit. With no first responder that accepts text, those keystrokes are simply dropped —
there is no fallback path that could route typing into the field. Nothing recovers until
the user clicks.

## Fix

Two small, deterministic changes, one per mechanism. Do both: they are independent, and
neither can be validated from Linux, so the change should be maximally likely to be
correct on first install.

### 1. Let the control that owns a focus target claim the request itself

Replace the container's one-shot mirror with adoption at the control. Add a private view
modifier in `CapturePanelView.swift` and apply it to both focusable controls:

```swift
private struct CaptureFocusAdoption: ViewModifier {
    let target: CapturePanelFocusTarget
    let request: CapturePanelFocusRequest
    var focus: FocusState<CapturePanelFocusTarget?>.Binding

    func body(content: Content) -> some View {
        content
            .focused(focus, equals: target)
            // `.task(id:)` runs after this control is installed and after the update
            // that installed it commits, so it can claim a request published in the
            // same transaction that created the control, and can re-claim one that was
            // dropped when another control resigned first responder in that
            // transaction. `id:` re-runs it for every later request (validation error,
            // Bob failure) without re-stealing focus the control already holds.
            .task(id: request) {
                guard request.target == target, focus.wrappedValue != target else {
                    return
                }
                focus.wrappedValue = target
                CaptureSignpost.event("focus-adopted")
            }
    }
}
```

- Apply it to the `TextEditor` in `AutosizingCaptureEditor` in place of its current
  `.focused(focus, equals: .editor)`, and to the block-ID `TextField` in
  `TaskIDPromptCard` in place of its current
  `.focused(focus, equals: .taskIDPromptBlockID)`. Both views already observe `model`,
  so they can pass `model.focusRequest`.
- Keep `CapturePanelView`'s existing `onAppear` / `onChange(of: model.focusRequest)`
  assignment as the synchronous fast path. It stays correct for every request whose
  target is already installed, and the adopter only acts when it did not take.
- Emit the `focus-adopted` signpost event next to the existing `editor-focus-requested`
  event in `CapturePanelController.show()`, using `CaptureSignpost.event` from
  `Signposts.swift`. This is metadata-only and makes manual verification objective via
  `log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"'`.

`.task(id:)` takes a `@Sendable` closure, and the closure here captures a
`FocusState.Binding`. The targets build in `.swiftLanguageMode(.v5)`, so this is
expected to compile, but the local host cannot verify it. If the toolchain rejects or
warns on the capture, keep the same semantics with `.onAppear` plus
`.onChange(of: request)` that each hop through `Task { @MainActor in ... }` before
assigning — the requirement is only that the claim runs after the control is installed
and after the presenting update commits, not that it uses `.task` specifically.

**Rejected alternative:** a "heal focus back to the field whenever `focusedControl`
becomes `nil` while the prompt is open" rule. With Full Keyboard Access on, clicking or
tabbing to **Cancel** / **Add & Select** produces exactly that `nil` transition, so the
healer would repeatedly steal focus away from the card's buttons. Do not implement it.

### 2. Stop the editor from resigning first responder in the presenting transaction

Give the model an explicit, deferred input lock so the block-ID field wins first
responder before the editor is disabled:

- Add `@Published private(set) var editorInputLocked = false` to `CapturePanelModel`,
  plus a generation/identity guard so a stale deferred update can never set it after the
  prompt has been dismissed (mirror the discipline already used by
  `activeTaskIDRequestID`).
- `presentTaskIDPrompt` publishes the prompt and the focus request exactly as today,
  then schedules the lock one main-actor hop later (`Task { @MainActor in ... }`),
  applying it only if the same prompt is still open.
- Every path that clears `taskIDPrompt` — `cancelTaskIDPrompt`, the success branch of
  `completeTaskIDAssignment`, `resetAnalysisState`, `prepareForDismissal` — must clear
  `editorInputLocked` synchronously and invalidate any pending deferred set.
- In `AutosizingCaptureEditor`, change the editor to
  `.disabled(model.isSubmitting || model.editorInputLocked)`. `isSubmitting` keeps its
  current immediate behavior; only the prompt-driven lock is deferred.

The one-runloop-turn window where the prompt is visible and the editor is not yet locked
is not reachable by a human keystroke, and the draft-snapshot check in
`submitTaskIDPrompt` ("Draft changed. Return to the task list and choose again.")
remains the authoritative guard against a draft edited between selection and
confirmation.

## Implementation Steps

1. Open the linked repo with `sase repo open bob-mac-capture` and work only from the
   printed path.
2. `CapturePanelModel.swift`: add `editorInputLocked` with its deferred set and its
   synchronous clears on every prompt-clearing path, guarded against stale application.
3. `CapturePanelView.swift`: add the `CaptureFocusAdoption` modifier, apply it to the
   capture editor and to the block-ID field, and switch the editor's `.disabled(...)` to
   read `model.editorInputLocked`.
4. Add the `focus-adopted` signpost event.
5. Extend `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` (see below).
6. Update `README.md`.
7. `just format-lint`, `just build`, `just test` cannot run on the Linux host — the
   package is `platforms: [.macOS("26.0")]` and `BobMacCapture` imports AppKit/SwiftUI.
   Land the change through the normal SASE stitch flow and require the
   `macOS 26 SwiftPM` GitHub Actions job (lint, build, test, bundle, launch smoke test)
   to be green.

## Tests

Add to `CapturePanelModelTests` (all `@MainActor`, using the existing
`installMissingTaskCompletion(on:)` helper):

- **Editor stays unlocked in the presenting turn.** After `acceptSelectedCompletion()`
  on a missing-ID candidate, assert `model.taskIDPromptVisible == true`,
  `model.focusRequest.target == .taskIDPromptBlockID`, and
  `model.editorInputLocked == false` — the assertion that encodes mechanism 2.
- **The lock lands on the next turn.** After a main-actor hop (`await Task.yield()` or
  the file's existing `waitUntil` helper), assert `model.editorInputLocked == true`.
- **A prompt canceled before the deferred set never locks the editor.** Present, call
  `cancelTaskIDPrompt()` in the same turn, hop, then assert `editorInputLocked == false`
  and `focusRequest.target == .editor`.
- **Successful assignment unlocks the editor.** Drive the existing fake-Bob success path
  and assert `editorInputLocked == false` once the draft becomes `@file+<id>`, alongside
  the existing focus-restoration assertion.
- **A Bob failure keeps the prompt open and the editor locked**, with
  `focusRequest.target == .taskIDPromptBlockID`.
- **`prepareForDismissal()` clears the lock** together with `taskIDPrompt`.

State plainly in the plan follow-up (and in the commit body) that the `.task(id:)`
adoption in step 1 has no automated coverage: SwiftUI focus resolution needs a hosted
window and an active app, which the CI job does not provide. It is covered by the manual
verification below.

## Documentation

Update `bob-mac-capture/README.md`:

- In the `@route+` bullet that describes the inline **Add block ID** prompt, state that
  opening the prompt moves keyboard focus into the block-ID field, so the ID can be
  typed immediately, and that canceling or completing the prompt returns focus to the
  capture editor.
- In the "Keyboard" table's "While Add block ID is open" column preamble or the
  paragraph under it, keep the keyboard-completeness claim and make the focus hand-off
  explicit so a future regression is a documented contract break.

## Verification

**Automated (CI, required):** the `macOS 26 SwiftPM` workflow must pass lint, build,
test, bundle, and the launch smoke test.

**Manual (on the Mac, required before the bead is closed):** install the built bundle
(`just install ~/Applications "<identity>"`), then, against a vault note containing at
least one open task with a block ID and one without:

1. Press the capture hotkey, type `note @file+`, and select a row under **Needs block
   ID** with the arrow keys, then press Return.
2. **Without touching the mouse**, type an ID. Every character must appear in the `^`
   field. This is the fix.
3. Press Return: the ID is written by Bob and the draft becomes `@file+<id>` with the
   caret after it, and typing continues in the editor.
4. Repeat, but press Escape at step 2: the task list returns and typing goes to the
   editor.
5. Repeat, submit an invalid ID (for example `bad_id`): the inline error appears and
   typing still goes to the block-ID field.
6. Regression: select a row under **Ready to use** and confirm it still inserts in one
   action with focus left in the editor.
7. Optional objective check:
   `log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"' --last 5m`
   should show `focus-adopted` at the moment the card opens.

If step 2 still fails after both changes, do not guess further: the remaining escalation
is to stop relying on SwiftUI focus for this one control and set the first responder
explicitly through AppKit (an `NSViewRepresentable` probe calling
`window.makeFirstResponder(_:)` when the prompt appears). Record that as a follow-up
with the observed signpost evidence rather than folding it into this change.

## Non-Goals

- No `bob-cli` change. Completion metadata and `capture-task-id` already behave
  correctly.
- Do not change the Add block ID card's layout, validation rule, key routing, or the
  draft/vault safety contract from `202608/file_plus_any_task.md`.
- Do not add missing-ID support to `@route:` Pomodoro completion.
- Do not rewrite the capture editor as an AppKit `NSViewRepresentable` as part of this
  fix; that is the documented last-resort escalation only.
