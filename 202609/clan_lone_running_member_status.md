---
tier: tale
title: Clan row mirrors its lone running member's status
goal:
  When exactly one direct clan member is running, the clan row and CLAN detail header
  show that member's own status label and styling (e.g. TESTING) instead of a generic
  RUNNING, and runner-slot refreshes keep it.
size: small
proposed_by: bbugyi200.athena.1m
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.1m](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.1m.md)
- **COMMITS:**
  - [21cdb65](https://github.com/sase-org/sase/commit/21cdb658b05246ca5b19efa1a81f73d19bc7a87d)
    — feat(ace): mirror a lone running clan member's status on the clan row

# Clan row mirrors its lone running member's status

## Problem

When exactly one direct member of an agent clan is running, the clan row should show
that member's status, not a generic `RUNNING`. Example: in clan `sase-100` the members
are `.1` DONE, `.2` DONE, `.3` DONE, `.land` WAITING, and the sequential family `.4`,
whose tree row reads `(TESTING)` while its `just check` monitor runs. Today the clan row
and the clan detail header read `RUNNING`. Both should read `TESTING`, styled the same
way as the family row.

## Root cause

- `_container_for_clan` in `src/sase/ace/tui/models/_agent_tree.py` sets the container
  status with `aggregate_clan_status(row.status for row in runtime_members)`. That calls
  `aggregate_agent_group_status` in `src/sase/agent/status_buckets.py`, which returns
  the literal `"RUNNING"` whenever the Running/Starting tier wins. Any refined label
  such as `TESTING`, `WORKING PLAN`, `RETRYING` or `ANSWERED` is lost.
- `refresh_runner_slot_context` in `src/sase/ace/tui/models/agent_runner_slots.py`
  recomputes the clan status the same way after projection. There are two copies:
  `_refresh_runner_slot_context_fallback` (about line 196) and
  `_apply_runner_capacity_snapshot` (about line 313). A fix only in the tree projection
  would be overwritten back to `RUNNING` on every refresh.
- Both sites aggregate raw `status` strings and ignore each row's `status_bucket`
  override. A family mirroring a settled monitor (for example status `TESTED` with
  bucket `Done`) is treated as running there. The clan count chip (`clan_member_counts`)
  already honors the override, so the aggregate and the chip can disagree.
- The family row's `TESTING` comes from `_mirror_root_from_child` in
  `src/sase/ace/tui/models/_agent_status_apply.py`. It copies `status`, `status_bucket`
  and the monitor/gate presentation fields (`monitor_start_status`,
  `monitor_stop_status`, `monitor_state`, `gate_start_status`, `gate_stop_status`,
  `gate_state`, `gate_accent`). `append_agent_row_status` in
  `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` uses those fields
  (through `gate_status_presentation` / `monitor_status_presentation`) to style the
  label. Without them, an unrecognized label such as `TESTING` renders `dim`.

This is presentation-only projection of already-loaded TUI rows. No Rust counterpart
exists (sase-core has no clan or group status aggregation), so the change stays in this
repo.

## Rule

The clan status is computed from direct clan members, deduplicated by `identity`, in
this order:

1. Aggregate over effective entries `(member.status, agent_status_bucket(member))` with
   the existing `aggregate_agent_group_effective_status`. The precedence ladder does not
   change: QUESTION, then pending PLAN/TALE/EPIC review, then FAILED, then
   RUNNING/STARTING, then QUEUED, then WAITING, then DONE.
2. If the aggregate is `"RUNNING"` and exactly one member has an effective bucket in
   `{"Running", "Starting"}`, and that member's bucket is `"Running"`, the clan mirrors
   that member:
   - copy its `status`;
   - set `status_bucket` to that member's effective bucket;
   - copy the seven monitor/gate presentation fields listed above.
3. Otherwise the clan takes the aggregate status. `status_bucket` is `None` and all
   seven presentation fields are cleared to `None`.
4. If there are no members (the aggregate is `None`), use the caller's fallback status
   and clear the same fields.

Mirroring only refines the label. The clan's bucket is unchanged, so BY_STATUS grouping,
clan-member ordering, count chips and summary counts behave as before. A lone `STARTING`
member therefore still leaves the clan at `RUNNING`. Clearing on every application keeps
repeated projections and runner-slot refreshes idempotent: when the lone member
finishes, the stale `TESTING` label and monitor styling are dropped.

## Changes

### 1. Shared helper in `src/sase/ace/tui/models/_agent_clan.py`

Add a public function, exported in `__all__`:

```python
def apply_clan_container_status(
    container: Agent,
    members: Iterable[Agent],
    *,
    fallback: str,
) -> None:
    """Project a clan container's status from its direct members (rule above)."""
```

Implementation notes:

- Deduplicate members by `identity`, keeping the first occurrence, as
  `clan_member_counts` does.
- Put the field copy/reset in one small private helper, for example
  `_copy_shell_status_presentation(target, source | None)`, so the seven field names
  appear once.
- Use `aggregate_agent_group_effective_status` and `agent_status_bucket` from
  `sase.agent.status_buckets`. Do not add a second precedence ladder.
- Write a docstring stating the lone-running-member rule and that the bucket is
  preserved.
- Leave `aggregate_clan_status` unchanged. Its other callers (completion candidates,
  directive completion, imported/parallel families, editor helper catalog) are out of
  scope.

### 2. Tree projection: `src/sase/ace/tui/models/_agent_tree.py`

In `_container_for_clan`, remove the `status = aggregate_clan_status(...) or "RUNNING"`
computation. Build the `Agent(...)` container as today with a placeholder
`status="RUNNING"`, then call
`apply_clan_container_status(container, runtime_members, fallback="RUNNING")` before
returning. Drop the now-unused `aggregate_clan_status` import if nothing else in the
module uses it.

### 3. Runner-slot refresh: `src/sase/ace/tui/models/agent_runner_slots.py`

In both `_refresh_runner_slot_context_fallback` and `_apply_runner_capacity_snapshot`,
replace the `if agent.is_clan_container:` block with:

```python
apply_clan_container_status(
    agent,
    clan_members(agent),
    fallback=runner_slot_display_status(agent.status, slot_queued=False),
)
```

Compute the fallback from `agent.status` before the helper resets anything. The function
call's argument order already does this. Update the local `from ._agent_clan import ...`
imports and remove `aggregate_clan_status` where it becomes unused, so ruff and
symvision stay clean. Keep the loop order unchanged.

### 4. Docs: `docs/ace.md`

In the "Clan rows aggregate member status…" paragraph (about line 1978), add one or two
sentences: when exactly one direct member is running, the clan row shows that member's
own status label and styling (for example a family running a `TESTING` monitor makes the
clan read `TESTING`) instead of the generic `RUNNING`. The status bucket, precedence and
count chip are unchanged. Keep the surrounding prose style.

## Tests

### New file `tests/ace/tui/models/test_agent_tree_clan_status.py`

Use the `_agent` helper from `tests/ace/tui/models/_agent_tree_helpers.py` (same clan
and generation defaults) together with `project_clan_tree`. Set extra fields such as
`status_bucket` and the monitor fields as attributes on the returned rows. Cover:

1. **Screenshot case.** Members: DONE, DONE, WAITING, and one row with
   `status="TESTING"`, `status_bucket="Running"`, `monitor_start_status="TESTING"`,
   `monitor_stop_status="TESTED"` and `monitor_state="running"`. Expect:
   - container `status == "TESTING"` and `agent_status_bucket(container) == "Running"`;
   - the monitor fields are copied;
   - `format_agent_option(container, ...)` plain text starts with `(TESTING`;
   - the style at the `TESTING` label equals the style at that label in the member row's
     own `format_agent_option` output (reuse the `_style_at` idea from
     `tests/ace/tui/models/test_agent_tree_rendering.py`).
2. **Two running members** (`TESTING` and `RUNNING`) give `RUNNING`, with the monitor
   fields `None`.
3. **Lone plain `RUNNING` member** alongside DONE/WAITING stays `RUNNING`.
4. **Precedence preserved.** A lone `TESTING` member plus one of `FAILED`, `QUESTION` or
   `PLAN` gives that higher-priority status, with no presentation fields copied.
5. **Lone `STARTING` member** plus DONE stays `RUNNING` (the bucket is preserved).
6. **Effective-bucket override.** A member with `status="TESTED"` and
   `status_bucket="Done"`, plus a DONE member, gives `DONE`, not `RUNNING`.
7. **Idempotent reprojection.** After case 1, set the lone member's `status="DONE"`,
   `status_bucket=None` and `monitor_state="completed"`, then call
   `project_clan_tree([container, *members])`. The new container reads `DONE`, with
   `monitor_start_status`, `monitor_stop_status` and `monitor_state` all `None`.
8. **Gate mirroring.** A lone running member carrying gate presentation fields
   (`gate_start_status`, `gate_stop_status`, `gate_state="settling"`, `gate_accent`) and
   a Running-bucket status copies all four gate fields.

### Runner slots: `tests/ace/tui/test_agent_runner_slots_families.py`

Using that file's `_agent` helper, build a clan from:

- one member with `status="TESTING"`, `status_bucket="Running"` and the monitor pair;
- one `WAITING` member with no slot request.

Both share `agent_clan` and `agent_clan_generation`. Call `project_clan_tree`, then
assert `projected[0].status == "TESTING"` after each of these:

- `refresh_runner_slot_context(projected)`, the fallback path with no limit;
- `refresh_runner_slot_context(projected, effective_limit=10)`, the Rust snapshot path.

Keep `test_first_refresh_promotes_all_slot_waiters_and_clan_aggregate` passing
unchanged: an all-queued clan must still read `QUEUED`.

### Existing tests

Run the existing clan, tree, rendering, summary-count and runner-slot suites. If any
existing test encoded the old `RUNNING` collapse for a lone refined label, update its
expectation to the new rule and note it in the final summary. Do not weaken precedence
or bucket assertions.

## Verification

- `just install` if the workspace virtualenv is stale.
- `just fmt`.
- `just check`: all lint gates plus the diff-scoped test lane. This change touches
  shared TUI projection and runner-slot code, so if the scoped run escalates or looks
  unusual, run `just check-full` through the `/sase_monitor` skill with the
  `TESTING`/`TESTED` status pair.
- Optional manual check: in `sase ace`, a clan whose only running member is a family
  running a `TESTING` monitor shows `(TESTING)` on the clan row and `Status: TESTING` in
  the CLAN detail header.

## Out of scope

- The CLAN MEMBERS roster line for a family still reads `RUNNING` when that family has
  more than one Running-bucket row (for example the agent and its monitor both reading
  `TESTING`). That is family-level aggregation in `_agent_display_clan_roster.py` and is
  not changed here.
- Clan status strings in completion candidates, directive completion, the editor helper
  catalog and the wait-dependency resolver are not changed.
