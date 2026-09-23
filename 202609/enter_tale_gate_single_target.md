---
tier: tale
size: small
title: Collapse Enter on a tale-plan family to its single tale gate
goal:
  Pressing Enter on a plan family whose tale gate is pending opens the tale gate's
  PlanApproval notification directly, with no "Act on" chooser and no duplicate "Review
  plan" row, even when the TUI loaded the gate row before the gate shell recorded its
  bundle path and notification id.
proposed_by: bbugyi200.athena.0pv
create_time: 2026-09-23 09:51:44
status: wip
---

# Collapse Enter on a tale-plan family to its single tale gate

## Symptom

On the Agents tab, selecting the stopped `0pt` plan family (`--plan` DONE, `--gate`
pending with status `TALE`) and pressing `<enter>` opened the "Act on 0pt" chooser with
two GATES rows:

1. `Review plan · PLAN · 35s · plan.md`. This is a **notification-only** target built
   from the PlanApproval notification
   (`_agent_enter_builders.notification_gate_target`).
2. `Review tale plan · TALE · 36s · wait for gate decision`. This is the **gate-row**
   target built from the `0pt--gate` member row (`gate_row_target`). Its detail falling
   back to `gate_reason` shows that `linked_notification()` returned `None`.

Both rows describe the same gate. The durable state agrees: there is exactly one
notification (`028c8e26…`, action `PlanApproval`, `action_data.request_id` =
`6a29cf58…`, `bundle_path` = `~/.sase/interaction_requests/plan/6a29cf58…`). The gate
shell's `agent_meta.json` carries that same `gate_id`, `gate_bundle_path`, and
`gate_notification_id`. Enter should have produced one target and dispatched it directly
(`_dispatch_agent_enter_resolution` skips the chooser when `len(targets) == 1`). Picking
row 2 was also broken: its `notification_id` and `bundle_path` were `None`, so
`_run_agent_enter_gate_target` would toast "Gate … is no longer pending".

## Root cause

The in-memory `0pt--gate` row was **stale**. It had been loaded from an
`agent_meta.json` written before the gate-shell transaction recorded `gate_bundle_path`
and `gate_notification_id`, and nothing re-read it afterwards.

1. **Write ordering.** `sase.gate_shell.transaction._create_gate_shell_transaction`:
   - creates the member directory and its `agent_meta.json`
     (`create_gate_shell_member`);
   - moves the workspace claim (`move_gate_shell_claim` / `_record_creator_claim`);
   - runs `create_gate(spec)`, which writes the bundle and appends the PlanApproval
     notification;
   - only then patches `gate_bundle_path` / `gate_notification_id` into the meta
     (`_record_with_gate_result`).

   In the observed run, about a second passed between creating the directory (09:32:38)
   and writing the notification (09:32:39.02).

2. **First load lands inside that window.** The watcher adds an inotify watch on the new
   directory when `IN_CREATE` arrives (`util/fs_watcher.py` `_add_watch_tree`). The
   directory creation, the claim write, and the new notification (via
   `_notification_gate_refresh.request_notification_agents_refresh` and
   `_last_pending_gate_artifact_dirs`) each queue an exact artifact delta. That delta
   re-scans the directory and gets a pending gate row that has neither field yet.
3. **The watch is pruned right after that apply.**
   `_live_watch_coverage.rearm_live_agent_watch_coverage` keeps watches only for rows
   where `agent_row_is_in_flight(agent)` is true. For gates that means
   `gate_state == "settling"` or `gate_execution_active`. A **pending** gate counts as
   terminal here, so `prune_agent_dir_watches` drops its directory watch. The later
   `_record_with_gate_result` meta write then produces no inotify event.
4. **The backstops miss it.**
   - `_pulse_after_gate_visible` touches the project-level `.ace_refresh_pulse`, which
     names no row and only sets `_dirty_agents`. In
     `event_refresh/_auto_refresh_surfaces.py`, any exact delta queued by one of the
     other running agents is consumed first and clears `_dirty_agents`, so the pulse
     never triggers a broad reload.
   - The in-flight marker poll (`_loading_refresh_polling.py`) seeds its baseline on
     first observation. If that first observation came after the final write, the poll
     never sees a change.

   So the stale row survives until some unrelated broad reload, which can take up to
   about five minutes.

5. **The resolver trusts those two fields only.**
   `_agent_enter_index.linked_notification` matches a gate row to its notification by
   `gate_notification_id`, then by `gate_bundle_path`.
   `_agent_enter_resolver._dedupe_notification_targets` drops a notification only when
   its id or `bundle_path` was already claimed by a row. With both fields missing, the
   PlanApproval notification (which does identity-match the family's `--plan` member)
   went through as a second, notification-only target.
6. **Label drift.** That notification-only target reads "Review plan · PLAN".
   `_agent_enter_labels.notification_pending_status` never looks at
   `action_data.plan_tier`, which every PlanApproval notification sets (to `"tale"`), so
   `gate_target_label` never produces "Review tale plan" for it.

## Changes

### 1. Keep inotify coverage on pending gate rows (freshness root cause)

In `src/sase/ace/tui/actions/agents/_live_watch_coverage.py`, stop treating pending
gates as terminal for watch coverage. A pending gate is still live: after the first load
its meta keeps changing (gate result fields, claim holder, answer / settling state).

- Add a small private predicate, e.g. `_row_needs_live_watch(agent)`. It returns
  `agent_row_is_in_flight(agent)`, **or**, for `agent.is_gate` rows,
  `not gate_row_is_settled(agent)`. Import `gate_row_is_settled` from
  `sase.ace.tui.models.agent_family_members`; it already treats a non-terminal
  `gate_state` with no `stop_time` as unsettled.
- Use this predicate in `rearm_live_agent_watch_coverage` in place of the bare
  `agent_row_is_in_flight` call. Pending gates then land in `in_flight_by_dir`, are
  never passed to `prune_agent_dir_watches`, and keep the newest-first
  `MAX_LIVE_AGENT_WATCHES` cap.
- Do **not** change `agent_row_is_in_flight` itself. Family bucket math uses it
  (`family_member_status_buckets`, `current_family_shell_row`), and a pending gate must
  not start counting as running there. Update the module docstring to say that pending
  gate rows are covered too.
- With the watch kept, the `_record_with_gate_result` atomic meta write fires a normal
  `agent_meta.json` event. `event_refresh/_artifact_paths.py` already maps that to an
  exact delta for the gate directory, so the row picks up both fields on the next tick.

### 2. Link gate rows to their notification by gate id (resolver robustness)

A gate shell's `gate_id` is its spec `request_id`. Every adapter-backed gate
notification carries `action_data.request_id`, equal to the bundle directory name. This
was checked against every gate action in the live notification store. Use that as a
third link key so a gate row whose bundle and notification fields have not loaded yet
still claims its notification.

- `src/sase/ace/tui/actions/agents/_agent_enter_models.py`: add
  `by_request_id: dict[str, Notification]` (default empty) to `GateNotificationIndex`.
- `src/sase/ace/tui/actions/agents/_agent_enter_index.py`:
  - In `build_gate_notification_index`, fill `by_request_id` with `setdefault` **only
    for notifications that already go into `gate_notifications`** (adapter-backed
    actions). This matters because non-gate notifications such as `GateExecutionFailed`
    reuse the same `request_id` and must not claim the link.
  - Extend `linked_notification(row, index)`: after the existing id and bundle-path
    lookups, fall back to `index.by_request_id.get(row.gate_id)` when `row.gate_id` is a
    non-empty string.
- `src/sase/ace/tui/actions/agents/_agent_enter_builders.py`, `gate_row_target`: when
  the row lacks `gate_notification_id` / `gate_bundle_path` but a `linked` notification
  exists, fill the target's `notification_id` from `linked.id` and its `bundle_path`
  from `linked.action_data["bundle_path"]`. The row's own values still win when present.
  This lets `_run_agent_enter_gate_target` find and open the notification instead of
  toasting "no longer pending". The label stays row-derived ("Review tale plan" from
  `gate_start_status == "TALE"`), and the detail already switches to
  `notification_gate_detail(linked)`.
- `src/sase/ace/tui/actions/agents/_agent_enter_resolver.py`:
  - In both the `container` and `member` branches, also mark the linked notification's
    `bundle_path` as seen. Recording the linked notification's id alone is not enough.
  - Collect the gate ids of the roster's pending and settled gate rows.
  - Make `_dedupe_notification_targets` skip a notification whose
    `action_data.request_id` is one of those gate ids. Pending rows already own the
    notification, and settled rows mirror the existing `settled_bundle_paths` rule.
  - Keep the existing id and bundle-path checks.
  - Thread the new id sets through as keyword arguments, following the existing `seen_*`
    / `settled_*` set style.

### 3. Label tale PlanApproval notifications as tale reviews

In `src/sase/ace/tui/actions/agents/_agent_enter_labels.py`,
`notification_pending_status`: when none of `pending_status` / `gate_start_status` /
`status` is present, and `notification.action == "PlanApproval"` and
`action_data.plan_tier` is `"tale"` (case-insensitive), return `"TALE"`. A
notification-only tale target then reads "Review tale plan · TALE", the same as the
row-backed one. It still happens when no gate row is loaded, for example when the row is
outside the bounded viewport window. Leave `EpicApproval` and all other actions
unchanged.

## Tests

- `tests/ace/tui/actions/test_live_watch_coverage.py`: add
  `test_apply_keeps_watch_for_pending_gate_row`. Build a gate row with
  `agent_family_role="gate"`, a `gate_id`, `gate_state="pending"`, and no `stop_time`.
  After `_apply`, its directory must be in `ensure_calls` and absent from `prune_calls`.
  Also assert that a settled gate row (`gate_state="answered"` or a `stop_time`) is
  still pruned. Follow the existing `_agent` / `_ApplyHarness` helpers, extending
  `_agent` with the gate fields if needed.
- `tests/ace/tui/test_agent_enter_targets_family.py`, using the helpers in
  `tests/ace/tui/_agent_enter_targets_helpers.py`:
  - `test_stale_pending_gate_row_links_notification_by_request_id`. This is the
    regression for the screenshot.
    - Build a family container whose members are a `--plan`-style member and a pending
      plan gate row: `kind="plan"`, `start_status="TALE"`, `gate_id="gate-tale-1"`,
      `notification_id=None`, `bundle_path=None`.
    - Add one `PlanApproval` notification. Its action data identity-matches the plan
      member and also has `request_id="gate-tale-1"`,
      `bundle_path="/tmp/bundles/gate-tale-1"`, and `plan_tier="tale"`.
    - Resolving the container yields exactly one target, with `source == "gate_row"`,
      label `"Review tale plan"`, `notification_id` equal to the notification's id, and
      `bundle_path` equal to the notification's bundle path.
  - `test_settled_gate_id_drops_notification_only_target`: the same idea as
    `test_settled_bundle_drops_notification_only_target`, but the settled row has no
    `bundle_path` and the drop happens through `request_id == gate_id`.
  - `test_non_gate_notification_request_id_does_not_link`: a `GateExecutionFailed`
    notification that shares the `request_id` does not become the row's linked
    notification.
- Label test (`tests/ace/tui/test_agent_enter_targets_scope.py`, or wherever
  `gate_target_label` / notification labels are already covered): a notification-only
  PlanApproval target with `plan_tier="tale"` gets label `"Review tale plan"` and a
  badge starting with `"TALE"`. A PlanApproval without `plan_tier` still reads
  `"Review plan"` / `"PLAN"`.
- Executor check (`tests/ace/tui/test_agent_enter_targets_executor.py`): a gate-row
  target whose notification id came from the request-id link opens through
  `open_notification_action` rather than toasting. Add this only if the existing
  executor tests don't already cover dispatch by `target.notification_id`.

## Verification

- Run the targeted tests above, then `just check` through `sase tool run check`, as the
  `lint_and_test` memory note describes. Do not run `just check-full`.
- No visual snapshot or keymap changes are expected. `src/sase/default_config.yml` is
  untouched.
- Manual repro, optional: propose a tale plan from any agent. While its gate is pending,
  select the family row in the Agents tab and press `<enter>`. The PlanApproval modal
  should open directly with no chooser, and the footer hint should read "review tale
  plan".
