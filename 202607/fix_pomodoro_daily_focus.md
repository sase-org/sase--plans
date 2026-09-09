---
tier: tale
title: Fix daily-note activation in two-step Pomodoro navigation
goal: "The Ctrl+9 daily-note fallback actually activates and focuses today's daily note
  (with its in-session cursor and scroll preserved), so the note is visibly selected
  after the first press and the second press reliably performs the Pomodoro jump instead
  of silently doing nothing.

  "
create_time: 2026-09-09 19:53:13
status: wip
---

# Plan: Fix daily-note activation in two-step Pomodoro navigation

## Root-cause diagnosis (verified, do not re-derive)

The two-step `<Ctrl+9>` behavior shipped in `bob-ledger-tools` 1.1.2 (linked
`bob-plugins` repo, commit `9be8ae4`) is broken in the field: after the fallback "jump"
to today's daily note the page is not selected (focus appears stuck on frontmatter
properties or nowhere), the remembered cursor is not visible, and a second `<Ctrl+9>`
does nothing. The deployed vault copy was hash-verified to match the repo, so this is a
code bug, not a stale deploy.

The root cause is in `activateMarkdownLeaf()` in `plugins/bob-ledger-tools/main.js`: it
treats `workspace.revealLeaf` and `workspace.setActiveLeaf` as _alternatives_
(`revealLeaf` else `setActiveLeaf`). Real Obsidian always provides `revealLeaf`, and
`revealLeaf` only brings the leaf's tab to the foreground — it does not update
`workspace.activeLeaf`, does not fire `active-leaf-change`, and does not move keyboard
focus into the editor. The reference implementation this design was modeled on —
`focusWorkspaceLeaf()` in `bob-navigation-hotkeys/main.js` — calls `revealLeaf` _and
then_ `setActiveLeaf(leaf, { focus: true })` (with a `leaf.focus()` fallback); the
ledger port collapsed that sequence into an either/or.

Consequences, matching the reported symptoms exactly:

1. First press from a note with no Pomodoro target while today's daily note is already
   open in another tab (the common case): `openTodayDailyNoteWithState()` takes the
   reused-leaf branch, `revealLeaf` shows the daily tab, but the previous note's leaf
   remains the workspace's active leaf and no editor has keyboard focus — the page looks
   unselected and no cursor is visible.
2. Second press: the command callback reads `getActiveViewOfType(MarkdownView)`, which
   still returns the _previous_ note's view, finds no target there, and re-runs the
   daily fallback, which `revealLeaf`s an already-front tab and returns silently
   (`reusedOpenLeaf` path shows no notice). Every subsequent press repeats this no-op.
3. Independently, `openDailyFallbackOnly()` never explicitly focuses the daily editor in
   any branch; the fresh-open path only appears focused because the core Daily Notes
   command focuses as a side effect.

The test suite passed 10/10 because the harness is unfaithful on exactly this point:
`TestWorkspace.revealLeaf` in `scripts/test-ledger-tools-navigation.cjs` activates the
leaf and emits `active-leaf-change`/`file-open`, which real `revealLeaf` does not do.

## Fix design

Work in the linked `bob-plugins` repository (open it with the `/sase_repo` skill). All
behavior from the approved two-step plan remains the contract; this plan only repairs
activation, focus, and test fidelity.

- Rework `activateMarkdownLeaf()` to mirror `bob-navigation-hotkeys`'
  `focusWorkspaceLeaf()` semantics: best-effort `revealLeaf` first (for
  sidebar/collapsed cases), then _always_ activate with
  `setActiveLeaf(leaf, { focus: true })`, tolerating older `setActiveLeaf(leaf)`
  signatures and falling back to `leaf.focus()` when needed, then keep the existing
  bounded wait for the leaf's Markdown view. Activation failure should still return null
  so callers report "Could not open daily note" instead of silently succeeding.
- In `openDailyFallbackOnly()`, after a successful open (both the reused-leaf and
  fresh-open branches), explicitly focus the daily editor via the existing
  `focusEditor()` helper so keyboard focus lands in the editor body even when the
  opening mechanism (e.g. the core Daily Notes command) leaves focus on the properties
  widget. Focus must not disturb the view: keep the existing order of
  snapshot-remembered-location → open → activate/focus → restore-or-capture, and do not
  let focus events count as capture-worthy updates (capture already reacts only to
  selection/doc/viewport changes — preserve that).
- Preserve the contract details: a reused live leaf keeps its live cursor and scroll (no
  restore); a closed-then-reopened daily note restores the remembered in-session
  location; the no-target notice still fires only when the daily note is already active;
  a rapid second press still cancels pending restoration. The standalone "Open today's
  daily note" command shares `openTodayDailyNote()` and must also end with the daily
  leaf active — verify it does not regress.

## Tests

- Fix harness fidelity first, in `scripts/test-ledger-tools-navigation.cjs`: model
  `revealLeaf` as reveal-only (record the call; do not change `activeLeaf`, emit no
  events, move no focus) and let `setActiveLeaf(leaf, { focus })` be the only activation
  path (sets `activeLeaf`, emits `active-leaf-change`/`file-open`, records focus,
  honoring the `focus` option). Track editor/leaf focus so assertions can distinguish
  "revealed" from "active and focused".
- Add regression coverage that fails against the 1.1.2 logic under the corrected harness
  before the fix is applied:
  - Reused-leaf fallback ends with the daily leaf as the workspace's active leaf _and_
    its editor focused, live cursor/scroll undisturbed.
  - A second press after the fallback performs the actual Pomodoro jump (guards against
    the silent no-op loop).
  - Fresh-open fallback ends focused with the remembered location restored.
- Update any existing assertions that relied on `revealLeaf` activating leaves, and keep
  the full existing scenario matrix green (target jump, deferred no-target notice, race
  cancellation, cleanup, clamping).

## Release and deployment

- Bump `bob-ledger-tools` to 1.1.3 in its manifest and keep the README plugin-version
  table synchronized.
- From the linked repository checkout run the focused ledger navigation tests, the
  complete `npm test` suite, and `npm run validate`; all must pass.
- Deploy with `bob plugins sync -p bob-ledger-tools`, passing `-r` pointed at the linked
  checkout's own path (the command's default source directory does not exist in
  workspace environments — never sync from any other source). Verify the deployed
  `main.js` and `manifest.json` match the tested sources byte-for-byte. Note in the
  final report that Obsidian must reload the plugin to pick up the new `main.js`.
- Commit only the files this fix touches using the `sase_git_commit` workflow after
  validation and deployment succeed.

## Boundaries and risks

- Do not change Pomodoro target selection, the remembered-location data model, or the
  session-only scope of the location map. This plan is activation/focus repair plus
  test-fidelity repair only.
- `setActiveLeaf` fires `active-leaf-change`, which runs the plugin's own
  capture/cleanup handler. The existing snapshot-before-open ordering protects the
  remembered location; keep it, and make sure activation events cannot cancel a restore
  targeting the same daily path.
- Focusing the editor interacts with Vim mode (codemirror-vim); entering the editor in
  normal mode at the preserved cursor is the expected outcome. Do not add extra
  scrolling on focus.
