---
tier: tale
title: Make runner-slot priority outrank capacity across threshold groups
goal:
  A better-priority runner-slot waiter wins over a worse-priority one even when its own
  threshold currently parks it, through bounded deference at the admission gate, and ACE
  and `sase agent list -j` rank the queue by priority first.
size: medium
proposed_by: bbugyi200.athena.zj
create_time: 2026-09-09 20:00:27
status: wip
---

# Plan: Make runner-slot priority outrank capacity across threshold groups

## Observed symptom

ACE shows two live runner-slot waiters while four agents hold slots:

```
❖ QUEUE · 2 waiting · 2 parked · 4/1 runners
   #1 toobig-2j.split_f… ≤3 p20 0s
  [#2] zh                ≤0 p1  0s
```

`zh` authored `%w(runners=0, priority=1)` — the best priority in the queue and an
explicit drain barrier — yet it ranks **behind** `toobig-2j…`, which authored the worst
priority in the queue (`p20`) and a looser threshold (`≤3`). The ranking is not a
rendering accident: it faithfully reports the admission order that will actually happen.
Both the display and admission are wrong in the same way, and the display is the
symptom, not the defect.

## Root cause

`%wait(priority=N)` only orders waiters **inside one threshold-eligibility group**. It
has no effect across groups, in any of the three places that matter:

1. **Admission ordering** — `may_start()` in
   `src/sase/core/runner_slots/_admission.py:228` picks the first waiter that is
   _eligible at the current running count_ and skips threshold-ineligible waiters
   entirely. At `running_count == 3`, `zh` (threshold `0`) is skipped and `toobig-2j…`
   (threshold `3`) is admitted, even though `zh` holds a strictly better priority.
   `toobig-2j…` then re-occupies the slot, so `zh`'s drain condition is never reached.

2. **Deference** — `better_priority_agent_pending()` at
   `src/sase/core/runner_slots/_admission.py:149` is the mechanism that exists precisely
   to let a better-priority agent win a freed slot, but it explicitly **skips any record
   that has already published a queue marker** (`waiting.slot_requested_at`, line 166).
   Its stated rationale (docs/troubleshooting/runner-slots.md:44-46, "the sort above
   only compares waiters already parked at that instant") assumes the `may_start()` sort
   already covers every queued waiter — which is exactly what point 1 shows it does not
   do for threshold-parked waiters. So a queued, threshold-parked, better-priority
   waiter is covered by **neither** mechanism. `toobig-2j…` at `p20` therefore claims
   the freed slot with _zero_ deference despite `zh` sitting at `p1` right next to it.
   `tests/test_runner_slots.py:141-151` pins this exclusion (`/parked` ⇒ `False`).

3. **Display ordering** — `runner_slot_queue_display_key()` at
   `src/sase/core/runner_slots/_admission.py:49` sorts by
   `(parked, -threshold, priority, fifo…)`, so capacity proximity outranks priority.
   Introduced deliberately in `fb7b8366e` ("order queue display by capacity") to make
   the ladder predict admission — and it does predict it correctly, because of points 1
   and 2. Fixing this alone would only move the lie: `zh` would render `#1` while
   `toobig-2j…` still started first.

Everything else in the screenshot is consistent and correct: `▶4→0 p1` on the row, the
`≤0 (drain barrier)` detail line, `2 parked`, and `4/1 runners` all match the markers.

## Fix

Give `priority` authority across threshold groups, then let the display follow.

**Semantic change (one sentence):** a waiter defers, for a bounded window, to any live
better-priority waiter — _including one currently parked by its own threshold_ — and the
window scales with the priority gap between them.

This is additive. It bites only when one waiter authored a strictly better priority than
another. Equal-priority waiters (the overwhelmingly common case: nobody sets `priority`)
behave exactly as today, so the documented drain-barrier contract — "a barrier is a
drain condition, not an exclusive fence" — is preserved for equal priorities, and both
existing end-to-end contracts stay green unchanged:

- `tests/fakey/test_runner_slots_e2e.py:302`
  `test_fakey_drain_barrier_waits_for_later_eligible_launch` (barrier and later launch
  are both default priority ⇒ no deference ⇒ `["running", "later", "barrier"]` still
  holds).
- `tests/fakey/test_runner_slots_e2e.py:345`
  `test_fakey_priority_admission_differs_from_park_order` (both default threshold ⇒
  unchanged).

`may_start()` itself is **not** changed. A parked better-priority waiter must not become
a hard fence: agents routinely launch other top-level agents and wait on them, so
head-of-line blocking on an unsatisfiable threshold could deadlock the host. Bounded
deference gives priority real teeth without that risk.

### Step 1 — generalize the deference window

`src/sase/core/runner_slots/_admission.py`

Change `deference_window_seconds()` to measure the gap to the better-priority waiter
rather than the distance from the default:

```python
def deference_window_seconds(
    priority: int,
    *,
    seconds_per_step: int,
    max_seconds: int,
    better_priority: int = DEFAULT_WAIT_PRIORITY,
) -> float:
    gap = priority - better_priority
    if gap <= 0:
        return 0.0
    return float(min(gap * seconds_per_step, max_seconds))
```

The existing formula is the special case `better_priority == DEFAULT_WAIT_PRIORITY`, so
every current call and every assertion in `tests/test_runner_slots.py:86`
(`test_deference_window_scales_only_worse_priorities_and_clamps`) keeps its meaning.

### Step 2 — count parked queued waiters as better-priority evidence

`src/sase/core/runner_slots/_admission.py`

Replace `better_priority_agent_pending()` (bool) with a function returning the best
better priority found, and add a second one for the queue. Both return `None` when no
better-priority candidate exists.

```python
def best_pending_better_priority(
    records: Iterable[AgentArtifactRecordWire],
    is_live: RecordLiveness,
    *,
    priority: int,
    me: str,
) -> int | None:
    """Best priority among live agents that could still join the queue."""
    # Same filters as today's better_priority_agent_pending: live, slot-participating,
    # not started, no slot marker yet, not me, strictly better priority. Return the
    # minimum matching priority instead of a bool.


def best_parked_better_priority(
    queue: Iterable[RunnerSlotWaiter],
    *,
    running_count: int,
    priority: int,
    me: str,
) -> int | None:
    """Best priority among queued waiters currently parked by their threshold."""
    # A waiter is parked when running_count > waiter.threshold. Skip me, skip
    # equal-or-worse priorities. Eligible waiters are intentionally excluded: may_start()
    # already blocks on those, so they can never reach the deference branch.
```

Update the `__init__.py` re-export list and `__all__` accordingly (drop
`better_priority_agent_pending`, add both new names).

Both helpers must exclude `me` and use `>` / `<` strictly, so equal priorities never
defer to each other. Deference therefore follows a strict partial order and cannot form
a cycle — two waiters can never defer to one another.

### Step 3 — consult both sources at the admission gate

`src/sase/axe/run_agent_wait_slots.py`, inside `_try_claim_runner_slot()` (the
`if eligible:` branch, currently lines 240-278). Replace the
`priority <= DEFAULT_WAIT_PRIORITY` fast path with:

```python
if eligible:
    candidates = [
        best_parked_better_priority(
            queue,
            running_count=running_count,
            priority=priority,
            me=artifacts_dir,
        )
    ]
    if priority > DEFAULT_WAIT_PRIORITY:
        candidates.append(
            best_pending_better_priority(
                records, is_live, priority=priority, me=artifacts_dir
            )
        )
    better = min((value for value in candidates if value is not None), default=None)
    if better is None:
        run_started_at = claim()
        remove_waiting_marker(artifacts_dir)
        return run_started_at, False
    deference_window = deference_window_seconds(
        priority,
        seconds_per_step=get_runner_slot_deference_seconds_per_step(),
        max_seconds=get_runner_slot_deference_max_seconds(),
        better_priority=better,
    )
    # ... unchanged from here: window <= 0 claims, otherwise the existing
    # deference_satisfied / _continuous_eligibility_start / eligible_since flow ...
```

Three properties of this shape are load-bearing and must be preserved:

- **Precise evidence is universal; speculative evidence is not.** A parked queued waiter
  is definite evidence — it is already at the gate — so it applies at every priority,
  including the default. The "unstarted agent that has not queued yet" heuristic stays
  gated on `priority > DEFAULT_WAIT_PRIORITY` exactly as today, because it can
  false-positive on an agent parked for hours behind a dependency wait.
- **The no-candidate fast path must stay free.** When `better is None` the gate must
  claim immediately without reading the deference config and without rewriting the
  marker. `tests/test_run_agent_runner_slot_priority.py:213`
  (`test_default_and_better_priorities_claim_without_deference_config`) pins this;
  compute the window only _after_ a candidate is found.
- **`eligible_since` semantics are unchanged.** The window still measures continuous
  eligibility and still resets when eligibility is lost.

Worked example from the screenshot: at `running_count == 3`, `toobig-2j…` (`p20`)
becomes eligible, finds parked `zh` (`p1`), and defers `min((20 - 1) * 3, 60) = 57s` of
continuous eligibility instead of claiming instantly. If the remaining agents drain
inside that window, `zh` gets its lull; if they do not, `toobig-2j…` still starts —
bounded, never starved.

### Step 4 — order the display by priority first

`src/sase/core/runner_slots/_admission.py`, `runner_slot_queue_display_key()`:

```python
return (
    normalize_wait_priority(priority),
    1 if parked else 0,
    -effective_threshold if parked else 0,
    *runner_slot_waiter_sort_key(...)[1:],   # invalid, parsed, timestamp, artifact_dir
)
```

Rank becomes **priority → capacity eligibility → nearest-opening threshold → FIFO**. The
tuple keeps its arity and element types, so the annotation
`tuple[int, int, int, int, datetime, str, str]` and both call sites
(`src/sase/ace/tui/models/agent_runner_slots.py:177` and
`src/sase/integrations/agent_list_entries.py:161`) need no change; ACE and
`sase agent list -j` stay consistent with each other automatically.

Capacity-awareness is retained _within_ a priority group, which is where `fb7b8366e`'s
benefit actually lives — every default-priority waiter shares one group, so a parked
`≤0` waiter still sorts behind admissible peers. The `parked` amethyst accent, the
`N parked` heading, and the `≤N` / `pN` annotations are unchanged and now carry the
capacity nuance that rank no longer encodes.

Expected result for the screenshot: `#1 zh ≤0 p1`, `#2 toobig-2j… ≤3 p20`.

## Tests

Update:

- `tests/test_runner_slots.py`
  - `test_display_key_places_parked_waiter_after_admissible_waiter` — the p1/p20 case
    now inverts to `["drain", "default"]`; rename it to describe priority winning over
    capacity, and add a same-priority variant that still asserts
    admissible-before-parked.
  - `test_display_key_orders_parked_waiters_by_threshold_descending` — unchanged
    expectation (equal priority), keep as the within-group guard.
  - `test_display_key_keeps_priority_then_fifo_for_admissible_waiters` — unchanged.
  - `test_display_key_degrades_to_admission_key_at_zero_running_count` — rewrite for the
    new tuple shape: at `running_count == 0` the key is
    `(priority, 0, 0, *admission[1:])`.
  - `test_deference_window_scales_only_worse_priorities_and_clamps` — add gap-based
    cases (e.g. `deference_window_seconds(20, better_priority=1, …) == 57.0`,
    `deference_window_seconds(10, better_priority=1, …) == 27.0`, and a clamp case).
  - `test_better_priority_agent_pending_finds_only_plausible_arrivals` — port to
    `best_pending_better_priority`, asserting returned priorities/`None` instead of
    bools; the `/parked` case still returns `None` here because parked waiters are now
    the other helper's job.
  - New `best_parked_better_priority` coverage: parked better priority found; eligible
    better-priority waiter ignored; equal and worse priorities ignored; `me` ignored;
    lowest of several parked candidates returned.

- `tests/test_run_agent_runner_slot_priority.py` — new runtime cases:
  - `p20` eligible with a parked `p1` waiter defers and writes `eligible_since`, then
    claims once the window elapses.
  - `p10` (default) eligible with a parked `p1` waiter defers — the case the current
    `priority <= DEFAULT_WAIT_PRIORITY` fast path lets through.
  - Equal priority parked waiter ⇒ no deference, no marker churn.

- `tests/fakey/test_runner_slots_e2e.py` — new end-to-end case: a parked
  `wait_runners=0, wait_priority=1` barrier holds back an eligible lower-priority launch
  for the window, and the drain completes in `zh`'s favour when capacity actually frees.
  Configure the deference knobs small (as the existing priority tests do) to keep
  runtime bounded. Leave the two existing contracts at lines 302 and 345 untouched — if
  either needs editing, the change has over-reached and the design needs revisiting.

- `tests/test_agent_list_entries.py`, `tests/ace/tui/test_agent_runner_slots.py`,
  `tests/ace/tui/widgets/test_agent_queue_section.py` — update any fixture that mixes
  priorities with thresholds so ranks reflect priority-first ordering.

- `tests/ace/tui/visual/snapshots/png/agents_runner_slot_queue_window_120x40.png` —
  check whether its fixture mixes non-default priorities; refresh with
  `just test-visual --sase-update-visual-snapshots` only if the rendering actually
  changes, and eyeball the diff artifacts under `.pytest_cache/sase-visual/` before
  accepting.

## Documentation

All of these state the current capacity-first / default-never-defers contract and must
be updated together:

- `docs/troubleshooting/runner-slots.md` — the admission-order paragraph (lines 17-25),
  the deference section and its three bullets (lines 44-62), the diagnosis note about
  `runner_slot_queue_position` (line 82), and the drain-barrier paragraph (lines
  114-118): a barrier is still not a fence, but a better-priority barrier now buys a
  bounded window.
- `docs/xprompt.md:1790-1812` — the `%wait(runners=, priority=)` contract paragraph.
- `docs/configuration.md:2588-2604` — the `runner_slots` rationale and the
  `min((priority - 10) * 3, 60)` formula, now gap-based.
- `docs/cli.md:60-63` and `docs/integrations.md:123-125` — the
  `runner_slot_queue_position` ordering description.
- `docs/ace.md` — the queue-ladder ordering wording, if it repeats the capacity-first
  phrasing.
- `src/sase/config/sase.schema.json` — both `runner_slots` property descriptions ("above
  the default priority of 10" is no longer accurate).
- `src/sase/default_config.yml:43-48` — the comment block above the knobs.
- `src/sase/xprompts/skills/sase_agents_status.md:27-30` — the display-order sentence.
  This is a generated skill source; run
  `sase memory read generated_skills.md --reason "Editing a generated skill source"`
  before touching it and follow whatever deployment step it prescribes.

Do not edit `sase/memory/*.md`, `AGENTS.md`, or the provider instruction shims.

## Verification

1. `just install`
2. `just check` (inline is fine; hand it to `/sase_monitor` if it runs long).
3. `just test-visual` for the ACE PNG suite.
4. `just check-full` **only** through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …`) with a `--next` action, before
   landing.

Symvision will flag the removed `better_priority_agent_pending` export — remove it from
`__init__.py` and `__all__` rather than pragma-ing it. Read
`sase memory read symvision.md --reason "Resolving symbol lint after a runner-slot API rename"`
if any lint failure is not obvious.

## Out of scope

- Porting runner-slot admission to the Rust core. The boundary rule in `CLAUDE.md`
  argues it belongs there, but `sase-core` currently carries only the `wait_priority` /
  `wait_runners` wire fields (`crates/sase_core/src/agent_scan/wire.rs`) and no
  admission module. Keep this fix in `src/sase/core/runner_slots/`; a port is separate
  work.
- Priority aging, preemption of running agents, and any hard head-of-line fence.
- The `0s` queue durations visible for both waiters in the screenshot.
  `slot_requested_at` is preserved across polls in the current code and both agents
  plausibly reached the gate in the same second, so there is no evidence of a defect
  here — noted only so the next reader does not mistake it for one.
