---
tier: tale
title: Migrate capture sub-bullet and ID-only markers
goal:
  Users use @file+id to add a sub-bullet beneath an existing task and @file^id to create
  an ordinary task with an authored block ID, consistently in bob-cli and
  bob-mac-capture.
size: medium
proposed_by: bbugyi200.athena.022.f1
create_time: 2026-09-09 19:59:50
status: wip
---

# Migrate capture sub-bullet and ID-only markers

## Goal

Reassign the capture marker punctuation so `@<route>+<block-id>` performs the existing
sub-bullet capture and `@<route>^<block-id>` performs the new ordinary task-with-ID
capture. Retire the temporary `@<route>::<block-id>` spelling, preserve the semantic
capture and JSON contracts behind both operations, and keep the linked `bob-mac-capture`
app's highlighting, completion, preview, and submission behavior in lockstep with
bob-cli's versioned editor interfaces.

This is a medium tale because one agent can implement and validate the coordinated
migration, but the work spans the shared Rust execution/editor grammar, intentional
breaking-syntax diagnostics, CLI documentation and integration coverage, and the Swift
client's semantic-span and subprocess fixtures.

## Required semantics and compatibility

- `@file+id` replaces `@file^id` as the only text-marker spelling for sub-bullet
  capture. It routes to `file.md`, finds the existing task carrying `^id`, and inserts
  the captured Markdown block beneath that task with the current indentation, placement,
  dry-run, stale-reference, and error behavior. The explicit `--route` plus
  `--task`/`--task-ref` interfaces remain unchanged.
- `@file^id` replaces `@file::id` as the only text-marker spelling for an ordinary task
  with an authored block ID. It writes an unchecked routed task ending in `^id`, reports
  JSON `kind: "task"` and `block_id: "id"`, reuses the destination duplicate-ID
  preflight, and never resolves or mutates a Pomodoro daily note or parent task.
- This is an intentional input-grammar break, not an alias expansion. Do not preserve
  caret as a sub-bullet alias or double colon as an ID-only alias. Detect a leading or
  trailing `@route::...` family deterministically and return an actionable usage error /
  editor diagnostic directing users to `@route^...`; it must not silently remain literal
  or be misreported as a malformed Pomodoro marker.
- Preserve `@file:id` Pomodoro-linked capture, `@file#section` bullet capture, plain
  `@file` routing, legacy `@!` Pomodoro aliases, authored child bullets (including a
  line-leading `+ ` Markdown bullet), forced `--route` literal behavior, terminal
  schedule/priority/clipboard composition, and leading/trailing/multiline marker
  precedence.
- Make mixed-separator precedence explicit. A marker separator is structural only when
  it occurs before competing `#`, `:`, `+`, or `^` separators as appropriate, so section
  suffix text is not stolen by the other grammars and malformed mixed markers fail under
  the semantic family they begin. The whitespace-free `@route+id` token must not
  conflict with a line-leading Markdown `+ ` authored-child marker.
- Keep semantic types and wire values stable: ordinary ID-only capture remains
  `TaskWithBlockId`, mode `task`, need `block_id`, and spans
  `task_block_id_route`/`task_block_id`; sub-bullet capture remains `SubBullet`, mode
  `sub_bullet`, need `task`, and spans `sub_bullet_route`/`sub_bullet_block_id`.
  Existing error-code families remain semantic rather than punctuation-specific.
- Keep `capture-parse` and `capture-complete` schema version 1 because their JSON shapes
  and semantic enum strings do not change. Complete and incomplete editor families do
  change punctuation: `@+`, `@route+`, `@+id`, and `@route+id` drive the existing-task
  picker flow, while `@^`, `@route^`, `@^id`, and `@route^id` drive the authored-ID
  flow. Completion offers routes on either family's left side and existing tasks only on
  the `+` family's right side; an authored ID on the `^` family's right side has no
  completion source.

## Implementation

1. Update the single-source grammar in `src/native/capture_language.rs`. Move sub-bullet
   execution parsing, candidate detection, editor classification, span calculation,
   diagnostics, and cursor completion from `^` to `+`; move the existing task-with-ID
   semantic path from `::` to `^`; and remove double-colon execution and interactive
   classification. Add a narrow retired-double-colon detector so old requests receive
   migration guidance without falling into the single-colon Pomodoro parser. Keep the
   shared marker-shape helper generic and semantic names independent of punctuation.

2. Expand the capture-language unit matrices around the complete and four incomplete
   forms of both new families. Cover leading and trailing positions, capture-wide
   markers on authored child lines, exact UTF-8 byte spans, needs, execution/editor
   parity, route-side versus right-side completion, invalid routes and block IDs, mixed
   `#`/`:`/`+`/`^` precedence, the retired `::` diagnostic, middle-token literal
   behavior, forced-route behavior, and unchanged Pomodoro/section/legacy forms.

3. Audit `src/native/capture.rs` to keep execution semantics tied to `CaptureKind`, not
   punctuation. No write-path redesign is intended: retain ordinary-task formatting and
   duplicate-ID preflight for caret ID-only captures, and retain existing-task lookup
   and nested insertion for plus sub-bullet captures. Update capture command help and
   focused unit tests/examples anywhere they expose the old spellings.

4. Update `src/native/capture_parse.rs`, `src/native/capture_complete.rs`,
   `tests/cli.rs`, and the bob-cli `README.md`. Document the new complete/incomplete
   families, migration error, semantic spans/needs, and asymmetric completion behavior.
   At CLI level prove that `@file^id` writes an ordinary `[ ]` task with a final block
   ID and no Pomodoro fields or daily-note dependency, while `@file+id` inserts beneath
   an existing task and reports `kind: "sub_bullet"`; cover dry-run/no-write failures
   and show `@file::id` is no longer accepted.

5. Before reading or changing the linked client, open it with
   `sase repo open bob-mac-capture`. Keep Swift dependent on bob-cli's semantic JSON
   rather than teaching it punctuation parsing. Ensure `task_block_id_route` uses the
   route highlight and cached/server route-completion paths, `task_block_id` uses the
   block-ID palette without invoking the existing-task picker, and the unchanged
   `sub_bullet_*` span names continue to drive route and task completion after their
   source punctuation becomes `+`.

6. Update `bob-mac-capture`'s `Tests/Fixtures/fake-bob`, process-client/model/panel
   tests, and `README.md`. Exercise parse and completion for plus sub-bullet markers and
   caret ID-only markers, decode and preview an ordinary task success carrying
   `block_id` but no Pomodoro fields, submit both semantic modes through direct argv,
   verify cached route completion on both route spans, verify task candidates only on
   the plus right-hand side, and verify the caret-authored ID side displays no picker.

## Validation

- In bob-cli, run focused Rust unit and CLI integration tests for capture execution,
  capture parsing, capture completion, ID-only duplicate preflight, sub-bullet task
  lookup/insertion, retired-syntax diagnostics, and Pomodoro/section regressions. Then
  run `just all` for formatting, Clippy, and the complete test suite.
- In `bob-mac-capture`, run `bash -n Tests/Fixtures/fake-bob`, `just format-lint`,
  `just test`, and `just build`. If the host lacks the macOS/Xcode toolchain, run every
  available source/fixture check and report the environmental limitation precisely.
- Run `git diff --check` in both repositories and re-read command help plus both READMEs
  to ensure every example, interactive-form list, completion statement, and migration
  message uses `+` for sub-bullets and `^` for ID-only tasks, with no stale
  documentation presenting `::` as supported.

## Acceptance criteria

- `bob capture 'Follow up @file^new-id'` writes exactly one ordinary unchecked routed
  task ending in `^new-id`, reports task semantics and `block_id`, succeeds without a
  usable daily note, and creates no Pomodoro or parent-task sub-bullet.
- `bob capture 'Add context @file+parent-id'` finds `^parent-id` in `file.md`, nests the
  new capture beneath it, and preserves all existing sub-bullet result metadata and
  failure atomicity.
- `@file::new-id` is no longer accepted and produces clear migration guidance;
  `@file:id`, `@file#section`, plain routing, authored Markdown `+ ` child lines, and
  legacy Pomodoro aliases retain their established behavior.
- `capture-parse`, `capture-complete`, and bob-mac-capture agree on the new punctuation:
  both families complete their route side, only `+` completes an existing task, and `^`
  treats its right side as a user-authored ID.
- All available bob-cli and bob-mac-capture formatting, lint, build, and test checks
  pass, with unrelated pre-existing or host-tooling failures clearly distinguished from
  regressions.
