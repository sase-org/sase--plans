---
tier: tale
title: Stop leaf agent rows from rendering the queued count chip
goal: Short-hand agent-count chips render only on aggregate rows (tribes and clans);
  an individual globally queued agent row no longer shows a stray `[Q1]`.
create_time: 2026-07-25 08:27:26
status: wip
---

- **PROMPT:** [202607/prompts/leaf_row_queued_chip.md](prompts/leaf_row_queued_chip.md)

# Stop leaf agent rows from rendering the `[Q1]` count chip

## Problem

In the `sase ace` Agents tab, individual (leaf) agent rows that are waiting on the global runner cap render a short-hand
status-count chip:

```
sase (WAITING ▶10/10) [Q1] kf
sase (WAITING ▶10/10) [Q1] ke
sase (WAITING ▶10/10) [Q1] kd
```

Short-hand agent-count chips (`[R2 Q3 W3 D9]`) are only meant for aggregate rows — agent tribes (panel titles) and agent
clans (clan container rows). A single agent is not an aggregate of anything, so it must never render a chip. The row
already communicates its queue state through `(WAITING ▶10/10)`, so the chip is pure noise.

Reproduced with the current tree:

```python
agent = Agent(..., status="WAITING", pid=100, wait_runners=9,
              slot_requested_at="2026-07-25T07:00:00Z")
clan_member_counts(agent)
# ClanStatusCounts(awaiting=0, failed=0, running=0, queued=1, waiting=0, unread=0, done=0)
format_agent_option(agent, 0, is_selected=False, now=...)[0].plain
# '[agent] solo (WAITING) [Q1] kf'
```

## Root Cause

`clan_member_counts()` in `src/sase/ace/tui/models/_agent_clan.py` is documented as "Count a clan container's loaded
members by display bucket", and every bucket except one is derived by looping over `clan_members(agent)`. The `queued`
bucket breaks that contract:

```python
queued = sum(
    agent_is_globally_queued(status.agent)
    for status in agent_status_projections((agent,))
)
```

It projects **the row's own agent** rather than its members. For a real clan container this is harmless, because
`_summary_projections()` expands a container into its members. For a leaf row there is nothing to expand, so the
projection is the row itself and a globally queued leaf scores `queued=1`.

Every other bucket is `0` for a leaf, which is exactly why the leak stayed invisible until `queued` was introduced in
`ca348d703` ("feat(ace): show globally queued agent counts"). `format_agent_count_chip()` suppresses an all-zero chip,
so before that commit leaf rows silently rendered nothing; now the stray `queued` makes the chip non-empty.

The Agents-tab row renderer (`format_agent_option()` in `src/sase/ace/tui/widgets/_agent_list_render_agent.py`) relies
on that all-zero suppression instead of an explicit container check, so the bad count reaches the screen directly.

## Approach

Fix the count at its source: make `queued` respect the same member set as every other bucket in `clan_member_counts()`.
This is a one-expression change that removes the leak for all three call sites of `clan_member_counts()` at once
(Agents-tab rows, the prompt-panel CLAN section, and the clan cleanup modal) without touching the renderer.

Verified against the existing clan fixtures: the candidate expression returns `0` for a globally queued leaf while still
returning `1` for both clan shapes covered by current tests (a queued parallel member nested behind a family, and a flat
clan built by `project_clan_tree`).

### Not doing

Adding a `agent.is_clan_container or agent.agent_family_parallel` guard in `format_agent_option()` was considered and
rejected. It would silently change any family-container row whose loaded children happen to share the clan reference,
which is a behavior change beyond this bug, and it would mask (rather than fix) an aggregate helper that reports counts
for an agent that owns no members.

## Implementation

### 1. Count `queued` over clan members

In `src/sase/ace/tui/models/_agent_clan.py`, inside `clan_member_counts()`, project the clan's members instead of the
row agent:

```python
queued = sum(
    agent_is_globally_queued(status.agent)
    for status in agent_status_projections(clan_members(agent))
)
```

`clan_members(agent)` is already called at the top of the function for the bucket loop; reuse that tuple (hoist it into
a local) rather than recomputing it, so the function stays a single pass over the same member set.

Keep `agent_status_projections()` (not the raw member list) so nested containers — e.g. a parallel family sitting
between the clan container and the queued agent — still resolve to concrete rows.

### 2. Regression test for the leaf row

Add a test to `tests/ace/tui/widgets/test_agent_parallel_family_count_chips.py` (next to the existing
`test_non_family_and_zero_bucket_rows_omit_count_chip`) that builds a plain, non-clan, non-family agent which is
globally queued — `status="WAITING"`, `pid` set, `wait_runners` set, `wait_runners_explicit` false, and
`slot_requested_at` set — and asserts the rendered row contains no `[Q` chip.

Pass an explicit `now=` to `format_agent_option()`; the runtime suffix otherwise reaches config loading.

Also add a model-level assertion in `tests/test_agent_clan.py` that `clan_member_counts()` returns an all-zero
`ClanStatusCounts()` for that same globally queued leaf. This pins the contract at the layer that owns it, so a future
change to the renderer cannot resurrect the leak unnoticed.

### 3. Regenerate the affected PNG golden

`tests/ace/tui/visual/snapshots/png/agents_runner_slot_waits_120x40.png` currently bakes in the bug: its
`visual-global-cap` leaf row reads `(WAITING ▶0/10) [Q1] global-cap`. After the fix the ` [Q1]` disappears from that row
while the `@default · 2 [Q1 W2]` tribe header is unchanged.

Regenerate it with `just test-visual --sase-update-visual-snapshots` (or the equivalent pytest invocation for that
test), then confirm the accepted diff removes only the leaf-row chip.

`tests/ace/tui/visual/snapshots/png/agents_queued_clan_counts_120x40.png` must **not** change — its
`(WAITING) ×2 [Q1 W2] queue-demo` clan row and `[Q1 W2]` CLAN-section chip are the correct behavior this fix preserves.
Treat any diff there as a regression.

## Verification

1. New leaf-row test fails before the source change and passes after.
2. `tests/test_agent_clan.py::test_clan_queue_count_projects_parallel_member_behind_family` and
   `tests/ace/tui/models/test_agent_summary_status_counts.py::test_queue_counts_are_orthogonal_and_dedupe_container_flat_rows`
   still pass unchanged — both assert `queued == 1` for clan containers.
3. `tests/ace/tui/widgets/test_agent_parallel_family_count_chips.py` still asserts `[S1 R2 Q1 W1 F1 D1]` for the
   parallel-family root, i.e. container rows keep their `Q` metric.
4. `agents_queued_clan_counts_120x40.png` is byte-identical; `agents_runner_slot_waits_120x40.png` is regenerated.
5. `just install` then `just check` passes.

## Notes

- Run `just install` before `just check` — workspace directories are ephemeral and may hold stale dependencies. The
  current tree's installed `sase_core_rs` wheel is stale enough that `format_agent_option()` raises on
  `validate_agent_username`, so the install is not optional for running the widget tests.
- No Rust-core work is needed. This is presentation-layer count aggregation for the Textual Agents tab, which lives in
  the Python repo.
