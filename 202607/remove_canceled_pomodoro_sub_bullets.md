---
tier: tale
title: Remove canceled Pomodoro sub-bullets
goal:
  Remove the complete open-Pomodoro list item for an unambiguously canceled task
  reference, leaving no empty or misleading ledger bullet behind.
create_time: 2026-09-09 19:53:21
status: wip
---

# Plan: Remove canceled Pomodoro sub-bullets

`bob task-status-setter` already recognizes conventional and custom Tasks statuses of
type `CANCELLED`, limits cleanup to block links beneath open Pomodoros, and rescans the
rewritten ledger before deriving task and dependency statuses. The recent implementation
performs a token edit for each qualifying link, however, so authored prose, the list
marker, and any nested content remain after the canceled task reference disappears.
Change the cancellation cleanup unit from a link token to the complete Markdown list
item.

## Behavior contract

- When an indented bullet beneath an open Pomodoro contains a block link whose every
  matching Tasks task has a recognized `CANCELLED` status, remove that bullet from its
  starting line through the end of its nested list-item subtree. A canceled link in a
  nested bullet removes that nested item, not its parent; a canceled link on the parent
  bullet removes the parent and all descendants.
- The whole item is removed even when its line also contains prose or live, completed,
  unresolved, embedded, aliased, marked, or struck links. Those sibling references no
  longer contribute desired task status because the already-established post-rewrite
  rescan observes only the surviving ledger.
- Preserve the existing safety boundaries: do not remove top-level Pomodoro lines, links
  beneath completed or canceled Pomodoros, fenced examples, unresolved/non-task targets,
  or references whose duplicate task matches are not all recognized as canceled.
- Preserve duplicate ownership precedence and deterministic reporting. A physical line
  already selected by cross-Pomodoro duplicate cleanup must not also produce a
  canceled-reference report. If a canceled parent removes descendant bullets, suppress
  structural edits, moves, and redundant cancellation reports for those covered
  descendants.
- Keep the stable `removed_canceled_references` JSON field and its
  per-qualifying-occurrence ordering so consumers are not forced through an output
  migration. Its records identify the canceled references that triggered list-item
  deletion; multiple qualifying references on the same bullet may still produce multiple
  records even though the bullet subtree is deleted once. Clarify the human-readable and
  documentation language so the mutation unit is not described as token-only cleanup.

## Implementation

1. In `src/native/task_status_setter.rs`, refactor canceled-reference planning into a
   deletion pass that runs after duplicate-line ownership is known and before
   completed-link token normalization and bullet relocation. Use each parsed
   `LinkBullet`'s existing `line_index..end_line` boundary to add the complete canceled
   list-item subtree to the structural deletion set. Track covered ranges/lines so
   overlapping parent and descendant bullets are processed once and content scheduled
   for deletion cannot also be edited, moved, or reported by later structural rules.
2. Keep duplicate cleanup's existing physical-line-only contract separate from
   canceled-subtree deletion. Compose both deletion sources in `apply_structural_plan`,
   including the existing case where a completed parent bullet is relocated but a
   canceled descendant is omitted from the moved subtree. Continue rescanning the
   structurally updated daily note before desired-status and dependency propagation so
   live or completed links removed alongside the canceled item stop affecting the same
   run.
3. Update the structural unit tests in `src/native/task_status_setter.rs` to replace
   token-removal expectations with full list-item removal. Cover mixed prose/link
   content, multiple canceled references on one item, nested descendants, a canceled
   nested item under an otherwise surviving or moving parent, duplicate-deleted lines,
   CRLF/no-final-newline preservation around surviving content, deterministic reports,
   and a second idempotent rewrite.
4. Update the end-to-end regression in `tests/cli.rs` so dry-run and apply modes prove
   that complete canceled sub-bullets disappear, canceled task files are unchanged,
   unrelated surviving bullets retain their behavior, collateral live/dependency
   references are excluded by the same-run rescan, ambiguous and out-of-scope links
   remain, output remains deterministic, and rerunning is a no-op.
5. Revise the command's long help, `README.md`, and `docs/task-status-setter.md` to
   state that a qualifying canceled reference removes its complete list-item subtree.
   Remove the prior promise that surrounding prose, sibling links, and nested content
   are preserved; document mixed-content consequences, safety exclusions, duplicate
   precedence, rescan behavior, and the retained `removed_canceled_references`
   compatibility contract.

## Validation

- Run the focused structural and CLI cancellation tests while iterating, along with
  adjacent completed-reference movement and duplicate-removal tests that share the
  rewrite planner.
- Run `cargo fmt --check` and `cargo clippy --all-targets --all-features`.
- Run the complete `cargo test` suite (or `just all`) to catch status propagation,
  output serialization, line-ending, movement, and idempotence regressions outside the
  focused cases.
- Confirm the worktree diff changes only the task-status-setter implementation, its
  tests, and the corresponding user documentation.
