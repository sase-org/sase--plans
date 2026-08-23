---
tier: tale
title: Fix completion notifications to identify the owning agent family
goal:
  Completion notifications present the owning agent family while preserving exact
  finishing-shell navigation and diagnostics.
size: small
proposed_by: bbugyi200.athena.0bz
create_time: 2026-08-23 15:15:17
status: wip
---

# Fix completion notifications to identify the owning agent family

## Goal

Make successful and failed agent-completion notifications identify the owning SASE agent
family instead of the concrete shell that happened to finish last, including monitor
follow-up shells such as `0bw--1`. Preserve the concrete artifact identity used to
navigate to and inspect the finishing shell.

## Root cause

Monitor `p8z6s9ejrqtk` records family `0bw`, starter shell `0bw--code`, monitor shell
`0bw--mon`, and follow-up shell `0bw--1`. The follow-up is correctly launched as a
member of family `0bw`, and its `agent_meta.json` carries the authoritative
`agent_family` value.

At shutdown, `RunnerRunState.agent_name` still represents the concrete executing shell.
`finalize_runner_shutdown()` passes that value unchanged to
`send_completion_notification()`, which currently interpolates it into both the visible
note and `action_data.agent_name`. The stored notification therefore contains
`agent_name: 0bw--1` and a note naming `@0bw--1`. Telegram then faithfully uses that
field for both its header and Fork button, so Telegram is not the source of the defect.
The missing step is projection from shell execution identity to the user-facing owner
identity at the SASE completion-notification boundary.

## Implementation

1. In `src/sase/axe/run_agent_runner_finalize.py`, add a small best-effort resolver for
   the completion notification's user-facing agent identity. Read the current artifact
   directory's `agent_meta.json` and prefer its non-empty `agent_family`; otherwise keep
   the supplied `agent_name` unchanged. Do not infer a family from dotted names, because
   valid standalone/bead agent names such as `sase-x.3` can resemble legacy family
   suffixes. Keep malformed or missing metadata notification-safe by falling back to the
   original name.
2. Use the resolved family-facing name consistently for the completion note,
   `action_data.agent_name`, and bead-display lookup in both `JumpToAgent` and
   `ViewErrorReport` payloads, including payloads later deferred for epic launch.
   Continue using the existing `cl_name` and `raw_suffix` values for exact
   shell/artifact navigation, unread projection, attachments, and diagnostics; do not
   rewrite artifact metadata or monitor/follow-up naming.
3. Extend `tests/test_run_agent_runner_notifications.py` (and its shared fixture only if
   useful) with regression coverage that writes family metadata for a numeric monitor
   follow-up shell and asserts that the emitted note and action data name the bare
   family while retaining the finishing shell's `raw_suffix`. Cover both success and
   error-report payload construction, plus missing/malformed metadata and a dotted
   standalone/bead name to prove fallback identities are not truncated.

## Validation

1. Run the focused completion-notification tests, including the new family-projection
   cases and existing suppression/deferred-epic cases.
2. Run the notification navigation/unread tests that depend on `cl_name` and
   `raw_suffix` to confirm exact finishing-shell routing and family-node projection are
   unchanged.
3. Run `just install`, then `just check` as required for changes in the SASE repository.
   If scoped selection escalates or reports an unusual selection, use `/sase_monitor` to
   run `just check-full` with the required `TESTING`/`TESTED` statuses and a concrete
   follow-up action.

## Non-goals

- Do not change monitor lane, starter, monitor-member, or follow-up allocation.
- Do not restore monitor-generated notifications; monitors remain notification-neutral.
- Do not modify `sase-telegram`: its formatter correctly renders and forks the
  `action_data.agent_name` supplied by SASE.
- Do not add a feature flag; this is a correction to existing user-facing identity
  semantics with a safe standalone fallback.
