---
tier: tale
title: Require uppercase D and place Delete All below the stash list
goal:
  The stash picker clears only for uppercase D and shows its destructive action below
  the scrollable draft rows.
size: small
proposed_by: bbugyi200.athena.02g.f0
create_time: 2026-09-09 20:00:31
status: wip
---

# Require uppercase D and place Delete All below the stash list

## Goal

Refine Bob Mac Capture's Control-S canceled-draft stash picker so its destructive
clear-all action only runs for an uppercase `D`, with lowercase `d` remaining inert, and
so the visible Delete All action sits at the bottom of the picker instead of above the
stashed-draft rows. Keep the existing session-only clearing semantics, row accelerators,
modal keyboard behavior, and Settings confirmation flow intact.

## Current behavior

- `CaptureKeyCommandRouter.stashPickerCommand` currently requires an empty modifier set
  and calls `isClearStashKey`, which uppercases the event character. As a result,
  lowercase `d` clears the stash, while the normal Shift-D event is consumed by the
  modal picker without clearing it.
- `CanceledDraftStashPicker` renders `clearAllButton` before the scrollable row list,
  which pins the destructive action to the top of the material card.
- `D` is already excluded from the 36 restore accelerators, and the model/controller
  clear path already clears all entries, closes only the picker, leaves the capture
  panel and editor intact, and emits metadata-only status text.

## Implementation

1. Tighten uppercase-D routing in `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`.
   - Replace the case-folded, unmodified-character match with an exact uppercase
     character match based on `NSEvent.characters`.
   - Accept capitalization-only modifier states so a normal Shift-D key event works and
     an uppercase `D` produced with Caps Lock works. Reject combinations carrying
     Command, Control, Option, Function, or other semantic modifiers.
   - Preserve modal behavior for lowercase `d`: it must resolve to `.consumeKey`, not
     `.clearCanceledDraftStash`, so it neither clears the stash nor leaks into the
     capture editor.
   - Preserve Command shortcut fallthrough, including Command-Shift-D, and preserve
     consumption of other modified printable keys while the picker is open.

2. Move and clarify the destructive picker action in
   `Sources/BobMacCapture/CapturePanelView.swift`.
   - Render the scrollable stash rows first and `clearAllButton` afterward in the picker
     `VStack`, leaving the button outside the `ScrollView` so it remains fixed at the
     bottom while long stash lists scroll above it.
   - Keep the existing destructive styling, click behavior, card sizing, selection
     scrolling, and accessibility containment.
   - Update the keycap and accessibility label/hint to communicate the uppercase
     requirement explicitly (using Shift-D notation for the ordinary keyboard path)
     without implying that lowercase `d` works.

3. Update focused routing coverage in
   `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`.
   - Assert Shift-D with uppercase event characters returns `.clearCanceledDraftStash`.
   - Assert an uppercase `D` produced under Caps Lock also clears.
   - Assert plain lowercase `d` returns `.consumeKey` and never the destructive command.
   - Retain or extend assertions that Command-D and Command-Shift-D fall through, while
     Control-D and Option-D remain modal-consumed.
   - Leave the existing controller/model clear tests in place as regression coverage for
     the consequences of a successfully routed uppercase D.

4. Align the stash-picker documentation in `README.md` with the UI and routing.
   - Describe Delete All as an uppercase-D action, show Shift-D in the key table and
     visible-action wording, and state that lowercase `d` does nothing destructive.
   - Keep the reserved-accelerator explanation and all clear-all lifecycle/privacy
     semantics unchanged.

## Validation

Run the repository's supported checks from the `bob-mac-capture` checkout:

1. `just format-lint`
2. `just build`
3. `just test`

Then manually exercise the Control-S picker with multiple retained drafts:

- Confirm the Delete All action is fixed below the row viewport and remains there while
  the rows scroll.
- Confirm lowercase `d` leaves the stash and picker unchanged.
- Confirm Shift-D clears every entry, closes only the picker, leaves the capture panel
  open with an empty editor, and reports `Canceled draft stash cleared`.
- Confirm clicking the bottom Delete All action has the same behavior.
- Confirm restore accelerators, selection navigation, Escape/Control-[ dismissal,
  Control-S toggling, and Command shortcuts continue to behave as documented.

## Non-goals

- Do not change stash capacity, persistence, ordering, privacy, Control-C stashing,
  restore/pop behavior, or the reserved 36-key accelerator table.
- Do not add confirmation to the modal picker action or alter the confirmed Settings
  `Clear Stash...` workflow.
- Do not change the clear operation's model/controller semantics, capture grammar, or
  bob-cli subprocess/JSON contracts.
