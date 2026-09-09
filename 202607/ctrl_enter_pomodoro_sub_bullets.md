---
tier: tale
title: Route Ctrl+Enter from Pomodoro sub-bullets
goal:
  Make normal-mode Ctrl+Enter on an open Pomodoro's sub-bullets complete that Pomodoro
  with the existing rollover behavior, while preserving selected embedded
  task-transclusion handling.
create_time: 2026-09-09 19:53:09
status: wip
---

# Plan: Route Ctrl+Enter from Pomodoro sub-bullets

## Context and behavior contract

Implement this in the linked `bob-plugins` repository's `task-status-cycler` plugin.
Open that repository through the required `sase_repo` workflow before reading or
changing it; do not edit the deployed copies in the Bob vault.

Today, normal-mode `<C-CR>`/`<C-Enter>` completes an open Pomodoro only when the cursor
is directly on its top-level checkbox. From a child bullet, the same dispatch instead
toggles a checkbox on that child, operates on a selected block link, or does nothing.
The plugin already owns the complete Pomodoro behavior that closes embedded task trees,
starts eligible bare task links, marks or retires historical links, moves `#`-marked
links without copying history, carries ordinary live links forward, creates or selects
the next Pomodoro, and places/centers the cursor. The new branch must reuse that
behavior rather than duplicate any part of the completion planner.

Treat a daily-ledger context the same way the existing Pomodoro section and task-line
helpers do: a recognized `## Pomodoros` section, a top-level Pomodoro checkbox, and its
contiguous block of indented list-item descendants. Do not introduce a separate
date/path detector or expand the existing section grammar, because pressing Ctrl+Enter
on the parent checkbox is already governed by this structural contract and the requested
child behavior should match it exactly.

For an open (`[ ]`) owning Pomodoro, the dispatch contract is:

| Active line                                                                         | Ctrl+Enter result                                                                                                                                          |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pomodoro checkbox                                                                   | Keep the existing full Pomodoro completion path.                                                                                                           |
| Any direct or nested child list item without a selected embedded block transclusion | Complete the owning Pomodoro through that same path, including all rollover and cursor behavior.                                                           |
| A selected `![[...#^block-id]]` child target                                        | Keep the existing selected-task behavior: recursively close an incomplete target tree, or root-only reopen a Done target, without completing the Pomodoro. |
| An unresolved, stale, non-task, or excluded-status embedded target                  | Preserve the current safe no-op; do not fall through and unexpectedly complete the Pomodoro after target resolution fails.                                 |
| A line outside the owning child range or outside `## Pomodoros`                     | Preserve the current checkbox and block-link behavior.                                                                                                     |

Limit the new parent-routing branch to an open owning Pomodoro, which is the state
accepted by the existing full completion planner and the state that can create or select
the next Pomodoro. Historical completed/cancelled Pomodoros must retain their current
child-line behavior, including selected Done-task reopening.

## Implementation

1. In `plugins/task-status-cycler/main.js`, extract or add one reusable structural
   resolver for the Pomodoro that owns an active child line. It should return the
   section, parent line, live parent task status, and a context compatible with
   `completeActivePomodoroTask` whenever the active line is an indented list item inside
   that parent's contiguous sub-bullet range. Keep this structural resolver
   status-neutral so existing embedded-transclusion handling also works beneath
   completed Pomodoros; gate only the new parent-completion route on an open `[ ]`
   owner. Reuse the resolver from the existing embedded-transclusion recognition where
   practical so the two paths cannot disagree about section boundaries, nearest-parent
   ownership, nested bullets, or range termination.

2. Refactor `handleVimTaskToggleOpenDone` into an explicit precedence order:
   - retain direct open/Done Pomodoro checkbox handling first;
   - detect whether the active line belongs to an open Pomodoro before the generic
     active-checkbox branch, so checkbox child bullets are included in “any sub-bullet”;
   - if the cursor selects an embedded block-link candidate, delegate to the existing
     `handleActiveTaskBlockLinkOpenDone` path and consider the keypress consumed even if
     resolution later fails;
   - otherwise call `completeActivePomodoroTask` with the owning parent context and the
     current `MarkdownView`, allowing the existing plan application to move and center
     the cursor exactly as it does from the parent checkbox;
   - leave generic task-checkbox and block-link dispatch unchanged for lines that do not
     qualify.

   Keep the syntax/selection rules already used by Ctrl+Enter: a single embedded
   candidate is selectable from anywhere on its line, while multiple candidates require
   the cursor to disambiguate. Plain `[[...#^...]]` links, retired or marked links,
   prose bullets, nested bullets, and child checkbox lines are not part of the exception
   and therefore complete the owner.

3. Remove or simplify any now-obsolete special-case direct handling for bare
   non-transcluded Pomodoro links if the new dispatch makes it unreachable, while
   retaining the full-Pomodoro logic that starts eligible bare linked tasks before
   rollover. Update nearby comments so they describe the new precedence and do not claim
   that Ctrl+Enter on a bare child link only starts that link.

4. Update `plugins/task-status-cycler/manifest.json` and the matching `README.md` plugin
   table entry. Treat the expanded Ctrl+Enter behavior as a minor feature release
   (`1.6.0` from `1.5.1`) and describe the child-to-parent Pomodoro completion behavior
   without obscuring the transcluded-task exception or the existing move-only workflow.

## Tests and validation

Extend `scripts/test-task-status-cycler.cjs` at both the structural-helper and
Vim-dispatch/runtime levels:

- prove that direct and deeper indented child bullets resolve to the nearest owning
  Pomodoro regardless of parent status, while a blank/prose boundary, next top-level
  Pomodoro, next level-two section, or non-list line does not resolve; then prove that
  only an open `[ ]` owner qualifies for the new parent-completion route;
- invoke the registered Ctrl+Enter action from prose, plain/marked block-link, nested,
  and child-checkbox bullets and verify that the owner is completed rather than the
  child or selected plain link;
- include an end-to-end rollover assertion showing the same output and cursor target as
  parent-line completion: the current Pomodoro becomes Done, carry-forward/move-only
  rules remain intact, and a placeholder is created only under the existing planner's
  conditions;
- keep an embedded open/in-progress/Next task transclusion on the selected-task path,
  with its target tree closed and the local Pomodoro left open; retain coverage that a
  selected Done target reopens root-only;
- verify that a selected stale/non-task embedded block link is consumed as a no-op
  rather than completing the owner;
- verify unchanged behavior for ordinary task lines and block links outside Pomodoro
  child ranges, and for child lines beneath non-open Pomodoros.

Run the focused test first, then the complete repository gates:

```bash
node --test scripts/test-task-status-cycler.cjs
npm test
npm run validate
```

After the source, tests, README, and manifest all pass, follow the linked repository
instruction by deploying the plugin with `bob plugins sync -p task-status-cycler`
(without `--force`) and verify the reported sync state. If the vault copy is dirty and
the safe sync refuses, leave it untouched and report that deployment blocker rather than
overwriting it.
