---
tier: tale
title: Restore Agents-tab unread markers for plan-family agent nodes
goal:
  An Agents-tab agent node owns the completion notification written under its own
  (cl_name, raw_suffix), so a completed plan-family agent renders its unread marker
  again and dismissing it clears the matching notification.
size: small
proposed_by: bbugyi200.athena.04z
create_time: 2026-08-17 11:59:11
status: wip
---

# Plan: Restore Agents-tab unread markers for plan-family agent nodes

## Symptom

Agent completion notifications arrive (toast + notification indicator), but the
corresponding row on the Agents tab is never marked unread. The `●` unread marker, the
leader-mode `j` "next unread done" jump, the `u` bulk mark-all-read toggle, and the
unread counts in panel/tribe summaries all behave as if nothing completed.

The user's suspicion that this regressed recently is correct.

## Confirmation (measured against real on-disk state)

Loading the Agents-tab roster the same way the TUI does and projecting the live
notification store onto it produces:

```
active completion keys: 12
loaded agents (non-clan): 401
nodes: 69
unread nodes: 0
```

Three of those 12 active completion notifications target agents that _are_ loaded and
_are_ in a terminal status (`TALE DONE`, `TALE DONE`, `EPIC CREATED`), yet the current
projection marks **zero** rows unread. Re-running the pre-regression matching rule over
the exact same roster and notification set yields the expected 3 unread rows.

Counting how many loaded nodes cannot possibly match their own completion notification:

```
total nodes: 69
nodes missing their own completion key: 24
  ...of which terminal (unread-eligible): 23
```

All 24 are plan-family container rows (`is_family_container_row=True`,
`is_plan_family_root_entry=True`) — the dominant shape for `sase run` agents, which
launch as a plan family (`--plan` root plus continuations).

## Root cause

`src/sase/ace/tui/models/agent_nodes.py` decides which completion-notification keys an
Agents-tab node owns:

```python
def _agent_node_owned_rows(agent: Agent) -> tuple[Agent, ...]:
    if not is_agents_tab_agent_node(agent):
        return ()
    if is_sequential_family_container(agent):
        rows = concrete_family_member_rows(agent)
        return rows or (agent,)
    return (agent,)
```

`concrete_family_member_rows()` is the **status-counting** projection, not a
notification-ownership projection. For a plan-family root it deliberately _substitutes_
the concrete `main` workflow step for the aggregate root row (see its docstring: "Plan
workflow roots are aggregate rows. When their concrete main agent step is loaded, that
step owns the planner phase"). Workflow step rows carry the **step name** as `cl_name`,
so the node's own `(cl_name, raw_suffix)` key silently disappears from
`completion_keys`.

Observed for a real node:

```
NODE cl_name='gh_sase-org__sase' raw_suffix='20260817102622'
  status='TALE DONE'  is_family_container_row=True  is_plan_family_root_entry=True
  own key = ('gh_sase-org__sase', '20260817102622')
  completion_keys = (('main',              '20260817102622'),
                     ('gh_sase-org__sase', '20260817104352'),
                     ('setup',             '20260817102622'),
                     ('prepare',           '20260817102622'),
                     ('checkout',          '20260817102622'),
                     ('diff',              '20260817102622'))
  own key present = False
```

`src/sase/axe/run_agent_runner_finalize.py` writes every completion notification with
`cl_name = <patch cl_name>` and `raw_suffix = <artifacts timestamp>` — exactly the key
that was dropped. So `projection_has_active_completion()` in
`_notification_unread_projection.py` can never match, and
`_reconcile_unread_from_completion_notifications()` skips the row.

Introduced by `49be5b0d3` ("fix(tui): count agent family containers as one node",
2026-08-16). Before that commit the reconcile loop walked every roster row and matched
`(agent.cl_name, agent.raw_suffix)` directly, so the root row matched its own
notification.

### Second-order consequence (same defect, different call site)

`AgentUnreadStateMixin._notification_keys_for_agents()` in
`src/sase/ace/tui/actions/agents/_unread_state.py` reuses the same
`projection.completion_keys`. It feeds
`dismiss_agent_completion_notifications_matching_agents()`, so dismissing (`x`) or
acknowledging a plan-family container also fails to dismiss that agent's own completion
notification — the one-to-one row/notification contract is broken in both directions,
and stale active completion notifications accumulate in the inbox.

### Latent false-positive vector

`projection_has_active_completion()` also matches on `(cl_name, None)` for notifications
written without a `raw_suffix`. Because the buggy key set injects step-name keys
(`main`, `setup`, `prepare`, `checkout`, `diff`, `fetch`), a suffix-less completion
notification for a patch literally named `main` (a very plausible bare-git branch name)
would mark unrelated nodes unread. In the real roster every workflow-step-child owned
row shares the node's own `raw_suffix`, so these step keys carry no information the
node's own key does not already carry.

## Changes

### 1. A node always owns its own completion key

In `src/sase/ace/tui/models/agent_nodes.py`, separate notification-key ownership from
the status-count projection. Introduce a dedicated helper, e.g.:

```python
def _agent_node_completion_rows(agent: Agent) -> tuple[Agent, ...]:
    """Return rows whose completion notifications belong to *agent*.

    A node always owns the notification written under its own
    ``(cl_name, raw_suffix)``: the runner keys completions by the artifacts
    directory the node row was loaded from. Rows the node subsumes for status
    counting are additive, never a substitute -- ``concrete_family_member_rows``
    deliberately swaps a plan-family root for its concrete ``main`` workflow
    step, and step rows carry the step name as ``cl_name``.

    Workflow step children are excluded: they always share their node's
    ``raw_suffix`` and never own a distinct completion notification, so their
    step-name keys only widen the ``(cl_name, None)`` match surface.
    """
    owned = tuple(
        row for row in _agent_node_owned_rows(agent) if not row.is_workflow_step_child
    )
    return (agent, *owned)
```

Use it in both key producers so the index and the standalone helper stay consistent:

- `agent_node_completion_keys(agent)` — currently
  `_completion_keys_for_rows(_agent_node_owned_rows(agent) or (agent,))`.
- `agent_node_projection_index()` — currently
  `completion_keys=_completion_keys_for_rows(owned_tuple)`.

`_completion_keys_for_rows()` already dedupes, so a plain standalone node still yields
exactly one key.

Leave `_AgentNodeProjection.owned_rows`, `owner_for_identity()`, and
`normalize_agent_node_identities()` untouched — the ownership/normalization semantics
they encode are correct and are what make the container (not the member shell) the row
that renders the marker. Only the key set changes.

### 2. Do not change the status-count projection

`concrete_family_member_rows()` and `is_sequential_family_container()` stay as they are.
They are correct for counting agents; the bug is that notification ownership borrowed
them.

## Tests

Add to `tests/ace/tui/models/test_agent_nodes.py`:

1. A plan-family root (`plan_chain_root=True`, `agent_family_role="root"`) with a loaded
   `main` workflow-step child (`step_type="agent"`, same `raw_suffix`) and a real
   continuation member. Assert `agent_node_completion_keys(root)` contains the root's
   own `(cl_name, raw_suffix)` and the continuation's key, and does **not** contain
   `("main", raw_suffix)`.
2. The same shape through `agent_node_projection_index()`: assert
   `projection.completion_keys` contains the root's own key, and that
   `projection_has_active_completion(projection, {root_key})` is `True`.
3. A plain standalone node still yields exactly one completion key (no duplicate).
4. A sequential (non-plan) family container still owns each member key _and_ its own
   key.

Add to `tests/ace/tui/test_agent_unread_projection.py`:

5. End-to-end reconcile regression test: build a plan-family container node with a
   `main` step child, feed `_reconcile_unread_from_completion_notifications()` a
   `JumpToAgent` notification keyed to the container's own `(cl_name, raw_suffix)`, and
   assert the container identity lands in `_unread_completed_agent_ids`. This is the
   exact case that is broken today and that the existing 63 passing tests do not cover.

Add to `tests/ace/tui/test_agent_unread_toggle.py` (or the closest dismissal-side test
module):

6. `_notification_key_dicts_for_agents([container])` includes the container's own
   `{"cl_name": ..., "raw_suffix": ...}` dict, so dismiss/mark-read actually dismisses
   the plan-family root's completion notification.

## Verification

- `just install` first (ephemeral workspace).
- `.venv/bin/python -m pytest tests/ace/tui/test_agent_unread_projection.py tests/ace/tui/test_agent_unread_finalizer.py tests/ace/tui/test_agent_unread_toggle.py tests/ace/tui/test_agent_panel_entry_unread.py tests/ace/tui/test_agent_unread_done_navigation.py tests/ace/tui/models/test_agent_nodes.py -q`
  — 63 of these pass today and must still pass, plus the new cases.
- `just check` before replying. Run `just check-full` through `/sase_monitor` if the
  scoped run escalates or reports an unusual selection — this touches a model shared by
  agent-list rendering, tribe summaries, and visual snapshots.
- Manual confirmation: with a completed plan-family agent whose completion notification
  is still active, `sase ace` should render the `●` unread marker on the family
  container row, leader-mode `j` should jump to it, and `x`/`u` should clear both the
  marker and the notification.

## Non-goals

- The 8 active completion notifications in the store whose agents are no longer in the
  loaded inbox (some dating to July) are not addressed here; they are outside the loaded
  roster, so no row can carry them. Worth a separate look at whether dismissal should
  sweep them.
- `apply_status_overrides()` mirrors a family root's status from a still-running
  descendant monitor (`_descendant_monitors`, added `eefc44983`). A node whose status is
  therefore non-terminal is skipped by `is_unread_completed_status()` and will not show
  unread while its monitor runs. That interaction is intentional-looking and is not
  reproducing on the current roster; it is called out here as a known adjacent behavior,
  not fixed.
