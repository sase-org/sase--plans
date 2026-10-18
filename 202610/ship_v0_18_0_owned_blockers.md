---
tier: epic
title: Own every release blocker, green Full CI, and ship sase v0.18.0 through ci_watch
goal: "sase 0.18.0 is on PyPI, published from release-please PR #299 after ci_watch
  merged it on a green Master Gate and a fresh green Full CI, with sase-core-rs 0.42.0
  published as its core floor. The release is confirmed before landing and recorded in
  an evidence artifact by the final epic lander.

  "
decisions:
  core_cut:
    ask:
      Let the release watcher dispatch sase-core's release-plz cut once the core release
      PR is green?
    default: true
    why:
      Core's release docs name dispatch as the urgent-fix path; the daily cut adds up to
      24 h
    answer: true
  stale_epics:
    ask:
      After the release is confirmed, how should the final lander treat the stale
      v0.18.0 release epics?
    choices:
      supersede:
        Close the sase-1lh, sase-1kx and sase-1c1 chains as superseded, citing the
        evidence
      notes:
        Append the release evidence as notes and close nothing outside this epic chain
    default: supersede
    why: They have no live agents and exist only to ship v0.18.0
    answer: supersede
phases:
  - id: core-fixture
    title: Make the core non-UTF-8 snapshot fixture filesystem-independent
    depends_on: []
    size: small
    description:
      "core-fixture: build the non-UTF-8 path into the Git index without writing it to
      disk, so sase-core's macOS CI and the 0.42.0 cut go green."
  - id: watcher-core
    title: Teach the release watcher about the core release PR
    depends_on: []
    size: medium
    description:
      "watcher-core: read the sase-core release-plz PR, stop with CORE_CI_RED when it is
      red, and optionally dispatch the core release cut."
  - id: deck-scroll
    title: Fix the files Ctrl-J anchor-scroll Full CI flake at its root
    depends_on: []
    size: medium
    description:
      "deck-scroll: make test_files_ctrl_j_scrolls_page_anchor_to_top (sase-1a7)
      deterministic under Full CI load."
  - id: tui-pilots
    title: Fix the archive, screenshot-export and statistics Full CI flakes
    depends_on: []
    size: medium
    description:
      "tui-pilots: root-cause sase-1mj, the screenshot frame-convergence timeout, and
      sase-1a2."
  - id: runtime
    title: Fix the usage-probe, git-maintenance and stderr-burst Full CI flakes
    depends_on: []
    size: medium
    description: "runtime: root-cause sase-1k9, sase-1j4 and sase-1ht outside the TUI."
  - id: new-project
    title: Fix the New Project create and configure Master Gate races
    depends_on: []
    size: medium
    description:
      "new-project: make the create-collision, configure-review and late-form New
      Project tests (sase-1mi, sase-1m9) settle deterministically."
  - id: gate-pilots
    title: Fix the link-follow and prompt mid-word Master Gate flakes
    depends_on: []
    size: medium
    description:
      "gate-pilots: root-cause sase-1lr, the bounded-panes scan one-off, and the
      mid-word peek family including sase-1l7."
  - id: watch
    title: Watch sase-core 0.42.0 and sase 0.18.0 to PyPI and always close
    depends_on:
      - core-fixture
      - watcher-core
      - deck-scroll
      - tui-pilots
      - runtime
      - new-project
      - gate-pilots
    size: small
    description:
      "watch: run one bounded tools/release_watch monitor that closes this phase bead on
      every outcome, so the lander always runs."
proposed_by: bbugyi200.athena.11b
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 09:58:26
status: wip
---

- **PROMPT:**
  [prompts/202610/ship_v0_18_0_owned_blockers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/ship_v0_18_0_owned_blockers.md)

# Plan: Own every release blocker, green Full CI, and ship sase v0.18.0 through ci_watch

## Why this epic exists

PyPI's newest sase is still 0.17.1 (2026-08-29). Release-please PR #299
(`chore(master): release 0.18.0`, head branch `release-please--branches--master`) has
been open since 2026-08-30. At least five epic chains have tried to ship it and ended
without a release: **sase-1c1**, **sase-1i5.9.1**, **sase-1jz**, **sase-1kx** (down to
sase-1kx.5.3.3.3) and **sase-1lh** (down to sase-1lh.7.5.4.3.3.4.4, whose last watch
ended `FULL_CI_RED` at 2026-10-18 12:09Z). They share one failure mode. Each repaired
the reds it had caused, then labeled the remaining ones "externally owned": the Full CI
flakes, the Master Gate flakes, and a macOS-only core fixture. Each then only observed.
None of those owners fixed anything.

**This epic owns every release blocker.** No phase or lander may defer a red to another
owner and wait. If another active epic introduced the root cause, fix it here anyway.
Then leave a resolution note on that epic so its lander does not redo the work.

"Full check" in the request means the **Full CI** heavy lane
(`.github/workflows/full.yml` calling `ci.yml`). ci_watch gates on it together with
Master Gate. `just check-full` is a local recipe that CI never runs. No phase or lander
runs it.

### What ci_watch needs, all in the same 5-minute tick

The logic is `_evaluate_release_branch` and `plan_release_merge` in
`bugyi_chops/ci_watch.py`. To read it, open the external repo `gh:bbugyi200/bugyi-chops`
with `/sase_repo`. The config is the `ci_watch` block in the chezmoi `sase_athena.yml`.

1. **Master Gate** has completed green at the exact master HEAD SHA
   (`gating_workflows`).
2. The **newest completed Full CI** run on master is green and finished within
   `heavy_max_age_hours: 6`. It may be on any master SHA, and a newer red run hides an
   older green one. ci_watch dispatches Full CI itself when the newest run is older than
   2 h and none is in flight.
3. **PR #299** is the only open release-please PR. It is `MERGEABLE` with
   `mergeStateStatus == CLEAN`, and every check is SUCCESS, SKIPPED or NEUTRAL. No
   release-please or Publish run is in flight.

ci_watch then merges with the `merge` method. Publish (`.github/workflows/publish.yml`)
runs on cron `17 */3` or on dispatch. In order, it runs release-please (tag plus GitHub
release), `build`, `install-smoke`, `install-smoke-core-floor`, then `publish` to PyPI.
Its `sync-release-metadata` job runs `tools/ratchet_core_window`, which moves the
release branch's `sase-core-rs` window to the newest published core.

### Blockers as of 2026-10-18 ~13:45Z (re-derive before acting)

1. **Core floor.** PR #299 head `572431efe0` fails `release-core-floor-smoke` (run
   39097781483). Its reconciled window is `sase-core-rs>=0.41.0,<0.42.0`, and 0.41.0 is
   missing 1 of the 866 required bindings: `mobile_pairing_issue`. Once a containing
   core is published, the reconciler fixes the window. That core is **sase-core-rs
   0.42.0**.
2. **Core 0.42.0 cannot cut.** sase-core's release-plz PR #331
   (`chore: release v0.42.0`) merges only from the `Merge release PR` job. That job runs
   on the daily `41 7 * * *` cut or on a `workflow_dispatch` with `dry_run=false`, and
   it waits on the PR's checks with `gh pr checks --watch --fail-fast`. The
   `cargo fmt + clippy + test (macos-latest)` job fails deterministically on core master
   `99b10bd0` (run 39128445283) and on PR head `e5ebd961` (run 39128523651). The failing
   test is
   `project_stats::snapshot::tests::odd_filenames_and_non_utf8_paths_are_preserved`,
   which panics at `crates/sase_core/src/project_stats/snapshot/tests.rs:222`:
   `fs::write` of `src/\xff.rs` returns OS error 92, `Illegal byte sequence`, because
   macOS rejects non-UTF-8 file names. The result is 5186 passed, 1 failed. Ubuntu and
   maturin jobs pass. Stats phase sase-1m3.7.1.2 introduced the test in `733256ed`. Epic
   sase-1m3.7.1 has carried a DISCOVERED ISSUE note about it since 03:15Z and has not
   fixed it.
3. **Full CI is a coin flip.** ae9f044bb3 has the same code as HEAD 7b9436bede apart
   from docs. Six Full CI runs on it were 3 green and 3 red, and each red was a
   different flake:
   - `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`:
     the 5 s `wait_for_anchor_scroll ... settled_on_live_target` wait timed out. Runs
     39104985574 and 39120426524, and earlier 38996026102 and 39009012994. Tracked as
     **sase-1a7** (+6). Earlier fixes in sase-1c1.7 and sase-1i5.7 did not hold.
   - `tests/ace/tui/test_screenshot_export.py::test_app_export_body_tolerates_signal_task_prepare_error`:
     `timed out waiting for screenshot frame convergence; stable_frames=2/3`, with
     identical digests and nothing pending. Runs 39122381498 and 39071692078. No bead.
   - `tests/llm_provider/test_muse_usage_probe.py::test_muse_usage_probe_missed_mint_is_a_timeout_not_absence`:
     `assert [] == ['initialize', ..., 'turn/start']` on Python 3.14. Runs 39122381498
     and 39074837393. Tracked as **sase-1k9**.
   - `tests/ace/tui/test_agent_archive_column_app.py::test_J_switches_to_the_pulse_and_enter_returns_to_the_inbox`:
     `'archive' == 'inbox'`. Run 39104985574. Tracked as **sase-1mj**; adjacent
     **sase-1ky** is a different node.
   - `tests/tool/test_observe.py::test_identical_contents_in_two_checkout_paths`:
     copytree races detached git maintenance's `.git/objects/maintenance.lock`.
     Run 39104985574. Tracked as **sase-1j4**.
   - Two more recur in the same code paths on older SHAs:
     `tests/llm_provider/test_subprocess_utf8_decode.py::test_stream_json_lines_dispatches_stderr_during_a_large_stdout_burst`
     (three runs, **sase-1ht**) and
     `tests/ace/tui/test_statistics_pane_filters.py::test_reverse_range_cycles_backward_wraps_and_reenters_from_custom`
     (**sase-1a2**).
   - The lint, visual-test, perf-floors and tutorial jobs are green on recent runs. The
     3.12 coverage leg takes 68–97 min against its 120-min ceiling; that margin is
     tracked in **sase-v8**.
4. **Master Gate flakes.** Master Gate is green at HEAD 7b9436bede right now, but recent
   runs show these flakes:
   - `tests/ace/tui/test_link_follow_seam.py::test_beads_sync_reports_visible_and_missing`
     reports PENDING instead of MISSING, 5 times in the last 40 runs (**sase-1lr**).
   - New Project races (**sase-1mi**, **sase-1m9**):
     `tests/ace/tui/modals/test_new_project_create_flow.py::test_name_collision_blocks_review_but_occupied_checkout_surfaces`,
     `tests/ace/tui/modals/test_new_project_configure_review.py::test_review_back_restores_draft_and_y_stores_model_submission`,
     `tests/ace/tui/modals/test_new_project_create_races.py::test_late_form_completion_keeps_current_owner_edits`
     and
     `tests/ace/tui/modals/test_new_project_create_flow.py::test_enter_with_available_name_creates_nothing`.
   - Mid-word peek:
     `tests/ace/tui/widgets/test_prompt_next_word_midword.py::test_midword_peek_reveals_then_finishes_word`
     and `::test_deferred_midword_defers_then_applies`, plus Full CI's **sase-1l7**
     (`test_ctrl_t_before_reveal_reveals_without_inserting`).
   - A one-off:
     `tests/ace/tui/test_link_follow_bounded_panes.py::test_archived_plan_reveals_through_path_context_after_scan`.
5. **Not ci_watch gates**, so triage only: Install smoke fails because `sase>=0.18` is
   not published yet and heals on release. Also out of scope are CI Telemetry's leak
   gate, Deploy Docs, the core-pin-ratchet PR #319, and the shard-timings PR #300.

### Existing tools to reuse

- **`tools/release_watch`.** Code is in `tools/_release_watch_api.py` and
  `tools/_release_watch_engine.py`, tests in `tests/test_release_watch*.py`, and
  contributor docs in `docs/contribute/rust-backend.md`.
  - **Reads:** PyPI, PR #299, Full CI runs whose SHA contains `--fixed-sha`, Master Gate
    at HEAD, Publish runs, PyPI sase-core-rs, and the ci_watch reason.
  - **Dispatches:** bounded Full CI reruns, a failed-shard Master Gate rerun, and the
    Publish floor-ratchet, PR-refresh and post-merge publish runs.
  - **Options:** `--await-core-version` keeps it watching while the published core is
    older. `--close-bead` closes the bead on **every** terminal verdict, using a
    detached closer when a signal arrives.
  - **Exit codes:** RELEASED 0, FULL_CI_RED 10, PUBLISH_FAILED 11, CORE_FLOOR_BLOCKED
    12, RELEASE_PR_GONE 13, INSTALL_FAILED 14, DEADLINE 15, WATCHER_BROKEN 16,
    MASTER_GATE_RED 17, STOPPED 18.
- **ci_watch state** lives in `~/.sase/axe/lumberjacks/ci_watch/`
  (`ci_watch_state.json`, `ci_watch_releases.json`).

### Concurrent work this epic must respect

- The stats epic **sase-1m3** / **sase-1m3.7.1** keeps pushing to sase-core master.
  Phase core-fixture changes only the fixture, then notes that epic.
- Other agents push to sase master. Every phase starts from a fetched, fast-forwarded
  `origin/master` and preserves concurrent changes.
- The stale release chains have no live agents. Do not relaunch them. Only the final
  lander touches them, as `stale_epics` directs.

## Rules for every phase

- Re-derive your area's current failures from fresh CI (`gh run list`,
  `gh run view --log-failed`) before editing. The snapshot above is a starting point.
- Fix root causes. Never skip, xfail, quarantine or delete a test or assertion. Never
  add rerun-on-failure, and never widen ci_watch or watcher gates. Raise a timeout only
  when the wait is already condition-based and you have evidence the latency is
  legitimate. The expected shape of a fix is a deterministic settle condition: wait for
  the actual state, drain the worker or timer that owns it, or inject the clock or
  scheduler. If the race is a real product bug, fix the product code.
- Prove each flake fix with a stress loop that mimics the CI lane. For example, run the
  node plus its module under `-n 8` for 20–30 iterations, with other test modules
  loading the host. Run the loop before and after the fix where it reproduces. Record
  iterations and failures on your phase bead. If it never reproduces, explain the race
  from the code and test the fixed condition directly.
- A failure in your area that fresh CI shows and the snapshot lacks is yours. For a
  failure outside every phase's area, append a `PROPOSED FOLLOW-UP:` note with run and
  job IDs to your phase bead. Phase workers never create beads and never close task
  beads. The watch phase is the only phase that closes a bead itself: its own.
- Touch shared helpers (`tests/conftest.py`, common Textual pilot wait helpers) only
  when the root cause lives there. Keep that change minimal and backward compatible, and
  note it on your phase bead, since sibling phases run in parallel.
- After editing sase files, read the lint_and_test memory, run `just fix`, then run
  `sase tool run check`. Never run `just check-full`. For core edits, use core's own
  recipes and `sase tool run check` from the opened core checkout.
- Never merge a PR by hand, edit versions, changelogs or dependency windows, or change
  ci_watch's code or config. Dispatch release workflows only where this plan says so.
- End with a phase note that names the task beads your fix addresses and the stress
  evidence, so the lander can close those beads with release evidence.

## Phase core-fixture

1. Use `/sase_repo` to open `sase-core`, read its `AGENTS.md`, then fetch and
   fast-forward. If core master already contains a fix for this fixture, verify it and
   go to step 4. Check `git log` for the test file and the macOS job of the newest
   master CI run.
2. Rewrite `odd_filenames_and_non_utf8_paths_are_preserved` in
   `crates/sase_core/src/project_stats/snapshot/tests.rs`. The non-UTF-8 path must reach
   the committed tree without any filesystem having to accept that name:
   - Write the blob with `git hash-object -w --stdin`.
   - Add the entry with `git update-index --add --cacheinfo 100644,<oid>,<raw path>`, or
     with `--index-info -z`, passing the raw bytes through `OsStr`.
   - Commit without `git add -A`, which would stage the missing worktree file as a
     deletion.
   - Keep `src/odd name.rs` on disk, both assertions (two Source files plus
     `assert_partition`), and the raw-byte path coverage.
   - Change the measurement code only if the test exposes a real bug.
3. Audit the other `project_stats` tests, and any core test, for file names that the
   macOS or Windows filesystems reject, and fix those the same way. Run the focused
   tests, core's format and lint recipes, and `sase tool run check` in the core
   checkout. If the `mac` tailnet host answers SSH, also run the focused test there.
   Otherwise core CI is the macOS proof, and the watch phase checks it.
4. Append a note to **sase-1m3.7.1** saying this phase resolves its DISCOVERED ISSUE.
   Name the change, but do not claim a commit SHA you do not yet have. Finalize
   normally. The host commits and pushes core master, and release-plz's push run then
   refreshes PR #331.

## Phase watcher-core

Extend `tools/release_watch` in place; do not create a second watcher. Reuse the
bounded-read, timeout, parsing and fake-runner patterns in `_release_watch_api.py`,
`_release_watch_engine.py` and the existing tests.

1. **Core release PR read.** Only when `--await-core-version` is set and the published
   core is older than it, read two things:
   - The open sase-core release-plz PR: head branch matching
     `^release-plz-\d{4}-\d{2}-\d{2}T\d{2}-\d{2}-\d{2}Z$` and title
     `chore: release vX.Y.Z`. Read its head SHA and check rollup.
   - The in-flight and recent sase-core `Release-plz` runs.

   Treat failed reads like the existing signals: three consecutive failures produce
   `WATCHER_BROKEN`.

2. **`CORE_CI_RED` verdict.** Append it to `EXIT_CODES` as exit code 19 and keep the
   existing codes. While awaiting core, stop with `CORE_CI_RED` when every check on the
   core PR's current head has completed, at least one failed, and that holds for a
   45-min grace. A new head SHA restarts the clock. The detail names the PR, the head
   and each failing check's name and URL. `--close-bead` closes it as canceled, like
   every other non-RELEASED verdict.
3. > [!decision] core_cut

   Add an opt-in `--dispatch-core-cut` flag. While awaiting core, dispatch
   `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`
   when all of these hold:
   - the core PR's checks are registered and none has failed (green or pending);
   - no `Release-plz` run is in flight;
   - no dispatch happened in the last 90 min.

   The merge job itself waits up to 45 min for pending checks. Cap dispatches at 3 per
   watch and record `dispatched:core_cut` in actions. Honor `--no-dispatch`, `--dry-run`
   and `--once`; the preflight prints `would dispatch:core_cut`.

   > [!decision] core_cut = no

   Do not add the dispatch flag. Implement steps 1, 2 and 4 only.

4. Render a `core_pr=` field in the state line, and add the core PR to the verdict
   evidence block. Add tests in the existing style for:
   - read parsing;
   - verdict grace and the reset on a new push;
   - dispatch gating, cooldown and cap, if the flag exists;
   - no-dispatch and dry-run;
   - rendering.

   Keep `--help` accurate. Read the docs_writing memory, then update the release-watch
   section of `docs/contribute/rust-backend.md`. Run `just fix` and
   `sase tool run check`.

## Phase deck-scroll

Target: `test_files_ctrl_j_scrolls_page_anchor_to_top` (**sase-1a7**). It is the most
frequent Full CI red, and two earlier fixes did not hold.

1. Read sase-1a7 and its corroborations. Read the earlier attempts, sase-1c1.7 ("files
   Ctrl-J settle races") and sase-1i5.7 ("Deterministic deck anchor-scroll settling",
   from sase-1br), and their commits. Explain why each did not hold under load.
2. Trace the Ctrl-J path from key to page-anchor scroll in the deck files view. Find
   what keeps the anchor off its live target for more than 5 s on a loaded runner.
   Candidates: layout not yet computed, a deferred refresh that recomputes the target,
   an animated or timer-driven scroll, or a worker that replaces the content after the
   scroll. Fix the production ordering, for example by scrolling after the layout that
   defines the target, or by making the test wait on the event that actually completes
   the scroll. Do not just extend the 5 s wait.
3. Stress-test as the rules require. Check the deck's sibling anchor tests in the same
   file and module for the same pattern, and fix those too.

## Phase tui-pilots

1. **sase-1mj**: `test_J_switches_to_the_pulse_and_enter_returns_to_the_inbox` stays on
   `archive`. Find which asynchronous step (focus, view switch or column mount) the test
   races, and whether adjacent sase-1ky (Today-banner repaint) shares the cause. If it
   does, fix both.
2. **Screenshot frame convergence**:
   `test_app_export_body_tolerates_signal_task_prepare_error` timed out with
   `stable_frames=2/3`. The digests were identical and no debouncer, worker, timer or
   animation was pending. Read the convergence loop (search for `stable_frames` under
   `src/sase/ace/tui/`) and the test's signal-task failure injection. Decide whether the
   loop's deadline is wall-clock rather than progress-based, or whether the injected
   failure path leaves a frame-producing callback. Make convergence complete
   deterministically once the app is quiescent, and keep its bounded deadline. Related
   history: sase-1dw (timer re-arm) and sase-123.7.6.4 (bounded finite visual settling).
3. **sase-1a2**: `test_reverse_range_cycles_backward_wraps_and_reenters_from_custom`
   fails once under load. Find the race between the range cycling and the pane's
   refresh.

## Phase runtime

1. **sase-1k9**: `test_muse_usage_probe_missed_mint_is_a_timeout_not_absence`. Under
   load the probe reports `usage probe timed out` before the fake server records any
   request, so the request list is empty. Separate process-start latency from the
   probe's mint deadline: synchronize on child readiness, or inject the clock or timeout
   so the test exercises the missed-mint path deterministically. Keep production timeout
   behavior.
2. **sase-1j4**: `test_identical_contents_in_two_checkout_paths`. Copying a repo with
   `shutil.copytree` races detached `git maintenance`, which deletes
   `.git/objects/maintenance.lock` mid-copy. Prefer turning off automatic maintenance
   for every test-created repo through the session git config that `tests/conftest.py`
   already writes (`[maintenance] auto = false`, `[gc] auto = 0`). Then confirm the
   observe tool's own copy path does not depend on that and needs no product change.
3. **sase-1ht**: `test_stream_json_lines_dispatches_stderr_during_a_large_stdout_burst`.
   Find whether the stderr dispatch assertion depends on reader-thread scheduling, and
   make the ordering contract explicit.

## Phase new-project

Targets: **sase-1mi** (the collision test leaves Review enabled), **sase-1m9**
(configure review asserts readiness before the debounced preview settles),
`test_late_form_completion_keeps_current_owner_edits` and
`test_enter_with_available_name_creates_nothing`. Also check **sase-1l8** (the bare-git
create PNG snapshot times out on its create check) for the same cause.

1. Map the New Project modal's asynchronous pieces: the name-availability and collision
   check, the debounced preview, and form-completion workers. For each test, identify
   which pieces it races.
2. Expose one deterministic settle point that tests can await, such as a
   validation-generation counter or a "preview settled" event, or drive the debounce
   through an injectable scheduler. Make Review's enabled state derive only from settled
   validation of the current input. If a stale validation can still enable Review, that
   is a product bug; fix it.
3. Stress every listed node, and the rest of `tests/ace/tui/modals/test_new_project_*`,
   under `-n 8`.

## Phase gate-pilots

1. **sase-1lr**: `test_beads_sync_reports_visible_and_missing` reports PENDING instead
   of MISSING under xdist load. Find what keeps the missing bead's resolution pending,
   such as a background resolver, a cache warmup or a timeout race, and make the
   reported state settle deterministically. Check
   `test_archived_plan_reveals_through_path_context_after_scan` in
   `tests/ace/tui/test_link_follow_bounded_panes.py` for a shared scan-completion race.
2. **Mid-word peek family**: `test_midword_peek_reveals_then_finishes_word`,
   `test_deferred_midword_defers_then_applies` and sase-1l7
   (`test_ctrl_t_before_reveal_reveals_without_inserting` inserts the peeked word under
   Full CI). Find the shared deferred-reveal timing and make reveal and insert ordering
   deterministic.

## Phase watch

This is the release-watch section. Nested epics copy it verbatim, close table included,
substituting only their own bead IDs. Make no source edits, and read `/sase_monitor`
first. **Every path through this phase ends with this phase bead closed**, so the lander
always runs.

1. If this phase bead is already closed, use `/sase_final` and stop.
2. Fetch and fast-forward the sase checkout. Confirm that every sibling repair phase's
   change is on `origin/master`, using its closing note and `git log`. If a change is
   missing, start one `/sase_monitor` 10-minute `sleep` whose `--next` resumes this
   step. If the change is still missing after that, close this bead with
   `sase bead close <watch-bead> -R canceled -r "WATCH_PRECONDITION: <what is missing>" --note "<evidence>"`
   and finalize. Otherwise set `FIXED_SHA` to the fetched `origin/master` tip.
3. Open sase-core with `/sase_repo` and fetch it. Set `CORE` as follows:
   - If the containing release is already published, use that version.
   - Otherwise use the version the open release-plz PR proposes (currently 0.42.0).
   - The chosen version must contain the core-fixture change and `mobile_pairing_issue`.
     Never use 0.41.0.

   If `CORE` is not yet published, confirm the open release PR's head contains the
   core-fixture commit. If every check on that head has completed and one failed, close
   this bead canceled with `-r "CORE_CI_RED: <failing checks>"` and the job URLs as the
   note, then finalize.

4. Run `sase bead epic-symbols <watch-bead>`; it must be empty. Never key an exemption
   to a watch bead. If an entry needs a Justfile edit, close this bead canceled with
   `-r "WATCH_PRECONDITION: epic symbols"` and an explanatory note, then finalize; the
   lander handles it.
5. If PyPI already lists sase 0.18.0, prove it yourself: install `sase==0.18.0` in a
   fresh temporary venv and run `sase --version`. Close this bead normally with that
   output and finalize.
6. Preflight with the concrete values:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --once --dry-run --close-bead <watch-bead>
   ```

   > [!decision] core_cut = no

   Omit `--dispatch-core-cut` from both the preflight and the monitor command.

   The preflight must exit 0 and print a fully parsed state line. If it does not, close
   this bead canceled with `-r "WATCHER_BROKEN (preflight)"` and the output as the note,
   then finalize. Record `FIXED_SHA`, `CORE`, the preflight output and the exact monitor
   command in a note on this bead.

7. Start exactly one monitor, with placeholders substituted:

   ```sh
   sase monitor start -C "$PWD" -r 'Watch sase-core <CORE> and sase 0.18.0 to PyPI; the watcher closes this phase bead' -t 14h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.18.0 release watch' -m '@small' -n '<NEXT>' -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --deadline 12h --close-bead <watch-bead>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <watch-bead> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   `<NEXT>` is this text, with its IDs substituted: "Release-watch closeout for
   `<watch-bead>` of epic `<epic-id>`. Do not restart the watcher, dispatch anything,
   merge, or edit files. Read the monitor result and the verdict file it names. Run
   `sase bead read <watch-bead>`. If the bead is not closed, close it: normally with the
   verdict file as the note only if the verdict is RELEASED; otherwise with
   `-R canceled -r "release not confirmed: <verdict or monitor outcome>"`. Append a
   one-paragraph summary to `<epic-id>` with `sase bead note`, giving the verdict, exit
   code, verdict-file path and last state line. Then use /sase_final."

   Do not pass `--idle-timeout`. Let `sase monitor start` itself run to completion. If
   it fails twice, close this bead canceled with `-r "WATCHER_NOT_STARTED"` and both
   errors as the note, then finalize.

Three layers close this bead: the watcher's own `--close-bead` on every verdict,
including signals; the shell fallback after the watcher exits; and the follow-up's
check. The fallback's canceled close is refused harmlessly when the watcher already
closed the bead as done. A canceled watch is not a release, and the lander decides what
follows.

| Watcher verdict                                                                                                | How this bead closes        | What the lander does                                     |
| -------------------------------------------------------------------------------------------------------------- | --------------------------- | -------------------------------------------------------- |
| RELEASED                                                                                                       | done, verdict file as note  | Confirms the full proof, then lands                      |
| FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED, CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE | canceled, verdict as reason | Repairs through a nested epic                            |
| DEADLINE, STOPPED, WATCHER_BROKEN, WATCH_PRECONDITION, WATCHER_NOT_STARTED                                     | canceled, verdict as reason | Runs its own watcher if only waiting remains, else nests |

## Landing

The land agent confirms the release before landing. Closed phases are not proof. Nested
epics copy this section verbatim.

1. **Integration audit.** Read this epic, every phase and note, and this plan. Fetch
   sase and sase-core. Verify each repair phase's change is on master and that no
   concurrent commit undid it. Run `sase bead epic-symbols <epic>` and retire each entry
   under Symvision policy: wire it, privatize it, add a justified pragma, or delete it.
   Re-key an entry only to a still-open nested child bead. If you edit sase files, run
   `just fix` and `sase tool run check`. Never run `just check-full`.
2. **Confirm the release.** Every one of these must hold:
   - sase-core-rs `<CORE>` is on PyPI with every expected distribution. Its `v<CORE>`
     tag contains the core-fixture change, and the macOS CI job for the released commit
     passed.
   - PR #299 was merged by ci_watch. Show the merge commit, `mergedBy`, and the matching
     row in `~/.sase/axe/lumberjacks/ci_watch/ci_watch_releases.json`.
   - The GitHub release v0.18.0 exists.
   - The Publish run after the merge succeeded in `release`, `build`, `install-smoke`,
     `install-smoke-core-floor` and `publish`. A skipped job is not proof.
   - `https://pypi.org/pypi/sase/json` lists 0.18.0.
   - In a fresh temporary directory outside every repo, `uv venv` followed by an
     unpinned `uv pip install --refresh sase` resolves sase 0.18.0, and `sase --version`
     prints it. Record the resolved sase-core-rs version, which must be at least `CORE`.
   - Master Gate was green at the tip ci_watch merged, and the Full CI run it relied on
     was green and less than 6 h old.
3. **If the release is not confirmed,** read the watcher verdict and its evidence, then:
   - **Only external waiting remains** (core wheels still publishing, Full CI in flight,
     Publish not yet run, ci_watch between ticks): start your own `/sase_monitor`
     running the watch phase's watcher command **without** `--close-bead`, with `-t 14h`
     and a `--next` that resumes this Landing at step 2. Do this at most once for an
     unchanged state.
   - **GitHub release v0.18.0 exists but PyPI lacks 0.18.0,** and Publish's build or
     publish failed transiently: run
     `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=true`
     once, then take the waiting path.
   - **Repair is needed** (FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED,
     CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE, or a waiting
     path that already ran once without progress): create a nested child epic with
     `/sase_plan`, setting `parent_bead` to this epic. The child gets:
     - one repair phase for every observed blocker, owned outright and never deferred to
       another epic;
     - a watch phase that copies "Phase watch" and its close table verbatim;
     - this Landing section verbatim, so its lander keeps nesting until the release is
       observed, creates the artifact, and resumes its ancestors.

     Record on this epic what the child owns, then propose the plan. That ends your
     turn, and the child's lander resumes this landing through `parent_bead`.

   - **ci_watch is not ticking, or it refuses to merge while all three gates are
     green:** ask through `/sase_questions`. Never merge by hand.
   - Never close this epic, or force-close it, without a confirmed release.

4. **Evidence artifact.** The first lander to confirm the release is the final epic
   lander. It writes `sase_v0_18_0_release_evidence.md` in a fresh temporary directory,
   not inside a repo. The file carries every proof from step 2:
   - run, job and PR IDs, SHAs, URLs and times;
   - PyPI file names with their sha256 for sase 0.18.0 and core `<CORE>`;
   - the unpinned install output;
   - the watcher verdict file's content;
   - the flake beads closed in step 5.

   Then run
   `sase artifact create -p <file> -l "sase v0.18.0 PyPI release evidence" --bead <this-epic>`.
   For each ancestor epic in this chain, add
   `sase artifact link add <artifact> related bead:<ancestor> "v0.18.0 release evidence"`.
   Ancestor landings reuse this artifact and never create a second one.

5. **Follow-ups.**
   - Triage every `PROPOSED FOLLOW-UP:` note through `/sase_new_task`, corroborating
     existing tasks instead of duplicating them.
   - Close each flake task bead that a phase note says it fixed (sase-1a7, sase-1mj,
     sase-1k9, sase-1j4, sase-1ht, sase-1a2, sase-1lr, sase-1mi, sase-1m9, sase-1l7,
     sase-1l8, sase-1ky) only when the release's green Full CI and Master Gate runs plus
     the phase's stress evidence support it. Otherwise append the evidence as a note.
   - Note the outcome on sase-1m3.7.1. Close sase-10d if the evidence proves it.
   - Triage Install smoke, which should heal after the release, and the 3.12
     coverage-leg time margin (sase-v8).
6. **Stale epics.**

   > [!decision] stale_epics = supersede

   After confirming the release, close the **sase-1lh** chain (deepest open descendant
   first, from sase-1lh.7.5.4.3.3.4.4 up to sase-1lh), the **sase-1kx** chain (from
   sase-1kx.5.3.3.3 up to sase-1kx) and **sase-1c1**. Close each with
   `-R superseded --force -r "superseded by <this-epic>: sase v0.18.0 released"`, with
   the artifact ref and a one-line summary as the note. **sase-1i5.9.1** and
   **sase-1jz** still have non-release scope (plugin floors and goldens), so give them
   an evidence note only.

   > [!decision] stale_epics = notes

   Append an evidence note to the same epics and close nothing outside this epic chain.

7. **Close and resume.**
   - Close this epic with
     `sase bead close <epic> --note "<proof summary, artifact ref, follow-up dispositions>"`.
     If the close is refused, fix the cited readiness issue and retry. Never use
     `--force` for a successful landing.
   - Run `just symvision`, through the tool wrapper if it is guarded.
   - Open the plans repo with `/sase_repo`, read this plan with `sase artifact read`,
     and set `status: done` in it.
   - If this epic has a `parent_bead`, resume that parent's Landing at step 4 and reuse
     the existing artifact. Repeat up the chain, and stop with a blocker note at the
     first ancestor that is incomplete.
   - Use `/sase_final`, and include every repository you changed.

## Out of scope

- Changing ci_watch's logic or config, or merging any PR by hand.
- Releasing sase-github or sase-telegram, and raising plugin floors (sase-1i5.9.1.4).
- CI Telemetry's leak gate, Deploy Docs, and the stats epic's feature work beyond the
  fixture.
