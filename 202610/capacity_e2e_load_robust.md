---
tier: tale
title: Make the weighted monitor-capacity test reliable under load
goal:
  Complete sase-1ou.3 with a verified cause and loaded before/after evidence while
  preserving five-second budgets and real capacity assertions.
size: medium
proposed_by: bbugyi200.athena.sase-1ou.3
proposed_with: codex/gpt-6-astra
bead: sase-1ou.3
create_time: 2026-10-27 10:43:55
status: wip
---

- **PARENT:**
  [202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:**
  [sase-1ou.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ou/sase-1ou.3.md)

# Make the weighted monitor-capacity test reliable under load

Complete phase `sase-1ou.3`, which addresses the flake tracked by `sase-12f`. Fix the
measured fixture drift, identify any remaining cause of the five-second fakey timeout
under load, and verify the real weighted-capacity lifecycle without increasing budgets
or weakening assertions.

This is one bounded coding task: the shared fakey harness, its monitor acceptance test,
focused regression coverage, and the node's flake-baseline entry. There are no earlier
implementation/remainder notes on this phase. A medium tale is appropriate; the
expensive stress runs can use monitor continuations.

## Governing scope

Read these audited records again before implementing:

- `sase bead read sase-1ou.3 -r "Need the assigned phase and final epic decisions"`
- `sase bead read sase-12f -r "Need current flake evidence and any intervening fix"`
- `sase artifact read plan:202610/backlog_epic_2_top_five_task_beads.md "Need the accepted capacity-e2e design"`

The epic's final decision is `dormant_flakes = close`; its dormant-flake sweep belongs
to the epic lander. Do not edit memory files, create beads, close the parent epic, or
close any ancestor plan bead. The current user explicitly says to close only
`sase-1ou.3`: leave evidence for the epic lander to close the underlying task
`sase-12f`, despite the older epic's instruction to close it here. Record discovered
work with `sase bead note sase-1ou.3 'PROPOSED FOLLOW-UP: <summary — evidence>'`.

Keep the test's five-second overrides. Do not raise or remove `_WAIT_TIMEOUT`,
`_FAKEY_RELEASE_TIMEOUT`, or production budgets; skip/xfail tests; weaken assertions; or
add suppressions. Preserve the real capacity scan, slot lock, reservation/adoption, and
proc settlement paths. Do not expand this phase into fixing the already-documented
monitor-to-successor competitor race.

## Findings at planning time

The clean planning base was `e608de2db6`, including phase `sase-1ou.1` and the earlier
capacity-scan optimization `e7d9f7a479`. The target still has a live entry under
`# sase-12f` in `tests/reproducible_flake_baseline.txt`:

```text
tests/fakey/test_monitor_capacity_e2e.py::test_weight_two_land_agent_session_retains_one_claim_through_real_dispatch_and_delayed_child_bootstrap
```

`sase-12f` includes repeated exit-124 failures in loaded suites and a sole-node Full CI
failure on 2026-10-25. Its original node name used `land_family`; the name above is
current. A passing isolated run alone is insufficient closure evidence.

The unchanged target passed once with an in-memory timing plugin:

| Observation                       |     Measured time |
| --------------------------------- | ----------------: |
| Starter wait for started marker   |           0.178 s |
| `start_monitor`                   |           0.439 s |
| Starter release through join      |           0.051 s |
| `settle_named_proc`               |          60.608 s |
| Successor wait for started marker |           0.121 s |
| Continuation adoption             |           0.005 s |
| Entire test call / pytest run     | 61.89 s / 65.58 s |

There is a concrete fixture-layout defect:

- `tests/fakey/_runner_slot_harness.py` constructs agents under
  `projects/fakey-slots/artifacts/ace-run/<timestamp>`.
- `src/sase/turns/followup.py::_starter_artifacts_dir` uses the canonical path binding.
  For timestamp `20260713090000`, the binding returns
  `projects/fakey-slots/artifacts/ace-run/202607/13/20260713090000`.
- Thus the starter has written `done.json`, but the production follow-up wait looks
  elsewhere and exhausts its 60-second settlement wait.
- An in-memory diagnostic monkeypatch changed only the harness's agent creation
  directory to that canonical date-sharded location. The target then passed in **1.85
  seconds of test-call time / 4.32 seconds total**. A wrapper around
  `wait_for_starter_artifacts_dir` observed the done marker already present and returned
  `True` in 0.000 seconds. Five-second budgets and all test assertions remained
  unchanged. No implementation source files were edited.

This proves the unnecessary minute-long wait, not yet the cause of the historical
load-only exit 124: that minute occurs after starter release and before successor
startup. Preserve that distinction in the evidence and final claims.

The test's dummy supervisor sleeps for only 30 seconds, so the old minute-long
settlement also outlives its intended live supervisor. The canonical-path probe kept
this supervisor alive through the lifecycle naturally. Avoid introducing another
arbitrary longer sleep to address liveness.

The harness isolates `SASE_HOME`, so its capacity scan is over temporary fixture
records. Do not attribute its failures to a large real host history merely from the
older epic's 11-second scan observation.

## Implementation and investigation

1. **Preserve comparable baseline evidence before edits.** Recheck HEAD and intervening
   changes. Instrument the existing test with a temporary pytest plugin or equivalent
   local diagnostic wrapper, retaining timings and the full stderr of any fakey
   exit 124. Record monotonic timestamps for agent claim, process spawn, started marker,
   release request, provider exit, join, monitor start phases, settlement, successor
   admission, and adoption. For slot work, distinguish lock acquisition from scan time
   and record the scanned root/count. Reuse `monitor_start_timing.json` where possible.
   Do not log every successful poll or add broad production telemetry just for this
   investigation.

   Run the repeat matrix below against this base before applying the fix. Keep one
   stress driver and load configuration for both before and after. A clean isolated pass
   is expected and does not negate loaded failure evidence.

2. **Create current-layout harness records.** In `tests/fakey/_runner_slot_harness.py`,
   use `sase.core.agent_artifact_paths.canonical_agent_artifact_path` when creating each
   agent, passing the harness's isolated projects root explicitly if needed. Create
   parent directories before writing metadata. Keep the workflow artifact root and
   project-file placement distinct from the per-agent dated directory; the current
   `self.artifacts_root.parents[1]` project-file derivation must not silently change
   meaning. Derive each timestamp's path independently rather than hard-coding the
   planning probe's `202607/13` directory.

   Keep `parent_timestamp`, artifact basenames, agent-session lineage, and index updates
   consistent. Use the existing path binding, not a Python reimplementation of
   path-layout rules. Audit the runner-slot and gate-capacity tests that share this
   harness for flat-path assumptions.

3. **Make completion and delayed-bootstrap assertions observe real events.** Strengthen
   the existing target to prove that the released starter's actual completion marker is
   the one production settlement observes, and that the starter-settlement wait succeeds
   rather than timing out into degraded launch. A narrow spy that delegates to the real
   wait is acceptable; returning a fabricated success is not. Assert the required
   lifecycle ordering without a tight total-runtime assertion that itself flakes under
   load.

   Preserve the real
   `settle_named_proc -> launch_followup_agent -> claim_ordinary_continuation_dispatch -> adopt_ordinary_continuation_delivery`
   sequence and parsing of the successor's real `%queue` prefix. Preserve owner key
   equality, explicit weight 2.0, competitor parking, and eventual admission after
   release. Keep the existing competitor boundary: do not silently claim coverage for
   the separately documented monitor-to-successor race.

   If replacing a fixed negative-observation sleep is warranted, use an observed
   admission attempt or marker transition before asserting the competitor remains
   parked. If a supervisor lifetime needs explicit control, use a parent-owned
   event/pipe and terminate/reap it in `finally`; do not increase its sleep. Ensure
   failure paths release owned fakey children and stop parked threads before fixture
   teardown, so failed repeats cannot contaminate later samples.

4. **Resolve any remaining measured five-second overrun at its source.** Re-run the
   identical stress driver after the layout correction. Attribute failures to the
   specific live-agent interval, rather than the old 60-second total. Correct a
   demonstrated fixture dependency or unnecessary polling/repeated work in the narrow
   owner. Do not renew the release deadline with heartbeats, move work outside an
   assertion window to hide it, or replace the real provider or capacity decision with a
   stub.

   If measurements instead demonstrate a production bottleneck, fix the measured
   operation and give it a focused regression. Shared domain behavior belongs in
   `sase-core`; first use `/sase_repo` to open it and read its instructions, then change
   its Rust implementation/binding and ratchet `sase-core-revision.txt` after the core
   change is host-committed. Python remains a thin adapter. Do not undertake a
   speculative capacity redesign or unrelated optimization.

## Verification and acceptance

Use `/sase_monitor` for long repeat loops, waiting for the handoff command itself to
exit. Keep evidence across continuations; do not rerun completed baselines. Retain
per-iteration node outcomes, elapsed times, failure messages, base/fixed revision or
diff identity, worker count, and host load measurements.

- **Isolated matrix:** at least 10 invocations of the exact target node with the same
  `.venv/bin/python -m pytest` command before and after.
- **Loaded matrix:** at least 20 invocations of the complete
  `tests/fakey/test_monitor_capacity_e2e.py` file at `-n 4`, with the same bounded
  concurrent load source before and after. Use the existing worker-token governor; do
  not disable it or launch uncontrolled CPU consumers. The load source must have fixed
  concurrency, an explicit stop condition, and unconditional cleanup. Record the
  target's failure count separately from sibling failures. Baseline and fixed runs must
  use comparable actual load, not merely identical flags.
- Obtain a before/after rate for the reported failure. If no baseline failure occurs,
  report `0/N` honestly and extend a bounded measured stress run or use a deterministic
  reproduction of the observed slow stage; do not invent a nonzero baseline or equate
  the path speedup with proof that exit 124 is fixed. A controlled reproducer must
  exercise the same causal stage, not simply sleep longer than the five-second watchdog.
  If recent upstream work already fixed the exit-124 cause, identify that change and
  document the current-load evidence separately from the fixture correction.
- The fixed target must preserve the original five-second budgets and all behavioral
  assertions, without exit 124 or lingering child/thread failures in the repeat matrix.
  The new regression must detect the flat/canonical mismatch without waiting a full
  minute to fail.
- Run the whole monitor-capacity file and the directly affected runner-slot and
  gate-capacity harness consumers. Run any focused regression for an additional measured
  fix. Avoid unrelated suites once the required checks pass.
- Read `lint_and_test.md`, format the changes, and run `sase tool run check` in each
  changed repository. Do not run `just check-full` or a global install. Reproduce
  unknown failures on a clean base before classifying them as unrelated.

Once the flake fix is established, replace only this node's active baseline entry with
`# fixed-at: <UTC instant> <node id>`, following the file header's preceding
bead/fixing-commit comment convention. Never invent a commit hash or use the planning
base as the fixing commit. If the implementation is still uncommitted, identify its
phase and concrete fix in the comment and record the verified UTC cutoff; explicitly
leave the epic lander the final reconciliation of the comment's fixing SHA and landing
instant after host finalization. This metadata reconciliation is not permission to
bypass the required flake proof or to commit manually. Coordinate this file with other
phases rather than overwriting their edits. Run
`.venv/bin/python tools/selection_health --fail-on-new-flake`; verify that the target's
retired evidence is absent, and record any unrelated remaining verdicts.

## Completion

Append the verified cause, exact stress commands/load configuration, before/after
failure counts, targeted results, check result, and baseline-retirement evidence to
`sase-1ou.3`. Include evidence the epic lander can use for `sase-12f`.

If a required check fails identically on the clean base, record a `PROPOSED FOLLOW-UP:`
on this phase with the matching evidence and existing owner bead, if known. Such an
unrelated failure does not keep this completed phase open.

Before closure run `sase bead epic-symbols sase-1ou.3` again (there were no entries at
planning time). Resolve any entries or re-key them to a still-open owning bead. Then
close only this phase:

```bash
sase bead close sase-1ou.3 --note "<verified cause; before/after stress counts; focused/check results; baseline evidence>"
```

Use `/sase_final` for normal turn completion; commits and publication remain host-owned.
Do not close `sase-1ou`, an ancestor plan bead, or `sase-12f` in this phase worker.
