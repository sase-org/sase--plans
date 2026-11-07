---
tier: epic
title: Make the default contention lane finish and pass
goal:
  Complete sase-1rq.2.1.4 by identifying and fixing the contention runner shutdown,
  repairing the failures exposed by a complete run, and recording a successful
  contention-test job on landed code without relaxing its workload or gates.
phases:
  - id: evidence
    title: Capture resource and progress evidence for the runner shutdown
    size: medium
    depends_on: []
    description:
      "evidence: Add bounded, outcome-neutral contention diagnostics and obtain a
      monitored reproduction that distinguishes collection pressure, runtime leaks, disk
      exhaustion, and process termination. Publish the measurements and the narrowest
      supported failure mechanism for the survival phase."
  - id: survival
    title: Fix the measured cause of contention runner death
    size: medium
    depends_on:
      - evidence
    description:
      "survival: Use the evidence phase's measurements to fix the dominant resource or
      process-lifecycle defect. Keep the full default selection, 26 workers, two pinned
      CPUs, one CI repeat, and 90-minute timeout. Produce a completed full repeat and
      failure inventory with before/after resource evidence."
  - id: failures
    title: Repair the failures exposed by the completed contention repeat
    size: medium
    depends_on:
      - survival
    description:
      "failures: Group the complete repeat's failing nodes by cause, repair timing and
      isolation defects, and quarantine only unchanged-tree proven flakes under the
      inherited decision. Verify on affected subsets, then run the full unchanged
      contention shape. Record every fix and any authorized remainder."
  - id: confirmation
    title: Confirm the contention job on landed master code
    size: small
    depends_on:
      - failures
    description:
      "confirmation: Confirm landed fix ancestry, reuse or dispatch a qualifying CI
      Telemetry run, and watch it through a zero-weight monitor. Record the
      contention-test result, complete tally, elapsed time, resource evidence, run
      attempt, and quarantine evidence for this child epic's land agent."
proposed_by: bbugyi200.athena.sase-1rq.2.1.4
proposed_with: codex/gpt-6-astra
parent_bead: sase-1rq.2.1.4
create_time: 2026-11-07 11:03:21
status: wip
---

- **PROMPT:**
  [prompts/202611/contention_repair.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/contention_repair.md)
- **PARENT:**
  [202611/sase_ci_green.md](https://github.com/sase-org/sase--plans/blob/main/202611/sase_ci_green.md)

# Make the default contention lane finish and pass

This is a child epic for phase `sase-1rq.2.1.4` of `plan:202611/sase_ci_green.md`. Its
inherited contract is `plan:202611/red_ci_repair.md`. Read both with
`sase artifact read` before working. The repository is the current sase workspace
checkout, GitHub `sase-org/sase`, default branch `master`.

The phases are sequential because each produces evidence needed to define the next
repair. Diagnostics, runner survival, test repairs, and hosted confirmation are separate
deliverables. The root cause and number of failing test families are not yet known, so
treating all of them as one medium tale would hide that uncertainty. Each coding phase
has a bounded deliverable; use the inherited remainder rule below if the actual findings
exceed one coder's capacity.

## Verified starting point

- The assigned bead is `in_progress`, with no previous work notes or recorded remainder.
  The initial source tree was clean at `b7ca9f3450`.
- Local history shows no recent contention repair: the recorder and its contract tests
  last changed in `2bac5ad9e2`; the workflow's latest changes renamed install recipes
  and split telemetry from Full CI. Recent runner changes concern scratch cleanup and
  demand recording. Recheck landed master and phase notes before implementing;
  concurrent work may supersede these observations.
- At planning time the latest Telemetry run was
  [42377692789](https://github.com/sase-org/sase/actions/runs/42377692789), SHA
  `0ba9cfc9f1ac2696cb6faeed515cd98483bcac3b`. Its
  [contention job 128421280632](https://github.com/sase-org/sase/actions/runs/42377692789/job/128421280632)
  had failed while other work in the workflow was still running.
- The log prints `contention soak: 1 repeat(s), 26 workers, dist=worksteal` at 13:48:36
  UTC on 2026-11-07, followed by worker startup warnings through 13:48:57. At 14:09:11
  it reports SIGTERM and a runner shutdown signal. The command ran for about 20 minutes
  35 seconds, far short of its 90-minute job limit. No test tally appears. The
  failure-record upload was skipped; the run artifact inventory then contained only
  `sase-core-wheel`.
- That proves termination, not OOM. Memory pressure, disk exhaustion, and erroneous
  process termination remain hypotheses. Buffered pytest stdout also means the absence
  of test progress is not proof that no tests ran.

## Fixed contract and coordination

The inherited decisions are final: `test_quarantine = quarantine_flakes` and
`release_merge = gate`. There are no new decisions or memory edits in this plan.

- Keep the default non-slow, non-visual test selection, 26 xdist workers,
  `taskset -c 0,1`, `SASE_CONTENTION_REPEAT=1` in CI, and the 90-minute job timeout.
  Preserve local default repeats and all existing override behavior. Do not introduce
  batching, reduced coverage, worker recycling that narrows concurrency, retries that
  conceal failures, a larger runner, swap, or gate relaxation as the repair. Temporary
  diagnostic subsets are permitted, but do not prove completion.
- Preserve first-nonzero exit semantics, per-repeat node attribution, and the isolation
  of contention from suite-gate leases, selection health, and timing stores. Neither
  `just check` nor `just check-full` should acquire this lane.
- Purely diagnostic workflow steps and artifact uploads are permitted and must be
  identified as such in the implementing agent's final response. Do not change job
  lists, permissions, secrets, variables, lint levels, budgets, or other CI limits.
- The coverage-contexts sibling is `sase-1rq.2.1.2`, test-cost is `sase-1rq.2.1.3`, Full
  CI is `sase-1rq.2.1.5`, and overall confirmation is `sase-1rq.2.1.7`. Read relevant
  notes before editing shared files or dispatching Telemetry. Avoid duplicate dispatches
  when a suitable run already exists. The known files-anchor timeout tracked by
  `sase-1qd` belongs primarily to the Full CI repair; reuse its landed fix if it appears
  in this lane.
- Test infrastructure can stay in this repository. Shared production behavior belongs in
  `sase-core`, opened through `/sase_repo`. Follow its binding and test conventions and
  the parent's published-core pin constraint; never silently reintroduce CORE_LAG. Read
  `ace`, `tui`, and other domain memory before changes in those areas. Read `tools`,
  `lint_and_test`, and the parent's required decision strands before landing. No memory
  note is authorized for editing here.
- Never create task beads from a child phase. Record discovered work with
  `PROPOSED FOLLOW-UP:` notes on that phase, and carry the full evidence to
  `sase-1rq.2.1.4` for the outer land agent's triage. A flake needs a failing and
  passing run on an unchanged tree under the same conditions. A focused serial pass
  following a contended failure is not that evidence. Do not skip deterministic
  failures. Cite an existing tracking bead in the skip reason when available.

## Phase: evidence

The deliverable is a reliable measurement and failure attribution, not a speculative fix
or a claim that added logging makes the lane healthy.

1. Refresh the latest runs and job/artifact records. Compare the current shutdown with
   run `42347892210` and any newer contention job. Record job timestamps and
   annotations. Check current landed commits before duplicating an existing repair.
2. Work from `.github/workflows/telemetry.yml`, `Justfile`'s `test-contention`,
   `tools/run_pytest::_run_contention`, and `tests/_contention_plugin.py`. Add a small
   test-tool helper if needed rather than growing the already large runner. Make output
   flush during execution; preserve ordinary command output and its exit status. Capture
   startup, worker collection completion, bounded progress, and recent/in-flight node
   IDs so a termination can be localized.
3. Sample at a modest interval, such as 15 seconds: elapsed time, controller and
   worker/descendant RSS, process and thread counts, host memory/swap availability,
   cgroup memory limits/events when readable, and filesystem bytes/inodes available.
   Include the actual scratch root chosen by `_prepare_pytest_tmpdir`, normally a
   dedicated directory under `/var/tmp`, as well as `/tmp` and the workspace. Measure
   directory growth sparingly; repeated recursive scans must not dominate the
   deliberately starved run. Avoid double-counting child trees and distinguish summed
   RSS from physical-memory consumption.
4. Keep diagnostics bounded and contention-only. Write periodic flushed summaries to the
   job log as well as a diagnostic file: a dying runner may never execute an `always()`
   upload. The current harness deletes its artifact directory before running; start
   recording after that deletion or use a separate diagnostic directory included in the
   upload. Do not let nested pytest processes overwrite controller evidence, and clean
   up only the sampler this run owns. Diagnostics failures must not replace the pytest
   exit status.
5. Test the helper with short controlled subprocesses: progress and resource samples,
   child failure/termination, no leaked sampler, missing proc/cgroup fields, and
   faithful nonzero exit propagation. Keep the existing contention contracts green. Do
   not launch real broad kill operations to test instrumentation.
6. Use a representative subset first. Then run a full unchanged-shape reproduction
   through `/sase_monitor` with a realistic capacity weight and at most one active
   whole-suite run. Prefer Python 3.12 with the same visual dependency setup as CI;
   record interpreter, dependency/core versions, SHA, affinity, and workload. More RAM
   on athena can conceal hosted pressure, so compare against measured CI resources
   instead of assuming local survival disproves OOM.
7. If local measurements cannot explain hosted termination, land the diagnostic change,
   then reuse or dispatch Telemetry on that landed SHA and monitor it. Publish a compact
   evidence record in bead notes: collection/runtime boundary, memory and disk
   curves/peaks, last active nodes, termination evidence, and the narrowest supported
   mechanism. Missing evidence must remain explicitly unknown.

## Phase: survival

This phase owns the dominant failure mechanism established by `evidence`, plus a
regression test and a full completed repeat. A red test tally is acceptable at this
boundary because the next phase owns those failures.

1. Re-read the evidence and isolate its cause with a bounded reproducer. Distinguish
   collection/import amplification across 26 workers from growth while tests run.
   Relevant seams include pytest fixtures and teardown, Textual app lifetimes, spawned
   child/thread ownership, and `_prepare_pytest_tmpdir` scratch handling. If resource
   headroom remains ample, investigate process-signal provenance and tests using
   synthetic PIDs or process groups before assuming infrastructure OOM.
2. Fix the measured owner: release retained state, repair teardown, avoid unnecessary
   collection/import allocations, or correct process targeting. Do not add generic
   forced GC, global process sweeps, or broad scratch deletion without evidence and
   ownership protection. A cleanup change must preserve active concurrent runs.
3. Add regression coverage for the actual defect. Use controlled fixtures/processes for
   ownership and cleanup; avoid tests whose only assertion is a fragile machine-specific
   RSS or elapsed-time threshold. Preserve collected node coverage.
4. Repeat the reproducer under matching contention and compare resource measurements
   before/after. Then obtain a complete full repeat using the original CI shape, under
   `/sase_monitor` locally and with hosted evidence when local conditions do not
   reproduce the original death. The command must reach its tally and preserve every
   real failure and exit code. Record elapsed time and resource headroom.
5. Hand the next phase every failing node, traceback/artifact location, repeat exit, SHA
   and environment, grouped provisionally by mechanism. If another independent survival
   defect appears, repair it if bounded; otherwise use the inherited remainder rule with
   the exact reproduction and evidence, without calling the lane fixed.

## Phase: failures

1. Read the completed repeat's full failures and sibling notes. Separate assertion
   failures, setup/teardown errors, collection errors, and crashed workers; an empty
   node JSON with a nonzero repeat exit is not green.
2. Group by mechanism and fix common causes before individual symptoms. Replace fixed
   sleeps and scheduling assumptions with existing event/condition-based
   synchronization, repair leaked global state and teardown, and preserve the behavior
   each test asserts. Do not merely inflate deadlines.
3. For a possible quarantine, demonstrate fail/pass on unchanged code with matching
   worker width, affinity, selection/load, interpreter, and dependencies. Record the
   evidence and skip reason under the inherited rule. Prefer a bounded root-cause repair
   when available. Coordinate any shared files-anchor/TUI issue with Full CI.
4. Run affected subsets first, then one whole default contention repeat with the
   unchanged CI shape. Record the complete tally, pytest exit code, number of tests
   completed, duration, and resources. The target is zero failing nodes, no red repeats,
   and no worker crash or infrastructure error within the existing timeout.
5. The parent plan expressly allows strict improvements plus documented remaining
   families when one phase discovers more independent defects than one coder can finish.
   In that case, land the fixes, record one `PROPOSED FOLLOW-UP:` per family with node
   IDs, same-condition reproductions, run URLs, and leads, and close only this child
   phase. Carry the inventory to the parent phase and overall `verify` phase; do not
   label a red contention job green or silently discard its failures.

## Phase: confirmation

1. Confirm each fix commit is on `origin/master`, refresh sibling/parent notes, and find
   a Telemetry run on a SHA containing all relevant fixes. Reuse one already
   queued/running, or dispatch `gh workflow run telemetry.yml --ref master`. This
   workflow publishes no release. Do not dispatch `release.yml`.
2. Watch through `/sase_monitor` with weight 0, `WATCHING CI` / `WATCHED CI` labels, and
   a successor prompt that names the run and expected evidence. Telemetry can queue
   behind an existing run and its other jobs take hours; use a realistic timeout. Finish
   the current turn through `/sase_final` after the monitor reports Staged; never end
   with an unowned background command.
3. Confirm the contention job succeeded on its first attempt and actually ran the full
   selection at 26 workers/two CPUs/one repeat. Inspect the final tally,
   nonzero/worker-crash indicators, artifact, wall time, and diagnostic headroom. If a
   rerun was needed, preserve both attempts and flake evidence. A green contention job
   does not establish that sibling Telemetry jobs are green.
4. If an in-scope failure remains, repair it when small or record it explicitly under
   the inherited remainder rule for overall verification. New unrelated regressions
   follow the parent scope rule. Never claim this plan's success criterion holds based
   solely on instrumentation, a subset, or a rerun with relaxed conditions.
5. Publish the evidence on this phase and `sase-1rq.2.1.4`: SHA ancestry, workflow/run
   and job URLs, attempt, timestamps, result, tally/counts, before/after resources, fix
   SHAs, quarantines and their evidence, and any remaining families.

## Verification and completion for every worker

Before landing tracked source changes, read the required project memory and run
`sase tool run check`. Use the harness's meaningful targeted tests, including
`tests/test_run_pytest_contention.py`, and tmpdir/worker/command tests when affected.
Use `/sase_monitor` for long commands and all whole-suite contention runs; run
formatting before a final verification monitor. `just check-full` is authorized by the
parent but is not a substitute for this lane or required routinely.

If a check failure reproduces identically on the clean base, append a
`PROPOSED FOLLOW-UP:` with both witnesses and any existing tracking bead; it does not
keep an otherwise complete child phase open. Keep experimental changes out of the landed
fix. Never mutate memory notes or screenshot goldens as incidental cleanup.

Before closing any assigned child phase, run `sase bead epic-symbols <phase-id>` and
resolve or re-key every remaining symbol to a still-open owner. Close only that assigned
phase with a verification note. Stage any monitor before `/sase_final`, wait for the
staging command to exit, and let host completion commit/publish work.

## Child epic land agent

Collect the closed child phases' evidence, refresh the latest qualifying contention job,
and provide the result to outer phase `sase-1rq.2.1.7`. Publish `ci_evidence` through
`/sase_var` with the parent schema: repository/default branch, tip SHA, workflow run
metadata, contention job result, fix SHAs, quarantines, and notes. Explicitly
distinguish this job's result from the overall Telemetry conclusion.

The ordinary completion criterion is a full successful contention job on landed code
containing the fixes, with unchanged gates and a complete zero-failure tally. If the
parent's authorized remainder rule was needed, preserve every remaining family on
`sase-1rq.2.1.4` and for the outer verify/land agents, and clearly report partial repair
instead of green. The parent research fallback requires its very-large-effort evidence;
mere diagnostic uncertainty does not qualify.

Under the user's requested child-epic handoff, this land agent finishes assigned parent
phase `sase-1rq.2.1.4` after the child phases and evidence are settled. Run
`sase bead epic-symbols sase-1rq.2.1.4`, resolve/re-key leftovers, and close that phase
with the concrete result and evidence. No child phase worker may close it. Never close
`sase-1rq.2.1`, `sase-1rq.2`, `sase-1rq`, or another agent's phase.
