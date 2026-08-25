---
tier: tale
title: Clan lane runtime uses the family total, not the running shell
goal:
  A collapsed clan container's live runtime suffix takes its left-hand value from the
  lowest total runtime among the clan's own in-flight member rows, so a sequential
  family lane contributes the family's total runtime instead of the runtime of whichever
  shell inside that family happens to be executing.
size: medium
proposed_by: bbugyi200.athena.0dh
create_time: 2026-08-25 12:02:37
status: wip
---

# Plan

## Problem

`86bbbb532` added the clan-wide minimum to a collapsed clan lane, so an active clan
container renders `🏃‍♂️ <lowest-running-lane-runtime> / <clan-total-runtime>`.

When the lowest lane is a **sequential agent family**, the left value is wrong: it is
the runtime of the family's currently executing _shell_ rather than the family's total
runtime. Only agent nodes that are direct clan members should be considered; agent shell
nodes nested inside one of those families must not be considered at all.

### Reproduction (verified against the current tree)

A clan with two lanes at `now = 2026-07-19 09:05:05`:

- lane `solo`: `RUNNING`, run start `09:00:00`
- lane `fam`: a plan family whose root row (`09:00:00`) owns a settled planner shell
  (`09:00:00` → plan submitted `09:02:00`) and a running coder shell (`09:04:00` → now)

Row-by-row, the individual rows render:

| row          | suffix             |
| ------------ | ------------------ |
| `fam` family | `🏃‍♂️ 1m05s / 3m05s` |
| `solo`       | `🏃‍♂️ 3m05s`         |
| clan (today) | `🏃‍♂️ 1m05s / 5m05s` |

The clan lane shows `1m05s` — the coder shell's own runtime. The family's total is
`3m05s`, and `min(3m05s, 3m05s) = 3m05s`, so the clan lane must show `🏃‍♂️ 3m05s / 5m05s`.

## Root cause

Two cooperating pieces both reach _inside_ the family:

1. `clan_current_lane_rows()` in `src/sase/ace/tui/models/_agent_clan.py` deliberately
   resolves a sequential-family member to `current_family_shell_row(member)` — the shell
   currently executing inside that family — and returns that shell as the lane's
   representative row.
2. `compute_lowest_row_runtime()` in `src/sase/ace/tui/models/agent_time.py` prefers
   `_leaf_runtime_interval(row)` (the row's _own_ interval, excluding descendants) and
   only falls back to `_runtime_interval(row)` (the aggregate total that the row
   actually displays) when the leaf interval is not live.

Together they compute "the running shell's own runtime", which is exactly the value the
family row already shows on the **left** of its own `current-shell / total` suffix.

## Desired rule

> Each in-flight lane of a clan contributes exactly the total runtime that the lane's
> own row displays — the right-hand value of that row's suffix. A lane is in flight
> while it is executing; for a sequential family that means while any of its shells is
> executing, even after the family root row itself has settled.

That rule is uniform: a plain member lane contributes its own runtime, and a family lane
contributes the family total. Nothing inside a family is ever inspected for a runtime
value again — only for liveness.

## Changes

### 1. `src/sase/ace/tui/models/_agent_clan.py`

Rename `clan_current_lane_rows` → `clan_running_lane_rows` (the returned rows are no
longer "current shells"), and return the clan's own member rows:

```python
def clan_running_lane_rows(agent: Agent) -> tuple[Agent, ...]:
    """Return the clan's own in-flight member rows, one per running lane.

    Each row is a direct clan member, never a shell nested inside one of the
    clan's sequential families: a family lane is represented by the family
    row itself so it contributes the family total rather than the runtime of
    whichever shell is currently executing. A family lane counts as in
    flight while any of its shells is executing, which outlives the family
    root row's own status and ``stop_time``. Nested clan containers are not
    clan members (matching :func:`clan_members` and
    ``_lane_summary_projections``), so they are not walked.
    """
    if not agent.is_clan_container:
        return ()
    rows: list[Agent] = []
    seen: set[tuple[AgentType, str, str | None]] = set()
    for member in clan_members(agent):
        if (
            current_family_shell_row(member) is None
            and not agent_row_is_in_flight(member)
        ):
            continue
        if member.identity in seen:
            continue
        seen.add(member.identity)
        rows.append(member)
    return tuple(rows)
```

Notes:

- `current_family_shell_row()` keeps its existing role as the _liveness_ probe for a
  family lane: it returns `None` for clan containers and for non-family rows, and
  returns the last in-flight shell (agent or monitor) otherwise. It is no longer used to
  pick the row whose runtime is read.
- The old `is_agents_tab_agent_node(member)` re-check is dropped: `clan_members()`
  already applies it on the `is_clan_container` branch, which is the only branch this
  function can reach. Keep the `import` of `is_agents_tab_agent_node` only if other
  callers in the module still need it (they do — `clan_members`).
- Update the `__all__` entry to the new name.

### 2. `src/sase/ace/tui/models/agent_time.py`

In `compute_lowest_row_runtime()`, swap the preference order so the row's displayed
total wins, and keep the leaf interval only as a fallback:

```python
def compute_lowest_row_runtime(
    rows: Sequence["Agent"],
    now: datetime | None = None,
) -> str | None:
    """Return the smallest still-active total duration among *rows*.

    Each row contributes the same total its own row displays -- the aggregate
    across its descendants -- so a family row contributes the family total,
    not the runtime of the shell currently executing inside it. A row whose
    aggregate is not live falls back to its own interval, so a live row is
    never dropped just because its descendants have not started.
    """
    reference = now if now is not None else local_now()
    lowest: float | None = None
    for row in rows:
        if not should_display_runtime_suffix(row):
            continue
        interval = _runtime_interval(row, reference)
        if interval is None or not interval.active:
            interval = _leaf_runtime_interval(row, reference)
        if (
            interval is None
            or not interval.active
            or interval.terminal_time is not None
        ):
            continue
        if lowest is None or interval.elapsed_seconds < lowest:
            lowest = interval.elapsed_seconds
    if lowest is None:
        return None
    return format_compact_duration(lowest)
```

The leaf fallback is load-bearing, not defensive decoration: see
[Follow-up 1](#follow-up-1-a-family-total-can-collapse-to-0s) below — a running family
root whose only runtime child is still queued currently aggregates to an inactive `0s`,
and without the fallback that lane would silently vanish from the minimum.

### 3. `src/sase/ace/tui/widgets/_agent_list_render_layout.py`

- Update the import to `clan_running_lane_rows` and the call site in
  `build_runtime_suffix()`.
- Rename the local `current_shell_elapsed` to `leading_elapsed`: it now holds either a
  family's current-shell runtime (family branch) or a clan's lowest lane total (clan
  branch), so the old name is only true on one of the two branches.

No change is needed in `_agent_list_render_cache.py`: `_runtime_signature()` already
recurses through `runtime_children` and `followup_agents`, so every input the new
computation reads is already folded into the cache key.

## Tests

### `tests/test_agent_clan.py`

Rename the six `clan_current_lane_rows` tests to `clan_running_lane_rows` and update the
two family expectations:

- `..._family_lane_resolves_to_current_shell` → `..._family_lane_represents_the_family`:
  same fixture (settled root + running coder), now asserting `== (root,)`.
- `..._family_lane_resolves_to_running_monitor` →
  `..._family_lane_counts_a_running_monitor_shell`: same fixture, now asserting
  `== (root,)` — a running monitor shell keeps the lane live but never represents it.
- The remaining four (single-shell lane, settled/waiting/failed lanes, other
  clan/generation, non-clan row) keep their current expectations; only the function name
  changes.

Add one case:

- `..._skips_a_family_lane_whose_shells_have_all_settled`: settled root + settled coder
  → `()`.

### `tests/ace/tui/widgets/test_agent_list_runtime_clan_rendering.py`

Add the regression test that pins the reported bug — a clan whose family lane's total
differs from its current shell:

```python
def test_format_agent_option_clan_family_lane_contributes_family_total() -> None:
    ...  # root 09:00 + settled planner (plan at 09:02) + coder running from 09:04
    ...  # solo lane running from 09:00; now = 09:05:05
    assert clan_suffix.plain == "🏃‍♂️ 3m05s / 5m05s"
    assert "1m05s" not in clan_suffix.plain  # the coder shell's own runtime
```

Build the family with the plan-family shape already used by
`test_format_agent_option_active_family_shows_current_continuation_first` in
`tests/ace/tui/widgets/test_agent_list_runtime_rendering.py` (root with a `--plan`
workflow step child and a `--code` follow-up, both in `root.runtime_children`), plus
`agent_clan` / `agent_clan_generation` on the clan-member rows. Assert in the same test
that the family row itself still renders `🏃‍♂️ 1m05s / 3m05s`, so the family row's own
two-value contract is pinned next to the clan's.

Also:

- Rename `test_format_agent_option_clan_single_family_lane_matches_family_current_shell`
  → `..._matches_family_total` and change its assertion to compare the clan's left value
  against the **right** half of the family row's suffix.
- Add `test_format_agent_option_clan_family_lane_falls_back_when_total_is_not_live`: a
  `RUNNING` family root (run start `09:00`) whose only `runtime_children` entry is a
  queued `WAITING` member with no `run_start_time`. The family row renders
  `🏃‍♂️ 3m05s / 0s` today (see
  [Follow-up 1](#follow-up-1-a-family-total-can-collapse-to-0s)), and the clan lane must
  still show `3m05s`, never `0s` and never an empty left value. Comment the test with a
  pointer to that follow-up so it is re-evaluated when the aggregate is fixed.

The existing clan rendering tests (two-plain-lane minimum, seconds-not-string ordering,
waiting-lane-has-no-slash, and the two async patching tests) must keep passing
unchanged.

### `tests/ace/tui/widgets/test_agent_render_cache.py`

No change expected; confirm the two clan cache tests still pass.

### Visual snapshots

`tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py` →
`running_clan_runtime_agents()` currently builds its family lane as a chain (root →
coder), where the family total happens to equal the current shell's runtime, so the
snapshot cannot tell the two apart. Give the family root a second, settled shell so the
values diverge:

- add `family_review`: `DONE`, `09:06:00` → `09:09:00`, `parent_timestamp` =
  `family_root.raw_suffix`, `role_suffix="--review"`, same clan/generation as the other
  members.

With `now = 09:45:00` that yields (verified by running the amended fixture through
`compute_row_runtime`/`build_runtime_suffix`): family total `38m` (`3m` review + `35m`
coder), family current shell `35m`, solo `45m`, clan total `45m`. The clan row renders
`🏃‍♂️ 35m / 45m` before the fix and must render `🏃‍♂️ 38m / 45m` after it.

In `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py` →
`test_running_clan_runtime_png_snapshots`:

- collapsed: only the clan row is visible, so assert `38m` and `45m` appear and that
  `35m` does **not** — that single absence is what fails loudly if the clan lane
  regresses to the current shell.
- expanded: `expect_state("agent_count", 5)` (one more row), and assert `35m`, `38m`,
  and `45m` all appear — the family row now reads `35m / 38m`.

Rebaseline the two goldens (`agents_running_clan_runtime_collapsed_120x40.png`,
`agents_running_clan_runtime_expanded_120x40.png`) with:

```bash
just test-visual -- --sase-update-visual-snapshots
```

and eyeball the regenerated PNGs (or the artifacts under `.pytest_cache/sase-visual/`)
before committing them.

## Docs

- `docs/ace.md`: the clan sentence added by `86bbbb532` says "a sequential family lane
  contributes its currently executing shell". Change it to say the lane contributes the
  family's **total** runtime, i.e. every lane contributes the same value its own row
  shows to the right of its suffix.
- `docs/agent_families.md`: same correction in the "Clan summary folding" vicinity —
  "where a sequential-family lane contributes its own currently executing shell" becomes
  the family total.

## Verification

```bash
just install
just check
just test-visual
```

Then, before landing:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '...'
```

## Out of scope — pre-existing defects found while diagnosing this bug

Both are real, reproduced against the current tree, and both live in
`_aggregate_runtime()` in `src/sase/ace/tui/models/agent_time.py`. Neither is fixed
here: each changes what **family rows** display, well beyond the clan lane this plan
repairs. File each with `/sase_new_task` (type `bug`) after landing, quoting the
reproductions below.

### Follow-up 1: a family total can collapse to `0s`

A running family root whose only runtime child is a queued member renders
`🏃‍♂️ 3m05s / 0s` — the total reads as zero while the row is visibly running for three
minutes. Reproduced through the real loader pipeline (`normalize_loaded_agents`, which
is what puts a queued follow-up into `runtime_children` as well as `followup_agents`):

- root: `RUNNING`, start/run start `09:00:00`
- child: `WAITING`, start `09:01:00`, `run_start_time=None`, `parent_timestamp` = root
- at `now = 09:03:05` the root row renders `🏃‍♂️ 3m05s / 0s`

Cause: `_aggregate_runtime()` skips the row's own interval whenever it has any runtime
children, and the single queued child contributes `run_started_at=None`, so the
aggregate is `0s` and inactive. Note that
`test_format_agent_option_active_family_shows_current_root_and_total` does not catch
this because its fixture puts the queued child only in `followup_agents`, while the real
loader puts it in both lists.

### Follow-up 2: chained family shells drop their ancestors' runtime

`_aggregate_runtime()` walks to the leaves and appends only rows with no runtime
children, so an ancestor's own interval is dropped as soon as it has one. For a family
whose members chain (root → A → B, which is what `sase pipe` style attachment produces,
since a successor's `parent_timestamp` names the shell that launched it), the family
total equals B's runtime alone; root's and A's intervals are silently discarded. The
already-committed `test_format_agent_option_completed_family_keeps_single_total_suffix`
encodes this: root `09:00`→`09:01` plus child `09:01`→`09:02` totals `1m`, not `2m`.

Fixing this would also change what the clan lane shows for chained family lanes, since
after this plan the clan reads exactly the family's total — which is the point.
