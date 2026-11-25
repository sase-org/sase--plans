---
tier: tale
title: Stop stranded-work toast floods from artifact-index lock contention
goal:
  ACE toasts each stranding incident once. Artifact-index lock contention never
  re-announces long-stranded beads or makes stranded badges and counts flicker.
size: medium
decisions:
  coalesce:
    ask:
      Should one warmup pass that finds more than three newly stranded beads collapse
      into a single summary toast?
    choices:
      coalesce:
        Bursts above three beads become one summary toast naming the first three ids
        plus a "+N more" count
      per_leaf:
        Every newly stranded bead keeps its own toast, even in large genuine bursts
    default: coalesce
    why:
      a genuine mass stranding (host reboot, killed clan) would otherwise still bury the
      screen
    answer: coalesce
proposed_by: bbugyi200.athena.1fz
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 10:15:26
status: wip
---

# Stop Stranded-Work Toast Floods From Artifact-Index Lock Contention

## Problem

The ACE TUI periodically shows a wall of toasts such as
`⚐ sase-10h is stranded: --plan ended without closing the bead; nothing resumes it.`.
They name beads that have been stranded for weeks, not new ones. In the `sase` project
there are about 137 stranded beads, so every flood is about 137 toasts. While a flood is
happening, the ⚐ row badges and the stranded count also blink off for one report cycle.

## Root Cause (reproduced)

1. `session_record_cache()` in `src/sase/bead/_stranded_facts_snapshot.py` queries the
   artifact index with `lock_timeout_seconds=DEFAULT_READ_LOCK_TIMEOUT_SECONDS` (0.05
   s). The lock is `_ARTIFACT_INDEX_OPERATION_LOCK`, a process-local `RLock`
   (`src/sase/core/agent_artifact_index_lock.py`). Inside the TUI, the agent loader,
   index refresh/mutation paths, and stats all hold it. When the stranded collector
   cannot get it within 50 ms, the chunk raises
   `RuntimeError("artifact index operation lock is busy")`, and every key in that chunk
   becomes `([], False)`.
2. The sase-core reducer then classifies every member-less candidate as
   `state="unknown", reason="records_unavailable"`. That is correct core behavior and
   needs no change. The report comes back with `collection_failed=False` and an empty or
   partial `stranded_ids`.
3. `warm_stranded_work_reports()` (`src/sase/ace/tui/models/agent_stranded.py`) installs
   that partial report over the previous complete one. Only `collection_failed` is
   guarded.
4. `announce_stranded_transitions()`
   (`src/sase/ace/tui/actions/agents/_stranded_toasts.py`) compares stranded ids with
   the previous report only. When the next uncontended report brings the beads back,
   every one looks "new" and gets its own toast.

Reproduction: hold `agent_artifact_index_operation_lock()` in a thread and call
`load_stranded_report("gh_sase-org__sase")`. The result has 0 stranded and 193
`unknown/records_unavailable` verdicts. The next uncontended load "re-discovers" 137
beads in the order
`sase-10h, sase-10w, sase-10w.5, sase-110.7, sase-112, sase-116.5, …`, which matches the
user's screenshot toast for toast. A standalone process never sees this because nothing
else holds the lock, so `sase doctor` is unaffected.

## Design

Fix it at two layers, both in this repo:

- Never let a lock-degraded report replace a complete one. This fixes the badge and
  count flicker as well as the toasts.
- Key toasts by stranding incident instead of by "membership in the previous report".
  Then any future flap, from lock contention, a partial store read, or cache eviction,
  can never re-announce the same incident.

Optionally (`coalesce`), collapse genuine bursts into one toast. Do not change the
sase-core reducer, the 50 ms bounded-reader default, or the doctor check.

### 1. A degraded signal on `StrandedReport`

- In `src/sase/core/stranded_work_facade.py`, export
  `STRANDED_UNKNOWN_RECORDS_UNAVAILABLE = "records_unavailable"` next to the schema
  version. Comment that it mirrors `STRANDED_UNKNOWN_RECORDS_UNAVAILABLE` in sase-core's
  `crates/sase_core/src/stranded_work/wire.rs`.
- In `src/sase/bead/stranded.py`, add `records_unavailable: int = 0` to
  `StrandedReport`, plus a `degraded` property that returns `records_unavailable > 0`.
  In `load_stranded_report()`, after reducing, set the field to the number of verdicts
  with `state == "unknown"` and `reason == STRANDED_UNKNOWN_RECORDS_UNAVAILABLE`. The
  empty failure report keeps 0, because `collection_failed` already covers it. Update
  the dataclass docstring.

### 2. The report cache never downgrades

In `warm_stranded_work_reports()`, right after the existing `collection_failed`
`continue`:

- If `snapshot.degraded` and a previous entry exists whose snapshot is not degraded,
  skip it: `continue` without `_STRANDED_REPORT_CACHE.set()` and without recording a
  change. The previous snapshot keeps its stale expiry and fingerprint, so the next
  warmup tick (the `runtime_tick` cadence) retries naturally. Add a telemetry counter,
  `stranded_degraded_skips`, using the same `telemetry.get(...) + 1` pattern.
- A cold project, or one whose previous entry is also degraded, still installs the
  degraded report, so rows show whatever is known.
- Extend the function docstring: lock-contended partial reports, like collector
  failures, never replace a complete generation.

### 3. Incident-keyed, baseline-safe toasts

Rewrite `src/sase/ace/tui/actions/agents/_stranded_toasts.py`, keeping the public
`announce_stranded_transitions(notify, project_keys) -> list[str]` contract and the call
site in `_loading_bead_warmup_apply.py` unchanged:

- Keep the module-level name `_SEEN_LEAVES`, but make it `dict[str, dict[str, str]]`:
  project key → {bead id → incident key}. Test teardown (`tests/_conftest_runtime.py`)
  and `tests/test_config_cache_teardown.py` clear it by that name.
- The incident key is
  `verdict.last_turn_artifact_dir or verdict.last_turn or verdict.session or verdict.bead_id`.
  Do not include `reason` or `since`. Today 129 of 137 stranded verdicts carry
  `last_turn_artifact_dir`, and the other 8 carry `session`.
- Baseline: when a project has no entry yet, record every stranded verdict's incident
  quietly. If that report is `degraded`, skip it entirely, so no baseline is set until a
  complete report arrives.
- After the baseline, announce a stranded verdict only when `seen.get(bead_id)` differs
  from its current incident key, then store the new key. Never remove entries when a
  bead leaves the stranded set, whether it goes unknown, absent, blocked, or owned. A
  flap therefore stays silent, while a genuine re-stranding after a new turn has a new
  incident key and is announced. Memory stays bounded by the number of beads stranded
  during one TUI session.
- Keep the deterministic ordering (projects sorted, bead ids sorted) and the existing
  per-bead message format.

> [!decision] coalesce = coalesce Collect the new verdicts across all projects in the
> pass. With three or fewer, emit the per-bead messages as today. With more than three,
> emit exactly one message:
> `⚐ {n} beads newly stranded: {id1}, {id2}, {id3} (+{n-3} more).` Return that single
> message as the list.

> [!decision] coalesce = per_leaf Keep one message per new verdict, with no summary
> path.

Update the module docstring so it describes what the module now does. Today it claims
"coalesced" notifications even though it emits one per leaf.

### 4. Docs

In `docs/reference/work-units/bead-lifecycles.md` § Stranded beads, add one or two
sentences: ACE toasts each stranding incident once (per the `coalesce` decision, bursts
collapse into one summary toast), and while the collector cannot read sessions
(`unknown`) ACE keeps showing the last complete report. Read the `docs_writing`
reference memory first.

## Tests

Add focused tests, preferably in new files, so the existing large test modules stay
under any size lint:

- `tests/test_stranded_degraded_refresh.py`. Reuse helpers and patterns from
  `tests/test_stranded_stable_refresh.py` and `tests/test_stranded_collector_batch.py`.
  - When `_query_session_records_chunk` raises
    `RuntimeError("artifact index operation lock is busy")`, `load_stranded_report()`
    returns `degraded` with a positive `records_unavailable`, and `collection_failed`
    stays `False`.
  - With a complete generation cached, a degraded load leaves
    `cached_stranded_report(project)` as the identical previous snapshot. It reports no
    changed project and increments `stranded_degraded_skips`. The next complete load
    installs normally.
  - A cold degraded load installs. A degraded load over a degraded previous load also
    installs.
- `tests/ace/tui/test_stranded_toasts.py`. Use the `_set_report` / `_verdict` style from
  `tests/ace/tui/test_agent_stranded_detail.py`, setting reports straight into the
  cache.
  - A flap through degraded `unknown/records_unavailable` and back to stranded with the
    same last turn produces no toast. This is the regression for the screenshot.
  - A flap through absence, where the bead is missing from one report and then back,
    produces no toast.
  - The same bead re-stranded with a different `last_turn_artifact_dir` produces exactly
    one toast.
  - A degraded first report sets no baseline, and the next complete report is a quiet
    baseline.
  - For the `coalesce` default: three new leaves give three messages, and five give one
    summary message with `(+2 more)`.
- Keep `test_stranded_transition_toast_uses_quiet_first_baseline` passing unchanged.

## Verification

Run the targeted tests above, then follow the `lint_and_test` reference memory for the
required repo checks before finishing.
