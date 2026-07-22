---
tier: tale
title: Show durable plan basenames in approval toasts
goal: Plan and epic approval toasts identify the archived SASE plan by its actual
  basename instead of the gate bundle's generic plan.md resource name.
create_time: 2026-07-22 12:32:15
status: wip
---

- **PROMPT:** [202607/prompts/plan_approval_toast_basename.md](prompts/plan_approval_toast_basename.md)

# Plan: Show durable plan basenames in approval toasts

## Context and root cause

`sase plan propose` consumes the authored scratch file and moves it into the durable `~/.sase/plans/<YYYYMM>/` archive
before the runner creates the approval gate. The runner passes that archived path to `create_plan_approval_gate`, and
the gate already preserves it as `original_plan_file` in both its payload and notification `action_data`.

For review and editing, every plan gate deliberately materializes a bundle-local resource named `plan.md`; the
notification's `files` list therefore points at `<gate bundle>/plan.md`. The TUI formatter in
`src/sase/ace/tui/actions/agents/_toasts.py` currently derives the displayed name from the first item in
`Notification.files`, so an agent-associated plan or epic toast reports `plan.md` even though the notification also
carries the actual archived plan path.

Keep the internal `PLAN_RESOURCE_PATH = "plan.md"` contract unchanged: the edit operation, validation flow, modal
preview, and gate resource hashing all depend on that stable bundle-local name. This change is only about choosing the
semantic plan identity when formatting the toast.

## Implementation

1. Update the shared `PlanApproval`/`EpicApproval` branch in `src/sase/ace/tui/actions/agents/_toasts.py` to prefer the
   basename of the nonblank `action_data["original_plan_file"]` value. This is the durable archived path supplied by the
   plan gate and is the source of truth for the user-facing plan identity.
2. Preserve compatibility with older or independently-created notifications by falling back to the basename of the first
   `files` entry when `original_plan_file` is absent or blank. Preserve the existing note fallback when there is no
   usable agent/name pair and the existing generic `Plan ready for review` fallback when neither identity nor note is
   available.
3. Continue displaying only the basename, not the full home-directory path, and apply the same selection behavior to
   both tale (`PlanApproval`) and epic (`EpicApproval`) notifications.

## Tests

1. Extend `tests/test_notification_toasts.py` with the real gate shape: `files` points to a bundle-local `plan.md`,
   while `action_data.original_plan_file` points to a descriptively named archived plan under `~/.sase/plans/<YYYYMM>/`.
   Assert that the toast uses the archived basename and never the generic resource name.
2. Cover both `PlanApproval` and `EpicApproval` through the shared formatting path, and retain explicit coverage that a
   legacy notification without `original_plan_file` still uses the first file basename and that missing identity data
   still uses the note/default fallbacks.
3. Run `just install` before repository checks as required for an ephemeral workspace, run the focused
   notification-toast tests for quick feedback, and finish with `just check` to exercise formatting, lint/type checks,
   SASE validation, and the full test suite. No PNG snapshot update should be needed because this changes dynamic toast
   text rather than a stable visual fixture.

## Acceptance criteria

- A toast backed by archived plan `~/.sase/plans/202607/agent_group_clan_collapse_precedence.md` reads
  `Plan ready for @<agent>: agent_group_clan_collapse_precedence.md`, even though its editable notification resource is
  `<bundle>/plan.md`.
- Tale and epic approval toasts share the corrected behavior.
- Legacy notifications without `original_plan_file` keep their current file, note, and generic fallback behavior.
- The notification-gate resource and approval/editing protocols remain unchanged.
