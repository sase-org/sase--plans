---
tier: tale
title: Stop stamping task sub-bullets with created dates
goal:
  Task sub-bullets render without a created Dataview property while their scheduling,
  nesting, output metadata, and capture behavior remain intact.
create_time: 2026-09-09 19:53:27
status: wip
---

# Stop stamping task sub-bullets with `created`

## Goal

Make every `bob capture` task sub-bullet render as an ordinary child bullet without a
`[created::YYYY-MM-DD]` inline Dataview property. Keep the rest of the sub-bullet
contract intact: a requested `s:<N>` marker still produces `[scheduled::YYYY-MM-DD]`,
clipboard content remains nested under the new child, note indentation and line endings
remain unchanged, and JSON output continues to report the capture date in its stable
`created` metadata field.

## Implementation

1. In `src/native/capture.rs`, separate sub-bullet rendering from ordinary
   section-bullet rendering instead of routing both `CaptureKind::Bullet` and
   `CaptureKind::SubBullet` through `format_bullet_line`. Add a focused sub-bullet
   formatter that emits `- <body>` and appends only the optional scheduled property.
   Leave task, Pomodoro-task, and ordinary section-bullet formatting unchanged,
   including their existing created stamps.
2. Extend the formatter unit coverage in `src/native/capture.rs` to pin both sub-bullet
   forms: an unscheduled child has no Dataview properties, while a scheduled child
   contains `[scheduled::...]` but never `[created::...]`.
3. Update the existing sub-bullet CLI regression cases in `tests/cli.rs` for block-ID
   markers, hidden task refs, the public `--task` option, clipboard children,
   indentation selection, CRLF preservation, dry runs, and JSON `task_line` output so
   their expected inserted child lines are unstamped. Explicitly retain/assert the JSON
   `created` date to distinguish stable result metadata from the removed inline Dataview
   property.
4. Update the `bob capture` long help in `src/native/capture.rs` and the sub-bullet
   example/explanation in `README.md` to say that task children are ordinary bullets
   with no created stamp, while scheduled captures may still carry the scheduled
   property. Keep the documented created-stamp behavior for top-level tasks and ordinary
   section bullets.

## Validation

1. Run the focused formatter unit tests and the `capture_sub_bullet` integration-test
   group to exercise each supported sub-bullet entry path and output mode.
2. Run `cargo fmt --check` and `git diff --check`.
3. Run `just all` to verify formatting, Clippy across all targets/features, the full
   Rust test suite, and doc tests.

## Acceptance criteria

- Sub-bullets created through `@<route>^<block-id>`, `--route ... --task`, and hidden
  `--task-ref` render without `[created::...]` in both the note and reported
  `task_line`.
- Scheduled sub-bullets retain `[scheduled::YYYY-MM-DD]`, and no top-level task,
  Pomodoro task, or ordinary section-bullet created-stamp behavior regresses.
- Clipboard payloads, parent selection, indentation, line endings, dry-run behavior,
  human output, and JSON parent metadata remain unchanged; JSON still exposes the
  capture date as `created`.
- Public help and README examples match the new rendered output, and all focused and
  repository-wide checks pass.
