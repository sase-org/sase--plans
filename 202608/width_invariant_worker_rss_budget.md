---
tier: tale
title: Make the suite-cost worker-RSS budget width-invariant
goal:
  "`just check-full`'s test-cost gate stops going red on clean master at every xdist
  width the runner legitimately grants, while still failing on a real per-worker memory
  regression."
size: medium
proposed_by: bbugyi200.athena.sase-j0
bead: sase-j0
create_time: 2026-08-11 12:02:12
status: wip
---

- **BEAD:**
  [sase-j0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j0/README.md)

# Plan: Make the suite-cost worker-RSS budget width-invariant

## Problem

`tools/check_test_cost_budgets` is the last step of `just check-full`. Bead `sase-j0`
was closed once after commit `c8e4016c7` recalibrated
`tests/perf/baselines/test_cost_budgets.json`, and was reopened the same day because the
gate went red again. Since the recalibration, every reported failure has been the _same
single metric_:

- reopen `+1` (2026-08-10): `peak_worker_rss_kib` actual `1325528` KiB exceeded budget
  `1100000` + 15% tolerance (`1265000`).
- `+1` from an epic land agent (2026-08-11): `peak_worker_rss_kib` actual `1342924` KiB,
  same budget; an immediate re-run of `just test-cost` alone, with no code change,
  passed.

Every other summary metric and every cause budget from the original six-way failure now
passes. The remaining defect is entirely in how `peak_worker_rss_kib` is defined and
budgeted.

## Evidence

All eight retained cost recordings on host `athena` (2026-08-11 13:15Z-15:37Z, node
counts 28,904-28,968), reading `summary.worker_rss_curve_kib`:

| workers | start   | post_collection | median  | peak          |
| ------- | ------- | --------------- | ------- | ------------- |
| 14      | 149,796 | 536,488         | 536,304 | 1,090,140     |
| 14      | 144,392 | 534,384         | 534,318 | 1,072,932     |
| 14      | 150,736 | 561,380         | 560,140 | 1,055,696     |
| 14      | 144,344 | 534,076         | 534,074 | 1,010,568     |
| 14      | 146,176 | 535,260         | 534,434 | 1,017,276     |
| 14      | 144,096 | 532,940         | 532,886 | 998,800       |
| **4**   | 144,292 | 539,084         | 535,442 | **1,342,924** |
| 14      | 144,364 | 534,264         | 533,204 | 1,000,900     |

- `post_collection` spread across all eight runs: 532,940-561,380 KiB (5.3%).
- `median` spread: 532,886-560,140 KiB (5.1%).
- `peak` spread: 998,800-1,342,924 KiB (34.5%), and the maximum is the single 4-worker
  run.

Per-worker detail from those two shapes:

- 4-worker run: workers executed 502/621/639/779 test files; their peaks were 1,044,240
  / 1,342,924 / 917,904 / 1,032,224 KiB; each worker's `post_collection` was
  534,228-539,084 KiB.
- 14-worker run: workers executed 88-275 test files; their peaks were 660,620-1,000,900
  KiB; each worker's `post_collection` was 532,768-534,264 KiB.

So the _per-worker steady footprint is identical_ at both widths, and only the
high-water mark moves.

## Root cause

`peak_worker_rss_kib` is built in `tests/_test_cost.py:build_cost_record()` as `max`
over workers of each worker's `peak_rss_kib`, and
`tests/_test_cost_plugin.py:_peak_rss_kib()` reads
`resource.getrusage(RUSAGE_SELF).ru_maxrss` — a monotone, never-decreasing kernel
high-water mark. A worker's high-water therefore grows with how many test files that
worker executes, which is `suite_size / width`.

`tools/run_pytest` deliberately varies that width. `_parallel_worker_grant()` asks
`WorkerTokenLease.acquire()` for anything inside
`tests/_suite_gate.py:automatic_worker_range()`, which on this host is **floor 4,
ceiling 14** (host token budget 32; floor `min(4, budget)`, ceiling
`min(28, budget) // 2`). A contended host grants the floor, a quiet host grants the
ceiling. Both are legitimate recordings of clean master.

A single flat suite-wide limit cannot serve both: set it to pass at width 4 and it is
~34% slack at width 14 (poor detection); set it to pass at width 14 and every contended
run is red. That is exactly the reopen history — the gate is measuring the grant width
as much as it measures the code.

The two width-invariant statistics that _do_ describe per-worker memory are already
recorded in `worker_rss_curve_kib` and are currently unused by any budget.

## Design decision

1. Promote the width-invariant curve statistics into flat `summary` keys so budgets can
   address them, and make **those** the per-worker memory regression detector, budgeted
   tightly.
2. Keep `peak_worker_rss_kib`, but re-derive and re-document it as an **OOM-headroom
   ceiling** that must hold across the whole legitimate 4-14 width band, not as a tight
   regression detector. Derive it from recordings at the _narrowest_ width, because that
   is the width that maximises it.

This is chosen over the alternatives deliberately:

- _Just raise the flat peak limit again_ — that is what `c8e4016c7` did, and it was
  reopened; on its own it also leaves the gate blind at width 14.
- _Skip the peak check below a reference width_ — hides the metric on exactly the
  contended runs where memory pressure matters most.
- _Fit a width model for the peak_ — data-poor (one width-4 sample retained,
  `KEEP_COST_RECORDINGS = 8`) and unfalsifiable in review.

## Work

### 1. Answer "stale budget or real regression" with evidence first

The bead's SCOPE requires deciding between recalibration and a genuine regression. One
fact makes this worth a bounded probe: the sase-ib.5 measurement quoted in
`tests/_suite_gate.py` recorded `post_collection=median=peak=500632` KiB — i.e. _no_
execution-phase growth above the post-collection footprint. Today `post_collection` is
essentially unchanged (~535,000 KiB) but the peak is 1.9-2.5x higher, so execution-phase
growth is new even though steady-state memory is not.

Localise it without touching any schema, using the existing directory override so probe
recordings never evict real history (`KEEP_COST_RECORDINGS = 8`):

```bash
export SASE_TEST_COST_DIR=<a scratch directory outside the repo>
SASE_PYTEST_WORKERS=1 just test-cost tests/ace
SASE_PYTEST_WORKERS=1 just test-cost --ignore=tests/ace
```

Compare `peak` against `post_collection` in each probe recording
(`tools/test_cost_report` already prints the whole curve). Narrow further into whichever
subtree carries the growth.

Decision gate:

- A small, obviously-fixable retainer (a fixture or module-level collection that
  accumulates across a worker's lifetime) — fix it in this plan if the fix is small and
  self-contained, and record the before/after curve.
- Growth that is diffuse across unrelated subtrees, or concentrated in inherently heavy
  machinery (Textual app instances, PNG/font loading) — record the finding verbatim in
  the budget file's `notes` array and in the bead close note, and file a follow-up task
  bead with `/sase_new_task` if a real but larger fix is implied.

Either way, continue to step 2: the budget shape is wrong at every width regardless of
what the probe finds.

### 2. Record the width-invariant metrics as flat summary keys

In `tests/_test_cost.py`:

- In `build_cost_record()`, add `median_worker_rss_kib` and
  `post_collection_worker_rss_kib` to the `summary` mapping, mirroring
  `worker_rss_curve_kib["median"]` and `worker_rss_curve_kib["post_collection"]`. Leave
  `worker_rss_curve_kib` itself untouched.
- Add both to `_SUMMARY_FIELDS` so `tools/test_cost_report` prints and diffs them
  alongside `peak worker RSS KiB`.

`_summary_value()` only resolves flat `summary` keys, which is why nested curve entries
cannot be budgeted today; this is the minimal change that makes them addressable. Older
recordings simply lack the keys, and `check_cost_budgets()` already skips a budget whose
actual is `None`.

In `tools/check_test_cost_budgets`, add both metrics to `SUGGESTED_SUMMARY_METRICS` with
`per_worker=False` (they are already per-worker readings; the existing `per_worker` flag
means "divide a summed total by the worker count" and must not be set here).

### 3. Collect a calibration sample that spans the width band

Run on a quiet host — check `uptime` (load average well under the core count) and that
no other `sase ace` agents, `cargo`/`rustc` builds, or hourly `rsync` backups are
running. A contended host both narrows the granted width and adds noise.

- At least **3 recordings at `SASE_PYTEST_WORKERS=4`** (band floor).
- At least **3 recordings at `SASE_PYTEST_WORKERS=14`** (band ceiling on this host;
  re-derive the ceiling with `automatic_worker_range()` if the host differs).

Wall-clock guidance from recorded history: a width-14 cost run takes ~10 minutes, a
width-4 run ~35 minutes. Copy every recording into one curated directory as it lands so
the 8-recording prune cannot evict the calibration sample.

### 4. Recalibrate and document

Derive the limits with the tool, not by hand:

```bash
SASE_TEST_COST_DIR=<curated directory> tools/check_test_cost_budgets --suggest
```

Apply to `tests/perf/baselines/test_cost_budgets.json`:

- `median_worker_rss_kib` and `post_collection_worker_rss_kib`: take the suggested
  limits directly. Sanity-check that both land within ~10% of 535,000 KiB; a suggestion
  far above that means the sample is contaminated.
- `peak_worker_rss_kib`: take the suggested limit and confirm it is driven by the
  width-4 runs. Do not lower it below what the observed width-4 maximum requires.
- Leave every other summary and cause budget alone unless `--suggest` shows it is now
  violated by the fresh sample; those metrics are not part of this defect.

Rewrite the `notes` array so a future reader can reproduce the numbers:

- The derivation formula and the `--suggest` provenance block (host, date range, worker
  counts, node counts) that the tool prints.
- That `median_worker_rss_kib` and `post_collection_worker_rss_kib` are the
  width-invariant per-worker memory regression detectors, with the measured spread that
  justifies calling them width-invariant.
- That `peak_worker_rss_kib` is `max(ru_maxrss)` over workers, grows as
  `suite_size / width`, is therefore an OOM-headroom ceiling for the whole 4-14 band
  rather than a tight regression detector, and must be derived from width-floor
  recordings.
- Whatever step 1 concluded about execution-phase growth.

Update or replace the now-stale claim in the existing notes that the post-collection
reading is "~520 MiB"; state the measured value and its spread.

### 5. Keep the gate provably sharp

In `tests/test_test_cost.py`:

- Assert that `build_cost_record()` sets the two new flat summary keys equal to the
  corresponding `worker_rss_curve_kib` entries.
- Add a width-invariance test: two synthetic records built from _identical_ per-worker
  RSS curves, one with 4 workers and one with 14, must produce the same
  `median_worker_rss_kib` / `post_collection_worker_rss_kib` actuals and both must pass
  the committed budgets.
- Add a detection test: a record whose per-worker steady RSS is raised (e.g. every
  worker's curve shifted up by 50%) must fail `median_worker_rss_kib` and
  `post_collection_worker_rss_kib`, at both widths.
- `test_every_committed_budget_flags_a_doubled_metric` is parametrized over the
  committed budget file, so it picks up the new budgets automatically — confirm it
  passes rather than editing it.
- `test_committed_pre_epic_baseline_still_fails_recalibrated_budgets` asserts an exact
  failure list. `tests/perf/baselines/test_cost_baseline.json` predates the new keys, so
  those budgets should be skipped for it and the list should be unchanged; verify rather
  than assume, and update the list if the baseline does carry curve data.

## Verification

- `just install` first — workspace directories are ephemeral and may have stale
  dependencies.
- `just check` after each edit.
- `just check-full` end to end, green, including `tools/check_test_cost_budgets`.
- Additionally prove the gate is not merely wide: after recalibrating, re-run
  `tools/check_test_cost_budgets` against the width-4 and width-14 recordings from step
  3 and confirm all pass, and confirm the step-5 detection test fails the budgets when
  steady RSS is inflated.
- `tests/test_test_cost.py` fully green.

## Known blockers and constraints

- `just check-full` also validates the `sase_core_rs` binding. If the local build is
  stale the run fails before the cost gate; rebuild the sibling Rust core, opening that
  repo through the `/sase_repo` skill and using only the path it prints.
- Full cost-lane runs are slow and memory-hungry; do not run them concurrently with
  other agents' suites, and re-check host quiet between calibration runs.
- Do not pass `-n`/`--numprocesses`; `tools/run_pytest` rejects it. Width is set with
  `SASE_PYTEST_WORKERS`.

## Out of scope

- `tests/_suite_gate.py`'s `_RESERVED_MEMORY_PER_TOKEN` reserves 700 MiB per token and
  its comment justifies that as "roughly 40% headroom" over the same stale sase-ib.5
  peak of 500,632 KiB. Measured worker peaks are now 998,800-1,342,924 KiB, so that
  headroom claim no longer holds. Changing the reservation changes host-wide test
  concurrency, which is a separate decision with separate blast radius — file it with
  `/sase_new_task`, referencing bead `sase-j0`, rather than changing it here.
- Cause-bucket granularity (`subprocess_run` / `subprocess_popen`) remains tracked by
  the canceled bead `sase-ip`; nothing here reopens it.
- Bead `sase-gk` recalibrates the diff-scoped lane's serial-runtime budget against real
  lane history. Different budget file and lane, but the same "committed perf budget no
  longer matches reality" shape — the methodology in step 3 and step 4 (curated
  recording directory, `--suggest` provenance in the notes array, span the band the
  runner can legitimately produce) should be reused there.
