---
tier: tale
title: Stop misclassifying monitor starter agents as monitor rows
goal:
  An agent that started a monitor is no longer classified as a monitor row, so it
  buckets by its own status and can be dismissed with `x` like any other finished agent.
size: medium
proposed_by: bbugyi200.athena.003
create_time: 2026-08-13 18:12:18
status: wip
---

# Plan: Stop misclassifying monitor _starter_ agents as monitor rows

## Symptom

In the `sase ace` Agents tab, the `zx` agent row cannot be removed:

- it renders `(DONE)` in green, but sits inside the **Running** group of the `@default`
  panel;
- it is prefixed with the `⏱` monitor glyph and painted in the monitor row color;
- pressing `x` (`kill_agent`) pops the warning **"Monitor has already finished"** and
  does nothing — the row can never be dismissed;
- the keybinding footer offers no `x` (and no other agent binding) for the row.

The same defect affects any family member that started a monitor. In the `zx` family,
`zx--0` and `zx--1` are both stuck; `zx--2`, which never started a monitor, is clean.

**User workaround until this lands:** press `m` (`toggle_mark`) on the row, then `x`.
`action_kill_agent` checks `self._marked_agents` _before_ the monitor branch, so the
bulk-cleanup path dismisses the row normally.

## Root cause

`monitor_id` in `agent_meta.json` is ambiguous — it is written both to the monitor
**member** and to the agent that **started** the monitor — but several readers treat a
non-empty `monitor_id` as "this row _is_ a monitor member".

### Writer

`src/sase/axe/run_agent_exec_monitor.py:81-88` stamps `monitor_id` and
`monitor_member_agent_name` onto the **starter's own** `agent_meta.json` as a
back-reference when it adopts the monitor handoff marker. The starter is an ordinary
agent (`agent_family_role: "root"`) and never gets a `monitor_state`.

### Canonical predicate

The correct "is a monitor member" test is `agent_family_role == "monitor"`. Two call
sites already get this right and document it:

- `src/sase/monitor/store.py:303-309` (`_monitor_records`), whose module docstring
  states the member predicate outright;
- `src/sase/agent/running_listing.py:340-348` (`_is_visible_monitor_record`).

The suffix carries the same signal independently:
`agent_family_role_for_suffix("--mon")` and `"--mon-0"` both resolve to `"monitor"`
(`src/sase/plan_chain.py:50-56`, `:201-207`).

### Evidence

`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/13/20260813151156/agent_meta.json`
(the stuck `zx` row):

```json
"name": "zx--0", "agent_family": "zx", "agent_family_role": "root",
"role_suffix": "--0",
"monitor_id": "w5qw0zs5fxq1", "monitor_member_agent_name": "zx--mon",
"stopped_at": "2026-08-13T19:45:13.407082+00:00"
```

There is no `monitor_state`, and the sibling `done.json` has `"outcome": "completed"`.
The real member (`.../20260813154230/`) is the row carrying
`"agent_family_role": "monitor"`, `"monitor_state": "failed"`.

Replaying that exact file through the loader reproduces the defect:

```
agent_family_role: root
status           : DONE
monitor_id       : w5qw0zs5fxq1
monitor_state    : None
is_monitor       : True      <-- wrong
status_bucket    : Running   <-- wrong
```

### Failure chain

1. `src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py:51` —
   `apply_monitor_meta` guards only on `monitor_id`, so it applies monitor row semantics
   to the starter. Line 82 then sets `agent.status_bucket = monitor_state_bucket(None)`,
   and `monitor_state_bucket` deliberately defaults an unknown state to `"Running"`
   (`src/sase/monitor_state.py:17-24`). **A DONE row is bucketed Running.**
2. `src/sase/ace/tui/models/agent.py:250-252` — `Agent.is_monitor` returns
   `bool(self.monitor_id) or self.agent_family_role == "monitor"`, so the starter reads
   as a monitor. That drives:
   - `src/sase/ace/tui/actions/agents/_kill_action_flow.py:66` — `x` is routed to
     `_handle_monitor_stop_action`, which sees `monitor_state != "running"` and
     dead-ends at `src/sase/ace/tui/actions/agents/_monitor_stop_flow.py:22-26`. **The
     row becomes permanently undismissable.**
   - `src/sase/ace/tui/widgets/_agent_list_render_agent.py:195,218,232,260` — `⏱` glyph,
     monitor row color, and `monitor_label` name substitution.
   - `src/sase/ace/tui/widgets/_keybinding_bindings.py:276-286` — early return, so the
     footer advertises nothing for the row.
   - `src/sase/ace/tui/models/agent_family_members.py:29-31,80-81,173-177` — the row is
     never "in flight" and is dropped from the family's concrete rows, corrupting
     panel/tribe agent counts.
   - `src/sase/ace/tui/models/agent_tribe_summary.py:204` — member kind reads as
     `"monitor"`.
3. `src/sase/agent/running_listing.py:358-369` (`_record_status_bucket`) —
   `if monitor_id or monitor_state: return monitor_state_bucket(monitor_state)` buckets
   any starter as `Running`. Line 386 also lets a starter with a `parent_timestamp`
   through a filter meant only for monitor members.
4. `src/sase/integrations/_agent_list_entry_builder.py:294-298` (`_is_monitor`) — same
   `monitor_id` conflation, plus a `done.outcome == "monitored"` clause that is _also_
   ambiguous: `src/sase/axe/run_agent_exec_finalize.py:142,183` writes
   `outcome: "monitored"` into the **starter's** `done.json` whenever the runner was
   SIGTERM'd into a monitor handoff. Consumed at lines 57, 114 and 281.
5. Direct `bool(monitor_id)` conflations in
   `src/sase/integrations/_agent_list_entry_models.py:144-146`,
   `src/sase/integrations/_mobile_agent_summary.py:172`, and
   `src/sase/integrations/_editor_helper_agents.py:159`.

## Scope decisions

- **Keep `monitor_id` on the starter.** It is a legitimate back-reference (the detail
  panel prints `sase monitor show <id>`). Only the _classification_ changes.
- **Leave the `MONITORED` status label alone.** A starter that genuinely handed off via
  SIGTERM keeps its `MONITORED` label (`src/sase/agent/running_listing.py:398-402`);
  that is intentional and is not what breaks dismissal. Only the _bucket_ and the
  _is-a-monitor_ predicate are corrected.
- **No `../sase-core` change.** `sase_core_rs` supplies the raw marker scan
  (`src/sase/core/agent_scan_facade.py`); it does not classify monitors or compute
  status buckets. `src/sase/monitor_state.py` is already the declared shared home for
  "monitor state semantics shared by monitor storage and projections", so the predicate
  belongs beside `monitor_state_bucket`. Routing a boolean over already-parsed meta
  fields through the Rust wire would add a cross-repo hop for no behavioral gain.

## Steps

### 1. Add one shared monitor-member predicate

In `src/sase/monitor_state.py`, next to `monitor_state_bucket`:

```python
MONITOR_FAMILY_ROLE = "monitor"


def is_monitor_member_role(
    agent_family_role: str | None,
    role_suffix: str | None = None,
) -> bool:
    """Return whether a row is a monitor member rather than a monitor starter.

    ``monitor_id`` is written to both the monitor member and the agent that
    started it, so it cannot classify a row on its own.
    """
    from sase.plan_chain import agent_family_role_for_suffix

    return (
        agent_family_role_for_suffix(
            role_suffix, agent_family_role=agent_family_role
        )
        == MONITOR_FAMILY_ROLE
    )
```

The suffix fallback covers `--mon` and `--mon-N` when metadata omits the role.
`sase.plan_chain` imports only `re`/`dataclasses`/`collections.abc`, so a module-level
import is also safe if the implementer prefers it over the local import.

Export both names from `__all__`.

### 2. Make `Agent.is_monitor` role-based

`src/sase/ace/tui/models/agent.py:249-252` → delegate to
`is_monitor_member_role(self.agent_family_role, self.role_suffix)`. Update the docstring
to say a starter's `monitor_id` back-reference does not make it a monitor row.

This alone fixes the `x` dead-end, the `⏱` glyph, the row color, the footer, and the
family-member row/count projections.

### 3. Stop stamping monitor row semantics onto starters

`src/sase/ace/tui/models/_loaders/_meta_enrichment_common.py`, `apply_monitor_meta`: add
a keyword-only `monitor_member: bool` parameter and gate **only** the presentation
overrides — line 82's `agent.status_bucket` assignment and the lines 83-88 `MONITORING`
status override — behind it. Every other field assignment stays unconditional so the
back-reference survives.

Update both call sites to pass the role-derived value:

- `src/sase/ace/tui/models/_loaders/_meta_enrichment_filesystem.py:380-396` —
  `is_monitor_member_role(data.get("agent_family_role"), data.get("role_suffix"))`
- `src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py:254-270` —
  `is_monitor_member_role(meta.agent_family_role, meta.role_suffix)`

In both loaders `agent_family_role` is already populated before this call
(`_meta_enrichment_filesystem.py:148`, `_meta_enrichment_wire.py:107`), so reading it
off `agent` instead is equally valid — pick one and be consistent.

### 4. Fix the CLI/listing bucket

`src/sase/agent/running_listing.py`:

- `_record_status_bucket` (358-369) — replace the `monitor_id or monitor_state` test
  with the member predicate over `meta`. A starter must fall through to
  `valid_status_bucket(raw_bucket)`.
- `_done_info_from_record` (386) — the `parent_timestamp and not meta.monitor_id` escape
  hatch exists to keep monitor members visible; change it to the member predicate so a
  starter with a `parent_timestamp` (like `zx--1`) is filtered like any other family
  member.

### 5. Fix the integrations predicates

- `src/sase/integrations/_agent_list_entry_builder.py:294-298` — `_is_monitor` must
  require the member role. Both clauses are ambiguous: gate the
  `done.outcome == "monitored"` clause on the role too.
- `src/sase/integrations/_agent_list_entry_models.py:144-146` — `is_monitor` should read
  the already-projected `agent_family_role`/`role_suffix` fields (both are on the entry)
  rather than `monitor_id`.
- `src/sase/integrations/_mobile_agent_summary.py:172` — use the entry's `is_monitor`
  property instead of `bool(agent.monitor_id)`.
- `src/sase/integrations/_editor_helper_agents.py:159` — use the member predicate over
  `meta`.

`src/sase/agents/cli_list.py:101` needs no edit; it reads the corrected property.

### 6. Harden the kill flow so no row can ever be undismissable

`src/sase/ace/tui/actions/agents/_kill_action_flow.py:66-68` currently returns
unconditionally for monitor rows. Change it so the monitor-stop branch is taken **only
while the monitor is actually running**:

```python
if agent.is_monitor and agent.monitor_state == "running":
    self._handle_monitor_stop_action(agent)
    return
```

A settled monitor member then falls through to the ordinary
`_plan_focused_agent_cleanup` path and can be dismissed like any other finished row.
This is defense in depth: it turns "the row can never be removed" into an impossible
outcome rather than a bug that was fixed once.

Consequences to carry through:

- `src/sase/ace/tui/actions/agents/_monitor_stop_flow.py:22-26` — the
  `monitor_state != "running"` guard becomes unreachable from `x`. Keep it as a
  defensive guard, but it is no longer the user-visible behavior.
- `src/sase/ace/tui/widgets/_keybinding_bindings.py:276-286` — the early `return` for
  non-running monitors must be dropped so a settled monitor row advertises the normal
  bindings (including `x`). Keep the `stop monitor` label for the running case.
- `tests/test_agent_monitor_stop_action.py:72-81`
  (`test_action_kill_agent_on_terminal_monitor_notifies_and_skips_confirm`) asserts
  today's dead-end and must be rewritten to assert the new fall-through to the cleanup
  path.
- `tests/test_keybinding_footer_agent.py:433-440`
  (`test_keybinding_footer_terminal_monitor_omits_stop_monitor`) must keep asserting no
  `stop monitor`, but should now also assert the ordinary bindings are present.

## Tests

Add regressions that would each have failed before the fix.

1. `tests/ace/tui/models/test_monitor_rows.py` — a starter row built from meta with
   `monitor_id` set, `agent_family_role="root"`, `role_suffix="--0"`, no
   `monitor_state`, status `DONE`:
   - `agent.is_monitor is False`
   - `agent.status_bucket` is not `"Running"` (bucket resolves to `Done`)
   - `agent.monitor_id` is still populated (back-reference preserved) Mirror it for the
     wire loader and the filesystem loader.
2. `tests/test_agent_monitor_stop_action.py` — pressing `x` on a DONE starter row that
   carries `monitor_id` does **not** notify `"Monitor has already finished"` and does
   reach the ordinary cleanup path; plus the rewritten settled-monitor-member case from
   step 6.
3. `tests/ace/tui/widgets/test_agent_list_monitor_rows.py` — a starter row renders no
   `⏱` glyph and no monitor row color.
4. `tests/test_keybinding_footer_agent.py` — a DONE starter row advertises the normal
   agent bindings; a settled monitor member does too (without `stop monitor`).
5. `tests/ace/tui/models/test_agent_family_members.py` — a family whose root started a
   monitor still projects that root through `concrete_agent_statuses` /
   `concrete_family_member_rows`, so panel and tribe counts stay correct.
6. `tests/test_agent_list_entries.py` / `tests/test_agent_list_entry_projection.py` — a
   starter record buckets by its own status, not by `monitor_state_bucket`, for both the
   `monitor_id` and the `done.outcome == "monitored"` shapes.
7. `tests/test_mobile_agent_listing.py:120-140` — this fixture sets only `monitor_id` on
   an agent named `alpha--mon`; add `agent_family_role` (`"monitor"`) so it still
   asserts `is_monitor is True`, and add a starter case asserting `False`.
8. New unit coverage for `is_monitor_member_role` in the appropriate `tests/monitor/`
   module: `"monitor"` role → `True`; `"root"` role with a `monitor_id`-bearing meta →
   `False`; role omitted but `role_suffix` of `--mon` / `--mon-0` → `True`.

## Verification

```bash
just install
just check
```

Then, because this touches the shared agent-list projection consumed by the TUI, the
CLI, the mobile summary, and the editor helper, run the full gate through a monitor
(never inline):

```bash
sase monitor start --command 'just check-full' \
  --reason 'Full gate for the monitor-starter misclassification fix' \
  --next '<follow-up action>'
```

Finally confirm in the real app that the previously stuck row is gone: open `sase ace`,
select a DONE agent that started a monitor, and press `x` — it must show the ordinary
kill/dismiss confirmation, and after confirming the row must disappear. Also confirm
such a row no longer appears under **Running** and no longer renders the `⏱` glyph.

## Follow-up candidates (do not implement here)

- `src/sase/monitor/models.py:104-110` — `MonitorRecord.from_record` validates with
  `not meta.monitor_id`. It is only ever fed by `_monitor_records`, which already
  filters on the role, so this is latent rather than live; tightening it to the member
  predicate would be a small hardening.
- The `MONITORED` status label applied to a handed-off starter
  (`src/sase/agent/running_listing.py:398-402`,
  `src/sase/integrations/_editor_helper_agents.py:186`) is arguably confusing next to
  the member's own `MONITORED` row, but it is pre-existing intentional behavior and
  orthogonal to dismissal.
