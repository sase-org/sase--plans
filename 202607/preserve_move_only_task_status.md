---
tier: tale
title: Preserve move-only task status on Pomodoro completion
goal: "Ctrl+Enter completes an open Pomodoro and rolls strict `#`-suffixed task block
  links forward without promoting their target tasks to In Progress, while ordinary task
  links retain their existing start behavior.

  "
create_time: 2026-09-09 19:53:18
status: wip
---

# Plan: Preserve move-only task status on Pomodoro completion

## Context and diagnosis

Implement this as one focused change in the linked `bob-plugins` source-of-truth
repository. The affected plugin is `task-status-cycler`, whose Ctrl+Enter Pomodoro path
currently does the following:

1. `completeActivePomodoroTask()` classifies the Pomodoro's child bullets.
2. Embedded task links are completed recursively, and bare non-embedded task links are
   promoted from Todo/Next to In Progress.
3. `buildPomodoroCompletionPlan()` completes the Pomodoro, retains ordinary links as
   history, and moves strict `[[note#^block]]#` bullets into the next Pomodoro without
   history.

The rollover half is already correct. `classifyPomodoroSubBullets()` recognizes a strict
`#`-suffixed link as move-only, but also places the same bullet in
`bareNonTranscludedTaskLinkBullets`. That second classification feeds
`startPomodoroNonTranscludedTaskBullets()`, so the linked Todo or Next task is
incorrectly rewritten to In Progress before the already-correct completion plan removes
and carries the bullet forward.

The intended categories and side effects are:

| Pomodoro child                             | Target-task effect on completion                             | Pomodoro rollover effect                                     |
| ------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Strict ordinary `[[note#^block]]` bullet   | Promote Todo/Next to In Progress                             | Keep marked history and carry a clean copy                   |
| Strict move-only `[[note#^block]]#` bullet | Preserve the target status exactly                           | Remove it from the completed Pomodoro and carry a clean copy |
| Embedded `![[note#^block]]` task           | Preserve existing recursive close/root-only reopen semantics | Preserve existing retirement behavior                        |
| Non-strict hash/prose/strikethrough form   | Preserve its current non-move-only classification            | Preserve current behavior                                    |

## Separate move-only rollover from task startup

Update `plugins/task-status-cycler/main.js` so the classifier exposes an explicitly
startable collection for ordinary bare non-embedded links and keeps move-only bullets
solely in `moveOnlyTaskLinkBullets`. Use the startable collection in
`completeActivePomodoroTask()` when dispatching task promotion. A `#`-suffixed link must
therefore reach the existing remove-and-carry planner without ever reaching target
resolution or the forced `"/"` status write.

Keep `getMoveOnlyPomodoroBlockLinkFromListItem()` and its strict immediate-suffix rules
unchanged. Also retain the ability of `getBareNonEmbeddedBlockLinkTargetFromListItem()`
and `buildPomodoroMoveOnlyTogglePlan()` to recognize both unmarked and already-marked
forms, because normal-mode `#` needs to toggle the directive in either direction. Encode
the distinction at the Pomodoro classifier/consumer boundary rather than weakening the
shared syntax helper.

Preserve these interaction rules:

- An ordinary link still starts a Todo or Next target, including same-note and
  cross-file targets.
- A move-only occurrence alone never mutates its target, whether that target is Todo,
  Next, already In Progress, Done, stale, or unreadable.
- If the same target appears through both an ordinary startable occurrence and a
  move-only occurrence, the ordinary occurrence still authorizes promotion; the
  directive applies to its marked occurrence, not globally to the task.
- Classification occurs before asynchronous target writes and the completion planner
  continues to re-read live editor text afterward, preserving the existing stale-write
  and cursor safeguards.

## Lock the behavior down with regression coverage

Extend `scripts/test-task-status-cycler.cjs` at both the classifier and runtime levels:

- Update the move-only classifier expectations so strict `#` bullets appear in the
  move-only/carry collection but not in the collection consumed by task startup;
  ordinary strict links remain startable.
- Change the existing same-note full-completion regression so its move-only target stays
  Todo while the source link is removed, the clean link is copied to the new Pomodoro,
  ordering is preserved, no tomato history is left for that occurrence, and the cursor
  lands at the same rollover target.
- Update the registered Ctrl+Enter parent-versus-child regression to prove both entry
  points produce identical text and cursor results: the ordinary link's target becomes
  In Progress, while the move-only link's target remains unchanged.
- Add or extend a cross-file runtime case covering the statuses that would otherwise be
  promoted (Todo and Next), so completing through the actual Ctrl+Enter action cannot
  write `[/]` into the linked task file for move-only bullets.
- Cover the mixed duplicate-target rule: an ordinary occurrence may still start the task
  even when another occurrence for the same block is move-only.

Keep the existing strict-suffix, counted `N#`, later-Pomodoro, stale-target, marker, and
EOL tests intact so the fix does not broaden the directive syntax or regress the
already-correct block-line movement.

## Release metadata and user-facing contract

Bump `plugins/task-status-cycler/manifest.json` from `1.6.1` to the next patch version
and revise its concise description only as needed to make status-preserving move-only
rollover part of the contract. Mirror the exact version and aligned wording in the Task
Status Cycler row of `README.md`; leave unrelated plugin metadata untouched.

## Validation and deployment

From the `bob-plugins` repository:

1. Run `node --test scripts/test-task-status-cycler.cjs` while iterating on the focused
   behavior.
2. Run `npm test` to exercise interactions with the other Bob plugins.
3. Run `npm run validate` to check every manifest and plugin entry point.
4. Review the final diff to confirm only Task Status Cycler implementation, tests,
   version metadata, and its README row changed, and that ordinary, embedded, and
   move-only paths remain distinct.
5. Preview deployment with `bob plugins sync -p task-status-cycler --dry-run`, then run
   `bob plugins sync -p task-status-cycler` as required for changes to the
   source-of-truth plugin repository. Confirm the sync updates the vault copy without
   forcing over dirty vault files.
