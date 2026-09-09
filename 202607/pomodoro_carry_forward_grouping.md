---
tier: tale
title: Group `#`-marked Pomodoro links below worked-on links on carry-forward
goal:
  Ctrl+Enter carries worked-on (unmarked) task-link sub-bullets into the new Pomodoro
  first and deferred `#`-marked links after them, each group in source order, instead of
  interleaving both kinds in plain source order.
create_time: 2026-09-09 19:53:18
status: wip
---

# Group `#`-marked Pomodoro links below worked-on links on carry-forward

## Goal

Change how `Ctrl+Enter` (Pomodoro completion in the `task-status-cycler` Obsidian
plugin) orders the task-link sub-bullets it carries into the freshly created `- [ ] ()`
Pomodoro.

Today both kinds of carried link are emitted in plain source order, interleaved:

- an **unmarked** task block-link sub-bullet is _copied_ forward and leaves a
  `🍅`-marked history entry behind under the now-completed Pomodoro (it was worked on);
- a **`#`-marked** sub-bullet (`[[...]]#`, the move-only directive) is _moved_ forward —
  its source bullet is removed and no `🍅` history entry remains (it was deferred, never
  worked on).

The new Pomodoro must instead group them: **worked-on (unmarked, `🍅`) links first, then
the deferred `#`-marked links**, with each group keeping its own relative source order.

Given:

```markdown
## Pomodoros

- [ ] (**1110-1135** [t:: 25m])
  - [[Tasks#^ordinary-one|Ordinary one]]
  - [[Projects/Focus#^move-one|Move one]]#
  - keep this note
  - [[#^move-two]]#
  - [[Tasks#^ordinary-two|Ordinary two]]
```

`Ctrl+Enter` must produce:

```markdown
## Pomodoros

- [x] (**1110-1135** [t:: 25m])
  - 🍅 [[Tasks#^ordinary-one|Ordinary one]]
  - keep this note
  - 🍅 [[Tasks#^ordinary-two|Ordinary two]]
- [ ] ()
  - [[Tasks#^ordinary-one|Ordinary one]]
  - [[Tasks#^ordinary-two|Ordinary two]]
  - [[Projects/Focus#^move-one|Move one]]
  - [[#^move-two]]
```

Note that only the _new_ Pomodoro is regrouped. The completed Pomodoro's own history
block keeps its original source order (`keep this note` stays between the two `🍅`
entries).

## Context

Open the `bob-plugins` linked repo with the `/sase_repo` skill before reading or editing
anything, and use the path it prints as the only path for that repo. All file paths
below are relative to that checkout.

The relevant code is `buildPomodoroCompletionPlan()` in
`plugins/task-status-cycler/main.js`. It builds the carried lines from the two bullet
groups produced by `classifyPomodoroSubBullets()`:

- `sourceBullets.copyableTaskLinkBullets` — live, non-embedded, non-struck task block
  links (the worked-on ones). Their carried text is
  `stripPomodoroMarkersFromLine(bullet.lineText)`.
- `sourceBullets.moveOnlyTaskLinkBullets` — bullets whose standalone body ends in the
  strict `]]#` directive. Their carried text is `bullet.destinationLineText` (the same
  line with only the directive `#` removed).

Both arrays are already populated in ascending source-line order, because
`classifyPomodoroSubBullets()` walks the sub-bullet range top-down. The current code
merges them and re-sorts by `line`, which is exactly the interleaving this plan removes:

```js
  const copyableBulletLines = [
    ...sourceBullets.copyableTaskLinkBullets.map((bullet) => ({ ... })),
    ...sourceBullets.moveOnlyTaskLinkBullets.map((bullet) => ({ ... })),
  ]
    .sort((left, right) => left.line - right.line)
    .map((bullet) => bullet.lineText);
```

The resulting `copiedBulletLines` are inserted verbatim beneath the new
`POMODORO_PLACEHOLDER_LINE` by the single `insertLines` edit at `sourceRange.endLine`.

## Design

1. Regroup the carried lines in `buildPomodoroCompletionPlan()`
   (`plugins/task-status-cycler/main.js`).
   - Replace the merge-then-sort-by-line construction of `copyableBulletLines` with a
     two-group concatenation: all copyable (unmarked) bullet texts first, then all
     move-only (`#`-marked) destination texts.
   - Sort each group by source `line` ascending _before_ concatenating rather than
     relying on the implicit ordering of `classifyPomodoroSubBullets()`, so the grouping
     stays correct if classification order ever changes.
   - Replace the existing
     `// Only insert a fresh placeholder Pomodoro when there is something to carry forward ...`
     comment block's ordering assumptions with a short comment stating the new rule and
     its rationale: the fresh Pomodoro reads as "keep going on what you were working on,
     then pick up what you deferred".

2. Keep everything else about completion planning byte-for-byte identical.
   - The completed Pomodoro's own sub-bullet block is untouched by this change: `🍅`
     normalization still happens in place, note bullets stay where they are, and
     move-only source bullets are still removed via `removeLine` edits.
   - `shouldCreatePomodoro` still depends only on
     `copyableBulletLines.length > 0 || isLastPomodoro`, so regrouping can never change
     _whether_ a placeholder Pomodoro is created.
   - The `insertLines` edit still targets `sourceRange.endLine` and still inserts the
     same number of lines, so `cursorTargetLine` and the `removedLineCountBefore()`
     arithmetic are unaffected. `Ctrl+Enter` must still land the cursor inside the new
     Pomodoro's `()`.
   - When nothing is carried, the `EMPTY_POMODORO_SUB_BULLET_LINE` fallback still
     applies.
   - Struck/retired links are still excluded from both groups, embedded (`![[...]]`)
     transclusions are still not carried, and `startableNonTranscludedTaskLinkBullets`
     (which drives target-task start side effects) is still derived only from the
     copyable group — none of that changes.
   - Do **not** introduce de-duplication across the two groups. A target that appears
     both as an ordinary bullet and as a `#`-marked bullet must still be carried twice,
     now once in each group.
   - Carried lines keep their own leading whitespace verbatim; regrouping must not
     renormalize indentation or bullet markers.

3. Update and extend the regressions in `scripts/test-task-status-cycler.cjs`.
   - Update
     `test("Pomodoro completion moves marked links in source order before a later Pomodoro", ...)`
     — the only existing test that asserts interleaved output. Rename it to describe
     grouping instead of source order, change the expected `plan.copiedBulletLines` to
     `ordinary-one, ordinary-two, move-one, move-two`, and update the expected applied
     editor text for the new Pomodoro. Its other assertions must survive unchanged:
     `plan.createdPomodoro` is still `true`, `plan.cursorTargetLine` is still `5`, the
     final cursor is still `{ line: 5, ch: 7 }`, exactly one `- [ ] ()` exists, no `]]#`
     survives, the moved link gains no `🍅`, and the completed Pomodoro's history block
     is still `🍅 ordinary-one`, `keep this note`, `🍅 ordinary-two` in that order.
   - Add a focused `buildPomodoroCompletionPlan()` test for a `#`-marked bullet
     appearing _above_ unmarked bullets with several members in each group, asserting
     the exact grouped `copiedBulletLines` and that the completed Pomodoro's own block
     still reflects source order.
   - Add a test for a duplicate target carried as both an ordinary and a `#`-marked
     bullet, asserting both copies survive with the ordinary one in the first group and
     the marked one in the second.
   - Add single-group tests (all links marked; all links unmarked) asserting the carried
     order is unchanged from source order, since only mixed Pomodoros are affected.
   - Add an end-to-end `completeActivePomodoroTask` / `Ctrl+Enter` regression that
     exercises the exact Goal example above through the editor, including the retained
     `keep this note` bullet.
   - Leave the following green without edits, as guards that nothing else moved:
     `"Pomodoro completion marks originals and carries clean live copies"`,
     `"retired Pomodoro links are not copied into the next Pomodoro"`,
     `"full Pomodoro completion moves a marked same-note link without history"`,
     `"child-line Ctrl+Enter produces the same rollover and cursor target as parent completion"`
     (its source already lists the unmarked link first, so its expected output must not
     change), and
     `"Ctrl+Enter preserves cross-file move-only targets while ordinary duplicates still start"`.

4. Release and deploy.
   - Bump `plugins/task-status-cycler/manifest.json` from `1.7.0` to `1.7.1` — a
     user-visible behavior refinement with no new syntax, matching the repo's precedent
     for behavior-only Pomodoro fixes.
   - Update the Task Status Cycler row in `README.md` to the new version and mention
     that deferred `#`-marked links roll forward beneath the worked-on links.
   - Run `npm test` and `npm run validate` in the `bob-plugins` checkout; both must
     pass.
   - Deploy with
     `bob plugins sync -p task-status-cycler -r <bob-plugins checkout path> --dry-run`
     first, then rerun without `--dry-run`. The `-r` flag is required because the
     command otherwise defaults to a source path that does not exist in a SASE
     workspace. Tell the user they must reload the plugin in Obsidian to pick up the new
     `main.js`.

## Out of scope

- The order of the completed Pomodoro's own history block. It stays in source order.
- `buildPomodoroMoveOnlyTogglePlan()` and the `#`-toggle keymap. Marking and unmarking a
  link in place is unchanged.
- The `block-id-prompt` plugin's link completion and future-Pomodoro duplicate pruning.
- Any sort key other than the marked/unmarked grouping (alphabetical, task text,
  priority, due date).
- Any `bob-cli` command or option change.

## Expected files

- `plugins/task-status-cycler/main.js`
- `scripts/test-task-status-cycler.cjs`
- `plugins/task-status-cycler/manifest.json`
- `README.md`

All in the `bob-plugins` linked repo, opened via the `/sase_repo` skill.
