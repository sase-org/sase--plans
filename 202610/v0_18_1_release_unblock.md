---
tier: epic
title: Unblock both release gates, cut sase-core 0.42.5, and ship sase v0.18.1
goal: Master Gate and Full CI are green on master, a complete sase-core-rs release
  contains sase's source pin, ci_watch merges release PR 321, Publish uploads sase
  0.18.1 to PyPI, and the lander that observes it retains clean-install evidence as
  an artifact before landing.
parent_bead: sase-1nn.8.4
decisions:
  full_ci_rerun:
    ask: Should the release watcher rerun a failed Full CI run's failed jobs once
      before counting it red?
    default: true
    why: Most recent Full CI reds are one-off flakes; a rerun cannot turn a deterministic
      failure green
    answer: true
phases:
- id: core-cut
  title: Start the sase-core-rs release that contains sase's source pin
  description: 'core-cut: confirm no complete sase-core-rs release contains the current
    source pin, then dispatch the guarded sase-core release cut once and record the
    run, without waiting for it.'
  depends_on: []
  size: small
- id: gate-tests
  title: Repair the Master Gate and Full CI test-lane reds
  description: 'gate-tests: add the missing monitor --weight completion metadata and
    regenerate the CLI completion snapshot, verify the autonomy contract cluster is
    resolved, and fix every deterministic or recurring non-visual Full CI failure.'
  depends_on: []
  size: medium
- id: visual
  title: Repair the Full CI visual-test reds
  description: 'visual: find out why the stranded Beads PNG and any other drifting
    goldens differ on CI, make their renders deterministic, regenerate them after
    semantic review, and prove the full screenshot check is clean.'
  depends_on: []
  size: medium
- id: watch
  title: Watch core, release gates, ci_watch, Publish, and PyPI with a self-closing
    monitor
  description: 'watch: verify the three sibling phases landed, then start one bounded
    watcher monitor with no follow-up that waits for the core release, refreshes the
    release PR, reruns flaky gates once, and always closes this bead.'
  depends_on:
  - core-cut
  - gate-tests
  - visual
  size: small
proposed_by: bbugyi200.athena.13x
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 16:37:45
status: wip
bead_id: sase-1nn.8.4.3
---

- **PROMPT:** [prompts/202610/v0_18_1_release_unblock.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_release_unblock.md)
- **BEAD:** [sase-1nn.8.4.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/sase-1nn.8.4.3.md)

# Unblock both release gates, cut sase-core 0.42.5, and ship sase v0.18.1

**Target.** The target is sase **0.18.1**. The request ends by saying "v0.18.0", but
0.18.0 has been on PyPI since 2026-10-19 11:09 UTC. Release-please PR **#321**
(`chore(master): release 0.18.1`, branch `release-please--branches--master`) is the
release that is waiting. Every "observe the release" step in this epic, and in any
nested epic it spawns, means 0.18.1.

**Where this sits.** This is a nested child of **sase-1nn.8.4**. That epic's chain is
**sase-1nn.8.4** (plan `plan:202610/v0_18_1_final_goldens.md`) → **sase-1nn.8** (plan
`plan:202610/v0_18_1_release_landing.md`) → **sase-1nn** (plan
`plan:202610/v0_18_1_release.md`). sase-1nn.8.4's phases are both closed. Its watch
phase .2 closed `canceled` (FULL_CI_RED). Its lander then ran five observation turns
(notes #1 to #4), found new blockers, and ended without starting another driver, so
nothing is currently moving the release. This epic is that driver. Its lander closes the
ancestors once the release is real. Read sase-1nn.8.4's notes #2 to #4 before starting
any phase.

**How the release happens.** `ci_watch` is an AXE job on athena. Its config is in
chezmoi `home/dot_config/sase/sase_athena.yml`, and its code is in bugyi-chops. Every 5
minutes it merges a green release-please PR, but only when all of these hold:

1. **Master Gate** is green at the exact master HEAD.
2. The newest completed **Full CI** run on master is green and at most 6h old.
   `ci_watch` dispatches Full CI itself when the newest run is more than 2h old and none
   is in flight. Full CI also runs on a `17 */2 * * *` cron, in concurrency group
   `full-ci` (queued, never canceled in progress).
3. Every check on the release PR is green (SKIPPED counts as green). On #321, the only
   check that does real work is `CI / release-core-floor-smoke`.

`Publish` (`publish.yml`) runs on a `17 */3 * * *` cron or by dispatch. Before the
merge, its `release` job regenerates the release branch from master. Its
`sync-release-metadata` job then runs `tools/ratchet_core_pin`. That tool raises the
branch's `sase-core-rs==X.Y.Z` floor to the newest complete PyPI core release whose tag
contains `sase-core-revision.txt`. If no such release exists, it exits 3 with
`CORE_LAG`. After the merge, the next Publish run creates the tag and GitHub release,
and then runs `build`, `install-smoke`, `install-smoke-core-floor`, and `publish` to
PyPI.

sase-core cuts a release once a day at **07:41 UTC**: release-plz's `Merge release PR`
job merges the open release-plz PR. The resulting push tags the release, builds the
five-distribution wheel matrix, and publishes. The documented urgent path is
`gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
This chain already used that path twice: sase-1nn.2 cut 0.42.2, and sase-1nn.8's
core-cut cut 0.42.3. It is the same argv the steward's `dispatch_core_cut` uses.

## Evidence at planning time (2026-10-24, about 20:10 UTC)

Master HEAD is `c594443dc7` (feat(monitor): add zero-weight wait monitors). PyPI serves
sase 0.18.0 (0.18.1 returns 404) and sase-core-rs 0.42.4 (5 files, 13:08 UTC). Each of
the four blockers below can stop the merge by itself:

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Evidence                                                                                                                                                                                                                           | Phase           |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| 1   | **Core lag.** `sase-core-revision.txt` is `7ec12d2d7ad8e670abe26aa396259a512f330391`, the sase-core master HEAD. It sits after v0.42.4 together with `55d90d5b` (autonomy question policy) and `e80f08da`. No published core contains it. #321's floor smoke fails because 0.42.3 lacks `autonomy_effective_record` and `classify_stale_publication_request`, and Publish cannot ratchet the floor. sase-core release-plz PR **#336** (`chore: release v0.42.5`) is OPEN and MERGEABLE, its base is exactly `7ec12d2d`, and all its checks are green. | Publish 40118535862: `CORE_LAG(pin=7ec12d2…, newest_complete=0.42.4)`, exit 3, with build and publish skipped. #321 head `ce94fbdb` pins `sase-core-rs==0.42.3`. Its floor smoke (run 40118610787) reports 2/879 bindings missing. | core-cut, watch |
| 2   | **Master Gate red at HEAD.** `c594443dc7` added `sase monitor start -w/--weight` without completion metadata. `tests/completion/test_kind_coverage.py::test_every_value_slot_is_kinded_choiced_or_hinted` reports `monitor/start:weight` as uncaptioned. Two nodes in `tests/completion/test_snapshot.py` report spec drift. A local structural diff shows that the drift is exactly the new `weight` option plus the `start` node's `description_digest`.                                                                                            | Master Gate 40124909544 failed jobs 120964257103 (test 6) and 120964257138 (test 3). sase-1nn.8.4 note #4 routed this to the sase-10h epic, whose land agent is not running.                                                       | gate-tests      |
| 3   | **Full CI test lanes red.** The latest completed run, 40109539212 at `7b1b437558`, fails 3.12, 3.13, and 3.14 on the autonomy and question-gate contract cluster (sase-1o3, 23 to 24 nodes). It also has a FileNotFoundError in `tests/fakey/test_gate_capacity_custom_e2e.py`, and on 3.12 an image-modal resize timeout. `602ff33c1b` (built-in questions wait for humans) rewrote exactly those autonomy test files, and its Master Gate 40114028467 passed, so the cluster is probably resolved. That is not yet proven.                          | Full CI 40122422443 at `0d11a115db` (after 602ff33c1b) was still running at planning time. The last green Full CI was 39260047671 at `41b0a73b06` on 2026-10-19.                                                                   | gate-tests      |
| 4   | **Full CI visual-test red.** `artifacts_beads_stranded_120x40` reports 6793/1520532 changed pixels in 40108758095 and 40109539212 (both at `7b1b437558`, sase-1o4). It reported 1171 px at `02cd3b39b6` in 40101618059, where `agents_weighted_runner_capacity_expanded_80x32` also drifted 14077 px. `c793bff54b` introduced the golden. Between those SHAs, `79fa96dfc7` (Artifacts Files pane behind a beta flag) and `206bda93be` (bead-anchored stranded flags) changed the presentation.                                                        | The pixel count changes between SHAs, so a stale golden and a nondeterministic input are both possible.                                                                                                                            | visual          |

Earlier one-off test failures, each with an existing owner:

- `tests/ace/tui/test_agent_archive_column_app.py`: two different nodes timed out on
  3.14 in 40085700689 and 40088671003 (sase-1mj). This file recurs.
- `tests/tool/test_settlement_retention.py::test_handoff_end_to_end_publishes_one_notification`:
  `database is locked`.
- `tests/ace/tui/test_notification_plan_gate.py::test_epic_plan_modal_renders_no_raw_editor_for_capacity`:
  `#gate-singleton-0` missing.
- `tests/ace/tui/widgets/test_prompt_todo_highlight.py` (sase-xh).
- `agents_final_rows_60x40/80x40` PNG drift (sase-1nx).

Other open flake beads: sase-1ns, sase-1o2, sase-1nr, and sase-1ny.

Other live work to coexist with:

- `sase-1n6.5.1.5.2.1.3.1` (monitor "WAITING FOR CORE") waits for a containing core
  release. When one exists, it will commit a `sase-core-rs` pin bump to master. Both
  outcomes are harmless: our cut helps it, and its bump does not conflict.
- **sase-1n6.9** (cut releases over to release.yml and hand merges to the steward) is
  IN_PROGRESS and blocked on sase-1n6.6. Until it closes `done`, `ci_watch` owns sase
  release merges.
- An unrelated agent may fix blocker 2 first. Every phase re-inventories.

## Rules for every phase

- Read your own bead, sase-1nn.8.4 (notes #2 to #4), and the beads your section names,
  with `sase bead read <id> -r '<why>'`. Read `lint_and_test.md` and `sase_beads.md`
  through `/sase_memory_read`. For TUI or visual work, also read `tui.md` and every
  screenshot note it links. For `src/sase/ace/` changes, also read `ace.md`. If a
  symvision gate fails, also read `symvision.md`.
- **Fetch and re-inventory on the current master tip before editing.** The Evidence is a
  starting point, not a whitelist. If another commit already fixed something, verify it
  and record that commit instead of redoing the work.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state the assertion is about, using `sase.ace.testing.wait.wait_for` or the
  file's existing helper. Never use a fixed `pilot.pause(<seconds>)` as a layout wait.
  If the product itself settles late or inconsistently, fix the product. Never copy a CI
  candidate PNG into the tree. Regenerate a golden only after its render is
  deterministic.
- Shared domain behavior belongs in sase-core (`rust_core_backend_boundary`). None of
  these fixes is expected to need a Rust change. Touch sase-core only through
  `sase repo open sase-core`. If you must change it, run its own `sase tool run check`
  there, never edit release-plz-owned versions or changelogs, and move sase's
  `sase-core-revision.txt` only as `docs/rust_backend.md` describes.
- Verify with `just fix`, then `sase tool run check`, in every checkout you changed.
  **Do not run `just check-full`.** GitHub Full CI is this epic's exhaustive signal, and
  the user's "full check" means that workflow. Reproduce CI failures with focused pytest
  node runs (add `-n 8` when load matters). Use `/sase_monitor` for anything long, such
  as full screenshot runs or CI and PyPI waits. Never end a turn waiting. The CI `lint`
  job enforces `toobig`, so do not grow a file past its line limit.
- **No hand merges, no new PRs, no merge-authority changes.** #321 merges only through
  `ci_watch`. sase-core release PRs merge only through their own `Merge release PR` job.
  Never dispatch `release.yml`, never run `tools/release_watch --live`, and never edit
  `ci_watch` config. You may dispatch workflows (`gh workflow run`) and rerun failed
  jobs (`gh run rerun --failed`) only where a section says so.
- Phase workers create no beads. Record `PROPOSED FOLLOW-UP: <summary — detail>` notes
  on your own phase bead. Name every existing task bead your evidence resolves or
  corroborates, with run and job IDs, so the land agent can close it or add a +1. Before
  closing, run `sase bead epic-symbols <own-phase-id>`; it must list nothing. Then close
  with `sase bead close <id> --note '<evidence>'` and land through `/sase_final`. Host
  finalization makes the commit. Never commit, branch, or push by hand. If you cannot
  finish, still close your bead with `-R canceled` and the evidence, so the land agent
  always runs.

## Phase core-cut

Goal: a sase-core-rs cut that contains sase's current source pin is merged and
publishing, or the reason it cannot start is on record. This phase makes **no repo
edits** and does not wait for the publish.

1. Fetch sase. Read the source pin with `git show origin/master:sase-core-revision.txt`.
   Open sase-core with `sase repo open sase-core -r '<why>'` and fetch it.
2. **Already published?** Take the newest PyPI version:
   `curl -fsS https://pypi.org/pypi/sase-core-rs/json | jq -r .info.version`. Its
   release is complete when it has five unyanked files (manylinux_2_28 x86_64 and
   aarch64, universal2, win_amd64, sdist). Check that its tag contains the pin with
   `git -C <sase-core> merge-base --is-ancestor <pin> v<version>`. If both hold, close
   this phase with that evidence, run `/sase_final`, and stop.
3. **In flight or imminent?** List recent cut runs with
   `gh run list -R sase-org/sase-core --workflow release-plz.yml --limit 10 --json databaseId,event,status,conclusion,createdAt,displayTitle`.
   - If a `workflow_dispatch` or 07:41 `schedule` run is still in progress, a cut is
     already happening. Record its ID and do not dispatch.
   - If the next 07:41 UTC daily cut is less than 90 minutes away, do not dispatch.
     Record that.
4. **Check the release-plz PR.** Run
   `gh pr list -R sase-org/sase-core --state open --search 'in:title "chore: release v"'`,
   then
   `gh pr view <n> --json title,state,mergeable,baseRefOid,headRefOid,statusCheckRollup`.
   At planning time this was #336, `chore: release v0.42.5`. It must be open and
   mergeable with every check green (SKIPPED is fine), and it must contain the pin
   (`git merge-base --is-ancestor <pin> <headRefOid>`). If release-plz has since renamed
   the version, use the new one everywhere.
   - If the PR is missing or red, or does not contain the pin, do not dispatch. That
     includes the case where sase-core master itself is red. Record the evidence (failed
     job IDs and the first error line) as `PROPOSED FOLLOW-UP`. Then close
     `-R canceled -r "core cut not started: <reason>"` and run `/sase_final`. Do not
     repair sase-core in this phase. The land agent plans that repair.
5. **Dispatch once:**
   `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
   Then confirm that a new `workflow_dispatch` run appears in the step-3 listing, and
   record its ID. Do not dispatch twice, and do not hand-merge the PR. The
   `Merge release PR` job waits for checks (up to 40 minutes) and then merges. The wheel
   build and publish usually finish about an hour after the dispatch. The watch phase
   observes the result and can retry once on its own.
6. Append a note to this phase with the pin, the release-plz PR and its version, the
   dispatch decision and run ID (or why there was none), and the expected version. Run
   `sase bead epic-symbols <this-phase-id>`, close with that evidence, and run
   `/sase_final`. There is no commit.

## Phase gate-tests

Goal: Master Gate is green at the tip, and Full CI's non-visual jobs have no
deterministic or recurring failure. `visual-test` belongs to the visual phase.

1. **Re-inventory.** Fetch. Record the Master Gate result at `origin/master` HEAD and
   every failed job in the newest two completed Full CI runs. 40122422443 should be done
   by then. Use `gh run view <id> --log-failed -R sase-org/sase` or
   `gh api repos/sase-org/sase/actions/jobs/<job>/logs`, and strip ANSI. Read sase-1o3,
   sase-10h note #4 or later (the c594443dc7 routing), and the owner beads named in the
   Evidence.
2. **Monitor `--weight` completion (blocker 2).** Unless master already fixed it:
   - Give the `weight` value slot of `sase monitor start` completion metadata. Use one
     of the three ways the coverage test names. The intended fix is a free-form `"int"`
     value hint keyed by dest `weight` in `src/sase/completion/kinds.py`
     (`_VALUE_HINT_TABLE`). No other CLI action uses that dest today. If one does by the
     time you look, use `HINT_PATH_OVERRIDES[(("monitor", "start"), "weight")]` instead.
     Do not add `choices=`: that would replace the handler's own "only 0 is accepted"
     validation and break `tests/main/test_monitor_handler_start_weight.py`.
   - Regenerate the snapshot with `just sync-completion-spec`. Then confirm that the
     `tests/completion/snapshots/cli_spec.json` diff touches only the `monitor start`
     node: its description digest plus the new `weight` option with its hint.
   - Run `tests/completion/` and `tests/main/test_parser_monitor.py`. If the repo's
     shell-completion deploy or generated docs check (`just docs-gen-check` inside
     `just check`) flags the new hint, follow its regeneration instruction.
3. **Autonomy cluster (blocker 3).** Prove whether `602ff33c1b` resolved sase-1o3.
   - Run every node sase-1o3 lists, plus `tests/autonomy_contract/`,
     `tests/test_autonomy_record.py`, `tests/test_user_question_gates.py`,
     `tests/test_notification_gates.py`, and
     `tests/fakey/test_gate_capacity_custom_e2e.py`. Run them against a core binding
     that contains the source pin, as CI does (CI builds it from
     `sase-core-revision.txt`). The published 0.42.4 lacks `55d90d5b` and `7ec12d2d`, so
     a run against it proves nothing. Use the local core build path in
     `docs/rust_backend.md`.
   - Cross-check the post-602ff33c1b Full CI test lanes.
   - If everything passes, record "sase-1o3 resolved by 602ff33c1b" with the evidence.
     If anything fails on current code with the current core, fix it. The intended
     behavior is that built-in profiles make every question wait for a human (sase-core
     `55d90d5b`, sase `602ff33c1b`), and an explicit question policy still
     auto-resolves.
4. **Every other non-visual red.** This covers `test (3.12|3.13|3.14)`, `lint`,
   `docs-build`, `perf-floors`, `tutorial-tests`, `ace-page-group-isolation`,
   `build-core`, and every Master Gate shard. Reproduce each failing node, focused first
   and then under `-n 8`.
   - **Deterministic:** fix the root cause.
   - **Recurring:** the same test file fails in two or more of the latest five completed
     Full CI or Master Gate runs, at different SHAs or attempts. This is a release
     blocker even though it is intermittent: with one rerun allowed, a recurring flake
     is the likeliest thing left to keep Full CI red. Fix its readiness wait or product
     race under the rules above, and name its owner bead. At planning time,
     `tests/ace/tui/test_agent_archive_column_app.py` (sase-1mj) qualifies; recheck it.
   - **One-off:** record a `PROPOSED FOLLOW-UP` naming the existing owner bead and the
     run and job IDs. Do not fix it here.
5. Run `just fix` and `sase tool run check`. Record every resolved or corroborated bead
   ID. Close with the exact nodes run and their outcomes, then run `/sase_final`. Do not
   wait for this turn's commit to reach CI; the watch phase does that.

## Phase visual

Goal: the Full CI `visual-test` job is clean at the tip.

1. **Re-inventory.** Fetch. In the newest two completed Full CI runs, read the
   `visual-test` job log for every `updated`, `created`, or `stale` image and its
   changed-pixel count. Download the run's visual report artifact if one exists, but use
   it only to inspect; never commit its PNGs. Read sase-1o4 and sase-1nx.
2. **`artifacts_beads_stranded_120x40` (sase-1o4).** Read
   `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads_stranded.py` and its
   fixture. Capture check-only locally, in the host zone and in `TZ=UTC`:
   `TZ=UTC just fix-tui-screenshots --check -- <node>`. Then decide which case applies:
   - **Stale golden.** It changed deterministically after `79fa96dfc7` or `206bda93be`.
     For example, the Artifacts sub-tab strip lost `files` behind the
     `artifacts_files_pane` beta flag, or the stranded presentation changed. Regenerate
     with `TZ=UTC just fix-tui-screenshots -- <node>`. Inspect the old and new images
     side by side and confirm that the intended stranded-work content is visible, not a
     loading frame or a different scroll position.
   - **Nondeterministic render.** The 1171 vs 6793 px history makes this plausible. Find
     the varying input, such as real wall-clock relative ages (`⧖ 4d`-style text),
     timezone, ambient flags or env, or an unsettled first frame. Pin it in the fixture,
     or add a semantic settle wait, before regenerating. Capture twice to prove it is
     stable.
3. **Other drifting images.** For `agents_weighted_runner_capacity_expanded_80x32`, the
   sase-1nx final-row images, and anything else the re-inventory found, regenerate only
   if the image recurs in the latest runs or reproduces locally. Use the same procedure.
   Leave unrelated good goldens alone.
4. **Full proof.** Run `TZ=UTC just fix-tui-screenshots --check` through `/sase_monitor`
   (it takes a long time). Give it a `--next` follow-up that inspects the result. The
   result must be no drift and no unstable recapture. Report infrastructure-only or
   non-reproducing failures with evidence.
5. Run `just fix` and `sase tool run check`. Record resolved beads (sase-1o4, and
   sase-1nx if it applies). Close with the evidence, then run `/sase_final`.

## Phase watch

This phase makes **no repo edits**. It ends by starting one monitor with **no
`--next`**, so no LLM turn runs after it and nothing can hang. The watcher script closes
this bead itself, and the monitored wrapper always exits 0. Every inline failure path
below closes the bead `canceled` before `/sase_final`. **This phase always closes its
bead, so the land agent always runs.**

1. **Idempotent start.** `sase bead read` this bead. If it is already closed (a relaunch
   after recovery), run `/sase_final` and stop.
2. **Already released?** If `curl -fsS https://pypi.org/pypi/sase/0.18.1/json` succeeds,
   close this bead with that evidence and run `/sase_final`.
3. **Prerequisites.** Fetch. Read the core-cut, gate-tests, and visual sibling beads.
   - A sibling closed `canceled` means a prerequisite failed. Close this bead
     `-R canceled -r "release watch not started: <phase> canceled"` with its close note,
     then run `/sase_final`.
   - Confirm each `done` sibling's commit is on `origin/master`
     (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note
     says no commit was needed (core-cut always) is fine. If a commit is still landing,
     use one short `/sase_monitor` wait with a re-fetch follow-up before calling it
     missing. Do not sleep in the provider turn.
   - Read the core-cut note. If its dispatched `release-plz` run (or a later cut)
     concluded `failure` and no complete containing core exists, close
     `-R canceled -r "release watch not started: core cut failed"` with the run and job
     evidence.
   - Confirm `gh pr view 321 -R sase-org/sase --json title,state` is still the open
     `chore(master): release 0.18.1`. If release-please retargeted the version or
     replaced the PR, close
     `-R canceled -r "release watch not started: release target changed"`.
   - Read **sase-1n6.9**. If it is closed `done`, `ci_watch` and `publish.yml` may no
     longer merge and publish #321. Close
     `-R canceled -r "release watch not started: release path cut over"` with the
     evidence.
4. **Epic-symbol guard.** `sase bead epic-symbols <this-phase-id>` must list nothing.
   Otherwise close `-R canceled -r "release watch not started: epic-symbol entries"` and
   run `/sase_final`.
5. **Write the watcher outside the repo.** Use the literal `/tmp`, not `$TMPDIR`,
   because agent tmp is reaped: `W=/tmp/sase-release-watch-<this-phase-id>` and
   `mkdir -p "$W"`. Write `$W/watch.sh` with a quoted heredoc (`<<'EOF'`), so nothing
   expands at write time:

   > [!decision] full_ci_rerun
   >
   > Keep `FULL_RERUN=1`. A failed attempt-1 Full CI run gets one
   > `gh run rerun --failed`. Only its final attempt counts toward FULL_CI_RED.

   > [!decision] full_ci_rerun = no
   >
   > Set `FULL_RERUN=0`. Every failed Full CI run counts immediately, and the watcher
   > never reruns Full CI. Master Gate reruns are unaffected; they follow sase-1nn's
   > final `gate_rerun=yes`.

   ```sh
   #!/bin/sh
   # sase v0.18.1 release watcher: sh watch.sh BEAD LOG [preflight]
   # Never edits files. In watch mode it closes BEAD itself and always exits 0.
   BEAD=$1 LOG=$2 MODE=${3:-watch}
   REPO=sase-org/sase CORE=sase-org/sase-core VERSION=0.18.1
   TITLE="chore(master): release $VERSION"
   FULL_RERUN=1   # decision full_ci_rerun: 1 = yes, 0 = no
   unset GH_FORCE_TTY
   export NO_COLOR=1 GH_COLOR=never CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0 GH_PROMPT_DISABLED=1
   START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   END=$(( $(date +%s) + 43200 ))
   outcome=DEADLINE detail= published= core_cut= lag_since= ratchets=0 last_ratchet=0
   gate_reruns= full_reruns= gate_red_since=
   say() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" | tee -a "$LOG"; }
   act() { [ "$MODE" = watch ] || return 1; "$@" >>"$LOG" 2>&1; }
   inflight() { gh run list -R "$1" --workflow "$2" --limit 10 --json status \
     --jq '[.[] | select(.status != "completed")] | length' 2>>"$LOG"; }
   while :; do
     now=$(date +%s)
     if curl -fsS -o /dev/null "https://pypi.org/pypi/sase/$VERSION/json" 2>/dev/null; then
       outcome=RELEASED; break
     fi
     merged=$(gh pr list -R "$REPO" --base master --state merged \
       --search "in:title \"release $VERSION\"" --json title,mergedAt \
       --jq "[.[] | select(.title == \"$TITLE\")][0].mergedAt // \"\"" 2>>"$LOG")
     if [ -n "$merged" ]; then
       pub=$(gh run list -R "$REPO" --workflow publish.yml --limit 10 \
         --json databaseId,createdAt,status,conclusion \
         --jq "[.[] | select(.createdAt > \"$merged\")] | sort_by(.createdAt) | last
               | if . == null then \"none\" else \"\(.databaseId):\(.status):\(.conclusion)\" end" 2>>"$LOG")
       if [ "$pub" = none ] && [ -z "$published" ]; then
         act gh workflow run publish.yml -R "$REPO" --ref master -f publish_existing=false && published=yes
       fi
       case "$pub" in
         *:completed:failure|*:completed:timed_out|*:completed:startup_failure)
           outcome=PUBLISH_RED detail="Publish run $pub"; break ;;
       esac
       say "state merged=$merged publish=${pub:-?} dispatched=${published:-no}"
     else
       head=$(gh api "repos/$REPO/commits/master" --jq .sha 2>>"$LOG")
       pin=$(gh api -H "Accept: application/vnd.github.raw" \
         "repos/$REPO/contents/sase-core-revision.txt?ref=$head" 2>>"$LOG" | tr -d '[:space:]')
       cv=$(curl -fsS https://pypi.org/pypi/sase-core-rs/json 2>>"$LOG" | jq -r '.info.version // empty')
       cn=$(curl -fsS "https://pypi.org/pypi/sase-core-rs/$cv/json" 2>>"$LOG" \
         | jq '[.urls[] | select(.yanked | not)] | length')
       cs=$(gh api "repos/$CORE/compare/$pin...v$cv" --jq .status 2>>"$LOG")
       core=lag
       if [ "${cn:-0}" -ge 5 ] && { [ "$cs" = ahead ] || [ "$cs" = identical ]; }; then core=ok; fi
       floor=$(gh api -H "Accept: application/vnd.github.raw" \
         "repos/$REPO/contents/pyproject.toml?ref=release-please--branches--master" 2>>"$LOG" \
         | sed -n 's/.*sase-core-rs==\([0-9.]*\).*/\1/p' | head -n 1)
       if [ "$core" = lag ]; then
         [ -n "$lag_since" ] || lag_since=$now
         if [ -z "$core_cut" ] && [ $((now - lag_since)) -ge 1800 ] \
           && [ "$(inflight "$CORE" release-plz.yml)" = 0 ]; then
           act gh workflow run release-plz.yml -R "$CORE" --ref master -f dry_run=false && core_cut=yes
         fi
         if [ $((now - lag_since)) -ge 14400 ]; then
           outcome=CORE_LAG detail="pin $pin not in a complete sase-core-rs $cv (files=$cn compare=$cs)"; break
         fi
       else
         lag_since=
         if [ "$floor" != "$cv" ] && [ "$ratchets" -lt 2 ] && [ $((now - last_ratchet)) -ge 3600 ] \
           && [ "$(inflight "$REPO" publish.yml)" = 0 ]; then
           act gh workflow run publish.yml -R "$REPO" --ref master -f publish_existing=false \
             && ratchets=$((ratchets + 1)) last_ratchet=$now
         fi
       fi
       gate=$(gh run list -R "$REPO" --workflow master-gate.yml --commit "$head" --limit 1 \
         --json databaseId,status,conclusion,attempt \
         --jq '.[0] | "\(.databaseId):\(.status):\(.conclusion):\(.attempt)"' 2>>"$LOG")
       case "$gate" in
         *:completed:success:*) gate_red_since= ;;
         *:completed:failure:1)
           id=${gate%%:*}
           case " $gate_reruns " in *" $id "*) ;; *)
             gate_reruns="$gate_reruns $id"; act gh run rerun "$id" --failed -R "$REPO" ;;
           esac ;;
         *:completed:failure:*)
           [ -n "$gate_red_since" ] || gate_red_since=$now
           if [ $((now - gate_red_since)) -ge 10800 ]; then
             outcome=GATE_RED detail="Master Gate $gate at $head"; break
           fi ;;
       esac
       runs=$(gh run list -R "$REPO" --workflow full.yml --branch master --limit 15 \
         --json databaseId,status,conclusion,createdAt,attempt,headSha \
         --jq "[.[] | select(.createdAt > \"$START\")] | sort_by(.createdAt) | reverse
               | map(\"\(.databaseId):\(.status):\(.conclusion):\(.attempt):\(.headSha[0:10])\") | join(\" \")" 2>>"$LOG")
       final=
       for r in $runs; do
         old_ifs=$IFS; IFS=:; set -- $r; IFS=$old_ifs
         [ "$2" = completed ] || continue
         case "$3" in
           success) final="$final g" ;;
           failure|timed_out)
             if [ "$FULL_RERUN" = 1 ] && [ "$4" = 1 ]; then
               case " $full_reruns " in
                 *" $1 "*) final="$final r" ;;
                 *) full_reruns="$full_reruns $1"; act gh run rerun "$1" --failed -R "$REPO" ;;
               esac
             else
               final="$final r"
             fi ;;
         esac
       done
       case "$final" in " r r"*) outcome=FULL_CI_RED detail="Full CI since $START: $runs"; break ;; esac
       say "state head=${head:-?} pin=${pin:-?} core=$core(v=${cv:-?} files=${cn:-?} cmp=${cs:-?}) floor=${floor:-?} gate=${gate:-?} full=[${runs}] final=[${final}] cut=${core_cut:-no} ratchets=$ratchets"
     fi
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

   What the watcher does on every 5-minute poll:
   - **Core.** It checks whether the newest PyPI sase-core-rs has all five files and a
     tag that contains master's current source pin. If the core lags for 30 minutes with
     no release-plz run in flight, it dispatches the guarded core cut **once**. This
     covers the case where another agent moves the pin past the cut that core-cut
     started.
   - **Release PR floor.** Once the core is current, if the release branch's floor
     differs from the newest core, it dispatches `publish.yml`
     (`publish_existing=false`). That rebuilds #321 from master and ratchets its floor.
     It does this at most twice, an hour apart, never while a Publish run is in flight.
     sase-1nn's final `publish_dispatch=yes` authorizes these dispatches.
   - **Master Gate.** A failed attempt-1 Master Gate run at HEAD gets one `--failed`
     rerun.
   - **Full CI.** A failed attempt-1 Full CI run gets one rerun, per decision
     `full_ci_rerun`. Cancelled runs (concurrency) are ignored.
   - **After the merge.** It dispatches one Publish run if none has started since the
     merge.

   The script has six outcomes. Every outcome except `RELEASED` closes the bead
   `canceled`, with the last 60 log lines as the note:

   | Outcome       | Condition                                                                       | Bead     |
   | ------------- | ------------------------------------------------------------------------------- | -------- |
   | `RELEASED`    | PyPI serves sase 0.18.1                                                         | done     |
   | `PUBLISH_RED` | a post-merge Publish run failed, timed out, or failed to start                  | canceled |
   | `FULL_CI_RED` | the two newest final Full CI results since start are both red                   | canceled |
   | `GATE_RED`    | Master Gate at HEAD has stayed red after its rerun for 3h, across new HEADs too | canceled |
   | `CORE_LAG`    | no complete core containing the source pin for 4h, after its one cut dispatch   | canceled |
   | `DEADLINE`    | 12h passed                                                                      | canceled |

6. **Preflight inline.** Run all three of these:
   - `sh -n "$W/watch.sh"`.
   - `timeout -k 5m 750m sh -n "$W/watch.sh"`, which proves the exact `timeout`
     durations parse. sase-1nn.7 died on `10h30m`, which GNU `timeout` rejects.
   - `sh "$W/watch.sh" <this-phase-id> "$W/watch.log" preflight`.

   Each must exit 0. The last must print one fully populated `state` line, or
   `outcome=RELEASED`, or a `merged=` line. Preflight never dispatches, reruns, or
   closes. This exact script printed a correct line at planning time
   (`core=lag(v=0.42.4 files=5 cmp=behind) floor=0.42.3 gate=…:completed:failure:1`). If
   a field is `?` or a command fails, fix only the script's gh or jq invocation, without
   touching the repo, and retry once. If it still fails, close
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` with the output,
   and run `/sase_final`.

7. **Record, then hand off.** Append a phase note with:
   - the master HEAD;
   - the source pin and the newest core;
   - the preflight line;
   - the decision value;
   - the exact monitor command.

   Then, from the workspace root, run this in the foreground until it exits:

   ```sh
   sase monitor start \
     -C "$PWD" -w 0 \
     -r 'Watch sase v0.18.1 through core, ci_watch, Publish, and PyPI; the watcher closes <this-phase-id>' \
     -t 13h \
     -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -L 'sase 0.18.1 watch' \
     -- 'timeout -k 5m 750m sh /tmp/sase-release-watch-<this-phase-id>/watch.sh <this-phase-id> /tmp/sase-release-watch-<this-phase-id>/watch.log; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 60; sase bead close <this-phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   - Pass the command as one quoted string with no `bash -c` wrapper.
   - `-w 0` makes this pure wait hold no runner capacity. If the installed `sase`
     rejects `-w`, rerun the same command without it.
   - Do **not** pass `--next`, `-m`, `-p`, or `--idle-timeout`.
   - A successful start ends this turn, and that handoff is exempt from `/sase_final`.
   - If `sase monitor start` exits nonzero twice, close
     `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` with both errors,
     then run `/sase_final`.

### Why the land agent always runs

Each layer ends inside the next one out:

1. The watcher stops at 12h, plus at most one poll.
2. The outer `timeout` stops at 750 minutes (12.5h), plus 5 minutes of kill grace.
3. The fallback close runs after a 60s pause.
4. The monitor's own 13h timeout never fires on a live host.

A fallback close after a successful close is refused as a conflicting re-close, and
`|| true` absorbs that.

| Path                                                   | Who closes the bead                               | What settles the session             |
| ------------------------------------------------------ | ------------------------------------------------- | ------------------------------------ |
| any of the six watcher outcomes                        | the watcher (3 tries)                             | wrapper exits 0, monitor `completed` |
| watcher hang, or outer `timeout` fires                 | the wrapper's fallback close                      | wrapper exits 0                      |
| watcher crash, or its close fails                      | the wrapper's fallback close                      | wrapper exits 0                      |
| prerequisite, epic-symbol, preflight, or start failure | this agent, inline                                | this agent's `/sase_final`           |
| a human stops the monitor                              | nobody automatically; the land agent can close it | monitor `stopped` counts as success  |
| reboot (`lost`) or an external SIGKILL                 | nobody                                            | a human: `sase bead work <epic-id>`  |

Step 1 makes recovery safe. `sase bead work <epic-id>` relaunches only the land agent if
this bead is closed, and relaunches the watch if it is open.

## Landing, ancestor handoff, and nested epics

This epic's `parent_bead` is **sase-1nn.8.4**. The lander owns every closure decision;
no phase closes an epic.

1. **Audit.** Re-read every child and note, the watcher log
   (`/tmp/sase-release-watch-<watch-id>/watch.log` and the watch close note), and every
   implementation commit and its source. Review post-start master drift. Resolve
   epic-caused issues. Triage every `PROPOSED FOLLOW-UP` with `/sase_new_task`:
   - corroborate existing owners with `sase bead +1` or a note;
   - close task beads the phases proved resolved (expected: sase-1o3, sase-1o4, and
     sase-1mj if fixed), with the evidence;
   - record the outcome of every item.

   A canceled watch is not proof of publication.

2. **Confirm the release before landing.** Observe actual **sase 0.18.1 on PyPI** and
   record:
   - wheel and sdist filenames, SHA-256 hashes, and upload times;
   - GitHub release v0.18.1 and its tag commit;
   - merged #321's mergedBy, mergedAt, and merge commit;
   - the qualifying green Full CI run ID, SHA, and attempt, and the green Master Gate at
     the merged head;
   - the Publish run ID and every job conclusion;
   - the sase-core-rs floor in the released metadata, and its PyPI file set.

   Retain the watcher and `ci_watch` evidence available for the merge. Do not infer
   `ci_watch` authorship from the merge account alone.

3. **Clean install.** Prove one outside the repository, with an isolated home directory
   given only to the child process. Never repurpose the shell's `HOME`. Create a Python
   3.12 venv and install `sase==0.18.1` from PyPI with `UV_NO_CACHE=1`. Verify that:
   - the installed sase is 0.18.1;
   - its sase-core-rs matches the exact dependency;
   - `sase core health --json` succeeds.

   Do not replace a global install.

4. **Evidence artifact.** Write `sase_v0_18_1_release_evidence.md` in a temporary
   directory with steps 2 and 3 and the audit summary. Register it with
   `sase artifact create -p <file> -l "sase v0.18.1 PyPI release evidence" -b <this-epic-id>`.
5. **Close this epic.** Run `sase bead epic-symbols <this-epic-id>`. Resolve each entry,
   or re-key it only to a still-open later bead that needs it. Close with
   `sase bead close <this-epic-id> --note "<verification and artifact ref>"`, run
   `just symvision`, open the plans repo with `/sase_repo`, and set `status: done` in
   this epic's linked plan. Never use `--force` on a successful landing.
6. **Resume the ancestors in order,** each only after the previous one closed:
   - **sase-1nn.8.4** with `plan:202610/v0_18_1_final_goldens.md`. If readiness requires
     its canceled watch **sase-1nn.8.4.2**, reopen it and close it normally with the new
     release evidence.
   - **sase-1nn.8** with `plan:202610/v0_18_1_release_landing.md`, and its canceled
     watch **sase-1nn.8.3** the same way.
   - **sase-1nn** with `plan:202610/v0_18_1_release.md`, and its canceled watch
     **sase-1nn.7** the same way.

   For each ancestor:
   - read its landing notes, every descendant and note, its plan, and post-child drift,
     and rerun descendant and linked-plan readiness;
   - add the artifact ref in a note;
   - retire its `sase bead epic-symbols` entries;
   - close it with a full recheck note, run `just symvision`, and set its plan to
     `status: done` in the plans checkout.

   Preserve historical cancellation notes. Stop at, and record, the first incomplete or
   ambiguous ancestor, and never force-close. sase-1nn has no parent; inspect its actual
   parent link before stopping.

7. **If 0.18.1 is not on PyPI, keep going through a nested epic.**
   - Leave this epic and every ancestor open, and record a LANDING AUDIT note with the
     watcher outcome and the concrete remaining blockers.
   - Author a nested child epic with `/sase_plan`, and run its
     validate/revalidate/propose loop. Set `parent_bead: <this-epic-id>`. Give it one
     repair phase per independent blocker you diagnosed, if any, plus a final
     self-closing watch phase. Even when no code change is needed, still propose a
     nested epic with just that watch phase. Never end the landing with nothing watching
     the release.
   - Carry forward this plan's rules, its watcher script (fix only diagnosed defects,
     and keep every close guarantee in the table above), and this whole landing section.
     That includes this step's recursion, step 2's confirmation, step 4's artifact, and
     the ancestor chain with this epic inserted as the first ancestor to resume.
   - Re-ask no decision of this epic; `full_ci_rerun` stays as answered. Reuse
     sase-1nn's final `publish_dispatch=yes` and `gate_rerun=yes`.
   - Every nested lander repeats this until one lander observes 0.18.1 on PyPI. That
     final lander creates the evidence artifact and closes the whole chain.
   - Do not close any release epic `done` merely because its descendants are closed.
