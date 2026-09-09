---
tier: tale
title: Remove canceled task links from open Pomodoros
goal: "bob task-status-setter removes open-Pomodoro block-link occurrences whose
  resolved Tasks tasks are canceled, reports the cleanup in dry-run and normal output,
  and derives task statuses from the rewritten daily ledger.

  "
create_time: 2026-09-09 19:53:20
status: wip
---

# Plan: Remove canceled task links from open Pomodoros

## Context and intended behavior

`bob task-status-setter` already scans the `## Pomodoros` section in today's daily note,
resolves block links to Tasks task blocks, removes duplicate lines, retires completed
references, normalizes Pomodoro markers, then scans the rewritten section again before
computing direct and dependency-derived task statuses. Canceled tasks are currently
protected from status changes, but their links remain beneath open Pomodoros and can
continue to seed dependency propagation.

Extend that same structural rewrite pass so every resolvable block-link occurrence
beneath an open Pomodoro is removed when all matching Tasks task blocks have a
recognized `CANCELLED` status. This includes the conventional `[-]` status and any
single-character custom status whose Tasks configuration declares type `CANCELLED`. The
task itself remains canceled; only its daily-note link occurrence is removed. Apply the
rule to plain, embedded, aliased, Pomodoro-marked, and exactly struck forms, removing
the complete managed token while preserving the containing bullet's other prose, links,
indentation, nested content, and line ending.

Keep the cleanup conservative and scoped:

- Only links owned by open top-level Pomodoro entries in today's selected daily note are
  candidates. Links beneath completed or canceled Pomodoros, outside the Pomodoros
  section, on top-level entry lines, or inside fenced examples remain untouched.
- Unresolved links and links that do not resolve to a scanned Tasks task remain
  untouched and retain the existing warning behavior.
- When one block ID matches multiple task lines, remove its occurrence only if every
  match is recognized as canceled. Mixed canceled/open, canceled/done, or
  canceled/unknown matches remain in place so an ambiguous identity cannot discard a
  live task source; retain or clarify the existing ambiguity warning.
- Preserve the existing cross-Pomodoro duplicate-line ownership pass. Lines already
  selected for duplicate removal must not also produce canceled-link edits or duplicate
  reports. On surviving mixed-content lines, remove only canceled occurrences and allow
  live/completed occurrences to follow their existing policies.
- Feed the structurally rewritten daily note into the existing final reference scan.
  Removed canceled roots and any otherwise-unreachable transcluded dependency chain
  therefore stop contributing desired Next/In-Progress state in the same run, while the
  legacy `references` count continues to describe the unique raw input references
  observed before cleanup.

## Implementation

1. In `src/native/task_status_setter.rs`, make semantic canceled-status classification
   available to the structural planner using the parsed Tasks status registry rather
   than hard-coding only `-`. Add a canceled-reference result record and a collection on
   the structural plan/result alongside the existing completed, marker, and duplicate
   cleanup records.

2. Extend structural planning to identify all-canceled resolved occurrences beneath open
   Pomodoros, schedule non-overlapping token removals, and exclude duplicate-deleted
   lines. Compose cancellation with completed retirement, relocation, and marker
   normalization without emitting competing edits for the same occurrence. Keep the
   current atomic output composition so daily-note cleanup and any status changes—
   including tasks stored in the daily note—are written together, and keep `--dry-run`
   on the identical no-write planning path.

3. Add a stable `removed_canceled_references` array to JSON output with enough identity
   and Pomodoro context to audit each removed occurrence. Include the new collection in
   no-op/change detection, add human `would remove`/`removed` sections and summary
   counts, and preserve deterministic file/occurrence ordering. The second identical run
   must report no canceled removals and no changes.

4. Update the command's long help, the task-status-setter synopsis in `README.md`, and
   `docs/task-status-setter.md` to distinguish the unchanged canceled task status from
   removal of its link beneath an open Pomodoro. Document custom `CANCELLED` symbols,
   scope exclusions, duplicate-ID safety, mixed-content preservation, final-graph
   effects, dry-run/human reporting, and the additive JSON field and input-count
   contract.

## Verification

Add focused unit coverage in `src/native/task_status_setter.rs` for semantic
cancellation classification and structural composition. Exercise conventional and custom
canceled symbols; plain, embedded, aliased, marked, and struck link syntax; multiple
occurrences on one line; mixed canceled/live/completed links; completed and canceled
Pomodoro owners; fenced and unresolved links; CRLF and final-line-ending preservation;
duplicate-deleted lines; and duplicate block IDs where all matches are canceled versus
statuses that conflict. Assert that the post-rewrite scan no longer exposes removed
occurrences and that a second planning pass is empty.

Add an end-to-end CLI regression in `tests/cli.rs` using a temporary vault and explicit
`BOB_DAY_FILE`. It should prove that:

- JSON dry-run reports the canceled occurrences but changes neither the daily note nor
  task notes.
- Applying the command removes only qualifying open-Pomodoro link tokens, leaves the
  canceled tasks and out-of-scope links unchanged, preserves live mixed-line content,
  and clears or avoids promoting dependency state that was reachable only through a
  removed canceled root.
- A custom Tasks status with type `CANCELLED` behaves like `[-]`, while a mixed
  duplicate block ID is retained.
- Human output and the summary include the cleanup, JSON ordering/content is stable, and
  a repeat run is an idempotent no-op.

Run targeted task-status-setter unit and CLI tests during development, then finish with
`cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test` (or
`just all`) to validate the full repository.

## Acceptance criteria

- After a successful run, no qualifying block-link occurrence to an unambiguously
  canceled task remains beneath an open Pomodoro in today's daily note.
- No task is uncanceled or otherwise transitioned because of this cleanup, and no
  unresolved, ambiguous-live, fenced, completed-Pomodoro, or canceled-Pomodoro content
  is destructively changed.
- Status propagation uses the rewritten ledger in the same invocation; dry-run, normal
  writes, atomic composition, deterministic reporting, and idempotence retain their
  existing contracts.
