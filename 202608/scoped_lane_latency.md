---
status: done
tier: epic
title: Diff-scoped lane latency — escalate on estimated runtime, not file count
goal: "`just check`'s scoped test stage is never slower than `just check-full` would
  have been, the full-suite escalations it does take are attributable to the agent's own
  diff, and `just selection-health` reports the tail and the flake-free false-negative
  count instead of hiding both.

  "
phases:
  - id: timings
    title: Per-test-file duration table recorded by the full lane
    depends_on: []
    size: medium
    description: "timings: record per-test-file wall seconds from full-lane runs into
      the host-local store and expose an `estimate_serial_seconds()` the selector can
      call, with an explicit no-data answer.

      "
  - id: budget
    title: Escalate on estimated serial runtime, not on the file-count ratio
    depends_on:
      - timings
    size: medium
    description: "budget: add a serial-runtime budget rule so a selection that would
      take longer than the governed full lane escalates, and keep the file-count ratio
      only as a fallback when no timing data exists.

      "
  - id: gear
    title: A bounded-parallelism middle gear for large selections
    depends_on:
      - budget
    size: medium
    description: "gear: let a scoped run that exceeds the serial budget take a small
      non-blocking lease instead of escalating, falling back to serial rather than ever
      queueing.

      "
  - id: identity
    title: Attribute and narrow the core-identity-changed escalation
    depends_on: []
    size: small
    description: "identity: record which fingerprint input changed, and stop forcing the
      whole suite for environment churn that is byte-identical or already covered by
      another rule.

      "
  - id: tail
    title: Report the scoped lane's tail, not just its median
    depends_on: []
    size: small
    description: 'tail: add duration percentiles and a "slower than the full lane"
      counter to `just selection-health` so a latency regression is visible in the
      project''s own health metric.

      '
  - id: flakes
    title: Stop charging known flakes to the false-negative metric
    depends_on: []
    size: small
    description: "flakes: require a full-run failure to be reproducible, or to be absent
      from a known-flake list, before it is correlated against a scoped selection.

      "
  - id: land
    title: Land the scoped-lane latency epic
    depends_on:
      - budget
      - gear
      - identity
      - tail
      - flakes
    size: small
    description:
      "land: re-measure the lane end to end against the real store, verify the combined
      tree, and state honestly which of the epic's numbers moved and which did not."
proposed_by: bbugyi200.athena.ue
bead_id: sase-gj
create_time: 2026-09-09 19:51:30
---

- **PROMPT:**
  [prompts/202608/scoped_lane_latency.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/scoped_lane_latency.md)
- **BEAD:**
  [sase-gj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-gj/README.md)

# Plan: Diff-scoped lane latency

## Context

Epics `sase-fp` (two-speed verification) and `sase-g3` (selection soundness) built the
diff-scoped `just check` lane and proved it **sound**: the backtest reports 100% recall
for closure-plus-contexts on all 31 usable commits, and the static closure alone
blind-spots on only 2. Nothing in this plan disputes that. This epic is about
**latency**, which neither predecessor measured: `sase-fp`'s success metric is
worker-seconds avoided, and `sase-g3`'s is recall. The lane's wall-clock cost to the
agent waiting on it was never a reported number, and it turns out to be the axis where
the lane currently misbehaves.

### What was measured

All figures below come from the real host-local record store
(`${SASE_HOME:-~/.sase}/test-selection/gh_sase-org__sase`, 63 scoped and 86 full-lane
records) plus two timed runs on athena at master `5da193482`, on 2026-08-06.

Ground truth for the comparison:

| Lane                                             | Wall clock |
| ------------------------------------------------ | ---------: |
| lint gates alone (`just check` minus the tests)  |       ~59s |
| `just test` (full, parallel, 28 workers granted) |       232s |
| `just check-full` (lint gates + full suite)      |      ~291s |
| `just check`, clean tree (contract-set floor)    |        83s |

The full run was 26,042 passed / 7 skipped / 227.76s of pytest, with two failures that
are the known ACE-TUI load-sensitive flake family already tracked on `sase-ct`.

Scoped-stage durations across the 39 non-escalated runs in the store:

| percentile |    duration |
| ---------- | ----------: |
| p50        |       37.6s |
| p75        |      104.3s |
| p80        |      354.3s |
| p90        |      435.8s |
| p95        |      626.3s |
| max        | **1032.6s** |

The median case is exactly what the epics promised — a ~97s `just check` against a ~291s
`just check-full`, a 3x win. The tail is the problem. **Eight of those 39 runs took
longer than the 232s full parallel lane**, and those eight consumed 4,138s of the lane's
5,524s total — **75% of all scoped-lane wall clock was spent on the 20% of runs that
would have been faster running everything.** The worst, a 494-file selection seeded by
five `src/sase/ace/tui/**` files, ran 1,032.6s serially; the same agent could have had
`just check-full` in ~291s. `tools/run_pytest` states the intent in a comment — _"Never
let the fast path become the slow path"_ — and the measurement says the guard
implementing that intent does not work.

### Why the guard does not work

`SASE_TEST_SELECTION_MAX_RATIO` escalates above 0.25 of all test files. File count is a
poor proxy for serial runtime:

| selected files | share of suite | serial duration | seconds/file |
| -------------: | -------------: | --------------: | -----------: |
|             94 |           4.0% |          465.7s |     **4.95** |
|            517 |          22.0% |          404.0s |     **0.78** |

A 6.3x spread. `tests/ace/*` is 750 of 2,349 test files (31.9%) and is
disproportionately slow, so the selections that blow the budget are precisely the ones
the ratio rates as cheap. 94 files — 4% of the suite, comfortably "scoped" — cost twice
a full parallel run.

### The second cost: escalations that are not about the diff

24 of 63 scoped runs (38.1%) escalated. The dominant trigger is `core-identity-changed`,
present in 16 and the **sole** reason in 8:

```
  8  core-identity-changed
  4  justfile + selection-tooling
  3  core-identity-changed + packaging-config
  3  core-identity-changed + justfile + selection-tooling
  2  selection-tooling
  1  core-identity-changed + selection-tooling
  1  core-identity-changed + root-conftest
  1  packaging-config
  1  justfile
```

`core_identity_changed` compares `environment_fingerprint()` against the previous
manifest's. That fingerprint is borrowed wholesale from
`tools/validate_test_environment._input_fingerprint`, which exists to invalidate a
**validator verdict cache** — a cheap thing to redo. Reusing one opaque digest as a
**whole-suite trigger** means any of its inputs forces 232s of tests: `pyproject.toml`,
`uv.lock`, `pyvenv.cfg`, the four validator scripts, the sibling `sase-core/Cargo.toml`,
every installed distribution's `METADATA`/`direct_url.json`/`entry_points.txt`/`*.pth`,
and a `stat()` of `bin/python`. The manifest records only the digest, so **no run in the
store can say which input changed**, and two of those inputs (`pyproject.toml`,
`uv.lock`) already have their own `packaging-config` rule when they are in the diff.

Two incidental facts found while reading it, both worth fixing here:

- `_STATED_EXTENSION_PATTERNS` globs `site-packages/sase_core_rs*.so`, but the extension
  actually lives at `site-packages/sase_core_rs/sase_core_rs.abi3.so`. Globs do not
  cross `/`, so the list is empty and the docstring's claim to digest "the
  `sase_core_rs` identity" is not met by that input at all. A rebuild is caught only
  indirectly, via the dist-info `METADATA` version.
- Because that stat-based input is dead, the digest is content-based in practice, which
  is the good news: a verified no-op `just install` does **not** flip it. Narrowing this
  rule is therefore about attribution and scope, not about chasing mtime noise.

### What the health metric hides

`just selection-health` reports `median duration` and nothing else about time — no p90,
no max, no comparison against the full lane. The lane's worst behaviour is invisible in
the project's own health report. The metric it does lead with, 136,826 worker-seconds
avoided (~38 core-hours in one day), is real and is **not** disputed here: on a host
running eight numbered workspaces against a memory-adaptive gate budget, avoiding host
demand is a genuine throughput win. It is simply a different quantity from the latency
an agent waits on, and only the first one is currently reported.

The false-negative metric has a separate problem, observed live during this research. It
stood at 1. Running one timed `just test` for the table above took it to **3**, because
that run hit two `sase-ct` ACE-TUI flakes. All three entries in the store are flakes;
none is a selection miss. The report nonetheless prints:

```
A non-zero count means the selection heuristic is unsound as tuned.
Raise SASE_TEST_SELECTION_DEPTH to 3 or add the missed tests to
tests/contract_manifest.txt, then re-measure.
```

That remedy is wrong for all three, and `sase-g3.5` already had to hand-filter the same
way in its landing note. The epics' shared exit criterion — zero false negatives over
30+ varied changes — cannot be reached while a flaky suite injects phantom entries
faster than genuine samples accumulate.

### The size of the prize

Charging every scoped run at its recorded duration and every escalated run at the 232s
full lane (a floor — escalations also queue for a gate lease, which is not counted):

| scenario                                                           | test-stage wall clock | saved vs. all-full |
| ------------------------------------------------------------------ | --------------------: | -----------------: |
| all 63 runs on the full lane                                       |               14,616s |                  — |
| **today**                                                          |           **11,092s** |            **24%** |
| \+ the 8 over-budget runs escalate (`budget`)                      |                8,810s |                40% |
| \+ the 8 sole-`core-identity` escalations stay scoped (`identity`) |                7,274s |            **50%** |

`gear` improves on the `budget` row rather than adding to it: at 4 workers the 494-file
selection is ~258s and at 8 it is ~130s, so a bounded lease beats both the 1,032.6s
serial run and the 232s escalation.

## Phases

### timings: Per-test-file duration table recorded by the full lane

Selection has no idea what anything costs. Give it one.

- Add a duration-recording sink to the existing full-lane pytest plugin (the one
  `_full_lane_recording_args` already installs for `FullRunFailureRecorder`),
  aggregating per-test wall seconds up to the **test file**. One `just test` covers all
  2,349 files in a single pass, so the table bootstraps from any full run an agent or CI
  already does; no new lane, no new CI job.
- Persist it host-locally next to the existing selection stores, keyed by nothing more
  than the recording host — this is a cost model, not ground truth, and it does not need
  per-commit identity. Merge successive recordings so a newer run refreshes files it
  covered without discarding files it did not. Keep the newest few and prune like the
  contexts cache.
- Expose `estimate_serial_seconds(paths)` from a new `tests/_test_selection_timings.py`.
  It must return an explicit "insufficient data" answer, not a guess, when the table is
  missing or covers too small a fraction of the selection; callers decide what to do
  about that, and `budget` below does something specific.
- Record on the manifest: the estimate, the table's SHA/mtime identity, and the coverage
  fraction. A scoped run that finishes then contributes its own real duration back, so
  the model self-corrects for the files the lane actually touches most.
- Tests belong in a new `tests/test_test_selection_timings.py`; watch the repo's
  per-file line budget and split `tests/_test_selection*.py` further if a module grows
  past it, as the existing split already anticipates.

Do **not** wire the estimate into any decision in this phase. It lands measured and
inert.

### budget: Escalate on estimated serial runtime, not on the file-count ratio

- Add `RULE_SERIAL_BUDGET_EXCEEDED` to `tests/_test_selection_rules.py` and to
  `FULL_SUITE_RULES`. It fires when `estimate_serial_seconds(selection)` exceeds
  `SASE_TEST_SELECTION_MAX_SERIAL_SECONDS`.
- Default that knob from the crossover the epic measured, not from a round number: the
  scoped stage is worth running only while it is cheaper than the full lane, so the
  default is the full lane's wall clock. Put the constant next to
  `FULL_SUITE_WORKER_SECONDS` in `tests/_test_selection_health.py`, with the same
  "measured, deliberately constant" treatment and the 2026-08-06 measurement (232s wall,
  28 workers, 26,042 tests) in its docstring.
- Keep `SASE_TEST_SELECTION_MAX_RATIO` as the fallback for the no-timing-data case, and
  say so in the rule's docstring. A fresh host with no table must keep working exactly
  as it does today.
- The manifest and `tools/select_tests --explain` must show the estimate and the budget
  whenever the rule is evaluated, including when it does not fire. An agent looking at a
  400-file selection should be able to see "estimated 180s, budget 232s" and understand
  why it stayed scoped.
- Re-run `just selection-backtest --include-descendant-baseline` afterwards and report
  what the new rule does to the replayed escalation rate. A runtime budget should
  escalate **fewer** total runs than the ratio does once `identity` lands, but it will
  escalate a different set; the backtest is how that is stated rather than assumed.

### gear: A bounded-parallelism middle gear for large selections

The lane has two gears — 1 worker with no lease, or the full governed suite — and
nothing between. A selection that is too big to run serially but far smaller than the
suite has no good option.

- When the serial estimate exceeds the budget, try a **bounded, non-blocking** lease
  before escalating: request at most `SASE_TEST_SELECTION_SCOPED_WORKER_CEILING`
  (default small, 4) tokens with a zero or near-zero timeout. If the gate grants them,
  run the selection at that width. If it does not, fall through to escalation. The
  lane's defining promise — it never queues behind another agent's run — must survive
  verbatim, so a lease that is not immediately available is simply not taken.
- Release the lease in the existing `finally`. Scoped mode is already the one mode that
  keeps a parent process alive rather than `execv`-ing, so the lifetime is
  straightforward; the current `SASE_TEST_GATE_DISABLED=1` belt-and-braces for
  subprocess-spawning tests needs revisiting once workers are real.
- `_reject_scoped_worker_overrides` currently rejects `-n` and `SASE_PYTEST_WORKERS`
  with a message asserting scoped runs are serial. That message becomes false. Keep
  rejecting the overrides — the width is the gate's decision, not the caller's — and
  rewrite the reason.
- Record the granted width and whether the lease was refused on the manifest, and
  surface both in the scoped summary line. `tail` reads them.
- The `duration` recorded for a parallel scoped run is wall clock at width N and is no
  longer comparable to a serial one. Record the width alongside it and make
  `estimate_serial_seconds`'s feedback path normalize, or discard, non-serial samples
  rather than poisoning the table.

### identity: Attribute and narrow the core-identity-changed escalation

- Replace the single opaque digest with a small **per-input** fingerprint map, and
  record on the manifest **which** inputs differ from the previous run. The escalation
  stops being unattributable; `just selection-health`'s histogram can then break
  `core-identity-changed` down by cause.
- Fix `_STATED_EXTENSION_PATTERNS` so it actually finds
  `sase_core_rs/sase_core_rs.abi3.so`, and hash its **content** rather than `stat()`-ing
  it. A rebuild that produces identical bytes is not a change; the docstring's claim
  about tracking the extension's identity should become true.
- Narrow what forces the full suite. A changed `pyproject.toml`/`uv.lock` already has
  `packaging-config` when it is in the diff; the environment check exists for the case
  where it arrived by `git pull` instead, which is worth escalating — but a changed
  validator script, or an unrelated third-party distribution's `METADATA`, is not.
  Decide each input on its merits, write the decision into the rule's docstring and
  `docs/development.md`, and state the measured frequency of each cause. Where an input
  does not justify the whole suite, the honest fallback is `contract-set-always` plus
  the normal closure, not silence.
- Keep the digest reuse from `tools/validate_test_environment` where it still fits; do
  not fork a second, divergent fingerprint for the validator cache, which is what the
  original docstring was rightly guarding against.

### tail: Report the scoped lane's tail, not just its median

- Add p75/p90/max scoped duration to `SelectionHealth` and both renderings, beside the
  existing median.
- Add the number that would have caught this epic's defect on day one: **how many scoped
  runs took longer than the full lane's wall clock**, with their selected-file counts
  and the rules that produced them. This is a latency-regression counter and should read
  as one.
- Report escalated runs' cost honestly. They currently record `duration: 0.0`, so a lane
  where every run escalates would report a median of 0s and 136,826 avoided
  worker-seconds. Either record the handed-off full run's real duration, or render
  escalated runs as "cost not measured" — silence that reads as zero is what let the
  tail hide.
- Once `gear` lands, group durations by granted worker width so a 130s run at 8 workers
  is not compared against a 130s serial one.

### flakes: Stop charging known flakes to the false-negative metric

- Before `find_false_negatives` charges a scoped selection with a full-run failure,
  require evidence the failure is real. Two mechanisms, and the phase should pick with a
  stated reason: a maintained known-flake list (`sase-ct`'s umbrella already enumerates
  the family) that excludes matching node IDs and counts them separately, or a
  requirement that the same node have failed in more than one full run at unrelated
  change sets.
- Excluded entries must be **counted and shown**, exactly as pre-schema-2 records
  already are. "3 flake-suppressed" is information; silently dropping them is how a
  metric stops being trustworthy.
- Fix the remedy text. "Raise `SASE_TEST_SELECTION_DEPTH` to 3 or add the missed tests
  to `tests/contract_manifest.txt`" is wrong advice for a flake, and it was wrong for
  all three entries in the store at the time this plan was written.
- Re-run `just selection-health` against the real host store and state the corrected
  count. If it is zero, say it is zero **of a small correlatable sample** — `sase-g3`
  was careful about exactly this and the distinction must not be quietly dropped.

### land: Land the scoped-lane latency epic

- Verify each phase's deliverable in the source rather than trusting its report.
- Re-measure the lane end to end: a fresh `just selection-health` against the real host
  store, a timed `just test` for the current crossover, and
  `just selection-backtest --include-descendant-baseline` for recall. Recall must not
  regress; `budget` and `gear` change only _where_ tests run, never _which_.
- Restate the savings table above with real post-change records rather than the
  projection, and say plainly if the projection was wrong.
- Update `docs/development.md`'s "Diff-scoped checks" section: the escalation trigger,
  the middle gear, the narrowed identity rule, and the new health fields. The doc
  currently states the lane "is serial (`-n 1`) and takes no suite-gate lease" — after
  `gear` the second clause still holds and the first does not.
- `just check-full` on the combined tree, plus `just symvision`.

## Out of scope

- **Breaking the 497-module `src/sase` import cycle** (`sase-fy`, closed as needing its
  own design). This epic makes an imprecise selection cheap to run; it does not make it
  precise. The two are complementary and this one is far smaller.
- **Fixing the ACE-TUI flake family** (`sase-ct`, in progress with 18 corroborations).
  `flakes` stops those failures from corrupting the selection metric; it does not stop
  them failing.
- **Contexts baseline ranking** (`sase-g9`, ready) and **depth-3 re-measurement**
  (`sase-fu`). Both are selection-recall questions, not latency ones.
