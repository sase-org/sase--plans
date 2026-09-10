---
tier: tale
title: Add one-key clear-all to the canceled-draft stash picker
goal:
  Bob Mac Capture users can delete every stashed draft from the Control-S picker with
  one D keypress.
size: small
proposed_by: bbugyi200.athena.02g
create_time: 2026-09-09 20:00:30
status: wip
---

# Add one-key clear-all to the canceled-draft stash picker

## Goal

Let a user press the unmodified `D` key while Bob Mac Capture's Control-S stash picker
is open to delete every stashed draft immediately. Make the destructive action visible
and accessible in the picker, preserve the existing session-only privacy boundary, and
retain unique one-key restore access for every supported stash slot.

This is a focused follow-up in the linked `bob-mac-capture` repository. It does not
change capture grammar, subprocess/JSON contracts, stash capacity or persistence,
Control-C stashing, per-entry restore/pop behavior, or the confirmed clear action in
Settings.

## Product and interaction contract

- Reserve the unmodified `D` key for **Delete All** only while the stash picker is
  presented. Use the same uppercase keycap notation as the existing row accelerators:
  the user presses the physical/layout-aware `d` key without Shift or another modifier.
  Outside the picker, `D` must remain ordinary editor input.
- Treat this as the requested single-key action: pressing `D` immediately clears every
  entry without opening a second confirmation dialog. The existing Settings **Clear
  Stash...** confirmation remains unchanged for the non-modal settings workflow.
- Clear through the existing app-lifetime `CanceledDraftStash` rather than duplicating
  storage logic. After clearing, close the now-empty picker, reset its selection, keep
  the capture panel open with an empty editor, and announce a metadata-only status such
  as "Canceled draft stash cleared." Never include deleted draft content in status,
  diagnostics, notifications, logs, signposts, or persistence.
- Make the action discoverable inside the Control-S panel with a compact destructive **D
  Delete All** affordance that can also be clicked. Keep it visually subordinate to the
  draft rows but visible while the list scrolls, and give it an accessibility label and
  hint that clearly say it permanently removes all retained drafts from the current app
  session.
- Preserve the modal safety contract: the clear command takes precedence over row
  accelerators; unrelated printable keys cannot edit the hidden editor; and unrelated
  Command shortcuts continue to fall through to AppKit. Modified `D` variants must not
  trigger deletion.
- Resolve the existing conflict where `D` currently restores row 14. Keep all 36 stash
  slots uniquely reachable with one key by changing the centralized accelerator order to
  `1`...`9`, `0`, `A`...`C`, `-`, `E`...`Z`. This reserves `D`, gives row 14 the visible
  `-` keycap, and leaves the later `E`...`Z` mappings at their current row positions.

## Implementation

1. Update `Sources/BobMacCapture/CanceledDraftStash.swift` so the single source of truth
   for row accelerators omits `D` and assigns `-` to the displaced slot. Keep the
   existing 36-entry maximum and make `accelerator(for:)` and
   `acceleratorIndex(for:entryCount:)` round-trip the revised, unique sequence without
   affecting entry ordering or payloads.

2. Extend `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` with an explicit
   clear-all stash command. In stash-modal routing, recognize exact unmodified `D`
   before generic accelerator decoding/printable-key consumption, route `-` through the
   revised accelerator table, and preserve the current behavior for Return, navigation,
   dismissal, Control-S, modified printable keys, and Command shortcuts.

3. Add one model-level clear operation in
   `Sources/BobMacCapture/CapturePanelModel.swift`. It should be safe to call only from
   the picker action path, synchronously invoke `canceledDraftStash.clear()`, rely on or
   explicitly preserve the existing empty-store picker/selection reset invariant, and
   publish a privacy-safe success announcement after the store change. It must not
   dismiss the capture panel, mutate the empty editor, or alter the configured capacity.

4. Route the new command in `Sources/BobMacCapture/CapturePanelController.swift` to the
   model operation and consume the event. Keep the existing routing context as the
   authority for whether the picker is modal, so no new global or editor-level `D`
   shortcut is introduced.

5. Refine `CanceledDraftStashPicker` in `Sources/BobMacCapture/CapturePanelView.swift`
   to expose a compact clickable destructive **D Delete All** action and the
   corresponding keyboard hint. Keep that action outside the scrolling row viewport so
   it remains available for long stashes, preserve the existing material card, measured
   auxiliary sizing, row selection and scrolling behavior, and update picker/row
   accessibility hints for the revised accelerators and clear-all command.

6. Update `README.md` to document `D` in the stash-picker key table, immediate
   session-only clear semantics, the `-` restore accelerator, the unchanged Settings
   confirmation, and the revised reason the 36-entry upper bound still provides a unique
   key for every row.

## Tests and verification

- Update `Tests/BobMacCaptureTests/CanceledDraftStashTests.swift` to assert the exact
  36-key sequence, uniqueness, `D`'s absence from restore decoding, `-`'s row-14
  round-trip, unchanged bounds behavior, and stable `E`...`Z` positions.
- Extend key-router coverage in `Tests/BobMacCaptureTests/BobMacCaptureTests.swift` to
  prove unmodified `D` wins over restore accelerators at full capacity, `-` restores row
  14, modified `D` variants do not clear, Command shortcuts still fall through, and all
  existing modal commands remain unchanged. Add a controller assertion that the routed
  command consumes the event and clears the model's shared stash without closing the
  panel.
- Extend `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` with multiple entries
  to verify clear-all removes every payload, dismisses the picker, resets selection,
  keeps the editor/panel intact, preserves capacity, and emits only the metadata-only
  cleared status/announcement. Retain coverage that external clear/capacity-zero
  operations still close an empty picker correctly.
- Add view-level assertions where the current test architecture permits for the visible
  destructive label and accessibility hint; otherwise cover them in the manual pass.
- From the linked `bob-mac-capture` repository, run `just format-lint`, `just build`,
  and `just test`. Manually open a full 36-entry stash and verify `D` clears it in one
  unmodified keypress, `-` restores row 14 instead, the action stays visible while rows
  scroll, the panel remains open and ready, pointer and VoiceOver actions work, modified
  `D` does not clear, and no draft text appears in persisted or diagnostic surfaces.
