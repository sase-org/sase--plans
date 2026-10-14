---
tier: epic
title: 'Ship sase v0.18.0: green CI, a floor-satisfying core, and a release wait that
  always closes'
goal: 'sase 0.18.0 is on PyPI and `pip install sase` works again. It ships through
  ci_watch''s guarded merge of release PR #299, after Master Gate and Full CI go green
  on master. The release-wait phase always closes its bead, whatever happens, so the
  land agent always runs. The land agent confirms the release before it lands the
  epic, and finishes the release itself when something unexpected blocks it.

  '
decisions:
  core_release:
    ask: Should the wait phase cut sase-core v0.40.0 now through release-plz's documented
      urgent dispatch?
    choices:
      dispatch: Dispatch release-plz dry_run=false at once; core 0.40.0 reaches PyPI
        in about 30 min
      daily_cut: Wait for release-plz's daily cron (lands ~15-17Z); can add most of
        a day of latency
    default: dispatch
    why: 'PR #299 cannot pass release-core-floor-smoke until a core >= 0.40.0 is on
      PyPI'
    answer: dispatch
  stale_epic_1jz:
    ask: How should landing release the stalled sase-1jz epic, whose .4/.5/.land agents
      still wait on .3?
    choices:
      notes: Close sase-1jz.3 and note .4/.5 with this epic's evidence; sase-1jz's
        own agents finish
      retire: Kill sase-1jz's 3 waiting agents and force-close the epic as superseded
        by this one
    default: notes
    why: notes lets sase-1jz's waiting agents finish themselves; it kills no agents
    answer: notes
phases:
- id: gate-reds
  title: Fix the Master Gate test reds at master HEAD
  depends_on: []
  size: medium
  description: 'gate-reds: delete the stale _RunManifest import, fix the lazy-export
    monkeypatch pollution that empties the bare-git provider registry, chase the archive-column
    flake, and fix every other deterministic red on the current tip.'
- id: visual-lane
  title: Make the Full CI visual lane pass and stay clean
  depends_on: []
  size: medium
  description: 'visual-lane: fix the three deterministic visual failure classes (fast-startup
    stub vs real loader, remote rows seeded through the local loader, stale Commits
    description wait), regenerate only the goldens those fixes change, then prove
    a clean TZ=UTC full check.'
- id: release-watch-tool
  title: A tested release watcher that always reports and can close its bead
  depends_on: []
  size: medium
  description: 'release-watch-tool: add tools/release_watch, a stdlib-only, unit-tested
    poller. It waits on PyPI, the release PR, Full CI, Master Gate, Publish, and sase-core-rs;
    dispatches Full CI or Publish only by fixed rules; fails fast on unparseable reads;
    and on every exit path writes a verdict and can close a bead.'
- id: release-wait
  title: Watch v0.18.0 to PyPI without editing code, then hand off
  depends_on:
  - gate-reds
  - visual-lane
  - release-watch-tool
  size: small
  description: 'release-wait: make no edits. Optionally cut sase-core, preflight the
    watcher inline, and start one monitor whose command runs tools/release_watch with
    --close-bead. Every outcome then closes this bead without a model turn, and the
    trivial follow-up only confirms the close.'
proposed_by: bbugyi200.athena.10b
decided_by: reviewer
decided_via: tui
create_time: 2026-10-14 07:18:01
status: wip
bead_id: sase-1kx
---

- **PROMPT:** [prompts/202610/v0_18_0_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_release.md)
- **BEAD:** [sase-1kx](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kx/README.md)

# Ship sase v0.18.0

PyPI's newest `sase` is 0.17.1, and it cannot be installed (`sase-16p`). v0.18.0 has
been waiting in release-please PR #299 since 2026-08-30. `ci_watch` (an AXE job on
athena, code in bugyi-chops `src/bugyi_chops/ci_watch.py`) merges that PR only when all
of these hold at once:

- Master Gate is green at the exact master HEAD.
- The newest completed Full CI run on master is green and its `updated_at` is at most 6h
  old.
- The release PR is `MERGEABLE` and `mergeStateStatus == CLEAN`, and every rollup check
  passed.
- No release-please or Publish run is in flight.

After the merge, a `Publish` run tags the release, builds it, smoke-installs it, and
uploads it with no manual approval. Publish's cron (`17 */3`) actually fires only every
4–9h, so it should be dispatched.

The previous attempt, epic `sase-1kl`, was canceled. Its fixes landed, but it never
released. This epic finishes the job and fixes the reason it stalled.

## Evidence (2026-10-14 ~10:40Z, master `8847d07a08`)

1. **Master Gate is red at HEAD** (run 38537047508):
   - Shard 6:
     `tests/instructions/test_scoreboard_conformance.py::test_conformance_warns_for_duplicate_root_and_leaked_helper_markers`
     raises `ImportError: _RunManifest`.
   - Shard 8: three `tests/project_onboarding/test_cli_bare_git.py` tests fail with
     `bare-git provider missing from registry: []`.

   Master Gate has been red on every push since `d69cd499dd`. Its lint job is green at
   HEAD, because `8847d07a08` removed the seven symvision leftovers.

2. **Full CI is red** (run 38530013190 on `3cff3bea4f`; it has not been green since
   2026-08-29):
   - **lint**: symvision. Already fixed at HEAD.
   - **All three test lanes** fail `test_scoreboard_conformance` (above) and
     `test_delivery_plan::test_declared_legacy_hookimpls_resolve_to_provider_legacy`.
     The `test_delivery_plan` failure is already fixed at HEAD.
   - **The 3.13 lane** also hit a one-off `test_agent_archive_column_app.py` `wait_for`
     timeout.
   - **visual-test**: 9 deterministic failures. The tests themselves fail; this is not
     golden drift. See Phase visual-lane.
3. **PR #299 is `UNSTABLE`.** Its `release-core-floor-smoke` check installs the declared
   floor, `sase-core-rs==0.39.1`, and reports that it is "missing 19 of 863 required
   bindings" (`artifact_link_index_*`, `*dismissed_bundle*`, `*agent_artifact_alias*`,
   …).
   - sase pins sase-core revision `a89eac09` (`sase-core-revision.txt`). That revision
     is unreleased.
   - sase-core PR #328 ("chore: release v0.40.0") is CLEAN and green, and its base is
     exactly `a89eac09`.
   - sase-core releases ship through `.github/workflows/release-plz.yml`. Its
     `release-plz-merge` job merges the release PR on the daily cron (`41 7`, which
     actually fires around 15–17Z) or on `workflow_dispatch -f dry_run=false`. The tag,
     build, and PyPI upload follow in about 30 minutes.
   - After that upload, a sase Publish run's `sync-release-metadata` job
     (`tools/ratchet_core_window`) moves PR #299's floor to the newest PyPI core and
     pushes the change. The PR's checks then re-run on their own.
   - ci_watch does not manage sase-core releases.
4. **Full CI's clock.** The `heavy_dispatch_every_hours` dispatcher from `sase-1kl.3`
   (bugyi-chops `5828753`) is configured but not live: the user's primary bugyi-chops
   checkout, which the `sase` tool env imports editable, is still at `285c5ad`. GitHub's
   cron starts Full CI only every 5–7h. Agents may dispatch Full CI with
   `gh workflow run full.yml -R sase-org/sase --ref master`. A run takes 80–180 min.
5. **Master moves constantly.** Epic `sase-1ko` (instruction delivery, run from apollo)
   still has `.7` and `.10` in progress. If `.7` turns `claude_explicit_instructions` on
   by default, it will flip `tests/instructions/test_delivery_plan.py:146` red again.

## Why the last two attempts stalled, and the closure contract

The land agent for an epic waits on two things:

- every phase bead is closed, with any resolution;
- the sink phase's agent session ends with a successful turn.

Nothing relaunches a phase whose bead is left open.

- **sase-1kl.4**, the release phase, found fixes in the middle of its wait. Its fixes
  could reach CI only after its own turn committed, so it left its bead open "on
  purpose", and the land agent never ran. Before that, it lost 4.5h to a monitor whose
  malformed `jq` polled nothing and still exited 0.
- **sase-1jz.3** exited without closing its bead after its joined check timed out. Three
  sase-1jz agents are still WAITING on it.

This epic therefore splits **fixing** (phases gate-reds, visual-lane, and
release-watch-tool, plus the land agent) from **waiting** (release-wait). The wait phase
is built so that closing its bead does not depend on any model's judgment:

1. **No edits.** release-wait changes no files and commits nothing, so there is never a
   reason to keep its bead open for its own CI. Any fix that turns out to be needed goes
   into its verdict, and the land agent handles it.
2. **Mechanical close.** The monitored command is
   `tools/release_watch --close-bead <id>`. On every exit path, the watcher writes a
   verdict and runs `sase bead close` itself:
   - release confirmed;
   - early-exit verdicts;
   - its own deadline;
   - an uncaught exception;
   - SIGTERM, SIGINT, or SIGHUP.

   An outer `timeout` sends SIGTERM if the watcher overruns. The monitor's `--timeout`
   is longer than both, so the supervisor never kills the close step. A bead closed from
   a monitor is attributed to the store owner, because monitors scrub agent identity.

3. **No silent polling.** The watcher validates every read. Three consecutive failed
   reads of the same signal end the watch with `WATCHER_BROKEN`, and that also closes
   the bead. The phase runs the watcher once inline (`--once --dry-run`) and requires a
   fully parsed state line before it starts the monitor.
4. **A failed start still closes.** If `sase monitor start` fails twice, the phase agent
   closes its bead inline with `WATCHER_NOT_STARTED`.
5. **The follow-up turn is trivial.** Its only job is to confirm the bead is closed (and
   to close it from the verdict if not), then submit `/sase_final`. A successful
   follow-up is what releases the land agent's agent-name wait.
6. **Residual risks:**
   - **athena reboots during the watch.** The monitor settles `lost` and launches no
     follow-up, so the bead stays open. Recovery is `sase bead work <this epic>`. That
     relaunches only the open phase, and the watcher is idempotent because every
     dispatch is deduplicated against GitHub's own state.
   - **The follow-up fails outright** (provider error after retries). The bead is
     already closed. `wait_checks` raises a can-never-resolve alert, and the remedy is a
     retry.

## Rules for every phase

- Read `lint_and_test.md` and `sase_beads.md` through `/sase_memory_read`. For TUI or
  visual work, also read `tui.md` and `tui_screenshot.md`.
- Re-inventory on the current master tip before editing. The Evidence above is a
  starting point, not a whitelist.
  - Use `gh run list -R sase-org/sase --workflow master-gate.yml` and
    `--workflow full.yml`.
  - Read job logs with
    `gh api --allow-escape-sequences repos/sase-org/sase/actions/jobs/<id>/logs` and
    strip the ANSI codes.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state an assertion or capture is about, never on a fixed pause or on visual
  idle.
- Change production code only for a real product bug. Shared domain behavior belongs in
  sase-core (`rust_core_backend_boundary`); none of this work is expected to need it.
- Never copy CI candidate PNGs into the tree. Regenerate a golden only after its test is
  deterministic.
- Run `just fix`, then `sase tool run check`.
  - Do not run `just check-full`; Full CI is the exhaustive signal.
  - CI's lint enforces `toobig`, so keep every file under its line limit.
  - athena's local `sase_core_rs` may be stale, which makes about 35 tests fail
    identically on a clean tree. Compare against the clean base (`git stash`) before
    blaming your diff.
- Run long commands through `/sase_monitor` (`TESTING`/`TESTED` for verification).
  Before you wrap any polling command in a monitor, run it once by hand and confirm it
  prints a parsed state.
- **Phase workers create no beads.** Record follow-ups as `PROPOSED FOLLOW-UP:` notes on
  your own phase bead. In your close note, name every existing task bead your evidence
  resolves.
- **Never leave a phase bead open.** The land agent waits on every phase bead. Record
  anything left over, or anything that reproduces on the clean base, as a note, and
  close. Run `sase bead epic-symbols <own-id>` before
  `sase bead close <own-id> --note '<evidence>'`.
- Do not edit `sase-1jz` or `sase-1ko` beads. Landing reconciles `sase-1jz`.

## Phase gate-reds

1. **Stale private import.** `tests/instructions/test_scoreboard_conformance.py:227`
   imports `_RunManifest` from `sase.instructions.manifests`. `e79a89d755` renamed that
   class to `RunManifest`, and nothing in the test uses the name. Delete the import.
   This failure is red on Master Gate shard 6 and on every Full CI test lane.
2. **Lazy-export monkeypatch pollution** (Master Gate shard 8, three
   `test_cli_bare_git.py` tests).
   - **How the export gets cached.** `sase.workspace_provider` resolves lazy exports
     through `src/sase/_lazy_exports.py`. On first access, that code stores the resolved
     value in the package globals (`module_globals[name] = value`).
   - **How the fake gets stuck.** `patch_git_metadata`, `patch_spy_metadata`, and
     `patch_no_workspace_metadata` (`tests/_workspace_provider_helpers.py:69-98`) patch
     `_registry.get_all_workflow_metadata` first, and only then the package attribute.
     On an xdist worker that never resolved the package attribute:
     1. monkeypatch's read of the "original" goes through the lazy `__getattr__`.
     2. That lookup resolves to the fake that was just installed, and caches it.
     3. Teardown then "restores" the fake permanently.
   - **Why the reset doesn't help.** `_use_real_provider_registry` in
     `test_cli_bare_git.py` resets only `_registry._manager` and the caches, so later
     tests on that worker still call the fake and get `()`.
   - **Reproduce:**
     `pytest tests/history/test_prompt_canonical_text.py tests/project_onboarding/test_cli_bare_git.py`
     gives 3 failures.
   - **Fix:** in all three helpers, resolve or patch the package attribute before
     `_registry`, so that monkeypatch records the real function. Use one shared private
     helper so the three cannot drift apart.
   - Add a regression test: delete the package global, apply and undo a helper, then
     assert
     `vars(sase.workspace_provider)["get_all_workflow_metadata"] is _registry.get_all_workflow_metadata`.
   - Correct `_use_real_provider_registry`'s docstring, which blames the wrong cache.
   - Grep for the same ordering bug wherever tests patch an implementation module and
     then its lazy package facade, and fix each instance.
3. **Archive-column flake** (timeboxed). In the 3.13 lane of 38530013190,
   `tests/ace/tui/test_agent_archive_column_app.py::test_archive_replaces_the_inbox_column_and_keys_navigate_it`
   timed out in `wait_for` at line 163, after `l` re-opens the Today banner.
   - Make that wait's timeout message include the archive state: selected key, focused
     section, folded banners, and the controller's generation, refreshing, and status
     values.
   - Stress the test under load (repeated `-n 8` runs) and fix the race if you can
     identify it. Otherwise, record the hypothesis and output as `PROPOSED FOLLOW-UP:`.
4. **Re-inventory and fix every other deterministic red** on the newest Master Gate and
   Full CI runs, newer than `8847d07a08`.
   - If a red comes from an in-progress `sase-1ko` commit and that commit makes the
     intended behavior clear (for example, `test_delivery_plan.py:146` after
     `sase-1ko.7`), fix the test's expectation and cite the commit.
   - Otherwise, record a `PROPOSED FOLLOW-UP:` naming the owning phase.
5. **Proof:**
   - Run each repaired node 20 times serially, then its file 3 times under `-n 8`.
   - Run the bare-git reproducer pair above until it is green.
   - Run `sase tool run check`.
   - In your close note, record the pass counts and the newest Master Gate run and SHA
     you checked.

## Phase visual-lane

Full CI 38530013190's `visual-test` had 9 failures and 1356 passes, with
`status=failed`. 38494718936 failed the same 9 plus `spread_deck_sticky`. All three
classes below reproduce locally with CI's exact `last_frame_digest` values, so the stale
local core is not the cause. Bisection points to `bda93bab64` (classes A and B,
sase-1kj.6) and `6da4076b8e` (class C).

1. **Class A: `agent_count` stays 0** (4 tests in
   `test_ace_png_snapshots_agents_retry_e2e.py`).
   - **What happens:** `patch_startup_loaders(use_real_agent_loader=True)`
     (`tests/ace/tui/visual/_ace_png_snapshot_startup.py:87`) leaves the disk loader
     real. But `src/sase/ace/testing/_startup.py:317-323` still swaps in
     `_run_fast_agent_startup`, which marks the first Agents load done with an empty
     roster.
   - **Why it used to pass:** the tab switch's reload rescued it through
     `_dirty_agents`. `bda93bab64` and `3cff3bea4f` now clear `_dirty_agents` in
     `_start_startup_agents_surface`
     (`src/sase/ace/tui/actions/_startup_loads_core.py:226`), so nothing reloads.
   - **Fix in the harness:** make the real-loader opt-in reach `AcePage`'s fast-startup
     override, so the real startup load reads the seeded agents. The smaller alternative
     is to have `_finish_fast_agent_startup` leave `_dirty_agents` set, since it read
     nothing.
   - Keep the production clear and the existing `expect_state("agent_count", N)` waits.
   - The current goldens already match once the agents load.
2. **Class B: machine tabs never render** (3 `test_ace_png_snapshots_agent_tab_strip.py`
   tests: `machine_mode`, `named_vs_machine_alias`, `stale_host`).
   - **What happens:** the tests inject remote-origin agents through the local disk
     loader. The tab switch's `_refilter_agents` then rebuilds rows through
     `_local_agents_from_mixed`
     (`src/sase/ace/tui/actions/agents/_fleet_projection.py:245`), which strips remote
     rows.
   - **Fix:** seed the remote rows through the fleet seam, the way
     `test_ace_png_snapshots_agents_fleet.py:77-107` does. Then wait until the tab
     catalog contains the machine tab keys before checking SVG sentinels.
   - Reuse the same `Agent` objects so the goldens stay valid.
3. **Class C: stale Commits wait text** (2 tests in `test_ace_png_snapshots_commits.py`,
   `:394` and `:477`).
   - **What happens:** `6da4076b8e` rewrote the Commits description
     (`src/sase/ace/tui/_artifact_tab_descriptions.py:36`), so the wait for "Commits
     landed through SASE" can never succeed.
   - **Fix:** wait on the pane's description as derived from the descriptor source, not
     on hard-coded copy.
   - Regenerate both goldens; they still show the old text and the singular labels.
     Inspect them, and cite `6da4076b8e`.
4. **Best effort:** `spread_deck_sticky` (`sase-1ii`) failed only in the loaded run.
   Look for a semantic-state wait it is missing. If you can't find one, record a
   hypothesis.
5. **Full proof.** Run the full `just fix-tui-screenshots --check`, with `TZ=UTC`
   exported for the whole command, through `/sase_monitor`. It must report `clean`, with
   `created=0 updated=0 stale=0` and no failed tests.
   - Classify any drift.
     - If an intended commit caused it, regenerate with a targeted
       `just fix-tui-screenshots -- <files>` and cite the commit.
     - If the drift comes from nondeterminism, fix the cause first.
   - Every update run must end `applied` or `clean`, never `partial`.
   - Then run each repaired test's targeted check 5 times, and run
     `sase tool run check`.

## Phase release-watch-tool

Add `tools/release_watch` as an extensionless, stdlib-only Python 3 script, following
`tools/last_workflow_set_status`. Do not import sase, so a broken venv or core binding
cannot break the watcher. If the file outgrows the line limit, put helpers in a sibling
`tools/_release_watch_*.py`. Add tests under `tests/test_release_watch_*.py`.

Its CLI (adjust flag names if you must, but keep their meanings):
`--repo sase-org/sase --version 0.18.0 --fixed-sha SHA --deadline 12h --poll 5m --max-red-full-ci 2 [--dry-run] [--once] [--verdict-file PATH] [--close-bead ID]`.
`--verdict-file` defaults to `$SASE_MONITOR_DIAGNOSTICS_DIR/release_verdict.md` when
that is set, and to a fresh temp directory otherwise. Print the path either way.

**Signals read on each poll.** Each `gh` or HTTP call gets its own timeout.

1. Whether PyPI's `https://pypi.org/pypi/sase/json` releases include `--version`.
2. The release PR: the single PR whose head branch starts `release-please--` and whose
   title carries the version. Read its state, `mergedAt`, `mergeStateStatus`, and each
   rollup check's name and conclusion, calling out `release-core-floor-smoke`.
3. Full CI (`full.yml`), from a per-workflow query on master. Read the in-flight runs,
   and the completed runs whose head contains `--fixed-sha`. Containment means
   `repos/{repo}/compare/{fixed}...{sha}` returns `ahead` or `identical`. For each run,
   record its conclusion, `updated_at`, and failed job names.
4. Master Gate (`master-gate.yml`) at the current master HEAD.
5. Publish (`publish.yml`): the newest runs, which are in flight, `created_at`,
   conclusion, and the `build` and `publish` job conclusions.
6. PyPI `sase-core-rs`: the newest version and its upload time.
7. Best effort only: ci_watch's latest `release_reason` for the repo, from its newest
   `~/.sase/axe/lumberjacks/ci_watch/chops/ci_watch/runs/*.result.decisions.json`. If it
   is missing, report `unknown`, never an error.

Print one line per poll: a UTC time, a `stage=`, one field for each signal, and
`action=`.

**Fail fast.**

- A required read (1–6) that errors or returns data that does not parse counts as a
  failure for that signal. Three consecutive failures of the same signal end the watch
  with `WATCHER_BROKEN`.
- Bad arguments, `gh` missing or unauthenticated, or an unknown workflow end the watch
  at once with `WATCHER_BROKEN`.
- Never treat a failed read as "nothing changed".

**Actions.** These run in live mode only. Under `--dry-run` or `--once`, print `would …`
instead. Every action is first deduplicated against GitHub's own runs, and then held
back by a cooldown.

- **Full CI.** Dispatch `gh workflow run full.yml -R <repo> --ref master` only when no
  Full CI run is in flight (`queued`, `in_progress`, `waiting`, `pending`, `requested`)
  and one of these holds:
  - no Full CI run exists yet on code containing `--fixed-sha`;
  - the newest such run completed red, and the red budget remains;
  - the newest such run is green but more than 4h old, and the PR is not merged (this
    keeps ci_watch's 6h window open).

  Cooldown: 20 minutes.

- **Floor ratchet.** Dispatch `publish.yml -f publish_existing=false` when
  `release-core-floor-smoke` is failing on the PR head, and no Publish run has been
  created since the newest `sase-core-rs` upload. Cooldown: 30 minutes.
- **Release.** Once the PR is merged and no Publish run has been created since
  `mergedAt`, dispatch the same Publish command. Cooldown: 20 minutes.
- Never merge, push, edit files, or dispatch any sase-core workflow.

**Verdicts.** Each has a distinct exit code.

- `RELEASED` (0): PyPI lists the version, and a fresh temporary venv outside the repo
  installs it. In that venv:
  - `uv venv` plus `uv pip install --refresh sase==<v>` (or the pip equivalent)
    succeeds;
  - `sase --version` prints the version;
  - the resolved `sase-core-rs` version is recorded.

  Retry the install for up to 20 minutes to absorb PyPI's CDN lag. The `--deadline` does
  not cut short an install proof that has already started.

- `FULL_CI_RED`: either a fixed-code run failed a deterministic job, or
  `--max-red-full-ci` fixed-code runs completed red. The deterministic jobs are lint,
  and visual-test when it reports golden drift rather than test failures. Include each
  run's ID, SHA, failed jobs, and a bounded `FAILED` excerpt.
- `PUBLISH_FAILED`: a Publish run created after the merge concluded with failure.
- `CORE_FLOOR_BLOCKED`: the ratchet had its chance and the floor check still fails. That
  is, `release-core-floor-smoke` failed on a PR head commit created after a Publish run
  that itself started after the newest PyPI `sase-core-rs` upload. A new sase-core
  release is needed.
- `RELEASE_PR_GONE`: no open or merged release PR carries the version.
- `INSTALL_FAILED`: the fresh-venv install never succeeded within its retry window.
- `DEADLINE`: the deadline passed. Include the last state line and the condition still
  blocking, such as ci_watch's reason.
- `WATCHER_BROKEN`: includes uncaught exceptions, with a short traceback summary.

The verdict file's first line is `RELEASE CONFIRMED` or
`RELEASE NOT CONFIRMED: <VERDICT>`. After it comes an evidence block: run IDs and URLs,
SHAs, times, the actions taken, and the last state line. Print the same block, and the
verdict file's path, as the watcher's final stdout lines, so a tailed monitor log always
ends with them. A signal that arrives before any verdict yields `DEADLINE` with the last
state.

**`--close-bead ID`.**

- On every exit path — a verdict, the deadline, an exception, or SIGTERM, SIGINT, or
  SIGHUP — write the verdict file first, then close the bead:
  - `RELEASED` runs `sase bead close ID --note @<verdict-file>`;
  - every other verdict adds `-R canceled -r 'release not confirmed: <VERDICT>'`.
- Retry the close up to 3 times, 30 seconds apart. Print the outcome, then exit with the
  verdict's code.
- `--once` and `--dry-run` never close a bead. They print the exact close argv instead.

**Tests.** Inject the `gh`, HTTP, clock, signal, and subprocess boundaries, and cover:

- every verdict;
- each dispatch rule: in flight (every status), due, cooldown, red budget, stale-green
  refresh, post-merge Publish, and floor ratchet;
- three consecutive malformed reads leading to `WATCHER_BROKEN`;
- fixed-SHA containment;
- the deadline;
- SIGTERM leading to a close;
- close retries;
- `--once`, which prints one fully parsed line and neither dispatches nor closes.

Add a short usage paragraph wherever the docs describe the release flow; find it with
`grep -rn release-please docs/`. Run `sase tool run check`.

## Phase release-wait

This phase makes **no file edits and no commits**, and `git status` must stay clean. Its
whole job is to start the watch correctly. The watch is what closes the bead.

1. **Check the prerequisites.**
   - Read this bead.
   - `git fetch`, and confirm that `origin/master` contains the commits from gate-reds,
     visual-lane, and release-watch-tool. Their bead IDs appear in the commit messages.
   - If this workspace is behind, fast-forward it (`git pull --ff-only` on a clean
     tree).
   - Set `FIXED_SHA` to the `origin/master` SHA.
   - Read the version from PR #299's title. Expect 0.18.0; if it differs, use the PR's
     version and note why.
2. **Core release.**

   > [!decision] core_release = dispatch
   >
   > Run `sase repo open sase-core`. Then check two things:
   >
   > - whether the newest PyPI `sase-core-rs` already contains sase's pinned revision
   >   (from `sase-core-revision.txt`);
   > - whether the open `release-plz-*` release PR is CLEAN and its head contains that
   >   revision.
   >
   > If PyPI does not have the revision yet and the PR is ready, run
   > `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`
   > once. If the pin is ahead of the release PR, still dispatch, and record that as a
   > note; the watcher will report `CORE_FLOOR_BLOCKED` if it matters.

   > [!decision] core_release = daily_cut
   >
   > Dispatch nothing. The watcher waits for the daily release-plz cut and then ratchets
   > the floor.

3. **Clock (optional, non-blocking).** If the user's primary bugyi-chops checkout
   (`~/projects/github/bbugyi200/bugyi-chops`) does not yet contain `5828753`, create a
   plain `/sase_gate` with no `turn` block. It should propose
   `git -C ~/projects/github/bbugyi200/bugyi-chops pull --ff-only` so ci_watch's
   two-hour Full CI dispatcher goes live. Do not wait for it. The watcher dispatches
   Full CI itself.
4. **Preflight inline.**
   - `sase bead epic-symbols <own-id>` must print nothing.
   - Then run:
     `tools/release_watch --repo sase-org/sase --version <v> --fixed-sha <FIXED_SHA> --once --dry-run --close-bead <own-id>`.
     It must exit 0, print one state line with signals 1–6 all parsed, and print the
     close argv.
   - If any check fails, do not fix anything. Close the bead with `-R canceled`,
     `-r 'release not confirmed: WATCHER_BROKEN (preflight)'`, and the output as the
     note. Then `/sase_final`.
5. **Start the one monitor.** Pass the command as a single quoted string:

   ```bash
   sase monitor start \
     --reason 'Watch sase v<v> through Full CI, the ci_watch merge, Publish, and PyPI; the watcher closes <own-id>' \
     --timeout 13h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -o tail --tail-lines 150 \
     --next "$NEXT" \
     -- 'timeout -k 5m 760m tools/release_watch --repo sase-org/sase --version <v> --fixed-sha <FIXED_SHA> --deadline 12h --close-bead <own-id>'
   ```

   GNU `timeout` takes a single-unit duration, so 12h40m is written `760m`. The nested
   deadlines are: the watcher's 12h, plus at most 20 minutes of install proof, inside
   the outer 760m, inside the monitor's 13h.

   If `sase monitor start` exits non-zero, nothing was handed off. Retry once. If it
   fails again, close the bead with `-R canceled`,
   `-r 'release not confirmed: WATCHER_NOT_STARTED'`, and the error as the note. Then
   `/sase_final`.

   **Follow-up text.** Put it in `NEXT` with a quoted heredoc
   (`NEXT=$(cat <<'EOF' … EOF)`) so its quotes survive. Keep it free of `#` or `%`
   directives:

   > You are the follow-up for release-wait <own-id>. Do not edit files, start a
   > monitor, or reopen <own-id>. Read the verdict block at the end of the monitor
   > output, or the verdict file it names. Run `sase bead read <own-id>`. If the bead is
   > not closed, close it now:
   >
   > - on RELEASE CONFIRMED, run
   >   `sase bead close <own-id> --note '<verdict and evidence>'`;
   > - otherwise, run
   >   `sase bead close <own-id> -R canceled -r 'release not confirmed: <VERDICT>' --note '<evidence>'`.
   >
   > Then finish with /sase_final. Closing is your whole job: the epic's land agent
   > waits on this bead and handles everything that is left.

## Landing

**Confirm the release before landing.** Do this before anything else.

- PyPI's JSON lists `sase` 0.18.0.
- `gh release view v0.18.0 -R sase-org/sase` succeeds.
- The Publish run's `build` and `publish` jobs succeeded.
- In a fresh temporary venv, an unpinned `pip install sase` resolves 0.18.0, and
  `sase --version` prints `0.18.0`.
- Record the Full CI run and SHA, the Master Gate run, the #299 merge commit, the
  Publish run, and the resolved `sase-core-rs` version.

**If the release is not confirmed, do not close the epic.** Read release-wait's verdict
and evidence, and finish the release:

- **No code change needed.** This covers a deadline expired because of Master Gate
  starvation, a single flake, Publish or core cron lag, a `lost` monitor, or
  `CORE_FLOOR_BLOCKED` that a core release fixes (dispatch it under `core_release`).
  - Run `tools/release_watch`, without `--close-bead`, through your own monitor.
  - Give it a `--next` that resumes this landing at "Confirm the release".
- **Code or config change needed.** Author a child epic whose parent is this epic. Its
  phases:
  - fix phase(s);
  - a release-wait phase that follows this plan's Phase release-wait contract verbatim;
  - its land agent, which resumes this landing.

  Never put the wait in the same phase as the fixes.

**Once the release is confirmed:**

1. Close the task beads the evidence resolves, each with that evidence:
   - `sase-16p`, by the release.
   - `sase-1gh` and `sase-19d`: the sudo canary fixes from `sase-1kl.1`, if the green
     runs show no recurrence.
   - `sase-1kg`, if `tests/ace/tui/test_proc_producer_inventory.py` passes on the tip.
   - `sase-1ii`, if `spread_deck_sticky` and `left_right_search_committed` passed in the
     green Full CI.
   - Any other bead the phases name.
2. Triage, through `/sase_new_task`:
   - this epic's `PROPOSED FOLLOW-UP:` notes;
   - the notes `sase-1kl` left untriaged when it was canceled (`sase-1kl.1` #1,
     `sase-1kl.2` #1–#3, `sase-1kl.3` #1, and `sase-1kl.4` #5–#7). Its bare-git note #5
     is fixed by gate-reds. Its silent-monitor note #7 is only partly addressed by
     `tools/release_watch`; the general monitor hardening remains open.
3. Release `sase-1jz`:

   > [!decision] stale_epic_1jz = notes
   >
   > - Close `sase-1jz.3` with a note: `4b99e45907` landed its deck-race fixes, and this
   >   epic's green Full CI passed every visual test.
   > - Add notes on `sase-1jz.4` and `sase-1jz.5` pointing to this epic's rebaselines
   >   and its green Full CI and Master Gate runs, so their waiting agents can verify
   >   and close quickly.

   > [!decision] stale_epic_1jz = retire
   >
   > 1. `sase agent kill` the WAITING agents `sase-1jz.4`, `sase-1jz.5`, and
   >    `sase-1jz.land`.
   > 2. Close the epic with
   >    `sase bead close sase-1jz --force -R superseded -r 'superseded by <this epic>'`.
   > 3. Leave a note citing this epic's evidence.

4. Report whether ci_watch's heavy dispatcher is live yet, meaning whether the
   bugyi-chops pull happened.
