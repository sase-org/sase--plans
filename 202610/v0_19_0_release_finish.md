---
tier: epic
title: Finish sase v0.19.0 by clearing the last blocking reds, pinning a published
  containing core, rehearsing, and watching the steward release
goal: 'Master Gate is green at master HEAD; a complete published sase-core-rs contains
  master''s core revision and master pins it exactly; a release.yml rehearsal at master
  is accepted; the release steward merges release-please PR #322; sase 0.19.0 is on
  PyPI; and the final lander records the release evidence as an artifact.

  '
phases:
- id: gate-reds
  title: Clear the footprint microbench, prompt-bar reveal, and any other blocking-lane
    red at the tip
  size: medium
  depends_on: []
  description: 'gate-reds: make Footprint bump() meet its 2 µs budget, measure it
    untraced, fix the hot-spare reveal subtitle race, then fix every other deterministic
    Master Gate or blocking Full CI red.'
- id: core-pin
  title: A complete published sase-core-rs contains master's source pin, and master
    pins it
  size: medium
  depends_on: []
  description: 'core-pin: wait for the sase-core daily cut (v0.43.3, PR #342) to publish
    a complete core containing pin 80f5af40 or later, then ratchet master''s exact
    pin and prove the bindings.'
- id: rehearse
  title: Prove a green release.yml rehearsal at master
  size: small
  depends_on:
  - gate-reds
  - core-pin
  description: 'rehearse: dispatch a release.yml rehearsal at master, judge it job
    by job, note the result on sase-1oe.4, and wake its cutover per the parent''s
    wake_cutover = wake.'
- id: watch
  title: Watch sase v0.19.0 through the steward, release.yml, and PyPI with a self-closing
    monitor
  size: small
  depends_on:
  - rehearse
  description: 'watch: after prerequisite and preflight checks, start a no-follow-up
    monitor whose watcher observes the steward, release.yml, and PyPI for up to 48h
    and always closes this bead.'
proposed_by: bbugyi200.athena.sase-1q2.land
proposed_with: claude/opus
parent_bead: sase-1q2
create_time: 2026-10-31 05:23:20
status: wip
bead_id: sase-1q2.9
---

- **PROMPT:** [prompts/202610/v0_19_0_release_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_19_0_release_finish.md)
- **PARENT:** [202610/v0_19_0_release.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_19_0_release.md)
- **BEAD:** [sase-1q2.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1q2/sase-1q2.9.md)

# Finish sase v0.19.0 by clearing the last blocking reds, pinning a published containing core, rehearsing, and watching the steward release

This epic is nested under **sase-1q2** ("Fix every release blocker and ship sase v0.19.0
through the release steward", plan `202610/v0_19_0_release.md`). Read that plan for the
release model. In short:

- The release steward (`.github/workflows/release-steward.yml` plus
  `tools/release_watch`) is the only thing that merges release-please PR **#322**
  (`chore(master): release 0.19.0`). `ci_watch` no longer merges sase releases.
- The steward merges only when all of these hold: merge authority is on
  (`RELEASE_STEWARD_MERGE_ENABLED` is `true`, which only the human-approved E2 cutover
  gate from **sase-1oe.4** flips); Master Gate is green at master HEAD; the release PR's
  required checks are green; and a complete published `sase-core-rs` contains
  `sase-core-revision.txt`.
- `release.yml` then qualifies the tag by calling `ci.yml`. The blocking jobs are `lint`
  and `test (3.12|3.13|3.14)`, and the 3.12 leg runs under coverage (`just test-cov`).
  `visual-test`, `perf-floors`, `tutorial-tests`, and `ace-page-group-isolation` are
  advisory. It then runs four exercise legs and publishes to PyPI.

The target is still **0.19.0**. "Observe the release" in this epic and in any further
nested epic means 0.19.0.

## What sase-1q2 already finished

Every one of these commits is on master:

| Commit       | Phase      | Fix                                                                                                                                         |
| ------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `deb8bb60b6` | sase-1q2.1 | `tools/_release_core.py` and `tools/release_candidate` count compressed macOS wheel tags as complete.                                       |
| `1a419071c2` | sase-1q2.2 | master's exact pin moved to `sase-core-rs==0.43.2`, which then contained pin `17e79c8c`.                                                    |
| `3f9fe9c107` | sase-1q2.3 | agent sync help assertion and agent load hostname fixed.                                                                                    |
| `e28334c45a` | sase-1q2.4 | pager notify teardown, TUI import budget, tool-runs visual sentinel, and visual drift fixed; footprint bench moved to warm best-of-batches. |
| `dab2d701fb` | sase-1q2.5 | suite-gate import closure and nested-preservation `OSError` wrapping fixed.                                                                 |
| `eec20f6ad5` | sase-1q2.6 | `handle_agents_sync` NameError and the prompt-format race fixed; the oversized monitor test file was split.                                 |

The lander closed tasks sase-1ov, sase-1py, sase-1px, sase-1pz, sase-1q7, and sase-1q1,
which these commits resolved.

## Evidence (observed 2026-10-31, about 09:30 UTC, master `eec20f6ad5`)

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Phase                  |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| 1   | **Master Gate is red on every push since 07:13 UTC, always in `tests/footprint/test_tui.py::test_bump_microbench_stays_within_two_microseconds`.** The best of 5×5000 batches was 3.151, 3.299, 3.319, and 3.380 µs in test (4) of runs 41182683159, 41181251890, 41181712170, and 41185361724 (bound 2 µs). Full CI 41174188852 measured 2.603 µs on 3.13 and 11.116 µs on the 3.12 coverage leg. Earlier Full CI 3.12 coverage legs measured 6.5 to 10.7 µs. On athena, an unloaded `bump()` costs about 1.23 µs. sase-1q2.4 already switched to a warm best-of-batches measurement, so this is not runner noise: the product misses its documented budget on CI runners, and line tracing makes it impossible to meet under coverage. The other shards of 41185361724 are green, so this is Master Gate's only red. | gate-reds              |
| 2   | **`tests/ace/tui/test_prompt_bar_hot_spare.py::test_reveal_matches_fresh_home[pairs1-]` failed on Full CI 41174188852 in test (3.13) and test (3.14),** which are blocking qualification lanes. The revealed hot spare's `border_subtitle` was `[dim]Ln …1…, Col …`, with no key hints and the column truncated to `…`. A fresh mount shows `[Enter] launch…  [Esc] normal  [^C] cancel · Ln 1, Col 1`. The test snapshots right after `_wait_for_active` plus one `pilot.pause()`, so the revealed bar is published before its width-dependent subtitle has been recomputed for the visible width. sase-1q2.6 could not reproduce it locally in 20 serial runs.                                                                                                                                                       | gate-reds              |
| 3   | **No published core contains master's source pin.** `sase-core-revision.txt` is `80f5af40` (moved there by sase-1q4's `eed4ef925b`, after `271632e9d7` and `cb230adb5e`). PyPI's newest complete core is 0.43.2. `python3 tools/ratchet_core_pin --check --fetch --ref origin/master` exits 3 with `CORE_LAG(pin=80f5af40…, newest_complete=0.43.2)`. sase-core master is at `80f5af40` and no tag contains it. sase-core release PR **#342** (`chore: release v0.43.3`, head `a05cb015`, all checks green) waits for the daily cut (`41 7 * * *` in `.github/workflows/release-plz.yml`). GitHub delays that cron by hours: the 2026-10-30 cut ran at about 14:29 UTC. The six-hourly heal schedule and push runs never merge.                                                                                        | core-pin               |
| 4   | **No rehearsal since the resolver fix.** sase-1q2.7 closed canceled on the CORE_LAG above. The last `release.yml` run is still 40234446645 (2026-10-25, red).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | rehearse               |
| 5   | **The cutover is still behind a stale poll.** sase-1oe.4 is IN_PROGRESS. Its monitor `c6xp53d5fv86` (member `sase-1oe.4--mon-2`, label "sase-1oe.4 core publication") polls with its old checkout's pre-fix matcher and times out around 2026-11-01 02:40 UTC. sase-1n6.10's monitor waits for the settings flip.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | rehearse (wake), watch |

**Advisory-lane flakes, already tracked; do not fix them here:** sase-1oa (stranded
resolve modal), sase-1nx (final rows), sase-1qb (capacity inspector), sase-1qc (renamed
session root), and sase-1qd (Files anchor trailing reserve).

## Rules for every phase

- **Read memory first.** Read `lint_and_test.md` and `sase_beads.md` through
  `/sase_memory_read`. Also read:
  - `tui.md` for TUI or visual work;
  - `ace.md` before any change under `src/sase/ace/`;
  - `tools.md` before changing anything under `tools/`;
  - `symvision.md` if a symvision gate fails.
- **Re-inventory first.** Fetch, then re-inventory on the current master tip before you
  edit anything. Other epics land on master continuously. The Evidence above is a
  starting point, not a whitelist.
- **Respect ownership.** Before you fix a red, run `sase bead search '<test name>'`.
  - If an IN_PROGRESS phase whose agent is live (check `sase agent list`) is already
    fixing it, leave it alone and say so in your close note.
  - Otherwise fix it, and name every existing task bead your evidence resolves.
- **Fix waits, fixtures, and products, never tolerance.** Do not raise timeouts,
  budgets, caps or perf bounds. Do not add retries around assertions. Do not skip or
  xfail. Do not loosen pixel comparison or delete assertions.
  - Wait on the semantic state an assertion is about, using
    `sase.ace.testing.wait.wait_for` or the file's existing helper.
  - If the product settles late or inconsistently, fix the product.
  - A perf test may change how it measures (for example, best-of-N batches, or timing in
    an untraced child interpreter) only when the product did not regress. It never gets
    a looser bound.
- **Goldens.** Never copy CI candidate PNGs into the tree. Regenerate a golden only
  after its test is deterministic.
- **Rust boundary.** Shared domain behavior belongs in sase-core
  (`rust_core_backend_boundary`). None of these fixes is expected to need a Rust change.
  If one does, work through `sase repo open sase-core` and run that repo's own checks
  there.
- **Verify.** Run `just fix`, then `sase tool run check`, in every checkout you changed.
  Do not run `just check-full` (`decisions:check-full-is-explicit`). The rehearsal's
  qualification is this epic's exhaustive CI signal. The CI `lint` job enforces
  `toobig`, so do not grow a file past its line limit.
- **Hand waits to monitors.** Hand every long command to `/sase_monitor` with an
  explicit deadline. That covers verification, CI waits, and PyPI waits. Never end a
  turn waiting.
- **No merges, PRs, settings changes, or core cuts.**
  - Never merge #322 or any sase-core PR (including #342).
  - Never run `gh variable set`, and never edit the `pypi` environment.
  - Never dispatch a sase-core cut (`release-plz.yml`). Sibling epics sase-1oe and
    sase-1pi run under the approved `core_cut = wait`.

  Allowed: `gh workflow run release.yml` with `mode=rehearse` (rehearse only), and
  `gh run rerun --failed` where a phase says so.

- **Leave E2's work to E2.** Do not close, edit or reassign sase-1n6, sase-1n6.10,
  sase-1oe, sase-1oe.4, sase-1pi, or any of their phases. Notes on those beads are
  allowed where a phase says so. Edit no memory.
- **No new beads; always close.** Phase workers create no beads. Record follow-ups as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead. To close:
  1. Run `sase bead epic-symbols <own-phase-id>`; it must list nothing.
  2. Close with `sase bead close <id> -R done -n '<evidence>'` and finish through
     `/sase_final`.

  If you cannot finish, still close with `-R canceled` and the evidence, so dependent
  phases and the land agent always run.

## Phase gate-reds

**Repo:** sase. Read `tui.md`, plus `ace.md` before touching `src/sase/ace/`.

1. **Footprint `bump()` budget** (Evidence #1). The bound is the product's documented
   contract ("the only call allowed on the event loop", 2 µs). Fix the product first,
   then make sure the bench measures the product rather than a tracer.
   1. Fix the product. Make `src/sase/footprint/_tui.py` `bump()` cheaper without
      changing what it does. It must still honor the `SASE_FOOTPRINT=off` kill switch,
      validate the via, collapse invalid IDs to `~other`, drop `ids.NAVIGATION_ACTIONS`,
      start the flusher lazily, reset after fork, never raise, and never do I/O. On
      athena, the per-call costs measured with best-of `timeit` were:
      - the `os.environ.get("SASE_FOOTPRINT")` read: about 286 ns;
      - `os.getpid()`: about 185 ns;
      - `ids.is_valid_feature_id` (regex): about 232 ns;
      - `_ensure_flusher` (`Thread.is_alive`): about 94 ns;
      - the locked dict increment: about 216 ns.

      Likely levers:
      - Detect forks with `os.register_at_fork(after_in_child=…)` registered once at
        import, as the module docstring's `sase.telemetry._registry` model does, instead
        of a `getpid()` per hit.
      - Resolve the kill switch once per process state, clearing it in
        `_reset_for_tests()` and after fork. Check every test that sets `SASE_FOOTPRINT`
        (`tests/footprint/`) still holds.
      - Memoize each raw feature ID's verdict (valid, collapsed, or dropped) in a
        bounded dict, so steady-state hits skip the regex.
      - Keep the lock: the flusher swaps the dict.

      Aim for at least a 2× drop in the bench's own warm best-of measurement on athena,
      and record the before and after numbers.

   2. Measure the product, not the tracer. Full CI's `test (3.12)` leg (and release
      qualification's) runs `just test-cov` with `--cov-branch`, and line tracing
      multiplies any microbench several-fold. Run the timed loop where no tracer is
      active, for example in a child interpreter (`sys.executable`) whose environment
      drops pytest-cov's and coverage's subprocess hooks. Keep the warm-up, the
      best-of-5×5000 method, and `best <= 2e-6`. The bench must keep running in Master
      Gate's fast lane. Do not mark it `slow`, skip it, or xfail it.
   3. Prove it. Run the node 20 times serially and the file 3 times under `-n 8`. Run
      the node under coverage
      (`.venv/bin/python -m pytest --cov=src/sase --cov-branch <node>`) and show it
      passes untraced. Then run it with 8 busy `-n 8` siblings, for example alongside
      `tests/ace/tui/`, to approximate a loaded runner.

2. **Hot-spare reveal parity** (Evidence #2). Find where the revealed spare publishes
   `app._active_prompt_bar` and where the bar's border subtitle (key hints plus
   `Ln/Col`) is computed against the bar's width. A user sees the truncated subtitle for
   at least one frame, so fix the product: compute the subtitle for the visible width
   before or during the reveal, or re-render it synchronously on reveal, so the reveal
   matches a fresh mount. Only if the product is already correct and the snapshot merely
   runs early, wait in the test on the semantic state (the subtitle computed for the
   final width) with `wait_for`. Never sleep, retry, or drop the subtitle from the
   parity snapshot. Prove it: run the node 20 times serially under Python 3.13 or 3.14
   (the failing lanes), and run the file 3 times under `-n 8` alongside
   `tests/ace/tui/`.
3. **Re-inventory.** Look at the newest Master Gate at master HEAD and the newest Full
   CI run (`gh run list -R sase-org/sase --workflow master-gate.yml` / `full.yml`). Fix
   every deterministic red in a blocking lane: Master Gate's `lint`, `docs`,
   `core-wheel`, and `test (N)` jobs, and Full CI's `lint` and `test (3.12|3.13|3.14)`
   jobs. For a flake, find the race. If you cannot, record a `PROPOSED FOLLOW-UP:` with
   the output and the existing owner bead. Leave advisory-lane reds to the tasks listed
   in Evidence.
4. Run `just fix` and `sase tool run check`, then land. Name every task bead your
   evidence resolves in the close note.

**Done when** both fixes pass the proof above, every other deterministic blocking-lane
red at the tip is fixed with the same proof, and the repo checks pass.

## Phase core-pin

**Repo:** sase, plus read-only checks in sase-core
(`sase repo open sase-core -r '<why>'`).

1. **Check the pin.** Fetch, then run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: master already pins a complete containing core. Record it, then go to
     step 4. No commit is needed; say so in the close note.
   - exit 2: a ratchet is pending; go to step 3.
   - exit 3 with `CORE_LAG`: go to step 2.
2. **Wait for a containing core to publish.**
   1. In the opened sase-core checkout, run `git fetch --tags` and
      `git tag --contains <pin>`.
   2. If a tag contains the pin, find its `Release-plz` push run with
      `gh run list -R sase-org/sase-core --workflow release-plz.yml --event push`. Rerun
      it once with `gh run rerun <id> --failed -R sase-org/sase-core` only if every one
      of these holds:
      - only the upload or attestation steps failed (`publish to PyPI`);
      - the failure is transient infrastructure (an HTTP 5xx from Sigstore, Rekor or
        PyPI, or a network timeout);
      - the run is still at attempt 1;
      - no newer attempt or run for that tag is queued or running.

      If the failure is deterministic (a build, dist-gate or twine error), record a
      `PROPOSED FOLLOW-UP:` with the job ID and log lines, and close `-R canceled`.

   3. If no tag contains the pin yet (expected at planning time: PR #342 waits for the
      daily cut), wait through sase-core's scheduled daily cut. Never dispatch one and
      never merge #342.
   4. Hand the wait to `/sase_monitor` with a follow-up (`--next`). Allow about 6h after
      a rerun, or about 36h for a daily cut. The poll fetches, then runs
      `python3 tools/ratchet_core_pin --check --fetch --ref origin/master` every 300 s
      from this phase's own checkout until the exit code leaves 3. If another epic moves
      `sase-core-revision.txt` again, the poll simply waits for a cut containing the new
      pin. In the follow-up, rerun the check at once.
   5. If the rerun fails again or the wait times out, record the run and job evidence as
      a `PROPOSED FOLLOW-UP:` and close `-R canceled`.

3. **Ratchet master.** In a checkout fast-forwarded to `origin/master`, run
   `python3 tools/ratchet_core_pin --allow-transitive-lock-refresh`, then `uv lock`.
   - The diff must touch only `pyproject.toml` and `uv.lock`, and should give
     `sase-core-rs==0.43.3` (or whatever version step 2 produced).
   - Keep `sase-core-revision.txt` unchanged.

   Then reproduce the `release-core-floor-smoke` job from `.github/workflows/ci.yml`:
   1. Create a fresh `uv venv --python 3.12` outside the repo.
   2. Run `uv pip install -e '.[dev]' "sase-core-rs==<X>"`.
   3. Run `tools/check_sase_core_rs_bindings`, then `tools/validate_sase_core_rs`.

   It must report 0 missing bindings. Paste the output into your close note.

4. **Note on sase-1oe.4.** Record the complete containing core (version, peeled tag SHA,
   and all five PyPI filenames) and that this phase ratchets master's exact pin.
5. Run `just fix` and `sase tool run check`, then land.

**Done when** the check reports a complete containing core, master's exact pin is
ratcheted to it (or already was), and the floor smoke reports 0 missing bindings.

## Phase rehearse

**Repo:** sase. Read the `/sase_monitor` skill first. This phase changes no files.

1. **Prerequisites.** gate-reds and core-pin must be closed `done`, and their commits
   must be on `origin/master`
   (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note says
   it needed no commit is fine. Re-fetch for up to about 5 minutes if needed. If either
   one closed `canceled`, close this phase
   `-R canceled -r "rehearsal not started: <phase> canceled"` with its close note.
2. **Pin.** In a refreshed checkout, run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: no override;
   - exit 2: add `-f core_version=<core-ensure pick>`, as sase-1oe.4's cutover step 1
     does, and record why;
   - exit 3 (`CORE_LAG`): the pin moved past the published core again after core-pin
     closed. Do not cancel yet. Hand a wait to `/sase_monitor` with `--next` and about a
     36h deadline that polls the same check every 300 s until it leaves 3. In the
     follow-up, redo this step. Close `-R canceled` with the evidence only if that wait
     times out.
3. **One rehearsal at a time.** Check that no `release.yml` run is queued or in progress
   (`gh run list -R sase-org/sase --workflow release.yml`).
   - If one is, and it is a `rehearse` run at a master SHA that contains every phase
     commit, watch that run instead of dispatching.
   - If it is some other run, wait for it to finish first.
4. **Rehearse.** Dispatch
   `gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse`, plus
   the override from step 2 if any. Find the run ID. Hand it to `/sase_monitor` with
   `--next` and about a 4h deadline, for example
   `gh run watch <id> -R sase-org/sase --interval 120 || true`. The run's overall
   conclusion is red even when everything is healthy (see the `publish` criterion
   below), so the follow-up must judge it job by job.
5. **Acceptance.** Accept the rehearsal only if all of the following hold:
   - `identity`, `build` and `qualify-prepare` are green.
   - `qualify` is green, which requires every blocking job green: `lint` and
     `test (3.12|3.13|3.14)`. An advisory red (`visual-test`, `perf-floors`,
     `tutorial-tests`, `ace-page-group-isolation`) does not block the release. Record it
     as a `PROPOSED FOLLOW-UP:`, but it does not fail acceptance.
   - All four `exercise-legs` and the merged `exercise` verdict are green. Both wheel
     legs name a resolved previous version (expected 0.18.1) and pass the N-1 upgrade.
   - `publish` (verify-only) fails only with the expected `PUBLISH_CONFLICT` for every
     sealed file. Master's version, 0.18.1, equals the last release, so the
     different-hash refusal is the designed outcome (`release-train.md`, "Publish and
     confirm"). Any other refusal is red: a predicate red, `PUBLISH_MISSING_ASSET`, or
     `PUBLISH_STASH_MISMATCH`.
   - `confirm` and `confirm-install` are skipped. They never run in `rehearse`.
6. **Note on sase-1oe.4.** It records:
   - the rehearsal URL and the verdict for each criterion;
   - the master SHA and its core pin;
   - the core-match commit (`deb8bb60b6`) and the core-pin ratchet commit;
   - that its running poll uses the pre-fix matcher from its old checkout.
7. **Wake the cutover** (accepted rehearsals only). The parent epic's final reviewer
   decision is `wake_cutover = wake`; it binds this phase.

   Act only if sase-1oe.4 is still IN_PROGRESS and its active monitor is the
   core-publication poll. Check with `sase monitor list` and `sase monitor show <id>`:
   the label is "sase-1oe.4 core publication" and the command runs
   `ratchet_core_pin --check`.
   1. Start step 6's note with this line: "WAKE: fast-forward this phase's checkout to
      origin/master before running ratchet_core_pin; the old checkout's matcher rejects
      maturin 1.16 macOS filenames."
   2. Run `sase monitor stop <monitor-id>`, then `sase monitor resume <monitor-id>`.
      Stop launches no follow-up. Resume launches the recorded follow-up from the frozen
      result without rerunning the command.
   3. Confirm with `sase agent list` that a sase-1oe.4 follow-up is queued or running.
      Record the monitor ID and the outcome in your close note.

   Retry a failed stop or resume at most once. Then record it and move on: sase-1oe.4
   still wakes at its own timeout. If sase-1oe.4 is closed, or is waiting on something
   else (its own rehearsal, a steward verdict, a release-PR sync), leave it alone.

8. **Close.** If the rehearsal was accepted, close `-R done` with the run URL and the
   verdict for each criterion. Otherwise, record the run URL, failing job and failing
   step as a `PROPOSED FOLLOW-UP:`, and close `-R canceled`. Either way, finish through
   `/sase_final`.

## Phase watch

This phase makes **no repo edits**. It ends by starting one monitor with **no
`--next`**, so no LLM turn runs after it and nothing can hang. The watcher script closes
this bead itself, and the monitored command always exits 0. A monitor member that
settles `completed` or `stopped` with no follow-up satisfies the land agent's
agent-session wait.

1. **Idempotent start.** `sase bead read` this bead. If it is already closed (a relaunch
   after recovery), run `/sase_final` and stop.
2. **Already released?** If `curl -fsS https://pypi.org/pypi/sase/0.19.0/json` succeeds,
   close this bead with that evidence in `--note` and run `/sase_final`.
3. **Prerequisites.** Fetch, then read each sibling phase's bead.
   - A phase that closed `canceled` means a prerequisite failed. Close this bead
     `-R canceled -r "release watch not started: <phase> canceled"` with that phase's
     close note, then run `/sase_final`.
   - Confirm each `done` sibling's commit is on `origin/master`
     (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note
     says it needed no commit is fine. Re-fetch for up to about 5 minutes before you
     declare a commit missing.
   - Run `python3 tools/ratchet_core_pin --check --fetch --ref origin/master` in the
     refreshed checkout and record the result. Exit 3 is a warning, not a cancel: the
     release then waits for sase-core's daily cut.
   - Confirm that a PR titled `chore(master): release 0.19.0` is open or merged
     (`gh pr list -R sase-org/sase --state all --search 'in:title "release 0.19.0"'`).
     If release-please retargeted the version, close
     `-R canceled -r "release watch not started: release target changed"` with the
     evidence.

   Any other failure also closes this bead `-R canceled` with the evidence, followed by
   `/sase_final`. Do not repair code in this phase.

4. **Epic-symbol guard.** `sase bead epic-symbols <this-phase-id>` must list nothing.
   Otherwise, close `-R canceled -r "release watch not started: epic-symbol entries"`
   and run `/sase_final`.
5. **Write the watcher outside the repo.** Use
   `W=/tmp/sase-release-watch-<this-phase-id>` and `mkdir -p "$W"`. Write `$W/watch.sh`
   with a quoted heredoc (`<<'EOF'`), so nothing expands at write time:

   ```sh
   #!/bin/sh
   # sase v0.19.0 release watcher: sh watch.sh BEAD LOG [preflight]
   # Never edits files. In watch mode it closes BEAD itself and always exits 0.
   BEAD=$1 LOG=$2 MODE=${3:-watch}
   REPO=sase-org/sase VERSION=0.19.0 TAG=v0.19.0
   TITLE="chore(master): release $VERSION"
   RED_STATES=" qualify_red exercise_red core_cut_red core_ci_red core_incompatible publish_conflict dependency_unpublished vetoed ambiguous_trains watcher_broken "
   unset GH_FORCE_TTY
   export NO_COLOR=1 CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0
   START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   END=$(( $(date +%s) + 172800 ))
   outcome=DEADLINE detail= reruns= seen_sid= steward=? red_ticks=0 train_red_since=
   say() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" | tee -a "$LOG"; }
   while :; do
     pypi=$(curl -fsS "https://pypi.org/pypi/sase/$VERSION/json" 2>/dev/null \
       | jq -r '[.urls[].packagetype] | (index("bdist_wheel") != null and index("sdist") != null)' 2>/dev/null)
     [ -n "$pypi" ] || pypi=absent
     if [ "$pypi" = true ]; then outcome=RELEASED; break; fi
     pr=$(gh pr list -R "$REPO" --base master --state all \
       --search "in:title \"release $VERSION\"" --json number,title,state \
       --jq "[.[] | select(.title == \"$TITLE\")][0]
             | if . == null then \"none\" else \"\(.number):\(.state)\" end" 2>>"$LOG")
     auth=$(gh api "repos/$REPO/actions/variables/RELEASE_STEWARD_MERGE_ENABLED" --jq .value 2>/dev/null) || auth=unset
     train=$(gh run list -R "$REPO" --workflow release.yml --event workflow_dispatch --limit 20 \
       --json databaseId,status,conclusion,attempt,headBranch \
       --jq "[.[] | select(.headBranch == \"$TAG\")][0]
             | if . == null then \"none\" else \"\(.databaseId):\(.status):\(.conclusion):\(.attempt)\" end" 2>>"$LOG")
     case "$train" in
       *:completed:failure:*|*:completed:cancelled:*|*:completed:timed_out:*|*:completed:startup_failure:*)
         now=$(date +%s)
         [ -n "$train_red_since" ] || train_red_since=$now
         if [ $((now - train_red_since)) -ge 3600 ]; then
           outcome=TRAIN_RED detail="release.yml $train"; break
         fi ;;
       *) train_red_since= ;;
     esac
     sid=$(gh run list -R "$REPO" --workflow release-steward.yml --status completed --limit 1 \
       --json databaseId --jq '.[0].databaseId // ""' 2>>"$LOG")
     if [ -n "$sid" ] && [ "$sid" != "$seen_sid" ]; then
       kind=$(gh run view "$sid" -R "$REPO" --log 2>>"$LOG" \
         | grep -o '{.*"schema_version".*}' | tail -n 1 \
         | jq -r '.primary.kind // "?"' 2>>"$LOG")
       seen_sid=$sid steward=${kind:-?}
       case "$RED_STATES" in *" $steward "*) red_ticks=$((red_ticks + 1)) ;; *) red_ticks=0 ;; esac
     fi
     if [ "$red_ticks" -ge 2 ]; then outcome=STEWARD_RED detail="steward $steward (run $sid)"; break; fi
     head=$(gh api "repos/$REPO/commits/master" --jq .sha 2>>"$LOG")
     gate=$(gh run list -R "$REPO" --workflow master-gate.yml --commit "$head" --limit 1 \
       --json databaseId,status,conclusion,attempt \
       --jq '.[0] | if . == null then "none" else "\(.databaseId):\(.status):\(.conclusion):\(.attempt)" end' 2>>"$LOG")
     case "$gate" in
       *:completed:failure:1)
         id=${gate%%:*}
         case " $reruns " in *" $id "*) ;; *)
           reruns="$reruns $id"
           [ "$MODE" = watch ] && gh run rerun "$id" --failed -R "$REPO" >>"$LOG" 2>&1 ;;
         esac ;;
     esac
     full=$(gh run list -R "$REPO" --workflow full.yml --branch master --limit 10 \
       --json databaseId,status,conclusion,createdAt,headSha \
       --jq "[.[] | select(.status == \"completed\" and .createdAt > \"$START\")][0:2]
             | map(\"\(.databaseId):\(.conclusion):\(.headSha[0:10])\") | join(\",\")" 2>>"$LOG")
     case "$full" in *:failure:*,*:failure:*) outcome=FULL_CI_RED detail="Full CI $full"; break ;; esac
     say "state pypi=$pypi pr=${pr:-?} authority=$auth steward=$steward(${sid:-?}) train=${train:-?} head=${head:-?} gate=${gate:-?} full_since_start=${full:-none}"
     [ "$MODE" = preflight ] && exit 0
     [ "$(date +%s)" -ge "$END" ] && break
     sleep 300
   done
   say "outcome=$outcome $detail"
   [ "$MODE" = preflight ] && exit 0
   if [ "$outcome" = RELEASED ]; then res=done reason="sase $VERSION confirmed on PyPI"
   else res=canceled reason="release not confirmed: $outcome${detail:+ ($detail)}"; fi
   tail -n 60 "$LOG" > "$LOG.note"
   i=0
   while [ "$i" -lt 3 ]; do
     sase bead close "$BEAD" -R "$res" -r "$reason" -n "@$LOG.note" && break
     i=$((i + 1)); sleep 30
   done
   exit 0
   ```

   The script has six outcomes:

   | Outcome       | Trigger                                                                                                                                                                                                                                                                            |
   | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | `RELEASED`    | PyPI serves a wheel and an sdist for 0.19.0. The bead closes `done`.                                                                                                                                                                                                               |
   | `TRAIN_RED`   | A `release.yml` run at `v0.19.0` stays red for 1h. The grace hour lets the steward start a rerun or a new generation first.                                                                                                                                                        |
   | `STEWARD_RED` | Two consecutive completed steward runs report a red primary state from `RED_STATES`. `gate_red`, `core_lag`, `waiting` and `releasable` are not red: master churns, the daily cut heals core lag, and `releasable` with suppressed merges means the human cutover gate is pending. |
   | `FULL_CI_RED` | The first two Full CI runs completed since the start both failed. New reds would fail qualification at the tag.                                                                                                                                                                    |
   | `DEADLINE`    | 48h passed. The steward's daily cadence and a human gate approval both fit inside that window.                                                                                                                                                                                     |

   Every outcome except `RELEASED` closes the bead `canceled`, with the last 60 log
   lines as the note. The watcher reruns a failed first attempt of Master Gate at master
   HEAD once, because the steward will not merge over a red gate. It only reports merge
   authority and the release PR.

6. **Preflight inline.** Run `sh -n "$W/watch.sh"`, then
   `sh "$W/watch.sh" <this-phase-id> "$W/watch.log" preflight`. It must exit 0 and print
   either `outcome=RELEASED` or one `state` line in which these fields are populated:
   `pr`, `steward` (a real kind, not `?`), `head`, and `gate`. `train=none` and
   `authority=unset` are expected before the cutover. Preflight never reruns or closes
   anything.

   If a required field is `?` or the command fails, fix only the script's `gh`, `jq` or
   `grep` invocation, without touching the repo, and retry once. If it still fails,
   close `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` with the
   output, and run `/sase_final`.

7. **Record, then hand off.** Append a phase note with:
   - the master HEAD;
   - the core pin and the `ratchet_core_pin` result;
   - the preflight line;
   - sase-1oe.4's status and whether the E2 cutover gate has been raised;
   - the exact monitor command.

   Then, from the workspace root, run this in the foreground until it exits:

   ```sh
   sase monitor start \
     -C "$PWD" \
     -w 0 \
     -r 'Watch sase v0.19.0 through the release steward, release.yml, and PyPI; the watcher closes <this-phase-id>' \
     -t 50h \
     -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -L 'sase 0.19.0 watch' \
     -- 'timeout -k 5m 48h30m sh /tmp/sase-release-watch-<this-phase-id>/watch.sh <this-phase-id> /tmp/sase-release-watch-<this-phase-id>/watch.log; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 60; sase bead close <this-phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Pass the command as one quoted string with no `bash -c` wrapper. Do **not** pass
   `--next`, `-m`, `-p`, or `--idle-timeout`. A successful start ends this turn, and
   that handoff is exempt from `/sase_final`. If `sase monitor start` exits nonzero
   twice, close `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` with both
   errors, then run `/sase_final`.

### Why the land agent always runs

Each layer ends inside the next one out:

1. The watcher stops at 48h.
2. The outer `timeout` stops at 48.5h, plus 5 minutes of kill grace.
3. The fallback close runs after a 60s pause.
4. The monitor's own 50h timeout never fires on a live host.

A fallback close after a successful close is refused as a conflicting re-close, and
`|| true` absorbs that.

| Path                                                              | Who closes the bead                               | What settles the session             |
| ----------------------------------------------------------------- | ------------------------------------------------- | ------------------------------------ |
| `RELEASED`, `TRAIN_RED`, `STEWARD_RED`, `FULL_CI_RED`, `DEADLINE` | the watcher (3 tries)                             | wrapper exits 0, monitor `completed` |
| watcher hang, or the outer `timeout` fires                        | the wrapper's fallback close                      | wrapper exits 0                      |
| watcher crash, or its close fails                                 | the wrapper's fallback close                      | wrapper exits 0                      |
| prerequisite, epic-symbol, preflight, or start failure            | this agent, inline                                | this agent's `/sase_final`           |
| a human stops the monitor                                         | nobody automatically; the land agent can close it | monitor `stopped` counts as success  |
| reboot (`lost`) or an external SIGKILL                            | nobody                                            | a human: `sase bead work <epic-id>`  |

Step 1 makes recovery safe. `sase bead work <epic-id>` relaunches only the land agent if
this bead is closed, and relaunches the watch if it is open.

## Landing

The land agent confirms the release before it lands anything as done.

1. **If sase 0.19.0 is on PyPI,** verify each item and record it:
   - PyPI (`https://pypi.org/pypi/sase/0.19.0/json`) lists a wheel and an sdist. Record
     the filenames, SHA-256 values, and upload times.
   - `gh release view v0.19.0 -R sase-org/sase` exists and is not a draft. Record its
     tag commit.
   - The merged release PR shows the steward's merge
     (`gh pr view <n> -R sase-org/sase --json mergedBy,mergedAt,mergeCommit`). Record
     the steward run whose train action executed the merge.
   - The `release.yml` run at `v0.19.0`: record its ID and every job's conclusion,
     including `qualify`, `exercise`, `publish`, `confirm` and `confirm-install`.
   - The green Master Gate run at the merge commit, and the newest Full CI run (ID, SHA
     and conclusion).
   - **Clean install proof.** Outside any repo, with an isolated `HOME`, create
     `uv venv --python 3.12` and run `uv pip install sase==0.19.0` with `UV_NO_CACHE=1`.
     `sase version -j` must report 0.19.0, and its `sase-core-rs` must equal the
     release's exact pin. `sase core health --json` must pass.

   Write these into `sase_v0_19_0_release_evidence.md` in a temporary directory outside
   the repo. Register it with
   `sase artifact create -p <file> -l "sase v0.19.0 PyPI release evidence" -b <this-epic-id>`.
   - If this epic is nested, also add a `sase bead note` on the root epic bead that
     names the artifact ref.
   - Add a note on sase-1n6.10 that names the artifact ref. It owns E2's first-release
     proof, so do not close it.
   - Close the task beads that phases named as resolved (for example, sase-1ov, and
     sase-1q1 once its split is on master), each with its evidence.

   Then close this epic.

2. **If 0.19.0 is not on PyPI,** read every phase's close note and the watcher log
   (`/tmp/sase-release-watch-<watch-id>/watch.log`, if present), and fix the blockers:
   - Fix small, clearly scoped blockers yourself, under the phase rules above.
   - For anything that needs CI round-trips, a core release, or waiting, author a
     **nested epic** with `/sase_plan`. It covers only the remaining blockers, plus a
     final self-closing watch phase built exactly like the watch phase above:
     prerequisites, preflight, a no-`--next` monitor, and a wrapper fallback close.
   - If the only remaining blocker is the E2 cutover (merge authority still off), never
     change the settings yourself.
     - If sase-1oe.4 is IN_PROGRESS with no live agent or monitor, preview
       `sase bead work sase-1oe -n`. Relaunch it only when the preview shows sase-1oe.4
       relaunching and nothing else being killed.
     - If the cutover gate is raised and waiting on a human, record that in the nested
       epic's description, so the next watch keeps waiting on it.
   - If a red `release.yml` run at `v0.19.0` is the blocker, coordinate with
     sase-1n6.10, which owns E2's first-release proof. A burned candidate is expected
     under E2. Fix the cause, and leave the rebuild or resume to the steward and the
     train.
   - The nested epic's Landing section must repeat this whole Landing section verbatim,
     so every lander keeps creating nested epics until 0.19.0 is observed on PyPI.
   - Only the lander that actually observes 0.19.0 on PyPI creates the evidence
     artifact.
   - Turn the phases' `PROPOSED FOLLOW-UP` notes into task beads with `/sase_new_task`
     only when they are not release blockers.

## Risks

- **Core lag can return.** Any commit that moves `sase-core-revision.txt` to an
  unpublished sase-core commit brings core lag back. core-pin and rehearse both wait
  through the next daily cut instead of canceling, and the watch reports it.
- **The daily cut runs late.** GitHub has delayed sase-core's `41 7 * * *` cron by about
  seven hours. core-pin's 36h poll covers more than one cut.
- **Late wheel-leg failures.** The rehearsal may surface failures in wheel-leg steps CI
  has never proven: the N-1 install, the fakey launch, or the upgrade. rehearse stops
  with the evidence, and the lander fixes them.
- **The human cutover gate.** Until someone approves the E2 cutover gate, the steward
  never merges. The watch then ends at its deadline and the lander nests another watch.
  Approve the gate when sase-1oe.4 raises it.
- **Master churn.** Many epics land on master every hour, so new reds can appear after a
  phase closes. gate-reds' re-inventory, the rehearsal's qualification, the watch's
  `FULL_CI_RED` exit, and the lander absorb that.

## Non-goals

- Making ci_watch merge sase release PRs again. The steward handover is the design.
- E2's memory decisions, and closing or editing any E2 phase.
- Advisory visual-lane flakes (sase-1oa, sase-1nx, sase-1qb, sase-1qc) and other tracked
  flakes such as sase-1qd.
- The `just check-full` cost budgets (sase-j0).
