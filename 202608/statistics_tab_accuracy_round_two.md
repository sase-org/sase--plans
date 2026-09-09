---
status: done
tier: epic
title: Fix the second round of inaccurate Statistics tab data
goal: "Every number, label, and metric definition rendered by the Admin Center
  Statistics tab matches what its producer actually computes, including the All time
  range, the Perf subsystem latency counts, the Overview commits tile, and the silently
  truncated XPrompts breakdowns.

  "
phases:
  - id: core-counters
    title: Correct the Rust statistics counters and expose breakdown truncation
    depends_on: []
    size: medium
    description: "core-counters: in ../sase-core, make `committing_agents` count
      distinct agent names and add `committing_runs`, stop counting a missing archive
      project spec as malformed, scope `user_hidden_skipped` to runner-eligible rows,
      add per-row xprompt breakdown truncation counts, bump the wire schema, and update
      the pinned version in this repo's core validator.

      "
  - id: perf-latency
    title: Stop asserting zero samples and meaningless shares in Perf latency rows
    depends_on: []
    size: small
    description: "perf-latency: make the By Subsystem Count column render an unknown
      count as an em dash instead of 0, remove the incommensurable Share denominator for
      that grouping, and define the Launch hero tile's statistic in the Perf legend.

      "
  - id: range-window
    title: Make the All time window and empty-window states honest
    depends_on: []
    size: medium
    description: "range-window: replace the epoch-derived All time absolute label, clamp
      the Overview runs-over-time buckets to the observed data instead of emitting one
      row per day since 1970, and stop the run-count empty state from hiding log-backed
      Activity and Plans & Questions data.

      "
  - id: consume-core
    title: Render the corrected core counters and disclose XPrompt truncation
    depends_on:
      - core-counters
    size: medium
    description:
      "consume-core: read the new wire fields so the Overview commits tile stops
      labelling runs as agents, correct the Projects malformed-file footnote and the
      stale Specs column, disclose the XPrompts breakdown truncation, split the XPrompts
      Share legend by denominator, and refresh the affected PNG goldens."
proposed_by: bbugyi200.athena.04y
bead_id: sase-oo
create_time: 2026-09-09 19:51:39
---

- **PROMPT:**
  [prompts/202608/statistics_tab_accuracy_round_two.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/statistics_tab_accuracy_round_two.md)
- **BEAD:**
  [sase-oo](https://github.com/sase-org/sase--beads/blob/main/pages/sase-oo/README.md)

# Plan: Fix the second round of inaccurate Statistics tab data

## Problem

`0229e3881` ("fix(ace-tui): correct inaccurate Statistics tab data and metric
definitions") fixed eight findings from a first audit, but that audit deliberately
declared `../sase-core` out of scope and only examined the numbers reachable from the
default range. A second audit of every rendered value found eleven further defects.

Four of them are wrong numbers, not wrong wording, and four require changes in
`../sase-core` because the defect is in the producer. Two more only appear when the
range is `All time`, which the first audit never exercised.

Every finding below was verified by running the real bindings against the live artifact
index and telemetry store from a workspace checkout, or by reading the exact producer.

Statistics code lives in:

- `src/sase/stats/` — I/O-free view builders and the thin binding adapters.
- `src/sase/ace/tui/modals/statistics_pane*.py` — the Textual pane and Rich renderables.
- `../sase-core/crates/sase_core/src/agent_stats/` — the run, activity, and runner
  producers.
- `../sase-core/crates/sase_core/src/perf_logs/` and
  `../sase-core/crates/sase_core/src/telemetry/store.rs` — the Perf producers.

Open `../sase-core` with the `/sase_repo` skill and use the path it prints; never guess
a sibling path.

## Findings

### F1. Perf "By Subsystem" reports Count 0 for rows that do have latency samples

`_SUBSYSTEM_ROWS` in `src/sase/stats/_perf_view_latency.py` declares no counter metric
for the `workflows` and `axe` rows, and `_subsystem_rows` turns that into a literal
zero:

```python
count = int(_counter_value(telemetry, count_key)) if count_key else 0
```

`PerfLatencyRow.count` is typed `int`, so the renderer prints `0`. Verified against the
live telemetry store over a seven-day window:

| row        | p50   | p95   | max    | rendered Count |
| ---------- | ----- | ----- | ------ | -------------- |
| Workflows  | 1298s | 1298s | 1298s  | `0`            |
| Axe cycles | 2.56s | 5.09s | 210.5s | `0`            |

The table therefore asserts "zero samples" beside three populated percentile columns.
The honest rendering for "this grouping queries no counter for this row" is `—`, which
the same table already uses for `Err%` and `Retry%` in exactly that situation.

### F2. The Perf "By Subsystem" Share column divides by a sum of different units

`_subsystem_rows` builds `total = sum(counts)` across all five rows and sets
`share = share(count, total)`. Those counts are agent runs, LLM invocations, and hook
executions. On the same live window the denominator was
`1139 + 1296 + 0 + 0 + 0 = 2435`, so the view rendered "Agent runs 46.8% / LLM calls
53.2%" — a percentage of a quantity that does not exist. The sibling `By Provider`
grouping has the same unit mixing but already carries
`_PERF_PROVIDER_COUNT_FALLBACK_NOTE` to disclose it; `By Subsystem` has no such note and
its mixing is worse, because agent runs and LLM invocations are not even alternatives
for one row, they are separate rows.

### F3. The Overview "Commits" tile labels a run count as an agent count

`statistics_pane_rendering.py` renders:

```python
f"{overview.commits}\n{overview.committing_agents} agents"
```

`committing_agents` comes straight from the wire, and `fold_commits` in
`../sase-core/crates/sase_core/src/agent_stats/run.rs` increments it once per **run**:

```rust
commits.total_commits += count;
if count > 0 {
    commits.committing_agents += 1;
}
```

Proof from the live index over thirty days: `committing_agents` was `888`, exactly equal
to `distribution.one + distribution.two + distribution.three_plus` (`23 + 226 + 639`),
which is by construction the number of runs with at least one commit.

Agent names do repeat across runs, so this is not a harmless synonym. From the same
payload's xprompt rows: `json` had 43 runs across 2 distinct agents, and `propose` had
32 runs across 1 distinct agent. The codebase already distinguishes the two ideas
elsewhere — `PatchWorkAccumulator.agents` and `XPromptAccumulator.agents` are
`BTreeSet<String>` of agent names — so the tile's "agents" reading is simply wrong.

`AgentCommitStatsWire::average_per_committing_agent` divides by the same per-run counter
and is misnamed for the same reason. It only reaches the unrendered `RunsView`, so it
carries no user-visible defect today, but leaving it inconsistent with a corrected
`committing_agents` would be a trap.

### F4. Every view claims the All time window starts on 1969-12-31

`resolve_preset("all")` sets `start_ts = _ALL_TIME_START_TS = 0` and then builds the
absolute label from that sentinel:

```python
def _absolute_label(start_ts: int, end_ts: int, timezone: tzinfo | None) -> str:
    start = datetime.fromtimestamp(start_ts, timezone)
    end = datetime.fromtimestamp(end_ts, timezone)
    return f"{start:%Y-%m-%d %H:%M %Z} – {end:%Y-%m-%d %H:%M %Z}"
```

Verified output for the `all` preset:

```
display_label: All time
label: 1969-12-31 19:00 EST – 2026-08-17 11:53 EDT
```

`_range_scope_text` appends `self._range.label` to the Range chip on every view whenever
the pane is not width-compacted, and `StatisticsHelpModal._freshness_text` prints
`Range — {self._selected_range.label}.` So the tab states a concrete analysis window
that begins before SASE existed. It also contradicts two of the tab's own disclosures:
the Runners legend says "All time = begins at earliest valid runner coverage", and the
Perf coverage note says "All time means as far back as retained telemetry and bounded
logs go".

### F5. Overview "Runs over time" emits 20,683 rows for All time, nearly all empty

`build_empty_buckets` in the Rust producer deliberately retains zero-count buckets, and
`bucket_seconds_for(0, now)` returns `86_400`. `build_overview_view` maps every bucket
into `OverviewView.buckets`, and `_overview_renderable` adds one table row per bucket.

Verified against the live index with the real `all` preset:

- `bucket_count`: `20683`
- buckets with a nonzero run count: `98`
- first bucket `start_ts`: `0`

So the panel renders more than twenty thousand rows labelled `Jan 01`, `Jan 02`, … from
1970 onward, each with an empty scale bar, to show 98 populated days. The `maximum` used
to scale every bar is computed across all of them. This is both unreadable and a serious
render cost on a tab whose responsiveness is covered by `sase/memory/tui_perf.md`.

### F6. "N malformed Patch files skipped." counts missing archive files and misnames them

`load_patch_metadata` in the Rust producer always appends the archive spec path and then
treats an unreadable path as malformed:

```rust
let archive = active
    .parent()
    .map(|parent| preferred_project_spec_path(parent, &basename, true));
let mut paths = vec![active.to_path_buf()];
if let Some(archive) = archive.filter(|path| path != active) {
    paths.push(archive);
}
for path in paths {
    let Ok(content) = fs::read(&path) else {
        malformed += 1;
        continue;
    };
```

`preferred_project_spec_path(.., archive = true)` returns the canonical
`<project>-archive.sase` path even when neither the canonical nor the legacy file
exists, so a project with attributed Patches and no archive file always contributes one
phantom "malformed" count. The existing Rust test bakes this in: the `other` project has
a deliberately corrupt `other.sase` (a single `0xff` byte) and no archive file, and the
test asserts `malformed_spec_files_skipped == 2`.

On the live host, 12 of the 13 projects that have a `.sase` file have no
`-archive.sase`. The counter currently reads `0` only because the single project with
attributed Patches in range (`gh_sase-org__sase`) happens to have one, so this is a
latent false footnote that fires as soon as any other project does Patch work.

Separately, the footnote text is wrong regardless of the count. These are project spec
files, not Patch files:

```python
f"{projects.malformed_spec_files_skipped} malformed Patch files skipped."
```

### F7. The XPrompts drilldowns silently truncate to five children per xprompt

`query_run_stats` sends `xprompt_breakdown_top_n = 5`, and `finish_xprompts` applies it:

```rust
models: ranked_counts(value.models, Some(breakdown_top_n)),
projects: ranked_counts(value.projects, Some(breakdown_top_n)),
partners: ranked_counts(value.partners, Some(breakdown_top_n)),
```

`_xprompts_drilldown_renderable` renders those children with a `Share` column and no
truncation footnote of any kind. Verified against the live index:

| xprompt | runs | model rows shown | model share covered |
| ------- | ---- | ---------------- | ------------------- |
| `gh`    | 3614 | 5                | 97.4%               |
| `fork`  | 282  | 5                | 93.6%               |

So `By Model`, `By Project`, and `Used With` present shares that do not sum to 100% with
nothing on screen explaining why. This is the same class of defect the first audit fixed
for the Activity view by retitling its panels "Top skills"/"Top memories"/"Top
workspaces"; the XPrompts drilldowns were missed. The truncation count is not on the
wire, so the producer has to publish it.

The focused breakdown is not affected: `finish_xprompts` passes `None` as the limit for
a focus payload.

### F8. The XPrompts "Share" legend documents the wrong denominator for child rows

The legend reads:

```python
_MetricLegend("Share", "share of runs that referenced any xprompt"),
```

That is correct for a top-level row, where `build_xprompts_view` computes
`share = ratio(row_runs, runs_with_xprompts)`. It is wrong for every child row and for
every focus breakdown table, where `_xprompt_count_rows` computes
`share = ratio(count, owning_runs)` — a share of that one xprompt's runs. The
`By Model`, `By Project`, and `Used With` tables put both denominators in a single
column called `Share` under a legend that only describes one of them.

### F9. The Runners "user-hidden rows" diagnostic counts rows that were never runners

The hidden carve-out in `query_run_stats_with_liveness` runs before the runner
eligibility test and before the launch-window branch:

```rust
if record_is_user_hidden(&record) {
    runner_stats.record_user_hidden();
    continue;
}
```

The loop reaches that point for any record that is launch-in-window **or** a runner
overlap candidate, so `user_hidden_skipped` also counts hidden records that are not
`ace-run` workflows, hidden records that never recorded `run_started_at`, and hidden
`ace-run` records that `is_runner_eligible_record` would reject (serial family children,
non-agent workflow steps). `AgentRunnerStatsWire` documents the field as "Candidate rows
omitted because `agent_meta.hidden` records user intent", and `_runner_context` renders
it under "Partial valid snapshot: … N user-hidden rows were not counted." So the Runners
view can declare its snapshot partial on the strength of rows that could never have held
a runner slot.

The live index currently has zero hidden records, so this is provable by reading the
producer rather than by observation.

Note for the implementer, not a finding to fix here: the same carve-out silently removes
hidden runs from every other view's totals with no disclosure anywhere on the tab. That
is a defensible product decision (the user asked to hide those agents) and changing it
would move real numbers, so it stays out of scope. See the Scope section.

### F10. The Perf "Launch" hero tile shows p95 and nothing says so

`build_launches` sets `value = p95_ms`, while its neighbour `build_startup` sets
`value = median`. The Perf legend defines the median one and not the other:

```python
_MetricLegend("Startup", "median visible-ready time; ok < 2s, warn < 5s"),
```

There is no `Launch` legend entry at all. The help modal's "Percentiles (logs)" row
names the shared _method_ ("Startup, Launch, and stall medians: Nearest-rank …") but
implies the Launch tile is a median, which it is not. Two adjacent duration tiles
therefore report different statistics with no way to tell them apart.

### F11. The run-count empty state hides log-backed Activity and Plans & Questions data

`_paint_current_view` gates on the run count:

```python
if result.views.empty and self._view not in {"runners", "xprompts", "perf"}:
```

`views.empty` is `totals.runs == 0`, which counts agent **launches** inside the window.
The Activity view's skills and memories come from `skill_uses.jsonl` and
`memory_reads.jsonl`, and Plans & Questions comes from durable gate bundles; all three
are timestamped by when the event happened, not by when the producing agent launched. A
window in which no agent launched but already-running agents used skills or submitted
plans therefore renders "No agent runs recorded in <range>." over data that exists.

This is ranked last deliberately: it could not be reproduced on the live host, because
every one-hour window in the last seven days contained at least one launch. It is a
structural defect in the gate, confirmed by reading both producers.

## Scope

In scope, in this repo:

- `src/sase/stats/_perf_view_latency.py`, `src/sase/stats/_view_models.py`,
  `src/sase/stats/_perf_view_sections.py`, `src/sase/stats/_view_builders.py`,
  `src/sase/stats/ranges.py`
- `src/sase/ace/tui/modals/statistics_pane_perf.py`, `statistics_pane_legends.py`,
  `statistics_pane_projects.py`, `statistics_pane_rendering.py`,
  `statistics_pane_views.py`, `statistics_pane_xprompts.py`,
  `statistics_pane_runners.py`, `statistics_help_modal.py`
- `tools/validate_sase_core_rs` (the pinned agent-stats wire schema version)
- The matching tests under `tests/stats/` and `tests/ace/tui/`, and the Statistics PNG
  goldens under `tests/ace/tui/visual/snapshots/png/`

In scope, in `../sase-core`:

- `crates/sase_core/src/agent_stats/wire.rs`
- `crates/sase_core/src/agent_stats/run.rs`

Out of scope, and intentionally left alone:

- The hidden-run carve-out itself (see the note in F9). Hidden runs stay excluded from
  every aggregate; only the Runners diagnostic counter is corrected. File a follow-up
  task bead through `/sase_new_task` if the exclusion should be disclosed on the tab.
- `RunsView` and `RuntimeView`. Both are still built on every load and never rendered.
  `202608/remove_statistics_runs_runtime_tabs.md` already owns their removal; this plan
  only keeps `build_runs_view` compiling and internally consistent.
- Reconciling the telemetry and artifact-index run populations. The help modal already
  discloses that they are not comparable.
- `stalls.top_contexts` deduplication, still blocked on hitch/stall pairing the JSONL
  records do not carry.
- The 2000–2007 artifact timestamps in the live index. They are real rows in
  `gh_sase-org__sase` and they widen the All time span, but F5 is fixed by clamping the
  rendered bucket window, not by filtering records.

## Implementation

### Phase `core-counters` — Correct the Rust statistics counters and expose breakdown truncation

All work in `../sase-core/crates/sase_core/src/agent_stats/`, plus one pinned version in
this repo.

**F3 — commit attribution.** In `wire.rs`, `AgentCommitStatsWire` keeps
`committing_agents` but changes its meaning to the number of distinct agent names with
at least one commit, and gains `committing_runs: u64` (`#[serde(default)]`) for the
existing per-run count. Document both fields. Change `average_per_committing_agent` to
divide by the corrected `committing_agents` so its name matches its value, and document
that `distribution` stays per-run.

In `run.rs`, `fold_commits` needs the resolved agent name, which `fold_work` already
derives:

```rust
let agent = normalized(
    record.agent_meta.as_ref().and_then(|meta| meta.name.as_deref())
        .or(row.agent_name.as_deref()),
);
```

Hoist that derivation to the call site so both folds share one value rather than
recomputing it, pass it into `fold_commits`, and collect committing names in a
`BTreeSet<String>` alongside the existing counters. Finish the set into
`committing_agents` next to the existing `average_per_committing_agent` computation.

**F6 — malformed spec files.** In `load_patch_metadata`, only treat a path as malformed
when it exists but cannot be read or parsed. The cheapest correct form is to skip a
candidate whose `fs::read` fails with `ErrorKind::NotFound`, which also stops a
never-archived project from being reported and keeps genuine permission and I/O errors
counted. Update the existing test that asserts `malformed_spec_files_skipped == 2` to
`1` — the corrupt `other.sase` — and add a case proving a project with a valid active
spec and no archive file reports `0`.

**F9 — runner diagnostic scope.** Keep the hidden `continue` exactly where it is so
hidden runs stay out of every launch aggregate, but make the diagnostic conditional on
the row actually being a runner row:

```rust
if record_is_user_hidden(&record) {
    if runner_candidate && is_runner_eligible_record(&record) {
        runner_stats.record_user_hidden();
    }
    continue;
}
```

`is_runner_eligible_record` is already `pub(crate)` in `agent_runtime.rs` and already
imported by `runner.rs`; import it in `run.rs` too. Add a test with a hidden
non-`ace-run` record and a hidden serial family child (`parent_timestamp` set,
`agent_family_parallel` false) proving neither reaches `user_hidden_skipped`, and a
hidden runner-eligible record proving it still does.

**F7 — breakdown truncation counts.** Add `models_truncated: u64`,
`projects_truncated: u64`, and `partners_truncated: u64` (each `#[serde(default)]`) to
`AgentXPromptStatsRowWire`. In `finish_xprompts`, compute each as the accumulator map
length minus the length of the ranked list actually emitted, before `ranked_counts`
consumes the map. Leave the focus payload untouched; it is already unbounded.

**Wire version.** Bump `AGENT_STATS_WIRE_SCHEMA_VERSION` from `5` to `6`. Then, in this
repo, update the pinned probe in `tools/validate_sase_core_rs`:

```python
if payload.get("schema_version") != 5:
```

Extend that probe to also assert `committing_runs`, `committing_agents`, and the three
new xprompt truncation fields are integers, matching how it already asserts the runner
counters.

Verify in `../sase-core` with its own Rust test command, then rebuild the binding into
this workspace with `just rust-install` (or `just install`) before running any Python
check that reads the new fields.

### Phase `perf-latency` — Stop asserting zero samples and meaningless shares in Perf latency rows

Python-only, in this repo. Independent of every other phase.

**F1.** Change `PerfLatencyRow.count` in `src/sase/stats/_view_models.py` to
`int | None`. In `_perf_view_latency.py`, have `_subsystem_rows` emit `None` when the
row declares no `count_key`, and keep an integer everywhere a counter was queried. In
`statistics_pane_perf.py`, render the Count cell through a helper that returns `—` for
`None`, mirroring `_perf_rate`. `_provider_rows` and `_workflow_rows` always query a
counter, so their counts stay integers and their sort key stays valid; keep the
`-row.count` sort well defined by treating `None` as `0` in the sort key only.

**F2.** For the subsystem grouping, stop publishing a share computed across
incommensurable units. Set `share = 0.0` for those rows and suppress the `Share` column
when `latency.group_by == "subsystem"`, so the table shows four duration and reliability
columns without a fabricated percentage. `By Provider` and `By Workflow` keep their
Share columns unchanged.

**F10.** Add a `Launch` entry to the `perf` legend in `statistics_pane_legends.py`
stating that the tile reports the p95 total launch time, and correct the help modal's
"Percentiles (logs)" row so it does not describe the Launch tile as a median. Keep the
nearest-rank method wording, which `percentile_sorted` in
`../sase-core/crates/sase_core/src/perf_logs/aggregate.rs` does implement.

Update `tests/stats/test_views_perf.py`, `tests/ace/tui/test_statistics_perf.py`, and
`tests/ace/tui/test_statistics_help_modal.py`. Add a regression test asserting a
subsystem row with histogram values and no counter renders `—` rather than `0`.

### Phase `range-window` — Make the All time window and empty-window states honest

Python-only, in this repo. Independent of every other phase.

**F4.** In `src/sase/stats/ranges.py`, stop deriving an absolute label from the
`_ALL_TIME_START_TS` sentinel. `resolve_preset("all")` must keep `start_ts = 0` — the
Rust producer relies on `request.start_ts == 0` to select its all-time behaviour — and
instead carry a label that states only what is known, such as
`"through {end} · start bounded by retained data"`. Keep `StatsRange` a `NamedTuple`
with the same four fields so no caller changes shape. Every other preset and every
custom range keeps its current absolute label.

**F5.** Clamp what the Overview chart renders rather than what the producer computes. In
`build_overview_view` (`src/sase/stats/_view_builders.py`), trim leading and trailing
zero-run buckets before building `OverviewView.buckets`, keeping interior zero buckets
so gaps stay visible, and keep the untrimmed sequence out of the view entirely. Then
bound what survives: if the trimmed sequence still exceeds a documented module constant
(start from 96 rows, which covers a 90-day daily range and a 48-hour hourly range),
aggregate adjacent buckets into equal-width groups and label each group from its first
bucket, so the panel stays a fixed height for any range. Disclose the aggregation in the
panel title the way the Activity view discloses its top-N, for example
`"Runs over time · 7-day buckets"`.

`OverviewView.buckets` also feeds the Agents Run tile sparkline through
`_paint_overview_tiles`; the trimmed and grouped sequence is strictly better there too,
so no separate handling is needed.

**F11.** Split the empty state so it only speaks for the views it describes. Keep the
run-count empty state for `overview`, `projects`, and `providers`, whose data is
entirely run-derived. For `activity` and `plans_questions`, gate on their own payloads
being empty — no skill, memory, plan, or question rows and zero counts — and word that
state for the data actually missing rather than claiming no agent runs were recorded.

Update `tests/stats/test_views_overview.py` (or the closest existing module) for the
bucket trimming and grouping, add a `ranges.py` test asserting the all-time label never
renders a 1969 or 1970 date, and add a pane test for a window with zero runs but
non-empty activity rows.

### Phase `consume-core` — Render the corrected core counters and disclose XPrompt truncation

Depends on `core-counters`. Run `just install` first so the rebuilt binding with wire
schema 6 is importable in this workspace.

**F3.** `build_overview_view` and `build_runs_view` already read `committing_agents`;
with the producer corrected they now get distinct agent names, so
`statistics_pane_rendering.py`'s `"{n} agents"` becomes true with no change. Add
`committing_runs` to `OverviewView` and `RunsView` and render the tile detail as both
values, for example `"{committing_agents} agents · {committing_runs} runs"`, so the tile
answers the question the commit distribution answers elsewhere. Add an `overview` legend
entry defining `Commits` as total commits with its distinct-agent and run counts. Update
the payload fixtures in `tests/stats/_views_payloads.py`,
`tests/ace/tui/_statistics_pane_helpers.py`, and
`tests/ace/tui/visual/_ace_config_center_statistics_views.py`, which all currently set
only `committing_agents`.

**F6.** Correct `_projects_footnotes` in `statistics_pane_projects.py` to describe what
the counter counts, for example `"{n} unreadable project spec files skipped."` While in
that file, retire the stale `Specs` column header for `Patches`, and drop the
now-redundant `_MetricLegend("Specs", "distinct Patches")` translation from the
`projects` legend in favour of a definition of the column itself.

**F7.** Consume the new per-row truncation counts. Add `models_truncated`,
`projects_truncated`, and `partners_truncated` to `XPromptRow` in
`src/sase/stats/_view_models.py`, populate them in `build_xprompts_view`, and have
`_xprompts_drilldown_renderable` append a dim indented row per parent whose selected
dimension is truncated, for example `"  +N more not shown"`, using the same dim italic
style as the existing `"(no model recorded)"` placeholder row.

**F8.** Split the `xprompts` `Share` legend into the two denominators it actually uses:
the xprompt row's share of runs that referenced any xprompt, and a child row's share of
that xprompt's own runs.

**Goldens.** Re-run the visual suite and accept the intentional changes with
`--sase-update-visual-snapshots`. Expect `config_center_statistics_overview_120x40.png`,
`config_center_statistics_projects_120x40.png`,
`config_center_statistics_projects_drilldown_120x40.png`,
`config_center_statistics_xprompts_model_120x40.png`, and
`config_center_statistics_help_120x40.png` to change; inspect
`.pytest_cache/sase-visual/` for any golden that changes unexpectedly before accepting
it.

## Verification

Per-phase, before handing off:

- `core-counters`: `../sase-core`'s own Rust test and lint commands, then `just install`
  in this workspace followed by `.venv/bin/python tools/validate_sase_core_rs` to prove
  the bumped schema and new fields satisfy the pinned probe.
- `perf-latency`, `range-window`, `consume-core`: `just check` inline, and
  `just test-visual` for `consume-core`.

Before landing the epic's combined tree, run `just check-full` through `/sase_monitor`
(`sase monitor start --command 'just check-full' …`) with a `--next` action, never
inline.

Manual confirmation, because most of these findings are only visible on screen: open
`sase ace`, go to the Admin Center Statistics tab, and for each of the eight views cycle
the range through every preset including `All time` and cycle every grouping. Confirm
that the Range chip never shows a 1969 or 1970 date, that Overview's runs-over-time
panel stays a fixed readable height on `All time`, that no Perf latency row shows `0`
beside populated percentiles, and that the XPrompts `By Model` drilldown discloses its
truncation for a high-volume xprompt such as `gh`.
