---
tier: tale
title: Make the task-move destination picker single-instance
goal:
  One Ctrl+Shift+M press opens exactly one task-move destination picker, and after the
  user selects a destination (or dismisses the picker) no picker remains on screen, with
  counted moves still honored.
create_time: 2026-09-09 19:53:24
status: wip
---

# Plan: Make the task-move destination picker single-instance

## Context

Bob Navigation Hotkeys 1.13.9 (linked `bob-plugins` repository) made the task-move
destination picker close synchronously when a destination is selected, and the user
confirms the related blank-line fix is live in the vault. Yet after selecting a
destination a menu still remains on screen. Because the picker the user interacts with
now closes unconditionally and synchronously before the move commits, the surviving menu
can only be a second, identically rendered `TaskMoveDestinationPickerModal` stacked
beneath the one that closed.

Investigation of `plugins/bob-navigation-hotkeys/main.js` found that duplicate picker
instances have unreconciled ways to appear for a single physical Ctrl+Shift+M press:

- Two independent open paths are bound to the same chord. The `move-tasks-to-note`
  command registers a default Ctrl+Shift+M hotkey dispatched by Obsidian's keymap, and a
  global capture-phase keydown listener (the counted task-move handler) separately
  handles every vim-normal-mode Ctrl+Shift+M and opens the picker directly. The capture
  handler's `stopImmediatePropagation` only suppresses Obsidian's dispatcher if the
  plugin's listener happens to run first, which depends on undocumented
  listener-registration order inside Obsidian that the plugin cannot control or observe.
  When Obsidian's dispatcher runs first, both paths fire and two pickers stack. Notably,
  the sibling counted bullet-property feature follows a safer convention — its physical
  handler only claims presses with an explicit vim count and its command registers no
  default hotkey — while task-move is the outlier that binds both paths to the same bare
  chord.
- Neither open path ignores auto-repeated keydown events (`event.repeat`), so holding
  the chord slightly too long can fire additional keydowns that each open another
  stacked picker.
- `openTaskMoveDestinationPicker` has no idempotence guard: every call unconditionally
  captures a session and opens a fresh modal.

Because the stacked pickers render identically and overlap exactly, the duplicate is
invisible until the top picker closes — which is precisely the reported symptom, both
before and after the 1.13.9 fix.

The fix must be dispatch-order-agnostic: rather than guessing Obsidian keymap internals,
make the task-move destination picker single-instance at the plugin level so every
duplicate-open mechanism collapses to one visible picker.

The linked `bob-plugins` repository is the source of truth. Open it through
`sase repo open bob-plugins` before implementation, make all plugin changes there, and
deploy the completed plugin through `bob plugins sync`; do not edit the vault's
installed plugin files directly.

## Implementation

1. Add a single-instance guard for the task-move destination picker in
   `plugins/bob-navigation-hotkeys/main.js`.
   - Track the currently open `TaskMoveDestinationPickerModal` on the plugin instance.
     Clear the tracking whenever the modal closes through any path: Escape/programmatic
     dismissal, and the close-before-commit path that selection now takes. Route the
     clearing through the modal's close lifecycle so no close path can leave stale
     tracking behind.
   - When `openTaskMoveDestinationPicker` is invoked while a tracked picker is already
     open, reconcile instead of stacking: if the new request carries an explicit vim
     count and the tracked picker's session does not, close the tracked bare picker and
     open the counted replacement (this is the same-keypress duplicate where Obsidian's
     hotkey dispatched first and the capture handler supplied the count second);
     otherwise ignore the duplicate request and keep the existing picker. The result of
     a single physical keypress must always be exactly one visible picker whose session
     reflects the explicit count when one was typed.
   - Keep every current entry point available and user-visible: the command palette
     entry, the default Ctrl+Shift+M hotkey binding, and the physical vim-normal-mode
     handler (including bare, uncounted presses) all continue to work.
   - Leave session capture, destination collection, commit, stale-state guards, notices,
     and rollback behavior unchanged.

2. Ignore auto-repeated keydown events in the counted task-move physical handler.
   - Return without handling (and without swallowing the event) when `event.repeat` is
     true, before any vim-state reads or picker opens, so a held chord cannot re-trigger
     the flow. Any duplicate that still arrives through the command path collapses in
     the single-instance guard.

3. Add focused regressions in `scripts/test-navigation-hotkeys.cjs` using the existing
   plugin/modal mock harness.
   - Duplicate opens within one dispatch: a bare open followed by a counted open leaves
     exactly one open picker whose session carries the count; bare-then-bare keeps the
     first picker and ignores the duplicate; counted-then-bare keeps the counted picker.
   - Close clears tracking: after the picker closes via Escape and via the selection
     close-before-commit path, a subsequent open succeeds and captures a fresh session.
   - An `event.repeat` keydown is declined by the physical handler and opens nothing,
     while an otherwise identical non-repeat keydown still opens the picker.
   - Existing spacing, picker-lifecycle, single-flight commit, and unrelated-picker
     close-contract coverage must remain green and unmodified in intent.

4. Release, validate, and deploy the corrected plugin.
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.13.9` to the next patch
     version and keep the Bob Navigation Hotkeys version in `README.md` synchronized.
   - Run the focused suite with `node --test scripts/test-navigation-hotkeys.cjs`, then
     the repository-wide `npm test` and `npm run validate` checks.
   - Deploy only Bob Navigation Hotkeys with
     `bob plugins sync -p bob-navigation-hotkeys`, passing the linked `bob-plugins`
     checkout path printed by `sase repo open` explicitly via the sync command's repo
     flag so the audited source checkout is deployed rather than the default canonical
     checkout, and without forcing over dirty vault files. Verify the sync/status output
     reports the installed plugin enabled at the new version with zero drift. If the
     deploy refuses because the vault copy is dirty, stop and report that state rather
     than overwriting it.

## Acceptance criteria

- A single Ctrl+Shift+M press in vim normal mode produces exactly one visible
  destination picker, and after choosing a destination by keyboard or mouse no picker
  remains on screen; dismissing with Escape likewise leaves nothing behind.
- Counted moves (a vim count typed before Ctrl+Shift+M) present one picker whose
  subtitle and session reflect the requested count, regardless of which open path fired
  first.
- Holding the chord long enough to auto-repeat does not stack additional pickers.
- The command palette entry and non-vim invocation contexts keep their current behavior,
  and stale-state and transactional failure notices/rollback guarantees are unchanged.
- Focused and full test suites pass, all plugin manifests validate, the manifest and
  README versions agree, and the updated source-of-truth plugin is synced to the vault
  at the new version.
