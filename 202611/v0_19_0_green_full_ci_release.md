---
tier: epic
title:
  Green Full CI on master, a containing published core, and sase v0.19.0 confirmed on
  PyPI
goal: "Full CI is green in every job on master. sase-core's master CI is green, and a
  complete published sase-core-rs contains master's source pin. A release.yml rehearsal
  at master is accepted. Release-please PR #322 merges, and sase 0.19.0 is confirmed on
  PyPI. The lander that observes the release records the evidence as an artifact.

  "
decisions:
  merge_path:
    ask: "Who should merge release PR #322 and start release.yml at the v0.19.0 tag?"
    choices:
      steward:
        The release steward, after E2's steward fixes land and you enable merge
        authority; slower
      agent:
        This epic's release phase, using the steward's own merge argv; faster, bypasses
        the steward
    default: steward
    why:
      The steward is the designed single decider, and E2's exit wants a steward release
    answer: steward
  reboot_watchdog:
    ask:
      Install a self-removing hourly crontab watchdog that relaunches this epic after a
      reboot strands it?
    default: true
    why:
      The 2026-11-01 reboot stranded the last v0.19.0 epic's agents and monitors for
      days
    answer: true
phases:
  - id: watchdog
    title: A reboot cannot strand this epic
    depends_on: []
    size: small
    description:
      "watchdog: install a self-removing hourly crontab watchdog that relaunches this
      epic's agents after a reboot, or close at once when declined."
  - id: gate-red
    title: Master Gate is green at master HEAD
    depends_on: []
    size: small
    description:
      "gate-red: fix the marker-path audit red left by the readout split, plus any other
      deterministic Master Gate red at the tip."
  - id: blocking-flakes
    title: The blocking test lanes stop flaking
    depends_on: []
    size: medium
    description:
      "blocking-flakes: fix the races behind intermittent reds in Master Gate's jobs and
      in Full CI's lint and test (3.12|3.13|3.14) jobs."
  - id: advisory-lanes
    title: Full CI's advisory jobs pass
    depends_on: []
    size: medium
    description:
      "advisory-lanes: make visual-test, perf-floors, tutorial-tests, and
      ace-page-group-isolation pass on Full CI."
  - id: core
    title: A complete published sase-core-rs contains master's source pin
    depends_on: []
    size: medium
    description:
      "core: turn sase-core's macOS CI green, wait for the daily core cut, and ratchet
      master's exact core pin to it."
  - id: full-ci
    title: One Full CI run on master is green in every job
    depends_on:
      - gate-red
      - blocking-flakes
      - advisory-lanes
    size: medium
    description:
      "full-ci: dispatch Full CI on a tip holding every fix, repair what is still red,
      and repeat until a first attempt is green."
  - id: rehearse
    title: An accepted release.yml rehearsal at master
    depends_on:
      - gate-red
      - blocking-flakes
      - core
    size: small
    description:
      "rehearse: dispatch and accept a rehearse-mode release.yml run at master, then
      make sure the human merge-authority step is covered."
  - id: release
    title: sase v0.19.0 reaches PyPI under a self-closing watcher
    depends_on:
      - rehearse
    size: small
    description:
      "release: drive or watch release PR #322 through release.yml to PyPI, with a
      watcher that always closes this bead."
proposed_by: bbugyi200.athena.178.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 11:09:28
status: wip
---

- **PROMPT:**
  [prompts/202611/v0_19_0_green_full_ci_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/v0_19_0_green_full_ci_release.md)

# Plan: Green Full CI on master, a containing published core, and sase v0.19.0 on PyPI

The user asked for three things:

1. Get Full CI green on master.
2. Get release-please PR **#322** (`chore(master): release 0.19.0`) merged.
3. Get sase **0.19.0** published to PyPI.

Every lander keeps nesting epics until 0.19.0 is observed on PyPI, and the final lander
records the evidence as an artifact. The target is **0.19.0**. "The release" always
means 0.19.0.

## How a sase release happens now

`ci_watch` no longer merges sase release PRs. The chezmoi commit `027a28b9` (2026-10-25)
dropped `sase-org/sase` from its release lists, and its job description says sase is
"watched for CI health only and never merged". The **release steward**
(`.github/workflows/release-steward.yml` plus `tools/release_watch`) is the single
decider. Read `docs/contribute/release-train.md` before working on any release phase.

Each steward tick asks the Rust policy for one decision. The order is:

1. `core_ci_red`
2. `core_cut_red`
3. `core_lag`: no complete published `sase-core-rs` contains `sase-core-revision.txt`.
4. Master Gate red at HEAD, with one rerun allowed.
5. A stale release PR head or core pin: dispatch `publish.yml` to refresh the metadata.
6. A `release-hold` label.
7. Master Gate green, then the release PR's checks green.
8. The PR's exact core pin equals the core-ensure pick.
9. Daily cadence.
10. `releasable`. The merge itself happens only when `RELEASE_STEWARD_MERGE_ENABLED` is
    the literal `true`.

After the merge:

- `publish.yml` (release-please) creates tag `v0.19.0` and a draft release.
- `release.yml -f mode=release` at the tag builds and seals the candidate.
- It qualifies the candidate by calling `ci.yml` at the tag. `lint` and
  `test (3.12|3.13|3.14)` are blocking; the other jobs are advisory.
- It then runs four exercise legs, publishes, confirms from PyPI, and un-drafts.

A workflow dispatched at a tag runs that tag's own files. A candidate broken by its own
`release.yml` is burned, so 0.19.0 can be lost. That is why the rehearsal must exercise
the `release.yml` that the tag will carry.

GitHub throttles this repo's crons heavily. The steward is scheduled every 15 minutes,
but it ran only eight times between 2026-11-01 16:40 and 2026-11-03 11:57 UTC.
sase-core's `41 7 * * *` daily cut has run hours late.

## Where things stand (measured 2026-11-03 ~15:45 UTC, master `bdbbce02e3`)

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Phase                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| 1   | **Master Gate is red at HEAD.** Every push since `323a91d69f` (`refactor(instructions): split readout.py into focused submodules`) has failed the same test. The test is `tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed`. It reports two unreviewed contexts: `src/sase/instructions/_readout_build.py:build_readout` and `src/sase/instructions/_readout_classify.py:classify_artifact_dir`. Runs: 41647853290, 41652212970, 41655264057, 41656828775.                                                                                                                                                                                                      | gate-red                         |
| 2   | **Blocking-lane flakes.** The newest Full CI (41592764416 at `d3518e567d`) failed `test (3.14)` on `tests/ace/tui/widgets/decks/test_deck_view_files_pilot.py::test_stale_results_dropped_and_reset_to_auto_redecides` (a 5 s `wait_for` timeout). Earlier blocking-lane reds include: `tests/monitor/test_monitor_supervise*.py` and `test_monitor_settlement_starter_race.py` on 3.12 and 3.13 (41546786403); `test_prompt_todo_highlight.py` (41479626139); `test_app_import_budget.py` on all three versions (41398737190); and `test_prompt_mini_macro_location_flow_existing.py` (41312037659). Master Gate flakes are tracked as **sase-1ql** (create-review button) and **sase-1ny** (process-group interruption). | blocking-flakes                  |
| 3   | **Advisory reds.** `visual-test` timed out in `test_archived_identity_reply_png_snapshot` ("timed out waiting for 'the archived prompt reads back verbatim' in the Archive view"). It failed in three of the last four Full CI runs, including serial recapture. `test_clan_waves_ladder_png_snapshots` failed once. `perf-floors` failed `tests/test_sase_tool_runs_smoke.py::test_tool_runs_harness_live_owner_cases_pass` (case `dod-17-live-escalate-join`) and the "Phase 7E floor check" step. Full CI has been red on every scheduled run since at least 2026-11-01.                                                                                                                                                | advisory-lanes                   |
| 4   | **sase-core master CI is red on macOS.** `cargo fmt + clippy + test (macos-latest)` fails in `cargo test --workspace` on every push since `b0a63dc3a2` (2026-11-01). `3c2a9dac` ("gate kernel ETXTBSY replay tests to Linux") was under CI at planning time. Core release PR **#343** (`chore: release v0.43.4`) is waiting for green checks. The steward reports `core_cut_red` (run 41491146023, macOS job cancelled), and before that it reported `core_ci_red`.                                                                                                                                                                                                                                                        | core                             |
| 5   | **Core lag.** Master's `sase-core-revision.txt` is `f1bbb4f3`, master pins `sase-core-rs==0.43.2`, and the newest complete core on PyPI is 0.43.3. `publish.yml`'s `sync-release-metadata` exits 3 on every run (`CORE_LAG(...)`). As a result, #322's `release-core-floor-smoke` stays red on a stale pin.                                                                                                                                                                                                                                                                                                                                                                                                                | core                             |
| 6   | **No rehearsal since 2026-10-25.** The last `release.yml` run is 40234446645, and it was red.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | rehearse                         |
| 7   | **The steward cannot merge or start a train yet.** Two E2 defects are owned by the live epic **sase-1n6.11**: `merge-checks` (phase .1, running) and `train-start` (phase .3, waiting). Its `containment` phase (.2, running) changes `release.yml`'s build and seal. Merge authority is unset. **sase-1oe.4** owns the human cutover gate, but it has no live agent or monitor.                                                                                                                                                                                                                                                                                                                                           | release (`merge_path`)           |
| 8   | **Stalled duplicates.** **sase-1q2** → **sase-1q2.9** (phases .2 core-pin, .3 rehearse, .4 watch) and **sase-1pi.1** (macOS core) are IN_PROGRESS with no live agent or monitor since the 2026-11-01 reboot. This epic takes over their remaining work.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | core, rehearse, release, landing |

Out of scope: `install_broken (macos)` (issue #325, task **sase-1r0**). The steward
lists it as a secondary finding that never blocks a qualified candidate.

## Decisions

> [!decision] merge_path = steward The release steward merges #322 and starts
> `release.yml`. That needs two things this epic does not own:
>
> - sase-1n6.11's `merge-checks` and `train-start` fixes on master;
> - `RELEASE_STEWARD_MERGE_ENABLED=true`, which only a human sets. rehearse step 9
>   raises that gate if nobody else has.
>
> The release phase watches for up to 72 h. Once merge authority is on, it pokes the
> throttled steward. sase 0.19.0 becomes E2's first steward release.

> [!decision] merge_path = agent The release phase merges #322 itself once the steward's
> own checks pass. It uses the steward's exact argv. The watcher then dispatches
> `publish.yml` and one `release.yml -f mode=release` at the tag. No merge-authority
> gate is raised. E2's first-steward-release proof (sase-1n6.11.4) moves to the next
> version, and the release phase records that on sase-1n6.11.4.

> [!decision] reboot_watchdog The watchdog phase installs a cron entry. After a reboot,
> the entry relaunches this epic's dead agents. It removes itself when the epic closes.

## Rules for every phase

- **Idempotent start.** A reboot or the watchdog may relaunch you.
  1. First run `sase bead read` on your own bead. If it is closed, run `/sase_final` and
     stop.
  2. Read your earlier notes and resume from them.
- **Read memory first.** Read `lint_and_test.md` and `sase_beads.md` through
  `/sase_memory_read`. Also read:
  - `tui.md` for TUI or visual work;
  - `ace.md` before changing `src/sase/ace/`;
  - `tools.md` before changing `tools/`;
  - `symvision.md` if a symvision gate fails.
- **Re-inventory first.** Fetch, then re-inventory on the `origin/master` tip before you
  edit anything. Other epics land on master every hour. The tables in this plan are a
  starting point, not a whitelist.
- **Respect ownership.** Run `sase bead search '<test or symptom>'` and
  `sase agent list`.
  - If an owner has a live agent or monitor, leave the work to it and say so in your
    close note.
  - Otherwise do the work, and append `TAKEN OVER by <your-phase-id>: <what>` as a note
    on the owner's bead.
  - Name every task bead your evidence resolves. The lander closes them.
- **Fix waits, fixtures, and products, never tolerance.**
  - Do not raise timeouts, budgets, perf floors, or caps.
  - Do not add retries around assertions, and do not skip or xfail.
  - Do not loosen pixel comparison or delete assertions.
  - Wait on the semantic state an assertion is about, with
    `sase.ace.testing.wait.wait_for` or the file's existing helper.
  - If the product settles late or inconsistently, fix the product.
  - A perf test may change how it measures (for example, warm best-of-N batches, or
    timing in an untraced child interpreter) only when the product did not regress. It
    never gets a looser bound.
- **Goldens.** Never copy CI candidate PNGs into the tree. Regenerate a golden only
  after its test is deterministic. Use `just fix-tui-screenshots` with a selector after
  `--`, run through `/sase_monitor`. Inspect the report and the golden diff.
- **Rust boundary.** Shared behavior belongs in sase-core
  (`rust_core_backend_boundary`).
  - Work there through `sase repo open sase-core -r '<why>'`, follow its `AGENTS.md`,
    and run `sase tool run check` in that checkout.
  - Move sase's `sase-core-revision.txt` only when sase needs the new core commit. Every
    pin move brings core lag back until the next daily cut.
- **Verify.** Run `just fix`, then `sase tool run check`, in every checkout you changed.
  Never run `just check-full`. CI's `lint` job enforces `toobig`, so do not grow a file
  past its line limit.
- **Hand waits to monitors.** Hand every long command to `/sase_monitor` with an
  explicit deadline: verification, CI waits, PyPI waits, and polls. Use `-w 0` for pure
  waits. Never end a turn waiting.
- **Outward actions you may take:**
  - `gh workflow run full.yml`;
  - `gh workflow run release.yml ... -f mode=rehearse`;
  - `gh run rerun <id> --failed`, where a phase says so;
  - the release phase's actions in its `merge_path` branch;
  - the rehearse phase's cutover gate.
- **Outward actions you must never take:**
  - Merge any sase-core PR, including #343.
  - Dispatch `release-plz.yml`. The user approved `core_cut = wait` in sibling epics, so
    core cuts wait for the daily cut.
  - Run `gh variable set`, or edit the `pypi` environment, outside a human gate.
  - Merge #322, except in the release phase's `agent` branch.
  - Create PRs, branches, tags, or commits by hand. Completion is host-owned.
- **E2 boundary.**
  - Do not close, edit, or reassign beads under sase-1n6, sase-1oe, or sase-1pi. Notes
    are allowed where a phase says so.
  - Do not touch the steward policy and facts code that sase-1n6.11 is changing
    (`merge-checks`, `containment`, `train-start`) while its owner is live.
- **Memory.** Edit no memory.
- **No new beads; always close.**
  - Record follow-ups as `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own
    phase bead.
  - `sase bead epic-symbols <own-phase-id>` must list nothing.
  - Close with `sase bead close <id> -R done -n '<evidence>'`.
  - If you cannot finish, still close with `-R canceled` and the evidence, so dependent
    phases and the land agent always run.
  - Finish through `/sase_final`.

## Phase watchdog

> [!decision] reboot_watchdog = no Close this bead at once with
> `-R canceled -r "declined by the reboot_watchdog decision"`, then run `/sase_final`.

> [!decision] reboot_watchdog This phase changes no repo files.

1. **Paths.** Use `D=$HOME/.local/state/sase-epic-watchdog/<epic-id>`. Never use `/tmp`,
   because it must survive a reboot. It holds:
   - `watchdog.sh`;
   - `boot_id` (the last boot handled);
   - `installed_at`;
   - `watchdog.log`;
   - `crontab.before`.

   Resolve the project's primary checkout (`sase repo list --json`) and bake its
   absolute path into the script.

2. **Write `watchdog.sh`** in POSIX sh.
   - Set `PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin` and `NO_COLOR=1`, and `cd`
     to the primary checkout.
   - Honor `WATCHDOG_STATE_DIR` (default `$D`) and `WATCHDOG_DRY=1`. Dry mode prints
     each gate's verdict and the command it would run, writes nothing, and never runs a
     non-preview `sase bead work`.

   Each run evaluates these gates in order:
   1. **Self-removal.** If `sase bead show <epic-id> --format json --no-links` reports
      the epic closed, or `installed_at` is more than 14 days old, remove its own
      crontab line with the safe edit in step 3 and exit 0. If the show command itself
      fails, log and exit 0. Never act on unknown state.
   2. **Boot gate.** If `/proc/sys/kernel/random/boot_id` equals the stored `boot_id`,
      exit 0. If `/proc/uptime` is under 900 s, exit 0 without recording, so host
      services have time to start.
   3. **Innermost epic.** If `sase bead show <epic-id>.. --format json --no-links` lists
      an open child epic, record the boot id and exit 0. A nested epic carries its own
      watchdog.
   4. **Liveness.** If `sase agent list -j` has any agent whose name starts with
      `<epic-id>.`, or `sase monitor list -j` has an active monitor for such an agent,
      record the boot id and exit 0.
   5. **Recovery.** Log `sase bead work <epic-id> -n`. If it fails, or its preview kills
      or cleans anything, note
      `WATCHDOG: post-reboot recovery preview refused: <reason>` on the epic and record
      the boot id. Otherwise run `sase bead work <epic-id> -y`, then note
      `WATCHDOG: relaunched after reboot <boot id>: <one-line result>` on the epic, and
      record the boot id.

3. **Safe crontab edit,** used for both install and removal.
   1. `crontab -l > "$D/crontab.before"` must exit 0.
   2. Build the new table from that file:
      - to install, append this line exactly once:
        `23 * * * * sh <D>/watchdog.sh >> <D>/watchdog.log 2>&1 # sase-epic-watchdog:<epic-id>`
      - to remove, drop the lines carrying that marker.

      Every other line must stay unchanged.

   3. Install the new table with `crontab <file>`.
   4. Verify that `crontab -l` matches the new file byte for byte. If it does not,
      restore it with `crontab "$D/crontab.before"`.

   Never pipe `crontab -l` straight into `crontab -`.

4. **Install.** Write the current boot id to `boot_id` and the time to `installed_at`,
   so the watchdog acts only after the next reboot. Then install the line.

5. **Prove it.**
   1. Run `sh -n` on the script.
   2. Run `WATCHDOG_DRY=1 WATCHDOG_STATE_DIR=<scratch dir whose boot_id is "test">`. The
      liveness gate must find this epic's live agents (at least you) and report no
      action.
   3. Run the real script once. It must exit 0 with "same boot; no action".
   4. Show that the crontab diff is exactly one added line.

6. Note on the epic bead the cron line, the paths, and the one-line removal command.
   Then close `done`.

**Done when** the line is installed and proven, or this bead is closed as declined.

## Phase gate-red

**Repo:** sase.

1. Fetch and read the newest Master Gate at `origin/master` HEAD
   (`gh run list -R sase-org/sase --workflow master-gate.yml --limit 5`). If the audit
   test already passes on master, skip to step 3.
2. **Marker-path audit** (Evidence #1). The readout split moved marker-path-passing code
   into `_readout_build.py:build_readout` and
   `_readout_classify.py:classify_artifact_dir`.
   1. Review both sites the way the audit's docstring requires.
   2. Update `_REVIEWED_PATH_PASSING_CONTEXTS`. If it still lists the pre-split
      `readout.py` contexts, replace those entries; otherwise add the new ones.
   3. If the review finds an unsafe pattern, fix the code instead.
   4. Check every sibling marker audit (`tests/test_agent_artifact_marker_*audit*.py`)
      for the same split.
3. **Re-inventory** Master Gate's `lint`, `docs`, `core-wheel`, and `test (1)` through
   `test (8)` at HEAD. Fix every deterministic red. Leave flakes to blocking-flakes, and
   name them in your close note.
4. **Prove it.** Run every marker audit file, then `just fix` and `sase tool run check`,
   and land.

**Done when** no deterministic Master Gate red remains at the tip and the repo check
passes.

## Phase blocking-flakes

**Repo:** sase. Read `tui.md`, and `ace.md` before touching `src/sase/ace/`.

1. **Inventory.** Collect the failing nodes (`gh run view <id> --log-failed`) from:
   - the last 10 completed Full CI runs;
   - the last 30 completed Master Gate runs.

   Use only the blocking jobs:
   - Master Gate's `lint`, `docs`, `core-wheel`, and `test (N)`;
   - Full CI's `lint` and `test (3.12|3.13|3.14)`. The 3.12 leg runs under coverage.

   Classify each node as one of:
   - fixed on master (name the fixing commit);
   - deterministic now (reproduce it, then fix it);
   - a flake.

2. **Known candidates** (Evidence #2):
   - the deck files pilot `test_stale_results_dropped_and_reset_to_auto_redecides`;
   - sase-1ql (`test_create_review_button_enters_review_and_back_restores_configure`
     sees `checking`);
   - sase-1ny
     (`test_interruption_kills_process_group_with_pipe_holding_descendant[False]`);
   - the monitor supervisor and settlement tests with leftover workspace claims;
   - `test_todo_background_yields_to_selection_search_yank_and_cursor`;
   - `test_existing_typeahead_seeds_finder_query`;
   - the TUI import budget. Check first whether a later commit already fixed it.

3. **Fix each live flake's race** under the phase rules. Prove each fix:
   - run the node 20 times serially, on the failing Python version where it matters;
   - run its file 3 times under `-n 8` alongside a busy sibling directory, for example
     `tests/ace/tui/`.

   If you cannot find the race, record a `PROPOSED FOLLOW-UP:` with the output and the
   owner bead.

4. Run `just fix` and `sase tool run check`, then land. Name the task beads your
   evidence resolves.

**Done when** every inventoried flake is fixed with that proof or recorded with its
evidence, and the repo check passes.

## Phase advisory-lanes

**Repo:** sase. Read `tui.md`. The jobs are `visual-test`, `perf-floors`,
`tutorial-tests`, and `ace-page-group-isolation`.

1. **Inventory** these jobs over the last 10 Full CI runs, and check owners with
   `sase bead search`.
2. **Known reds** (Evidence #3):
   - **Archived identity reply.**
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_archive_states.py::test_archived_identity_reply_png_snapshot`
     fails at `_ace_agents_archive_png_snapshot_shared.py:137`. It times out before
     capture, even on serial recapture, so it is a settle or product problem, not a
     pixel diff.
   - **Clan waves ladder.** `test_clan_waves_ladder_png_snapshots` failed on render
     convergence.
   - **DoD-17 live harness case.** `test_tool_runs_harness_live_owner_cases_pass` fails
     case `dod-17-live-escalate-join`. sase-1cx.7 added that case; read its notes.
   - **Phase 7E floor check.** Read the step log for the violated floor. sase-1lh.4
     fixed an earlier flake in that ceiling.
   - `tutorial-tests` and `ace-page-group-isolation`: fix them only if the inventory
     shows them red.
3. **Prove each fix:**
   - a visual node passes 3 consecutive `just test-visual -- <selector>` runs;
   - a perf node or floor script passes 5 consecutive local runs of the job's own
     command (`-m slow` where the node is marked slow);
   - every perf change records before and after numbers.
4. Run `just fix` and `sase tool run check`, then land.

**Done when** each advisory red is fixed with that proof or recorded as a
`PROPOSED FOLLOW-UP:` with its evidence.

## Phase core

**Repos:** sase-core (`sase repo open sase-core -r '<why>'`) and sase. Start by
appending `TAKEN OVER by <this-phase-id>` notes to sase-1pi.1 and sase-1q2.9.2. Recheck
that neither has a live agent first.

1. **sase-core master CI.** Run
   `gh run list -R sase-org/sase-core --workflow ci.yml --branch master --limit 5`.
   - If the newest completed push run is green on both OSes, go to step 3.
   - If one is still running, hand
     `gh run watch <id> -R sase-org/sase-core --interval 120 || true` to `/sase_monitor`
     with `--next` and about a 2 h deadline.
2. **Fix macOS.** Read the failing `cargo test --workspace` output.
   - Distinguish a failed job from a cancelled one. `runner-starvation-retry.yml` reruns
     starved jobs.
   - Fix the cause in the sase-core checkout, and verify there.
   - A test-only core fix does **not** move sase's `sase-core-revision.txt`.
   - After landing, monitor the new push run until the macOS job is green.
3. **Core release PR.** Find the open release-plz PR (#343 at planning time). Every
   check at its head must be green for the daily cut to merge it. A newer push to
   sase-core master refreshes it. Never merge it, and never dispatch a cut.
4. **Containing core.** In a sase checkout fast-forwarded to `origin/master`, run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: record it and go to step 6;
   - exit 2: go to step 5;
   - exit 3 (`CORE_LAG`): hand a poll to `/sase_monitor` with `--next` and about a 40 h
     deadline. That covers more than one late daily cut. Every 300 s, run
     `git fetch -q origin && python3 tools/ratchet_core_pin --check --fetch --ref origin/master`
     until the exit code leaves 3. If the pin moves again, the poll simply waits for a
     cut that contains the new pin.

   You may rerun a failed release-plz push run once with
   `gh run rerun <id> --failed -R sase-org/sase-core`, but only if all of these hold:
   - its tag contains the pin;
   - only the upload or attestation steps failed, with a transient infrastructure error;
   - the run is at attempt 1;
   - nothing newer is queued for that tag.

5. **Ratchet master.**
   1. Run `python3 tools/ratchet_core_pin --allow-transitive-lock-refresh`, then
      `uv lock`. The diff touches only `pyproject.toml` and `uv.lock`, and
      `sase-core-revision.txt` stays unchanged.
   2. Reproduce `release-core-floor-smoke`:
      1. Create a fresh `uv venv --python 3.12` outside the repo.
      2. Run `uv pip install -e '.[dev]' "sase-core-rs==<X>"`.
      3. Run `tools/check_sase_core_rs_bindings`, then `tools/validate_sase_core_rs`.

      It must report 0 missing bindings.

   3. Run `just fix` and `sase tool run check`, then land.

6. **Notes.** On sase-1pi.1, sase-1q2.9.2 and sase-1oe.4, record:
   - the containing core's version and peeled tag SHA;
   - its five PyPI filenames;
   - the ratchet commit.

If the wait times out or the core release fails deterministically, record a
`PROPOSED FOLLOW-UP:` and close `-R canceled`.

**Done when** all of these hold:

- core master CI is green on both OSes;
- the check reports a complete containing core;
- master pins that core exactly;
- the floor smoke reports 0 missing bindings.

## Phase full-ci

**Repo:** sase. This phase is the user's "full check".

1. **Prerequisites.** Find the commits of gate-red, blocking-flakes and advisory-lanes
   on `origin/master` with `git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`.
   - A phase whose note says it needed no commit is fine.
   - A canceled prerequisite is not fatal. Proceed, and let the run show what is red.
2. **Dispatch.**
   1. If a Full CI run is already queued or running at a SHA containing every
      prerequisite commit, watch it. Otherwise run
      `gh workflow run full.yml -R sase-org/sase --ref master` and find its run ID.
   2. Hand `gh run watch <id> -R sase-org/sase --interval 120 || true` to
      `/sase_monitor` with `--next` and about a 4 h deadline.
3. **Judge every job.** Each of these must be `success`:
   - `prepare`;
   - `build-core / build-core`;
   - `lint`;
   - `test (3.12)`, `test (3.13)` and `test (3.14)`;
   - `visual-test`;
   - `perf-floors`;
   - `tutorial-tests`;
   - `ace-page-group-isolation`.

   `release-core-floor-smoke` and `docs-build` are skipped outside pull requests, and a
   skip there is not red.

4. **Repair.** Fix each red under the rules and with the proof of the phase that owns
   its lane, land the fix, and dispatch again. Allow at most 3 rounds.
   - A green scheduled Full CI run at a tip containing the fixes also counts.
   - Only a run whose **first attempt** is green in every job satisfies this phase.

Close `done` with the run URL, its SHA and each job's conclusion. After 3 rounds without
such a run, close `-R canceled` with every red and its evidence.

**Done when** one Full CI first attempt is green in every job at a SHA that contains
every fix.

## Phase rehearse

**Repo:** sase. This phase changes no files. Read `/sase_monitor` first.

1. **Prerequisites.**
   - If core closed `canceled`, close this phase
     `-R canceled -r "rehearsal not started: core canceled"`.
   - A canceled gate-red or blocking-flakes phase is recorded, not fatal: the
     qualification shows it.
2. **E2 containment.** sase-1n6.11.2 changes `release.yml`'s build and seal, and the
   rehearsal must exercise the `release.yml` that the tag will carry.
   - If `git log origin/master --grep 'SASE_BEAD=\[sase-1n6.11.2\]'` is empty and
     sase-1n6.11.2 has a live agent, hand a 300 s poll for that commit to
     `/sase_monitor` with `--next` and about a 24 h deadline.
   - On timeout, or if the owner is dead, proceed and record that.
3. **Pin.** Run `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: no override;
   - exit 2: add `-f core_version=<core-ensure pick>` and record why;
   - exit 3: poll as in core step 4 (about 40 h, `--next`), then redo this step. Close
     `-R canceled` only on timeout.
4. **One rehearsal at a time.** If a `release.yml` run is queued or in progress, check
   what it is:
   - a `rehearse` run at a master SHA containing the prerequisites: watch it instead of
     dispatching;
   - anything else: wait for it to finish first.
5. **Rehearse.**
   1. Dispatch
      `gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse`, plus
      any override from step 3.
   2. Hand `gh run watch <id> -R sase-org/sase --interval 120 || true` to
      `/sase_monitor` with `--next` and about a 4 h deadline.
   3. Judge the result job by job. The overall conclusion is red even when everything is
      healthy, because of the `publish` criterion below.
6. **Acceptance.** Accept only if all of the following hold:
   - `identity`, `build` and `qualify-prepare` are green.
   - `qualify` is green, which requires `lint` and `test (3.12|3.13|3.14)` green.
     - If attempt 1 is red only through failing test jobs, run
       `gh run rerun <id> --failed` once, as the train itself may, and judge again.
     - An advisory red does not fail acceptance. Record it as a `PROPOSED FOLLOW-UP:`.
   - All four `exercise-legs` and the merged `exercise` verdict are green. Both wheel
     legs resolve a previous version (expected 0.18.1) and pass the N-1 upgrade.
   - `publish` (verify-only) refuses with `PUBLISH_CONFLICT` for every sealed file and
     nothing else. Master's version, 0.18.1, equals the last release.
   - `confirm` and `confirm-install` are skipped.
7. **Notes.** On sase-1oe.4 and sase-1q2.9.3, record:
   - the run URL and the verdict for each criterion;
   - the master SHA and its pin;
   - whether containment had landed.
8. **Close.**
   - If accepted, close `-R done`.
   - If not, record the failing job and step as a `PROPOSED FOLLOW-UP:` and close
     `-R canceled`.
9. **Cutover.**

   > [!decision] merge_path = agent Skip this step and finish through `/sase_final`.

   > [!decision] merge_path = steward This step runs after the close, and only if the
   > rehearsal was accepted.
   >
   > 1. If
   >    `gh api repos/sase-org/sase/actions/variables/RELEASE_STEWARD_MERGE_ENABLED --jq .value`
   >    prints `true`, finish through `/sase_final`.
   > 2. If a pending gate already covers it (check with `/sase_notify`), or sase-1oe.4
   >    has a live agent or monitor, record which one in a note and finish.
   > 3. Otherwise, raise one `/sase_gate` as your last action:
   >    - Title: "Enable release-steward merge authority for sase v0.19.0", icon 🚀.
   >    - Notes:
   >      - the accepted rehearsal URL;
   >      - what the switch enables: the steward merges #322 once releasable, starts
   >        `release.yml` at the fresh tag once E2's train-start lands, and may dispatch
   >        one sase-core cut per UTC day;
   >      - the status of sase-1n6.11.1 and sase-1n6.11.3;
   >      - the command that reverses it.
   >    - Query `(merge_authority AND pypi_master_policy) OR reject`, with the `AND`
   >      branch as the primary branch.
   >    - `merge_authority` runs
   >      `gh variable set RELEASE_STEWARD_MERGE_ENABLED --body true -R sase-org/sase`.
   >    - `pypi_master_policy` deletes only the `pypi` environment's `master`
   >      branch-deployment policy, looked up by name (51457521 at E2's last check). It
   >      refuses if the `v*` tag policy is missing.
   >    - `reject` is a no-op that prints JSON.
   >
   >    First append a note on sase-1oe.4 and sase-1n6.11.4 saying that this phase
   >    raised the cutover gate, so they must not raise a duplicate. Then follow
   >    `/sase_gate` for how the turn ends.

**Done when** a rehearsal is accepted, or the phase closes `canceled` with the evidence.

## Phase release

This is the phase dedicated to waiting for the release. **It always closes its own
bead.** It makes no repo edits. It ends by starting one monitor with **no `--next`**,
and the monitored watcher closes the bead itself.

1. **Idempotent start.**
   - If this bead is closed, run `/sase_final` and stop.
   - If `sase monitor list -j` shows an active "sase 0.19.0 watch" monitor for this
     phase, note that and stop the same way.
2. **Already released?** If `curl -fsS https://pypi.org/pypi/sase/0.19.0/json` succeeds,
   close `done` with that evidence and run `/sase_final`.
3. **Prerequisites.**
   - If rehearse closed `canceled`, close this bead
     `-R canceled -r "release watch not started: rehearse canceled"`.
   - A PR titled `chore(master): release 0.19.0` must be open or merged. If
     release-please retargeted the version, close
     `-R canceled -r "release watch not started: release target changed"`.
   - `sase bead epic-symbols <this-phase-id>` must list nothing.

   Every failure here closes `-R canceled` with the evidence, then runs `/sase_final`.
   Never repair code in this phase.

4. **Drive the merge.**

   > [!decision] merge_path = steward Skip this step. The steward merges.

   > [!decision] merge_path = agent All waits below go to `/sase_monitor` with `--next`.
   > Any timeout or red closes `-R canceled` with the evidence.
   >
   > 1. **Train freshness.** If any of `.github/workflows/release.yml`,
   >    `tools/release_candidate`, `tools/exercise_release`, `tools/_release_core.py` or
   >    `tools/_release_qualification*.py` changed on `origin/master` since the accepted
   >    rehearsal's SHA, run a new rehearsal and accept it by rehearse's criteria first.
   > 2. **Steward position.** Run `.venv/bin/python tools/release_watch --once --json`
   >    from a fresh clone of `origin/master` with its own `.venv`, built as
   >    `release-train.md` describes. Proceed only on one of two states:
   >    - `releasable`;
   >    - `waiting` with the detail "expected required checks unreadable". That is the
   >      known merge-checks defect, and the policy reaches it only after its core,
   >      gate, stale-PR and hold checks pass.
   >
   >    For any other state, poll every 15 minutes for up to 24 h. On a stale head or
   >    pin, you may dispatch `gh workflow run publish.yml -R sase-org/sase` at most
   >    once an hour.
   >
   > 3. **Independent checks**, at merge time:
   >    - Master Gate at `origin/master` HEAD is completed, and every job is `success`.
   >    - #322 is open, carries no `release-hold` label, and is `MERGEABLE`.
   >    - Every check at #322's head is `success`.
   >    - `python3 tools/ratchet_core_pin --check --fetch --ref origin/release-please--branches--master`
   >      exits 0.
   > 4. **Merge.** Run
   >    `gh pr merge 322 -R sase-org/sase --merge --match-head-commit <head-sha>`.
   >    Record the merge commit.
   > 5. **Notes.** On sase-1n6.11.4, record that this phase merged 0.19.0, so E2's first
   >    steward release is the next version.

5. **Write the watcher outside the repo.** Use
   `W=$HOME/.local/state/sase-release-watch/<this-phase-id>`, so the log survives a
   reboot, and run `mkdir -p "$W"`. Write `$W/watch.sh` from the script below with a
   quoted heredoc.
6. **Preflight inline.**
   1. Run `sh -n "$W/watch.sh"`.
   2. Run `sh "$W/watch.sh" <this-phase-id> "$W/watch.log" <mode> preflight`, where
      `<mode>` is the `merge_path` value.

   It must exit 0 and print either `outcome=RELEASED` or one `state` line in which `pr`,
   `steward` (a real kind, not `?`), `head` and `gate` are populated. If not, fix only
   the script's `gh`, `jq` or `grep` usage and retry once. If it still fails, close
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"`.

7. **Record, then hand off.** Append a note with:
   - the master HEAD;
   - the `ratchet_core_pin` result;
   - the preflight line;
   - the merge-authority value;
   - sase-1n6.11's phase states;
   - the exact monitor command.

   Then, from the workspace root, run (for `steward`: `72h30m` and `-t 76h`; for
   `agent`: `48h30m` and `-t 52h`):

   ```sh
   sase monitor start -C "$PWD" -w 0 \
     -r 'Watch sase v0.19.0 to PyPI; the watcher closes <this-phase-id>' \
     -t 76h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.19.0 watch' \
     -- 'timeout -k 5m 72h30m sh <W>/watch.sh <this-phase-id> <W>/watch.log <mode>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 60; sase bead close <this-phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   - Pass the command as one quoted string, with no `bash -c` wrapper.
   - Do **not** pass `--next`, `-m`, `-p` or `--idle-timeout`.
   - Follow `/sase_monitor` and `/sase_final` for how a staged start ends the turn.
   - If the start fails twice, close
     `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` with both errors.

### Watcher script

```sh
#!/bin/sh
# sase v0.19.0 release watcher: sh watch.sh BEAD LOG MODE [preflight]
# MODE is steward or agent. Never edits repo files. In watch runs it closes
# BEAD itself and always exits 0. Preflight never acts and never closes.
BEAD=$1 LOG=$2 MODE=$3 RUN=${4:-watch}
REPO=sase-org/sase VERSION=0.19.0 TAG=v0.19.0
TITLE="chore(master): release $VERSION"
RED=" qualify_red exercise_red core_cut_red core_ci_red core_incompatible publish_conflict dependency_unpublished vetoed ambiguous_trains watcher_broken "
case "$MODE" in steward) HOURS=72 ;; agent) HOURS=48 ;; *) echo "MODE must be steward or agent" >&2; exit 2 ;; esac
unset GH_FORCE_TTY
export NO_COLOR=1 CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0
END=$(( $(date +%s) + HOURS * 3600 ))
outcome=DEADLINE detail='' reruns='' seen_sid='' steward='?' sdetail='' red_ticks=0
train_red_since='' draft_since='' last_poke=0 last_publish=0 dispatched=''
say() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" | tee -a "$LOG"; }
act() { [ "$RUN" = watch ] && "$@" >>"$LOG" 2>&1; }
finish() {
  trap - HUP INT TERM
  say "outcome=$outcome $detail"
  [ "$RUN" = preflight ] && exit 0
  if [ "$outcome" = RELEASED ]; then res="done" reason="sase $VERSION confirmed on PyPI"
  else res="canceled" reason="release not confirmed: $outcome${detail:+ ($detail)}"; fi
  tail -n 60 "$LOG" > "$LOG.note"
  i=0
  while [ "$i" -lt 3 ]; do
    sase bead close "$BEAD" -R "$res" -r "$reason" -n "@$LOG.note" && break
    i=$((i + 1)); sleep 30
  done
  exit 0
}
trap 'outcome=SIGNALED detail="watcher terminated by a signal"; finish' HUP INT TERM
while :; do
  now=$(date +%s)
  pypi=$(curl -fsS "https://pypi.org/pypi/sase/$VERSION/json" 2>/dev/null \
    | jq -r '[.urls[].packagetype] | (index("bdist_wheel") != null and index("sdist") != null)' 2>/dev/null)
  [ -n "$pypi" ] || pypi=absent
  if [ "$pypi" = true ]; then outcome=RELEASED; break; fi
  pr=$(gh pr list -R "$REPO" --base master --state all --search "in:title \"release $VERSION\"" \
    --json number,title,state \
    --jq "[.[] | select(.title == \"$TITLE\")][0] | if . == null then \"none\" else \"\(.number):\(.state)\" end" 2>>"$LOG")
  auth=$(gh api "repos/$REPO/actions/variables/RELEASE_STEWARD_MERGE_ENABLED" --jq .value 2>/dev/null) || auth="unset"
  rel=$(gh release view "$TAG" -R "$REPO" --json isDraft \
    --jq 'if .isDraft then "draft" else "public" end' 2>/dev/null) || rel="none"
  train=$(gh run list -R "$REPO" --workflow release.yml --event workflow_dispatch --limit 20 \
    --json databaseId,status,conclusion,attempt,headBranch \
    --jq "[.[] | select(.headBranch == \"$TAG\")][0] | if . == null then \"none\" else \"\(.databaseId):\(.status):\(.conclusion):\(.attempt)\" end" 2>>"$LOG")
  case "$train" in
    *:completed:failure:*|*:completed:cancelled:*|*:completed:timed_out:*|*:completed:startup_failure:*)
      [ -n "$train_red_since" ] || train_red_since=$now
      if [ $((now - train_red_since)) -ge 3600 ]; then outcome=TRAIN_RED detail="release.yml $train"; break; fi ;;
    *) train_red_since='' ;;
  esac
  if [ "$rel" = draft ] && [ "$train" = none ]; then
    [ -n "$draft_since" ] || draft_since=$now
    if [ $((now - draft_since)) -ge 43200 ]; then outcome=TRAIN_NOT_STARTED detail="draft $TAG has no release.yml run after 12h"; break; fi
  else draft_since=''; fi
  if [ "$MODE" = agent ]; then
    case "$pr" in *:MERGED)
      if [ "$rel" = none ] && [ $((now - last_publish)) -ge 3600 ]; then
        last_publish=$now; act gh workflow run publish.yml -R "$REPO"
      fi
      if [ "$rel" = draft ] && [ "$train" = none ] && [ -z "$dispatched" ]; then
        dispatched=$now; act gh workflow run release.yml -R "$REPO" --ref "$TAG" -f mode=release
      fi ;;
    esac
  fi
  sid=$(gh run list -R "$REPO" --workflow release-steward.yml --status completed --limit 1 \
    --json databaseId --jq '.[0].databaseId // ""' 2>>"$LOG")
  if [ -n "$sid" ] && [ "$sid" != "$seen_sid" ]; then
    dj=$(gh run view "$sid" -R "$REPO" --log 2>>"$LOG" | grep -o '{.*"schema_version".*}' | tail -n 1)
    steward=$(printf '%s' "$dj" | jq -r '.primary.kind // "?"' 2>>"$LOG")
    sdetail=$(printf '%s' "$dj" | jq -r '.details // ""' 2>>"$LOG")
    seen_sid=$sid steward=${steward:-'?'} red=no
    case "$RED" in *" $steward "*) red=yes ;; esac
    # A fresh unsealed tag reads as a tag/manifest mismatch until E2's train-start lands.
    case "$steward:$sdetail" in watcher_broken:*"tag/manifest mismatch"*) red=no ;; esac
    if [ "$red" = yes ]; then red_ticks=$((red_ticks + 1)); else red_ticks=0; fi
  fi
  if [ "$red_ticks" -ge 2 ]; then outcome=STEWARD_RED detail="steward $steward (run $sid): $sdetail"; break; fi
  if [ "$auth" = true ] && [ $((now - last_poke)) -ge 1800 ]; then
    newest=$(gh run list -R "$REPO" --workflow release-steward.yml --limit 1 \
      --json status,createdAt --jq '.[0] | "\(.status) \(.createdAt)"' 2>>"$LOG")
    st=${newest%% *} ts=${newest#* }
    created=$(date -d "$ts" +%s 2>/dev/null || echo "$now")
    if [ "$st" = completed ] && [ $((now - created)) -ge 2700 ]; then
      last_poke=$now; act gh workflow run release-steward.yml -R "$REPO" -f live=true
    fi
  fi
  head=$(gh api "repos/$REPO/commits/master" --jq .sha 2>>"$LOG")
  gate=$(gh run list -R "$REPO" --workflow master-gate.yml --commit "$head" --limit 1 \
    --json databaseId,status,conclusion,attempt \
    --jq '.[0] | if . == null then "none" else "\(.databaseId):\(.status):\(.conclusion):\(.attempt)" end' 2>>"$LOG")
  case "$pr:$gate" in
    *:OPEN:*:completed:failure:1)
      id=${gate%%:*}
      case " $reruns " in *" $id "*) ;; *) reruns="$reruns $id"; act gh run rerun "$id" --failed -R "$REPO" ;; esac ;;
  esac
  full=$(gh run list -R "$REPO" --workflow full.yml --limit 1 --json databaseId,status,conclusion,headSha \
    --jq '.[0] | "\(.databaseId):\(.status):\(.conclusion):\(.headSha[0:10])"' 2>>"$LOG")
  say "state mode=$MODE pypi=$pypi pr=${pr:-?} authority=$auth release=$rel train=${train:-?} steward=$steward(${sid:-?}) head=${head:-?} gate=${gate:-?} full=${full:-?}"
  [ "$RUN" = preflight ] && exit 0
  [ "$(date +%s)" -ge "$END" ] && break
  sleep 300
done
finish
```

| Outcome             | Trigger                                                                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RELEASED`          | PyPI serves a wheel and an sdist for 0.19.0. The bead closes `done`.                                                                                                             |
| `TRAIN_RED`         | The newest `release.yml` run at `v0.19.0` stays red for 1 h. The grace hour lets the train or the steward rerun or resume first.                                                 |
| `TRAIN_NOT_STARTED` | A `v0.19.0` draft exists with no `release.yml` run for 12 h. In the steward path, this usually means train-start has not landed.                                                 |
| `STEWARD_RED`       | Two consecutive steward ticks report a red primary state. `waiting`, `core_lag`, `gate_red` and `releasable` are not red, and neither is a tag/manifest mismatch on a fresh tag. |
| `SIGNALED`          | The outer `timeout`, a monitor stop, or a shutdown signals the watcher.                                                                                                          |
| `DEADLINE`          | 72 h (steward) or 48 h (agent) passed.                                                                                                                                           |

Every outcome except `RELEASED` closes the bead `canceled`, with the last 60 log lines
as the note.

The watcher takes three actions on its own:

- It reruns a failed first attempt of Master Gate at HEAD once while #322 is open.
- Once merge authority is on, it pokes a steward whose newest tick is over 45 minutes
  old, at most every 30 minutes. It does not poke earlier, because every live tick also
  comments on issue #325.
- In the agent path, it creates the tag through `publish.yml` and makes the single
  `release.yml` dispatch.

### Why the land agent always runs

| Path                                            | Who closes the bead                                                                                                                     |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| any watcher outcome                             | the watcher (3 tries)                                                                                                                   |
| watcher hang, timeout, crash, or a failed close | the outer `timeout` signals it (`SIGNALED`), then the wrapper's fallback close runs after 60 s                                          |
| a human stops the monitor                       | the signal trap (`SIGNALED`)                                                                                                            |
| prerequisite, preflight, or start failure       | this agent, inline                                                                                                                      |
| a reboot or SIGKILL (the monitor is `lost`)     | the reboot watchdog relaunches this phase, and step 1 restarts the watch. Without the watchdog, a human runs `sase bead work <epic-id>` |

The monitor's own 76 h (or 52 h) timeout never fires on a live host. A fallback close
after a successful close is refused as a conflicting re-close, and `|| true` absorbs
that.

## Landing

The land agent confirms the release before it lands anything as done.

1. **If sase 0.19.0 is on PyPI,** verify each item and record it:
   - PyPI lists a wheel and an sdist. Record the filenames, SHA-256 values and upload
     times.
   - `gh release view v0.19.0 -R sase-org/sase` exists and is not a draft. Record the
     tag commit.
   - The merged release PR's `mergedBy`, `mergedAt` and merge commit, plus which path
     merged it: the steward run that executed the merge, or the release phase.
   - The `release.yml` run at `v0.19.0`: its ID and every job's conclusion, including
     `qualify`, `exercise`, `publish`, `confirm` and `confirm-install`.
   - The Master Gate run at the merge commit.
   - **Full CI.** The newest Full CI first attempt that is green in every job, at a SHA
     containing this epic's fixes: its ID, SHA and job conclusions.
     - If none exists, dispatch one more Full CI and fix small reds yourself.
     - Any red that does not block a release becomes a task bead through
       `/sase_new_task`, listed in the evidence.
   - **Clean install proof.**
     1. Outside any repo, with an isolated `HOME` and `UV_NO_CACHE=1`, create
        `uv venv --python 3.12` and run `uv pip install sase==0.19.0`.
     2. `sase version -j` must report 0.19.0 and the release's exact `sase-core-rs` pin.
     3. `sase core health --json` must pass.

   **Record the evidence.**
   1. If no epic in this lineage has recorded an evidence artifact yet (check the root
      epic's notes), write `sase_v0_19_0_release_evidence.md` in a temporary directory
      outside the repo.
   2. Register it with
      `sase artifact create -p <file> -l "sase v0.19.0 PyPI release evidence" -b <this-epic-id>`.
   3. Note the artifact ref on:
      - the root epic, if this one is nested;
      - sase-1n6.11.4, which owns E2's first-release proof. Do not close it;
      - sase-1pi.1 and sase-1oe.4.

   **Clean up, then close.**
   1. Close the task beads that phases named as resolved (for example, sase-1ql and
      sase-1ny), each with its evidence.
   2. Retire the stalled duplicate. Close sase-1q2.9.2, sase-1q2.9.3 and sase-1q2.9.4,
      then sase-1q2.9, then sase-1q2, each with
      `-R superseded -r "superseded by <root-epic-id>: sase 0.19.0 released, evidence <artifact-ref>"`.
      Use `--force` only where `sase bead close` names unfinished descendants.
   3. Remove the watchdog's crontab line with the safe edit, or confirm it is gone.
   4. Close this epic.

2. **If 0.19.0 is not on PyPI,** read every phase's close note and the watcher log
   (`~/.local/state/sase-release-watch/<release-phase-id>/watch.log`). Then fix the
   blockers:
   - **Small, clearly scoped blockers:** fix them yourself under the phase rules.
   - **Anything else:** author a **nested epic** with `/sase_plan`. It contains:
     - only the remaining blockers;
     - this plan's "Rules for every phase";
     - a watchdog phase, if `reboot_watchdog` was accepted;
     - a final release phase built exactly like this one, in the same `merge_path`
       branch;
     - this whole Landing section, verbatim.

     Carry this epic's DECISIONS forward as final.

   - **Steward path, merge authority still off.** Never change the settings yourself. If
     no pending gate covers the step, the nested epic's first phase raises it, as
     rehearse step 9 does.
   - **Steward path, E2's `merge-checks` or `train-start` not landed.**
     - If their agents are live, the nested watch keeps waiting.
     - If they are dead, preview `sase bead work sase-1n6.11 -n`. Relaunch only when the
       preview shows just those beads relaunching and nothing killed.
     - Otherwise the nested epic may take over that work under sase-1n6.11's plan, with
       `TAKEN OVER` notes on the E2 beads.
   - **A red `release.yml` at `v0.19.0`.** Coordinate with sase-1n6.11.4 and fix the
     cause. In the steward path, leave rebuild and resume to the steward. In the agent
     path, use `mode=resume` (or `rebuild: true`) at the same tag, per
     `release-train.md`.
   - **0.19.0 burned** (abandoned draft, or a defect frozen into the tag). Never
     retarget silently. Ask the user with `/sase_questions` whether the next
     release-please version satisfies the request, and nest per the answer.
   - Only the lander that actually observes 0.19.0 creates the evidence artifact.
   - Turn non-blocking `PROPOSED FOLLOW-UP` notes into task beads with `/sase_new_task`.

## Risks

- **Core lag returns.** sase-1n6.11's phases move `sase-core-revision.txt` when their
  core changes land. The daily cut heals that, and the watcher treats `core_lag` as
  waiting, not red.
- **Throttled crons.** Steward ticks arrive hours apart, and the core cut can run about
  8 h late. Deadlines are sized for that, and the watcher pokes the steward once merge
  authority is on.
- **A burned candidate loses 0.19.0.** Rehearsing the same `release.yml` the tag carries
  is the guard. The lander asks the user before retargeting.
- **The human cutover gate (steward path).** Nothing merges until you approve it. The
  watch then ends at its deadline, and the lander nests another watch.
- **Master churn.** Many epics land every hour. full-ci, the rehearsal's qualification,
  the watcher, and the lander absorb new reds.
- **A follow-up turn that never runs.** The core, full-ci and rehearse waits rely on
  `--next` follow-ups. The watchdog covers reboots; other host failures need
  `sase bead work <epic-id>`.

## Non-goals

- Making ci_watch merge sase release PRs again. The steward handover is the design.
- E2's steward code (`merge-checks`, `containment`, `train-start`), its memory
  decisions, and closing or editing E2 beads.
- `install_broken (macos)` (issue #325, sase-1r0).
- `just check-full` and its cost budgets.
- Dispatching sase-core cuts.
