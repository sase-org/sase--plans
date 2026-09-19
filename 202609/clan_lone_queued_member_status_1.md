---
tier: tale
title: Clan row mirrors its lone queued member's queue rank
goal:
  "When a clan's aggregate status is QUEUED and exactly one direct member is queued, the
  clan Agents-tab row and CLAN detail Status line show that member's admission rank
  (QUEUED #3/4) the same way the member row does, while the independent [Q… D…] count
  chip is unchanged."
size: small
proposed_by: bbugyi200.athena.0no
create_time: 2026-09-19 08:09:47
status: wip
---

# Clan row mirrors its lone queued member's queue rank

## Problem

Clan `sase-133` has three done families and one live member, `sase-133.land`, whose
Agents-tab row reads `QUEUED #3/4` (runner-slot admission rank 3 of 4). The synthetic
clan container still reads `QUEUED [Q1 D4]` on both surfaces:

- the Agents-tab clan row parenthetical
- the CLAN detail header `Status:` line

The `[Q1 D4]` count chip is correct (one queued direct member, four done). The missing
piece is the admission rank `#3/4` that the only queued member already has. A sequential
family whose next shell is queued already mirrors that shell's `QUEUED #N/M` on the
family row; a clan with exactly one queued direct member should do the same.

This extends the already-landed tale `plan:202609/clan_lone_running_member_status.md`,
which taught the clan row to mirror a lone running member's refined label (`TESTING`,
`TESTED`, …). Queued rank is a separate attachment on the same helper.

## Root cause

`apply_clan_container_status` in `src/sase/ace/tui/models/_agent_clan.py` already
aggregates to `QUEUED` when any direct member is queued and nothing higher-priority is
present. It also mirrors a refined label (`TESTING`, `TESTED`, …) when exactly one
member is outside the queued/waiting/done buckets. Queued members are intentionally
**ignored** for that mirror (`_CLAN_IGNORED_MEMBER_STATUS_BUCKETS` includes
`QUEUED_STATUS_BUCKET`) so a queued companion cannot block a lone `TESTING`/`TESTED`
source. That is still the right rule for Failed/Running/Stopped labels.

What it does **not** do is attach the queued member's wait-display row to the clan.
Admission rank is not part of `status`. `append_agent_row_status` in
`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` draws `#3/4`, `pN`, and
`held by …` from `wait_display_agent(agent).runner_slot_queue_position` /
`runner_slot_queue_size`. Sequential families get that because `_agent_status_apply.py`
sets `parent.wait_display_source` to the next queued shell. Clan containers never set
`wait_display_source`, and the CLAN detail Status line in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py` prints only
`display_status` plus the count chip.

Do **not** add `Queued` to `_CLAN_INHERITABLE_STATUS_BUCKETS` or remove it from
`_CLAN_IGNORED_MEMBER_STATUS_BUCKETS`. That would make a queued companion a competing
"relevant" member and break the existing lone-`TESTED`/`TESTING` rule
(`test_clan_queued_and_waiting_companions_do_not_mask_lone_failed_label`).

This is presentation-only projection of already-loaded TUI rows. Queue positions still
come from the existing runner-slot snapshot. No sase-core change. No feature flag.

## Rule

Keep the current aggregate and Failed/Running/Stopped mirror unchanged. Add a second,
independent attachment that runs on every `apply_clan_container_status` call (including
the Failed/Running/Stopped early path, so a later reprojection cannot leave a stale
pointer):

1. Deduplicate direct members by `identity` as today.
2. If the aggregate bucket is `Queued` **and** exactly one unique member has effective
   bucket `Queued`, set `container.wait_display_source` to
   `wait_display_agent(that_member)` so a sequential-family member yields the queued
   shell (whose `runner_slot_queue_position` is populated) rather than the family root
   (which often has none).
3. Otherwise set `container.wait_display_source = None`.
4. Do not copy `runner_slot_queue_position` / `runner_slot_queue_size` onto the clan
   row. Renderers already go through `wait_display_agent`. Copying those fields inside
   `apply_clan_container_status` races the runner-slot loop: the clan may be visited
   before the member's position is written. The pointer is enough; by paint time the
   member has its rank. This matches sequential families
   (`test_first_refresh_promotes_sequential_family_root_from_slot_waiter` asserts
   `root.wait_display_source is child`, not a position on the root).
5. Count chips, `BY_STATUS` grouping, clan-member ordering, and summary counts stay on
   `clan_member_counts` / the aggregate bucket. A lone queued member therefore reads
   `QUEUED #3/4 [Q1 D4]`, not `QUEUED #3/4` with the chip dropped. Two queued members
   keep generic `QUEUED [Q2]` with no rank (the visual fixture
   `agents_queued_clan_counts_120x40` is this case).
6. Waiting companions do not block the rank. One queued member plus waiters still
   attaches that member, so `QUEUED #3/4 [Q1 W2]` is allowed. Failed/Running/Stopped
   still outrank queued work, and those mirrors still ignore queued members.

Runner-slot refresh already re-calls `apply_clan_container_status` after it promotes
waiters to `QUEUED` (`_refresh_runner_slot_context_fallback` and
`_apply_runner_capacity_snapshot` in `src/sase/ace/tui/models/agent_runner_slots.py`).
No extra call site. Tree projection (`_container_for_clan`) already calls the helper
before that refresh; the first pass may see `WAITING` members, then the slot pass
promotes and re-attaches. Idempotent re-application must clear the pointer when a second
member becomes queued or the lone waiter starts.

## Changes

### 1. `apply_clan_container_status` in `src/sase/ace/tui/models/_agent_clan.py`

- Stop returning early after the Failed/Running/Stopped mirror. After that branch (or
  the generic aggregate/fallback branch) call a private helper, for example
  `_set_clan_queued_wait_display_source(container, unique_members, aggregate_bucket)`,
  that implements rule 2–3.
- Import `wait_display_agent` from `sase.ace.tui.models.agent_time` (or `.agent`) rather
  than inlining, so family-root dereference stays in one place. Avoid a cycle through
  `agent.py` if one appears; `agent_time` does not import `_agent_clan`.
- Extend the `apply_clan_container_status` docstring: queued rank is attached via
  `wait_display_source` when the aggregate is Queued and exactly one unique member is
  queued; queued members remain ignored for the Failed/Running/Stopped label mirror.
- Leave `aggregate_clan_status` and `_CLAN_INHERITABLE_STATUS_BUCKETS` /
  `_CLAN_IGNORED_MEMBER_STATUS_BUCKETS` unchanged.

### 2. Shared `#N/M` extras in the list-row status helper

In `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`, extract the queued
extras currently inlined under `elif agent.status == QUEUED_STATUS` (admission rank,
legacy `▶occupied→threshold` when capacity-budget display is off, explicit `pN`,
`held by <armer>`) into a public helper, for example
`append_queued_status_extras(text, agent) -> None`, that reads through
`wait_display_agent`. Keep styles identical (`#N/M` in `QUEUED_STATUS_COLOR`, dim
priority/held-by). Call it from `append_agent_row_status`.

Do not start drawing `cN`/`wN` on clan rows. `append_agent_queue_badges` already skips
clan containers; leave that.

### 3. CLAN detail Status line

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`, after appending
`display_status` and before the count chip, if `agent.status == QUEUED_STATUS` call the
same `append_queued_status_extras` helper. Import `QUEUED_STATUS` from
`sase.agent.status_buckets`. Result for the screenshot case:

`Status: QUEUED #3/4 [Q1 D4]`

A separate `Queue:` block on the clan header is out of scope.

### 4. Docs: `docs/ace.md`

In the "Clan rows aggregate member status…" paragraph (about line 2110), add one or two
sentences after the existing lone-member label rule: when the aggregate is `QUEUED` and
exactly one direct member is queued, the clan row and CLAN `Status:` line also show that
member's admission rank (`QUEUED #3/4`), plus the same `pN` / `held by` extras the
member row shows. Two queued members stay generic `QUEUED` with no rank. The count chip
is still independent (`QUEUED #3/4 [Q1 D4]`). Near the Queued-row paragraph (about line
2296), mention clans next to sequential families.

## Tests

### `tests/ace/tui/models/test_agent_tree_clan_status.py`

Add cases with `project_clan_tree` (same helpers as the existing file):

1. **Screenshot case.** Members: three `DONE` plus one `QUEUED` with
   `runner_slot_queue_position=3`, `runner_slot_queue_size=4`. Expect
   `container.status == "QUEUED"`, `container.wait_display_source` is the queued member,
   `format_agent_option(container, …)` contains `(QUEUED #3/4)` and `[Q1 D3]`, and the
   style at `#3/4` matches the member row.
2. **Two queued members** (positions 2 and 3) give `QUEUED`,
   `wait_display_source is None`, and no `#` rank on the clan row. Count chip `[Q2]`.
3. **Queued plus waiting companions** still attach the unique queued member and show
   `#N/M` plus `[Q1 W…]`.
4. **Queued companions do not mask lone `TESTED`.** Existing
   `test_clan_queued_and_waiting_companions_do_not_mask_lone_failed_label` must keep
   passing, and `wait_display_source` must be `None` on that container.
5. **Idempotent drop.** After case 1, set the queued member to `DONE` and reproject:
   `wait_display_source is None` and the rank disappears.
6. **Family member.** Lone queued sequential-family root whose `wait_display_source` is
   its queued shell: the clan pointer is that shell (via `wait_display_agent`), and the
   clan row shows the shell's `#N/M`.

### `tests/ace/tui/widgets/test_agent_display_clan.py`

- Screenshot header: lone queued member with position 3/4 →
  `"Status: QUEUED #3/4 [Q1 D3]\n"` (or `[Q1 D4]` if the fixture has four done members).
  Rank color matches the list-row extras.
- Existing `test_clan_header_queue_count_excludes_explicit_and_dependency_waits` stays
  `Status: QUEUED [Q1 W2]` unless that fixture grows a position; do not add a rank when
  `runner_slot_queue_position` is `None`.

### `tests/ace/tui/test_agent_runner_slots_families.py`

- After `refresh_runner_slot_context` on a clan with one slot waiter and one `DONE`
  member, the container is `QUEUED`, `wait_display_source` is the waiter, and
  `format_agent_option` includes that waiter's `#N/M`. Cover both `effective_limit=None`
  and `effective_limit=10`.
- `test_first_refresh_promotes_all_slot_waiters_and_clan_aggregate` (two waiters) must
  still read `QUEUED` and now also `wait_display_source is None`.

### Render cache

In `tests/ace/tui/widgets/test_agent_render_cache_clan.py`, once `wait_display_source`
is set, changing the source member's `runner_slot_queue_position` must invalidate the
cached clan row (the cache key already includes
`wait_display_agent(…).runner_slot_queue_position`).

### Visual snapshots

`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py` / `queued_clan_agents()`
is two waiters: keep asserting `(QUEUED) ×2 [Q2]` and `Status: QUEUED [Q2]` with **no**
`#N/M`. If a targeted `just fix-tui-screenshots -- agents_queued_clan_counts` rewrites
that golden, inspect the report; a rank appearing there is a bug in the two-queued rule.
Do not add a new PNG for the one-queued case; unit tests cover the strings.

## Verification

- `just install` if the workspace virtualenv is stale.
- `just fmt`.
- `just check`: all lint gates plus the diff-scoped test lane. This touches shared TUI
  projection and Agents-tab rendering; if the scoped run escalates or looks unusual, run
  `just check-full` through `/sase_monitor` with `TESTING`/`TESTED`.
- If any existing PNG golden other than the two-waiter clan snapshot shifts, run
  targeted `just fix-tui-screenshots` and inspect every update group before treating
  generation as approval.

## Out of scope

- A `Queue:` / wait-lane section on the CLAN detail header.
- Drawing `cN`/`wN` badges on clan rows.
- Inheriting a lone `WAITING` member's countdown or bead summary onto the clan.
- Changing count-chip contents, `BY_STATUS` grouping, or clan-member sort.
- Completion-candidate / CLI / mobile clan status strings.
- sase-core runner-slot math (positions are already correct on the member).
