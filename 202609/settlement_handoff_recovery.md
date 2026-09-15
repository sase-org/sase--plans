---
tier: tale
title: Complete production settlement handoff recovery
goal:
  Epic-launch completion notifications survive every monitor settlement and recovery
  path while driving exact family-chain TUI convergence once.
size: medium
proposed_by: bbugyi200.athena.sase-117.5.land
bead: sase-117.5
status: done
---

- **PARENT:**
  [202609/production_settlement_notification.md](https://github.com/sase-org/sase--plans/blob/main/202609/production_settlement_notification.md)
- **BEAD:**
  [sase-117.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-117/sase-117.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-117.5.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-117.5.land.md)
- **COMMITS:**
  - [824ef83](https://github.com/sase-org/sase/commit/824ef831c88a5c78f33b3ae2cd0d308f5afd41b2)
    — fix(epic-launch): recover monitor completion handoff

# Plan: Complete production settlement handoff recovery

## Context

Epic `sase-117.5` added monitor-owned deferral of the existing epic-launch completion
notification in commit `dcfb1c1d` and taught notification-driven TUI refresh to target
the settled monitor-to-root family chain in `9afd1bb0`. The landing audit found that the
happy-path focused tests pass, but two requirements from the approved epic plan are not
actually complete:

- `src/sase/monitor/reconcile.py` settles a dead supervisor through durable
  `agent_meta.json`, `done.json`, workflow-state, index, and refresh-pulse writes
  without publishing a pending monitor-owned completion. The normal supervisor and
  proc-backed settlement paths do publish it, so a supervisor death can permanently lose
  the notification.
- The monitor handoff claim deletes the pending payload before notification storage is
  durable and does not write its monitor-settled marker on the successful publication
  call. A crash in that window can lose the notification, while a retry after an
  uncertain append cannot prove that it should not append a duplicate. Restored
  `*.monitor-pending.json` files and `*.monitor-settled.json` markers are also outside
  the existing orphan/TTL sweep.
- The TUI acceptance replay directly constructs `CompletionNotificationPayload` and
  directly calls the defer/publish helpers. It also gives the monitor a synthetic
  monitor-specific `cl_name`, whereas production monitor members inherit the family's
  Patch/bead `cl_name`. It therefore does not prove the requested production user-agent
  emitter -> `finish_epic_launch()` -> terminal monitor settlement -> stored
  notification -> exact TUI delta path.

The audit found no `PROPOSED FOLLOW-UP:` notes on either child phase. The non-epic
commits that landed after `dcfb1c1d` changed TUI self-update restart filtering, renamed
the public `ace` command to `tui`, adjusted lint-memory wording, and repaired routed
bead operations. None duplicates the handoff, but new tests and terminology must use the
current tree after that drift.

## Implementation

1. Make monitor-owned completion publication a recoverable, idempotent outbox transition
   in `src/sase/bead/epic_launch_handoff.py`.
   - Keep the pending payload durable until the notification append is known durable.
     Give the handoff a stable notification identity (including a backward-compatible
     derivation for already-written payloads) so recovery can distinguish "not sent"
     from "append succeeded before the publisher crashed".
   - Serialize competing publishers with the existing handoff lock. On replay, detect
     the already-durable notification by that stable identity, then complete the
     pending-to-settled transition without producing a second row, toast, bell, or
     notification `+1`. On an append failure that wrote nothing, retain a retryable
     pending payload. Preserve all sender, action, success/failure, notes, files,
     silence, tags, and targeting fields.
   - Preserve the early-settlement behavior: if terminal settlement wins the race, a
     later epic completion publishes immediately exactly once and consumes the marker.
     Extend orphan cleanup and settled-marker TTL handling to the monitor-owned file
     shapes so a best-effort publication failure is retried and stale state is bounded,
     without applying the legacy "unknown epic outcome" rewrite to an already-folded
     monitor payload.

2. Apply the same post-durability publisher at every monitor terminal boundary.
   - Retain publication after `monitor_settled`, `done.json`, final workflow state,
     artifact-index mutation, and refresh pulse in the standalone supervisor and
     proc-backed settlement paths.
   - Add the missing call to dead-supervisor reconciliation only after it has made the
     same terminal state durable. Repeated reconciliation/settlement calls must remain
     notification-idempotent, and monitor settlement must remain notification-neutral
     when there is no deferred epic completion.

3. Replace the synthetic acceptance seam with the production handoff.
   - In the family convergence replay, emit the original planner/user-agent completion
     through its real completion emitter, fold it through `finish_epic_launch()` under
     the owning-monitor environment, and settle through the production proc or
     supervisor settlement entry point. Use production-shaped monitor metadata where the
     family `cl_name` is inherited and the monitor is distinguished by its name, role,
     parent timestamp, and raw artifact timestamp.
   - Keep watcher delivery disabled for the notification acceptance path. Delay the
     artifact-index mutation at the established test seam while allowing terminal
     markers and the refresh pulse to become durable, then load the stored production
     notification through normal polling/targeting and assert the exact monitor -> gate
     -> root delta and `EPIC CREATED` convergence.
   - Retain the watcher-only recovery replay, post-index-upsert convergence, exact-delta
     queue-loss coverage, and the quiet token-gated tick invariant.

4. Add focused regression coverage for both ordering and recovery.
   - Cover normal success, launch failure, folded planner completion, early settlement,
     non-monitor immediate fallback, standalone supervisor settlement, proc settlement,
     and dead-supervisor reconciliation.
   - Assert no notification is observable before terminal monitor markers and refresh
     state are durable; assert the stored action data carries the monitor raw suffix and
     original family-root compatibility keys while preserving the existing
     user-agent/epic-launch notification contract.
   - Simulate retry before append and an uncertain crash/failure after durable append;
     both recovery attempts must converge to one stored notification. Cover monitor
     pending retry and monitor settled-marker reaping through the orphan sweep.

## Verification

- Run the focused epic-launch handoff, monitor supervisor/proc/reconciliation, TUI
  convergence/notification polling, and auto-refresh suites.
- Run the existing Agents j/k benchmark guardrail and confirm a quiet token-gated tick
  reloads zero surfaces.
- Run `just check` after `just install`, following the repository verification policy.
