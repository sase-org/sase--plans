---
tier: tale
title: Add Vim counts to Pomodoro movement and open-task navigation
goal:
  Ctrl+Shift+J/K consumes a Vim count for exact atomic Pomodoro reordering or circular
  counted task navigation while preserving bare behavior.
size: medium
proposed_by: bbugyi200.athena.0ff.f0
create_time: 2026-09-09 19:59:58
status: wip
---

# Plan

## Goal

Teach the existing `Ctrl+Shift+J` / `Ctrl+Shift+K` Vim-normal-mode hotkeys to consume a
numeric Vim count:

- On a movable planned Pomodoro entry, `N<Ctrl+Shift+J/K>` moves that whole Pomodoro
  block exactly `N` positions down/up among adjacent planned Pomodoros.
- Everywhere else, the same count jumps to the `N`th next/previous eligible open-task
  navigation target, retaining the current circular wrap behavior.
- A bare chord remains a one-step operation and behaves exactly as it does now.

This is one bounded feature in one plugin, with one shared input route and one test
suite, so it should be implemented by one follow-up coding agent. All source changes
land in the **`bob-plugins`** linked repository; `bob-cli` needs no source changes.

Open the linked repository first with the `/sase_repo` skill:

```bash
sase repo open bob-plugins -r "Implement counted Ctrl+Shift+J/K Pomodoro movement and task navigation"
```

Use the path printed by that command for all reads, writes, tests, and deployment below.
Paths in this plan are relative to that checkout root.

## Existing behavior and reusable machinery

The implementation is in `plugins/bob-navigation-hotkeys/main.js` and its tests are in
`scripts/test-navigation-hotkeys.cjs`.

The two Obsidian commands retain the ids `jump-to-next-open-task` and
`jump-to-prev-open-task`. Their editor callbacks call
`jumpToOpenObsidianTask(editor, 1)` and `jumpToOpenObsidianTask(editor, -1)`. In Vim
normal mode, CodeMirror consumes these physical chords before Obsidian can dispatch
them, so the plugin's capture-phase `handleOpenTaskJumpPhysicalKeydown` fallback calls
the same method. The shared method is protected by the existing per-editor/per-direction
macrotask dispatch guard so one physical event cannot apply through both routes.

The previous change added a second behavior to that shared route: when the cursor is
exactly on an open placeholder Pomodoro entry in the first `## Pomodoros` section,
`movePlannedPomodoroEntry` uses `planPomodoroEntryReorder` to swap the whole entry and
its owned child block with one adjacent planned entry. It refuses without jumping when
that neighbor is missing or is current, closed, or cancelled. Every non-movable cursor
context falls through to the original circular task/Pomodoro navigation computed by
`getOpenObsidianTaskJumpLine`.

Count infrastructure already exists and should be reused:

- `getPendingVimRepeat(cm)` reads a positive numeric prefix from CodeMirror Vim's
  pending `inputState`, returning `{ repeat, explicit }` and defaulting to
  `{ repeat: 1, explicit: false }`.
- `resetPendingVimInputState(cm, reason)` consumes the pending Vim prefix safely.
- `normalizeVimRepeat(value)` normalizes invalid, missing, zero, or negative repeat
  values to `1`.
- `resolveVimCodeMirror`, `isVimNormalModeEditor`, and the physical count handlers for
  `Ctrl+Shift+M`, `Ctrl+Shift+P`, and `!` demonstrate the established
  capture/read/reset/dispatch pattern.

Unlike the `Ctrl+Shift+M/P` batch APIs, J/K counts are ordinary Vim repeats, not "N
additional items": `3<Ctrl+Shift+J>` means three positions/targets total. Counts exist
only in Vim normal mode. Insert/visual/replace mode and non-Vim editing continue through
Obsidian's command route with the implicit repeat of one.

## Count semantics

### Planned Pomodoro reordering

A counted reorder is exact and atomic. A repeat of `N` succeeds only if all `N` Pomodoro
entries crossed in the requested direction are themselves open placeholders. If the
contiguous planned run does not contain that many siblings, or a
current/closed/cancelled entry blocks the route, refuse the whole request:

- do not partially reorder;
- do not create an editor transaction or undo group;
- do not move the cursor;
- show one count-aware refusal notice; and
- do not fall through to task navigation.

This preserves the invariant that planned Pomodoros cannot cross non-planned entries and
preserves the current refusal-as-no-mutation contract. Repeat `1` must retain the
existing refusal strings byte-for-byte. For repeat greater than one, use a truthful
message such as
`BODY cannot move down 3 positions without crossing the last planned Pomodoro` (with
`up`/`first` in the reverse direction).

On success, all crossed planned entries retain their relative order while the source
block moves into the requested destination slot. Child bullets and all their descendants
move with their owning entry. Blank lines or stray prose between entries remain attached
to their physical gaps, matching the result of performing the current adjacent swap `N`
times. Apply the final content in one editor transaction/undo group and keep the cursor
on the moved entry. Preserve the existing success notice for repeat `1`; for a larger
repeat report the distance, for example `Moved BODY down 3 positions`.

### Circular open-task navigation

Outside the movable-entry route, a repeat of `N` selects the `N`th eligible target in
the requested direction. Eligibility remains exactly the current
`getOpenTaskNavigationLines` contract: Ready/In Progress/Next `#task` lines plus the
accepted top-level Pomodoro ledger lines, with frontmatter/fences skipped and Blocked
tasks excluded.

Advance circularly, so counts can wrap once or many times. Determine the first target
with the current strict-neighbor-then-wrap rule, then advance another `N - 1` positions
modulo the number of targets. Preserve these edge cases:

- no eligible target still returns `null`;
- when the sole eligible target is already under the cursor, every repeat still returns
  `null` and produces the existing next/previous notice;
- when a sole target is elsewhere, any positive repeat lands on it;
- with multiple targets, a count equal to a complete cycle may legitimately land back on
  the starting target and is still a successful counted jump;
- successful counted jumps set column `0` and schedule exactly one existing `zz`-style
  center operation.

## Implementation

### 1. Extend the pure jump helper with a repeat

Change `getOpenObsidianTaskJumpLine` to accept an optional fourth argument,
`repeat = 1`, normalized with `normalizeVimRepeat`.

Keep target discovery untouched. Resolve the index of the first strict next/previous
target exactly as today, including wrap, and retain the existing sole-current-target
`null` check. From that first index, apply the remaining `repeat - 1` steps modulo
`taskLines.length`; avoid repeated rescans or a loop proportional to an arbitrarily
large typed count. The default repeat must yield the exact current result for every
existing call and test.

`getOpenObsidianTaskJumpLine` is already exported through `module.exports.helpers`; no
new export is needed.

### 2. Extend `planPomodoroEntryReorder` for an exact repeat

Add `options.repeat`, normalize it with `normalizeVimRepeat`, and include the normalized
`repeat` in every frozen result. Keep all current context, staleness, and movable-source
validation unchanged.

Compute `targetEntryIndex = entryIndex + direction * repeat`. The plan is valid only
when that target exists and every entry between the source and target, inclusive of the
target and exclusive of the source, passes `isMovablePomodoroEntryContext`'s underlying
open-placeholder rule. Do this validation before building output so an impossible
counted request always returns `after === content` and `movedEntryLine === null`.
Preserve the current repeat-1 boundary error exactly; use the count-aware refusal
described above for larger repeats.

For a valid plan, treat every affected entry as an owned block
`[entryLine, childEndLineExclusive)` and capture each gap between consecutive blocks
separately. Rotate only the affected blocks:

- moving down rotates `[source, next1, ..., nextN]` left to
  `[next1, ..., nextN, source]`;
- moving up rotates `[prevN, ..., prev1, source]` right to
  `[source, prevN, ..., prev1]`.

Re-interleave the original gaps by slot between the rotated blocks, leaving all lines
before and after the affected span untouched. Track `movedEntryLine` while rendering the
rotated span rather than assuming equal block sizes. Keep `neighborEntry`
backward-compatible for repeat `1`; for a larger repeat let it refer to the entry
originally occupying the final destination slot and document that meaning in the helper
comment. Total line count and original LF/CRLF line ending are invariants.

### 3. Thread repeat through the runtime move/jump route

Change `movePlannedPomodoroEntry(editor, direction, repeat = 1)` to normalize and pass
the repeat into the planner. Do not loop editor mutations. Continue to use one
`applyEditorContentTransaction` call, its post-write verification, and the planner's one
final cursor position. Preserve all tri-state routing:

- `false` only when the cursor is not on a movable planned entry, so navigation may run;
- `true` for both successful moves and refusals, so a counted planned-Pomodoro chord
  never falls through to a jump.

Keep repeat-1 success text exactly `Moved <label> up/down`; use
`Moved <label> up/down N positions` for repeat greater than one.

Change `jumpToOpenObsidianTask(editor, direction, repeat = 1)` to normalize the repeat
once, retain the dispatch guard in its current position, pass the repeat to
`movePlannedPomodoroEntry`, and otherwise pass it to `getOpenObsidianTaskJumpLine`. The
guard should remain keyed by editor and direction rather than repeat: its purpose is to
suppress the duplicate command route for the same physical chord, including when the
capture route carries a count and a duplicate Obsidian callback would default to one.

The two command ids and editor callbacks need no behavior change: calls without a third
argument intentionally keep repeat `1`, preserving the vault's existing bindings and all
non-Vim behavior.

### 4. Consume the pending count in the physical J/K fallback

In `handleOpenTaskJumpPhysicalKeydown`, after resolving the focused view and confirming
Vim normal mode, resolve CodeMirror and call `getPendingVimRepeat(cm)`. Only after the
event has been accepted and marked handled:

1. prevent its default/propagation exactly as today;
2. consume the Vim input with `resetPendingVimInputState`, using a diagnostic reason
   such as `counted-open-task-jump` for an explicit prefix and `open-task-jump`
   otherwise; and
3. call `jumpToOpenObsidianTask(view.editor, direction, pendingRepeat.repeat)`.

Keep the same-event `WeakSet` protection. Do not add the `event.repeat` rejection used
by picker-opening keymaps: J/K currently permits deliberate presses and OS key repeat,
and after the first counted event consumes the prefix any subsequent auto-repeat event
should naturally act as a bare one-step chord. Non-matching modifiers, unfocused
editors, disabled Vim, and insert/visual/replace modes must continue to fall through
without consuming Vim state.

Update comments around the input fallback and shared dispatch route to describe counted
moves/jumps rather than one-step-only behavior.

## Tests

Add focused coverage in `scripts/test-navigation-hotkeys.cjs`, preserving all existing
tests unchanged unless a signature assertion must explicitly exercise the new optional
repeat.

### Pure navigation helper

1. Existing uncounted forward/backward, skipped-Blocked, and wrap assertions still pass
   with the omitted repeat.
2. A repeat of `2` and `3` selects the second/third eligible target in both directions,
   including when the cursor starts on a target and when it starts between targets.
3. Counted navigation skips Blocked tasks and wraps across the file boundary; a count
   larger than the target list works modulo its length.
4. A full-cycle count with multiple targets may return the starting line, while a sole
   current target remains `null` for both directions and large counts.
5. A sole target away from the cursor is selected for any positive count, and
   missing/invalid/nonpositive repeat values retain the repeat-1 behavior.

### Pure Pomodoro reorder planner

6. A named planned Pomodoro moves down three positions in one plan, carrying a
   differently-sized child subtree; crossed siblings keep their order and
   `movedEntryLine`, `neighborEntry`, `direction`, and `repeat` are correct.
7. The symmetric multi-position upward move works, and moving down then back up by the
   same repeat restores the original bytes.
8. Multiple unequal blocks and multiple distinct gaps (blank lines and stray prose)
   demonstrate that blocks rotate while gaps retain their slots.
9. A multi-position request that runs off the planned run or encounters a current,
   closed, or cancelled entry is refused atomically with the count-aware error,
   `after === content`, and no destination line.
10. Repeat `1` retains the existing output, metadata, boundary errors, and all current
    planner tests; invalid repeat values normalize to one.
11. Multi-position plans preserve CRLF and line count and leave everything outside the
    affected span/first `## Pomodoros` section byte-for-byte intact.

### Runtime and physical dispatch

12. `jumpToOpenObsidianTask(editor, direction, 3)` on a movable entry applies exactly
    one transaction and one undo group, lands the cursor on the moved entry with `ch`
    clamped, and emits one counted success notice.
13. An impossible counted move performs no transaction, leaves content/cursor unchanged,
    emits one count-aware refusal, returns handled, and never invokes navigation.
14. The same runtime call from a current/closed Pomodoro entry, a Pomodoro child, a task
    line, and a plain line follows the counted jump helper, mutates no content, and
    schedules only the existing single center operation.
15. A physical normal-mode `3<Ctrl+Shift+J>` and the reverse K chord read the pending
    Vim repeat, consume/reset `inputState`, stop the event once, and pass repeat `3` to
    the shared route. A bare chord passes repeat `1`; delivering the same event through
    window and document still dispatches once.
16. Non-normal modes, missing Vim state, a non-editor target, and wrong modifier
    combinations do not consume/reset pending input and leave the event for
    Obsidian/CodeMirror.
17. The macrotask dispatch guard still permits exactly one counted reorder when the
    capture route and a default-repeat command route both reach the shared method; no
    second move or notice occurs.
18. Preserve the existing no-target notices, uncounted move/refusal notices, command
    ids, one-step navigation behavior, and deferred centering tests.

Use the existing `TransactionEditor`, module-level `notices`, and plugin stubs. For
physical input tests, mirror the existing counted M/P/`!` fixtures with
`cm.state.vim.inputState`, and override `getFocusedMarkdownEditorView` where that keeps
the fixture independent of Obsidian's `MarkdownView` constructor.

Run from the `bob-plugins` repository root:

```bash
npm test
npm run validate
```

## Documentation, release, and deployment

1. Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.30.0` to `1.31.0`
   (feature release, no command-id or compatibility break).
2. Update the `bob-navigation-hotkeys` row in `README.md` to version `1.31.0` and state
   that `N<Ctrl+Shift+J/K>` moves a planned Pomodoro exactly N positions atomically or
   jumps to the Nth circular navigation target elsewhere. Update the later version
   example from `1.30.0` to `1.31.0` as well.
3. Leave the manifest description alone; it remains accurate and concise.
4. After tests and validation pass, deploy the changed source-of-truth plugin to the
   vault as required by the linked repository:

   ```bash
   bob plugins sync -r "<path printed by sase repo open>"
   ```

## Acceptance criteria

- In Vim normal mode, `3<Ctrl+Shift+J>` / `3<Ctrl+Shift+K>` on a movable planned
  Pomodoro moves its complete owned block exactly three planned positions down/up in one
  undoable edit.
- A counted Pomodoro move is all-or-nothing and can never cross a current, closed,
  cancelled, or section-boundary entry; refusal leaves content and cursor untouched and
  never turns into a jump.
- On every other cursor context, the same prefix jumps to the third eligible
  next/previous task-or-Pomodoro target with circular wrapping, existing target
  filtering, column-zero placement, and one deferred center.
- The prefix is consumed exactly once from CodeMirror Vim input, and duplicate
  physical/Obsidian delivery cannot double-apply the operation.
- Bare chords, explicit count `1`, command ids, non-Vim/mode fallthrough, notices,
  filtering, wrapping, transaction behavior, and centering remain compatible with the
  current implementation.
- Multi-position edits preserve child subtrees, inter-entry gap text, content outside
  the affected span, line count, and LF/CRLF style.
- `npm test` and `npm run validate` pass, and `bob plugins sync` deploys the completed
  plugin.
