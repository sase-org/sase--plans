---
tier: tale
title: Refresh pending-review agent status on the same tick as the gate toast
goal:
  When ACE toasts a new PlanApproval, EpicApproval, or UserQuestion notification, the
  Agents tree shows the pending TALE, EPIC, or QUESTION status in the same poll, using
  the existing exact artifact-delta path rather than waiting for a later auto-refresh.
size: medium
proposed_by: bbugyi200.athena.0nq
create_time: 2026-09-19 08:54:55
status: wip
---

# Refresh pending-review agent status on the same tick as the gate toast

## Goal

When a planner proposes a tale or epic, or an agent asks a question, ACE already shows
the review toast as soon as the notification store watcher fires. In that same poll the
Agents tree must show the pending gate status (`TALE`, `EPIC`, or `QUESTION`) on the
family that owns the gate. Today the toast lands immediately and the row stays `DONE`
until a later auto-refresh or sanity load.

This is one medium tale. It does not own post-acceptance labels, receipt refresh, or
Telegram.

## Why this is not sase-zr

`sase-zr` / `sase-zr.7` / phase `sase-zr.7.3` cover the **acceptance** moment: durable
receipts, `TALE APPROVED` vs `PLAN COMMITTED`, disappearance of answered notifications,
and making the **acceptance** pulse target exact rows. The owner's remaining pain there
is "gate responses made from the TUI are slow to appear."

This bug is the **arrival** moment. The 0nn incident
(`file:explicit:46fe660fb57b808d40f13e5f`) toasted `Tale ready for @0nn` at 07:45:17
while the family still showed `DONE` at 07:45:26. The pending gate member `0nn--gate`
already existed at `20260919074517` with `pending_status: TALE`. sase-zr.7.3 will not
wire new PlanApproval rows into the toast poll, and this tale must not rewrite the
receipt, disappearance, synchronous count, or `partial_attempt` work that phase owns.

Reuse, do not fork, the settlement targeting machinery from `sase-117.5` /
`plan:202609/ace_family_status_convergence.md`: exact family-chain artifact deltas on
the notification's own tick, no new refresh spine.

## Incident (athena, 2026-09-19)

Notification `780f9d14-cbe1-4f4c-9fe9-21802ab7a447`:

- 07:44:40 EDT — `sase plan propose` wrote `.sase_plan_pending`, pulsed
  `artifacts/.ace_refresh_pulse`, and SIGTERM'd 0nn (`plan_submitted_at`).
- 07:45:17 EDT — gate shell `0nn--gate` materialized (`20260919074517`); `request.json`
  `created_at` and the PlanApproval notification share this second.
- 07:45:26 EDT — screenshot: toast visible, 0nn still `DONE`, header
  `auto-refresh in 9s`.

The 37s propose-to-gate stall is **out of scope** (canceled `sase-ww`). The user-facing
defect is the ≥9s desync **after** the notification exists.

## Confirmed diagnosis

Family presentation already mirrors the pending gate child (`apply_status_overrides` /
`gate_start_status`). The TALE row is a **new** artifact directory, not a mutation of
the planner dir. Until that child is loaded and merged, the planner stays `DONE`.

Three independent misses keep the child off the toast tick:

1. **Notification poll does not target pending-review gates.**
   `_poll_agent_completions_once` toasts every new unread row, then records
   `_last_new_completion_notifications` only for `is_active_agent_refresh_notification`
   (user-agent completions and epic-launch / monitor-settlement senders). `PlanApproval`
   / `EpicApproval` / `UserQuestion` are excluded. `request_notification_agents_refresh`
   therefore sees an empty list, finds no exact dirs, and either no-ops off the Agents
   tab (`allow_broad_fallback=False`) or falls back to a broad `latest_only` load on the
   Agents tab.

2. **Broad fallback cannot be the same-tick contract.** A bounded/incomplete load after
   complete history is patched through `merge_incomplete_load_after_complete_history`.
   That path is the wrong tool for inserting one new family child and remirroring the
   root. Completions already avoid it by scheduling
   `_schedule_agent_artifact_delta_refresh` with the new dirs, including unloaded
   timestamps via `artifact_dirs_for_normalized_timestamps`. Pending-review gates need
   that same path. The gate member timestamp is not in today's PlanApproval
   `action_data` (`agent_timestamp` is the planner's `20260919072902`, not
   `20260919074517`).

3. **The producer pulse fires before the TALE member exists.**
   `plan_propose_handler._touch_shell_refresh_pulse_for_artifacts_dir` runs before
   SIGTERM. `create_gate_shell` / `create_gate_shell_member` do not pulse after the
   member is indexed and the notification is written. The artifacts watcher is
   non-recursive on `artifacts/`, so the new sharded dir
   `ace-run/<YYYYMM>/<DD>/<timestamp>` is invisible. The toast watcher is a separate
   inotify surface, which is why the chip can appear while Agents stay stale until the
   next auto-refresh (5s load floor, ~10s tick in the screenshot).

Canceled `sase-zc` was the wrong-directory propose pulse; that path now writes
`artifacts/.ace_refresh_pulse`. It does not cover gate-creation time.

## Constraints

Read `tui_perf` before editing TUI refresh code. Acceptance criteria:

- No new refresh spine. Route through `_schedule_agent_artifact_delta_refresh` and the
  existing notification poll (`_poll_agent_completions_once` /
  `_run_scheduled_notification_poll`).
- Do not shorten `AGENTS_LOAD_MIN_INTERVAL_SECONDS` or `FULL_SANITY_REFRESH_SECONDS`.
- Do not restore reload-everything, per-tick index revalidation, or
  `not watcher_active ⇒ reload everything`.
- Exact deltas only (planner + gate member + family root). Broad fallback stays
  tab-gated and last-resort, as today.
- Disk / `find_gate_shell_by_gate_id` work stays on the worker thread that already
  prepares notification reconciliation. Use the indexed exact lookup; do not call
  `list_gate_shells` from the poll.
- Quiet idle ticks still reload no surfaces (`refresh.auto_tick`). j/k p95 stays on the
  existing bench recipe.
- Do not edit sase-zr.7.3's files/responsibilities: `_request_gate_decision_refresh`,
  `prepare_disappeared_plan_notification_refresh`, `_refresh_notification_count`,
  `_notification_gate_execution.py` journal reads, `_notification_plan_gate.py`
  `partial_attempt`, or `decision.py::_touch_gate_shell_refresh_pulse`.
- If sase-zr.7.3 has already landed a shared "notification → shell + planner + family
  dirs" helper, call it for arrivals rather than forking a second resolver. Otherwise
  add an arrival-specific helper next to `_completion_notification_delta_dirs` and leave
  the decision-refresh function untouched.

## Implementation

### 1. Stamp the pending shell on the notification

After `create_gate_shell_member` returns and before `create_gate` publishes the row,
attach the new member's identity to the gate spec / notification `action_data`. Mirror
the sase-117.5 settlement fields so ACE can resolve an unloaded dir:

- gate member `raw_suffix` (14-digit artifacts timestamp)
- gate member `artifacts_dir`
- keep existing planner `agent_timestamp` / `agent_root_timestamp` / `artifacts_dir`

Do this in the shell transaction (`src/sase/gate_shell/transaction.py`) so tale, epic,
and question shells all get it, not only `plan_gate._plan_action_data`. Legacy in-flight
notifications without the new keys remain valid; the poll falls back to indexed
`find_gate_shell_by_gate_id(request_id)` on the worker thread.

### 2. Same poll: toast and exact family-chain delta

In `src/sase/ace/tui/actions/agents/_notification_utils.py` (and the poll that already
calls it):

- Treat active (not dismissed) `PlanApproval`, `EpicApproval`, and `UserQuestion` as
  refresh notifications. Keep this as a sibling predicate; do not overload settlement
  senders.
- Resolve exact dirs: gate member, planner, family root. Unloaded suffixes go through
  `artifact_dirs_for_normalized_timestamps`, the same as completions.
- `_poll_agent_completions_once` must include those notifications in
  `_once_new_completion_notifications` so the toast tick and
  `request_notification_agents_refresh` share one list.
- Off-tab exact deltas stay allowed; unresolvable rows stay tab-gated for broad
  fallback.

`apply_status_overrides` already mirrors `TALE` / `EPIC` / `QUESTION` once the child is
in the roster. The delta apply path must insert the new child and remirror the cached
family root (BY_STATUS grouping membership will change; that is an allowed rebuild of
the affected group, not a full Agents reload).

### 3. Pulse after the gate is actually visible

At the end of a successful `create_gate_shell` (member indexed, notification id
recorded), touch a refresh pulse. Use `touch_shell_refresh_pulse` so we do not invent a
second pulse writer. This is a backup for sessions that miss the notification poll; the
same-tick contract is step 2.

Do not move or duplicate the early propose-time pulse; it still wakes the TUI for the
planner's DONE transition. The new pulse is the one that can name the TALE member.

## Tests

Extend `tests/test_notification_toast_polling_agent_refresh.py` (and a narrow gate-shell
creation test) with real dirs, not mocks of the resolver:

1. A new PlanApproval toast poll schedules one exact delta covering planner + gate
   member + family root, even when the gate dir is **not** in `_agents_with_children`.
   Assert no broad `request_agents_refresh`.
2. The same poll still toasts. Off the Agents tab, the exact delta still runs.
3. EpicApproval and UserQuestion share the predicate and targeting.
4. A cached family whose planner is `DONE` plus a delta that adds a pending `TALE` gate
   child remirrors the family/root to `TALE` through
   `merge_incomplete_load_after_complete_history` / prepared apply.
5. `create_gate_shell` writes the new `action_data` keys and pulses only after the
   member exists (pulse mtime ≥ member dir; pulse is not under `ace-run/<YYYYMM>/`).
6. A legacy PlanApproval without the new keys still resolves via
   `find_gate_shell_by_gate_id` on the worker path.
7. Idle-tick guard: the new predicate does not make a quiet tick reload Agents when
   there is no new notification.

Do not add wall-clock TUI tests. Do not touch real user gates.

## Verification

- `just fix` then `just check`.
- If the change touches TUI refresh selection, confirm `tools/select_tests --explain`
  includes the toast-polling and delta-merge tests.
- Do not run `just check-full` unless `just check` escalates.

## Out of scope

- Propose-to-gate handoff duration / telemetry (`sase-ww`).
- `TALE APPROVED` / `PLAN COMMITTED` honesty (`sase-zr.7.2`).
- Receipt, disappearance, count-refresh, and `partial_attempt` (`sase-zr.7.3`).
- Settlement `EPIC APPROVED` → `EPIC CREATED` (`sase-117` / `sase-117.5`).
- Making the artifacts watcher recursive.
