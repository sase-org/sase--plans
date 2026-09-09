---
tier: tale
title: Make the bullet-property picker single-instance
goal:
  One Ctrl+Shift+P press — counted or bare — opens exactly one Set bullet property
  picker, and after the user completes or dismisses the picker no duplicate picker
  remains on screen, with counted batch sessions still honored.
create_time: 2026-09-09 19:53:03
status: wip
---

# Plan: Make the bullet-property picker single-instance

## Context

A counted `N<Ctrl+Shift+P>` press in vim normal mode can leave a duplicate **Set bullet
property** menu on screen: after the user finishes or dismisses the picker they
interacted with, an identically rendered second picker remains beneath it. Bare
(uncounted) presses do not reproduce the problem. This is the same class of bug just
fixed for Ctrl+Shift+M in the task-move destination picker (plan
`202607/task_move_picker_single_instance.md`, shipped in Bob Navigation Hotkeys
1.13.10), and diagnosis of `plugins/bob-navigation-hotkeys/main.js` in the linked
`bob-plugins` repository confirms the same unreconciled dual-open race — with one twist
that explains why the earlier fix's analysis missed it.

That earlier plan judged the counted bullet-property feature safe because its
`set-bullet-property` command registers no plugin-default hotkey. But the vault's
user-level `hotkeys.json` binds Ctrl+Shift+P to
`bob-navigation-hotkeys:set-bullet-property`, which recreates exactly the two-path race
at the Obsidian keymap level:

- On a counted press, the plugin's global capture-phase keydown listener
  (`handleCountedBulletPropertyPhysicalKeydown`) claims the chord and opens the counted
  picker, while Obsidian's keymap dispatcher independently runs the user-bound command,
  whose `editorCallback` opens a bare single-bullet picker. The capture handler's
  `stopImmediatePropagation` only suppresses Obsidian's dispatcher when the plugin's
  listener happens to run first — listener-registration order inside Obsidian is
  undocumented and uncontrollable. When Obsidian's dispatcher runs first, both paths
  fire and two pickers stack. The capture handler still opens its counted picker in that
  order because it resolves the editor from `event.target`, which stays valid for the
  whole dispatch even after the bare modal has stolen focus.
- On a bare press the capture handler deliberately falls through (no explicit pending
  vim count), leaving only Obsidian's dispatch path — one picker. This matches the
  observed count-only reproduction.
- `openBulletPropertyPicker` has no idempotence guard: every call unconditionally
  constructs and opens a fresh `BulletPropertyPickerModal`. The task-move fix added a
  single-instance guard only to
  `openTaskMoveDestinationPicker`/`TaskMoveDestinationPickerModal`; the bullet-property
  flow received none.
- Unlike the fixed counted task-move handler,
  `handleCountedBulletPropertyPhysicalKeydown` never checks `event.repeat`, so a held
  chord can emit additional keydowns; once the first press consumes the vim count, the
  repeats fall through as bare presses to Obsidian's dispatcher and can stack further
  pickers.

Because the stacked pickers render identically and overlap exactly, the duplicate is
invisible until the top picker closes — the reported symptom. The fix must be
dispatch-order-agnostic, mirroring the proven task-move approach: make the
bullet-property picker single-instance at the plugin level so every duplicate-open
mechanism collapses to one visible picker, while a counted request still wins over a
bare duplicate from the same physical keypress.

The linked `bob-plugins` repository is the source of truth. Open it through
`sase repo open bob-plugins` before implementation, make all plugin changes there, and
deploy the completed plugin through `bob plugins sync`; do not edit the vault's
installed plugin files directly.

## Implementation

1. Add a single-instance guard for the bullet-property picker in
   `plugins/bob-navigation-hotkeys/main.js`, mirroring the existing
   `activeTaskMoveDestinationPicker` pattern.
   - Track the currently open `BulletPropertyPickerModal` on the plugin instance,
     initialized alongside the task-move tracking field. Clear the tracking whenever the
     modal closes through any path by routing the clearing through the modal's existing
     `onClose` lifecycle (which already drops pending batch state), so Escape,
     programmatic closes, and stage-completion closes all reset it; also clear it if
     `open()` throws after tracking was set.
   - When `openBulletPropertyPicker` is invoked while a tracked picker is already open,
     reconcile instead of stacking: if the new request carries an explicit vim count and
     the tracked picker's session does not (a counted session is one whose task session
     is explicit; bare bullet invocations may have no task session at all), close the
     tracked bare picker and open the counted replacement — this is the same-keypress
     duplicate where Obsidian's user-bound hotkey dispatched first and the capture
     handler supplied the count second. Otherwise ignore the duplicate request and keep
     the existing picker. A single physical keypress must always yield exactly one
     visible picker whose session reflects the explicit count when one was typed.
   - Perform the reconciliation before the validity notices so an ignored duplicate does
     not emit a spurious "Cursor is not on a bullet"-style notice over an
     already-correct open picker.
   - Keep every current entry point available and user-visible: the command palette
     entry, the vault's Ctrl+Shift+P hotkey binding dispatched by Obsidian, and the
     counted vim-normal-mode capture handler all continue to work. Bare Ctrl+Shift+P
     behavior outside the duplicate scenario is unchanged.
   - Leave the picker's stage flow, counted target discovery, aggregation, batch
     planning, stale-snapshot guards, project-note `scheduled` semantics, `dependsOn`
     flows, notices, and transactional apply behavior unchanged.

2. Ignore auto-repeated keydown events in `handleCountedBulletPropertyPhysicalKeydown`.
   - Return without handling (and without swallowing the event) when `event.repeat` is
     true, before any vim-state reads or picker opens, matching the counted task-move
     handler's convention. Any duplicate that still arrives through the command path
     collapses in the single-instance guard.

3. Add focused regressions in `scripts/test-navigation-hotkeys.cjs` using the existing
   plugin/modal mock harness, modeled on the task-move single-instance coverage.
   - Duplicate opens within one dispatch: a bare open followed by a counted open leaves
     exactly one open picker whose session carries the count; bare-then-bare keeps the
     first picker and ignores the duplicate; counted-then-bare keeps the counted picker.
   - Close clears tracking: after the picker closes via Escape and via a completed
     property operation, a subsequent open succeeds and captures a fresh session.
   - An `event.repeat` keydown is declined by the physical handler without swallowing
     the event and opens nothing, while an otherwise identical non-repeat counted
     keydown still opens the picker and consumes the vim count.
   - Existing counted bullet-property target-discovery, batch-planning,
     physical-dispatch, and unrelated picker coverage must remain green and unmodified
     in intent.

4. Release, validate, and deploy the corrected plugin.
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.13.10` to the next
     patch version and keep the Bob Navigation Hotkeys version in `README.md`
     synchronized.
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

- A counted `N<Ctrl+Shift+P>` press in vim normal mode produces exactly one visible Set
  bullet property picker whose session reflects the requested count, regardless of
  whether Obsidian's hotkey dispatch or the plugin's capture handler fired first;
  completing or dismissing that picker leaves no second picker behind.
- A bare Ctrl+Shift+P press keeps its current behavior in vim normal mode, insert mode,
  and non-vim editing, and never yields more than one picker.
- Holding the chord long enough to auto-repeat does not stack additional pickers.
- The command palette entry keeps its current behavior, and all counted batch guarantees
  (all-or-nothing application, stale-snapshot rejection, notices) are unchanged.
- Focused and full test suites pass, all plugin manifests validate, the manifest and
  README versions agree, and the updated source-of-truth plugin is synced to the vault
  at the new version.
