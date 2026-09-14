---
tier: tale
title: Finish sase-10w.5 and verify sase-10w.6 starts
goal:
  Restore green CI, prove fresh scoped baseline use, close sase-10w.5, and verify its
  existing successor starts after dependency release.
size: medium
proposed_by: bbugyi200.athena.sase-10w.5.f0.f0
create_time: 2026-09-14 14:03:29
status: wip
---

# Finish sase-10w.5 and verify sase-10w.6 starts

## Outcome and scope

Complete the existing `sase-10w.5` phase: restore green Master Gate, obtain a fully
green Full CI with a usable coverage-contexts artifact, install that baseline on this
host, and prove an ordinary scoped check consults it fresh. Then close only `sase-10w.5`
and verify its already-created successor agent `sase-10w.6` crosses its dependency
barrier and starts execution.

This is a medium tale: one closeout worker can implement the bounded repair and
operational sequence, with SASE monitor continuations for long work. There is no new
bead DAG or independent implementation program to create. This plan updates the
execution details of `plan:202609/close_sase_10w_5_green_ci_baseline.md` and retains the
phase-five requirements of `plan:202609/green_ci_fast_lane_v0_17_2.md`. The governing
research is `research:202609/test_feedback_speedup_decision.md`; consume these with
`sase artifact read` if additional context is needed.

Release PR #299, publishing, the package dependency floor, `sase-10w.6`, and the parent
epic remain their owners' work. Do not hand-merge a release PR, publish packages, close
the parent, change bead statuses by hand, or create beads. Record discovered work on
`sase-10w.5` as `PROPOSED FOLLOW-UP:` notes.

## Verified starting state — recheck before implementation

Observed 2026-09-14 around 18:00 UTC:

- SASE HEAD is `378f18b2ef0ccb6b12a91408ddf9adb80fa8511e`; the tree is clean. The
  earlier pin repair landed as `df585611746deb530c58c30f219ebc888fbd0c53`.
- That repair recorded a nonexistent full SHA in `sase-core-revision.txt`:
  `5ea49f5eda1e6bfcb880535c99717231801aa77e`. The actual managed-tmp commit is
  `5ea49f5b6199da130f58923cec609d70a775eaa8`. They share seven leading characters;
  abbreviated comparison missed the error.
- Master Gate runs `34871205869` and `34874998721` fail at
  `core-wheel / Check out Rust core`. The latter's log reports
  `upload-pack: not our ref` for the nonexistent SHA; lint and tests are skipped. This
  is a deterministic checkout failure, not a flaky test.
- Audited `sase repo open sase-core` resolves a clean checkout whose `origin/master` is
  `3566872b4916123fedf100b7c5684c701085655c`. `git ls-remote origin refs/heads/master`
  independently confirms that full SHA. It contains the actual managed-tmp fix plus
  later sudo, disk-pressure, and forced-reuse contracts. This is the preferred pin
  candidate, subject to a fresh remote and binding check at implementation time.
- Local `just install` builds from the linked core checkout, not necessarily the text in
  the CI pin file. The previous green local checks therefore do not establish that CI
  can fetch the pin or that the tested build matches it.
- Latest sampled Full CI is failed run `34846053851`, on old SHA `f86056c7fd…`. There is
  no qualifying green Full CI in the sampled recent runs.
- `sase-10w.5` is in progress; phases 1–4 are closed; it has no epic symbols.
  `sase-10w.6` is assigned/in progress but its runner is `WAITING`. Its waiting marker
  contains both `waiting_for: [sase-10w.5]` and `wait_for_beads: [sase-10w.5]`. The
  original `sase-10w.5` agent's `done.json` records `outcome: completed`; the bead is
  still open.
- Successor PID 2008384 is live. Its artifact timestamp is `20260914111743`;
  `wait_completed_at` and `run_started_at` are absent. Rediscover its current artifact
  directory through `sase agent list -a -j`; do not hard-code paths. `sase agent show`
  can label this parked process RUNNING, so that panel or a live PID alone is not proof
  that its provider has started.
- AXE is healthy. The `waits` lumberjack runs `sidecar_auto_sync` and `wait_checks`.
  Default sync cadence is 30 seconds; wait_checks has a 120-second quiet backstop. The
  runner also re-resolves dependencies every 60 seconds. Closure must reach canonical
  bead state before either resolver can release it.

## Execution rules

Use explicit `-R sase-org/sase` for GitHub workflow commands and `GH_REPO=sase-org/sase`
for tools that call `gh` internally. Query current state after every handoff; recorded
SHAs, run IDs, and PIDs are evidence, not permanent selectors. Use `/sase_repo` before
opening another repository; never use a sibling workspace or a web fetch as a
substitute.

Use `/sase_monitor` for builds, CI waits, and long verification. The live CLI accepts
the command after `--`; inspect `sase monitor start --help` instead of copying obsolete
`--command` examples. Provide explicit present/past status labels, a timeout, and
`--next` covering success, failure, and timeout. Use ordinary success-continuing
monitors here: the `verify` profile normally has no continuation on success, and
prepared host completion also ends after successful finalization. Neither alone
schedules the remaining closeout.

Record the step, exact revision/run, verification outcome, and remaining work before
every handoff. Monitors may poll quietly (for example every 120 seconds for CI); API
failures must produce diagnostics, not an endless empty loop. Allow about 45 minutes for
Master Gate and four hours for queued Full CI.

Repository repairs must land through `/sase_final`, with `bead_action: keep` when the
host template requires it. A normal finalizer turn ends before the new remote CI exists;
that is an intermediate landing checkpoint, not phase completion. Its durable bead note
must say to resume this approved plan at step 2 after the host lands the repair. Do not
claim an automatic continuation was scheduled unless the host actually supplied one, and
do not watch the old red SHA as though it contained a still-local fix. No manual
commits, branches, or PRs are needed.

## 1. Repair and verify the full core pin

1. Read `sase bead show sase-10w.5`, `sase bead show sase-10w.6`, the current
   dependencies, and `sase bead epic-symbols sase-10w.5`. Preserve assignment and
   status. If the bead has since closed, verify its evidence and resume successor
   observation instead of re-closing or reopening it.
2. Fetch `origin master` and inspect the latest Master Gate explicitly in
   `sase-org/sase`. If another agent already repaired the pin, consume that landed fix.
   Otherwise open `sase-core` through `/sase_repo`, read its instructions, and compare
   the pin against full object IDs and remote refs.
3. Replace the bad pin with the validated remote core tip. Prefer the existing
   `tools/ratchet_core_revision` path: review its `--report-only` output, then apply and
   verify the exact resulting 40-character value. Exit 2 means a ratchet was
   proposed/applied, not failure. The planned candidate is
   `3566872b4916123fedf100b7c5684c701085655c`; inspect any intervening commits before
   accepting a newer one. Require object existence, ancestry from
   `5ea49f5b6199da130f58923cec609d70a775eaa8`, and containment in a remotely advertised
   branch. Do not invent a suffix from an abbreviated SHA.
4. Install/build against the same exact clean core revision. Explicitly set
   `SASE_CORE_DIR` to the audited checkout when needed, and compare its full HEAD with
   the pin before and after installation: setup can refresh a linked checkout. If they
   differ, align the intended revision and repeat relevant verification; do not present
   a test against newer core as an exact-pin test. Record the installed binding/build
   identity.
5. Run `tools/check_sase_core_rs_bindings` through the workspace interpreter, and the
   targeted `tests/test_managed_tmp_reaper.py` and
   `tests/test_check_sase_core_rs_bindings_tool.py` tests. Verify all current required
   bindings, not just the seven from the first incident.
6. Follow audited `lint_and_test.md`: run `just check`; because the pin is a broadening
   change, also satisfy its required `just check-full` gate through a monitor. Repair
   only demonstrated failures and record any reproduced CI/proc/queue flakes. No new
   implementation-mirroring test is needed for a one-line pin correction; remote object
   identity is the missing check.
7. Record the full incorrect/correct hashes, remote reachability, build identity, and
   check results on `sase-10w.5`. Have the host land the repair through the normal final
   declaration, keeping this bead open. Resume at step 2 only after confirming the
   actual landed commit and pin.

## 2. Obtain green CI and a complete contexts artifact

1. Require a successful Master Gate on the current master tip containing the repaired
   pin. Locate by exact `headSha`, not just newest run ID. If master advances before
   closeout, refresh this check on its new tip.
2. For red jobs, inspect failed step logs. Retry infrastructure failures or suspected
   flakes once on unchanged code and record both attempts. A repeated `not our ref` is a
   pin/reachability error requiring step 1, not another blind rerun. For test failures,
   check the active domain owner's landed fixes before making a narrow repair. Keep the
   bead open until the repaired revision itself has passed the gates.
3. List `full.yml` runs before dispatching. Reuse a qualifying successful or in-flight
   master run containing the repair; otherwise run
   `gh workflow run full.yml -R sase-org/sase --ref master`. Capture the actual run ID,
   SHA, event, and attempt. Observe it through a monitor. Full CI's single concurrency
   group does not cancel the active run, but a newer pending run can replace an older
   pending run; follow the actual qualifying replacement if this happens.
4. Require overall success and successful required jobs. Only workflow- conditioned
   skips count: `docs-build` and `release-core-floor-smoke` on master, and
   `contention-test` for a dispatch rather than schedule. Upstream failure causing
   downstream jobs to skip is not success.
5. Inspect the run's artifacts with
   `gh api repos/sase-org/sase/actions/runs/<run-id>/artifacts`. Require an unexpired,
   nonempty `sase-coverage-contexts-<full-sha>` and a successful coverage-contexts job.
   Download/read the database to confirm `.coverage` exists and contains per-test
   contexts; upload steps run under `always()` and cannot alone prove a complete
   successful producer. Preserve run URL, attempt, artifact ID/name/size, and full SHA
   in the evidence note.

## 3. Install the proven baseline and demonstrate its use

1. Use a clean workspace containing the green Full CI revision, with
   `git merge-base --is-ancestor <full-ci-sha> HEAD` succeeding and commit distance at
   most 50. Fast-forward the current clean workspace when needed; never discard other
   work. All permanent repair changes must already be landed before making the proof
   diff.
2. Capture `just selection-health --json` before the proof; preserve a clean JSON result
   separately from Justfile headers. Record scoped run count, escalation rate,
   median/p90 duration, context contribution, and the `rule_histogram` entries for
   stale/missing contexts, depth boost, and serial budget. Keep full evidence in a
   durable artifact or concise bead note.
3. Run `GH_REPO=sase-org/sase just refresh-contexts-baseline` and check the selected
   full SHA against the green producer. The fetcher does not filter run conclusion, and
   `--force` only re-downloads; it does not choose a run. Increase `--limit` only if the
   target run was not searched.
4. If the fetcher chooses another artifact, verify that producer independently or
   download the exact green run with
   `gh run download ... --name ... --dir <staging-dir>` and install its `.coverage`
   using the existing
   `tools/install_coverage_contexts --database <file> --sha <full-ci-sha>`. Keep normal
   dirty/partial/thin guards enabled. This targeted fallback uses the same host cache;
   do not alter fetch/selection behavior just to pass the proof. Resolve the cache
   location via its existing helper rather than assuming an obsolete unnamespaced
   directory.
5. Verify baseline readability and breadth metadata. The selector ranks ancestors by
   attribution breadth and then distance; a cached fresh file does not guarantee it will
   win. If an older broader baseline wins, diagnose producer quality/selection and keep
   the acceptance criterion unsatisfied. Do not delete another workspace's useful
   baseline or raise freshness limits to manufacture a pass.
6. Choose a small leaf Python module under `src/sase/` with tests and existing coverage
   attribution. Save its original bytes and add one temporary comment on a covered
   source line. Avoid core pin/config/Justfile/root conftest/data assets/selection
   tooling. Use `tools/select_tests --explain` to confirm a meaningful selection and
   identify unexpected broadening.
7. Run **`just check`** on that small diff through an ordinary monitor with a success
   continuation. This fulfills the original scoped-check acceptance and the
   temporary-edit verification rule. Inspect and preserve
   `.pytest_cache/sase-selection/manifest.json` before another check overwrites it.
   Require a passing check, `contexts.consulted == true`, `contexts.stale == false`,
   `contexts.baseline == <green-producer-sha>`, and `contexts.distance <= 50`. Require
   none of `context-baseline-stale`, `context-baseline-missing`, or
   `no-baseline-depth-boost` in `rules_fired`. Record matched files, context-selected
   count, total selected count, escalation, outcome, and duration. Prefer a probe with
   actual attributed tests over one that merely reports consultation.
8. Budget/ratio escalation after context consultation does not invalidate the baseline
   proof, but obey the verification note for unusual/escalated selections and record
   residual cost as a follow-up. An early `core-identity-changed` escalation does
   invalidate the proof: stabilize the build identity and repeat on the leaf diff.
9. Restore only the temporary edit, including after a failed proof/monitor. Confirm the
   original bytes and clean tracked diff. Capture after-health metrics and distinguish
   this proof's result from the cumulative fleet history; one new run cannot erase weeks
   of stale-baseline counts.

## 4. Close the phase with evidence and release the existing waiter

1. Before closing, refresh `sase agent list -a -j` for `sase-10w.6`, its waiting marker,
   and `sase axe status -j`. Inspect the configured `wait_checks` and
   `sidecar_auto_sync` entries from `sase axe chop list -j`. Verify the named
   predecessor's successful outcome as well as the pending bead dependency. Record the
   pre-close successor identity/PID, wait fields, and absent start timestamps.
2. If the existing waiter is dead/missing, diagnose and arrange its supported recovery
   before closure rather than assuming a chop will create a new runner. Prefer
   preserving the assigned identity and stored prompt. Inspect
   `sase agent restart sase-10w.6 --dry-run` for collateral effects; use the current
   `/sase_run` LaunchApproval requirements for any agent-requested relaunch. Do not
   force-wipe a live runner, rerun the entire epic, or launch a duplicate to hide failed
   dependency resolution.
3. Append one evidence note on `sase-10w.5` with both green CI run URLs/SHAs, Full CI
   attempt and artifact facts, cache installation, complete proof manifest facts,
   before/after health, and remaining release-floor context for phase 6. Add
   `PROPOSED FOLLOW-UP:` for residual budget/staleness dominance or confirmed flaky
   failures. Do not require historical aggregate escalation to fall immediately as a
   condition of this bead's closure.
4. Recheck the current master gate, Full CI ancestry/freshness, and clean tracked tree.
   Run `sase bead epic-symbols sase-10w.5` immediately before closure. If entries
   appeared, resolve or re-key them to a still-open owning bead, verify and land the
   change, and refresh invalidated CI/proof evidence before closing.
5. Only once all criteria hold, run:

   ```sh
   sase bead close sase-10w.5 --note "Verified Master Gate <id>/<sha> green; Full CI <id>/<sha> green with contexts artifact <id/name>; baseline installed; scoped just check passed consulting that fresh baseline with no stale/missing/depth-boost rules; successor wait preflight passed."
   ```

   Substitute the actual evidence. Do not use `--no-push`; allow the bead command's
   normal publication. Confirm closure through `sase bead show`.

6. Observe automatic propagation: published close → canonical beads sync → wait_checks
   readiness (or the runner's fallback) → dependency barrier completion → runner
   admission → provider start. Allow several normal ticks through a bounded monitor,
   with success/failure/timeout continuations back to this step. A five-minute first
   observation budget covers multiple quiet backstops; it is a diagnostic budget, not
   permission to assume start.
7. Count success only after `sase agent list -a -j` shows the same intended successor
   executing, backed by `wait_completed_at` and `run_started_at` and/or provider
   execution output/checkpoint written after closure. If it starts and finishes between
   observations, completed execution evidence is also valid. `WAITING`, an old live PID,
   a transient `ready.json`, or `QUEUED` alone is insufficient. A queued runner has
   passed dependency resolution but still needs admission; continue monitoring its slot
   state.
8. If still dependency-waiting, inspect sync and wait-check results. After checking
   normal progress, run the supported chops sequentially if needed:

   ```sh
   sase axe chop run sidecar_auto_sync -L waits -V
   sase axe chop run wait_checks -L waits -V
   ```

   These are real operations; `-n` only suppresses proposed launches and is not a
   general no-write sandbox. Do not hand-edit waiting/ready markers, closed IDs, or
   canonical sidecar files. Resolve concrete sync/runtime errors through supported
   commands; keep recording the actual blocker if start is not yet verified. Do not
   reopen the completed phase just to retrigger an event.

9. Append a post-close note to `sase-10w.5` recording the close time, observed
   barrier/start times, successor identity, and whether normal ticks, runner fallback,
   or a manual chop invocation released it. A note on a closed bead is supported. Finish
   through `/sase_final` and report that phase 5 is closed and phase 6 started; do not
   wait for or perform phase 6's release.

## Completion checklist

- Correct full core object is published and matches the revision verified.
- Current Master Gate is green; a qualifying Full CI and contexts producer are green,
  with usable artifact provenance.
- Host cache installation and a passing scoped `just check` prove fresh consultation;
  the temporary edit is gone and evidence is durable.
- `sase-10w.5` is closed with no stale epic symbols, and the existing `sase-10w.6` agent
  has demonstrably started after dependency release.
- Parent/release work remains with its owners. Until successor start is verified, report
  phase closure and startup verification as distinct states.
