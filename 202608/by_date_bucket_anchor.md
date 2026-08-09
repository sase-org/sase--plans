---
tier: tale
title: Bucket BY_DATE agent groups on the same anchor their subgroups already use
goal:
  In the Agents tab's `by date` grouping, an agent's L0 date bucket, its L1 subgroup
  label, its sort position, and the finish timestamp rendered on its row all derive from
  one shared anchor, so a clan that started Friday night and finished Saturday morning
  renders under `Yesterday`, not under `This Week`.
proposed_by: bbugyi200.athena.wc
create_time: 2026-08-09 07:51:30
status: done
---

- **PROMPT:**
  [prompts/202608/by_date_bucket_anchor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/by_date_bucket_anchor.md)

# Fix the BY_DATE bucket/subgroup anchor mismatch

## Symptom

On the Agents tab with `by date` grouping (`o`), on 2026-08-09 at 07:37 local, the
`@epic` tribe panel rendered:

```
Yesterday                                        55 agents
  21:00 ... 00:00                                (7 hour subgroups, all Aug 8)
This Week                                        19 agents
  Sat Aug 8                                      19 agents
    (DONE) ×10 [D10] sase-h8       Aug 8 10:56 · 4h37m
    (DONE)  ×9  [D9] sase-ha       Aug 8 10:06 · 2h33m
```

Two clans that finished on Aug 8 landed under `This Week` → `Sat Aug 8` while
`Yesterday` _is_ Aug 8. The same panel showed the mirror-image case: clan `sase-hn` sat
under `Yesterday` → `00:00` while its row timestamp read `00:10:44` with no date prefix
— the renderer's "same calendar day as now" form, i.e. it actually finished _today_.

## Root cause

`src/sase/ace/tui/models/agent_groups/_buckets.py` uses **two different anchors** for
one row:

- `date_bucket_for()` (L0: `Today` / `Yesterday` / `This Week` / `Earlier`) buckets on
  `agent.start_time`, unconditionally.
- `hour_anchor_time()` — used by `date_subgroup_bucket_for()` for the L1 label, and by
  `walk_anchors()` in `_keys.py` for sort position — returns `stop_time` (falling back
  to `start_time`) for agents whose status is in `_TERMINAL_STATUSES` (`DONE`,
  `PLAN DONE`, `TALE DONE`, `PLAN REJECTED`, `EPIC CREATED`, `STOPPED`, `FEEDBACK`).

The rendered right-hand timestamp is a third consumer of the finish time:
`compute_row_runtime()` in `src/sase/ace/tui/models/agent_time.py` formats the row's
terminal time, which for a clan container is `max()` of its members' finish times.

So any row whose start and finish fall on different calendar days gets its L0 bucket
from the **start** day and its L1 label plus its visible timestamp from the **finish**
day. Nothing in the tree reconciles the two, which is why a `Sat Aug 8` banner can
appear inside `This Week` while a sibling `Yesterday` bucket covers the same date.

Clan containers make this easy to hit: `_build_clan_container()` in
`src/sase/ace/tui/models/_agent_tree.py` sets `start_time = min(member starts)` and
`stop_time = max(member stops)`, so any clan that spans an overnight approval gap is
mis-bucketed.

### Evidence from the artifact index for the screenshot

| clan      | container `start_time` (min member start) | container `stop_time` (max member stop) | L0 today  | L1 today    | L0 after fix |
| --------- | ----------------------------------------- | --------------------------------------- | --------- | ----------- | ------------ |
| `sase-h8` | 2026-08-07 18:09:14                       | 2026-08-08 10:56:18                     | This Week | `Sat Aug 8` | Yesterday    |
| `sase-ha` | 2026-08-07 20:47:40                       | 2026-08-08 10:06:24                     | This Week | `Sat Aug 8` | Yesterday    |
| `sase-hn` | 2026-08-08 13:08:02                       | 2026-08-09 00:10:49                     | Yesterday | `00:00`     | Today        |

(`sase-h8`'s displayed `4h37m` is the clan's aggregate wall clock, not `stop - start`,
which is why the overnight span is not obvious from the row.)

## Chosen fix, and why this direction

Bucket on the anchor, not on `start_time`. Reasons:

1. **The documented contract already says so.** `docs/ace.md` (Agents-tab `BY_DATE`
   section) states: "The time anchor is `stop_time` for terminal agents and `start_time`
   otherwise; both buckets and their subgroups sort newest-first." The code is what
   diverged.
2. **It matches the sibling implementation.** The Patches tab
   (`patch_groups/_buckets.py`) derives `date_bucket_for_patch()` _and_
   `date_subgroup_for_patch()` from the single `latest_patch_timestamp()` value, so its
   buckets and subgroups cannot disagree.
3. **It matches what the row shows.** A finished row displays its finish time; the
   bucket it sits in should be the day of that timestamp.
4. Anchoring the bucket on `start_time` instead (i.e. changing subgroups/sorting to
   start-anchored) would be the larger, more disruptive change and would contradict the
   established "recently-finished agents float to the top of their date segment"
   behavior that `walk_anchors()` documents and tests pin.

## Implementation

### 1. Single anchor for BY_DATE (`src/sase/ace/tui/models/agent_groups/_buckets.py`)

- Rename `hour_anchor_time()` → `date_anchor_time()`. The helper now anchors day and
  week granularity as well as hour windows, so the `hour_` prefix is misleading. It is
  package-internal (imported only by `_keys.py`; not in `agent_groups/__init__.py`'s
  `__all__` and not referenced by any test), so no alias is needed. Update its docstring
  to say it is _the_ BY_DATE anchor: bucket, subgroup label, and sort position.
- Move `date_anchor_time()` above `date_bucket_for()` so the file reads top-down.
- Rewrite `date_bucket_for()` to bucket on that anchor:

  ```python
  def date_bucket_for(agent: Agent, now: datetime) -> str:
      anchor = date_anchor_time(agent)
      if anchor is None:
          return "Earlier"
      today = now.date()
      anchor_date = anchor.date()
      if anchor_date == today:
          return "Today"
      if anchor_date == today - timedelta(days=1):
          return "Yesterday"
      if anchor_date > today - timedelta(days=7):
          return "This Week"
      return "Earlier"
  ```

- Update the two call sites in `src/sase/ace/tui/models/agent_groups/_keys.py`
  (`grouping_keys_for()`'s `anchor=` and `walk_anchors()`) plus the import.

Behavior deltas to keep in mind while editing:

- Non-terminal agents are unaffected: the anchor is still `start_time`.
- A terminal agent with no `stop_time` still falls back to `start_time`, so
  `test_build_agent_tree_by_date_done_without_stop_time_falls_back_to_start_time` stays
  green.
- A terminal agent with `stop_time` but no `start_time` now buckets on `stop_time`
  instead of falling into `Earlier` — an improvement, worth a test.
- `Earlier` remains the no-anchor bucket (`anchor is None`).
- Future-dated anchors (clock skew) keep falling through to `This Week`, exactly as
  today. Do not "fix" that here.

### 2. Docstrings that currently state the old rule

- `GroupingMode.BY_DATE` docstring in `_buckets.py`: "derived from each agent's
  `start_time`" → derived from the shared BY_DATE anchor.
- `date_subgroup_bucket_for()` and `date_subgroup_sort_key()` docstrings in
  `_buckets.py`: they reference `walk_anchors`; make them point at `date_anchor_time()`
  as the single source.
- `walk_anchors()` docstring in `_keys.py`: note that the same anchor decides the L0
  bucket, so bucket and subgroup cannot disagree.

### 3. Tests

`tests/ace/tui/models/test_agent_groups_grouping_mode_date.py` — the existing cases all
use the default `RUNNING` status, so they stay as-is. Add (`_NOW` is `2026-04-26 12:00`,
a Sunday, so `2026-04-25` is `Sat Apr 25` — the same shape as the screenshot):

1. `test_date_bucket_terminal_agent_uses_stop_time_not_start_time` — `status="DONE"`,
   `start_time=datetime(2026, 4, 24, 18, 9)`, `stop_time=datetime(2026, 4, 25, 10, 56)`
   → `"Yesterday"` (the `sase-h8` regression).
2. `test_date_bucket_terminal_agent_that_finished_today_is_today` — `status="DONE"`,
   `start_time=datetime(2026, 4, 25, 13, 8)`, `stop_time=datetime(2026, 4, 26, 0, 10)` →
   `"Today"` (the `sase-hn` regression).
3. `test_date_bucket_terminal_agent_without_stop_time_falls_back_to_start_time` —
   `status="DONE"`, `stop_time=None`, `start_time=datetime(2026, 4, 25, 15, 0)` →
   `"Yesterday"`.
4. `test_date_bucket_running_agent_still_uses_start_time` — `status="RUNNING"`,
   `start_time=datetime(2026, 4, 25, 23, 0)`, `stop_time=None` → `"Yesterday"`.
5. `test_date_bucket_terminal_agent_without_start_time_uses_stop_time` —
   `status="DONE"`, `start_time=None`, `stop_time=datetime(2026, 4, 26, 9, 0)` →
   `"Today"`.

`tests/ace/tui/models/test_agent_groups_grouping_mode_tree_date.py` — add the tree-level
regression that reproduces the screenshot, since the unit test alone would not catch a
future divergence between the two levels:

- `test_build_agent_tree_by_date_overnight_done_agent_buckets_with_its_subgroup`: build
  a tree (`mode=GroupingMode.BY_DATE`, `now=_NOW`) from an overnight `DONE` agent (start
  `2026-04-24 18:09`, stop `2026-04-25 10:56`) plus a same-day `DONE` agent (start
  `2026-04-25 20:00`, stop `2026-04-25 21:35`). Assert the L0 banners are exactly
  `[("Yesterday",)]` — no `("This Week",)` — and the L1 keys are
  `[("Yesterday", "21:00"), ("Yesterday", "10:00")]`, with no
  `("This Week", "Sat Apr 25")`. Use the `_group_keys(entries, level)` helper from
  `._agent_groups_helpers`.

Both files already import everything needed; `_agent()` in
`tests/ace/tui/models/_agent_groups_helpers.py` accepts `status` and `stop_time`.

### 4. Docs

`docs/ace.md`, Agents-tab `BY_DATE` paragraph (the one beginning "In `BY_DATE` mode, ACE
chooses one L1 subgroup style..."): keep the existing anchor sentence and make the
bucket half explicit — the same anchor selects the L0 date bucket, so an agent that
started Friday evening and finished Saturday morning renders under Saturday's bucket,
matching the finish timestamp on its row. Leave the Patches-tab `BY_DATE` table (which
describes `TIMESTAMPS`-based bucketing) untouched.

## Out of scope

- **No Rust core change.** Verified against the `sase-core` linked checkout: there is no
  date-bucket logic there (no `Yesterday` / `This Week` strings in any crate). BY_DATE
  grouping is presentation-only and lives entirely in Python for both the Agents and
  Patches tabs, so this stays on the Python side of the core boundary.
- **No BY_DATE row-patch guard.** `_display_panel_patches._try_patch_agent_row()` has a
  `BY_STATUS`-only `_status_row_patch_is_safe()` guard, and the fix makes an L0 bucket
  theoretically mutable when a row transitions to terminal. Its callers
  (`_unread_state`, `_approve`, `_loading_bead_warmup`, `_notification_*`) only mutate
  badge-level state on an already-loaded row and never rewrite `start_time` /
  `stop_time`; real status transitions arrive through the full refresh path. Adding a
  guard would force extra full-panel rebuilds, which `sase/memory/tui_perf.md` rule 6
  warns against, for no observable benefit.
- **Row timestamp vs anchor for plan-only rows.** `_row_runtime_terminal_time()` can
  return a `plan_times` / `questions_times` value for rows that have no `stop_time`,
  while `date_anchor_time()` falls back to `start_time` there. That is a separate,
  pre-existing, lower-severity divergence; it is not what the screenshot shows and this
  plan does not change it.
- **Stale fold state.** `~/.sase/ace_agents_fold_state.json` may hold keys such as
  `("This Week", "Sat Aug 8")` that no longer match any group. Unmatched keys are simply
  never consulted; no migration is needed.
- **PNG snapshots.** No visual golden under `tests/ace/tui/visual/` exercises BY_DATE
  grouping, so `just test-visual` is unaffected.

## Verification

```bash
just install
just test tests/ace/tui/models/test_agent_groups_grouping_mode_date.py \
          tests/ace/tui/models/test_agent_groups_grouping_mode_tree_date.py \
          tests/ace/tui/models/test_agent_groups_grouping_mode_subgroups.py \
          tests/test_timezone_runtime_consistency.py
just check
```

`tests/test_timezone_runtime_consistency.py::test_by_date_agent_grouping_reference_uses_configured_tz`
monkeypatches `_keys.date_bucket_for`, so keep that name and its `(agent, now)`
signature intact.

Manual confirmation (optional but cheap): open `sase ace`, press `o` until the header
reads `[group: by date]`, and confirm that every subgroup banner under `This Week` /
`Earlier` names a date that no `Today` / `Yesterday` bucket also covers.
