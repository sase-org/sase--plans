---
tier: tale
title: Add a keyboard-first canceled-draft stash to Bob Mac Capture
goal:
  Bob Mac Capture safely stashes canceled drafts and restores any retained entry through
  a polished one-key picker.
size: medium
proposed_by: bbugyi200.athena.024
create_time: 2026-09-09 19:59:47
status: wip
---

# Add a keyboard-first canceled-draft stash to Bob Mac Capture

## Goal

Add a bounded stash for the exact contents of drafts canceled with Control-C, plus a
polished Control-S picker that restores and removes any retained draft with one
keypress. The feature must preserve Bob Mac Capture's compact panel, native keyboard
behavior, accessibility, failure isolation, and existing promise that captured text is
not persisted to disk by the app.

This is entirely a `bob-mac-capture` feature. It does not change capture grammar, JSON
interfaces, subprocess behavior, or any other `bob-cli` contract.

## Product and interaction contract

- Keep canceled drafts in a dedicated, app-lifetime in-memory store. Persist only the
  capacity preference in `UserDefaults`; never log, notify, signpost, diagnose, or write
  stash contents to disk. Quitting or restarting the app clears the stash, and Settings
  will say so explicitly.
- Default the capacity to 10. Let the user choose 0 through 36 in Settings, where 0
  turns the feature off. The upper bound guarantees that every retained entry can always
  have a unique single-key accelerator: `1` through `9`, then `0`, then `A` through `Z`.
- Treat Control-C and the visible **Discard** button differently and make that
  distinction discoverable. Exact Control-C on a semantically nonempty draft pushes its
  complete `String` (including Unicode, line breaks, and surrounding whitespace) to the
  newest end of the stash, trims the oldest overflow, clears all draft/analysis state,
  and closes the panel. Control-C on an empty/whitespace-only draft simply closes.
  **Discard** remains an explicit permanent discard and never adds an entry.
- Preserve repeated cancellations as separate history entries; do not silently
  de-duplicate them. Changing the capacity immediately keeps the newest `N` entries and
  drops older overflow. Setting it to zero clears the in-memory history and prevents new
  additions.
- Add a subtle, count-bearing **Stash** action to the capture footer for pointer and
  VoiceOver discoverability. Exact Control-S toggles the same picker. If there is no
  retained entry, report a metadata-only "No canceled drafts yet" status. If the current
  editor is nonempty, refuse to open the picker and explain that the current draft must
  be captured, retained, or canceled first; never overwrite or implicitly stash live
  work.
- Present the picker as a compact elevated material card in the panel's measured
  auxiliary region, using the existing completion-list visual language rather than a
  separate window. Show newest first. Each row gets a crisp accelerator keycap, a safely
  truncated first meaningful line, and secondary metadata such as line count and
  relative age. Never alter the stored string to produce its display preview.
- Make the picker modal while visible: its selected row starts at the newest entry;
  arrows and Control-N/Control-P move with wraparound, Return restores the selected row,
  Escape or Control-[ closes only the picker, and Control-S toggles it closed. A
  displayed accelerator restores its row immediately. Consume unrelated unmodified
  printable keys while the picker is open so typing cannot mutate the editor behind it,
  while leaving unrelated Command shortcuts to AppKit.
- On pointer click, Return, or accelerator selection, first install the exact chosen
  text in the empty editor with the caret at its UTF-8 end, close the picker, invalidate
  stale completion/preview work, and schedule normal parse/preview analysis; then remove
  only that entry from the stash. Keep the panel open and focused for editing or
  capture. This makes "pop" explicit and prevents selection from destroying a current
  draft.
- Give the picker and every row complete accessibility labels, selected/button traits,
  and hints that include the accelerator and restore action. Announce picker-open,
  empty, refused, and restored states through metadata-only status text without copying
  private draft contents into diagnostics.

## Implementation

1. Add `Sources/BobMacCapture/CanceledDraftStash.swift` with an immutable entry
   identity, exact text, and in-memory timestamp plus a `@MainActor` observable bounded
   store. Keep newest-first ordering and expose deterministic push,
   lookup/remove-by-identity, clear, capacity-update/trim, selection-clamping helpers,
   and pure row-preview/accelerator formatting where useful. Keep every mutation
   synchronous on the main actor so cancel, restore, capacity changes, and UI
   observation cannot race.

2. Extend `Sources/BobMacCapture/AppSettings.swift` with the persisted capacity
   preference, a default of 10, and defensive clamping to 0...36 for both loaded and
   assigned values. Wire one app-lifetime `CanceledDraftStash` through
   `Sources/BobMacCapture/AppDelegate.swift` into the panel model/view and settings
   scene, and observe capacity changes so trimming is immediate. Do not put entry text
   in `AppSettings`, `UserDefaults`, diagnostics, or status history.

3. Add a **Canceled Draft Stash** section to `Sources/BobMacCapture/SettingsView.swift`.
   Provide a labeled stepper with a clear Off state at zero, the current retained count,
   a concise session-only privacy/lifetime note, and a destructive **Clear Stash...**
   action with confirmation. Disable clearing when the store is empty and ensure every
   label is meaningful to VoiceOver.

4. Extend `Sources/BobMacCapture/CapturePanelModel.swift` with explicit stash-picker
   state and operations: stash-and-close for the Control-C path, guarded picker
   presentation, dismissal, wrapped selection, and restore/pop by stable entry identity.
   Reuse the existing draft-reset and analysis-generation machinery so canceling clears
   errors/completions and restoring cannot accept stale subprocess results. Keep
   `discardDraftAndClose()` as the non-stashing button path, reset picker state on panel
   presentation/dismissal, and clamp the selection when capacity changes or Settings
   clears the store.

5. Refine `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` around an explicit
   routing context instead of adding loosely related booleans. Add exact Control-S and
   stash-modal commands, one-key accelerator decoding from layout-aware event
   characters, modal navigation/acceptance/dismissal precedence, and printable-key
   consumption. Update `Sources/BobMacCapture/CapturePanelController.swift` to supply
   stash state/count, route Control-C to stash-and-close, preserve the existing
   completion precedence outside the picker, and restore focus without interfering with
   native text editing, IME, paste, undo, or Command shortcuts.

6. Add the footer affordance and a dedicated stash picker/row to
   `Sources/BobMacCapture/CapturePanelView.swift`. Integrate picker visibility into the
   existing auxiliary-content height measurement, show the picker instead of completion,
   preview, destination, or error details while it is modal, cap its scrolling viewport,
   follow existing material/corner/shadow/contrast conventions, keep multiline previews
   restrained, scroll keyboard selection into view, and support pointer selection and
   the accessibility contract above. The compact empty panel and top-anchored resize
   behavior must remain unchanged when the picker is closed.

7. Update `README.md` as the user-facing contract: document Control-C as
   session-stashing, Control-S and all picker navigation/accelerators, the permanent
   **Discard** distinction, default/configurable bounds, pop ordering,
   empty/nonempty-editor behavior, Settings clear controls, and session-only privacy.
   Revise the current statements that Control-C always discards and explain that app
   quit/restart intentionally clears canceled text.

## Tests and verification

- Add focused unit coverage for the store: exact Unicode/multiline preservation,
  newest-first ordering, duplicate entries, overflow eviction, arbitrary identity-based
  pop, immediate capacity reduction, disabled capacity, clear, stable accelerators
  through all 36 slots, and preview formatting that never mutates the payload.
- Extend AppSettings and lifecycle tests for the default of 10, persisted choices,
  corrupt or out-of-range value clamping, live capacity propagation, and the invariant
  that stash payloads never enter `UserDefaults` or diagnostic history.
- Extend `CapturePanelModelTests` for nonempty Control-C stash/clear/dismiss behavior,
  whitespace-only cancellation, the non-stashing **Discard** path, guarded picker
  opening, empty-state feedback, wrapped selection, settings-driven clear/trim clamping,
  exact restore with caret-at-end and fresh analysis, and removal of only the
  successfully restored entry.
- Extend key-router/controller tests for exact Control-S modifiers, Control-C's new
  command, precedence over visible completion, all numeric/alphabetic accelerators,
  out-of-range and modified keys, modal printable-key consumption,
  arrow/Control-N/Control-P wrapping, Return, Escape/Control-[, toggle-close, and
  unchanged behavior when the picker is absent.
- Add view/presentation-policy coverage for row labels and metadata, selection traits
  where testable, picker scrolling, auxiliary height inclusion, and regression checks
  that the existing one-line compact height and completion UI still behave as before.
- From the `bob-mac-capture` repository, run `just format-lint`, `just build`, and
  `just test`. Manually exercise a signed/bundled debug build with empty, single-line,
  multiline, long, duplicate, and Unicode drafts; verify `1`...`9`, `0`, and letter
  accelerators, pointer and VoiceOver interaction, reduced/increased contrast, panel
  top-anchor resizing, Settings capacity/clear behavior, no overwrite of a live draft,
  exact pop semantics, and session clearing after restart.
