---
tier: tale
title: Start scheduled captures in Blocked status
goal:
  Scheduled bob capture tasks render with the Blocked checkbox immediately while
  unscheduled and non-task capture behavior remains unchanged.
size: small
proposed_by: bbugyi200.athena.0fg
create_time: 2026-09-09 19:59:43
status: wip
---

# Start scheduled captures in Blocked status

## Goal

Make `bob capture` render every checkbox-bearing capture with a resolved
`[scheduled::YYYY-MM-DD]` property as an Obsidian Tasks Blocked task (`[?]`)
immediately, instead of relying on a later `bob task-status-hooks` run to replace the
initial Ready or Next marker. This applies whether the date came from explicit `s:<N>`
syntax, a rolled `p:<N>` priority window, or `p:<N>` combined with an overriding
`s:<N>`.

The status rule is based on the presence of the scheduled property, not on whether the
date is later than today, so `s:0` also creates `[?]`. Unscheduled ordinary tasks remain
Ready (`[ ]`), and unscheduled Pomodoro-linked tasks remain Next (`[*]`).

## Scope and constraints

- Change the canonical Markdown rendering in `src/native/capture.rs` so scheduled
  `CaptureKind::Task`, `CaptureKind::TaskWithBlockId`, and `CaptureKind::Pomodoro`
  results use `?` as their checkbox symbol. Keep the existing unscheduled symbol for
  each kind.
- Keep ordinary bullet, nested sub-bullet, and Pomodoro-note captures checkbox-free;
  their optional scheduled and priority properties must continue to render exactly as
  before.
- Preserve the rest of each line byte-for-byte: `#task`, body,
  created/priority/scheduled property order, and final block ID. Preserve authored
  children, clipboard children, Pomodoro links, priority-roll Schedule Logs, placement,
  and atomic-write behavior.
- Keep the JSON schema unchanged. Human output, dry-run output, JSON `task_line`, live
  preview consumers, and the committed vault text should all receive the same new
  rendered status through the shared formatter.
- Do not invoke `bob task-status-hooks`, add Tasks-plugin registry validation, or alter
  its future-date reconciliation rules. This change only chooses the initial capture
  marker.
- No `bob-mac-capture` source change is required: its live preview displays Bob's
  returned `task_line` as opaque Markdown and already carries the separate `scheduled`
  value.

## Implementation

1. In `src/native/capture.rs`, centralize the checkbox-symbol choice for task lines:
   scheduled tasks select Blocked (`?`), while unscheduled tasks retain the formatter's
   kind-specific default (` ` for ordinary and ID-bearing tasks, `*` for Pomodoro-linked
   tasks). Continue letting the ID-bearing formatter delegate to the ordinary task
   formatter so those two paths cannot diverge.
2. Update the renderer unit tests in `src/native/capture.rs` to cover both branches for
   ordinary, block-ID, and Pomodoro-linked task lines. Assert that scheduled cases use
   `[?]`, unscheduled cases retain `[ ]` or `[*]`, metadata ordering and final block IDs
   do not move, and bullet/sub-bullet formatting remains unaffected.
3. Update and strengthen the end-to-end capture assertions in `tests/cli.rs` across the
   shared rendering surfaces: explicit `s:<N>` (including `s:0`), seeded `p:<N>`,
   combined `p:<N> s:<N>`, routed and ID-bearing tasks, Pomodoro-linked tasks,
   multi-item or authored-child captures, clipboard/Schedule Log composition, dry-run
   output, JSON `task_line`, and written note contents. Retain an unscheduled capture
   assertion for `[ ]` and an unscheduled Pomodoro assertion for `[*]`; scheduled
   bullets and sub-bullets should remain plain list items.
4. Update the `bob capture --help` text in `src/native/capture.rs`, the scheduling and
   priority contract in `docs/capture.md`, and the concise marker descriptions in
   `README.md` to state that a resolved scheduled property makes checkbox-bearing
   captures start Blocked. Remove the obsolete claim that capture always writes `[ ]`,
   explain the `s:0` behavior, and keep the documented exception for checkbox-free
   bullet forms and the existing Schedule Log semantics.

## Acceptance criteria

- `bob capture buy milk s:1` writes and reports a line beginning `- [?] #task` with the
  same created and scheduled properties as today.
- `s:0`, rolled `p:<N>`, and `p:<N> s:<N>` task captures also begin `[?]`; priority
  values, seeded dates, and Schedule Log creation/non-creation behavior are unchanged.
- Scheduled tasks with authored block IDs end in the same `^block-id`, and scheduled
  Pomodoro-linked tasks are `[?]` while still receiving the same daily-ledger link.
- Unscheduled ordinary/ID-bearing tasks remain `[ ]`, unscheduled Pomodoro-linked tasks
  remain `[*]`, and every non-task bullet form remains checkbox-free.
- Human, dry-run, JSON preview, multi-item, and persisted-note representations agree on
  the rendered line, with no response-schema additions or linked-repository changes.
- User-facing help and capture documentation describe the new initial status without
  implying that all scheduled metadata-bearing bullets are tasks.

## Verification

Run the focused renderer and CLI capture tests first, then the repository's complete
quality gate:

```bash
cargo test native::capture::tests::formats_
cargo test --test cli capture_
just all
```
