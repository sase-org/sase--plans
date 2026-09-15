---
tier: epic
title: Deliver production settlement notification targeting
goal: 'Epic-launch monitor completion is notified exactly once only after the monitor''s
  terminal state is durable, and that existing notification targets the settled monitor
  plus its full family chain so ACE converges without watcher assistance.

  '
parent_bead: sase-117
phases:
- id: settlement_handoff
  title: Post-settlement notification handoff
  depends_on: []
  size: medium
  description: 'settlement_handoff: defer the existing epic-launch completion notification
    through monitor settlement and attach the settled monitor and family-root identities.'
- id: production_replay
  title: Production-path incident replay
  depends_on:
  - settlement_handoff
  size: small
  description: 'production_replay: replace the synthetic notification-only proof with
    a real monitored epic-launch replay and verify exact family-chain convergence
    and delivery invariants.'
proposed_by: bbugyi200.athena.sase-117.land
create_time: 2026-09-15 13:09:40
status: wip
bead_id: sase-117.5
---

- **PROMPT:** [prompts/202609/production_settlement_notification.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/production_settlement_notification.md)
- **PARENT:** [202609/ace_family_status_convergence.md](https://github.com/sase-org/sase--plans/blob/main/202609/ace_family_status_convergence.md)
- **BEAD:** [sase-117.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-117/sase-117.5.md)

# Plan: Deliver production settlement notification targeting

## Context

The `sase-117` landing audit confirmed that its queue-retention, merge/apply,
surface-token, and notification-resolution changes work as implemented. The focused
convergence, notification-polling, and epic-launch handoff suites pass. The audit also
found that the notification-only acceptance path is not connected to production:

- `finish_epic_launch()` receives the planner/root `artifacts_dir` passed to the
  monitored `sase bead work` command. `settlement_notification_action_data()` therefore
  emits `raw_suffix` and `family_root_suffix` with the same root timestamp.
- `_completion_notification_delta_dirs()` correctly walks ancestors when a notification
  identifies the settled monitor. For the production epic-launch notification, however,
  it starts at the root and schedules only the root artifact directory; it cannot
  discover the monitor or intermediate gate by walking upward from the root.
- The monitor supervisor currently strips the starter's `SASE_ARTIFACTS_DIR`, runs the
  command, and only after the command returns writes the monitor's terminal metadata,
  `done.json`, final workflow state, index update, and refresh pulse. Consequently the
  epic-launch notification is emitted before monitor settlement, as in the original
  incident timeline.
- The passing notification-only regression manually constructs a post-settlement
  `sender="monitor-settlement"` notification with the monitor timestamp. No production
  code emits that notification shape.

The fix must preserve the notification users already receive. It must not add a second
toast/bell, change the sender/action/tags/notes contract, add a TUI refresh path, or
depend on watcher delivery. Non-monitor epic-launch execution (including the leased proc
fallback) must retain its current immediate-notification behavior.

## Phase 1: Post-settlement notification handoff

Move monitored epic-launch notification publication behind the monitor's durable
terminal write while preserving exactly-once and recovery behavior.

- Give the monitored command an explicit, non-agent identity for its owning monitor
  artifact directory. Do not reuse or leak `SASE_ARTIFACTS_DIR`; the supervisor
  deliberately strips stale agent identity from child commands.
- When `finish_epic_launch()` runs under that monitor, persist the already-composed
  completion payload for the supervisor instead of appending the notification
  immediately. Retain the current immediate path when no owning monitor exists.
- After the supervisor has made `monitor_settled`, `done.json`, the final workflow
  state, the artifact-index mutation, and the refresh pulse durable, publish the
  deferred payload once. Apply the same recovery rule in monitor reconciliation so a
  supervisor crash cannot lose or duplicate the notification.
- Preserve the existing notification's sender, action, success/failure, notes, files,
  silence, and tags. Enrich only its targeting identity: `cl_name` / `raw_suffix`
  identify the settled monitor shell, while `family_root_suffix` (and the compatibility
  root key) identify the original planner/root artifact. Do not create a new
  user-visible `monitor-settlement` notification in addition to the existing
  epic-launch/user-agent completion.
- Keep best-effort notification failure semantics, bounded/atomic payload persistence,
  and orphan cleanup consistent with the existing epic-launch completion handoff.

Add focused tests for normal success, launch failure, deferred planner-completion
folding, early-settle ordering, non-monitor fallback, supervisor
recovery/reconciliation, and exactly-once publication. Assert that no notification is
observable before terminal monitor markers are durable and that the published action
data distinguishes monitor and root timestamps.

## Phase 2: Production-path incident replay

Exercise the actual emitter and monitor settlement rather than supplying a notification
shape directly to ACE.

- Extend the sandbox family harness to run the production monitored epic-launch
  completion handoff through terminal settlement, with watcher delivery disabled and the
  artifact-index update deliberately delayed where needed to retain the original
  incident pressure.
- Poll the stored production notification and pass it through the normal ACE
  notification polling/targeting seam. Assert that it schedules the exact monitor → gate
  → root artifact chain and that the monitor and mirrored family root both become
  `EPIC CREATED` on that notification tick.
- Keep a watcher-only replay green as an independent recovery path, and retain the
  existing exact-delta queue-loss and post-index-upsert coverage.
- Delete or rewrite synthetic `monitor-settlement` cases that could pass without a real
  producer. Unit coverage for the resolver may use constructed notifications, but the
  acceptance test must originate from the production emitter/handoff.
- Re-run the focused convergence, notification polling, epic-launch handoff, monitor
  supervisor/reconciliation, and auto-refresh suites. Confirm a quiet token-gated tick
  still reloads zero surfaces and run the existing Agents j/k benchmark guardrail
  because this is the final integration proof for `sase-117`.

## Landing evidence already established

The parent landing audit found no `PROPOSED FOLLOW-UP:` entries on phases `sase-117.1`
through `sase-117.4`, and `sase bead epic-symbols sase-117` currently reports no
entries. Six non-epic commits landed after the first parent commit; their retention,
usage, completion, bead-context, workspace, and sudo changes do not overlap the
convergence call paths. The sudo parser drift was already incorporated by the parent's
Phase 4 completion snapshot commit. No additional drift integration belongs in this
child plan.
