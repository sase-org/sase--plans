---
tier: tale
title: Gate suite cost on contention-stable metrics instead of wall clock
goal:
  "`just test-cost` stops failing `just check-full` because another agent was busy on
  the same host: the suite-cost gate hard-fails on CPU seconds, invocation counts, and
  RSS, and reports wall-clock overages as advisories."
size: medium
proposed_by: bbugyi200.athena.0bv
---

# Plan: Gate suite cost on contention-stable metrics instead of wall clock

## Problem

`just check-full` fails at the `test cost` step. `just test-cost` runs the fast suite,
records a cost attribution recording, then runs `tools/check_test_cost_budgets`, which
exits 1:

```text
test cost budget regression: <SASE_HOME>/test-selection/gh_sase-org__sase/timings/cost/20260823T144953Z-3572032.json
budgets: tests/perf/baselines/test_cost_budgets.json
- causes.ace_page_enter: actual 680.949 exceeds budget 540.000 + 20% tolerance (648.000)
- causes.pilot_pause_delay: actual 285.369 exceeds budget 230.000 + 20% tolerance (276.000)
- causes.textual_app_run_test_enter: actual 592.216 exceeds budget 470.000 + 20% tolerance (564.000)
error: recipe `test-cost` failed on line 407 with exit code 1
```

## Root cause

**The suite did not get more expensive. The host got busier, and every failing budget is
a wall-clock total.**

All three failing metrics are `causes.*` entries. A cause is accumulated in
`tests/_test_cost_plugin.py` by `CostRecorder.measure()`, which records
`time.perf_counter()` deltas around a patched call site — wall clock only, no CPU time.
The three that failed wrap the ACE/Textual async paths (`AcePage.__aenter__`,
`App.run_test.__aenter__`, `Pilot.pause(delay)`), which are dominated by event-loop
scheduling and awaited settle work. Those stretch under CPU contention that has nothing
to do with this repository's code.

The eight retained recordings on the development host make this unambiguous. Invocation
counts are flat while per-invocation wall cost rises monotonically with host load:

| recording                              | workers | `ace_page_enter` n | ms/call    | `textual_app_run_test_enter` n | ms/call   | `pilot_pause_delay` n | ms/call   |
| -------------------------------------- | ------- | ------------------ | ---------- | ------------------------------ | --------- | --------------------- | --------- |
| 20260823T014411Z                       | 14      | 661                | 868.6      | 3570                           | 141.3     | 13147                 | 18.11     |
| 20260823T084055Z                       | 14      | 661                | 908.5      | 3570                           | 148.2     | 13805                 | 18.38     |
| 20260823T090219Z                       | 14      | 661                | 906.6      | 3570                           | 144.7     | 13683                 | 18.52     |
| 20260823T100643Z                       | 14      | 661                | 899.9      | 3570                           | 146.6     | 13271                 | 18.93     |
| 20260823T104022Z                       | 14      | 661                | 898.5      | 3570                           | 147.9     | 13551                 | 18.20     |
| 20260823T135648Z                       | 9       | 661                | 927.1      | 3570                           | 152.3     | 13723                 | 19.93     |
| 20260823T143358Z                       | 14      | 661                | 983.7      | 3570                           | 158.6     | 13541                 | 21.55     |
| **20260823T144953Z (the failing run)** | 13      | **672**            | **1013.3** | **3581**                       | **165.4** | **13291**             | **21.47** |

`ace_page_enter` was invoked 661 times in seven consecutive recordings and 672 times in
the failing one. The eleven extra entries come from tests added by commits `3d2065412`
and `2e0ac0f37` and account for roughly 10s of the 141s overage. The other ~130s is
per-call inflation: 868.6 ms/call on a quiet host, 1013.3 ms/call on a busy one, **+17%
for identical work**. Same story for the other two causes (+17% and +19%).

Price the failing recording at the quiet-host per-call rate and every failing budget
passes, including the eleven new AcePage entries:

| cause                        | actual | at quiet per-call rate     | allowed |
| ---------------------------- | ------ | -------------------------- | ------- |
| `ace_page_enter`             | 680.9  | 672 × 0.898 = **603.5**    | 648.0   |
| `textual_app_run_test_enter` | 592.2  | 3581 × 0.1466 = **525.0**  | 564.0   |
| `pilot_pause_delay`          | 285.4  | 13291 × 0.0185 = **245.9** | 276.0   |

The corroborating host evidence: this is a 64-CPU box shared by many concurrent SASE
agents. The last three recordings landed inside a burst of ~15 commits between 13:26Z
and 14:51Z on 2026-08-23; the host's load average shortly after the failing run was
13.48 / 21.05 / 28.05, and the suite gate granted 9 and 13 workers instead of its usual
14 because peers held tokens. The suite gate in `tests/_suite_gate_budget.py` governs
only this project's own pytest workers — it cannot see other agents' builds, lint gates,
ACE sessions, or Rust compiles, so it cannot protect a wall-clock measurement.

### Why the budgets cannot simply be raised

`tests/perf/baselines/test_cost_budgets.json` was calibrated from an eight-recording
sample taken 2026-08-19/20 on a quiet host, and its own notes forbid raising a limit "to
hide a one-off regression." Re-deriving the limits from busy-host recordings would bake
host noise into the gate permanently and destroy what little sensitivity it has. The
whole metric family is the problem, not the numbers.

### The metric that does not move

The same recordings already carry CPU time, and it is dramatically more stable. Deltas
below are against the mean of the five quiet recordings (wall 3809.9s, CPU 1937.2s):

| recording        | `total_file_wall_seconds` | Δ      | `total_file_cpu_seconds` | Δ      |
| ---------------- | ------------------------- | ------ | ------------------------ | ------ |
| 20260823T014411Z | 3725.7                    | −2.2%  | 1901.1                   | −1.9%  |
| 20260823T084055Z | 3783.9                    | −0.7%  | 1953.2                   | +0.8%  |
| 20260823T090219Z | 3590.9                    | −5.7%  | 1940.9                   | +0.2%  |
| 20260823T100643Z | 4186.1                    | +9.9%  | 1948.3                   | +0.6%  |
| 20260823T104022Z | 3763.1                    | −1.2%  | 1942.4                   | +0.3%  |
| 20260823T135648Z | 4481.4                    | +17.6% | 2026.1                   | +4.6%  |
| 20260823T143358Z | 6182.3                    | +62.3% | 2082.8                   | +7.5%  |
| 20260823T144953Z | 5600.6                    | +47.0% | 2138.6                   | +10.4% |

Across the whole range — quiet and heavily contended — wall spans a 68-point band while
CPU spans a 12-point band. Within the quiet band alone wall spans 15.6 points and CPU
spans 2.7. **CPU time is roughly six times more stable, which makes a CPU-based gate
roughly six times more sensitive to a real regression at the same false-failure rate.**
Invocation counts are better still: `ace_page_enter` moved 661 → 672 (+1.7%) across
every recording, and only because tests were genuinely added.

## Approach

Re-aim the gate at what it is actually trying to protect — how much work the suite does
— and stop hard-failing on how fast the box happened to be that minute.

1. **Attribute CPU time per cause**, alongside the wall time already recorded.
2. **Hard-fail on contention-stable metrics**: CPU seconds, invocation counts, and RSS.
3. **Demote wall-derived metrics to advisories**: reported with full detail, never
   fatal, excluded from the exit code.
4. **Keep the calibration workflow honest**: `--suggest` learns the new dimensions so
   limits stay derived from recordings with provenance, never hand-picked.

Two rejected alternatives, recorded so they are not re-litigated:

- _Normalize wall seconds by a measured contention index_ (for example
  `worker_wall_seconds / worker_cpu_seconds`). Rejected on two grounds. The index does
  not separate cleanly — quiet recordings span 1.254–1.316 and contended ones
  1.393–1.488, a gap of under 6% that a classifier cannot be trusted with, and it is
  confounded by worker count because the controller payload's weight changes with it.
  Worse, a genuine regression that adds waiting (a new sleep in a fixture) would raise
  the index and be normalized away — the gate would mask exactly the regression it
  exists to catch.
- _Classify runs as clean/degraded from sampled load average and downgrade the gate on
  degraded runs._ Workable, but it needs a threshold nobody can calibrate today (no
  recording carries load data), it adds platform-specific sampling, and it still leaves
  the gate blind on the busy host where `check-full` actually runs. Measuring CPU
  directly is strictly better: no threshold, no new syscall, no CI/non-CI divergence.

## Work

Read `sase/memory/tui_perf.md` with `/sase_memory_read` before starting; it owns TUI
performance context that the ACE causes touch.

### 1. Record per-cause CPU seconds

`tests/_test_cost_plugin.py`:

- `CostRecorder.measure()` currently times only `time.perf_counter()`. Capture
  `time.process_time()` at enter and exit too, and pass both deltas to
  `_record_cause()`.
- `_record_cause()`, the `causes` property, and `_normalised_files()` gain a
  `cpu_seconds` field next to `count` and `seconds` in both the suite-level and per-file
  cause buckets.

Attribution note to carry into the docs: `time.process_time()` is process-wide, so CPU
burned by other coroutines on the same event loop during an awaited span is attributed
to that span. This matches how wall time already behaves, and within one xdist worker
tests run sequentially, so the attribution stays meaningful. Sleep-dominated causes such
as `pilot_pause_delay` will report near-zero CPU — that is correct and is why counts
matter for them.

`tests/_test_cost_records.py`:

- `_coerce_cause()` defaults `cpu_seconds` to `0.0` and clamps at zero, so the retained
  pre-change recordings still load.
- `_merge_causes()`, `_merged_files()`, and `_merged_worker_causes()` sum and round the
  new field.
- Leave `TEST_COST_SCHEMA` at `1`. The change is purely additive, and bumping it would
  make every retained recording unloadable, emptying the `--suggest` history exactly
  when it is needed.

`tests/_test_cost_records.py` also gains a `_cause_cpu_seconds()` and `_cause_count()`
accessor beside the existing `_cause_seconds()`, for the budget checker and the report
to share.

### 2. Split budgets into hard and advisory

`tests/_test_cost_budgets.py`:

- `CostBudgetFailure` gains a `severity` field, `"hard"` or `"advisory"`, included in
  `format()` output.
- A cause budget entry may now carry `limit` (wall seconds), `cpu_limit` (CPU seconds),
  and `count_limit` (invocations). Each present dimension is checked and reported under
  its own metric label: `causes.<name>`, `causes.<name>.cpu`, `causes.<name>.count`.
- An optional `enforce: "hard" | "advisory"` key on any budget entry sets severity
  explicitly. When absent, severity is derived from the metric family:

  | family | metrics                                                                             | default  |
  | ------ | ----------------------------------------------------------------------------------- | -------- |
  | wall   | `causes.*` `limit`, `total_file_wall_seconds`, `idle_seconds`, `collection_seconds` | advisory |
  | cpu    | `causes.*` `cpu_limit`, `total_file_cpu_seconds`, `collection_cpu_seconds`          | hard     |
  | count  | `causes.*` `count_limit`                                                            | hard     |
  | memory | `peak_worker_rss_kib`, `median_worker_rss_kib`, `post_collection_worker_rss_kib`    | hard     |

  `--suggest` always writes `enforce` explicitly so the committed file never depends on
  the implicit default.

- Tolerance gains a `cpu` key alongside `ci` and `local`. CPU metrics use it; count
  metrics use **no** tolerance — a count limit is compared directly, because counts are
  near-deterministic and the headroom belongs in the limit itself (see §3).
- Per-worker normalization (`worker_divisor`, the `per_worker` flag) is unchanged and
  applies to `collection_cpu_seconds` exactly as it does today.

`tools/check_test_cost_budgets`:

- Print hard failures and advisories in separate labelled sections. Exit `1` only when a
  hard failure exists; exit `0` with the advisory section printed otherwise. The
  advisory section must say plainly that wall-clock overages usually mean the host was
  busy, and name the CPU and count figures for the same metric so a reader can tell the
  two apart at a glance.
- Add `--strict` to make advisories fatal, for deliberate investigation on a quiet host.
- Add `--report-advisories`, which re-reads the newest recording, prints only the
  advisory section, and always exits `0`.
- Extend `SUGGESTED_SUMMARY_METRICS` and the cause-suggestion loop to emit `cpu_limit`,
  `count_limit`, and `enforce` for every cause, and keep printing per-metric
  min/median/max plus the provenance notes.

### 3. Recalibrate the committed budgets

`tests/perf/baselines/test_cost_budgets.json`, in two passes.

**Pass 1 needs no new suite runs.** Counts, `total_file_cpu_seconds`, and
`collection_cpu_seconds` are already in every retained recording. Run
`tools/check_test_cost_budgets --suggest` and commit:

- `count_limit` for every cause that has a wall `limit` today. Count limits use the
  policy `round_up_nice(max_observed_count × 1.25)` with no runtime tolerance — roughly
  25% headroom for organic test growth, which trips on a refactor that doubles a call
  site but not on a handful of new tests. For `ace_page_enter` (max observed 672) that
  lands near 840.
- `total_file_cpu_seconds` and `collection_cpu_seconds` hard limits.
- `enforce: "advisory"` on every wall entry, keeping the existing wall limits as
  advisory thresholds rather than deleting them — an advisory that still fires on a
  quiet host is useful signal.

**Pass 2 needs fresh recordings**, because per-cause CPU does not exist in the retained
history. After §1 lands, take at least two recordings and derive `cpu_limit` for each
budgeted cause from them. A full cost run takes roughly an hour, so run it through
`/sase_monitor`, never inline:

```bash
sase monitor start --command 'just test-cost' \
  --start-status TESTING --stop-status TESTED \
  --next 'Run tools/check_test_cost_budgets --suggest and commit the derived cpu_limit values.'
```

Acceptance criteria for the committed CPU limits, to be checked and stated in the budget
file's `notes`:

- Every quiet-band observation sits at least 15% below the effective allowance
  (`limit × (1 + cpu tolerance)`), so ordinary contention cannot trip it.
- No limit admits a 1.5× regression of the quiet-band median.
- Start with `"cpu": 0.25` in `tolerance`. Observed contention pushed suite CPU +10.4%;
  25% is deliberately conservative for the first cycle. Record in `notes` that it should
  be tightened once ten or more recordings carry per-cause CPU.

Update the `notes` array with the new provenance, the count-limit policy, the severity
model, and the reason wall limits are advisory.

### 4. Stop the ambient `CI` variable from changing the tolerance

`tools/check_test_cost_budgets` defaults `--ci` to `os.environ.get("CI") == "true"`.
Nothing in this repository sets `CI`, but the failing run reported the 20% CI tolerance
rather than the 15% local one, so an agent runtime exported `CI=true` into the monitor's
environment. The effective tolerance therefore depends on which agent runtime happens to
launch `just check-full` — which contradicts the "Uniform Agent Runtimes" convention in
`CLAUDE.md`.

Default `--ci` to `os.environ.get("GITHUB_ACTIONS") == "true"` instead. GitHub Actions
always sets it and nothing else does, so CI behavior is unchanged while local runs stop
silently inheriting a runtime's environment. Keep the explicit `--ci` flag. Note this in
the runbook.

### 5. Surface advisories through `check-full`

`tools/run_silent` discards captured output on success, so an advisory printed by a
now-exit-0 budget check inside `tools/run_silent "test cost" just test-cost` would never
reach the operator. Mirror the existing `tools/probe_core_floor --advisory` precedent:
in the `check-full` recipe, follow the wrapped `just test-cost` line with an unwrapped

```
@{{ venv_bin }}/python tools/check_test_cost_budgets --report-advisories
```

It re-reads the recording that `just test-cost` just wrote, so it costs milliseconds and
does not re-run anything.

Also give the `test-cost-budget` recipe `[positional-arguments]` and `*args`
pass-through so `just test-cost-budget --strict` works.

### 6. Tests

`tests/test_test_cost.py`:

- `CostRecorder` records non-zero `cpu_seconds` for a CPU-bound cause and near-zero for
  a sleep-bound one; the field survives the worker-payload round trip through
  `build_cost_record()` into both `summary.causes` and `files.*.causes`.
- A pre-change recording with no `cpu_seconds` still loads and merges, defaulting to
  `0.0`.
- `check_cost_budgets()` classifies each severity correctly, including the family
  defaults and an explicit `enforce` override.
- A record that blows a wall limit but meets its CPU and count limits produces
  advisories only, and `tools/check_test_cost_budgets` exits `0` for it. Build this case
  from the real failing numbers above (`ace_page_enter` 680.949 with count 672) so the
  reported bug has a direct regression test.
- `--strict` makes that same record exit `1`.
- Count budgets compare without tolerance; CPU budgets use the `cpu` tolerance.
- Extend `test_every_committed_budget_flags_a_doubled_metric` to cover `cpu_limit` and
  `count_limit` so a typo'd new dimension is a test failure rather than a silent no-op,
  and assert the expected severity for each.
- Update `test_committed_pre_epic_baseline_still_fails_recalibrated_budgets`: the
  committed pre-epic baseline predates per-cause CPU, so it must still name
  `causes.parser_create` and `causes.yaml_load` as advisories, with the hard list
  documented as empty for that reason.
- `--suggest` output still round-trips through `load_cost_budgets()` with the new keys
  present.

### 7. Docs

`docs/perf_runbook.md`, "Suite test-cost gate" section: replace the
per-worker/suite-wide framing with the severity model (which metrics bite and which
advise, and why), the count-limit policy, the CPU attribution note from §1, the
`--strict` and `--report-advisories` flags, and the `GITHUB_ACTIONS` tolerance rule.
State plainly that a wall advisory on the shared development host usually means a
neighbouring agent was busy, and that the way to confirm a suspected real regression is
a recording taken when the host is quiet.

Add a `CHANGELOG.md` entry if `just _lint-changelog` requires one for this change.

## Verification

- `just install` first — workspaces are ephemeral and dependencies may be stale.
- `just check` for the lint gates and the diff-scoped test lane.
- `tools/check_test_cost_budgets --recording <the failing recording>` must exit `0` with
  three advisories and no hard failures. That recording is retained under
  `${SASE_HOME:-~/.sase}/test-selection/gh_sase-org__sase/timings/cost/`; if pruning has
  removed it, the equivalent synthetic case from §6 covers the same assertion.
- `just check-full` through `/sase_monitor`, and confirm the advisory section actually
  appears in its output rather than being swallowed by `tools/run_silent`.

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next '...'
```

## Out of scope

- The unrelated CI failures on `master`. Every recent completed `ci.yml` run on `master`
  is red across `test (3.12)`, `test (3.13)`, `test (3.14)`, `lint`, `visual-test`, and
  `coverage-contexts`, with the 3.13 leg failing on four `xprompt` parser-parity
  snapshot mismatches (`default_child` / `mutex_groups` keys) well before the budget
  check runs. That is a separate Rust-core parity problem and belongs in its own task
  bead.
- The `parser_create` cause dropping from ~60s to ~33s between the 104022Z and 135648Z
  recordings. It is a genuine improvement, not a regression, and needs no action here
  beyond letting the next `--suggest` cycle see it.
- Changing the suite gate's worker budgeting in `tests/_suite_gate_budget.py`. It
  governs this project's own workers correctly; it is simply not the right instrument
  for foreign host load.
