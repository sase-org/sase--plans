---
tier: epic
title: Finish hosted contention runtime and failure repair
goal:
  The unchanged full default contention job completes within 90 minutes with zero
  failures on landed master, and interrupted runs retain actionable diagnostics.
parent_bead: sase-1rq.2.1.4.1
phases:
  - id: runtime
    title: Reduce measured hosted runtime and preserve interruption evidence
    depends_on: []
    description:
      "runtime: profile the Python 3.12 contention workload, repair the dominant
      measured overhead and interrupted-run diagnostics, and produce a completed full
      repeat with resource and failure evidence."
    size: medium
  - id: failures
    title: Repair the remaining failures under the real contention shape
    depends_on:
      - runtime
    description:
      "failures: repair every failure family from the completed repeat without weakening
      workload or gates, and prove a complete zero-failure contention repeat."
    size: medium
  - id: confirmation
    title: Confirm the full contention job on landed master
    depends_on:
      - failures
    description:
      "confirmation: reuse or dispatch Telemetry on landed master containing the
      repairs, verify an unchanged-shape successful contention job, and publish complete
      evidence for nested landing."
    size: small
proposed_by: bbugyi200.athena.sase-1rq.2.1.4.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-11-07 23:46:17
status: wip
---

- **PROMPT:**
  [prompts/202611/contention_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/contention_completion.md)
- **PARENT:**
  [202611/contention_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/contention_repair.md)

# Finish hosted contention runtime and failure repair

This is the remaining work from the landing audit of `sase-1rq.2.1.4.1`, whose plan is
`plan:202611/contention_repair.md`. Its parent is phase `sase-1rq.2.1.4` of
`plan:202611/sase_ci_green.md`; the root contract is `plan:202611/red_ci_repair.md`.
Read these through `sase artifact read`, and read the landing audit and follow-up triage
notes on `sase-1rq.2.1.4.1`.

Repository: the current sase checkout, `sase-org/sase`, branch `master`.

This needs distinct runtime, failure-repair, and landed-confirmation phases. The latest
hosted repeat completed fewer than half its tests in the allotted time, and its 17
failed reports have no retained node inventory. Neither the dominant runtime cost nor
the remaining failure families is established. This is larger than one bounded
direct-implementation tale. The already-completed collection OOM investigation and
sharding implementation are not repeated here.

## Verified baseline and unfinished acceptance

At the landing audit, HEAD and freshly fetched `origin/master` were both
`539afb211e381bd3b1d708baa19eefc649132aa7`. Five epic commits are already ancestors:

- `f97019e7e8`: resource/progress/signal diagnostics.
- `618dbb5fcf`: disjoint file-shard collection, replacement-worker ownership,
  import-cycle repair, and forced worker-configuration progress persistence.
- `291265f08a`: streaming feature-flag AST scanning, measured at 61 MB instead of 1.71
  GB peak standalone.
- `4a7ce0f330`: repairs for the first completed local repeat's 94 failures.
- `539afb211e`: repairs for the next local repeat's eight failures.

Latest qualifying hosted evidence is
[Telemetry 42437776062](https://github.com/sase-org/sase/actions/runs/42437776062),
attempt 1, on that exact SHA. Its
[contention job](https://github.com/sase-org/sase/actions/runs/42437776062/job/128603421769)
was cancelled at the unchanged 90-minute limit. The harness ran 02:30:40 to 04:00:27 UTC
on 2026-11-08 (89m47s), collected 60,569 items across all 26 workers, and finished only
29,354, with 24 in flight and 17 failed reports. There is no final pytest tally. The
workflow as a whole failed; this plan owns contention, not its coverage-contexts or
test-cost siblings.

Audited report: `file:explicit:693da93bb045b80e58b1448c`. GitHub artifact
`default-lane-contention-records`, ID `13603715213`, contains `shards.json`,
`diagnostics/samples.jsonl`, and repeat-01 progress/kill records. Preserve relevant
downloaded evidence as explicit SASE artifacts before GitHub retention expires.

The log and retained report show zero OOM kills, zero soak swap, memory PSI full zero,
at least 2.6 GiB host memory available, and CPU PSI around 97-98 late in the run.
Collection completed in about five minutes. CPU cost is the current lead; do not assume
memory exhaustion or infer the final runtime by linear item count. Tests have very
different costs.

Two corrections to the confirmation report matter: `WorkerProgress.finished_at` means
the latest test finished, not that the entire worker shard finished. Two populated
markers cannot establish that only two workers completed. The earlier 4,548.58-second
repeat (`aa3fzgn042n3`, SHA `4a7ce0f330`, eight failures) was local Python 3.14, not
hosted Python 3.12.

Current source confirms two diagnostic gaps: `ContentionFailureRecorder` writes its node
IDs only in `pytest_sessionfinish`, losing them on job cancellation; `Sampler.sample`
discovers descendants from the root, losing surviving children after root death. Both
must be repaired as part of this epic's evidence contract.

The lander ran the four contention contract files: **59 passed in 8.96 seconds**. That
proves harness contracts, not full-lane success. No quarantines were added.

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
  with no failure file is never success.
- Diagnostic additions must be bounded, best effort, and contention-only. A diagnostic
  I/O failure must not change pytest's exit status. Do not change workflow permissions,
  secrets, variables, job lists, or budgets.
- Do not run `just check-full`: the current landing instruction overrides the older
  parent plan's authorization. File-change verification is `sase tool run check`
  (`just check`). Read `tools`, `lint_and_test`, and relevant domain memory before
  edits, plus `decisions:ci-two-speed-split` and `decisions:check-full-is-explicit`.
  Read `ace`/`tui` before changing those areas.
- Shared backend policy belongs in `sase-core`, opened through `/sase_repo`; Python may
  remain thin host/process glue and presentation. If a real backend fix is needed,
  update the Rust binding/tests and pin only to a commit contained by a published core
  release, preserving the parent's CORE_LAG constraint. Do not turn this into an
  unrelated backend migration.
- Use `/sase_monitor` for every full contention run or CI wait, with realistic local
  capacity weight and at most one full run at a time. CI watches use weight zero. Wait
  for staging to exit, then finish through `/sase_final`; never leave a provider-owned
  background command running. Never manually commit or push.

## Phase: runtime

1. Refresh master and the latest Telemetry results. Review sibling `sase-1rq.2.1.3`
   notes before changing shared fixtures; reuse landed improvements. The lander reviewed
   eight intervening commits: `da1561407a`, `92fc89d83f`, `578a317bcf`, `960646e8ce`,
   `5922991bcf`, `ca13317d6e`, `dd5567e580`, `8668755b35`. The Files reserve fix,
   stylesheet initial restyle and cache resets are already present. Dynamic shard
   discovery includes the added test files. Preserve these integrations and inspect
   subsequent drift.
2. Repair interruption evidence in `tests/_contention_plugin.py`,
   `tests/_contention_progress.py`, and `tests/_contention_sampler.py` as needed.
   Persist failed node IDs, report phase, and bounded failure details as reports arrive,
   so termination retains a useful inventory. Separate partial evidence from a completed
   tally. Keep nested pytest isolation and first-error semantics. Track previously
   observed survivors with process identity (PID plus start time) or another run-owned
   membership proof; do not count unrelated group members, PID reuse, or scan/kill
   foreign processes. Make the final root-death sample truthful and bounded, with
   explicit limits on post-root observation/reaping. Test early cancellation, orphan
   membership, PID reuse, missing proc fields, write failure, and nested pytest without
   hazardous real broad signals.
3. Profile a representative subset in the hosted Python 3.12 environment and the actual
   26-worker/two-CPU shape before another full run. Measure CPU time and wall time by
   fixture/hook/test family, comparing against the local Python 3.14 evidence. Avoid
   attributing all delay to interpreter speed without a profile. Inspect common fixture
   resets, TUI startup/render costs, import duplication, child startup, and repeated
   whole-repo AST scans. Prior leads from survival note #5 are contract-manifest,
   artifact audits, deleted-proc-queue imports, project-display audits, markdown-width
   audits, and host-config/invoke audits; the feature-flag scan itself is already
   streaming. Use bounded profiles, not repeated blind full runs.
4. Remove the dominant demonstrated redundant work without weakening audit findings or
   assertions. Cache only with a valid invalidation model; mutable fixture trees and
   monkeypatches must not inherit stale results. Preserve sharded collection and
   exactly-once node ownership. Add regression or mutation checks that would fail if the
   optimization silently omitted coverage.
5. Run the affected tests and `sase tool run check`. Obtain one complete full repeat
   through a monitor with unchanged CI shape, preferably matching Python 3.12. Compare
   time and headroom with the baseline. A complete red tally is an acceptable handoff
   only to `failures`; timeout is not this phase's completion. Publish per-node
   failures/tracebacks, counts, exits, SHA/tree identity, environment, worker
   crashes/replacements, elapsed time and resources.

## Phase: failures

1. Consume the complete inventory from `runtime`, and current sibling fixes. Recover the
   historical 17 reports only where evidence permits; sampled in-flight nodes are leads,
   not proof that those nodes failed.
2. Repair common mechanisms before symptoms, preserving the assertions and
   behavior-under-test deadlines. Use readiness conditions for real races and isolate
   fake data from real services. Distinguish startup hang guards from tested deadlines;
   do not apply blanket timeout multipliers. Shared production changes must respect the
   Rust boundary and atomic state transitions.
3. Use focused contended repros and regressions, then a full unchanged-shape repeat.
   Require zero failed nodes, zero errors/crashed workers/red repeats, complete
   collected-versus-finished accounting and sufficient time for the existing CI setup.
   Any quarantine needs fail/pass on an unchanged tree under matching load, selection,
   interpreter and dependencies, with tracking evidence and the inherited skip reason. A
   serial pass after a contended failure is insufficient.
4. Read `sase-1s2`, `sase-1s3`, `sase-1s4`, `sase-xo`, `sase-1rf`, and `sase-1nf.10.1`
   if those mechanisms recur. They are triaged below. Reuse their fixes or coordinate an
   in-scope prerequisite; do not silently drop a failing node because it has external
   tracking. Run `sase tool run check` for changes.

## Phase: confirmation

1. Verify the implementation commits are ancestors of current `origin/master`. Reuse a
   qualifying Telemetry run, otherwise dispatch
   `gh workflow run telemetry.yml --ref master`. Do not dispatch release.yml or merge a
   release PR. Watch through a zero-weight SASE monitor.
2. Require a successful `contention-test` job with the full current default selection,
   26 workers/two CPUs/one repeat, a complete zero-failure pytest tally, no
   infrastructure errors, and unchanged 90-minute job timeout. Record attempt, run/job
   URLs, timestamps, SHA ancestry, interpreter/dependencies, selected and finished
   counts, exit code, resources and quarantines. A cancelled/partial run must remain a
   blocker; never repeat the prior confirmation's closure while success is still
   missing. Iterate actual remaining failures within scope.
3. Publish `ci_evidence` through `/sase_var`, with `repo`, `default_branch`, `tip_sha`,
   `runs` (workflow, run_id, url, head_sha, event, attempt, conclusion, completed_at),
   `fixes`, `quarantined`, and `notes`. Add contention job details separately from the
   workflow conclusion. Attach durable evidence and record it on this phase and
   `sase-1rq.2.1.4` for the outer verifier.

## Follow-up triage already completed by the lander

Do not recreate these tasks. The complete disposition of all 15 proposal entries is
recorded on `sase-1rq.2.1.4.1`:

- `.1` note #2 and `.3` #9: one pre-existing import-time SIGTERM defect, **sase-1s2**,
  small bug, independently reproduced by the lander.
- `.1` #4: root-death sampler membership, **in scope here**. `.1` #6: worker count
  persistence, already fixed in `618dbb5fcf` and covered by passing tests.
- `.2` #4: resource headroom remains an acceptance measurement. `.2` #5 and `.4` #2:
  runtime/hosted completion, **in scope here**.
- `.2` #6: cosmetic header proposal declined; the full summed count already prints
  immediately before xdist's first-shard count in the hosted log.
- `.3` #2: swallowed drain alarm, corroborated **sase-xo**. `.3` #3: unbounded Linux
  lock-owner probe. Both routed to active causal owner **sase-1nf.10.1** and its
  timeout-cleanup phase `.2`, with measured evidence.
- `.3` #4: buffered-progress false stall, corroborated **sase-1rf** and routed to
  **sase-1nf.10.1**. Preserve inconclusive owner-probe results on deadlines.
- `.3` #5: missing shared fake Footprint-card seam, **sase-1s3**, small bug.
- `.3` #8: real-checkout index-lock leak, **sase-1s4**, large bug; causal writer
  unproven, so no blanket lock deletion.
- `.3` #6 and #10: blanket fakey wait increases and generic CPU-scaled hang guards
  declined for lack of a remaining concrete failing node; repair measured recurrences
  individually without weakening tested deadlines.

## Worker completion and nested landing

Workers close only their own fully completed phases with evidence after resolving their
`sase bead epic-symbols <phase-id>` entries. No worker creates task beads; new
discoveries go into `PROPOSED FOLLOW-UP:` notes for the child lander.

This child epic's lander must verify every phase and post-start drift, triage new
proposals, resolve this child's symbol entries, close this child normally, run
`just symvision`, and mark this child's linked plan done. Its `parent_bead` is the
handoff back to **sase-1rq.2.1.4.1**. Do not add the parent's close, symvision
verification, or plan-status update as a child phase.

After the child lands, re-audit **sase-1rq.2.1.4.1** using its prior landing note, all
descendants and notes, `plan:202611/contention_repair.md`, and intervening commits.
Recheck descendant and linked-plan readiness. Only when the full hosted completion
criterion and all epic-caused defects are resolved, resolve/re-key every
`sase bead epic-symbols sase-1rq.2.1.4.1` entry and close it normally with
`sase bead close sase-1rq.2.1.4.1 --note "<verified evidence and triage>"`. Run
`just symvision`, open the plans repo through `/sase_repo`, and set `status: done` in
`202611/contention_repair.md` within the printed plans path.

Then re-read **sase-1rq.2.1.4**, which is a phase. Verify the completed child plan meets
that phase's full passing-contention scope, resolve any symbol entries, and close only
that phase normally with a concrete evidence note. Leave its containing epic
**sase-1rq.2.1** to the already-waiting land agent. If either parent is incomplete or
ambiguous, record its blocker and leave it open. Never use `--force` to advance a
successful nested landing.
