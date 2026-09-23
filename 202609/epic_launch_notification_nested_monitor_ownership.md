---
tier: tale
title: Epic-launch notifications clear when the launching family is read
goal:
  Reading an Agents-tab family row dismisses the epic-launch notification that names its
  gate-launched monitor shell, because node ownership follows the full parent_timestamp
  chain.
size: small
proposed_by: bbugyi200.athena.0q4
create_time: 2026-09-23 11:36:14
status: wip
---

# Plan: Epic-launch notifications clear when the launching family is read

## 1. Symptom

An `[epic-launch] Epic sase-XXX launched from <plan>.md` notification is supposed to be
dismissed automatically when the agent that launched the epic is marked read in the
Agents tab. Epic sase-14l shipped that behavior on 2026-09-20, but in practice most
epic-launch notifications are never auto-dismissed. They pile up in the notification
inbox (`read: true`, `dismissed: false`) until the user dismisses them by hand.

## 2. Root cause (verified against live data)

### 2.1 Which row the notification names

`finish_epic_launch` (`src/sase/bead/epic_launch.py`) sends the notification through
`defer_epic_completion_until_monitor_settlement`
(`src/sase/bead/epic_launch_handoff.py`). That call deliberately retargets
`action_data.raw_suffix` to the **epic-launch monitor shell row** (`<family>--mon`,
status `EPIC CREATED`). It does not name the planner.

In the normal ACE approval flow, that monitor is started by the **EpicApproval gate
shell**. It is not started by the family's root agent. The durable chain is:

```
<fam>--plan  (family container = Agents-tab agent node, raw_suffix R)
  └─ <fam>--gate  (gate shell, parent_timestamp = R, raw_suffix G)
       └─ <fam>--mon  (monitor shell, parent_timestamp = G, raw_suffix M)  ← epic-launch names M
```

Live example from today (family `0q3`, epic `sase-16z`): the notification carries
`raw_suffix=20260923110604`, which is `0q3--mon`. The monitor's parent is `0q3--gate`
(`20260923110242`), and the gate's parent is the node `0q3--plan` (`20260923104834`).

### 2.2 Why nothing owns that row

Every read-ack and dismissal path asks one index which notification keys an agent node
owns: `agent_node_projection_index` in `src/sase/ace/tui/models/agent_nodes.py`. Those
paths are `_clear_agent_unread_and_dismiss_notification`,
`_mark_current_unread_done_agents_read`,
`_dismiss_agent_completion_notifications_for_dismissed_agents` (which marked-group save
also calls), and the unread projection
`_reconcile_unread_from_completion_notifications`.

That index attaches a non-node row (a gate, monitor, or family-member shell) to a node
**only when the row's `parent_timestamp` is exactly a node's `raw_suffix`**. It follows
one hop only:

```python
parent = node_by_raw_suffix.get(agent.parent_timestamp or "")
if parent is None:
    continue
```

`concrete_family_member_rows` deliberately leaves out gate and monitor shells, so
nothing else covers them either. The gate-launched monitor is two hops from the node, so
it has no owner. That causes two failures:

1. **The row never turns unread for it.** `projection_has_active_completion` never sees
   the monitor's key, so the active epic-launch notification never flags the family row
   unread. The gate flow sends no `user-agent` completion for the planner, so the family
   row usually has no unread marker. Selecting it then does nothing, because
   `_clear_agent_unread_and_dismiss_notification` returns early unless the row is in
   `_unread_completed_agent_ids`. That is the "agent doesn't get marked as read"
   symptom.
2. **Reading the row wouldn't help anyway.** Even when the row is unread for another
   reason, or manually toggled with `U`, `_notification_keys_for_agents` returns the
   node's keys without `(cl_name, M)`. So the Rust
   `DismissAgentCompletionsMatchingAgents` update never matches the epic-launch row.

### 2.3 Why it is "unreliable" rather than always broken

It works whenever the monitor's starter is the node itself. For example, a land agent
runs `sase bead work` in its own turn, so `sase-16n.land--mon` has parent =
`sase-16n.land--plan`. I checked every `epic-launch` notification since the fix landed
(2026-09-21):

| monitor parent        | auto-dismissed | still active |
| --------------------- | -------------- | ------------ |
| family root (1 hop)   | 3              | 0            |
| EpicApproval gate (2) | 6 (by hand)    | 7            |

The read-ack path cannot reach a two-hop row, so the 6 dismissed gate-launched rows must
have been cleared through the Notifications modal or another path, not by reading the
agent. The prior epic's tests passed because they only ever put the settlement row on
the node's own `(cl_name, raw_suffix)`. They never put it on a nested shell, which is
the only shape production actually emits. The prior plan's assumption was: "family reach
is already handled on the host side; `_notification_keys_for_agents` expands an agent
node to its owned rows' keys". That holds for one hop only.

Replaying the live Agents roster with a transitive owner walk attaches exactly the five
currently-unowned two-hop monitors (`0q3--mon`, `0q0--mon`, `0pq--mon` from gates;
`0ps--mon`, `0pn--mon` from tale coders). Each goes to the same node the tree projection
already renders as its `family_container`. It changes no existing ownership: all 313
already-owned rows keep their current owner.

## 3. The fix

Scope: host-side only, in `src/sase/ace/tui/models/agent_nodes.py`. No Rust or wire
change. The Rust store rule stays exact `(cl_name, raw_suffix)` matching. The "family
reach is host-side" decision from sase-14l stays. We only make that host-side reach
actually follow the durable causal chain.

### 3.1 Resolve the owning node transitively

In `agent_node_projection_index`, replace the single-hop parent lookup with a
causal-chain walk:

1. After the node pass, build `parent_suffix_by_suffix: dict[str, str]` from every
   **non-node** roster row that has both a `raw_suffix` and a `parent_timestamp`,
   **skipping any row whose `raw_suffix` is a node suffix**. Workflow step children
   share their node's `raw_suffix` and must never shadow the node. Use `setdefault` so
   the first-seen row wins, which keeps the result deterministic.
2. Add a small private helper, e.g.
   `_owning_node_for_row(row, node_by_raw_suffix, parent_suffix_by_suffix)`. It starts
   at `row.parent_timestamp` and, at each step, returns the node if the current suffix
   is a node suffix. Otherwise it moves to `parent_suffix_by_suffix.get(current)`. It
   stops (returning `None`) on a missing link or a suffix it has already visited. Keep a
   `visited` set so a malformed or cyclic chain cannot loop, including a row whose
   `parent_timestamp` equals its own `raw_suffix`.
3. Use that helper in the second roster loop in place of
   `node_by_raw_suffix.get(agent.parent_timestamp or "")`. Leave the rest alone:
   skipping node rows, the "append if not already owned" dedupe by identity, and
   `_agent_node_completion_rows` (which still drops workflow step children from keys).

Update the `agent_node_projection_index` docstring, and the
`_agent_node_completion_rows` docstring if it helps, to say that non-node rows are owned
by the nearest agent node on their `parent_timestamp` chain. Examples: a gate shell's
launch monitor, or a family member's monitor.

Do **not**:

- match on `family_root_suffix` / `agent_root_timestamp`. That would let one read clear
  settlement rows owned by sibling families, which the sase-14l plan ruled out.
- rely on the presentation-only `family_container` / tree links. `parent_timestamp` is
  the durable identity the existing code already uses.
- change `concrete_family_member_rows`, `is_agents_tab_agent_node`, the Rust matcher,
  `SETTLEMENT_NOTIFICATION_SENDERS`, or the epic-launch producer
  (`settlement_notification_action_data` / `monitor_settlement_payload`). The producer
  correctly names the monitor row.

All callers of the index get the fix with no new plumbing. That covers row-selection
read-ack, the unread-jump shortcut, the `U` toggle back to read, the bulk read toggle,
single-row dismissal, and marked-group save (which calls
`_dismiss_agent_completion_notifications_for_dismissed_agents` on the UI thread), plus
the unread projection and identity normalization in `_loading_finalize.py`. Confirm this
by reading the call sites. Do not add call-site changes.

### 3.2 Expected behavior after the fix

- When an epic-launch (or `monitor-settlement`) notification names a nested shell row,
  it flags the owning family row unread, because it is now in that node's
  `completion_keys`.
- Selecting, jumping to, or bulk-reading that family row dismisses the notification in
  the store and drops it from the cached snapshot, through the existing
  `_remove_agent_completion_notifications_from_cache` +
  `agent_row_notification_matches_agent` path.
- Existing orphaned epic-launch notifications whose family rows are still loaded will
  show those rows as unread once, and clear on the next read. This is intended; no
  migration is needed. Rows for families no longer loaded are unaffected.

## 4. Tests

### 4.1 `tests/ace/tui/models/test_agent_nodes.py`

Add a fixture that builds the production shape with the module's `_agent` helper:

- a family container node `build--plan` (`agent_family="build"`,
  `agent_family_role="root"`, `plan_chain_root=True`);
- a gate shell `build--gate` with `parent_timestamp=<node suffix>`,
  `agent_family_role="gate"`, and a `gate_id`. `Agent.is_gate` is
  `is_real_gate_member(agent_family_role, gate_id)`.
- a monitor shell `build--mon` with `parent_timestamp=<gate suffix>`,
  `agent_family_role="monitor"`, `role_suffix="--mon"`, `monitor_id=...`.
  `Agent.is_monitor` is `is_monitor_member_role(agent_family_role, role_suffix)`.

Assert:

- `index.owner_for_identity(monitor.identity)` is the node's projection.
- `(monitor.cl_name, monitor.raw_suffix)` is in that projection's `completion_keys`, and
  `projection_has_active_completion(projection, {monitor_key})` is true.
- A monitor started by a family-member shell (`build--code` → `build--mon`) is owned the
  same way.
- **Cycle guard:** two non-node rows whose `parent_timestamp`s point at each other, and
  a row whose `parent_timestamp` equals its own `raw_suffix`, both stay unowned and the
  call returns (no hang).
- **Dangling chain:** a shell whose chain ends at a suffix with no loaded row stays
  unowned.
- **No shadowing:** a workflow step child that shares the node's `raw_suffix` (the
  existing `_plan_family_root_with_main_step_and_continuation` shape) still leaves the
  node's own key present and the `("main", suffix)` key absent.
- **No cross-family reach:** a second family's monitor with the same `cl_name` but its
  own gate/root chain is owned by its own node, not the first.

### 4.2 `tests/ace/tui/test_agent_unread_projection.py`

Next to `test_reconcile_marks_unread_for_active_settlement_row`, add a reconcile test.
Load the roster with a terminal family node (`EPIC CREATED`), its gate shell, and the
gate's monitor. An active `epic-launch` notification whose `raw_suffix` is the
**monitor's** marks the **node** unread. Dismissing that notification clears the node's
unread marker on the next reconcile.

### 4.3 `tests/ace/tui/test_agent_unread_selection.py`

Next to `test_agent_row_selection_dismisses_matching_settlement_row_from_cache`, add a
selection test with the same three-row family in `_agents` (the roster
`_notification_keys_for_agents` reads). Mark the node unread and put an `epic-launch`
row keyed to the monitor's `(cl_name, raw_suffix)` in the cached snapshot. Selecting the
node must:

- pass a key list containing the monitor's key to the mocked
  `dismiss_agent_completion_notifications_matching_agents`;
- remove that settlement row from `_notification_snapshot_cache`, while keeping an
  unrelated `epic-launch` row with a different `raw_suffix`.

This is the regression test the sase-14l suite was missing. It must fail on the current
tree before the fix and pass after it. Confirm both.

## 5. Documentation

In `docs/notifications.md`, in the Agents-tab unread paragraph ("Unread state on the
Agents tab is projected from…") and the acknowledgment paragraph right before it, say
that a row "contains" every shell on its `parent_timestamp` chain, not only direct
children. Give the concrete case: an epic launch approved through the EpicApproval gate
names the gate's launch monitor, and reading the family row clears it. Keep the existing
exact-match / no-`cl_name`-fallback wording unchanged.

## 6. Verification

- `just install` if the workspace venv is stale, then `just fix`.
- Targeted:
  `pytest tests/ace/tui/models/test_agent_nodes.py tests/ace/tui/test_agent_unread_projection.py tests/ace/tui/test_agent_unread_selection.py tests/ace/tui/test_agent_unread_toggle.py tests/ace/tui/test_agent_settlement_notification_match.py`.
- `sase tool run check` (the repo's agent-default verification; do not run
  `check-full`).
- Read-only live sanity check, with no store writes: load
  `sase.ace.tui.models.agent_loader.load_all_agents()`, project it with
  `project_mixed_agent_tree(agents, [])`, drop clan containers, build
  `agent_node_projection_index`, and confirm every loaded `is_monitor` row whose
  `raw_suffix` is named by an active `epic-launch` notification
  (`sase notify list -j --all --sender epic-launch`) now has a non-`None`
  `owner_for_identity`, and that the owner is the row's `family_container`.
