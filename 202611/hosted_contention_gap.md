---
tier: epic
title: Close the hosted contention runtime gap and confirm on master
goal: A successful hosted contention-test on landed master finishes the full default
  job within 90 minutes with a complete zero-failure pytest tally, and confirmation
  evidence is published for nested landing of sase-1rq.2.1.4.1.5.3.
parent_bead: sase-1rq.2.1.4.1.5.3
phases:
- id: hosted-runtime
  title: Make the hosted default contention job finish within 90 minutes
  depends_on: []
  description: 'hosted-runtime: profile the GitHub two-CPU Python 3.12 gap against
    the local green tally, remove the dominant hosted overhead without weakening the
    workload, and prove a complete hosted contention tally inside 90 minutes.'
  size: medium
- id: hosted-failures
  title: Repair remaining failures from a complete hosted tally
  depends_on:
  - hosted-runtime
  description: 'hosted-failures: repair every failure family from the completed hosted
    tally without weakening workload or gates, and prove a complete zero-failure hosted-shape
    repeat.'
  size: medium
- id: hosted-confirmation
  title: Confirm hosted contention-test on landed master
  depends_on:
  - hosted-failures
  description: 'hosted-confirmation: reuse or dispatch Telemetry on landed master
    containing the hosted repairs, verify an unchanged-shape successful contention
    job, and publish complete evidence for nested landing.'
  size: small
proposed_by: bbugyi200.athena.sase-1rq.2.1.4.1.5.3--1
proposed_with: grok/grok-4.6
create_time: 2026-11-09 04:31:21
status: wip
bead_id: sase-1rq.2.1.4.1.5.3.1
---

- **PROMPT:** [prompts/202611/hosted_contention_gap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/hosted_contention_gap.md)
- **BEAD:** [sase-1rq.2.1.4.1.5.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.2.1.4.1.5.3.1.md)

# Close the hosted contention runtime gap and confirm on master

This is the remaining work from confirmation phase `sase-1rq.2.1.4.1.5.3` of
`plan:202611/contention_completion.md`. That phase dispatched Telemetry on landed master
and required a successful hosted `contention-test`. The job cancelled at the unchanged
90-minute limit with a partial tally, so confirmation stayed open.

Repository: the current sase checkout, `sase-org/sase`, branch `master`.

## Verified blocker

At confirmation inspection, HEAD and freshly fetched `origin/master` were both
`7a7ecd6885b32ea969d462881673e96d776a37d1`. Failures-phase `sase-1rq.2.1.4.1.5.2`
already closed with a local Python 3.12 default-shape zero-failure repeat (60,608
passed, 21 skipped, 0 failed in 4,570.78s). That local tally is not hosted success.

Qualifying hosted evidence is
[Telemetry 42574447281](https://github.com/sase-org/sase/actions/runs/42574447281),
`workflow_dispatch` attempt 1, on that exact SHA. Its
[contention-test job](https://github.com/sase-org/sase/actions/runs/42574447281/job/129023334146)
was cancelled at `timeout-minutes: 90`. The harness ran 06:54:11Z to 08:23:59Z on
2026-11-09 (~89.7 minutes after configure). Controller snapshot: 60,600 collected across
26 workers, 28,380 finished, 23 in flight, 9 failed reports, `exitstatus` null. Two
workers set `finished_at`. There is no complete pytest tally.

The same class of hosted timeout already appeared on
[Telemetry 42437776062](https://github.com/sase-org/sase/actions/runs/42437776062)
(`539afb211e`, 29,354 / 60,569 finished, 17 reports). Hosted rate on this SHA is about
5.3 tests/s versus about 13.3 tests/s on the local Python 3.12 proof — a stable ~2.5×
gap.

Resources at cancel: `oom_kill` 0, soak swap 0, memory PSI full 0, min available memory
2.83 GiB, CPU PSI some up to 98.86, loadavg ~26 on two soak CPUs. CPU cost on the pinned
pair is the lead. Do not infer remaining runtime by linear item count; TUI-heavy shards
finished far fewer nodes (`gw24` 386, `gw19` 432, `gw13` 438) than `gw14` (2,501, shard
complete).

The nine reports and 23 in-flight node IDs are leads, not a complete failure inventory.
Preserve the downloaded GitHub artifact `default-lane-contention-records` (id
`13658742947`) through the explicit SASE snapshots attached to `sase-1rq.2.1.4.1.5.3`.

A same-SHA scheduled Telemetry run
[42578053332](https://github.com/sase-org/sase/actions/runs/42578053332) was in progress
after this cancelled run released concurrency group `ci-telemetry`. Reuse it only if its
`contention-test` later finishes the full default shape with a complete zero-failure
pytest tally on `7a7ecd6885`. Do not treat an in-progress or cancelled job as
confirmation.

## Fixed constraints

Inherited decisions remain final: `test_quarantine = quarantine_flakes` and
`release_merge = gate`. There are no new reviewer decisions or memory changes.

- Keep every default non-slow, non-visual test, 26 workers, `taskset -c 0,1`, one CI
  repeat, Python 3.12 in the existing visual environment, and the 90-minute job timeout.
  Keep local repeat defaults and explicit override behavior.
- Do not repair this by reducing selection or concurrency, recycling workers into a
  smaller effective pool, batching the suite, using a larger runner, adding swap,
  concealing failed attempts with retries, or relaxing gates.
- Preserve first-nonzero exit semantics, collection/crash attribution, and isolation
  from suite-gate leases, selection-health records, and timing stores. A partial run
  with no complete pytest tally is never success.
- Diagnostic additions must be bounded, best effort, and contention-only. A diagnostic
  I/O failure must not change pytest's exit status. Do not change workflow permissions,
  secrets, variables, job lists, or budgets.
- Do not run `just check-full`. File-change verification is `sase tool run check`
  (`just check`). Read `tools`, `lint_and_test`, and relevant domain memory before
  edits, plus `decisions:ci-two-speed-split` and `decisions:check-full-is-explicit`.
  Read `ace`/`tui` before changing those areas.
- Shared backend policy belongs in `sase-core`, opened through `/sase_repo`; Python may
  remain thin host/process glue and presentation. If a real backend fix is needed,
  update the Rust binding/tests and pin only to a commit contained by a published core
  release, preserving the ancestor CORE_LAG constraint. Do not turn this into an
  unrelated backend migration.
- Use `/sase_monitor` for every full contention run or CI wait, with realistic local
  capacity weight and at most one full run at a time. CI watches use weight zero. Wait
  for staging to exit, then finish through `/sase_final`. Never manually commit or push.
- A local Python 3.12 green tally is supporting evidence only. Hosted completion is the
  acceptance gate for this child.

## Phase: hosted-runtime

1. Refresh master and the confirmation records on `sase-1rq.2.1.4.1.5.3`. Reuse sibling
   notes on `sase-1rq.2.1.4.1.5.1` and `sase-1rq.2.1.4.1.5.2`. Preserve landed
   collection, progress, sampler, and failure-repair work.
2. Profile in the hosted Python 3.12 visual environment and the actual 26-worker/two-CPU
   shape before another full hosted run. Measure CPU and wall time by fixture, hook, and
   test family against the local 4,570s green tally. Start from the long in-flight nodes
   at cancel (`test_main_static_on_repo_exits_zero`,
   `test_tracked_marker_mutation_sites_are_reviewed`,
   `test_only_boundary_calls_provider_invoke`) and the slow TUI shards. Do not attribute
   the whole gap to interpreter 3.12.16 versus 3.12.13 without a profile.
3. Remove the dominant demonstrated redundant hosted work without weakening audit
   findings or assertions. Cache only with a valid invalidation model. Preserve sharded
   collection and exactly-once node ownership. Cost-aware shard balance is allowed when
   it keeps the same selected nodes and worker count. Add regression or mutation checks
   that would fail if the optimization silently omitted coverage.
4. Run the affected tests and `sase tool run check`. Prove completion with one hosted
   `contention-test` (Telemetry `telemetry.yml --ref master` after the repairs land,
   never `release.yml`) that finishes inside 90 minutes with a complete pytest tally. A
   complete red tally is an acceptable handoff only to `hosted-failures`. Timeout is not
   this phase's completion. Publish per-node failures, counts, exits, SHA, environment,
   worker crashes, elapsed time, and resources.

## Phase: hosted-failures

1. Consume the complete hosted inventory from `hosted-runtime`. Recover the nine partial
   reports from Telemetry 42574447281 only where that complete tally still fails them.
   In-flight nodes from the cancelled job are leads, not proof.
2. Repair common mechanisms before symptoms, preserving assertions and
   behavior-under-test deadlines. Use readiness conditions for real races. Distinguish
   startup hang guards from tested deadlines. Shared production changes must respect the
   Rust boundary.
3. Use focused contended repros and regressions, then a full unchanged-shape
   hosted-shape repeat. Require zero failed nodes, zero errors/crashed workers/red
   repeats, complete collected-versus-finished accounting, and enough time for the
   existing CI setup. Any quarantine needs fail/pass on an unchanged tree under matching
   load with the inherited skip reason. A serial pass after a contended failure is
   insufficient.
4. Read `sase-1s2`, `sase-1s3`, `sase-1s4`, `sase-xo`, `sase-1rf`, and `sase-1nf.10.1`
   if those mechanisms recur. Run `sase tool run check` for changes.

## Phase: hosted-confirmation

1. Verify the hosted-runtime and hosted-failures commits are ancestors of current
   `origin/master`. Reuse a qualifying Telemetry run on that tip, including 42578053332
   only if it is still that tip and its `contention-test` already succeeded. Otherwise
   dispatch `gh workflow run telemetry.yml --ref master`. Do not dispatch `release.yml`
   or merge a release PR. Watch through a zero-weight SASE monitor.
2. Require a successful `contention-test` with the full current default selection, 26
   workers/two CPUs/one repeat, a complete zero-failure pytest tally, no infrastructure
   errors, and unchanged 90-minute job timeout. Record attempt, run/job URLs,
   timestamps, SHA ancestry, interpreter, selected and finished counts, exit code,
   resources, and quarantines. A cancelled or partial run remains a blocker.
3. Publish `ci_evidence` through `/sase_var`, with `repo`, `default_branch`, `tip_sha`,
   `runs` (workflow, run_id, url, head_sha, event, attempt, conclusion, completed_at),
   `fixes`, `quarantined`, and `notes`. Add contention job details separately from the
   workflow conclusion. Attach durable evidence and record it on this confirmation
   phase, `sase-1rq.2.1.4.1.5.3`, and `sase-1rq.2.1.4`.

## Worker completion and nested landing

Workers close only their own fully completed phases with evidence after resolving their
`sase bead epic-symbols <phase-id>` entries. No worker creates task beads; new
discoveries go into `PROPOSED FOLLOW-UP:` notes for this child's lander.

This child's lander must verify every phase and post-start drift, triage new proposals,
resolve this child's symbol entries, close this child normally, run `just symvision`,
and mark this child's linked plan done. Its `parent_bead` is the handoff back to
**sase-1rq.2.1.4.1.5.3**. Do not add that confirmation phase's close, the parent epic
close, or `plan:202611/contention_completion.md` status update as a child phase.

After this child lands, re-read **sase-1rq.2.1.4.1.5.3** and close it normally with the
hosted `contention-test` evidence. Leave **sase-1rq.2.1.4.1.5** and every ancestor to
their own land agents. If the parent confirmation phase is still missing hosted success,
record the blocker and leave it open. Never use `--force` to advance a successful nested
landing.
