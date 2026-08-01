---
tier: tale
title: Stop double counting queued agents as waiting agents in Agents-tab counts
goal: Every Agents-tab agent count treats queued agents as a disjoint subset of waiting
  agents, so a six-member clan with two global-cap waiters renders `[Q2 W4]` instead
  of `[Q2 W6]`, while every displayed WAITING status stays exactly as it is today.
create_time: 2026-07-25 09:43:37
status: done
---

- **PROMPT:** [prompts/202607/queued_waiting_double_count.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/queued_waiting_double_count.md)
- **AGENTS:**
  - [bbugyi200.athena.kh.f2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.kh.f2/README.md)
  - [bbugyi200.athena.kh.f2--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.kh.f2.md#member-code)
- **COMMITS:**
  - [b1b5db1](https://github.com/sase-org/sase/commit/b1b5db1fd7392a731ffdf7ec3c0fb848a7489498) — fix(ace): exclude queued agents from waiting counts

# Stop double counting globally queued agents as waiting agents

## Problem

Every Agents-tab status-count surface counts a globally queued agent twice: once under `queued` (`Q`) and again under
`waiting` (`W`). The chip therefore claims more agents than the aggregate actually contains.

From the reported screenshot, the `sase-99` clan has exactly **6** members, all `WAITING`, of which 2 are waiting on the
global runner cap:

```
(WAITING) [Q2 W6] sase-99          ← clan row       (should be [Q2 W4])
Status: WAITING [Q2 W6]            ← CLAN section   (should be [Q2 W4])
@epic · 93 [R7 Q7 W28 D58]         ← tribe title    (7+28+58 = 93 only because Q is inside W; should be W21)
109 [10/10 running · 7 queued · 28 waiting · 71 done]   ← info strip (should be 21 waiting)
```

Only the counts are wrong. The status labels themselves are correct and must not change: individual rows keep
`(WAITING ▶10/10)`, clans keep the aggregate `WAITING` status, and the `Q` metric keeps its meaning ("waiting
specifically on the global runner cap").

## Root cause

`queued` is computed as a second, independent pass rather than as a partition of the `Waiting` bucket. Both count
producers in `src/sase/ace/tui/models/_agent_clan.py` do this:

1. `clan_member_counts()` — buckets each direct clan member by `status_bucket_for_values(member.status)`, so a globally
   queued member lands in `waiting`. Separately it sums `agent_is_globally_queued()` over
   `agent_status_projections(members)` into `queued`.
2. `_status_counts_for_projections()` — increments `queued` unconditionally at the top of the loop, then falls through
   the `if/elif` chain and increments `waiting` for the very same projection.

`agent_is_globally_queued()` (`src/sase/ace/tui/models/agent_runner_slots.py`) requires `agent.status == "WAITING"`, and
`"WAITING"` is neither an asking status nor a Failed/Done status. So **every globally queued row is always also a
`Waiting`-bucket row** — the overlap is total, not incidental, and `queued + waiting` always overstates the aggregate by
exactly the queued count.

The `Q` metric was added in `ca348d703` ("feat(ace): show globally queued agent counts") as an additive metric, which is
where the overlap entered.

## Approach

Make `queued` a **disjoint refinement of the `Waiting` bucket** in both producers: a row that is globally queued
increments `queued` _instead of_ `waiting`, never both. Nothing else about bucketing changes.

This gives the invariant the user is asking for — for any aggregate,
`stopped + running + queued + waiting + failed + done` never exceeds the number of agents in that aggregate — and it
fixes every display surface at once, because all of them read from these two functions:

| Surface (Agents tab)                                                                                       | Producer                                                       |
| ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Clan / parallel-family row chip (`src/sase/ace/tui/widgets/_agent_list_render_agent.py:372`)               | `clan_member_counts()`                                         |
| Right-panel `CLAN` section (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py:153`)            | `clan_member_counts()`                                         |
| Clan cleanup modal rows (`src/sase/ace/tui/modals/agent_cleanup_clan_modal.py:351`)                        | `clan_member_counts()`                                         |
| Tribe panel border title (`src/sase/ace/tui/actions/agents/_display_panel_titles.py:68`)                   | `agent_lane_status_counts()`                                   |
| Top info strip `N waiting` (`src/sase/ace/tui/actions/agents/_display_detail_info.py:57`)                  | `agent_lane_status_counts()`                                   |
| Right-panel `TRIBE` section + roster unit chips (`src/sase/ace/tui/models/agent_tribe_summary.py:255,271`) | `agent_summary_status_counts()` / `agent_lane_status_counts()` |

No renderer, chip formatter, or dataclass field changes are needed — `format_agent_count_chip()` already zero-suppresses
`W0`, so a fully queued clan simply renders `[Q6]`.

### Deliberate consequences

- `clan_member_counts().queued` becomes a **member count** (how many direct members are queued) on the same axis as
  every other bucket, rather than a count of projected concrete rows. For the one shape where those differ — a clan
  whose direct member is a container holding queued rows — the member contributes at most 1. This is required for the
  invariant; the existing nested-family test still yields `queued == 1`.
- A container member whose own bucket is not `Waiting` no longer contributes to `queued`. Its aggregate status already
  says it is not waiting, so reporting it as queued was the inconsistency, not the fix.

### Not doing

- **Not** changing any status string, `aggregate_clan_status()`, the `(WAITING ▶10/10)` row suffix, or the runner-slot
  queue-position detail. The user explicitly wants the displayed waiting statuses preserved.
- **Not** re-sourcing the info strip's `· N queued` segment. It comes from
  `RunnerCapacitySnapshot.global_cap_queue_count`, computed over every loaded waiter in `refresh_runner_slot_context()`,
  and it sits with `10/10 running` as part of the runner-capacity story. Only the `N waiting` metric beside it is
  double-counted, and only that is being fixed. (This sourcing asymmetry — capacity-wide `queued` next to visible-lane
  `waiting` — predates this bug and stays as-is.)
- **Not** touching the group banner summary (`banner_summary_text()` in
  `src/sase/ace/tui/models/agent_groups/_tree.py`); it renders only `N agents · X running · Y failed · Z awaiting` and
  has no queued/waiting pair.
- **Not** touching the Admin Center Statistics `Waiting` lifecycle metric — different tab, different data source.

## Implementation

All source changes are in `src/sase/ace/tui/models/_agent_clan.py`.

### 1. `clan_member_counts()` — partition the Waiting bucket

Remove the trailing `queued = sum(...)` pass and decide queued-vs-waiting inside the existing member loop, so both
metrics are produced by one pass over the same member set:

```python
def _member_is_globally_queued(member: Agent) -> bool:
    """Return whether a clan member's concrete row waits on the global cap."""
    return any(
        agent_is_globally_queued(status.agent)
        for status in agent_status_projections((member,))
    )
```

and in the bucket chain:

```python
        elif bucket == "Waiting":
            if _member_is_globally_queued(member):
                queued += 1
            else:
                waiting += 1
```

Initialize `queued` alongside the other accumulators and drop the now-unused local `members` hoist if nothing else uses
it (`clan_members(agent)` can go back to being iterated directly).

Keep `agent_status_projections((member,))` rather than a bare `agent_is_globally_queued(member)` — that is what lets a
container member (e.g. a parallel family sitting between the clan container and the queued agent) still resolve to its
concrete row. For a plain leaf member the projection is the member itself, so the common case is unchanged.

Perf note: this is the same aggregate work as today. The removed call already fanned out to one `_summary_projections()`
per member internally; it is now called per member directly, with no I/O and no new data structures. The row render
cache key in `src/sase/ace/tui/widgets/_agent_list_render_cache.py:208` already embeds the whole `family_counts`
dataclass, so changed counts invalidate cached rows automatically.

### 2. `_status_counts_for_projections()` — same partition

Hoist the queued test into a local and gate the waiting branch on it:

```python
    for projection in projected:
        projected_agent = projection.status.agent
        is_queued = agent_is_globally_queued(projected_agent)
        ...
        if is_queued:
            queued += 1
        ...
        elif bucket == "Waiting":
            if not is_queued:
                waiting += 1
```

`total` is unaffected, so `agent_lane_status_counts().total` and the info strip's headline lane count stay identical.

### 3. Document the contract

Add a one-line note to the `ClanStatusCounts` and `_AgentSummaryStatusCounts` docstrings stating that `queued` is a
disjoint subset of the `Waiting` bucket (queued rows are excluded from `waiting`), so a future metric addition does not
reintroduce an additive overlay. `unread` remains an intentional overlay and is unchanged.

### 4. Update the count assertions that encode the old overlap

Each of these currently asserts the double-counted value. Recompute, do not blanket-decrement — verify each fixture's
member set:

| File                                                                                           | Now                                                                                | After                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tests/test_agent_clan.py:98` (`test_clan_queue_count_projects_parallel_member_behind_family`) | `(counts.queued, counts.waiting) == (1, 1)`                                        | `(1, 0)`                                                                                                                                                         |
| `tests/ace/tui/models/test_agent_summary_status_counts.py:299-301`                             | `(3, 1, 3)` / `(3, 1, 3)` / `(1, 3)`                                               | `(3, 1, 2)` / `(3, 1, 2)` / `(1, 2)`                                                                                                                             |
| `tests/ace/tui/models/test_agent_tribe_summary.py:265,268-270`                                 | `(1, 3)` twice                                                                     | `(1, 2)` twice (the `unrelated` `(0, 1)` assertion is unchanged)                                                                                                 |
| `tests/ace/tui/test_agent_panel_titles.py:418-420`                                             | `queued == 1`, `(running, waiting, read) == (2, 1, 1)`, `sum(metric_items()) == 5` | `queued == 1`, `(2, 0, 1)`, `sum(metric_items()) == 4`                                                                                                           |
| `tests/ace/tui/test_agent_cleanup_clan_modal.py:148`                                           | `"[Q1 W2]"`                                                                        | `"[Q1 W1]"`                                                                                                                                                      |
| `tests/ace/tui/widgets/test_agent_display_clan.py:294`                                         | `"Status: WAITING [Q1 W3]\n"`                                                      | `"Status: WAITING [Q1 W2]\n"`                                                                                                                                    |
| `tests/ace/tui/widgets/test_agent_display_tribe.py:234`                                        | `detail.plain.count("[Q1 W2]") == 2`                                               | `count("[Q1 W1]") == 2`                                                                                                                                          |
| `tests/ace/tui/widgets/test_agent_render_cache.py:188`                                         | `"[Q1 W1]" in global_wait[0].plain`                                                | `"[Q1]"` (the `"[W1]"` dependency-wait assertion above it is unchanged)                                                                                          |
| `tests/ace/tui/widgets/test_agent_parallel_family_count_chips.py:76,100,121`                   | `[S1 R2 Q1 W1 F1 D1]`                                                              | `[S1 R2 Q1 F1 D1]` — the fixture's single `WAITING` member is the queued one, so `W` drops out entirely; the surrounding `" ×9 ↻1 "` / `" ×9 +2 "` prefixes stay |

`tests/ace/tui/widgets/test_agent_info_panel.py:252` (`"[10/10 running · 1 queued · 4 waiting]"`) must **not** change —
it feeds `update_agent_counts()` hand-built numbers and only exercises the widget's formatting.

In `test_agent_panel_titles.py`, the `sum(metric_items())` assertion becoming `4` now equals `counts.lane_count == 4`.
Add a short comment there noting that the metric sum matching the lane count is the anti-double-count invariant.

### 5. New regression pinning the invariant

Add one model-level test to `tests/ace/tui/models/test_agent_summary_status_counts.py` (next to
`test_queue_counts_are_orthogonal_and_dedupe_container_flat_rows`) that reproduces the reported screenshot shape: a
six-member clan where every member is `WAITING` and exactly two are globally queued (`pid` set, `wait_runners` set,
`wait_runners_explicit` false, `slot_requested_at` set). Assert:

- `clan_member_counts(container) == ClanStatusCounts(queued=2, waiting=4)`
- for the same rows, `agent_lane_status_counts(...)` gives `total == 6`, `queued == 2`, `waiting == 4`
- `queued + waiting == 6`, i.e. the sum of every non-`unread` metric equals the member count

Build the clan with `project_clan_tree()`, matching the existing tests in that file.

### 6. Regenerate the two affected PNG goldens

Only two goldens contain a queued row.

`tests/ace/tui/visual/snapshots/png/agents_queued_clan_counts_120x40.png` — fixture `queued_clan_agents()` has one
global-cap waiter and one explicit drain-barrier waiter. Update the inline assertions in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`:

- line 52: `"▲ @epic · 2 [Q1 W2]"` → `"▲ @epic · 2 [Q1 W1]"`
- line 57: `"(WAITING) ×2 [Q1 W2]"` → `"(WAITING) ×2 [Q1 W1]"`
- line 59: `"Status: WAITING [Q1 W2]"` → `"Status: WAITING [Q1 W1]"`
- line 62: `"2  [0/10 running · 1 queued · 2 waiting]"` → `"... · 1 queued · 1 waiting]"`

`tests/ace/tui/visual/snapshots/png/agents_runner_slot_waits_120x40.png` — fixture `runner_slot_wait_agents()`. Update
`tests/ace/tui/visual/test_ace_png_snapshots_agents.py:152` from `"2  [0/10 running · 1 queued · 2 waiting]"` to
`"... · 1 queued · 1 waiting]"`. The `drain-barrier` / `global-cap` / `drain barrier` / `eligible` SVG assertions and
the `runner_slot_queue_position == 2` assertion are unchanged — the wait rows themselves are untouched.

Regenerate with `just test-visual --sase-update-visual-snapshots` (or the equivalent pytest invocation scoped to those
two tests), then confirm the accepted diffs change only the `W` digit in the chips and the `N waiting` segment in the
info strip. `agents_parallel_family_counts_120x40.png` has no queued member and must stay byte-identical; treat a diff
there as a regression.

## Verification

1. The new six-member-clan regression fails before the source change (`waiting == 6`) and passes after (`waiting == 4`).
2. `pytest tests/test_agent_clan.py tests/ace/tui/models/test_agent_summary_status_counts.py tests/ace/tui/models/test_agent_tribe_summary.py tests/ace/tui/test_agent_panel_titles.py tests/ace/tui/test_agent_cleanup_clan_modal.py tests/ace/tui/widgets/test_agent_display_clan.py tests/ace/tui/widgets/test_agent_display_tribe.py tests/ace/tui/widgets/test_agent_render_cache.py tests/ace/tui/widgets/test_agent_parallel_family_count_chips.py tests/ace/tui/widgets/test_agent_info_panel.py`
   passes.
3. `test_clan_member_counts_ignores_globally_queued_leaf` still returns an all-zero `ClanStatusCounts()` — the leaf-row
   fix from `53d36a298` must not regress.
4. Both PNG snapshot tests pass without `--sase-update-visual-snapshots` after regeneration;
   `agents_parallel_family_counts_120x40.png` is byte-identical.
5. `just install` then `just check` passes.

## Notes

- Run `just install` before `just check`: workspace directories are ephemeral, and a stale `sase_core_rs` wheel makes
  `format_agent_option()` raise on `validate_agent_username`, which breaks the widget tests before they can run.
- No Rust-core work. This is presentation-layer count aggregation for the Textual Agents tab, which lives in the Python
  repo; there is no cross-frontend backend behavior here.
- TUI-performance memory reviewed: the change is pure in-memory aggregation of the same order as today, does no I/O, and
  adds no render-path stat/glob. The existing render cache key already covers the changed counts.
