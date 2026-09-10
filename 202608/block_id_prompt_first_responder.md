---
tier: tale
title: Make the Add block ID field take keyboard focus by owning its first responder
goal:
  Accepting a "Needs block ID" task from `@file+` completion puts the caret in the Add
  block ID field in the same interaction, so the very next keystroke lands in that field
  without the user touching the mouse — and it does so deterministically, without
  depending on SwiftUI focus resolution.
size: medium
proposed_by: bbugyi200.athena.0c9
create_time: 2026-09-09 19:59:43
status: wip
---

# The Add Block ID Prompt Still Does Not Take Keyboard Focus

## Symptom

In the Bob Mac Capture panel: type `@file+`, select an Obsidian task from the **Needs
block ID** group, press Return. The inline **Add block ID** card opens as expected, but
nothing in it has keyboard focus. Typed characters go nowhere. The user must click the
`^` field before the ID can be typed.

This is the second report of the same defect. It was already "fixed" once by
`bob-mac-capture@0350c8c` ("fix(capture): focus the Add block ID field when a missing-ID
task is selected", plan `202608/task_id_prompt_focus.md`). That commit is still on
`master` and no later commit reverted it, so the previous fix landed and does not work.

It breaks two shipped contracts:

- Epic `202608/file_plus_any_task.md`: "a fixed `^` prefix beside a **focused**
  monospaced text field", keyboard-complete, with "predictable first-responder/focus
  restoration".
- `bob-mac-capture/README.md`, "Keyboard": "Opening the **Add block ID** prompt moves
  keyboard focus into the block-ID field; canceling or completing the prompt restores
  focus to the capture editor."

## Where To Work

Entirely in the linked `bob-mac-capture` repository. No `bob-cli` change:
`bob capture-complete --all-tasks` already returns the missing-ID candidate with
`requires_block_id`, and `bob capture-task-id` already performs the write. Before
reading or editing anything:

```bash
sase repo open bob-mac-capture -r "Fix Add block ID prompt keyboard focus"
```

Use only the path that command prints.

## Diagnosis

### The model layer is correct and is not the problem

`CapturePanelModel.presentTaskIDPrompt(candidate:draftSnapshot:replacementRange:)`
(`Sources/BobMacCapture/CapturePanelModel.swift:724`) sets `taskIDPrompt`, sets
`statusText = "Add block ID"`, calls `requestFocus(.taskIDPromptBlockID)`
(`CapturePanelModel.swift:495` bumps a monotonic `focusSequence` and republishes
`focusRequest`), and schedules the deferred editor lock. `CapturePanelModelTests`
already asserts all of that and passes. The break is downstream, in how the view turns
that request into an actual AppKit first responder.

### What the previous fix did

`0350c8c` added two mechanisms:

1. `CaptureFocusAdoption` (`CapturePanelView.swift:706`) — a `ViewModifier` carrying
   `.focused(focus, equals: target)` plus a `.task(id: request)` that re-asserts
   `focus.wrappedValue = target` after the control is installed. Applied to both the
   capture editor and the block-ID `TextField`.
2. `editorInputLocked` (`CapturePanelModel.swift:64`, `518`, `529`) — a deferred
   `.disabled(...)` lock so the capture editor does not resign first responder in the
   same transaction that presents the prompt.

Both were kept alongside the original eager mirror in `CapturePanelView`:

```swift
.onAppear { applyFocusRequest(model.focusRequest) }
.onChange(of: model.focusRequest) { _, request in applyFocusRequest(request) }
...
private func applyFocusRequest(_ request: CapturePanelFocusRequest) {   // line 514
    focusedControl = request.target
}
```

### Root cause: the repair path is unreachable, so a stored-but-unresolved focus value is never retried

The adopter is guarded on the _stored_ `@FocusState` value:

```swift
.task(id: request) {
    guard request.target == target, focus.wrappedValue != target else { return }
    focus.wrappedValue = target
}
```

Ordering inside the presenting transaction is fixed and unfavorable:

1. `acceptSelectedCompletion()` mutates `taskIDPrompt` and `focusRequest` synchronously.
2. SwiftUI recomputes the body. `TaskIDPromptCard`, and with it the `TextField` carrying
   `.focused(focus, equals: .taskIDPromptBlockID)`, is inserted for the first time.
3. `.onChange(of: model.focusRequest)` runs — during that same update, after body
   evaluation — and writes `focusedControl = .taskIDPromptBlockID`.
4. Only afterwards, on a later main-actor turn, does the field's `.task(id:)` body run.
   It reads `focus.wrappedValue`, finds `.taskIDPromptBlockID`, and **returns without
   doing anything**.

So `@FocusState` reliably ends up _holding_ `.taskIDPromptBlockID` while AppKit's first
responder is not the field. That is the well-known SwiftUI desync: assigning
`@FocusState` to a target whose backing `NSView` is not yet installed in the window
stores the value without resolving it to a first responder, and SwiftUI does not
guarantee reconciling it back to `nil` afterwards. The adopter was written to be exactly
that reconciliation, and the eager mirror in step 3 makes it a no-op every single time.

There is no other recovery path. While the prompt is visible,
`CaptureKeyCommandRouter.taskIDPromptCommand(for:modifiers:)`
(`CaptureKeyCommandRouter.swift:127`) returns `nil` for ordinary printable keys, so the
local monitor passes them through to AppKit; with no view holding first responder they
are dropped. Nothing recovers until the user clicks.

### Contributing race, still live

`editorInputLocked` is set from a `Task { @MainActor in ... }`
(`CapturePanelModel.swift:518`). That hop is _not_ ordered against SwiftUI's update
commit. If the lock lands before the update that installs the card, the capture editor
is disabled while its `NSTextView` is still first responder, AppKit resigns it, and
SwiftUI writes the focus binding back — racing step 3 above. The deferral narrowed this
window without closing it, and it only ever existed to protect a SwiftUI focus
assignment that we are about to stop relying on.

### Why not another SwiftUI focus heuristic

Every remaining variant (drop the eager mirror; use a `Bool` `@FocusState` local to the
card; hop through `DispatchQueue.main.async` before assigning) still asks SwiftUI to
resolve a focus value at a moment we cannot observe, and still gives us no way to tell
"focused" from "stored but unresolved". This has now failed once in production, cannot
be reproduced or tested from the Linux host, and costs a full build-install-retry cycle
per attempt. `202608/task_id_prompt_focus.md` pre-authorized the escalation for exactly
this outcome:

> If step 2 still fails after both changes, do not guess further: the remaining
> escalation is to stop relying on SwiftUI focus for this one control and set the first
> responder explicitly through AppKit.

Take that escalation. AppKit's `NSWindow.makeFirstResponder(_:)` is synchronous, returns
a `Bool`, works on a non-key window, and is directly assertable from a headless XCTest —
so this fix is also the first version of this behavior that CI can actually cover.

## Fix

Five changes. 1–3 are the fix; 4 is a bounded safety net for the one residual failure
mode; 5 makes the state visible to the user and to manual verification.

### 1. Give the block-ID field an AppKit-owned first responder

New file `Sources/BobMacCapture/BlockIDField.swift` (internal, not `private`, so tests
can reach it), containing an `NSTextField` subclass and an `NSViewRepresentable`.

```swift
/// Lets `CapturePanelController` find this field in the window's view tree without a
/// reference back into SwiftUI.
let blockIDFieldAccessibilityIdentifier = "org.bobs.bob-mac-capture.block-id-field"

final class BlockIDNSTextField: NSTextField {
    private var wantsFirstResponder = false

    /// Claims first responder now if the field is already in a window, and otherwise
    /// arms the claim so `viewDidMoveToWindow` performs it. Order-independent with
    /// respect to when SwiftUI installs the hosting view.
    func requestFirstResponder() {
        wantsFirstResponder = true
        claimFirstResponderIfPending()
    }

    func claimFirstResponderIfPending() {
        guard wantsFirstResponder, let window else { return }
        wantsFirstResponder = false
        CaptureSignpost.event(
            window.makeFirstResponder(self) ? "block-id-focus-claimed" : "block-id-focus-claim-failed"
        )
    }

    /// True when this field or its field editor currently holds first responder.
    var holdsFirstResponder: Bool {
        guard let responder = window?.firstResponder else { return false }
        return responder === self || (currentEditor().map { $0 === responder } ?? false)
    }

    override func viewDidMoveToWindow() {
        super.viewDidMoveToWindow()
        claimFirstResponderIfPending()
    }

    /// Hand first responder back to the window before the card is torn down, so the
    /// panel is in a clean state when SwiftUI resolves the follow-up `.editor` request.
    override func viewWillMove(toWindow newWindow: NSWindow?) {
        if newWindow == nil, holdsFirstResponder {
            window?.makeFirstResponder(nil)
        }
        super.viewWillMove(toWindow: newWindow)
    }
}
```

The representable, `BlockIDField: NSViewRepresentable`, takes `text: Binding<String>`,
`isEnabled: Bool`, `focusRequest: CapturePanelFocusRequest`, and
`focusDidChange: (Bool) -> Void`. Requirements:

- `makeNSView` configures a `BlockIDNSTextField` to match the current SwiftUI field's
  appearance: `isBordered = false`, `drawsBackground = false`, `focusRingType = .none`
  (the card draws its own border; see change 5), `usesSingleLineMode`,
  `lineBreakMode = .byClipping`, `placeholderString = "block-id"`, monospaced system
  font at `NSFont.systemFontSize`, `delegate` and `accessibilityIdentifier` set, and
  `setAccessibilityLabel("Block ID")`. Set a low horizontal content-hugging priority so
  it fills the row exactly as the `TextField` does today.
- The `Coordinator` conforms to `NSTextFieldDelegate`; `controlTextDidChange(_:)` writes
  `parent.text = field.stringValue`. `updateNSView` **must** refresh
  `context.coordinator.parent = self` first — `BlockIDField` is a value type rebuilt on
  every update, and a stale copy writes through a stale binding.
- `updateNSView` syncs `stringValue` only when it differs from `text`, syncs
  `isEnabled`, then applies focus once per focus-request sequence:

  ```swift
  guard focusRequest.target == .taskIDPromptBlockID,
        context.coordinator.appliedFocusSequence != focusRequest.sequence
  else { return }
  context.coordinator.appliedFocusSequence = focusRequest.sequence
  field.requestFirstResponder()
  // Re-arm after the current AppKit/SwiftUI transaction commits, so a first-responder
  // resignation published in the same transaction cannot land after the claim.
  DispatchQueue.main.async { [weak field] in
      guard let field, !field.holdsFirstResponder else { return }
      field.requestFirstResponder()
  }
  ```

  Use `DispatchQueue.main.async`, not `Task { @MainActor in ... }`: the runloop hop is
  the conventional "after the transaction commits" ordering, and the cooperative hop is
  what already proved insufficient. Neither is a _guarantee_ — that is why
  `viewDidMoveToWindow` and change 4 exist. Do not weaken any of the three.

- Do not override AppKit's select-all-on-focus behavior. The field is empty on the first
  claim, and select-all is the behavior we want on a re-claim after a validation error
  (the next keystroke replaces the rejected ID).
- Return is already consumed by the key monitor and routed to
  `model.submitTaskIDPrompt()`, so the field needs no `target`/`action`. Do not add one;
  it would double-submit if the routing ever changes.

In `TaskIDPromptCard` (`CapturePanelView.swift:731`) replace the `TextField` and its
`CaptureFocusAdoption` modifier with `BlockIDField`, keeping the existing
`Binding(get:set:)` onto `model.taskIDPrompt?.authoredID` /
`model.updateTaskIDPromptBlockID(_:)`, the existing padding, and
`isEnabled: !prompt.isSaving`. Drop the `.onSubmit` (see above).

### 2. Stop SwiftUI from competing for that target

`CapturePanelView.applyFocusRequest` (`CapturePanelView.swift:514`) becomes:

```swift
private func applyFocusRequest(_ request: CapturePanelFocusRequest) {
    // Only `.editor` is resolved by SwiftUI. `.taskIDPromptBlockID` is owned by AppKit
    // (`BlockIDField`), so clear SwiftUI's focus rather than asking it to resolve a
    // target it does not manage — a stored-but-unresolved value is exactly what
    // silently defeated the previous fix.
    focusedControl = request.target == .editor ? .editor : nil
}
```

Document on `CapturePanelFocusTarget` (`CapturePanelModel.swift:24`) that
`.taskIDPromptBlockID` is a model-level intent, never a `@FocusState` value. The model
API is unchanged: every existing `requestFocus(.taskIDPromptBlockID)` call site stays as
it is.

This also strengthens the return path. Because `focusedControl` is `nil` while the
prompt is open, the later `requestFocus(.editor)` is a real `nil` → `.editor` transition
on a control that already exists in the window, and `CaptureFocusAdoption` on the editor
gets a second chance at it.

Leave `CaptureFocusAdoption` on the capture editor exactly as it is — that path is not
reported broken and is not worth churning — but extend its comment to record that its
`focus.wrappedValue != target` guard makes it a no-op whenever the eager mirror already
stored the target, so it must not be mistaken for a general focus repair.

### 3. Lock the editor synchronously and delete the deferral

With AppKit owning the claim, the deferral has no job left and only preserves a race. In
`CapturePanelModel`:

- Delete `editorInputLockToken` (`:92`) and `scheduleDeferredEditorInputLock()`
  (`:518`).
- In `presentTaskIDPrompt` (`:742`) replace `scheduleDeferredEditorInputLock()` with
  `editorInputLocked = true`.
- `clearEditorInputLock()` (`:529`) reduces to clearing the flag; keep it called from
  `clearTaskIDPrompt()` so every prompt-clearing path still unlocks.

Keep `@Published private(set) var editorInputLocked` and the editor's
`.disabled(model.isSubmitting || model.editorInputLocked)`: the lock is a real contract
(the editor is not typable while the prompt is open) and now resolves in the presenting
transaction, which is what we want — the `NSTextView` resigns immediately and is not
competing when `BlockIDField` claims on the following runloop turn.

### 4. Repair an orphaned first responder at the keystroke

The one residual failure mode is "the claim was made and something later cleared first
responder". Close it where it is observable: in the local key monitor, before routing.

In `CapturePanelController`:

```swift
/// While the Add block ID prompt is open, a key event that arrives with no view holding
/// first responder would be dropped. Re-claim the block-ID field in exactly that case.
/// When any real control holds first responder — the field editor, or Cancel /
/// Add & Select under Full Keyboard Access — this does nothing.
private func repairBlockIDFocusIfOrphaned() {
    guard model.taskIDPromptVisible,
          model.taskIDPrompt?.isSaving != true,
          let panel,
          Self.blockIDFocusIsOrphaned(in: panel),
          let field = Self.findBlockIDField(in: panel.contentView)
    else { return }
    field.requestFirstResponder()
    CaptureSignpost.event("block-id-focus-repaired")
}

/// Orphaned means no view owns first responder: either the window itself does, or the
/// hosting view does. Neither is a control the user could have deliberately focused.
static func blockIDFocusIsOrphaned(in window: NSWindow) -> Bool {
    guard let responder = window.firstResponder as? NSView else { return true }
    return responder === window.contentView
}

/// Depth-first search for `blockIDFieldAccessibilityIdentifier`.
static func findBlockIDField(in view: NSView?) -> BlockIDNSTextField? { ... }
```

Call `repairBlockIDFocusIfOrphaned()` from the monitor closure in
`installKeyMonitorIfNeeded()` (`CapturePanelController.swift:642`), after the
`panel?.isKeyWindow == true` check and **before** `keyRouter.command(for:context:)`.
Changing first responder inside a local monitor takes effect for the same event, because
`NSWindow.sendEvent` resolves the responder after the monitor returns — so the
triggering keystroke lands in the field rather than being eaten.

Both statics must be pure enough to unit test: `blockIDFocusIsOrphaned(in:)` takes a
window, `findBlockIDField(in:)` takes a view.

**Deliberately not implemented:** the general "re-focus the field whenever `@FocusState`
becomes `nil` while the prompt is open" healer that `202608/task_id_prompt_focus.md`
rejected. The orphan test above is the narrow version that cannot steal focus from
Cancel or Add & Select.

### 5. Make focus visible

Give the user (and manual verification) an unambiguous signal that typing will land.
`BlockIDField` reports focus changes through `focusDidChange` from
`controlTextDidBeginEditing(_:)` / `controlTextDidEndEditing(_:)`; `TaskIDPromptCard`
holds it in `@State private var blockIDFieldIsFocused = false` and uses it for the
existing border overlay only:

```swift
RoundedRectangle(cornerRadius: 6)
    .strokeBorder(
        blockIDFieldIsFocused ? Color.accentColor.opacity(0.8) : .secondary.opacity(0.24),
        lineWidth: blockIDFieldIsFocused ? 1 : 0.5
    )
```

Color and stroke width only. No layout, no geometry, no new rows — the card's measured
height must not change, because `CapturePanelController` sizes the panel from it.

## Implementation Steps

1. `sase repo open bob-mac-capture` and work only from the printed path.
2. Add `Sources/BobMacCapture/BlockIDField.swift` (change 1). Match the file-level
   conventions of the target: `@available(macOS 26.0, *)` where the surrounding types
   use it, and no redundant actor annotations — `NSView` subclasses are already
   main-actor isolated by the SDK.
3. `CapturePanelView.swift`: use `BlockIDField` in `TaskIDPromptCard`, narrow
   `applyFocusRequest` (change 2), extend the `CaptureFocusAdoption` comment, and add
   the focused-border state (change 5).
4. `CapturePanelModel.swift`: synchronous editor lock, delete the deferral (change 3).
5. `CapturePanelController.swift`: orphan repair plus the two testable statics (change
   4).
6. Tests, below.
7. `README.md`, below.
8. `just format-lint`, `just build`, and `just test` cannot run on the Linux host: the
   package is `platforms: [.macOS("26.0")]` and `BobMacCapture` imports AppKit/SwiftUI.
   Land through the normal SASE stitch flow and require the `macOS 26 SwiftPM` GitHub
   Actions job (lint, build, test, bundle, plist/signature, launch smoke test) to be
   green. `swift build --target CaptureCore` still works locally and should still pass;
   nothing in this change touches `CaptureCore`.

## Tests

### Update `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`

The deferral assertions now encode the removed behavior and must be inverted, not
deleted — the synchronous lock is a real contract:

- `testMissingTaskCompletionOpensTaskIDPromptWithoutChangingDraft` (`:1294`):
  `XCTAssertFalse(model.editorInputLocked)` → `XCTAssertTrue`.
- `testTaskIDPromptDefersEditorInputLockUntilNextTurn` (`:1314`) → rename to
  `testTaskIDPromptLocksEditorInPresentingTurn`: assert `editorInputLocked` is `true`
  immediately after `acceptSelectedCompletion()`, then hop once and assert it is still
  `true` and the prompt is still visible.
- `testCancelTaskIDPromptBeforeDeferredLockNeverLocksEditor` (`:1331`) → rename to
  `testCancelTaskIDPromptUnlocksEditorImmediately`: present, assert locked, cancel,
  assert unlocked synchronously and `focusRequest.target == .editor`, then hop and
  assert nothing re-locks.
- Replace every `await waitUntil { model.editorInputLocked }` (`:1324`, `:1352`,
  `:1528`) with a direct assertion.

### New `Tests/BobMacCaptureTests/BlockIDFieldFocusTests.swift` (`@MainActor`)

`NSWindow.makeFirstResponder(_:)` does not require a key or ordered-in window, so these
run headless on the CI runner. Do not call `makeKeyAndOrderFront`, and do not gate any
of them behind a skip — if they cannot run, that is a finding, not a skip.

- **Claims immediately when already in a window.** Add a `BlockIDNSTextField` to an
  offscreen `NSWindow`'s `contentView`, call `requestFirstResponder()`, assert
  `holdsFirstResponder`.
- **Claims on entry when armed before it has a window.** Call `requestFirstResponder()`
  on a detached field, then add it to the window; assert `holdsFirstResponder` — this is
  the ordering that the previous fix could not survive.
- **Takes focus away from another responder.** Make a second `NSTextField` first
  responder, then claim; assert the block-ID field holds it and the other does not.
- **Releases on teardown.** With the field focused, `removeFromSuperview()`; assert the
  window's first responder is no longer a view in the removed subtree.
- **`blockIDFocusIsOrphaned(in:)` truth table.** First responder = the window → `true`;
  = `contentView` → `true`; = a focused text field's field editor → `false`; = an
  `NSButton` → `false`.
- **`findBlockIDField(in:)` finds a nested field and returns `nil` for a tree without
  one.**
- **The hosted card exposes the field to the controller.** Host a view that uses
  `BlockIDField` in an `NSHostingView` inside an `NSWindow`, pump the runloop briefly,
  then assert `CapturePanelController.findBlockIDField(in: window.contentView) != nil`.
  This is the guard on the accessibility-identifier contract that change 4 depends on;
  it deliberately asserts reachability, not SwiftUI focus resolution.

State plainly in the plan follow-up and the commit body what remains uncovered: the
end-to-end path from `acceptSelectedCompletion()` through a real SwiftUI update to a
first responder inside the live panel. Everything under it is now covered.

## Documentation

`bob-mac-capture/README.md`:

- In the `@route+` bullet describing the inline **Add block ID** prompt (around the
  "Ready tasks stay first" paragraph), keep the focus contract and add that the field's
  border highlights while it holds focus.
- In the "Keyboard" paragraph that states the focus hand-off, add that the block-ID
  field's first responder is owned directly by AppKit rather than SwiftUI focus, and
  that a keystroke arriving while nothing holds focus re-claims the field.
- In "Troubleshooting", add an entry for "typing does not reach the Add block ID field":
  check `log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"'` for
  `block-id-focus-claimed` (normal), `block-id-focus-repaired` (the safety net fired),
  or `block-id-focus-claim-failed` (report it).

## Verification

**Automated (CI, required):** the `macOS 26 SwiftPM` workflow must pass lint, build,
test, bundle, plist/signature, and the launch smoke test.

**Manual (on the Mac, required before the bead is closed).** First confirm you are
testing the new code — this defect has already been reported once against a build that
did contain its fix:

0. `just install ~/Applications "<identity>"`, relaunch the app, and confirm the
   installed bundle is the new build (compare `Contents/MacOS` mtime against the build,
   or check for a `block-id-focus-claimed` signpost in step 2).

Then, against a vault note with at least one open task that has a block ID and one that
does not:

1. Press the capture hotkey, type `note @file+`, arrow to a row under **Needs block
   ID**, press Return. The card opens and its `^` field border is highlighted.
2. **Without touching the mouse**, type an ID. Every character appears in the `^` field.
   This is the fix.
3. Press Return: Bob writes the ID, the draft becomes `@file+<id>` with the caret after
   it, the border highlight is gone, and typing continues in the editor.
4. Repeat and press Escape at step 2: the task list returns and typing goes to the
   editor.
5. Repeat and submit an invalid ID (`bad_id`): the inline error appears, the field keeps
   focus, and typing still goes to the block-ID field.
6. Repeat and submit a valid ID while Bob is unresolved or failing: the error appears
   and focus stays in the field.
7. With Full Keyboard Access on, Tab from the field to **Cancel** and **Add & Select**
   and press Space on each: the buttons act normally and focus is not yanked back to the
   field. This is the regression guard on change 4.
8. Regression: select a row under **Ready to use** and confirm it still inserts in one
   action with focus left in the editor.
9. Objective check:
   `log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"' --last 5m`
   shows `block-id-focus-claimed` when the card opens. `block-id-focus-repaired` means
   the primary claim is still losing a race and should be reported; a
   `block-id-focus-claim-failed` is a bug in this change.

If step 2 still fails after all of this, record the signpost output and the value of
`panel.firstResponder` (add a temporary diagnostic) on the bead before attempting
anything further. Do not add another speculative focus heuristic on top.

## Non-Goals

- No `bob-cli` change. Completion metadata and `capture-task-id` already behave
  correctly.
- Do not rewrite the capture editor as an `NSViewRepresentable`. Only the block-ID field
  moves to AppKit.
- Do not reimplement text editing in `CaptureKeyCommandRouter` — the block-ID field
  stays a native single-line control with native editing, selection, paste, undo, and
  IME. The router's printable-key pass-through for the prompt is unchanged.
- Do not add a general "re-focus whenever `@FocusState` becomes `nil`" healer.
- Do not change the card's layout, validation rule (`isValidBlockID`), key routing, or
  the draft/vault safety contract from `202608/file_plus_any_task.md`. The only visual
  change is the border stroke's color and width.
- Do not add missing-ID support to `@route:` Pomodoro completion.
