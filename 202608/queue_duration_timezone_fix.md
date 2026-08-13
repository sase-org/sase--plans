---
tier: tale
title: Fix queued-agent durations always rendering as 0s
goal:
  QUEUED agents show their true queue time on the Agents-tab detail header and QUEUE
  ladder in any configured timezone, backed by one shared helper and by tests that
  assert the duration value rather than just its label.
size: small
proposed_by: bbugyi200.athena.zk
create_time: 2026-08-13 10:50:36
status: wip
---

# Fix `0s in queue`: queue durations compare a naive local clock against an aware-UTC timestamp

## Symptom

On the ACE Agents tab, a QUEUED agent's detail panel reports `0s` for how long it has
been waiting for a runner slot, no matter how long it has actually been queued. Both
surfaces that render a queue duration are affected:

- The `Queue:` header field: `Queue: #1 of 1 · at the front · 0s in queue`
- Every row of the `❖ QUEUE` ladder: `#1 zh   ≤0 p1 0s`

Observed against a real agent that had been queued for ~23 minutes. Its on-disk marker
was correct and stable, so this is purely a **read/render-side** defect — no data is
lost or mis-written.

## Root cause

`sase.core.time.local_now()` returns a **naive configured-timezone** datetime. Its
docstring says so explicitly, and `src/sase/core/time.py` documents two side-by-side
conventions: a _display_ convention (`parse_local` / `format_local`, aware) and an
_arithmetic_ convention (`local_now` / `to_local`, naive configured-tz).

Two duplicated helpers mix the domains. They parse the stored marker into an **aware
UTC** datetime, then "reconcile" the mismatch by relabeling the naive local `now` as
UTC:

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py:82-101`
(`_queued_for_label`) and
`src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py:237-249`
(`_queue_duration`) both do:

```python
requested = datetime.fromisoformat(requested_at.replace("Z", "+00:00"))
if requested.tzinfo is None:
    requested = requested.replace(tzinfo=UTC)
now = local_now()                    # naive configured-tz wall time
if now.tzinfo is None:
    now = now.replace(tzinfo=UTC)    # BUG: stamps local wall time as UTC
elapsed = max(0.0, (now - requested.astimezone(now.tzinfo)).total_seconds())
```

`replace(tzinfo=UTC)` does not convert — it **relabels**. A `10:40:21` local wall time
becomes `10:40:21+00:00`, which is a different instant than the one it names.

Worked example from the observed agent (configured tz `America/New_York`, UTC−4):

| value                      |                                                       |
| -------------------------- | ----------------------------------------------------- |
| stored `slot_requested_at` | `2026-08-13T14:17:27.107866+00:00` (= 10:17:27 local) |
| `local_now()` at render    | naive `2026-08-13 10:40:21`                           |
| after the buggy relabel    | `2026-08-13 10:40:21+00:00`                           |
| `elapsed`                  | `10:40:21Z − 14:17:27Z` = **−13026.1 s**              |
| after `max(0.0, …)`        | **0.0** → `format_compact_duration(0.0)` → `"0s"`     |
| correct answer             | `1373.9 s` → `"22m53s"`                               |

The `max(0.0, …)` clamp is what makes this invisible: the error is a _sign_ error, and
the clamp converts every negative result to a flat `0s` instead of an obviously wrong
value.

**Blast radius.** In any configured timezone behind UTC, every queue duration shorter
than the UTC offset renders as `0s` (4 hours' worth in US Eastern). Past that offset the
value becomes nonzero but still wrong — understated by exactly the offset. In a timezone
_ahead_ of UTC the duration is overstated by the offset. Only UTC itself renders
correctly, which is why this survived CI.

## Why the existing tests miss it

Three tests cover this rendering and none of them fail today:

1. `tests/ace/tui/widgets/test_agent_queue_section.py:130` and
   `tests/ace/tui/widgets/test_agent_list_status_indicators.py:508` assert only
   `" in queue" in header.plain` — presence of the label, never its value. `0s in queue`
   satisfies that.

2. `tests/ace/tui/visual/test_ace_png_snapshots_agents.py:162` _does_ assert a value
   (`"3m in queue"`), but its fixture data cancels the bug out. The fixture in
   `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py:301,320` uses
   `slot_requested_at="2026-07-12T12:01:00Z"` / `"2026-07-12T12:00:00Z"` while
   `pin_agents_visual_now` pins `local_now()` to naive `2026-07-12 12:03:00`. The
   fixture implicitly assumes local wall time _equals_ UTC — the exact assumption the
   buggy code makes — so the two errors cancel and it reports `3m`.

That last point matters for the fix: the fixture encodes the bug, so it must be
retargeted as part of this change (see step 3).

## Fix

### Step 1 — Add one shared, correct helper

The two buggy copies are the same 12 lines. Replace them with a single helper in
`src/sase/ace/tui/models/agent_time.py`, next to `format_compact_duration` (both call
sites already import from that module, and `local_now` is already imported there).

Add `to_local` and `parse_local` to the existing `sase.core.time` import, then:

```python
def queued_for_label(
    requested_at: str | None,
    now: datetime | None = None,
) -> str | None:
    """Return a compact elapsed label for a runner-slot request timestamp.

    ``requested_at`` is a stored ``slot_requested_at`` marker value, which the
    runner writes as an aware-UTC ISO string. Both sides are normalized to the
    naive configured-tz arithmetic convention before subtracting, so the two
    operands stay in one time domain; comparing an aware instant against
    ``local_now()`` directly is what produced a negative elapsed time clamped
    to ``0s``.

    Returns ``None`` when *requested_at* is absent or unparseable.
    """
    if not requested_at:
        return None
    parsed = parse_local(requested_at)
    if parsed is None:
        return None
    reference = local_now() if now is None else now
    elapsed = max(0.0, (to_local(reference) - to_local(parsed)).total_seconds())
    return format_compact_duration(elapsed)
```

Notes for the implementer:

- `parse_local` yields an aware configured-tz datetime; `to_local` flattens it to naive
  configured-tz. `to_local` returns naive inputs unchanged, so a caller passing a naive
  `now` (the ladder does) and a caller passing none both work.
- Keep the `max(0.0, …)` clamp — clock skew can legitimately produce a small negative —
  but it must no longer be load-bearing, which the step 4 tests enforce.
- This changes how a _naive_ `requested_at` is interpreted: previously "assume UTC", now
  "assume configured-tz wall time", matching `to_local`/`parse_local` convention and the
  rest of the repo. This is safe: the only writer,
  `src/sase/axe/run_agent_wait_slots.py` (lines 169, 286), always writes
  `datetime.now(UTC).isoformat()`, and an audit of all 45 live `waiting.json` markers on
  this host found **zero** naive values.
- Presentation-only formatting of an already-persisted timestamp, so per the Rust core
  backend boundary this correctly stays in Python. No `sase-core` change is needed;
  there is no Rust queue-duration API today.

### Step 2 — Route both call sites through it

- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`: delete
  `_queued_for_label` (lines 82-101) and call the shared helper at line 251. Drop the
  now-unused function-local `datetime` / `UTC` / `local_now` / `format_compact_duration`
  imports.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_queue_section.py`: reduce
  `_queue_duration` (lines 237-249) to a thin wrapper preserving its `"?"` sentinel for
  missing/unparseable input:

  ```python
  def _queue_duration(requested_at: str | None, now: datetime) -> str:
      return queued_for_label(requested_at, now=now) or "?"
  ```

  `"0s"` is truthy, so a genuine zero still renders as `0s` rather than `?`. Keep the
  existing `now = local_now()` hoist at line 125 (one clock read per ladder render, and
  the visual harness patches that module's `local_now`). Remove the `UTC` import if it
  becomes unused.

Verify with `just lint` that no now-unused imports remain in either file.

### Step 3 — Retarget the visual snapshot fixture

In `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py`, change the two
`slot_requested_at` values so they name the _real instants_ the fixture intends, given
`local_now()` is pinned to naive local `2026-07-12 12:03:00` (= `16:03Z`, EDT, UTC−4):

- line 301: `"2026-07-12T12:01:00Z"` → `"2026-07-12T16:01:00Z"` (renders `2m`)
- line 320: `"2026-07-12T12:00:00Z"` → `"2026-07-12T16:00:00Z"` (renders `3m`)

This was verified numerically: the rendered strings stay **exactly** `3m` and `2m`, so
`test_ace_png_snapshots_agents.py`'s `"3m in queue"` assertion still holds and the PNG
golden `agents_runner_slot_waits_120x40` **does not need regenerating**. If that golden
does report a diff, stop and inspect `.pytest_cache/sase-visual/` rather than accepting
it with `--sase-update-visual-snapshots` — an unchanged golden is the expected outcome
and a diff means something else moved.

### Step 4 — Regression tests that would fail today

The whole reason this shipped is that no test asserted a queue-duration _value_ under a
non-UTC clock. Add coverage that does.

Unit tests for the new helper (co-locate with the other `agent_time` tests, e.g.
`tests/ace/tui/models/test_agent_time.py` — follow whatever file already covers
`format_compact_duration`):

- Aware-UTC input with an explicit naive-local `now` returns the true elapsed time. Use
  the real-world shape: `requested_at="2026-08-13T14:17:27+00:00"`,
  `now=datetime(2026, 8, 13, 10, 40, 21)` → `"22m53s"`. Under the current code this
  returns `"0s"`, so the test fails before the fix and passes after.
- A sub-offset duration is not flattened to zero: 90 s before `now` → `"1m30s"`.
- `None`, `""`, and an unparseable string return `None`.
- A future `requested_at` still clamps to `"0s"` (clock skew guard preserved).

Rendering tests that pin the value, not just the label:

- Strengthen `tests/ace/tui/widgets/test_agent_queue_section.py`'s
  `test_queue_field_renders_front_label_and_section_requires_real_position` (line ~130)
  from `" in queue"` to an exact expected duration. Give `_agent` / `_entry` a
  `slot_requested_at` at a known offset from a pinned `local_now()` instead of the
  current hard-coded `"2026-07-25T12:00:00Z"` compared against the real wall clock.
- Add a ladder test asserting the per-row duration column, so both surfaces are pinned.

Use the existing `tz_divergence` fixture (`tests/_conftest_runtime.py:151`) for at least
one test. It forces configured tz (`America/New_York`) ≠ system tz (`UTC`), which is
precisely this bug class, and its docstring already names it. Note the suite-wide
`_pin_configured_timezone` fixture pins every test to `America/New_York`, so a
value-asserting test catches the regression even without `tz_divergence`.

### Step 5 — Sweep for the same pattern

`local_now()` has 134 call sites. Grep for the specific hazard — a `local_now()` result
being relabeled rather than converted:

```bash
rg -n 'replace\(tzinfo=' src/ | rg -v 'tests/'
```

Cross-check each hit against whether the other operand is aware. The audit done while
diagnosing this found the two helpers fixed here as the only queue-duration offenders.
`src/sase/ace/tui/models/agent_time.py:358` is a _different_, benign use (stamping naive
model datetimes for the Rust `aggregate_clan_runtime` wire, where both sides share the
convention) — leave it alone. Other `slot_requested_at` consumers
(`src/sase/integrations/agent_list_entries.py`, `src/sase/agent/running_listing.py`,
`src/sase/agents/cli_list.py`) use the value only as a boolean or a lexicographic sort
key, never for arithmetic, so they are unaffected.

If the sweep turns up unrelated offenders outside the queue-duration path, do **not**
fix them here — file them with `/sase_new_task` so this change stays reviewable.

## Verification

1. `just install` first — workspace virtualenvs are ephemeral and may be stale.
2. Confirm the new unit test fails before the step 1/2 edits and passes after.
3. `just test-visual` for the PNG suite; expect the `agents_runner_slot_waits_120x40`
   golden to be unchanged.
4. `just check` for lint gates plus the diff-scoped test lane.
5. Because this touches a shared time helper and the visual snapshot fixtures, finish
   with `just check-full`. Run it **only** through `/sase_monitor`
   (`sase monitor start --command 'just check-full' …`) with a `--next` action, never
   inline.

Real-world confirmation of the expected result: with the fix, the agent from the report
(queued at `2026-08-13T14:17:27+00:00`, observed at `10:40:21` local) renders
`22m53s in queue` instead of `0s in queue`.

## Out of scope

- **No writer changes.** `src/sase/axe/run_agent_wait_slots.py` is correct: it preserves
  `slot_requested_at` across poll iterations and only rewrites `waiting.json` when the
  marker actually changes. The observed marker was written once and left alone for the
  full 23-minute wait.
- **Queue duration on left-panel Agents-tab rows.** A QUEUED row currently shows no
  elapsed suffix at all, because `_leaf_runtime_interval`
  (`src/sase/ace/tui/models/agent_time.py:270-340`) only produces an interval for
  `RUNNING`/`RETRYING`/`ANSWERED`, or `WAITING` with a `run_start_time`; a queued agent
  has no `run_start_time` yet by definition. Adding a queue-time suffix there is a
  behavior addition needing its own tick/refresh wiring (`row_runtime_or_wait_ticks`),
  row-render cache-key work — `slot_requested_at` is currently a static cache-key
  component and would need a time-varying trigger — and new PNG goldens. This plan
  restores _correct_ durations on the two surfaces that already display them. If the
  intent was also to put a queue time next to every queued row in the left-hand list,
  that is a separate follow-up worth filing.
