---
tier: tale
title: Recalibrate the suite-cost budgets against real recorded history
goal:
  "`just check-full` passes end to end on clean master because
  `tests/perf/baselines/test_cost_budgets.json` compares each recorded metric against a
  limit derived from that same metric's real recorded history, the worker-count-scaled
  metrics are budgeted per worker instead of as a cross-worker sum, and the gate still
  fails when a real cost regression is introduced."
size: medium
proposed_by: bbugyi200.athena.sase-j0
bead: sase-j0
create_time: 2026-08-10 14:05:34
status: wip
---

- **BEAD:**
  [sase-j0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j0/README.md)

# Plan: Recalibrate the suite-cost budgets against real recorded history

## Symptom

`just check-full` on clean master fails at `just test-cost` →
`tools/check_test_cost_budgets`, after every lint gate, SASE validation, committed-plan
validation and the full pytest run itself pass. Six budgets are exceeded at once:

```
- collection_seconds: actual 272.301 exceeds budget 15.000 + 15% tolerance (17.250)
- idle_seconds: actual 2754.083 exceeds budget 900.000 + 15% tolerance (1035.000)
- peak_worker_rss_kib: actual 1028064.000 exceeds budget 716800.000 + 15% (824320.000)
- total_file_wall_seconds: actual 4064.488 exceeds budget 2300.000 + 15% (2645.000)
- causes.ace_page_enter: actual 390.155 exceeds budget 230.000 + 15% (264.500)
- causes.textual_app_run_test_enter: actual 347.313 exceeds budget 250.000 + 15% (287.500)
```

This is deterministic, not a flake: it reproduces from both `just check-full` and an
isolated `just test-cost`, and every one of the eight suite-cost recordings currently
retained by the host-local store (`~/.sase/test-selection/<project>/timings/cost/`,
`KEEP_COST_RECORDINGS = 8`) exceeds the same limits.

## Root cause

The budgets in `tests/perf/baselines/test_cost_budgets.json` were transcribed from the
sase-ib epic's **acceptance targets and partial-lane measurements**, and were never
validated against a completed full-lane recording. Two of them are also compared against
a metric that does not have the same shape as the number they were derived from.

The evidence, in order:

1. **`sase-ib.7` never validated them.** The phase that added the file (`ee9603d31`, the
   only commit that has ever touched it) closed with notes saying its full lane never
   completed — it was blocked by a stale contract manifest, ACE shared-state failures,
   and a teardown hang. Its budget proof was "the old committed _starting_ baseline
   fails the new checker", which only shows the gate is not vacuous; it does not show
   the limits match a real run.

2. **`collection_seconds` is a per-worker target compared against a cross-worker sum.**
   The epic plan's `footprint` phase says "Acceptance: collection under 15s **per
   worker**", and the budget file's own notes cite sase-ib.5's measured 12.399s. But
   `build_cost_record()` in `tests/_test_cost.py` sums `collection_seconds` across every
   worker payload, and each xdist worker collects the whole suite. Per-worker collection
   in the recorded history is 15.5–18.9s wall on an uncontended run — right at the
   target. The "18x over" reading is 12–14 workers each paying ~18s:

   | recording (UTC) | workers | collection sum | per worker |
   | --------------- | ------- | -------------- | ---------- |
   | 15:21:57        | 14      | 240.0s         | 17.1s      |
   | 16:36:49        | 11      | 204.8s         | 18.6s      |
   | 17:17:00        | 12      | 205.5s         | 17.1s      |
   | 17:22:46        | 7       | 132.6s         | 18.9s      |
   | 17:24:55        | 4       | 235.6s         | 58.9s      |
   | 17:25:44        | 12      | 351.2s         | 29.2s      |
   | 17:35:58        | 14      | 217.4s         | 15.5s      |
   | 17:46:07        | 14      | 272.3s         | 19.4s      |

   A fixed sum budget cannot be meaningful for a metric that scales with a worker count
   the suite gate hands out dynamically (4–14 in this history).

3. **`peak_worker_rss_kib` is a post-collection reading compared against a peak.**
   sase-ib.5's 500632 KiB came from a `--collect-only` cost lane (its note records
   `start=144292 / post_collection=500632 / median=500632 / peak=500632`, `samples=3` —
   a single worker that never ran a test). The committed _pre-epic_ baseline
   `tests/perf/baselines/test_cost_baseline.json` already recorded
   `worker_rss_kib: {post_collection: 512000, peak_low: 716800, peak_high: 1126400}`,
   and the epic plan's own baseline table says workers "start at ~0.5 GiB after
   collection and reach 0.7–1.1 GiB after 13 minutes". The 716800 budget is that table's
   `peak_low`; the recorded history's `post_collection` is a stable 519–531 MiB (i.e.
   sase-ib.5's number, +4%) while `peak` is 1.03–1.20 GiB. The metric never described
   the same thing the limit did.

4. **The two cause budgets encode intended reductions the full lane never showed.** The
   committed pre-epic baseline records `ace_page_enter: 390.0s / 506 calls` and
   `textual_app_run_test_enter: 422.0s / 2148 calls`. Today's median across the eight
   recordings is `ace_page_enter: 388.8s / 549 calls` and
   `textual_app_run_test_enter: 345.6s / 2726 calls`. Per call that is −8% and −35%
   respectively — the sase-ib work is visible — but the suite also grew from 27,988 to
   ~28,465 nodes and the call counts grew +8% and +27%, so the aggregate seconds the
   budget measures never fell to 230/250.

5. **`total_file_wall_seconds` and `idle_seconds` are simply below anything ever
   recorded.** The pre-epic committed baseline is 3719s of per-test wall; the 2300/900
   limits are targets. Recorded history is 3392–4551s wall and 2119–3254s idle.

### What is _not_ the cause

Today's ACE/Textual wait-idiom churn is exonerated. `c49452c47` ("widen test wait helper
gate") landed 11:48 EDT and `ebd3a91bc` ("stabilize contention-sensitive TUI waits")
landed 12:41 EDT, but the oldest retained recording — 15:21:57 UTC, i.e. **11:21 EDT,
before both** — already shows `ace_page_enter: 372.4s` and
`textual_app_run_test_enter: 340.6s`, far above the 230/250 limits. Across all eight
recordings, spanning both commits, those two causes move by ±10% with no step change.
The call counts are also flat (548–549 and 2725–2726), so nothing started entering ACE
pages or Textual apps more often.

### Why the numbers are noisy, and what that means for the fix

This lane runs on a host that runs many agents concurrently; three overlapping
`just test-cost` runs are visible in the history above (17:22, 17:24, 17:25). Wall-clock
metrics inflate under that contention — per-worker collection wall goes 15.5s → 58.9s in
the 4-worker run — while CPU-time metrics barely move (per-worker collection **CPU** is
15.1–21.5s across the whole history, including that run). A budget on wall seconds must
therefore carry contention headroom, or be stated in CPU seconds where the metric allows
it.

## Changes

### 1. `tests/_test_cost.py` — make worker-scaled metrics budgetable

- Add `collection_cpu_seconds` to the record summary in `build_cost_record()`, summed
  across worker payloads exactly like `collection_seconds` (each worker payload already
  reports it; only the summary lacks it). Keep the existing keys unchanged.
- Teach `check_cost_budgets()` a `per_worker: true` flag on a **summary** budget entry:
  when set, divide the recorded actual by the record's worker count before comparing.
  Resolve the divisor as `record["worker_count"]`, falling back to the number of worker
  payloads that reported collection time, then to `1`; never divide by zero.
- Report normalized failures unambiguously — e.g. metric
  `collection_cpu_seconds (per worker)` — so a failure message cannot be misread as the
  cross-worker sum again. Keep `CostBudgetFailure`'s fields and `format()` shape.
- Leave `causes` budgets un-normalized: they are suite-wide totals that do not scale
  with worker count.

Do not bump `TEST_COST_SCHEMA`. Adding a summary key is forward-compatible, and
`_summary_value()` already returns `None` (skip) for a metric an older recording lacks.

### 2. `tools/check_test_cost_budgets` — add `--suggest`

Add a `--suggest [--history N]` mode that reads the newest N host-local recordings
(default: all retained) and prints a budget JSON derived by the documented rule below,
together with the provenance line for the `notes` array (sample size, host, UTC date
range, worker-count range, node-count range, and per-metric min/median/max). The printed
object must load cleanly through `load_cost_budgets()`.

This is the mechanism that stops the next recalibration from being a guess, and it is
what makes "record how the numbers were derived" cheap enough to actually do.

### 3. `tests/perf/baselines/test_cost_budgets.json` — recalibrate

**Derivation rule (write it into `notes`):** a limit is
`ceil(worst value in recorded history ÷ (1 + local tolerance)) `, rounded up to a round
number, so that every recording in the sample passes and the effective ceiling sits just
above the worst observed run — including the worst observed contention. The gate then
fires on growth, not on how many other agents happened to be testing.

Re-derive with `--suggest` against the history present at implementation time; the table
below is the derivation from the eight recordings retained today (host `athena`,
2026-08-10 15:21Z–17:46Z, 4–14 workers, 28,380–28,470 nodes) and is what the result
should look like:

| budget                                        | min     | median  | max     | old limit | new limit      | allowed (+15%) |
| --------------------------------------------- | ------- | ------- | ------- | --------- | -------------- | -------------- |
| `summary.total_file_wall_seconds`             | 3392.2  | 4029.1  | 4551.2  | 2300      | 4000           | 4600           |
| `summary.idle_seconds`                        | 2119.3  | 2675.6  | 3254.0  | 900       | 2900           | 3335           |
| `summary.peak_worker_rss_kib`                 | 1028064 | 1071180 | 1203824 | 716800    | 1100000        | 1265000        |
| `summary.collection_seconds` (per worker)     | 15.5    | 18.8    | 58.9    | 15 (sum)  | 60             | 69             |
| `summary.collection_cpu_seconds` (per worker) | 15.1    | 16.4    | 21.5    | —         | 20             | 23             |
| `causes.ace_page_enter`                       | 363.4   | 388.8   | 439.3   | 230       | 400            | 460            |
| `causes.textual_app_run_test_enter`           | 335.3   | 345.6   | 391.0   | 250       | 350            | 402.5          |
| `causes.subprocess_run`                       | 303.2   | 330.1   | 353.7   | 300       | 320            | 368            |
| `causes.ace_settle_pilot`                     | 253.9   | 304.0   | 383.2   | —         | 340            | 391            |
| `causes.pilot_pause_delay`                    | 173.7   | 177.7   | 209.2   | —         | 190            | 218.5          |
| `causes.parser_create`                        | 27.5    | 30.5    | 34.0    | 50        | 35             | 40.25          |
| `causes.yaml_load`                            | 13.0    | 13.3    | 15.0    | 20        | 20 (unchanged) | 23             |

Notes on individual rows:

- **`collection_seconds` stays budgeted, per worker, as a blunt backstop only.** Its
  wall time is the most contention-sensitive number in the record (15.5s → 58.9s), so 60
  is deliberately loose: it catches an import-cost explosion, not drift. The tight
  collection gate is the new CPU one.
- **`subprocess_run` is currently the fragile pass.** Its history maxes at 353.7s
  against an allowed 345.0 — it has already been exceeded once inside this sample and
  merely was not the recording that got checked. Recalibrating it is in scope here;
  splitting the bucket is not (that is the separately tracked granularity work).
- **`ace_settle_pilot` and `pilot_pause_delay` are new budgets.** Together they are
  ~480s of the suite's hottest attributed cost and the gate is blind to them today; they
  are as stable as the causes already budgeted, so add them by the same rule.
- **`parser_create` is tightened** from 50 to 35: the sase-ib.4 reduction held and the
  headroom is real slack the gate should not be carrying.

Replace the `notes` array so it states, in this order: the derivation rule; the sample
provenance; that `collection_seconds` and `collection_cpu_seconds` are per-worker limits
while every other entry is a suite-wide total; that `peak_worker_rss_kib` is the peak of
a worker's RSS _curve_ (not the post-collection reading, which is ~520 MiB); and that
raising a limit requires a fresh recording plus `--suggest` output rather than a
hand-picked number.

### 4. `tests/test_test_cost.py` — keep the gate provably sharp

The bead's verification requires the checker still fail on a real regression. Prove it
deterministically in CI rather than by hand:

- A table-driven test that, for **every** key in the committed budget file, builds a
  record whose corresponding metric is twice the limit (worker count fixed for the
  per-worker entries) and asserts `check_cost_budgets()` reports exactly that metric.
  This makes an accidentally-vacuous budget (typo'd metric name, unreachable cause key)
  a test failure.
- A test that the committed pre-epic `test_cost_baseline.json` still fails the committed
  budgets — the same artificial-regression proof sase-ib.7 used, now re-asserted against
  the recalibrated numbers (it still trips the Textual, YAML and parser causes).
- Tests for `per_worker` normalization: a record with `worker_count: 10` and 160s of
  summed collection CPU passes a 20s per-worker limit and fails a 15s one; a record with
  a null `worker_count` falls back rather than dividing by zero.
- A test that `build_cost_record()` puts `collection_cpu_seconds` in the summary.
- A test that `--suggest` output round-trips through `load_cost_budgets()`.

### 5. `docs/perf_runbook.md` — document the semantics and the workflow

In the "Suite test-cost gate" section: state which budgets are per-worker and which are
suite-wide totals; state that `peak_worker_rss_kib` is the curve's peak, not its
post-collection value; and replace the current "raise them only with a fresh full-suite
recording and an explanation" line with the concrete workflow (`just test-cost`, then
`tools/check_test_cost_budgets --suggest`, then paste the limits _and_ the provenance
note). Keep the existing "do not raise them to hide a one-off regression" warning.

## Verification

1. `just install` first (workspaces are ephemeral).
2. Focused: `.venv/bin/python -m pytest tests/test_test_cost.py -q`.
3. Regression proof, by hand as well as by test:
   `.venv/bin/python tools/check_test_cost_budgets --recording tests/perf/baselines/test_cost_baseline.json`
   must still exit 1 and name the causes it trips.
4. `just test-cost` end to end: the run must pass `tools/check_test_cost_budgets` on its
   own fresh recording. Record the recording path and the reported actuals in the bead
   note. If the fresh recording exceeds a proposed limit (the host may be more contended
   than during the sample), re-derive that limit with `--suggest` and say so — do not
   hand-pick a number to make one run green.
5. `just check`, then `just check-full` end to end.
6. Sanity-check that the recalibration did not make the gate vacuous: every budget key
   must still be within ~1.5x of the observed median (the table above holds this), and
   the step-4 report's top causes must still be the ones the budget file names.

### Known blocker outside this change

`just check-full`'s pytest leg is currently red on master for an unrelated reason: the
stale `tests/contract_manifest.txt` and its 36-entry contract-set budget
(`tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`),
which is separately tracked and also fails the CI test matrix on all three Python legs.
If it is still open at implementation time, run `just check-full` far enough to prove
the `test cost` gate passes, report the contract-manifest failure as pre-existing with
its exact node ID rather than fixing it here, and do not close this bead claiming a
clean check-full it did not get.

## Follow-ups to file (not in scope here)

Use `/sase_new_task`, naming this bead as the source:

- **Worker RSS growth still misses the epic's 700 MiB target.** Workers stabilize ~520
  MiB after collection and grow to 1.03–1.20 GiB during a run — the same 0.7–1.1 GiB
  growth the pre-epic baseline recorded. Recalibrating the budget to reality retires an
  acceptance target that was never met, and the suite-gate memory reservation is derived
  from that number, so it is real lost concurrency.
- **The CI leg's cost budgets are calibrated on the wrong host.** CI runs
  `just test-cost` on `ubuntu-latest` under the same committed limits with only a 20%
  tolerance, and those limits are now explicitly derived from a 64-core host. CI has
  never reached the check (its pytest leg fails first), so there is no data to calibrate
  against yet; once CI is green, either validate the limits there or give the budget
  schema per-environment limits.
- **`causes.subprocess_run` has now genuinely been exceeded** (353.7s against an allowed
  345.0 in the retained history), which is the corroboration the closed granularity task
  asked for before being reopened.
