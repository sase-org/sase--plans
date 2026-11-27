---
tier: tale
title: Restore measured test-cost budgets and retain CI evidence
size: medium
goal:
  Rebaseline justified suite-growth misses from a fresh full recording, preserve useful
  regression detection, and verify the test-cost gates for sase-1vn.2.
proposed_by: bbugyi200.athena.sase-1vn.2
proposed_with: codex/gpt-6-astra
bead: sase-1vn.2
create_time: 2026-11-27 10:50:23
status: wip
---

- **PARENT:**
  [202611/backlog_epic_1.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_1.md)
- **BEAD:**
  [sase-1vn.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vn/sase-1vn.2.md)

# Restore measured test-cost budgets

Implement the `test-cost` phase of `plan:202611/backlog_epic_1.md` for assigned bead
`sase-1vn.2`. The parent's final decision is `cost_budgets = rebaseline`; neither the
cut nor retire branch applies. No memory edits are authorized or needed.

This is one medium coding task: the existing recorder, budget checker, severity model,
worker controls, and regression tests already exist. The remaining work is a measured
budget update, focused test changes, and a small CI evidence change. Long measurements
continue through `/sase_monitor`; their elapsed time alone does not require an epic. The
phase has no earlier-work notes or abandoned implementation to resume.

## Governing decisions and scope

- `decisions:check-full-is-explicit`: this assigned phase and this plan explicitly
  authorize `just test-cost` and `just check-full`. Run both only through
  `/sase_monitor` with the verify profile and TESTING/TESTED statuses.
- `decisions:guarded-recipes`: use `sase tool run check` and the monitored
  `sase tool run check-full` wrapper for guarded verification.
- `decisions:ci-two-speed-split`: keep cost attribution in CI Telemetry, outside Master
  Gate and Full CI. Preserve the whole fast-suite selection.
- `decisions:triage-annotates-does-not-change-exit-codes`: unrelated failure evidence
  may justify completion with a recorded exception; never make a failing recipe return
  zero or weaken its leak detector.
- `decisions:handoffs-end-through-host-completion` and
  `decisions:host-owned-completion`: stage monitors, wait for the staging command to
  exit, then finish through `/sase_final`. Never wait for a staged successor or commit
  manually.
- `decisions:rust-core-required`: the planned files are Python test infrastructure and
  workflow configuration, not shared application domain behavior. No Rust API or pin
  change is needed.

Do not repair ACE lifecycle leaks, redesign suite admission, add sharding, increase CI
timeouts, or build a general recording-provenance subsystem as part of this task.

## Evidence checked before planning

The workspace starts at master-derived commit `f079628697`, containing the preceding
phase's landing-safety fixes. Budget history still ends at `932e6ffae2` on 2026-09-18.
Read `sase-j0` through `sase bead read` for its complete description and all 13 notes.
Its repeated partial repairs explain why another unmeasured limit increase is not
enough.

The retained full recording `20260919T182806Z-2099141.json` has 43,271 nodes, 3,993
files, four workers, and 3,860.711 total file CPU seconds. Replaying it through current
`check_cost_budgets` still reports these hard misses (actual versus effective
allowance):

| Metric                  |  Actual | Allowance |
| ----------------------- | ------: | --------: |
| `ace_settle_pilot.cpu`  | 538.531 |   525.000 |
| `parser_create.cpu`     |  86.304 |    85.000 |
| `pilot_pause_delay.cpu` | 470.900 |   462.500 |
| `subprocess_run.count`  |  53,787 |    52,000 |

The seven newer local records contain only 59–613 nodes. They are targeted runs, not
calibration samples. `--suggest` currently loads retained records without distinguishing
partial runs, and report/check commands default to the latest shared recording. Use a
fresh isolated `SASE_TEST_COST_DIR` for this task and explicit recording paths for
replay; do not infer a passing full lane from those partial records or mix them into
suggestions.

The original epic's CI timeout diagnosis is stale. In
[Telemetry run 45971108412, test-cost job 140417887999](https://github.com/sase-org/sase/actions/runs/45971108412/job/140417887999),
on `6b58ad0fe2c05ab1b6cd4fddf3e91ac2224ef85e`, Python 3.13.18 used three workers. Pytest
completed with 63,645 passed and 140 skipped in 7,306.11 seconds (2h 1m 46s), within the
180-minute job cap. The workflow-level cancelled status came from another job;
`test-cost` itself failed at the global leak gate. Its three reported nodes leaked
`sase.config.core._live_config_token_refresh_threads`, the existing `sase-1vg` issue.
The recipe stopped before running either cost reporting or budget evaluation. This run
proves current runtime headroom, not passing budgets.

The existing behavior is already mostly correct:

- `tests/_test_cost_budgets.py` makes wall-clock overages advisory. Collection wall and
  CPU totals are normalized per worker. Cause invocation counts have no runtime
  tolerance. Local CPU and RSS remain hard; CI CPU is advisory because the calibration
  host differs from GitHub runners, while CI count and RSS remain hard.
- `tests/test_test_cost_budgets.py` covers these rules.
- `tests/test_test_cost_committed_budgets.py` checks every doubled committed metric, CLI
  wall-only versus `--strict` behavior, and historical wall regression evidence.
- `tools/check_test_cost_budgets --suggest` emits provenance and measured ranges, but
  its time-limit formula divides the observed maximum by tolerance before rounding. The
  effective allowance therefore has little spare margin. Count suggestions already
  include 25% headroom. Do not mistake the 25% CPU tolerance for 25% additional headroom
  above the sampled maximum.

## Implementation

### 1. Capture a trustworthy full-suite baseline

Read `lint_and_test.md` and `tools.md` using `/sase_memory_read`. Confirm the current
tree, environment, and phase notes before starting; preserve any newer completed work.

Create a unique, initially empty cost directory under `.pytest_cache/` and pass its
absolute path as `SASE_TEST_COST_DIR` to the monitored `just test-cost` command. Choose
one bounded local worker count using the existing `SASE_PYTEST_WORKERS` setting within
the actual suite-gate budget. Record the granted count, CPU capacity/load, available
memory, Python version, host, git SHA, command, start/end times, and pytest outcome. Use
a quiet interval where possible; never stop unrelated work or bypass admission.

The continuation must check that the recorded node coverage matches the complete fast
suite, all workers reported, and no early stop, worker crash, selected path, shard,
`-k`, or changed marker selection truncated the run. Preserve the raw recording and logs
as explicit artifacts with `sase artifact create`, associated with `sase-1vn.2`, before
retention can remove them. Full-suite node counts are a sanity check, not a hardcoded
new budget. Do not use a 64,000-node threshold as a completeness heuristic.

If the recipe fails after a complete run, distinguish budget failure from functional or
leak failure. Replay the exact JSON with `tools/check_test_cost_budgets --recording` to
inspect costs even if the Justfile stopped before its checker. An unrelated complete-run
failure can supply diagnostic cost evidence, but record its outcome and independently
verify its clean-base signature before calling it unrelated. A crash or partial run
cannot establish a new baseline.

### 2. Classify misses and rebaseline with explicit headroom

Use `tools/test_cost_report`, the per-file/per-cause data, the old full record, and
suite growth to classify each missed key as suite growth, a named regression, or host
contention. Compare CPU per cause invocation and invocation counts where possible;
aggregate test-count growth alone is insufficient to excuse one disproportionately
expensive cause. Fix a specific regression rather than incorporating its cost into a new
limit. Record an out-of-scope finding on this phase as `PROPOSED FOLLOW-UP:` if it
requires unrelated product work; do not silently hide it in the baseline.

Run `--suggest` against only the isolated full-record directory and retain the exact
output. Update `tests/perf/baselines/test_cost_budgets.json` from those measurements:

- Keep the existing metric families, summary gates, enforcement severities, and
  tolerance policy. Rebaseline stale summary and cause limits with measured evidence; do
  not scale every value by the suite's node-count ratio.
- For changed CPU/RSS budgets, use the existing two-significant-figure upward rounding
  and choose `limit = round_up(max_sample / (0.85 * (1 + tolerance)))`. This leaves
  every accepted observation at least 15% below its effective allowance. Retain the
  count formula `round_up(max_count * 1.25)` with zero runtime tolerance. Explain any
  rounding effect explicitly. A doubled healthy observation must remain a regression.
- Keep wall-derived metrics advisory and collection totals per worker. A busy-host wall
  increase alone must not fail either gate. Advisory thresholds may be refreshed from
  the same controlled sample; do not turn suite-wide totals into per-worker means merely
  to suppress growth.
- Add an authoritative dated note with artifact refs, SHA, host/Python, worker and
  node/file counts, per-key old/new values, classification, arithmetic, and resulting
  headroom. Clearly identify the new derivation where historical notes describe older
  suggestion formulas. Do not rewrite historical evidence as if it were current.

One clean full calibration run is required. Request a second only if evidence is
ambiguous or a named regression/noisy sample needs distinguishing. The later
`check-full` run supplies an independent post-change check.

### 3. Keep the gate demonstrably useful and CI evidence available

Update the focused budget tests as needed. Preserve the existing local/CI severity,
per-worker normalization, count tolerance, and `--strict` contracts. Add CLI-level
negative coverage for the committed budgets: a healthy compact fixture derived from the
fresh full sample passes; a doctored hard CPU/count/RSS metric just beyond its effective
allowance exits nonzero. Exercise count/RSS failures in CI mode too. Keep the existing
doubled-metric checks. Do not replace them with assertions that merely repeat the new
JSON numbers.

If refreshed advisory limits invalidate the old pre-epic fixture's exact list of misses,
replace that brittle assertion with explicit healthy-versus-regressed evidence. Keep the
old artifact as historical evidence; do not change its measurements to manufacture a
failure. Avoid committing the full large telemetry payload when a small documented
fixture plus an artifact reference suffices.

In `.github/workflows/telemetry.yml`, set a job-scoped `SASE_TEST_COST_DIR` under
`.pytest_cache/` so cost records have a stable upload location. Add an `if: always()`
artifact upload for that directory, named with SHA and run attempt and with bounded
retention, alongside the existing global-leak report. Use an explicit non-hidden
directory path or the upload action's hidden-file option so `.pytest_cache` ancestry
does not silently discard the evidence. Retain the exact `just test-cost` invocation,
the existing worker admission, the 180-minute cap, and all blocking failure behavior.
Extend the workflow contract tests in `tests/_github_actions_ci_master_gate_full.py` and
their `tests/test_github_actions_ci_master_gate.py` facade as appropriate to prove the
producer and upload paths agree and failed jobs still upload.

Recheck the current job's own runtime rather than the whole workflow conclusion. If a
new timeout actually reproduces, investigate the recorded worker grant and use existing
bounded worker controls only when evidence supports the change. The current three-worker
run already fits; no speculative tuning or larger timeout is part of this plan. If an
unrelated CI defect remains, record the exact job URL and existing owning bead in a
`PROPOSED FOLLOW-UP:` note rather than suppressing it.

## Verification and completion

1. Run focused budget/record tests and the affected workflow contract tests. Check that
   a doctored recording fails the real CLI, not just its internal helper.
2. Run formatting and `sase tool run check`. Fix failures caused by this work.
3. Through `/sase_monitor`, run `sase tool run check-full` with another unique
   `SASE_TEST_COST_DIR` and the recorded bounded local worker count. Inspect the exact
   new recording against the committed budgets. Read the visual report and inspect any
   golden changes before finalization; do not use prepared completion that skips that
   inspection. `check-full` already includes `just test-cost`, so do not repeat another
   full cost run merely to duplicate the result.
4. The target is a passing `just check-full`, passing local and CI-mode checks of
   representative complete recordings, and no loss of the synthetic regression signal.
   An actual subsequent CI job is stronger evidence than a local `--ci` replay; do not
   describe replay or workflow tests as a green GitHub job. Reuse current full-runtime
   evidence for the timeout requirement unless a code change affects that path.
5. Follow the user's clean-base exception precisely: if a failure reproduces identically
   on the clean base, record a `PROPOSED FOLLOW-UP:` on `sase-1vn.2` with the signature,
   comparison evidence, and existing owner (for example `sase-1vg`). Preserve its
   nonzero result and state that limitation. It does not keep this phase open. Do not
   call a new or unexplained failure unrelated.
6. Append concise implementation and verification evidence to `sase-1vn.2`, including
   the recording refs, revised budgets, regression tests, and any remaining CI limit.
   Run `sase bead epic-symbols sase-1vn.2`; resolve leftovers or re-key them to a
   still-open owning bead before closing.
7. Close **only** `sase-1vn.2` with
   `sase bead close sase-1vn.2 --note "<verified result>"`. Do not manually set its
   status or reopen it. Do not close `sase-j0`, `sase-1vn`, or another ancestor/plan
   bead: the current user instruction narrows the earlier epic's close instructions.
   Supply evidence for the epic land agent to settle the source task later. Do not
   create follow-up beads.
8. Finish through `/sase_final`; the host owns commits and any plan settlement.
