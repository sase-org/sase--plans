---
tier: tale
title: Remove test-cost snapshot overhead without weakening leak detection
goal:
  Make the complete test-cost recipe finish within its existing CI timeout while
  preserving leak detection, attribution, and budgets.
size: medium
proposed_by: bbugyi200.athena.sase-1rq.2.1.3
proposed_with: codex/gpt-6-astra
bead: sase-1rq.2.1.3
create_time: 2026-11-07 10:51:28
status: wip
---

- **PARENT:**
  [202611/sase_ci_green.md](https://github.com/sase-org/sase--plans/blob/main/202611/sase_ci_green.md)
- **BEAD:**
  [sase-1rq.2.1.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.2.1.3.md)

# Make the test-cost lane finish without weakening its gates

Implement assigned phase `sase-1rq.2.1.3` of `plan:202611/sase_ci_green.md`. The phase
is already in progress. It had no earlier work notes when inspected, and the relevant
hot path is still present at checkout HEAD `ef00c5097153bd36b318543b5089725be3706577`.
This is one bounded coding task: reduce the leak detector's repeated observation work,
prove equivalent detection, and validate the cost recipe, including its report and
budget checks.

## Contract

The approved epic and `plan:202611/red_ci_repair.md` retain these final decisions:
`test_quarantine = quarantine_flakes` and `release_merge = gate`. Keep the 180-minute
job timeout, three-worker CI shape, budgets, enforcement severities, leak flags,
measured causes, and test selection unchanged. Do not edit memory. Do not dispatch
`release.yml`, merge release PRs, or change other telemetry jobs. No production backend
feature is needed: this work belongs in Python's pytest instrumentation. If evidence
instead requires shared product behavior, follow the Rust core boundary and the epic's
published-core-pin constraint.

Read the assigned bead and its latest notes before implementation. Read both parent
plans through `sase artifact read`. Follow `lint_and_test`, and read `tools` memory
before editing anything under `tools/`. Use the other applicable memory notes if an
actual leak repair reaches their areas.

## Evidence and implementation map

- CI Telemetry run
  [42347892210](https://github.com/sase-org/sase/actions/runs/42347892210), job
  [128332909273](https://github.com/sase-org/sase/actions/runs/42347892210/job/128332909273),
  started its cost step at 06:54:45 UTC on November 7. It ran 60,399 items on three
  workers, reached 93%, and was cancelled at 09:54:35. The newer run
  [42377692789](https://github.com/sase-org/sase/actions/runs/42377692789), at
  `0ba9cfc9`, still had its cost job running when inspected. Recheck it first.
- `.github/workflows/telemetry.yml` runs Python 3.13 and `just test-cost`. The Justfile
  runs `tools/run_pytest cost`, `tools/test_cost_report`, then
  `tools/check_test_cost_budgets`. A failing pytest step prevents both tail steps.
- `tools/run_pytest` adds the cost recorder and the global leak detector, including both
  `--sase-detect-global-leaks` and `--sase-fail-on-global-leaks`.
- `tests/_global_state_leak_detector.py` takes a fresh `_snapshot()` immediately before
  and after every test protocol. Worker reports are written at session finish, so there
  is no per-test report write to batch in this implementation.
- `tests/_global_state_leaks/fingerprints.py` walks every attribute of every loaded SASE
  module on each snapshot, tests its private-global name, and probes every value for
  `cache_info` and `cache_clear`. It serializes supported private globals and captures
  environment, `sys.path`, cwd, and live config refresh threads. `diffing.py` classifies
  poisoning versus warming, cooling, and invalidation.
- A read-only probe importing `tests.conftest` and `sase.ace.tui.app.AceApp` on Python
  3.14.7 loaded 2,811 SASE modules with 88,294 attribute bindings and 48,860 distinct
  objects. Three unprofiled batches of 20 snapshots took 1.323, 1.173, and 1.109
  seconds: approximately 55–66 ms per snapshot.
- cProfile over 20 snapshots recorded 14.93 million calls and 4.45 seconds wall time.
  `_cache_fingerprint` ran 1,765,880 times (1.704 seconds cumulative),
  `_is_private_global_name` also ran 1,765,880 times (0.886 seconds), and
  `_global_fingerprint` ran 280,840 times (0.769 seconds). The module enumeration itself
  took only 0.075 seconds. These are hotspot leads, not a measured CI speedup; the
  interpreter differs from CI and cProfile inflates timings.
- The cost recorder's other hot path is `CostRecorder.measure()` in
  `tests/_test_cost_plugin.py`, which defines a measurement class per invocation.
  `tests/_test_cost_plugin_patches.py` installs cause wrappers. Change these only if the
  matched measurements show material remaining overhead.

## 1. Measure comparable configurations

Choose and freeze a representative subset of a few thousand existing items, including
TUI and non-TUI work. Start with manageable modules under `tests/ace/tui` plus
`tests/history`; record exact selectors and collected counts. Compare four
configurations on the same checkout, interpreter, and worker count: plain fast mode,
recorder only, detector only, and ordinary cost mode with both. Use
`SASE_PYTEST_WORKERS=3` and the governed `tools/run_pytest` launcher. Construct the
diagnostic single-plugin arms with the existing `-p` options and recorder request
environment; loading the recorder module without `SASE_TEST_COST_RECORD` does not enable
measurement. Confirm which plugins are active in each arm.

Give each recording/report a separate scratch location using existing overrides; do not
let subset measurements overwrite or masquerade as a full-suite baseline. Record wall
and CPU time, counts, interpreter, worker count, and failures. Use unprofiled timings
for comparisons and profile one worker to locate overhead. Run arms sequentially. Use
`/sase_monitor` before starting commands that can outlast the turn; whole-suite commands
always need a realistic nonzero weight, with at most one heavy lane for this phase at a
time.

## 2. Remove redundant observation work

Optimize the measured snapshot path first. Keep its public fingerprint/report shape and
classification rules. Prefer bounded name classification caching, cheap cache-protocol
rejection for types provably unable to expose that protocol, short-circuiting failed
probes, and avoiding repeated fingerprint work for aliases within a single snapshot
where observational equivalence holds. Preserve every qualified binding in the snapshot
even when the underlying object is shared. Use the smallest combination that produces a
substantial measured improvement.

Any cached candidate discovery must still observe added, removed, replaced, and reloaded
modules, rebound attributes, newly attached cache APIs, and in-place container edits.
Never reuse mutable fingerprints across tests merely because identity or length is
unchanged. Nested mutable values can change their repr without either changing. Do not
cache all module candidates at collection time, sample only some tests, skip modules,
narrow watched globals, or reuse the previous test's after-snapshot as the next
before-snapshot. Cache observations and live-thread/environment state must remain fresh
at both test boundaries.

If the recorder remains material, replace per-call measurement class creation with a
reusable helper while preserving sync/async behavior, exceptions, ContextVar file
attribution, subprocess double-count prevention, and CPU/wall cause metrics. Do not
optimize by omitting causes or relabeling their cost.

## 3. Prove detection and attribution still work

Extend focused tests under `tests/` as needed. Existing anchors are
`test_global_state_leak_detector.py`, `test_test_cost_plugin.py`,
`test_test_cost_records.py`, `test_test_cost_budgets.py`, and
`test_run_pytest_command.py`. Exercise the real snapshot path, not only fabricated
fingerprint pairs:

- A private dict/list changes in place at the same size, including a nested mutable
  value; rebinding, deleting, and changing the tracked kind remain observable.
- A module or tracked attribute appears after an earlier snapshot; a module is replaced
  under the same name. Aliased state still has every qualified name.
- Real LRU caches and supported custom cache APIs are observed after warming, clearing,
  replacement, or attachment. Dynamic objects must not be silently lost because a cached
  negative classification became stale.
- Environment secrets stay hashed, environment ignore rules remain intact, and cwd,
  `sys.path`, and leftover config refresh threads retain their behavior.
- A subprocess run containing an intentional poisoning mutation still exits nonzero with
  `--sase-fail-on-global-leaks`, while report-only mode preserves its existing outcome.
  A clean run passes. Cover controller and xdist aggregation.

Use meaningful behavioral regressions for the optimizations actually selected. Keep
performance measurements reproducible without a flaky wall-clock threshold in ordinary
unit tests; a deterministic work-count assertion may protect removal of redundant work
where useful. Existing recorder attribution tests must pass.

## 4. Validate the complete recipe

Repeat the matched subset measurements after the fix. Report the before/after ratios and
the contribution of each plugin. The goal is a comfortable margin inside 180 minutes,
ideally about 120 minutes or less on the CI runner. Use the epic's approximately
61-minute Full CI baseline only as an estimate; it uses a different interpreter and
coverage configuration, so do not claim CI success from that extrapolation.

Run a full `just test-cost` through `/sase_monitor` after subset validation. Keep the
existing worker policy and select three workers when making the CI-width comparison.
Preserve the exact full-run recording and leak report before another run can replace
them. Confirm pytest, the report, and the budget check each complete. If pytest fails,
also invoke the two existing tail scripts against that exact recording to expose all
failures rather than only the first recipe exit.

The budget tool already uses `GITHUB_ACTIONS=true` to select its CI behavior; CPU
overages are advisory there, while count and RSS gates remain hard. Report local and
CI-mode results accurately, using the existing `--ci` option on the same full recording
where appropriate. Do not change this policy or fake a successful recipe by checking a
smaller or older recording.

Fix causal leaks or avoidable repeated work found by the full run. Never raise budgets
or add broad leak exclusions. For an unchanged-tree flake, follow the epic's existing
quarantine rules and record the evidence. For unrelated failures that reproduce
identically on the clean base, or independent families beyond the epic's bounded phase
scope, append one `PROPOSED FOLLOW-UP:` note to this bead per family, with exact
nodes/metrics, base evidence, existing task IDs if any, and run/artifact links. Do not
create task beads. Legitimate suite-growth breaches still need their exact
actual/allowed values recorded and budgets left unchanged.

## 5. Verify, land, and finish this phase

Run `sase tool run check` after changes and resolve new failures. Use the project's
monitor/finalizer workflow for long verification. No screenshot updates are expected.
Record the implementation, measured speedup, full-run counts, leak result, report/budget
outcomes, and any unresolved baseline failures in this bead. The containing epic's
verify phase owns the combined CI Telemetry dispatch and workflow-green evidence; give
it the landed fix and complete local evidence. Do not claim the CI runtime target as
verified until a suitable CI run proves it.

Before completion, rerun `sase bead epic-symbols sase-1rq.2.1.3`; there were no entries
at planning time, but resolve or re-key any later entries before closing. Close only
`sase-1rq.2.1.3` with `sase bead close ... --note` once its scope is verified, applying
the user's explicit clean-base-failure exception and the approved epic's
bounded-remainder rule. Never reopen it or close ancestors. Land changes through
`/sase_final` and host completion, never manual git commits or pushes. Keep the bead
open during intermediate verification handoffs.
