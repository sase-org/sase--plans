---
tier: tale
title: Restore truthful runner-occupancy history
goal:
  The Statistics "Runners" sub-tab measures every runner-slot lane that ran in the selected window, using only
  trustworthy end timestamps, so occupancy stops collapsing to near zero for windows older than one day and discloses
  any lane it could not measure.
proposed_by: bbugyi200.athena.rn.f1
create_time: 2026-08-02 08:14:13
status: done
---

- **PROMPT:**
  [prompts/202608/runner_occupancy_history.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/runner_occupancy_history.md)

# Restore Truthful Runner-Occupancy History

## Problem

The **Statistics → Runners** sub-tab is wrong for every window longer than about a day, and the error grows with the
window. Measured against the live artifact index on 2026-08-02 07:46 (effective runner limit `10`):

| Window | Metric      | Reported | Actual    |
| ------ | ----------- | -------- | --------- |
| 24h    | peak        | 9        | 9         |
| 24h    | average     | 0.30     | 0.25      |
| 24h    | busy        | 6.4%     | 5.9%      |
| 24h    | runner time | 7.1h     | 6.1h      |
| 7d     | peak        | 9        | **10**    |
| 7d     | average     | 0.04     | **1.54**  |
| 7d     | busy        | 0.9%     | **39.3%** |
| 7d     | runner time | 7.1h     | **258h**  |
| 30d    | average     | 0.02     | **1.61**  |
| 30d    | busy        | 1.1%     | **54.5%** |
| 30d    | runner time | 16.9h    | **1160h** |

The last-24h window is roughly right; everything older is not. The clearest symptom: the 7-day window reports the
**same** 7.1 runner-hours as the 24-hour window, because the 7-day chart contains essentially nothing older than one
day. Of the 789 runner-eligible lanes that overlap the 7-day window, **15** reach the chart.

Everything on the sub-tab except the `Current limit` tile derives from that one aggregation: the five summary tiles,
`Concurrency over time`, `Occupancy by runner count`, `Busiest slices`, and the `% at/above` limit figure. All of them
are affected.

## Root Causes

### 1. `WHERE hidden = 0` deletes most of the history (primary cause)

`query_run_stats_with_liveness` (`crates/sase_core/src/agent_stats/run.rs:198-212`) reads the artifact index behind
`WHERE hidden = 0`. This is the same filter the gate-backed plan/question work routed around; runner occupancy still
sits behind it.

Rows become hidden as index bookkeeping, not by user intent. A lane that does not exit through the normal done-marker
path (killed from the TUI, crashed, superseded) leaves no `done.json`, `running.json`, `waiting.json`,
`pending_question.json`, or `workflow_state.json`. After 24 hours of directory staleness,
`record_is_terminalization_candidate` (`crates/sase_core/src/agent_scan/index.rs:1344`) accepts it and
`terminalized_abandoned_record` (`index.rs:1432`) writes a synthetic `done` marker with `outcome: "abandoned"` and
`hidden: true`, which `RecordSummary::from_record` ORs into the row's `hidden` column.

Measured on the live index for the 7-day window:

- 805 launches; **581 sit on hidden rows** and 224 are visible.
- `agent_meta.hidden` — the flag `%hide` and auto-dismiss set, i.e. actual user intent — is **false on every one of
  them**. Zero rows in the window were hidden by the user.
- Of 789 runner-eligible lanes overlapping the window: **774 dropped as hidden**, 28 dropped as open-ended, 15 counted.
  (A further 262 visible records are plan-chain/family continuations that are correctly excluded — see below.)

Because occupancy integrates over time, losing 96% of lanes collapses `busy` from 39.3% to 0.9%.

### 2. The synthetic terminal timestamp is fabricated, so simply deleting the filter is worse

`terminalized_abandoned_record` sets `finished_at: current_unix_timestamp_secs()` — the moment the _index_ noticed the
row, not the moment the agent stopped. Measured over the in-window abandoned rows, that stamp lands **24h to 98h (median
43h)** after the artifact directory's last real activity.

Removing `WHERE hidden = 0` with no other change therefore produces garbage: on the same 7-day window it yields peak
**223** concurrent runners and an average of **120**, because hundreds of dead lanes appear to run for days.

The honest end time is already present in most cases: `agent_meta.stopped_at` is set on **537 of the 567** in-window
abandoned lanes (median real runtime 22.7 minutes), and `member_terminal` (`crates/sase_core/src/agent_runtime.rs:~196`)
already takes `min(stopped_at, done.finished_at)`, so those lanes resolve correctly the moment they stop being filtered
out. The remaining ~30 have no usable end at all and must be skipped explicitly rather than stretched to the
terminalization stamp.

Note this fabricated stamp is not only a statistics problem — every consumer that reports an abandoned agent's runtime
inherits a median 43-hour inflation.

### 3. The human-wait model does not match runner-slot semantics

The chart compares occupancy against the admission limit, so "occupied" must mean what admission means. Live admission
(`running_root_agent_count`, `src/sase/core/runner_slots/_admission.py:150`) releases a slot **only** while
`pending_question.json` exists; a planner blocked in `handle_plan_approval` keeps its slot for the entire review wait.

`member_human_waits` (`crates/sase_core/src/agent_runtime.rs:236`) models something different, and is wrong in both
directions:

- it subtracts plan-review waits, which admission never released;
- it subtracts question waits only when `question_response_path` is unset, so a question that _was_ answered — the case
  where the slot really was released — is not subtracted at all.

Measured magnitude on the current 7-day window is small (0.6h excluded out of 258h) because plan approvals here are
fast, but the definition should still match the limit the chart is drawn against.

That function is shared with clan/family wall-clock runtime, where "exclude human waits" is the correct policy. The fix
must not change that consumer.

### 4. Coverage loss is silent

The view surfaces only `malformed_rows_skipped` and `invalid_intervals_skipped` (24 on the current window). The 774
hidden lanes and the lanes with no usable end are invisible, so `Busy 0.9%` reads as a measurement rather than as
missing data.

### Correctly excluded, for the record

`is_runner_eligible_record` (`agent_runtime.rs:191`) drops records with `agent_meta.parent_timestamp` unless
`agent_family_parallel`. That is right: a plan chain is one process holding one slot, and the lane root and its
continuation record share the same `stopped_at` (verified on `rn--plan` / `rn--code`, both
`2026-08-02T11:32:02.517646+00:00`), so the root's interval already spans the whole lane. This plan does not change lane
eligibility.

## Approach

Keep the existing interval/sweep machinery in `crates/sase_core/src/agent_stats/runner.rs` — the occupancy segmentation,
distribution, and trend math are correct. Fix the three inputs feeding it: which rows are read, which terminal
timestamps are trusted, and which waits count as released. Then make what was skipped visible instead of silently
absorbing it into idle time.

Steps 1–6 are in the **`sase-core`** repo (open it with `/sase_repo`; this is core backend logic per the
`rust_core_backend_boundary` memory). Steps 7–8 are in the **`sase`** repo.

**Decision to confirm at review:** step 1 stops the statistics query from filtering on the index `hidden` column, and
excludes only rows the user hid (`agent_meta.hidden`). Because that column feeds the whole run payload, this also raises
the other Statistics sub-tabs — `Runs` for the 7-day window goes from 224 to 805. That is the intended reading: the
panel should report what actually ran, and `%hide` is an agent-list preference, not an analytics opt-out. The practical
difference for the user-hidden carve-out is currently zero rows.

## Implementation

### Step 1 — Read the rows that actually ran

In `query_run_stats_with_liveness` (`crates/sase_core/src/agent_stats/run.rs:198-212`), drop `hidden = 0` from the
statement, keeping the `project_name` predicate and `ORDER BY timestamp ASC`.

Add a small predicate next to the existing row helpers that returns whether the _user_ hid a record — true when
`record.agent_meta.hidden` is true — and skip such rows for both the launch-window aggregates and the runner pass. Do
not consult `done.hidden`: the terminalizer sets it, so it does not carry user intent.

Cost check: the query decodes `record_json` per candidate row, so this roughly triples decode work. The full-index
(`all` preset) query currently runs in 0.15s over 5,686 rows; confirm it stays well under the TUI's off-thread budget
after the change (see `sase/memory/tui_perf.md`).

### Step 2 — Never trust a synthesized terminal timestamp

Add `terminal_is_synthesized: bool` to `ClanRuntimeMemberWire` (`crates/sase_core/src/agent_runtime.rs:15`), set in
`from_record` from `done.outcome == "abandoned"`. That outcome is written only by `terminalized_abandoned_record`
(`ABANDONED_DONE_OUTCOME`, `index.rs:52`); no Python writer emits it, so it is a reliable synthesized-marker signal.

In `member_terminal`, ignore `finished_at` when `terminal_is_synthesized` is set, and treat the member as having an
unusable terminal when no other candidate (`stopped_at`) survives. Return a distinct `ActiveIntervalError` variant for
that case — `UnusableTerminal` — so callers can count it separately from the malformed-terminal case.

`aggregate_clan_runtime` must keep its current behavior of skipping members it cannot derive; only the error label
changes there.

### Step 3 — Match slot-yield semantics in the runner pass

Give `derive_active_intervals` an explicit wait policy rather than branching inside `member_human_waits`. Add a
`WaitPolicy` enum with two variants:

- `HumanWaits` — today's behavior, used by `aggregate_clan_runtime` / clan and family wall-clock runtime. Unchanged.
- `SlotYield` — used by `RunnerStatsBuilder::add_record`. Excludes only intervals where the slot was actually released:
  a live `pending_question.submitted_at` through the member end, and each `questions_submitted_at` entry through its
  answer. Plan-review waits are **not** excluded.

For a resolved question the answer time is not in the artifact record. Resolve it from the question gate bundle the
existing reader already parses — `sase_home/interaction_requests/question/<request_id>/response.json`
(`crates/sase_core/src/agent_stats/gate_bundles.rs`) — matching on the producer's artifacts directory, and fall back to
"not excluded" when no bundle resolves. Falling back to _not excluded_ keeps the runner series conservative: it may
count a released slot as held, but it never invents idle time.

`RunnerStatsBuilder::add_record` needs the resolved question-answer times passed in (or a lookup closure) rather than
opening bundles per record; build the map once in `query_run_stats_with_liveness` and thread it through.

### Step 4 — Count what was skipped

Extend `AgentRunnerStatsWire` (`crates/sase_core/src/agent_stats/wire.rs`) with:

- `lanes_counted: u64` — lanes contributing at least one clipped interval.
- `lanes_without_end_skipped: u64` — runner-eligible lanes whose terminal was unusable (step 2) or open-ended without
  live corroboration. Split the existing open-ended liveness rejection out of `invalid_intervals_skipped` into this
  field so "invalid" keeps meaning "malformed interval bounds".
- `user_hidden_skipped: u64` — rows skipped by the step 1 carve-out.

Populate these in `RunnerStatsBuilder`. Document each field's meaning in the wire doc comment.

### Step 5 — Stop fabricating the timestamp at the source

In `terminalized_abandoned_record` (`index.rs:1432`), set `finished_at` from the best available truth instead of `now`:
`agent_meta.stopped_at` when parseable, else the artifact directory's latest modification time — which
`artifact_dir_is_stale` already computes via `artifact_dir_latest_modified` (`index.rs:1365`); thread that value into
the terminalizer rather than re-walking the directory. Fall back to `None` when neither is available, rather than
stamping a fabricated time.

Add `finished_at_estimated: bool` (serde default `false`) to `DoneMarkerWire` (`agent_scan/wire.rs:110`) and set it on
the synthesized marker. The Python wire converts through `known_field_kwargs`
(`src/sase/core/agent_scan_wire_conversion.py:143`), so it tolerates the new key; add the field to
`src/sase/core/agent_scan_wire_markers.py` only if a Python consumer needs to read it.

This corrects new terminalizations. Existing rows keep their fabricated stamps, which step 2 already distrusts; do not
attempt a backfill — directory mtimes on historical rows have been overwritten by later bulk maintenance (a large batch
all reports the same `2026-07-27 06:11` mtime), so a rewrite would replace one wrong number with another.

### Step 6 — Rust tests

In `crates/sase_core/src/agent_stats/run.rs` and `runner.rs` tests:

- **Primary regression:** a lane on a `hidden = 1` row with a synthesized `abandoned` done marker and a real
  `agent_meta.stopped_at` contributes its true interval — this is the exact shape of the 774 dropped lanes.
- The synthesized `finished_at` is never used as an interval end: a lane with `stopped_at` at T+10m and a synthesized
  stamp at T+40h ends at T+10m.
- A lane with a synthesized marker and no `stopped_at` is skipped and increments `lanes_without_end_skipped` rather than
  extending to the synthesized stamp.
- A row with `agent_meta.hidden = true` is excluded and increments `user_hidden_skipped`.
- `SlotYield` keeps a plan-review wait occupied while `HumanWaits` excludes it, over the same member.
- `SlotYield` excludes a resolved question wait using the gate-bundle response time, and does not exclude it when no
  bundle resolves.
- `aggregate_clan_runtime` results are unchanged for a member with a plan-review wait (guards the shared consumer).

### Step 7 — Python view model

Add `lanes_counted`, `lanes_without_end_skipped`, and `user_hidden_skipped` to `RunnersView`
(`src/sase/stats/_view_models.py:292`) and read them in `build_runners_view` (`src/sase/stats/_runner_view.py`), using
the existing `integer` coercion and defaulting to zero for older payloads. Keep the unavailable-view branch typed and
complete.

### Step 8 — TUI disclosure

In `src/sase/ace/tui/modals/statistics_pane_runners.py`, extend `_runner_context` so the existing partial-snapshot note
also reports lanes excluded for want of a usable end, e.g.
`N lanes ran in this window with no recorded end time and are not counted.` Keep it on the existing gold dim style and
keep the note absent when all counters are zero.

Check `src/sase/ace/tui/modals/statistics_pane_legends.py` for any runner legend text that describes the old filtered
behavior and correct it.

## Verification

1. `just check` in the `sase` repo and the crate test suite in `sase-core`; rebuild the binding with `just install`.
2. Re-run the exact comparison from the Problem table:

   ```python
   from sase.stats.query import query_run_stats
   from sase.stats.ranges import resolve_preset
   for preset in ("24h", "7d", "30d"):
       rng = resolve_preset(preset)
       print(preset, query_run_stats(start_ts=rng.start_ts, end_ts=rng.end_ts)["runners"])
   ```

   Expect the 7-day window near **peak 10, average ~1.5, busy ~39%, ~258 runner-hours** — not 0.04 / 0.9% / 7.1h. Drift
   of a few percent from new activity is expected; a 7-day figure still in single-digit runner-hours, or equal to the
   24-hour figure, means the fix did not take. The 24-hour window should move only slightly.

3. Sanity-check the ceiling: peak must not exceed the configured limit (`10`) by more than the small overshoot real
   admission allows. A peak in the hundreds means a synthesized timestamp is still being trusted.
4. Open the panel and confirm `Occupancy by runner count` now shows a populated 1–10 distribution rather than ~99% idle,
   that `Busiest slices` shows differentiated days, and that `Concurrency over time` is not flat.
5. Switch the project filter (`p`/`P`) and confirm the runner figures still change with it.

## Out Of Scope

- Reworking `hidden`/terminalization semantics for the Agents tab. Marking handed-off and killed lanes `abandoned` and
  hidden is questionable on its own terms, but this plan corrects the timestamp it records and stops statistics from
  inheriting the display decision, rather than perturbing agent-list behavior.
- Backfilling fabricated `finished_at` values on already-terminalized rows (see step 5).
- Reviewing the non-runner Statistics sub-tabs metric by metric. Step 1 raises their inputs to the true population, but
  auditing each of those views for its own defects is separate work.
- Whether runner-slot admission should count a plan-chain continuation record — it currently does not, because the lane
  root holds the slot for the whole chain.
