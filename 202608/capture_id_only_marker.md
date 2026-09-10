---
tier: tale
title: Add ID-only capture markers across Bob CLI and macOS
goal:
  Users can assign a task block ID with @file::id without creating a Pomodoro task link,
  consistently in bob-cli and bob-mac-capture.
size: medium
proposed_by: bbugyi200.athena.022
create_time: 2026-09-09 19:59:49
status: wip
---

# Add an ID-only `bob capture` marker across CLI and macOS capture

## Goal

Add a canonical `@<route>::<block-id>` capture marker that creates an ordinary open task
with the requested trailing Obsidian block ID, but does not create or modify a Pomodoro
task-block-link bullet. Preserve every existing `bob capture` marker and wire contract,
and make the linked `bob-mac-capture` app understand, highlight, preview, and submit the
new syntax through bob-cli's versioned JSON interfaces.

This is a medium tale because one agent can implement the coordinated contract change
end to end, but it spans the shared Rust grammar, filesystem mutation and atomicity,
CLI/editor JSON contracts, documentation, and the Swift client and its fixtures.

## Required semantics and compatibility

- `@file::id` is recognized only in the same leading/trailing marker positions as the
  other route forms. It routes to `file.md`, renders an ordinary unchecked task such as
  `- [ ] #task <body> [created::<date>] ^id`, and keeps priority/scheduled properties
  and the block ID in their established order, with the block ID last.
- The result remains an ordinary task: `bob capture --format json` reports
  `kind: "task"`, adds `block_id: "id"`, and omits `day_file`, `block_link`, and
  `pomodoro_link_placement`. `bob capture-parse` reports mode `task`, the resolved route
  and `block_id`, and no needs for a complete marker.
- The ID-only path must not resolve, require, read, validate, or write a daily note. In
  particular, an invalid or missing `BOB_DAY_FILE` cannot affect it. `@file:id` retains
  its current Pomodoro-linked `[*]` task and daily-ledger behavior unchanged.
- Reuse bob-cli's existing block-ID validator. Before any target mutation, reject an ID
  that already appears anywhere in the destination note; a failed real or dry-run
  capture leaves the target unchanged. A missing target may still be created just like
  an ordinary routed task.
- Give double-colon recognition precedence over the single-colon Pomodoro grammar, while
  preserving caret sub-bullet and hash section precedence, middle-token literal
  behavior, forced `--route` literal behavior, legacy `@!` behavior, multiline
  capture-wide marker rules, and all current `@file:id` diagnostics.
- Treat `@::`, `@::id`, `@route::`, and `@route::id` as the editor-facing interactive
  family. Add a `block_id` need for a missing authored ID and dedicated
  `task_block_id_route` / `task_block_id` span kinds plus stable
  `invalid_task_block_id_route` / `invalid_task_block_id` diagnostics. Route completion
  remains available on the left side, including an empty route. The right side is a new
  user-authored ID, so it must not query existing tasks or offer candidates that would
  necessarily collide with the destination; `capture-complete` returns no active field
  while the caret is in that component.
- Keep the `capture-parse` and `capture-complete` schema version at 1: these are
  additive enum/string values and an already-optional `block_id`, not a structural
  breaking change. Document every new wire value.

## Implementation

1. Extend the shared Rust capture grammar in `src/native/capture_language.rs`. Introduce
   an explicit ordinary-task-with-ID representation, classify the double-colon token
   before Pomodoro tokens, and share parsing/validation between execution, editor
   parsing, and cursor completion. Generalize marker span/range calculations where
   needed for the two-byte separator. Add focused unit matrices for complete and
   incomplete forms, leading/trailing and terminal-marker composition, invalid route/ID
   components, UTF-8 byte spans, precedence, execution/editor parity, multiline
   placement, and route- versus-ID completion behavior.

2. Extend capture rendering and write planning in `src/native/capture.rs`. Render the
   ordinary `[ ]` task with a final block ID, extract/reuse destination duplicate-ID
   preflight rather than entering the Pomodoro two-note planner, carry the new ID into
   successful human/JSON output, and keep ordinary placement, authored children,
   clipboard children, schedule logs, dry-run, line-ending, and atomic single-file write
   behavior intact. Update command help and examples.

3. Add CLI integration coverage in `tests/cli.rs` and contract documentation in the
   bob-cli `README.md`. Cover a successful capture into an existing note and creation of
   a new routed note, JSON output and property ordering, absence/unmodified state of the
   daily note, dry-run, duplicate-ID no-write behavior, malformed marker usage errors,
   `@file:id` regression behavior, parse spans/needs/diagnostics, and completion on only
   the new marker's route side.

4. Open the linked `bob-mac-capture` repository through `sase repo open bob-mac-capture`
   before reading or editing it. Update the semantic span mapping and the panel's route
   completion/cached-route span sets so `task_block_id_route` behaves like a destination
   and `task_block_id` uses the block-ID palette without triggering an existing-task
   picker. Keep the raw `mode`, `needs`, and optional `block_id` decoding additive; do
   not duplicate the capture grammar in Swift.

5. Extend the mac app's `Tests/Fixtures/fake-bob` and Swift tests to exercise the real
   client boundary: decode an ordinary-task success carrying `block_id` but no Pomodoro
   fields, highlight both new span kinds, use cached/server route completion while the
   caret is in the route portion, avoid completion in the authored-ID portion, and show
   the ID-bearing ordinary task in live preview. Document the syntax and the deliberate
   lack of right-hand task completion in the linked repository's `README.md`.

## Validation

- In bob-cli, run focused Rust unit/integration tests for capture language, capture
  parse, capture completion, ordinary capture, the new ID-only path, and existing
  Pomodoro capture first. Then run `just all` to enforce formatting, Clippy, and the
  full test suite.
- In `bob-mac-capture`, run `just format-lint` and `just test`; run `just build` as an
  additional compile check for the executable target. If the host lacks the required
  macOS/Xcode toolchain, report that environmental limitation explicitly after still
  running every available source/fixture check.
- Re-read both READMEs and both command help contracts after tests to ensure
  terminology, examples, span/need/diagnostic lists, completion behavior, and JSON
  fields agree with the implementation.

## Acceptance criteria

- `bob capture 'Do work @file::id'` succeeds without a usable daily note, writes exactly
  one ordinary unchecked routed task ending in `^id`, and creates no Pomodoro
  sub-bullet.
- `bob capture 'Do work @file:id'` continues to create a next-status task and its
  Pomodoro task block link exactly as before.
- Duplicate and malformed ID-only requests fail before writes with stable, documented
  diagnostics, while dry-run performs the same preflight without changing files.
- bob-cli parse/complete output and bob-mac-capture highlighting, route completion, live
  preview, and submission agree on the new syntax without adding a second grammar to the
  app.
- All available bob-cli and bob-mac-capture formatting, lint, build, and test checks
  pass, with unrelated pre-existing failures clearly distinguished from regressions.
