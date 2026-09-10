---
tier: tale
title: Keep captured task children above managed logs
goal:
  Sub-bullet captures land before direct-child Schedule Log and Work Log blocks without
  disturbing existing task structure or capture semantics.
size: medium
proposed_by: bbugyi200.athena.041
create_time: 2026-09-09 19:59:50
status: wip
---

# Keep captured task children above managed logs

## Goal

Change `bob capture` sub-bullet mode so a capture targeting an existing task
(`@<route>+<block-id>`, `--route ... --task ...`, or the picker-only `--task-ref`)
inserts the complete new child block before that task's first direct-child Schedule Log
or Work Log. If neither managed log exists, retain the current append-at-the-end
behavior.

The new capture block includes the captured bullet and any authored children, clipboard
children, or priority-generated Schedule Log nested beneath that bullet. It must move as
one unit and must not alter the selected parent or any pre-existing log subtree.

## Current behavior and relevant code

- `src/native/capture.rs::plan_sub_bullet_capture` resolves the selected task through
  `note_tasks::scan`, chooses the new child's indentation, then always inserts at
  `parent.block_end`. That byte offset is the end of the selected task's entire
  descendant block, so existing managed logs necessarily remain above the new child.
- All three sub-bullet selectors converge on this planner. Keeping placement there
  preserves selector parity, dry-run behavior, and the in-memory batch planner's rule
  that later capture items see earlier planned edits.
- `assemble_capture_block` already keeps a captured child's authored and clipboard
  descendants ahead of its own generated Schedule Log. This internal order is
  independent of where the complete block lands under the existing task and should
  remain unchanged.
- The Bob plugins recognize managed logs only when their marker bullet is a direct child
  of the task. Schedule Log accepts canonical `🗓️ **SCHEDULE LOG**`, emoji-less
  `**SCHEDULE LOG**`, and legacy `**Schedule log:**` forms. Work Log accepts canonical
  `🛠️ **WORK LOG**`, emoji-less `**WORK LOG**`, and legacy `**Work log:**` forms. Their
  marker grammar permits `-`, `*`, `+`, and ordered-list markers, an optional matching
  emoji, and an optional colon. Matching should mirror that grammar rather than relying
  on a substring.

## Implementation

1. In `src/native/capture.rs`, add focused parsing/scanning helpers for managed task-log
   marker bullets. Mirror the plugin-compatible Schedule Log and Work Log spellings and
   list-marker grammar, while rejecting trailing content and unrelated case variants so
   ordinary user bullets are not treated as logs.

2. Scope marker discovery to the already-resolved task's descendant byte range and
   require the marker to be a direct child of that task. Determine ancestry from
   Markdown list indentation/list-item structure, not merely from being indented more
   than the task: a Schedule Log or Work Log nested beneath a child task or freeform
   child must not affect the parent's insertion point. Ignore blank lines while
   resolving the nearest shallower list-item parent, consistent with the plugin
   behavior.

3. Derive the insertion byte offset for the earliest matching direct-child log marker.
   Insert `capture_block` immediately before that marker; otherwise use
   `parent.block_end` exactly as today. Continue choosing the new bullet's indentation
   from the selected task's existing children and continue using
   `insertion_text_preserving_line_endings`, so tabs versus spaces, nested parent
   indentation, LF/CRLF, EOF-without-newline handling, and atomic staging remain intact.
   Do not move, normalize, merge, or rewrite either log or its entries.

4. Keep the placement decision inside `plan_sub_bullet_capture` so typed `@file+id`,
   forced `--task`, hidden `--task-ref`, dry runs, and multi-item captures all receive
   identical ordering. In a batch, repeated captures for the same task should retain
   input order and all land before the first managed log. A captured child that
   generates its own nested Schedule Log through `p:<N>` must remain one contiguous
   subtree above the selected parent's logs.

5. Update `bob capture`'s long help in `src/native/capture.rs` and the sub-bullet
   documentation/example in `README.md` to state that new task children are placed
   before existing direct-child Schedule Log and Work Log blocks, while tasks without
   managed logs still append normally.

## Tests

Add focused helper/unit coverage in `src/native/capture.rs` where useful and CLI
integration coverage in `tests/cli.rs` for the observable note edits:

- A typed `@route+block-id` capture lands before canonical Schedule Log and Work Log
  subtrees, including when both exist in either order, while preserving their nested
  entries byte-for-byte.
- Emoji-less and legacy spellings, optional colons, and supported unordered or ordered
  list markers are recognized; lookalikes with trailing text or unsupported casing are
  left as ordinary children.
- A matching marker nested under another direct child is ignored, and a task with no
  direct managed log retains append-at-end behavior.
- Tabs, two-space indentation, an indented parent task, CRLF input, and a final log at
  EOF preserve existing formatting.
- `--task-ref`/`--task` use the same ordering as typed syntax; a multi-item capture
  targeting one task keeps capture order above the log.
- A sub-bullet capture with authored/clipboard descendants or a `p:<N>`-created nested
  Schedule Log is inserted as one subtree before the parent's logs and retains the
  existing JSON/human output contract.

Run the targeted capture tests first, then the full repository checks:

```bash
cargo test capture_sub_bullet
cargo test capture_priority_sub_bullet
just all
```

## Acceptance criteria

- Every successful `bob capture` sub-bullet write places the complete new child before
  the earliest recognized Schedule Log or Work Log that is a direct child of the
  selected task.
- Nested or non-matching log-like bullets do not change placement, and tasks without a
  managed log behave exactly as before.
- Existing logs, their entries, other children, parent metadata, indentation, line
  endings, dry-run semantics, batch atomicity/order, and output schemas remain unchanged
  apart from the requested insertion position.
- The new focused tests and `just all` pass, and user-facing capture help and README
  documentation describe the ordering rule.
