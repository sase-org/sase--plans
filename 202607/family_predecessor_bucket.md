---
tier: tale
title: Settle finished sequential-family predecessors by invariant, not by status
  allow-list
goal: A sequential family's non-final member stops claiming an in-flight bucket once
  it has finished, so the sase-ae clan chip reads [R1 D6] instead of [R2 D5], the
  sase-ae.6 FAMILY MEMBERS roster renders member 0 with the Done glyph, and the rule
  closes the whole class instead of one more status.
create_time: 2026-07-28 10:49:31
status: done
---

- **PROMPT:** [prompts/202607/family_predecessor_bucket.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/family_predecessor_bucket.md)
- **AGENTS:**
  - [bbugyi200.athena.mz--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mz.md#member-code)
- **COMMITS:**
  - [13e3b1d](https://github.com/sase-org/sase/commit/13e3b1ddc93a77b1fdc1aece5c26aecad554eae1) — fix: settle stopped family predecessors

# Settle finished sequential-family predecessors by invariant, not by status allow-list

## Symptom

On the ACE Agents tab the `sase-ae` clan row renders `[R2 D5]`. Its seven member rows are:

| member         | row status | correct bucket |
| -------------- | ---------- | -------------- |
| `sase-ae.land` | `RUNNING`  | Running        |
| `sase-ae.6`    | `DONE`     | Done           |
| `sase-ae.5`    | `DONE`     | Done           |
| `sase-ae.4`    | `DONE`     | Done           |
| `sase-ae.3`    | `DONE`     | Done           |
| `sase-ae.2`    | `DONE`     | Done           |
| `sase-ae.1`    | `DONE`     | Done           |

The chip should read `[R1 D6]`. The enclosing `@epic` panel title is correct, so the chip contradicts both the rows
underneath it and the panel above it — the same split as the 2026-07-27 defect fixed by
`202607/clan_row_running_count.md`.

A second symptom on the same data: selecting `sase-ae.6` renders its `FAMILY MEMBERS` roster as
`0 --plan · PLANNER · ▶ ANSWERED · 2m14s` — the ▶ Running glyph on a member that stopped at 09:32:11, inside a family
whose own row reads `DONE`.

## Root cause

`family_member_status_buckets()` in `src/sase/ace/tui/models/agent_family_members.py`:

```python
final_index = len(members) - 1
return tuple(
    "Done"
    if index < final_index and member.status in APPROVED_PLAN_STATUSES
    else status_bucket_for_values(member.status)
    for index, member in enumerate(members)
)
```

It settles a handed-off predecessor **only when that predecessor's status is literally `PLAN APPROVED` or
`TALE APPROVED`**.

`sase-ae.6` is a two-member sequential family: `--plan` (`ANSWERED`, stopped 09:32:11) then `--1` (`DONE`, stopped
09:49:28). `ANSWERED` is not on the allow-list, and `status_bucket_for_values()` deliberately maps it to `Running` ("the
transient post-answer state ... buckets with the actively-running rows"). So the family projects `("Running", "Done")`,
`aggregate_agent_group_bucket()` short-circuits on `"Running" in buckets`, and `clan_member_counts()` counts the entire
lane as running.

### Confirmed against live agent state

`sase.ace.tui.models.agent_loader.load_all_agents()` on 2026-07-28 reproduces the screenshot exactly:

```text
clan sase-ae: clan_member_counts   -> running=2, done=5   # the rendered chip
              agent_lane_status_counts -> running=1, done=6   # correct

member 'sase-ae.6' own='DONE' -> bucket 'Done'
    'sase-ae.6--plan' status='ANSWERED' bucket='Running' stop=2026-07-28 09:32:11.412852
    'sase-ae.6--1'    status='DONE'     bucket='Done'    stop=2026-07-28 09:49:28.823301
```

A sweep of every non-clan row in that tree found exactly one row whose re-aggregated bucket disagreed with the bucket of
its own displayed status: `sase-ae.6` (`own='DONE'` vs `agg='Running'`). A sweep of every loaded sequential family found
exactly one non-final member holding an in-flight bucket: `sase-ae.6--plan`. The defect is precisely this rule and
nothing else on the current tree.

## Why the 2026-07-27 fix did not cover it

`202607/clan_row_running_count.md` fixed the _plumbing_: `clan_member_counts()` now honors the effective bucket that the
projection computed rather than re-bucketing raw statuses. It left the _policy_ — which predecessors count as settled —
as a two-status allow-list. Every status that buckets as in-flight but is not on that list reproduces the same chip:

- `ANSWERED` — today's report.
- `RETRYING`, `EPIC APPROVED`, `PLAN COMMITTED`, `WORKING PLAN`, `WORKING TALE`.
- **Any status `status_bucket_for_values()` does not recognize.** Its last line is `return "Running"`, so a status added
  anywhere in SASE lands in the Running bucket by default and silently re-opens this bug.

An allow-list cannot close that class. The rule has to be structural.

## The structural invariant

`docs/agent_families.md`: "An agent family is a strictly sequential chain. ... If the parent is still running, the new
member appears immediately as WAITING and starts only after that exact parent artifact completes successfully."

So at most one member of a sequential family is ever executing. Any _other_ member showing an in-flight bucket is
displaying a sticky label, not live work.

The codebase already owns the trusted test for "actually executing" — `_is_active_root_mirror_candidate()` in
`src/sase/ace/tui/models/_agent_status_apply.py`, which decides the family root row's own visible status:

```python
if not agent_is_active(agent.status):
    return False
if agent.stop_time is not None:
    return False
```

That is exactly why the `sase-ae.6` **row** correctly reads `DONE` while the chip summarizing it reads Running: the row
asks `agent_is_active(...) and stop_time is None`; the bucket policy asks a status allow-list. Two ladders for one
question, so they drift.

Both halves of that predicate are load-bearing, and the live tree contains a witness for each:

| shape                                | live example                              | verdict           |
| ------------------------------------ | ----------------------------------------- | ----------------- |
| active status, `stop_time` set       | `sase-ae.6--plan` `ANSWERED`              | finished → settle |
| inactive status, `stop_time is None` | `my--plan` `TALE APPROVED`                | finished → settle |
| active status, `stop_time is None`   | pre-queued `%i(...)` member still running | executing → keep  |

The allow-list misses row 1; a `stop_time`-only rule would misclassify row 2 (`my--plan` is live right now with
`stop_time=None`) and would regress the 2026-07-27 fix; an unconditional "settle every non-final member" rule would
misclassify row 3 and count a genuinely running lane as waiting. Only the conjunction is correct.

## Fix

### 1. `src/sase/ace/tui/models/agent_family_members.py` — one shared in-flight predicate

Add `agent_is_active` to the existing `sase.agent.status_buckets` import and define:

```python
def agent_row_is_in_flight(agent: Agent) -> bool:
    """Return whether one row represents work that is still executing."""
    return agent_is_active(agent.status) and agent.stop_time is None
```

### 2. `src/sase/ace/tui/models/agent_family_members.py` — replace the allow-list with the invariant

```python
#: Buckets that claim an agent process is still executing.
_IN_FLIGHT_BUCKETS: frozenset[str] = frozenset({"Running", "Starting"})


def _settled_member_bucket(member: Agent) -> str:
    """Return the effective bucket for one non-final sequential-family member."""
    bucket = status_bucket_for_values(member.status)
    if bucket not in _IN_FLIGHT_BUCKETS or agent_row_is_in_flight(member):
        return bucket
    return "Done"


def family_member_status_buckets(members: Sequence[Agent]) -> tuple[str, ...]:
    """Return effective buckets for an ordered sequential family.

    A family advances one member at a time: a successor is attached only once
    its predecessor has finished, so a non-final member that is no longer
    executing has handed the work off and is settled.  Sticky handoff labels
    (``TALE APPROVED``), transient post-answer labels (``ANSWERED``), and every
    status that falls through ``status_bucket_for_values``'s ``Running``
    default would otherwise keep a finished predecessor counted as running.  A
    non-final member that *is* still executing keeps its bucket: a member
    attached to a running parent is created as ``WAITING``, so the running
    predecessor is the real state of the lane.  The final member always keeps
    the global bucket.
    """
    final_index = len(members) - 1
    return tuple(
        _settled_member_bucket(member)
        if index < final_index
        else status_bucket_for_values(member.status)
        for index, member in enumerate(members)
    )
```

The `APPROVED_PLAN_STATUSES` import becomes unused in this module — drop it from the import list. It stays defined and
used in `sase/agent/status_buckets.py`; see `sase/memory/symvision.md` if the unused-symbol lint complains.

Buckets that are deliberately **not** settled, because the sequential invariant does not refute them:

- `Failed` — a predecessor that failed is real terminal information the summary should keep.
- `Stopped` — a predecessor still holding an unanswered question or an unreviewed plan is actionable.
- `Waiting` / `Queued` — cannot occur in a well-formed chain; leaving it visible surfaces a scheduling anomaly rather
  than hiding it.

Only the in-flight claim is corrected, which is the only claim the invariant contradicts.

### 3. `src/sase/ace/tui/models/_agent_status_apply.py` — consume the shared predicate

```python
def _is_active_root_mirror_candidate(parent: Agent, agent: Agent) -> bool:
    """Return whether *agent* represents active work for root mirroring."""
    if not agent_row_is_in_flight(agent):
        return False
    if (
        agent is not parent
        and agent.raw_suffix is None
        and canonical_plan_chain_suffix(agent.role_suffix) == root_child_suffix(parent)
    ):
        return False
    return True
```

Behavior-preserving on its own; the point is that the row status and the bucket policy can no longer drift apart, which
is the mechanism that produced both this defect and the 2026-07-27 one.

Import direction is already safe: `_agent_status_apply` imports from `._agent_clan`, `._agent_clan` imports from
`.agent_family_members`, and `agent_family_members` imports only `.agent` and `sase.agent.status_buckets`. No cycle.

### Surfaces fixed by this one policy change

Everything that reads the sequential-family effective bucket:

- Agents-tab clan / family row chip — `src/sase/ace/tui/widgets/_agent_list_render_agent.py`
- Clan detail panel chip — `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`
- Cleanup clan modal chip — `src/sase/ace/tui/modals/agent_cleanup_clan_modal.py`
- `FAMILY MEMBERS` roster glyph — `src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py`
- Nested family children in the `CLAN MEMBERS` roster —
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`
- Family children in tribe panels — covered by `test_tribe_family_children_use_effective_status_glyphs`
- `agent_summary_status_counts` / `agent_lane_status_counts` wherever they project a sequential family through
  `concrete_agent_statuses`

The row render cache key already includes the computed counts, so corrected counts invalidate the cache on their own.

## Scope and non-goals

- **Do not** change `clan_member_counts()` or `aggregate_agent_group_bucket()`. Counting each direct member by
  `status_bucket_for_values(member.status)` instead of by its projection looks tempting and does fix this screenshot,
  but it was prototyped and it breaks
  `tests/test_agent_clan.py::test_clan_queue_count_projects_parallel_member_behind_family`: the projection expansion is
  load-bearing for legacy parallel families, where a directly constructed parallel root carries `WAITING` while its
  member carries `QUEUED`. The 2026-07-27 plumbing is correct; only the policy it consumes was wrong.
- **Do not** touch `_lane_summary_projections` / `agent_lane_status_counts`. That path is the reference the fix is
  validated against.
- `clan_roster_entries()` in `_agent_display_clan_roster.py` still builds its `family_status` from raw statuses. It was
  re-checked against today's live tree: its `family_rows()` projection yields the container row plus `--1`, never
  `--plan`, so no in-flight predecessor reaches that aggregate. Still latent, still out of scope; leave the note in
  place for a future cleanup.
- `agent_lane_status_counts` counts a _legacy parallel_ family as one row per member while `clan_member_counts` counts
  it as one lane (`research.h.final`: lane `D3` vs chip `D1`). That cardinality difference is intentional and unrelated;
  do not "fix" it here.
- No Rust change. `sase-core` has no counterpart to `status_buckets` or the family projection, so this stays
  presentation-side Python.

## Tests

### `tests/ace/tui/models/test_agent_family_members.py`

The module's `_agent()` helper does not set `stop_time`; add a `stop_offset: int | None = None` parameter that sets
`stop_time = _STARTED + timedelta(minutes=stop_offset)` when provided.

1. `test_stopped_non_final_family_member_projects_done` — **this is the screenshot.** `--plan` `ANSWERED` with a
   `stop_time`, `--1` `DONE` → `("Done", "Done")`. Fails before the fix with `("Running", "Done")`.
2. `test_unknown_status_on_stopped_non_final_member_projects_done` — predecessor carries a status the bucket map does
   not know (e.g. `"SOME NEW STATUS"`) with a `stop_time`; successor `DONE` → `("Done", "Done")`. Pins the
   `status_bucket_for_values` catch-all class, which is the reason an allow-list could never be enough.
3. `test_running_non_final_family_member_keeps_running_bucket` — `--0` `RUNNING` with `stop_time=None`, `--reviewer`
   `WAITING` → `("Running", "Waiting")`. Pins the documented pre-queued shape and guards against over-settling.
4. `test_failed_and_question_non_final_members_keep_their_buckets` — `FAILED` predecessor → `Failed`; `QUESTION`
   predecessor → `Stopped`. Pins the deliberate non-goals above.
5. `test_approved_non_final_family_member_projects_done` — keep unchanged (`TALE APPROVED`, no `stop_time` → `Done`).
   The generalized rule must subsume the rule it replaces; this test is the proof.
6. `test_approved_final_family_member_keeps_global_running_bucket` — keep unchanged.

### `tests/ace/tui/models/test_agent_summary_status_counts.py`

Add `test_clan_counts_settle_answered_family_planner_as_done`, mirroring the live shape: a clan container holding one
plain `RUNNING` member plus a family container row (own status `DONE`) whose `--plan` child is `ANSWERED` with a
`stop_time` and whose `--1` child is `DONE`.

Assert `clan_member_counts(container) == ClanStatusCounts(running=1, done=1)`, and assert it agrees with
`agent_lane_status_counts((container,), ())` on running/waiting/done — the cross-path agreement is the invariant the
screenshot violated.

Follow the fixture note from `202607/clan_row_running_count.md`: assert `clan_members(container)` returns exactly the
family container row plus the plain member _before_ asserting counts, otherwise the fixture leaks `--1` as a second
direct clan member.

### Existing tests

The whole suite was run against a monkeypatched prototype of exactly this policy: **22645 passed, 7 skipped**, with no
test edits. If an existing test needs editing, stop and re-check — that is a signal the change went wider than intended.

## Verification

1. `just install` first — workspace directories are ephemeral and dependencies may be stale.
2. Confirm the new `test_stopped_non_final_family_member_projects_done` fails before the change (`("Running", "Done")`)
   and passes after.
3. `just check`.
4. `just test-visual`. No golden should move — no visual fixture carries a stopped in-flight predecessor. If one does
   move, stop and re-examine rather than accepting it with `--sase-update-visual-snapshots`.
5. Live check: run `sase ace`, select the `sase-ae` clan row and confirm the chip reads `[R1 D6]` and reconciles with
   the `@epic` panel title; then select `sase-ae.6` and confirm `FAMILY MEMBERS` member `0 --plan` renders the Done
   glyph while still showing its `ANSWERED` status label.
