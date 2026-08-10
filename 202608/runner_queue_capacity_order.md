---
tier: tale
title: Order the runner-slot queue display by admission eligibility
goal:
  The ACE QUEUE ladder, QUEUED row ranks, and agent-list queue positions rank waiters in
  the order they will actually be admitted, so agents parked behind an unmet
  %wait(runners=N) threshold sort after the waiters that can start now.
size: medium
proposed_by: bbugyi200.athena.x2
create_time: 2026-08-10 09:59:24
status: wip
---

# Plan: Order the runner-slot queue display by admission eligibility

## Problem

The ACE agent metadata panel's `QUEUE` ladder (and the `QUEUED #N/M` rank on agent rows,
and `Queue: #N of M` in the detail header) ranks runner-slot waiters by `priority` then
request-FIFO only. It ignores each waiter's `runners` threshold, so agents that cannot
possibly start at the current occupancy are displayed ahead of the agents that will
actually be admitted next.

Observed by the user with 6 waiters and a nearly-full pool:

```
❖ QUEUE · 6 waiting · 10/10 runners
   #1 audit_improvement…  ≤0 0s
   #2 audit_bugs.sase.7   ≤0 0s
   #3 ci_fix.sase.u       ≤0 0s
   #4 chop.refresh_docs…  ≤0 0s
   #5 x0                     0s
   #6 x1                     0s
```

`#1`–`#4` authored `%wait(runners=0)` (a drain barrier: start only when nothing is
running). They cannot be admitted until occupancy reaches 0. `x0` and `x1` carry the
implicit `max_running_agents - 1` threshold and are the two agents that actually start
next, yet they are shown at the back of the line.

## Root cause

Admission and display use the same sort key, but admission additionally filters by
threshold eligibility while display does not.

- `may_start()` (`src/sase/core/runner_slots/_admission.py:204`) admits the first waiter
  in queue order **whose threshold the current occupancy satisfies**; waiters that fail
  `running_count <= waiter.threshold` are skipped over entirely, no matter how long they
  have waited or how good their priority is.
- The display paths order waiters with the raw `runner_slot_waiter_sort_key()`
  (`src/sase/core/runner_slots/_admission.py:22`), which is
  `(priority, invalid, slot_requested_at, timestamp, artifact_dir)` and contains no
  threshold term:
  - `refresh_runner_slot_context()` /`_waiter_sort_key()`
    (`src/sase/ace/tui/models/agent_runner_slots.py:58` and `:167`) — feeds the TUI
    ladder, `QUEUED #N/M` row ranks, `Queue: #N of M`, clan/roster digests.
  - `_attach_runner_slot_context()` / `_runner_slot_waiter_sort_key()`
    (`src/sase/integrations/agent_list_entries.py:107` and `:149`) — feeds
    `sase agent list -j`, skills, and integration consumers.

So the queue is rendered in "who is polite" order rather than "who is next" order. This
is display-only: `may_start()` is already correct and admits `x0`/`x1` ahead of the
drain barriers. Nothing about actual admission behavior changes in this plan.

Secondary consequence in the same section: `runner_queue_selection()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py:45`) compensates for the
wrong order after the fact — it computes `ahead_count` by filtering earlier entries down
to those with `threshold >= selected_threshold`, and `_append_queue_entry()` paints
skipped-over earlier entries in a "parked" amethyst accent. That is why the screenshot
correctly reports `1 ahead` for `x1` while showing it at `#6 of 6`. Once the ladder is
ordered correctly, that compensation is both wrong (it compares thresholds instead of
eligibility) and unnecessary.

## Design decision: partition by current eligibility, not by static threshold

**Chosen rule** for display order:

1. **Admissible waiters first** (`running_count <= threshold`), in the existing
   `priority` → request-FIFO order — unchanged and exactly what `may_start()` picks
   from.
2. **Parked waiters after them** (`running_count > threshold`), ordered by threshold
   descending (the barrier that opens soonest first), then the same `priority` → FIFO
   key among equal thresholds.

This makes the ladder a faithful projection of `may_start()`. It also matches what
`docs/xprompt.md:1882` already documents about admission: "Among waiters eligible at the
current running count, the lowest numeric `priority=N` starts first, with FIFO ordering
among equal priorities."

**How this relates to the user's stated rule.** The user asked that agents with a lower
`runners` value always sort after agents with a higher one. Under the chosen rule that
holds in every situation where it is observable: whenever a low-`runners` waiter is
blocked by occupancy it sorts behind every waiter that is not. The one divergence is at
low occupancy — with `running_count = 0`, a `%wait(runners=0)` waiter _is_ admissible
and can legitimately be admitted before a later-arriving default waiter, so the ladder
shows it in FIFO position. Ranking it last there would be a lie about what starts next,
and it would contradict the existing regression test
`test_drain_waiter_joins_fifo_order_when_running_count_reaches_zero`
(`tests/ace/tui/test_agent_runner_slots.py:147`), whose name states the intended
behavior.

**Rejected alternative:** a static "threshold descending, then priority, then FIFO"
order that ignores occupancy. It is simpler and never reshuffles, but it misreports the
drained-pool case above, and it would need the existing drain-barrier FIFO test deleted
rather than kept. If the user prefers it at review time, it is a one-line change to the
shared key added in step 1 (drop the `parked` partition and always use the `-threshold`
term).

**Missing thresholds.** `wait_runners` is `int | None` on the display models. Treat
`None` as `0`, consistent with `_threshold()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py:242`) and with
`live_runner_slot_waiters()` (`src/sase/core/runner_slots/_admission.py:192`), which
already defaults a non-int marker value to `0`.

**Rust core boundary.** `crates/sase_core` carries only the `wait_runners` /
`wait_runners_explicit` wire fields (`src/agent_scan/wire.rs:359`); runner-slot
admission and queue ordering are not implemented there. The shared key belongs next to
the existing Python admission code in `src/sase/core/runner_slots/`. Do not port
runner-slot admission to Rust as part of this change.

## Implementation

### 1. Add the shared display-order key

In `src/sase/core/runner_slots/_admission.py`, next to `runner_slot_waiter_sort_key()`:

```python
def runner_slot_queue_display_key(
    *,
    running_count: int,
    threshold: int | None,
    priority: object,
    slot_requested_at: str | None,
    timestamp: str | None,
    artifact_dir: str | None,
) -> tuple[int, int, int, int, datetime, str, str]:
    """Return the capacity-aware presentation order for one slot waiter.

    ``may_start`` skips waiters the current occupancy cannot satisfy, so a
    parked waiter never starts before an admissible one however long it has
    waited. Ranking a display queue by the raw admission key therefore puts
    drain barriers ahead of the agents that actually start next. Admissible
    waiters keep the exact admission key; parked ones follow, ordered by how
    close their barrier is to opening.
    """
    effective = threshold if threshold is not None else 0
    parked = running_count > effective
    return (
        1 if parked else 0,
        -effective if parked else 0,
        *runner_slot_waiter_sort_key(
            priority=priority,
            slot_requested_at=slot_requested_at,
            timestamp=timestamp,
            artifact_dir=artifact_dir,
        ),
    )
```

Export it from `src/sase/core/runner_slots/__init__.py` (import list and `__all__`).
Leave `runner_slot_waiter_sort_key()`, `live_runner_slot_waiters()`, and `may_start()`
untouched — real admission keeps its current key.

### 2. Order the TUI queue by the new key

`src/sase/ace/tui/models/agent_runner_slots.py`:

- `refresh_runner_slot_context()` already computes `running_count` (line 57) before
  sorting waiters (line 58). Pass it into the sort: replace `key=_waiter_sort_key` with
  a `running_count`-bound key, and change `_waiter_sort_key()` (line 167) to take
  `running_count` and call `runner_slot_queue_display_key()`.
- Add a `parked: bool = False` field to `RunnerQueueEntry` (line 21) and populate it
  when building entries (`running_count > (agent.wait_runners or 0)`, written with an
  explicit `is None` check). The renderer needs the flag and must not recompute it from
  `RunnerCapacitySnapshot.slots_in_use`, which is `0` on the neutral
  `effective_limit is None` fallback path even though the queue was ordered with the
  real count.
- Update the `RunnerQueueEntry` docstring ("canonical admission order") to say the order
  is capacity-aware.

### 3. Order the integrations queue identically

`src/sase/integrations/agent_list_entries.py`: `_attach_runner_slot_context()` already
receives `runner_slots_in_use`. Thread it into `_runner_slot_waiter_sort_key()`
(line 149) and call `runner_slot_queue_display_key()` there too, so
`sase agent list -j`, the `sase_agents_status` skill, and integration consumers agree
with the TUI. `tests/test_agent_list_entries.py:320` already asserts TUI/integrations
rank parity, so these two must move together.

Also refresh the stale comment at `src/sase/integrations/_agent_list_entry_models.py:31`
("by priority and request FIFO") and at `src/sase/ace/tui/models/_agent_state.py:230`.

### 4. Simplify the queue section now that order is correct

`src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py`:

- `runner_queue_selection()` (line 45): every entry before the selected index is now
  genuinely ahead of it, so `ahead_count` becomes `selected_index`. Delete the
  `selected_threshold` filter (lines 63–67) and the module-private `_threshold()` helper
  if nothing else needs it. Keep the `Queue: … · N ahead / at the front` line in
  `_agent_display_header_metadata.py:239` as-is — "at the front" stays a useful
  affordance for `#1`.
- `_append_queue_entry()` (line 177): drop the `selected_threshold` parameter and key
  the accent off `entry.parked` instead of a threshold comparison:
  - selected row → `bold black on {QUEUED_STATUS_COLOR}` (unchanged),
  - `entry.parked` → `_PARKED_COLOR` rank and `dim {_AGENT_NAME_ANNOTATION_STYLE}` name,
    wherever it sits in the ladder,
  - otherwise ahead of the selection → `QUEUED_STATUS_COLOR`,
  - otherwise behind the selection → `dim`. The threshold column still prints `≤N` for
    explicit thresholds only, unchanged.
- Heading (lines 88–108): when any entry is parked, add a `· N parked` segment in
  `_PARKED_COLOR` between the waiting count and the runners fraction, so the reordering
  is legible: `❖ QUEUE · 6 waiting · 4 parked · 10/10 runners`. Omit the segment when
  nothing is parked.

Note `build_wait_lanes()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py:139`) declares a
`runner_queue_ahead_count` parameter it never reads. Dropping that dead parameter and
its two call sites is a small, in-scope cleanup while this code is open; skip it if
symvision or a caller makes it awkward.

### 5. Tests

Update the assertions that encode the old order and add regression coverage:

- `tests/test_runner_slots.py` — new unit tests for `runner_slot_queue_display_key()`:
  parked waiter sorts after an admissible one regardless of priority; two parked waiters
  order by threshold descending; two admissible waiters keep priority-then-FIFO;
  `threshold=None` behaves as `0`; at `running_count = 0` nothing is parked and the key
  degrades to the admission key.
- `tests/ace/tui/test_agent_runner_slots.py` —
  `test_refresh_runner_slot_context_ranks_all_waiters_while_pool_is_full` (line 45)
  currently asserts the `≤0`/`priority=1` waiter at `#1` with one runner live; it must
  now expect the threshold-9 waiter at `#1`. Keep
  `test_drain_waiter_joins_fifo_order_when_running_count_reaches_zero` (line 147)
  passing unchanged — it is the guard for the drained-pool case. Add a case reproducing
  the screenshot: four `≤0` explicit waiters plus two implicit waiters with the pool
  full, asserting the two implicit waiters take `#1`/`#2` and that
  `RunnerQueueEntry.parked` is set on exactly the four barriers.
- `tests/ace/tui/widgets/test_agent_queue_section.py` —
  `test_ahead_count_uses_selected_threshold_and_treats_missing_as_zero` (line 108) is
  replaced by a test that `ahead_count` equals the selected index;
  `test_queue_ladder_demotes_deeper_barrier_and_marks_selected_rank` (line 164) keeps
  its style assertions but with the barrier at the back of the ladder; add a heading
  test for the `N parked` segment.
- `tests/test_agent_list_entries.py` — the parity test at line 320 must still pass; add
  or extend a case with a mixed-threshold queue under a full pool
  (`test_runner_slot_wait_info_includes_live_count_and_queue_position`, line 133,
  encodes the old order and needs updating).
- `tests/ace/tui/visual/` — the queue-ladder fixtures
  (`_ace_agents_png_snapshot_fixtures.py:287` and `:370`) contain drain barriers, so PNG
  goldens may shift. Run `just test-visual`; if the only differences are the expected
  rank/accent changes, accept them with `--sase-update-visual-snapshots` and eyeball the
  diff artifacts under `.pytest_cache/sase-visual/` first.

### 6. Docs

Update the prose that states the old ordering:

- `docs/troubleshooting/runner-slots.md:16-25` — the "Admission sorts waiters by…"
  paragraph, and `:33-43` — the `QUEUE` ladder paragraph, which currently documents the
  `threshold >= selected` definition of `N ahead` and describes the parked accent as
  applying to _earlier_ entries.
- `docs/integrations.md:121-125` and `docs/cli.md:56-60` — both describe
  `runner_slot_queue_position` as priority-then-FIFO.
- `docs/ace.md:3346` (`queue #N of M` admission rank) — check and adjust wording only if
  it now reads as inaccurate.
- `src/sase/xprompts/skills/sase_agents_status.md:26-29` — "in priority-then-request-
  FIFO admission order". This is a **skill source template**: edit the template only.
  Per the generated-skills workflow, deployment (`sase skill init --force`) happens from
  a clean, merged tree after this lands and is **not** part of this change.

## Verification

```bash
just install
just check
```

Then, before landing:

```bash
just check-full
just test-visual
```

Manual confirmation: with a full or near-full pool and at least one live
`%wait(runners=0)` agent, open `sase ace`, select a queued agent, and confirm the
`QUEUE` ladder lists the implicit-cap waiters first, the drain barriers last in amethyst
with their `≤0` marker, and that `Queue: #N of M · K ahead` agrees with the ladder
position. The `QUEUED #N/M` ranks on the agent rows must match the ladder.

## Out of scope

- **Admission behavior.** `may_start()`, `live_runner_slot_waiters()`,
  `runner_slot_waiter_sort_key()`, and the deference window keep their current
  semantics. This change only makes the display agree with them.
- **The banner-vs-panel count difference.** In the screenshot the header reads
  `9/10 running` while the `QUEUE` heading reads `10/10 runners`. Those are two
  deliberately different quantities — the header's `running` is a visible-status count
  passed from `_display_detail_info.py:162`, while the ladder's numerator is
  `runner_slots_in_use` from `_holds_runner_slot()`. Do not change either here; if the
  user wants them reconciled, file it separately with `/sase_new_task`.
- Porting runner-slot admission into `sase-core` (see the boundary note above).
