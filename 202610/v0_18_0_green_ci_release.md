---
tier: epic
title: Green Master Gate and Full CI, then ship sase v0.18.0 through ci_watch
goal: 'Master Gate is green at master HEAD and a fresh Full CI run passes every job,
  so ci_watch merges release-please PR #299 and sase 0.18.0 installs from PyPI. The
  release-wait phase closes its bead on every path that does not need a human. The
  land agent confirms the release before it lands, and repairs the release itself
  when something unexpected blocks it.

  '
decisions:
  visual_flakes:
    ask: How should the Full CI visual lane handle captures that drift once and then
      match?
    choices:
      recapture: Re-capture failed nodes serially; only exact matches pass; one-off
        drift is reported
      strict: Keep one strict pass; root-cause each nondeterministic node before release
    default: recapture
    why: Five capture-race repair epics did not converge; recapture still requires
      exact pixels
    answer: recapture
  dispatch:
    ask: Let the release watcher dispatch Full CI, Publish, and one Master Gate rerun
      when due?
    default: true
    why: Publish only runs every 3h on its own, so dispatch cuts hours off the wait
    answer: true
  stale_epics:
    ask: Once v0.18.0 is confirmed, what should the lander do with the stalled sase-1kx
      tree and sase-1jz?
    choices:
      supersede: Close both as superseded, with this epic's release evidence attached
      notes: Append release-evidence notes only; you close the stale epics yourself
    default: supersede
    why: A later bead-work retry would relaunch a zombie lander that re-chases 0.18.0
    answer: supersede
  memory_visual_check:
    ask: Update lint_and_test.md's visual --check sentence to describe serial re-capture
      (recapture only)?
    memory:
    - lint_and_test.md
    corrects: '`--check` stays strict: it never writes and exits 1 on required drift.'
    default: true
    answer: true
phases:
- id: gate-fix
  title: Fix every Master Gate red at master HEAD
  depends_on: []
  size: medium
  description: 'gate-fix: clear the mypy, docs-test, and docs-lint reds the docs restructuring
    left on master, then prove every lint-job step locally.'
- id: visual-drift
  title: Fix the deterministic visual-test failures
  depends_on: []
  size: medium
  description: 'visual-drift: repair the two goldens that fail on every run and the
    CI-only agents-deck readiness timeouts.'
- id: visual-stability
  title: Stop one-off capture drift from failing the visual lane
  depends_on: []
  size: medium
  description: 'visual-stability: apply the accepted visual_flakes policy so a capture
    race no longer reds Full CI by itself.'
- id: ci-budget
  title: Fix the perf-floor flake and the Full CI time budget
  depends_on: []
  size: small
  description: 'ci-budget: recalibrate the Phase 7E perf ceiling that fails about
    30% of runs, and check every Full CI job''s headroom against its timeout.'
- id: watcher
  title: Harden tools/release_watch so its bead close is guaranteed
  depends_on: []
  size: medium
  description: 'watcher: make every exit path close the bead even under a process-group
    kill, and end the watch early on a stuck red Master Gate.'
- id: release-watch
  title: Watch v0.18.0 to PyPI without editing files
  depends_on:
  - gate-fix
  - visual-drift
  - visual-stability
  - ci-budget
  - watcher
  size: small
  description: 'release-watch: make no edits, preflight the hardened watcher, monitor
    it until a verdict, and always close with an honest verdict.'
proposed_by: bbugyi200.athena.10q
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-16 13:00:17
status: wip
bead_id: sase-1lh
---

- **PROMPT:** [prompts/202610/v0_18_0_green_ci_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_green_ci_release.md)
- **BEAD:** [sase-1lh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1lh/README.md)

# Plan: Green Master Gate and Full CI, then ship sase v0.18.0 through ci_watch

## Why this epic exists

The newest sase on PyPI is 0.17.1, from 2026-08-29. Release-please PR #299
(`chore(master): release 0.18.0`) has been open since 2026-08-30. Five nested repair
epics under **sase-1kx** each ended with the release not shipped. Their release-wait
phases always closed their beads; what never happened was all three of ci_watch's gates
being green together. The last attempt stalled for a different reason: its monitor and
its land agent were stopped by hand, so its land agent never ran.

"Full check" in the request means the **Full CI** heavy lane. ci_watch gates on Full CI
together with Master Gate. `just check-full` is a local recipe that CI never runs, and
no phase runs it.

### What ci_watch needs, all in the same 5-minute tick

The logic lives in `bugyi_chops/ci_watch.py`, with config in the chezmoi
`sase_athena.yml`:

1. **Master Gate** is green at the exact master HEAD SHA. It is re-checked just before
   the merge.
2. The **newest completed Full CI** run among recent master runs is green and finished
   within 6 h. It can be on any master SHA. A newer red run blocks an older green one,
   and in-flight runs are ignored.
3. **PR #299** is the only release-please PR. It is `MERGEABLE` and `CLEAN`, and every
   check is SUCCESS, SKIPPED, or NEUTRAL. No release-please or Publish run is in flight.

ci_watch then runs `gh pr merge 299 --merge --match-head-commit <oid>`. Publish is not
triggered by a push. It runs on cron (`17 */3`, in practice every 4–7 h) or on dispatch.
Its job chain is release, build, install smokes, then publish. One run takes 2–5
minutes.

### Snapshot of the reds (2026-10-16, ~16:40Z)

Treat this as a starting point, not a contract. Every phase re-derives its own failures
from current CI before editing anything.

- **Master Gate**: red at HEAD since `ad3aa463e4`. The last green run was 38820988371 at
  `741afba3b5`. Every current red comes from the in-flight docs restructuring epic
  **sase-1le** (its child **sase-1le.20.1** is still landing phases):
  - **mypy** in `tools/render_keymap_macro_reference:303-306`: `relative` is reused as a
    `Path` and then as a tuple.
  - **mypy** in `tools/docs_split:488`, a real bug. In `_section_chunks`, `target` is
    read before `target = request.target` in the same loop iteration. The pointer
    therefore uses the previous request's target, or raises `UnboundLocalError` on the
    first one.
  - **mypy** in `tools/docs_split:674/685/794`: variable names are reused with different
    types.
  - **docs tests**:
    - `tests/test_docs_moves.py::test_docs_split_supports_h4_nested_intro_drop_and_finish`
      fails because a relative pointer is rewritten from the wrong origin, producing
      `../../source.md#deep-heading`.
    - `tests/main/test_init_memory_markdown_templates.py::test_plus_one_close_boundary_guidance_stays_current`
      reads `beads.md`, but the phrases now live in the `bead-cli-records.md` and
      `bead-triage.md` pages under `docs/reference/work-units/`.
    - `tests/test_plugins_required_gate_preview.py::test_checked_in_plugins_required_docs_use_batch_wording`
      reads the old page; the wording moved to
      `docs/reference/config/scheduler-behavior.md`.
    - `tests/test_sdd_canonical_layout.py::test_active_sources_and_docs_use_only_canonical_plan_paths`
      exempts `perf_runbook.md` by its old name; the page is now
      `docs/contribute/perf-runbook.md`.
    - `tests/test_macro_terminology.py::test_macro_docs_and_memory_avoid_xprompt_terms`
      trips on migration anchors and history in `docs/_meta/moves.yml`,
      `docs/_meta/splits/macros.yml`, `docs/macros.md`, and
      `docs/reference/config/feature-flags.md`.
  - **Masked by the mypy failure**: the `docs-lint` hub budget fails
    (`docs/reference/index.md` has 412 words against a limit of 400). The later lint-job
    steps (`just validate`, `just validate-committed-plans`, `just build-check`) have
    not run on master for about 15 hours.
- **Full CI**: it has not been green since run 33261893215 on 2026-08-29.
  - The `test` legs (3.12, 3.13, and 3.14) currently pass.
  - **`lint`** shows the same mypy errors as Master Gate.
  - **`visual-test`** failed in all 80 runs since 2026-10-02. There are three causes:
    1. CI-only readiness timeouts in
       `test_ace_png_snapshots_agents_agent_sessions.py::test_renamed_generic_agent_session_root_png_snapshot`
       and
       `test_ace_png_snapshots_agents_neighbors.py::test_agents_session_lane_neighbors_png_snapshot`.
       Both wait for "Main view visibly applies the selected reply card", a stricter
       wait added to `_ace_agents_png_snapshot_helpers.py` by `be73b6021a`.
    2. Two goldens that drift on every run. `help_guide_axe_120x40` drifts because
       `8676f92f2f` changed help text in `src/sase/ace/tui/widgets/axe_onboarding.py`,
       so the scrollbar thumb changed size. `agents_lane_neighbors_above_context_160x50`
       lost a wave divider `┊`, likely in `d5efed6535`.
    3. Drift sets that change from run to run at the same commit.
  - **`perf-floors`** failed in 23 of 80 runs. The anchor
    `evaluate_query_many.synthetic_1000_specs.persistent_query_keystroke` measures
    193.9–194.9 µs against a 193.44 µs ceiling.
  - **Timeout risk**: the 3.12 coverage leg runs 90–101 minutes against a 120-minute
    timeout.
- **PR #299**: head `1f7754f0d9` fails `release-core-floor-smoke`. The core floor on the
  release branch is 0.40.0, and that version lacks the `project_clan_waves` binding.
  sase-core-rs **0.40.1** (uploaded 2026-10-16 13:46Z) exposes all 864 required
  bindings. The next Publish `sync-release-metadata` run ratchets the floor; the watcher
  dispatches that run when it is due.
- **Not ci_watch gates, and out of scope for code changes**:
  - **Deploy Docs**: the docs-contract key `covers:` clashes with mkdocs-exporter. This
    belongs to the docs epic.
  - **Install smoke**: `sase>=0.18` does not exist yet. It heals once 0.18.0 publishes.
  - **CI Telemetry**: never green; its leak gate fails.

### Concurrent work this epic must respect

- **sase-1le / sase-1le.20.1** (docs restructuring) is actively pushing to master. Fix
  its reds forward and never revert its page moves. Record what you fixed as a
  `DISCOVERED ISSUE:` note on **sase-1le.20.1** so its land agent does not redo the
  work. More docs commits may land and go red while this epic runs. The watcher and the
  land agent handle that; see Phase watcher and Landing.
- **sase-1kx** and every descendant down to **sase-1kx.5.3.3.3** are IN_PROGRESS with
  all phases closed and no live agent. **sase-1jz** has phases .3, .4, and .5
  IN_PROGRESS with no agent. Do not relaunch them. Only the land agent touches them, as
  directed by `stale_epics`.

## Rules for every phase

- Start from a fetched, fast-forwarded `origin/master`. Re-read the current failing
  Master Gate and Full CI runs before editing (`gh run list/view`, artifacts). Fix what
  is actually red now.
- Fix root causes. Do not add skips, xfails, raised test-wait timeouts, or goldens
  copied from CI. Do not loosen exact pixel comparison.
- Verify through `sase tool run check` (run `just fix` inline first), plus the
  phase-specific commands below. Do not run `just check-full`.
- Never merge PR #299 by hand, and never push to its branch. ci_watch owns the merge.
- Phase workers create no beads. Record discovered work as `PROPOSED FOLLOW-UP:` notes
  on your own phase bead, including any unrelated flake you observe, with its run ID.

## Phase gate-fix

Goal: every step of the Master Gate `lint` and `test` jobs passes at the commit this
phase lands.

1. List the failing steps and tests from the newest completed Master Gate run on master,
   and from the newest Full CI `lint` job.
2. Fix the mypy errors in `tools/docs_split` and `tools/render_keymap_macro_reference`.
   For `docs_split`, fix the stale-`target` bug itself: assign the target before the
   pointer is built. Add a regression test in `tests/test_docs_moves.py` that splits two
   sections with different targets and asserts each pointer names its own target.
3. Fix the docs tests by following the moved facts. A test that guards wording should
   read the page that now owns that wording; do not delete the assertion. For
   `test_docs_split_supports_h4_nested_intro_drop_and_finish`, keep generated pointer
   lines out of the source-origin rewrite pass so a relative pointer resolves from the
   page that contains it. Then assert the correct link form. For the xprompt-terminology
   test, allowlist the migration anchors and history (for example `docs/_meta/` and
   `renamed-from-xprompts` anchors) narrowly, by path or anchor, rather than weakening
   the term scan.
4. Run every lint-job step locally in workflow order, because the earlier failure hid
   the later ones: `just fmt-py-check`, `just fmt-md-check`, `just fmt-docs-check`,
   `just model-policy-check`, `just lint`, `just validate`,
   `just validate-committed-plans`, and `just build-check`. Fix each red. For the
   `docs/reference/index.md` hub budget, trim words without moving pages. Phase
   sase-1le.20.1.21 owns final hub content, so keep the edit minimal.
5. Run the formerly failing test files directly with `.venv/bin/pytest <files>`, then
   run `sase tool run check`.
6. Fetch again just before finishing. If the docs epic landed new reds, fix them the
   same way. If it already fixed one of yours, drop your duplicate edit.
7. Append a `DISCOVERED ISSUE:` note to **sase-1le.20.1**. List each red you fixed, its
   introducing commit, and your fix. Name the Deploy Docs `covers:` / mkdocs-exporter
   clash as theirs, unfixed.

## Phase visual-drift

Goal: no visual-test node fails deterministically on GitHub's runners.

1. Collect every visual node that failed in the 10 newest Full CI `visual-test` jobs,
   using the uploaded visual report artifacts and job logs. Classify each node as
   deterministic (fails in every run that reached it) or intermittent. Record the table
   as a note on this phase bead. The visual-stability phase and the land agent use it.
2. `help_guide_axe_120x40`: confirm that the text change in `axe_onboarding.py` was
   intended. Then regenerate with a targeted `just fix-tui-screenshots -- <selector>`
   and inspect the new golden.
3. `agents_lane_neighbors_above_context_160x50`: find out whether the missing `┊` wave
   divider is intended under `d5efed6535` ("scope layered wave dividers to clan roster")
   or is a rendering regression. Fix the code if it is a regression; otherwise
   regenerate the golden. Explain the call in the commit message.
4. CI-only readiness timeouts (`test_renamed_generic_agent_session_root_png_snapshot`,
   `test_agents_session_lane_neighbors_png_snapshot`, and any other node from step 1
   that times out): reproduce under load, using three xdist workers plus CPU contention.
   Make the readiness wait observe the semantic state that implies the repaint, instead
   of relying on timing. Raising the wait budget is not a fix.
5. Fix any other deterministic node from step 1 the same way.
6. Read `tui_screenshot.md` and `tui_perf.md` with `/sase_memory_read` before touching
   capture or wait logic.
7. Prove host independence for each changed node: run the targeted
   `just fix-tui-screenshots --check` twice, once with `TZ=UTC` exported. Inspect every
   golden change before committing. Then run `sase tool run check`.

## Phase visual-stability

> [!decision] visual_flakes = recapture

Make `just fix-tui-screenshots --check` tolerate a capture race without accepting drift.

- After the first pass, re-run only the failed visual nodes serially, with workers=1 and
  a fresh capture directory, for up to two more attempts. Reuse the update-mode
  verify-attempt machinery in `tests/ace/tui/visual/_visual_maintenance_verify.py`; do
  not write a second retry loop.
- A node passes only when one attempt both passes its test and matches its golden
  exactly. A node that fails every attempt keeps the exit code at 1.
- When more than 25 nodes fail the first pass, skip the retries and fail. Widespread
  drift is systemic, not a race.
- Check mode still never writes goldens.
- Report each node that passed only on retry as `unstable` in the run's report and
  manifest. Add a `GITHUB_STEP_SUMMARY` table listing those nodes even when the job
  succeeds, and upload the first-attempt diffs as an artifact. Flakes stay visible.
- Keep the visual-test job inside its 45-minute timeout. Measure the current job
  duration from recent runs and size the retry budget so the worst case still fits.
- Add tests for the CLI and report behavior: pass on the first attempt, pass on a retry
  (`unstable`), fail every attempt, and the over-cap fast failure.
- Update the reference prose in `docs/contribute/development.md` (and any other page
  that documents `--check`) per `docs_writing.md`.

> [!decision] memory_visual_check

Use `/sase_memory_write`, then edit only the sentence in `sase/memory/lint_and_test.md`
that says `--check` stays strict. It must say that check mode never writes, re-captures
failed nodes serially, passes only exact matches, and exits 1 on drift that reproduces.
Then run `sase memory init`.

> [!decision] visual_flakes = strict

Do not change the harness. Take every intermittent node from the visual-drift phase's
classification, or from your own pass over the same 10 runs if that note is not there
yet. Root-cause each one and pin the seam that races: the readiness predicate, the
deferred worker, or the clock or host input. Use the proof routine from
`tui_screenshot.md`, and record each node's root cause on this phase bead. Make no edit
to `lint_and_test.md` in this branch.

## Phase ci-budget

1. Pull the `persistent_query_keystroke` measurements from the perf-floors logs of the
   newest 80 Full CI runs. Recalibrate its regression-floor override in
   `tests/perf/baselines/phase7_regression_floor.json` from that hosted distribution,
   with documented margin. Keep the override's evidence test current
   (`tests/perf/phase7/test_phase7_check_regression.py::test_persistent_query_keystroke_override_documents_hosted_evidence`).
   Apply the same treatment to any other anchor whose recent hosted values sit within 2%
   of its ceiling.
2. Gather each Full CI job's duration over the last 20 runs. If any job's worst run used
   more than 90% of its `timeout-minutes` (the 3.12 coverage leg is the known
   candidate), raise that job's timeout so the worst observed run uses at most 75%. Cite
   the measured durations in the commit message. This is a job budget, not a test wait.
3. Run `sase tool run check`.

## Phase watcher

`tools/release_watch` already closes its bead on every verdict. Its known gaps:

- The bead close can be killed mid-push. The monitor supervisor sends SIGKILL to the
  process group 5 s after SIGTERM, and a close includes a push.
- Output is block-buffered under a monitor, so live progress is invisible.
- A close retry can leave a duplicate note.
- A red Master Gate at HEAD never ends the watch early.

Changes, all in `tools/_release_watch*.py`, with tests in
`tests/test_release_watch*.py`:

1. **Unbuffered output.** Flush every state line, and reconfigure stdout to line
   buffering at startup.
2. **Signal-path close that survives a group kill.** On SIGTERM, SIGINT, or SIGHUP,
   write the verdict file first. Then launch the `sase bead close` argv in a new session
   (`start_new_session=True`, output to a log beside the verdict file) and exit at once.
   The closer outlives the supervisor's group SIGKILL. Report signal exits as a new
   `STOPPED` verdict (exit 18) instead of `DEADLINE`. Normal-path closes keep their
   in-process retries.
3. **Idempotent closes.** Before each close attempt and each retry, read the bead.
   Already closed means success: no second note. At startup, if the `--close-bead` bead
   is already closed, print `ALREADY_CLOSED` and exit 0 without watching.
4. **`MASTER_GATE_RED`** (exit 17). With dispatch allowed, the watcher reruns failed
   jobs once per HEAD SHA (`gh run rerun --failed`) when only test shards failed. It
   ends with `MASTER_GATE_RED` after `--max-red-gate-minutes` (default 120) with no
   green Master Gate observed at HEAD. In-flight runs neither count as red nor reset the
   clock.
5. **Fresh red budget.** Count only Full CI runs created after the watch started toward
   `--max-red-full-ci`.
6. **`--no-dispatch`.** An observe-only mode that still closes the bead. It serves the
   `dispatch = false` answer.
7. Update the exit-code table and the "Watching a release" section in
   `docs/contribute/rust-backend.md`.
8. Prove it with the release-watch tests plus `sase tool run check`. Add a subprocess
   test showing that a SIGTERM followed 5 s later by a SIGKILL to the watcher's process
   group still closes a fake bead.

## Phase release-watch

This phase makes no file edits and always closes its bead with an honest verdict.

1. **Idempotent start.** Read this phase bead. If it is already closed, because this is
   a relaunch after recovery, run `/sase_final` and stop.
2. Fetch and fast-forward. Confirm that every sibling phase's commit is on
   `origin/master`, and use the newest `origin/master` SHA as FIXED_SHA. Pick CORE as
   the newest sase-core-rs on PyPI that passes `tools/check_sase_core_rs_bindings`
   against FIXED_SHA (0.40.1 at planning time). Inspect current Full CI, Master Gate,
   and PR #299, and do not duplicate an in-flight workflow.
3. **Preflight inline.** Append `--no-dispatch` to both commands when
   `dispatch = false`.

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <phase-id>
   ```

   It must exit 0 and print a fully parsed state line. On failure, close this phase with
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` and the output,
   then run `/sase_final`. Do not repair code in this phase.

4. **Start one `/sase_monitor`.** Use timeout `14h`, statuses `WATCHING RELEASE` /
   `RELEASE WATCHED`, and `--tail-lines 200`, around:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <phase-id>
   ```

   The `--next` follow-up instruction must say all of the following:
   - Make no edits.
   - Read `sase monitor show <id>`, the verdict file, and the phase bead.
   - If the bead is still open, close it. Use `--note @<verdict file>` for `RELEASED`.
     Otherwise use `-R canceled -r "release not confirmed: <VERDICT>"`, or
     `UNKNOWN (<monitor state>, exit <code>)` when no verdict exists.
   - Never reopen the bead, restart the watch, merge, dispatch, or create beads.
   - Finish with `/sase_final`.

   If `sase monitor start` exits nonzero twice, close canceled with
   `release not confirmed: WATCHER_NOT_STARTED` and both errors, then run `/sase_final`.
   Run the start command in the foreground until it exits.

### Why this bead always closes

Each layer has its own deadline, and each deadline finishes inside the next layer out:
the watcher stops at 12 h, the outer `timeout` at 12.5 h (plus a 10-minute SIGTERM
grace), and the monitor at 14 h.

| Path                                                 | Who closes the bead                                               | What releases the land agent              |
| ---------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------- |
| Any verdict, including the watcher's 12 h `DEADLINE` | Watcher, in-process retries                                       | Monitor follow-up runs `/sase_final`      |
| Outer `timeout` at 12.5 h                            | Watcher's signal path, a detached closer, inside the 10 min grace | Follow-up (monitor `failed`)              |
| Monitor timeout at 14 h (SIGTERM, then SIGKILL)      | Detached closer, or the follow-up if the close never ran          | Follow-up (monitor `timeout`)             |
| Watcher crash or hang                                | Outer `timeout` path above, or the follow-up                      | Follow-up                                 |
| Preflight or monitor-start failure                   | The phase agent, inline                                           | The phase agent's own `/sase_final`       |
| A human stops the monitor                            | Detached closer (`STOPPED`)                                       | Needs a human: `sase bead work <epic-id>` |
| Reboot (monitor `lost`) or a SIGKILL from outside    | Nobody; the bead stays open                                       | Needs a human: `sase bead work <epic-id>` |

The last two rows can only come from a human or the host, and SASE surfaces both.
`wait_checks` raises "Wait dependency can never self-resolve" for the parked land agent.
The recovery is safe because of step 1:

- `sase bead work <epic-id>` relaunches only the land agent when this bead is closed.
- It relaunches this phase when the bead is open, and the phase then watches again with
  a fresh preflight.

## Landing

The land agent confirms the release before landing. Closed phases are not proof.

1. **Confirm the release.** All of the following must hold:
   - PR #299 is merged by ci_watch.
   - The GitHub release v0.18.0 exists.
   - The Publish run's build and publish jobs succeeded.
   - `https://pypi.org/pypi/sase/json` lists 0.18.0.
   - In a fresh temporary environment outside the repo, unpinned `pip install sase`
     resolves 0.18.0, and `sase --version` prints it. Record the resolved sase-core-rs
     version.
   - Master Gate is green at the tip that ci_watch merged, and the Full CI run it relied
     on is green.
2. **Release not confirmed.** Read the release-watch verdict and evidence.
   - If only external waiting remains, start your own `/sase_monitor` running the
     watcher without `--close-bead`. Examples: ci_watch is merely waiting on an
     in-flight Full CI, or Publish has not run yet. Continue landing in the follow-up.
   - If repair is needed (`FULL_CI_RED`, `MASTER_GATE_RED`, `CORE_FLOOR_BLOCKED`,
     `INSTALL_FAILED`, or `PUBLISH_FAILED`), fix it in a child epic. It has a repair
     phase and a fresh release-watch phase that copies this plan's release-watch section
     and its close table verbatim. The child's land agent confirms the release and
     resumes this landing through `parent_bead`. A child epic is required because a tale
     cannot wait for the release and then resume this landing.
   - If ci_watch itself is not ticking or its merge path is broken, ask through
     `/sase_questions` rather than merging by hand.
   - Never land, or force-close this epic, without a confirmed release.
3. **Follow-ups.** Triage every `PROPOSED FOLLOW-UP:` note and every node that
   visual-stability reported `unstable` in the release's Full CI, through
   `/sase_new_task`. Corroborate existing tasks instead of duplicating them. Also triage
   these known non-blockers:
   - the `<deleted>` working-directory warnings from `os.chdir(ctx.workspace_dir)` in
     `src/sase/axe/run_agent_exec_retry.py` tests;
   - the intermittent Master Gate flakes, such as
     `test_link_follow_seam::test_beads_sync_reports_visible_and_missing`;
   - CI Telemetry's leak gate;
   - Install smoke, verified as healed.

> [!decision] stale_epics = supersede

After the release is confirmed:

- Close **sase-1kx** with `-R superseded --force`, a reason naming this epic, and the
  release evidence as the note. Before that, apply the evidence-based closeout its own
  plan assigned to its land agent for sase-16p, sase-1gh, sase-19d, sase-1kg, and
  sase-1ii: close only the tasks the release evidence proves, and note the rest.
- Close **sase-1jz** the same way only if this epic's Full CI evidence meets its goal.
  Otherwise append the evidence and leave it open.

> [!decision] stale_epics = notes

Append the release evidence to **sase-1kx.5.3.3.3**, **sase-1kx**, and **sase-1jz** as
notes. Close nothing outside this epic.

## Out of scope

- Deploy Docs (`covers:` / mkdocs-exporter): the docs epic owns it.
- CI Telemetry's leak gate.
- Releases of sase-github and sase-telegram.
- Changing ci_watch's gates or config.
