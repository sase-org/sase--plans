---
tier: tale
title: Read the Vim count on the Obsidian command route for Ctrl+Shift+J/K
goal:
  Fix silently-dropped Vim counts on Ctrl+Shift+J/K by resolving the pending count
  inside the shared jump/move route instead of only in the capture-phase fallback.
size: small
proposed_by: bbugyi200.athena.0ff.f0.f1
create_time: 2026-09-09 19:59:58
status: wip
---

# Plan

## Goal

`N<Ctrl+Shift+J>` / `N<Ctrl+Shift+K>` currently behave exactly like a bare chord: the
Pomodoro moves one position and the jump lands on the first target, no matter what count
is typed. Make the count actually apply, in a way that does not depend on which of the
two dispatch routes wins the race for a given physical keypress.

All source changes land in the **`bob-plugins`** linked repository; `bob-cli` needs no
source changes. Open the repository first with the `/sase_repo` skill:

```bash
sase repo open bob-plugins -r "Fix dropped Vim counts on Ctrl+Shift+J/K"
```

Use the path printed by that command for all reads, writes, tests, and deployment. Every
path below is relative to that checkout root. The implementation is in
`plugins/bob-navigation-hotkeys/main.js`; its tests are in
`scripts/test-navigation-hotkeys.cjs`.

## Root cause

One physical `Ctrl+Shift+J`/`K` press reaches `jumpToOpenObsidianTask` through **two**
routes, which is why the pre-existing dispatch guard exists at all. Its own comment
(`main.js`, in `jumpToOpenObsidianTask`) states this:

> A single physical Ctrl+Shift+J/K can reach this method twice in the same dispatch
> turn: once via the Obsidian hotkeys.json command and once via the Vim-normal capture
> fallback.

The two routes are:

1. **The Obsidian command route.** `hotkeys.json` binds `Ctrl+Shift+J`/`K` to
   `bob-navigation-hotkeys:jump-to-next-open-task` / `jump-to-prev-open-task`, whose
   `editorCallback`s call `this.jumpToOpenObsidianTask(editor, 1)` and `(editor, -1)` —
   with **no third argument**, so `repeat = 1`.
2. **The capture-phase fallback.** `handleOpenTaskJumpPhysicalKeydown` reads the pending
   Vim count with `getPendingVimRepeat(cm)`, resets the Vim input state, and calls
   `this.jumpToOpenObsidianTask(view.editor, direction, pendingRepeat.repeat)`.

The count is read **only** on route 2. The guard, keyed by `(editor, direction)`, lets
whichever route arrives first win and makes the second one return `false` immediately.

The capture handler calls `preventDefault()`, `stopPropagation()` _and_
`stopImmediatePropagation()` from a `window` capture-phase listener. If route 2 ran
first, route 1 could never also run, and the guard would never have been needed. The
guard exists and predates both recent features (it came in with the initial monorepo
import, `ef15fc6`), so in the live app **route 1 wins**: Obsidian installs its global
keymap handler at app boot, ahead of the plugin's capture listener on the same target
and phase, so the plugin's `stop*` calls arrive too late to prevent the command.

Therefore, in the real vault:

- Route 1 runs first with `repeat = 1` → the Pomodoro moves one position (or the jump
  goes one target), and the guard is marked.
- Route 2 runs second, correctly reads the pending count of `3`, resets the Vim input
  state so the `3` disappears — and is then discarded by the guard.

Net effect: the typed count is consumed and thrown away, and the chord behaves exactly
as it did before counts were added. That is precisely the reported symptom.

Note also that this failure mode is invisible to the current test suite. The test
`dispatch guard allows exactly one counted Pomodoro move when the command route also fires`
only exercises the capture-first ordering (`jump(editor, 1, 3)` then `jump(editor, 1)`),
never the command-first ordering that actually happens.

## Fix

Resolve the pending Vim count **inside the shared entry point**, so whichever route
arrives first reads the still-pending count from CodeMirror Vim. The second route is
suppressed by the existing guard exactly as it is today, so the count can be consumed at
most once per press regardless of ordering.

### 1. Distinguish "no repeat given" from "repeat 1"

Change the signature to `jumpToOpenObsidianTask(editor, direction, repeat)` — drop the
`= 1` default. `undefined`/`null` now means "resolve the count from Vim myself" (the
Obsidian command route); an explicit number still wins verbatim (the capture route, and
every existing caller/test that passes one).

Move the repeat normalization to **after** the dispatch-guard check and marking, so a
duplicate route that the guard suppresses can never consume Vim input state:

```js
if (this.isOpenTaskJumpDispatchPending(editor, direction)) {
  return false;
}
this.markOpenTaskJumpDispatch(editor, direction);

const normalizedRepeat =
  repeat === undefined || repeat === null
    ? this.consumePendingOpenTaskJumpRepeat(editor)
    : normalizeVimRepeat(repeat);
```

Keep the `No active markdown editor` early return, the guard,
`movePlannedPomodoroEntry`, `getOpenObsidianTaskJumpLine`, the column-zero placement,
and the deferred centering exactly where and as they are.

### 2. Add `consumePendingOpenTaskJumpRepeat(editor)`

A new plugin method that reads **and consumes** the pending Vim count for `editor`,
returning `1` whenever there is no explicit normal-mode count:

- Resolve the active view defensively — many existing tests construct the plugin with
  `plugin.app = {}`, so guard the lookup rather than calling `getActiveMarkdownView()`
  unconditionally:

  ```js
  const activeView =
    this.app &&
    this.app.workspace &&
    typeof this.app.workspace.getActiveViewOfType === "function"
      ? this.getActiveMarkdownView()
      : null;
  const view = activeView && activeView.editor === editor ? activeView : null;
  ```

  Scoping to the view that actually owns `editor` keeps a background pane's Vim state
  from leaking into another pane's chord; when it does not match, pass `null` and let
  `resolveVimCodeMirror` fall back to `editor.cm.cm`.

- `if (!this.isVimNormalModeEditor(editor, view)) return 1;` — this already returns
  `false` before touching `this.app` when the editor has no CodeMirror Vim adapter, so
  non-Vim editing, insert/visual/replace mode, and every plain test editor keep repeat 1
  and consume nothing.
- Otherwise resolve `cm` with `resolveVimCodeMirror(editor, view)` and read
  `getPendingVimRepeat(cm)`. When `explicit` is false, return `1` **without** resetting.
  When it is true, call `resetPendingVimInputState(cm, "counted-open-task-jump")` and
  return `normalizeVimRepeat(pending.repeat)`.

Document in a comment why the shared method reads the count itself: the Obsidian
hotkeys.json command route reaches it without a repeat and, in the live app, wins the
race against the capture-phase fallback, so reading the count only in the fallback loses
it.

### 3. Leave the capture-phase fallback's contract intact

`handleOpenTaskJumpPhysicalKeydown` keeps reading, resetting, and passing an explicit
repeat. Its reset is now sometimes redundant (route 1 may already have consumed the
count), which is harmless and still the right behavior when the capture route wins or
when Obsidian's binding is absent. Update its comment to say the shared route also
resolves a count for the command path.

Both command `editorCallback`s keep calling `this.jumpToOpenObsidianTask(editor, 1)` /
`(editor, -1)` with no third argument — that is now load-bearing, so add a short comment
saying the omitted repeat means "resolve the pending Vim count".

### 4. Secondary hardening: a `prefixRepeat` source in `getPendingVimRepeat`

`getPendingVimRepeat` currently reads `inputState.keyBuffer` digits, then falls back to
`inputState.getRepeat()`. Add a third source between them that joins
`inputState.prefixRepeat` and `inputState.motionRepeat` (each an array of digit strings
in CodeMirror Vim) and multiplies them, matching Vim's own `getRepeat()` semantics:

```js
const digits = (value) =>
  Array.isArray(value) ? value.join("") : typeof value === "string" ? value : "";
```

Return `{ repeat: product, explicit: true }` only when the joined text matches
`/^[1-9]\d*$/` for at least one of the two and the product is finite and positive.

This is purely additive — when `getRepeat()` exists it derives from the same two arrays,
so the result is unchanged — and it covers CodeMirror adapters that expose the arrays
but not the method. `task-status-cycler/main.js` carries a byte-identical copy of this
helper; leave that copy alone (its own chords work today) and note the intentional
divergence in a comment on the navigation-hotkeys copy.

## Tests

Extend `scripts/test-navigation-hotkeys.cjs`. Every existing test must keep passing; the
only existing test that should change is the command-id test, which gains a clarified
name/comment while keeping its `repeat: undefined` assertion.

Build a small Vim-editor fixture that wraps the existing `TransactionEditor` with a
`cm: { cm }` property whose `cm.state.vim` is `{ mode, inputState }` and whose
`cm.getCursor` is a function, so `resolveVimCodeMirror` and `isVimNormalModeEditor`
accept it. Reuse `countedPomodoroReorderLines()` and `pomodoroFixtureLines()`.

1. **Command-route count, Pomodoro move.** With `keyBuffer: ["3"]` pending and the
   cursor on a movable planned entry, `plugin.jumpToOpenObsidianTask(editor, 1)` (no
   third argument) moves the entry three positions in exactly one transaction and one
   undo group, matches `planPomodoroEntryReorder(..., { repeat: 3 })`, emits
   `Moved BODY down 3 positions`, and clears `inputState` with reason
   `counted-open-task-jump`.
2. **Command-route count, circular jump.** From a non-movable context the same uncounted
   call lands on `getOpenObsidianTaskJumpLine(lines, cursorLine, direction, 3)`, mutates
   no content, sets column `0`, and schedules exactly one center operation.
3. **Command-first then capture-second ordering** (the real-world sequence): call
   `jump(editor, 1)` with a pending `3`, then `jump(editor, 1, 1)` as the suppressed
   capture route. Exactly one transaction, one undo group, one `Moved ` notice, and the
   three-position result. This test fails on the current code and is the regression this
   plan exists for.
4. **Capture-first then command-second ordering** still works: `jump(editor, 1, 3)` then
   `jump(editor, 1)` gives one three-position move, and the suppressed second call
   consumes nothing — assert the Vim `inputState` it would have read is untouched by the
   guard-suppressed call (i.e. repeat resolution happens after the guard).
5. **No explicit count.** With an empty `keyBuffer` and `getRepeat()` returning null, an
   uncounted command-route call moves/jumps one step and leaves `inputState.reason`
   unset — nothing is reset when there is no count to consume.
6. **Non-Vim and wrong-mode editors.** A plain `TransactionEditor` (no `.cm`), an editor
   in `insert`/`visual`/`visual-line`/`replace` mode, and a plugin whose `app` is `{}`
   all resolve repeat `1`, reset nothing, and reproduce today's behavior byte-for-byte.
7. **Foreign-editor scoping.** When the active view's `editor` is a different object,
   the count is still resolved from the passed editor's own `cm` and no other pane's Vim
   state is touched.
8. **Explicit repeat wins.** `jump(editor, direction, 2)` uses `2` even when a different
   count is pending, and does not re-read or reset Vim state through
   `consumePendingOpenTaskJumpRepeat`.
9. **`getPendingVimRepeat` `prefixRepeat` fallback.** With `keyBuffer` empty, no
   `getRepeat`, and `prefixRepeat: ["1", "2"]`, the helper returns
   `{ repeat: 12, explicit: true }`; with `prefixRepeat: ["2"]` and
   `motionRepeat: ["3"]` it returns `6`; with both empty it returns
   `{ repeat: 1, explicit: false }`.
10. **Preserved behavior.** The existing counted planner/jump-helper tests, the physical
    `Ctrl+Shift+J/K` count test, the uncounted move/refusal notices, the no-target
    notices, the dispatch-guard tests, and the deferred-centering assertions all keep
    passing unchanged.

Run from the `bob-plugins` repository root:

```bash
npm test
npm run validate
```

## Documentation, release, and deployment

1. Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.31.0` to `1.31.1` (bug
   fix; no command-id, binding, or compatibility change).
2. In `README.md`, update the `bob-navigation-hotkeys` version cell (line 16) and the
   later "`bob-navigation-hotkeys` is ahead of the others at `1.31.0`" sentence to
   `1.31.1`. The feature prose already describes `N<Ctrl+Shift+J/K>` correctly and
   becomes accurate once this fix lands, so leave it as-is.
3. Leave the manifest description alone.
4. After tests and validation pass, deploy to the vault:

   ```bash
   bob plugins sync -r "<path printed by sase repo open>"
   ```

5. Tell the user the fix needs an Obsidian reload (or a plugin disable/enable) to take
   effect, and ask them to confirm `3<Ctrl+Shift+J>` on a planned Pomodoro and
   `3<Ctrl+Shift+J>` on an ordinary task line. Automated tests cannot observe the live
   listener ordering that caused this bug, so that manual check is the real acceptance
   signal.

## Acceptance criteria

- In Vim normal mode, `N<Ctrl+Shift+J>` / `N<Ctrl+Shift+K>` apply the count regardless
  of whether the Obsidian command route or the capture-phase fallback reaches
  `jumpToOpenObsidianTask` first.
- The count is consumed from CodeMirror Vim at most once per physical press; the
  guard-suppressed route consumes nothing and performs no second move, notice, or jump.
- Bare chords, an explicit count of `1`, both command ids, non-Vim editing,
  insert/visual/replace mode, notices, Blocked filtering, wrapping, transaction and undo
  grouping, cursor placement, and deferred centering are unchanged.
- Counted planned-Pomodoro moves stay exact and atomic: they never cross a current,
  closed, cancelled, or section-boundary entry, and a refusal leaves content and cursor
  untouched and never becomes a jump.
- A test reproduces the command-route-first ordering and fails against the current code.
- `npm test` and `npm run validate` pass, and `bob plugins sync` deploys the plugin.
