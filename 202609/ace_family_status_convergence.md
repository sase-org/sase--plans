---
tier: epic
title: Converge ACE family status with settlement notifications
goal: "When a family shell settles (e.g. an epic-launch monitor flips EPIC APPROVED to
  EPIC CREATED), the ACE agents tree converges to the new status within one auto-refresh
  tick of the settlement notification, through every load path, with no new TUI
  performance cost.

  "
phases:
  - id: repro-harness
    title: Deterministic repro harness
    depends_on: []
    size: medium
    description:
      "repro-harness: build a sandbox-home pytest harness that replays the recorded
      stale-family incident step by step through the real loader/merge/apply seams, pins
      the exact in-session pinning mechanism, and commits xfail tests plus a unit test
      for the exact-delta queue-loss defect."
  - id: convergence-fix
    title: Convergence fix
    depends_on:
      - repro-harness
    size: medium
    description:
      "convergence-fix: stop dropping queued exact artifact dirs in fallback state, fix
      the mechanism Phase 1 pinned in the merge/token-accept/apply path, and turn the
      harness xfails green under the tui_perf acceptance criteria (no new refresh paths,
      quiet idle ticks, unchanged j/k p95)."
  - id: settlement-notify-targeting
    title: Settlement notification targeting
    depends_on:
      - repro-harness
    size: medium
    description:
      "settlement-notify-targeting: carry the settled shell's identity in
      monitor-settlement and epic-launch notification action_data and resolve those
      notifications to an exact family-chain artifact-dir delta so the status flip lands
      on the notification's own tick."
  - id: incident-verification
    title: Incident verification
    depends_on:
      - convergence-fix
      - settlement-notify-targeting
    size: small
    description:
      "incident-verification: run the integrated harness green without xfails, confirm
      quiet-tick and j/k bench guardrails, and replay the live-shaped epic-launch
      scenario through the watcher-only and notification-only paths."
proposed_by: bbugyi200.athena.0l7
create_time: 2026-09-15 09:49:36
status: wip
---

- **PROMPT:**
  [prompts/202609/ace_family_status_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/ace_family_status_convergence.md)

# Converge ACE Family Status With Settlement Notifications

## Goal

When a family shell settles (e.g. an epic-launch monitor flips `EPIC APPROVED` ->
`EPIC CREATED`), the ACE agents tree must converge to the new status within one
auto-refresh tick of the settlement notification, through every load path, with no new
TUI performance cost. Today a family root can stay pinned on the stale status for many
minutes (until a complete-history reload or a TUI restart) even though the on-disk
state, the artifact index, and the notification are all correct.

## Background: incident forensics (2026-09-15, times UTC)

Symptom: family root `0l4` stayed at `EPIC APPROVED` with its `--mon` monitor shell
rendered as running for 8+ minutes after its epic launched; a TUI restart rendered it
correctly. The family is a chain: plan-workflow root, gate shell child
(`parent_timestamp` = root), monitor shell grandchild (`parent_timestamp` = gate). The
monitor was started by `src/sase/bead/epic_launch.py` with
`start_status="EPIC APPROVED"`, `stop_status="EPIC CREATED"`.

Host-side settlement was fully correct and prompt:

- 13:05:26 - epic-launch notification appended (sender `epic-launch`, `action: null`,
  empty `action_data`, unread, not silent).
- 13:05:30 - monitor supervisor `_finish_monitor` wrote `agent_meta.json`
  (`monitor_settled`, `monitor_state=completed`), `done.json`
  (`status_label: EPIC CREATED`), finalized workflow state, and touched the project
  `.ace_refresh_pulse`.
- 13:05:43 - the artifact index row for the monitor dir reached
  `status=completed, has_done_marker=1` (the settlement upsert lagged ~13 s, consistent
  with index operation-lock contention).

TUI-side refresh _scheduling_ also fired exactly as designed (from
`tui_agent_loads.jsonl`): a notification-source full load at 13:05:37, a watcher
artifact-delta of 3 dirs at 13:05:49, and auto-refresh full loads at 13:06:17 and
13:07:28. Despite all of that, at 13:07:55 the tree row, the family-shells panel, and
the live-counting elapsed timer all still showed the pre-settlement model. A fresh
session at 13:13 (startup + prefix completion + `tier1_index_revalidate`) rendered
correctly.

Post-hoc, every individually testable component behaves correctly on settled data:

- A delta load of the monitor dir alone produces a correctly linked (`parent_timestamp`
  -> gate) `EPIC CREATED` row.
- `merge_incomplete_load_after_complete_history` applied to that delta over an
  artificially staled cached family flips the root to `EPIC CREATED` (the still-open,
  settling gate row - which presents as an active `EPIC APPROVED` row - does not win
  root mirroring).
- A viewport-bounded index-backed Tier-1 load (~170 rows, matching the incident's load
  sizes) includes the whole family and computes root `EPIC CREATED`.
- Tier-1 merge keys for the monitor row agree between the index-backed and direct-scan
  load paths.

So the defect lives in in-session state interaction (the pre-settlement cached row
shape, exact-delta queue loss, and/or the app-level merge/apply path), not in any single
loader on settled data. Two contributing defects are already confirmed by code
inspection:

1. **Exact-delta queue loss.** `_enqueue_agent_artifact_delta_paths`
   (`src/sase/ace/tui/actions/event_refresh/_artifact_delta.py`) returns early and drops
   the incoming changed paths entirely whenever `_dirty_agent_artifact_fallback_reason`
   is already set. Settlement markers are written exactly once; if that batch arrives
   while the fallback flag is set, no later filesystem event ever re-covers the settled
   dir. The compensating broad load is viewport-bounded and index-backed with
   `index_freshness="cached"`, so during the window where the settlement index upsert is
   still lock-delayed it re-serves the pre-settlement row - and `_accept_surface_token`
   then absorbs the pulse-driven token drift, so idle ticks stop retrying.
2. **Settlement notifications are not completion-shaped.** The epic-launch completion
   notification (`src/sase/bead/epic_launch.py` / `src/sase/bead/epic_launch_handoff.py`
   via `notify_workflow_complete`) and monitor settlement notifications carry no
   `cl_name` / `raw_suffix` in `action_data` and do not satisfy
   `is_active_agent_completion_notification` (sender `user-agent`, action
   `JumpToAgent`/`ViewErrorReport`). `active_completion_agent_keys`
   (`src/sase/ace/tui/actions/agents/_notification_utils.py`) therefore cannot resolve
   them to an exact artifact-dir delta; notification-triggered reconciliation degrades
   to a broad bounded load. This breaks the intended contract that a gate/status change
   lands in the tree on the same tick its notification is delivered.

## Constraints

- Read the `tui_perf.md` sase memory before touching any refresh/loading code; its rules
  are acceptance criteria here. In particular: route refreshes through the existing fast
  paths (no new refresh code paths), prefer exact deltas and selective row updates over
  broad rebuilds, keep idle ticks quiet (`ace_refresh_tokens` stat-probe gating stays),
  and do not "fix" convergence by restoring reload-everything behavior, shortening the
  sanity interval, or forcing per-tick index revalidation.
- All expected changes are ACE TUI refresh/loading logic and Python notification
  emitters in this repo. If the repro pins the defect inside the Rust index/scan
  bindings instead, stop and record a follow-up proposal on the phase bead rather than
  working around core behavior in Python.

## Phases

### Phase 1: Deterministic repro harness

Build a pytest harness under `tests/ace/tui/` that reproduces the incident sequence
against a sandbox SASE home (temp projects root + artifact index), using the real
loader/merge helpers rather than mocks:

- Fixture: a plan-family chain (plan-workflow root dir; gate shell child; monitor shell
  grandchild with `monitor_start_status="EPIC APPROVED"` /
  `monitor_stop_status="EPIC CREATED"`) in its pre-settlement shape (running monitor, no
  `done.json`, live-runner semantics), plus enough sibling agent dirs to make
  viewport-bounded loads meaningful. Build the artifact index from that state.
- Replay the recorded sequence, asserting the family root and monitor member status
  after each step:
  1. viewport-bounded index-backed load caches the roster (root mirrors the running
     monitor: `EPIC APPROVED`);
  2. settlement lands on disk (`done.json`, `monitor_settled` meta, finalized workflow
     state, refresh pulse) while the index row is deliberately left stale;
  3. notification-source viewport-bounded broad load;
  4. watcher artifact-delta that excludes the monitor dir (simulating the queue loss);
  5. index upsert for the monitor dir;
  6. subsequent auto-refresh viewport-bounded loads. The suite must assert convergence
     to root `EPIC CREATED` no later than the first load after step 5, and must exercise
     the real app-level seams (`merge_incomplete_load_after_complete_history`, the
     prepared-apply path, and the auto-refresh surface-token accept logic) so the
     failing step pinpoints the actual pinning mechanism.
- Add a focused unit test for confirmed defect (1): changed marker paths delivered to
  `_enqueue_agent_artifact_delta_paths` while `_dirty_agent_artifact_fallback_reason` is
  set must not be silently lost.
- Deliverable: committed tests that expose the defects (mark the ones that fail on the
  current tree `xfail` with a reference to this plan), plus a written pinpoint of the
  exact pinning mechanism (which step, which code path, which cached-row fields)
  recorded on this phase's bead for Phase 2 to consume.

### Phase 2: Convergence fix

Fix the pinned mechanism so every Phase 1 test passes with its `xfail` markers removed:

- Stop dropping queued exact artifact dirs in `_enqueue_agent_artifact_delta_paths` when
  a fallback reason is set: retain them (bounded, deduped) so that whichever refresh
  consumes the fallback also covers them with an exact delta, or force-includes them in
  the load it schedules.
- Apply whatever additional fix Phase 1 pinned. Candidate areas, in likelihood order:
  cached-row retention/dedup in
  `src/sase/ace/tui/actions/agents/_loading_compute_merge.py`; surface-token acceptance
  in `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py` absorbing drift for work
  the tick did not actually cover (prefer leaving the token un-accepted over widening
  reloads); the prepared-apply/row-patch path.
- Perf acceptance criteria (from the `tui_perf.md` memory): no new refresh code paths;
  no per-tick index revalidation or unconditional broad reloads; a quiet idle tick still
  reloads zero surfaces (verify with the `refresh.auto_tick` trace counters); j/k p95
  unchanged per the existing bench recipes in `docs/perf_runbook.md`.

### Phase 3: Settlement notification targeting

Make settlement notifications resolvable to exact rows so the status flip lands on the
same tick the notification is delivered, even when watcher events were missed:

- Include the settled shell's identity (`cl_name`, `raw_suffix`, and the family root
  suffix) in `action_data` for monitor-settlement and epic-launch completion
  notifications (emitters: `src/sase/monitor/settlement.py` / the senders it uses, and
  `src/sase/bead/epic_launch.py` + `src/sase/bead/epic_launch_handoff.py`), without
  changing their sender/action UX semantics (toast text, tabs, and gating must not
  change).
- Teach `active_completion_agent_keys` / `request_notification_agents_refresh`
  (`src/sase/ace/tui/actions/agents/_notification_utils.py`) to resolve these settlement
  notifications to an exact artifact-dir delta covering the settled shell dir plus its
  family root and intermediate shell dirs (the family is a chain, so root mirroring
  needs the whole chain refreshed). Keep the existing broad fallback for unresolvable
  notifications, tab-gated as today.
- Tests: unit coverage for the poll -> targeted-delta path with these notification
  shapes, and extend the Phase 1 harness so convergence is asserted already at the
  notification step (step 3) once this lands.

### Phase 4: Incident verification

End-to-end confirmation on the integrated tree:

- Run the full Phase 1 harness green with all `xfail` markers removed.
- Run the quiet-tick check (`refresh.auto_tick` with `SASE_TUI_TRACE=1`: a quiet tick
  reloads no surfaces) and the j/k bench comparison per `docs/perf_runbook.md` to
  confirm no responsiveness or idle-CPU regression.
- Script one live-shaped replay (epic-launch monitor family: approve -> monitor start ->
  settlement) against a sandbox home and confirm the roster converges within one tick of
  the settlement notification through the notification-targeting path alone (watcher
  disabled), and through the watcher path alone (notifications suppressed).
- Summarize the verification results on this phase's bead.
