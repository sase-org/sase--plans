---
tier: tale
title: Fix clan-row agent counts miscounting a handed-off family planner as Running
goal: A clan row's status chip buckets each member by the effective bucket its projection
  already computed, so a family whose approved planner has handed off to a finished
  coder counts as Done. The sase-9x clan row reads [R2 W2 D3], and clan chips reconcile
  with their enclosing tribe panel title.
create_time: 2026-07-27 07:38:44
status: done
---

- **PROMPT:** [202607/prompts/clan_row_running_count.md](prompts/clan_row_running_count.md)

# Fix clan-row agent counts miscounting a handed-off family planner as Running

## Symptom

On the ACE Agents tab, the `sase-9x` clan row renders `[R3 W2 D2]`. Its seven visible members are:

| member         | status      | correct bucket |
| -------------- | ----------- | -------------- |
| `sase-9x.5`    | `RUNNING`   | Running        |
| `sase-9x.3`    | `RUNNING`   | Running        |
| `sase-9x.land` | `WAITING`   | Waiting        |
| `sase-9x.6`    | `WAITING`   | Waiting        |
| `sase-9x.4`    | `DONE`      | Done           |
| `sase-9x.2`    | `DONE`      | Done           |
| `sase-9x.1`    | `TALE DONE` | Done           |

So the chip should read `[R2 W2 D3]`.

A second, independent symptom of the same defect: the `@epic` tribe panel title reads `· 20 [R4 W5 D11]`, but its three
clan rows sum to `R5 W5 D10` (`sase-9y` `[R2 W3]`, `sase-9x` `[R3 W2 D2]`, `sase-9w` `[D8]`). The panel title is right
and the clan row is wrong — they are computed by two different code paths.

## Root cause

`clan_member_counts()` in `src/sase/ace/tui/models/_agent_clan.py` buckets each clan member like this:

```python
projected_statuses = agent_status_projections((member,))
aggregate_status = aggregate_clan_status(
    status.agent.status for status in projected_statuses
)
bucket = status_bucket_for_values(aggregate_status or member.status)
```

`agent_status_projections()` returns `ConcreteAgentStatus(agent, bucket)` pairs where `bucket` is the **effective**
bucket. For a sequential family container that bucket is produced by `family_member_status_buckets()` in
`src/sase/ace/tui/models/agent_family_members.py`, which deliberately settles a _non-final_ member whose status is in
`APPROVED_PLAN_STATUSES` (`PLAN APPROVED` / `TALE APPROVED`) to `Done` — that planner has already handed its work off to
its successor.

`clan_member_counts()` discards that correction. It re-reads the **raw** `status.agent.status` and re-buckets it. Raw
`TALE APPROVED` is a member of `ACTIVE_PLAN_HANDOFF_STATUSES`, so `status_bucket_for_values()` maps it back to
`Running`; `aggregate_agent_group_status()` then short-circuits on `if "Running" in buckets ... return "RUNNING"`, and
the entire family member is counted as Running — overriding both its own effective buckets and the `TALE DONE` status
its own row displays.

### Confirmed against live agent state

Loading the real tree with `sase.ace.tui.models.agent_loader.load_all_agents()` on 2026-07-27 reproduces the screenshot
exactly:

```
sase-9x.1   own='TALE DONE'   aggregate='RUNNING' -> bucket 'Running'
  projections = [('sase-9x.1--plan', 'TALE APPROVED', 'Done'),
                 ('sase-9x.1--code', 'TALE DONE',     'Done')]

clan_member_counts(sase-9x)          -> running=3, waiting=2, done=2   # the rendered chip
agent_lane_status_counts([sase-9x])  -> running=2, waiting=2, done=3   # correct
```

Both projections already carry effective bucket `Done`, yet the aggregate over their raw statuses returns `RUNNING`.

`sase-9x.1` is the only clan member with `container_kind: family` in the agent name registry; the other six are plain
agents whose projections are just themselves, so their raw status and effective bucket always agree. That is why exactly
one of seven rows is misattributed.

### Why the tribe panel title is correct

`agent_panel_counts()` in `src/sase/ace/tui/actions/agents/_display_panel_titles.py` calls `agent_lane_status_counts()`,
whose `_lane_summary_projections()` buckets each clan member from _its own_ status (`TALE DONE` -> `Done`) and never
re-aggregates sub-rows. Only the clan chip re-aggregates.

Note also that `clan_member_status_priority()` — which orders rows inside the clan — already buckets from the member's
own status, which is why `sase-9x.1` sorts last with the Done rows while the chip simultaneously counts it as Running.
The chip is the sole outlier.

## Fix

Make the clan aggregate honor the effective buckets the projection already computed.

### 1. `src/sase/agent/status_buckets.py` — add a bucket-aware aggregate

```python
#: Canonical status per bucket, used when a caller supplies an effective
#: bucket that intentionally overrides a row's raw status.
_BUCKET_REPRESENTATIVE_STATUS: dict[str, str] = {
    "Stopped": "QUESTION",
    "Failed": "FAILED",
    "Starting": "STARTING",
    "Running": "RUNNING",
    QUEUED_STATUS_BUCKET: QUEUED_STATUS,
    "Waiting": "WAITING",
    "Done": "DONE",
}


def aggregate_agent_group_bucket(
    entries: Iterable[tuple[str, str]],
) -> str | None:
    """Return the aggregate display bucket for ``(status, effective_bucket)`` rows.

    A row whose effective bucket already matches its raw status keeps that raw
    status, so the priority ladder in :func:`aggregate_agent_group_status` still
    sees the ``QUESTION`` / pending-review / ``KILLED`` special cases verbatim.
    A row whose projection deliberately overrode its bucket — a handed-off
    approved planner settles to ``Done`` — is substituted with that bucket's
    canonical status so the override survives aggregation.
    """
    resolved = tuple(
        status
        if status_bucket_for_values(status) == bucket
        else _BUCKET_REPRESENTATIVE_STATUS.get(bucket, status)
        for status, bucket in entries
    )
    aggregate = aggregate_agent_group_status(resolved)
    return None if aggregate is None else status_bucket_for_values(aggregate)
```

Delegating to the existing ladder rather than writing a second bucket-level ladder is deliberate: it keeps the two from
drifting, and it makes the change provably a no-op for every row whose effective bucket already equals its natural
bucket. Every entry in `_BUCKET_REPRESENTATIVE_STATUS` must round-trip (`status_bucket_for_values(value) == key`); add a
test that asserts this so a future bucket rename cannot silently break the map.

Do **not** change `aggregate_agent_group_status`'s signature or behavior — its other callers rely on it returning a
_status_ string, not a bucket.

### 2. `src/sase/ace/tui/models/_agent_clan.py` — use it in `clan_member_counts()`

```python
projected_statuses = agent_status_projections((member,))
bucket = aggregate_agent_group_bucket(
    (status.agent.status, status.bucket) for status in projected_statuses
) or status_bucket_for_values(member.status)
```

This replaces the `aggregate_clan_status(...)` + `status_bucket_for_values(...)` pair. Empty-projection behavior is
preserved: the helper returns `None` for an empty iterable, so the fallback to `member.status` still applies.
`aggregate_clan_status` stays exported — it has several other callers.

### Surfaces fixed by this one change

All three read `clan_member_counts()`:

- Agents-tab clan / family row chip — `src/sase/ace/tui/widgets/_agent_list_render_agent.py` (`family_chip`)
- Clan detail panel chip — `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`
- Cleanup clan modal chip — `src/sase/ace/tui/modals/agent_cleanup_clan_modal.py`

The row render cache key in `src/sase/ace/tui/widgets/_agent_list_render_cache.py` already includes the computed counts
(`parallel_family_counts`), so corrected counts invalidate the cache on their own; no cache handling is needed.

## Tests

### `tests/test_agent_status_buckets.py`

Unit-cover `aggregate_agent_group_bucket`:

1. Effective-bucket override wins: `[("TALE APPROVED", "Done"), ("TALE DONE", "Done")]` -> `"Done"`.
2. Passthrough parity: for a spread of statuses, assert
   `aggregate_agent_group_bucket((s, status_bucket_for_values(s)) for s in xs)` equals
   `status_bucket_for_values(aggregate_agent_group_status(xs))`. This is the guarantee that the fix is a no-op wherever
   no override is present.
3. Empty input returns `None`.
4. `KILLED` and `WAITING INPUT` still take their special-case paths (`Failed` and `Stopped` respectively). Both have no
   natural bucket of their own, so they must be passed through unrewritten — a naive bucket-level ladder would break
   them, and this test pins that.
5. `_BUCKET_REPRESENTATIVE_STATUS` round-trips: every `(bucket, status)` pair satisfies
   `status_bucket_for_values(status) == bucket`.

### `tests/ace/tui/models/test_agent_summary_status_counts.py`

Add `test_clan_counts_settle_handed_off_family_planner_as_done`, mirroring the real loader shape: a family container row
(own status `TALE DONE`, `is_family_container_row` true) owning both a `--plan` child (`TALE APPROVED`) and a `--code`
child (`TALE DONE`), plus one plain `RUNNING` clan member.

Assert `clan_member_counts(container) == ClanStatusCounts(running=1, done=1)`, and assert it agrees with
`agent_lane_status_counts((container,), ())` on running/waiting/done — the cross-path agreement is the actual invariant
the screenshot violated.

Implementation note for whoever writes it: the module's existing `_agent()` / `_active_family()` helpers build a two-row
family where the planner _is_ the container row. That shape does reproduce the aggregate bug, but the lane path also
reads `Running` for it because the container row's own status is the underived `TALE APPROVED`, so it cannot show the
cross-path divergence. Build the three-row shape instead, and assert `clan_members(container)` returns exactly the
family container row plus the plain member before asserting counts — a naive fixture leaks `--code` as a second direct
clan member.

Confirm the new test fails before the fix (`running=2, done=0`) and passes after.

### Existing tests

`tests/test_agent_clan.py` and the existing `clan_member_counts` assertions in
`tests/ace/tui/models/test_agent_summary_status_counts.py` should stay green **unchanged** — none of their fixtures
involve an approved-planner family, so no substitution occurs. If any of them needs editing, that is a signal the fix is
broader than intended; stop and re-check.

## Verification

1. `just install` first — workspace directories are ephemeral and dependencies may be stale.
2. `just check`.
3. `just test-visual`. The clan PNG fixture family in `tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py`
   is `RUNNING`/`RUNNING`, so no golden should move. If a golden does move, stop and re-examine rather than accepting it
   with `--sase-update-visual-snapshots`.
4. Optional live check: run `sase ace`, expand a clan holding a family whose planner is `PLAN APPROVED` /
   `TALE APPROVED` and whose coder has finished, and confirm the clan chip matches the visible member rows and
   reconciles with the enclosing tribe panel title.

## Scope and non-goals

- Do not touch `agent_lane_status_counts` / `_lane_summary_projections`. That path is already correct and is the
  reference the fix is validated against.
- Several other call sites aggregate raw statuses the same way. They were checked during diagnosis and are **not**
  user-visible today; leave them alone here and note them as follow-up candidates:
  - `clan_roster_entries` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py` builds
    `family_status` from raw statuses while computing `family_member_status_buckets()` for its children two lines later.
    It happens to render correctly because its `family_rows()` projection yields the container row plus `--code` (not
    `--plan`), so no `TALE APPROVED` row reaches the aggregate. It is latent, not active.
  - `agent_tribe_summary.py` (tribe aggregate status) and `_agent_tree.py` (clan container display status) aggregate raw
    statuses too; both produced correct results on the live tree.
- No Rust change. `sase-core` was checked: it has no counterpart to `status_buckets` or `clan_member_counts`, so this
  stays presentation-side Python.
