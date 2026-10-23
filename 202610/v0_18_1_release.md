---
tier: epic
title: Green master's release gates and ship sase v0.18.1 to PyPI through ci_watch
goal: 'Master Gate is green at master HEAD, a fresh Full CI run on master is green,
  ci_watch merges the release-please PR for 0.18.1, Publish uploads sase 0.18.1 to
  PyPI, a clean install of it works, and the final lander records that evidence as
  an artifact.

  '
decisions:
  publish_dispatch:
    ask: Should the watch dispatch Publish to refresh the release PR and to publish
      right after ci_watch merges?
    default: true
    why: Publish's 3-hourly cron has been firing 4-7h apart, which can outlast the
      watch
    answer: true
  gate_rerun:
    ask: Should the watcher rerun a failed Master Gate at master HEAD once per run
      to absorb a flake?
    default: true
    why: ci_watch needs green at the exact HEAD; one flake otherwise stalls until
      the next push
    answer: true
phases:
- id: core-merge
  title: Repair sase-core's release-plz merge filter
  depends_on: []
  size: small
  description: 'core-merge: fix the invalid jq escape in sase-core''s release_plz_merge.py
    PR filter and pin it with a jq-backed regression test.'
- id: core-pin
  title: Cut sase-core-rs 0.42.2 and raise sase's exact core pin
  depends_on:
  - core-merge
  size: medium
  description: 'core-pin: dispatch the guarded sase-core release cut, wait for a complete
    sase-core-rs 0.42.2 on PyPI, then ratchet sase''s ==0.42.0 pin and prove the floor
    smoke locally.'
- id: ci-workflows
  title: Restore Full CI's build-core job and authenticate release metadata sync
  depends_on: []
  size: small
  description: 'ci-workflows: move build-core''s setup-uv step after the sase-core
    checkouts, give Publish''s ratchet step a GitHub token, and pin both with workflow
    tests.'
- id: modal-sizing
  title: Make the custom gate modal sizing tests deterministic
  depends_on: []
  size: small
  description: 'modal-sizing: replace pause-then-assert resize checks in test_custom_gate_modal_sizing.py
    with waits on the settled layout so Master Gate shard 8 goes green.'
- id: visual-lane
  title: Make the visual screenshot check clean on UTC runners
  depends_on:
  - modal-sizing
  size: medium
  description: 'visual-lane: inventory golden drift with a full TZ=UTC screenshot
    check, fix nondeterminism, regenerate only intended goldens, and prove the check
    clean.'
- id: full-ci
  title: Run Full CI on the repaired tip and fix every other red
  depends_on:
  - ci-workflows
  size: medium
  description: 'full-ci: get a Full CI run on a tip containing the build-core fix,
    triage every non-visual red job plus any other Master Gate red, and fix what is
    deterministic.'
- id: watch
  title: Watch v0.18.1 through ci_watch, Publish, and PyPI with a self-closing monitor
  depends_on:
  - core-pin
  - ci-workflows
  - modal-sizing
  - visual-lane
  - full-ci
  size: small
  description: 'watch: make no repo edits; verify prerequisites, then start one no-follow-up
    monitor whose watcher script always closes this bead and exits 0.'
proposed_by: bbugyi200.athena.137
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-23 11:30:03
status: wip
bead_id: sase-1nn
---

- **PROMPT:** [prompts/202610/v0_18_1_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_release.md)
- **BEAD:** [sase-1nn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nn/README.md)

# Green master's release gates and ship sase v0.18.1

**Target.** The target is sase **0.18.1**. The request also mentions "v0.18.0", but
v0.18.0 has been on PyPI and GitHub Releases since 2026-10-19. Release-please PR
**#321** (`chore(master): release 0.18.1`, branch `release-please--branches--master`) is
the release that is waiting. Every "observe the release" step in this epic and in any
nested epic means 0.18.1.

**How the release happens.** `ci_watch` is an AXE job on athena. Its config is in
chezmoi `home/dot_config/sase/sase_athena.yml`, and its code is in bugyi-chops. Every 5
minutes it merges a green release-please PR, but only when all of these hold:

1. **Master Gate** is green at the exact master HEAD (`gating_workflows`).
2. The newest completed **Full CI** run on master is green and at most 6h old
   (`heavy_max_age_hours: 6`). `ci_watch` also dispatches Full CI itself when the newest
   run is more than 2h old and none is in flight (`heavy_dispatch_every_hours: 2`).
3. Every check on the release PR is green (SKIPPED counts as green).

After the merge, the next `Publish` run (`publish.yml`) does the rest. Release-please
creates the tag and GitHub release. Then come `build`, `install-smoke`,
`install-smoke-core-floor`, and `publish` (PyPI, `skip-existing`). Publish runs on a
3-hourly cron or on `gh workflow run publish.yml -f publish_existing=false`. The same
Publish run's `sync-release-metadata` job also reconciles the release branch's
`sase-core-rs` pin. `tools/release_watch` (the Rust steward) only observes this flow.
Its scheduled workflow does not drive it. Never use `release_watch --live` and never
dispatch `release.yml` in this epic.

## Evidence (observed 2026-10-23 at master `7552ebe590`)

Three of the four blockers are independent, and each one alone prevents the merge:

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Evidence                                                                                                                                    | Phase                 |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 1   | **Full CI red since 2026-10-20.** Every run fails in `full / build-core / build-core`: `##[error]No file matched to [sase-core/**/pyproject.toml]`. `9edbacd4ef` (feat(ci): supply verified published core artifacts to setup-sase) moved `astral-sh/setup-uv@v4` (`cache-dependency-glob: "sase-core/**/pyproject.toml"`) ahead of the `Check out Rust core` steps in `.github/workflows/build-core.yml`. Every downstream job is skipped, so Full CI's test and visual lanes have reported nothing since then. `telemetry.yml` uses the same reusable workflow.                                                                                                                   | The last green Full CI was run 39260047671 at `41b0a73b06` (2026-10-19). The most recent red run is 39961665089.                            | ci-workflows, full-ci |
| 2   | **Master Gate red since `50ef6ba498`** (feat(ace): widen custom gate modal layouts). Only `test (8)` fails, on `tests/ace/tui/test_custom_gate_modal_sizing.py`: `test_compact_width_scales_and_caps_at_120` (`assert 120 == 90`), `test_compact_scroll_refits_after_resize` (`39 > 39`), `test_custom_preview_rail_grows_and_stacks_at_narrow_breakpoint` (`66 == 42`), and sometimes `test_compact_scroll_fits_and_footer_stays_inside_shell` (`29 > 30`). The file passes 6/6 locally in isolation. Each test calls `pilot.resize_terminal(...)`, then a single `pilot.pause()`, then asserts geometry, so on a loaded runner the reflow has not landed yet.                     | Every Master Gate run from 39604800734 through 39973146562 (HEAD `7552ebe590`) fails this way, with the same tests in shard 8.              | modal-sizing          |
| 3   | **Release PR #321 `release-core-floor-smoke` red.** Master pins `sase-core-rs==0.42.0`, but master now calls 8 bindings that 0.42.0 lacks: `classify_publication_error`, `classify_replay_paths`, `decide_publication_failure_accounting`, `decide_replay_apply`, `decide_replay_byte_divergence`, `decide_replay_hood_merge`, `repair_session_manifest_entry`, `select_publication_drain`. They exist at the source pin `sase-core-revision.txt` = `1503d604844a` (sase-core master HEAD), which no tag contains yet. sase-core release-plz PR **#333** (`chore: release v0.42.2`, CLEAN, checks green) contains exactly those commits. 0.42.1 is the newest sase-core-rs on PyPI. | PR #321 floor-smoke job 120246968965 reports "sase_core_rs 0.42.0 is missing 8 of 874 required binding(s)".                                 | core-merge, core-pin  |
| 4   | **The core cut cannot merge.** The release steward reports `core_cut_red` (`CORE_CUT_RED check=release-plz`). sase-core's `Merge release PR` job runs `.github/scripts/release_plz_merge.py`, whose `PR_FILTER` passes `test("^chore: release v[0-9]+\.[0-9]+\.[0-9]+$")` to `gh --jq`. `\.` is an invalid escape inside a jq string literal: `jq` reports `Invalid escape … (while parsing '"\."')`, and gh's gojq reports `invalid escape sequence "\." in string literal`. The bug arrived with sase-core `3f0920f8` (2026-10-19). sase-core-rs 0.41.0, 0.42.0, and 0.42.1 were all merged by hand before that.                                                                  | sase-core run 39956404021 (job 120423633150) shows the error. `tools/release_watch --once --json` currently returns primary `core_cut_red`. | core-merge            |

Related symptom: every scheduled `Publish` run since 2026-10-20 is red in
`sync-release-metadata`, failing with
`[ratchet_core_pin] unreadable GitHub API repos/sase-org/sase-core/contents/.github/workflows/release-plz.yml?ref=v0.34.39: HTTP 403`.
The source pin is in no released core yet, so `tools/_release_core.py ensure` walks
every PyPI core version, and it reads each tag's `release-plz.yml` through the GitHub
API. That step sets no `GH_TOKEN`. `gh api` therefore fails and the urllib fallback runs
unauthenticated, until the 60-per-hour limit returns 403. This resolves itself once
0.42.2 exists, because the first version checked contains the pin. ci-workflows still
adds the token.

Hidden risk: Full CI's test and visual lanes have not run since 2026-10-20 (about 60
commits). The last runs that executed them were 39338279208 at `03c7231e11`, where
visual-test drifted with `updated=16`, and 39294492670, where all three test lanes were
red. Expect new reds. The full-ci and visual-lane phases own them.

Not blockers: `CI Telemetry` is red only because it reuses build-core. Open PRs #319
(stale core-pin ratchet) and #320 (shard timings) do not gate anything.

## Rules for every phase

- Read `lint_and_test.md` and `sase_beads.md` through `/sase_memory_read`. For TUI or
  visual work, also read `tui.md`. For any `src/sase/ace/` change, also read `ace.md`.
  Read `symvision.md` if a symvision gate fails.
- Fetch and re-inventory on the current master tip before editing. Other work lands
  continuously, so the Evidence above is a starting point, not a whitelist.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state the assertion is about (`sase.ace.testing.wait.wait_for` or the file's
  existing helper), never on a fixed `pilot.pause(<seconds>)`. If the product really
  settles late or inconsistently, fix the product.
- Never copy CI candidate PNGs into the tree. Regenerate a golden only after its test is
  deterministic.
- Shared domain behavior belongs in sase-core (`rust_core_backend_boundary`). None of
  these fixes is expected to need a Rust change. Work in sase-core only through
  `sase repo open sase-core`, and run its own `sase tool run check` there.
- Verify with `just fix`, then `sase tool run check`, in every checkout you changed. Do
  not run `just check-full`: Full CI on GitHub is this epic's exhaustive signal. The CI
  `lint` job enforces `toobig`, so do not grow a file past its line limit.
- Hand every long command to `/sase_monitor`: verification (`-p verify`), CI waits, PyPI
  waits. Never end a turn waiting.
- **No hand merges, no new PRs.** #321 merges only through `ci_watch`, and sase-core
  #333 merges only through its own `Merge release PR` job. Dispatching workflows
  (`gh workflow run`) and rerunning failed jobs (`gh run rerun --failed`) are allowed
  where a phase says so.
- Phase workers create no beads. Record `PROPOSED FOLLOW-UP: <summary — detail>` notes
  on your own phase bead. Name every existing task bead your evidence resolves, so the
  land agent can close it. Before closing, run `sase bead epic-symbols <own-phase-id>`
  (it must list nothing). Then close with `sase bead close <id> --note '<evidence>'` and
  land through `/sase_final`. If you cannot finish, still close the bead with
  `-R canceled` and the evidence, so the land agent always runs.

## Phase core-merge

Work in sase-core (`sase repo open sase-core -r '<why>'`).

1. In `.github/scripts/release_plz_merge.py`, rewrite `PR_FILTER`'s title regex so the
   jq program contains no `\.` escape. Use character classes:
   `^chore: release v[0-9]+[.][0-9]+[.][0-9]+$`. That form is immune to Python, shell,
   and jq escaping. Keep the head-ref and body selectors unchanged, and check every
   other `--jq` string in the file for the same mistake.
2. Add a regression test in `.github/scripts/test_release_plz_merge.py`. It feeds
   `PR_FILTER` to a real `jq` binary (`shutil.which("jq")`) over a JSON PR list that
   includes:
   - a matching release-plz PR;
   - a title with a non-dot separator (`v0x42x2`), which must not match;
   - a non-release-plz head ref;
   - a body without the release-plz marker.

   Assert that it selects exactly the matching PR and that an empty list yields empty
   output. If `jq` is absent, skip with a reason, so the unit lane still runs elsewhere.
   Confirm that sase-core CI runs this file and that its runner has `jq`. ubuntu-latest
   ships jq; confirm the job's runner.

3. Look at the merge job's last red log (sase-core run 39956404021) and the code path
   after `find_release_pr`. If another defect would stop the merge on its first real run
   (check polling, rerun-once, the merge call), fix it here under the same rules.
4. Run `sase tool run check` in sase-core.

## Phase core-pin

Goal: a complete `sase-core-rs` 0.42.2 is on PyPI, and sase master pins it exactly.

1. Confirm the core-merge commit is on sase-core `origin/master`
   (`git -C <sase-core> log origin/master --grep 'SASE_BEAD=\[<core-merge-id>\]'`).
   Re-fetch for up to about 5 minutes if a host push is still landing. If the commit is
   missing, or core-merge closed `canceled`, close this phase `-R canceled` with that
   evidence and run `/sase_final`.
2. If PyPI already serves a sase-core-rs version whose tag contains the source pin in
   `sase-core-revision.txt`, with its full file set, skip to step 5. The full file set
   is the `EXPECTED_DIST_SUFFIXES` in that tag's `release-plz.yml`: manylinux_2_28
   x86_64 and aarch64, universal2, win_amd64, and the sdist.
3. Confirm the open release-plz PR. At planning time it is #333
   (`chore: release v0.42.2`). It must be open, green, and contain `1503d604`. If
   release-plz has since retargeted it, use the version it now names everywhere below.
   Then dispatch the guarded cut, which is the same argv the steward's
   `dispatch_core_cut` uses:
   `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
   The `Merge release PR` job waits for checks (up to 40 minutes) and merges. The
   resulting push runs `Release-plz`, which tags `v0.42.2`, builds the wheel matrix, and
   publishes.
4. Wait through `/sase_monitor`, with `--next` and a timeout of about 3h. Poll until
   `https://pypi.org/pypi/sase-core-rs/0.42.2/json` exists and lists every expected
   suffix, or until a release-plz run for that cut fails. In the follow-up:
   - If the merge job failed on a small, clear script or workflow bug, fix it in
     sase-core under the core-merge rules, land it, and record a `PROPOSED FOLLOW-UP`
     asking the land agent to re-dispatch the cut. You cannot re-dispatch on your own
     unlanded fix.
   - If a wheel leg failed (macOS has failed before), or the file set is incomplete,
     record the job IDs and logs as `PROPOSED FOLLOW-UP`.
   - In either case, close `-R canceled` with the evidence and run `/sase_final`. The
     watch phase will see the missing core and release the land agent quickly.
5. In sase, run `python tools/ratchet_core_pin` with the flags its `--help` documents
   for a local master ratchet, then `uv lock`. It must rewrite the pyproject dependency
   to `sase-core-rs==0.42.2` and refresh `uv.lock`. Keep `sase-core-revision.txt`
   unchanged unless a version check proves the source pin must move. If it must move,
   follow `docs/rust_backend.md` and `docs/contribute/rust-backend.md`. CI's
   source-built wheel is installed through the setup-sase overrides file, so the pin and
   the source-built version may legitimately differ.
6. Reproduce the floor smoke locally. Do it exactly as the `release-core-floor-smoke`
   job in `.github/workflows/ci.yml` does: a fresh `uv venv --python 3.12` outside the
   repo, `uv pip install -e '.[dev]' "sase-core-rs==0.42.2"`,
   `tools/check_sase_core_rs_bindings`, then `tools/validate_sase_core_rs`. It must
   report 0 missing bindings. Paste the output into your close note.
7. Run `just fix` and `sase tool run check`, then land.

## Phase ci-workflows

1. In `.github/workflows/build-core.yml`, move the `astral-sh/setup-uv@v4` step
   (`cache-dependency-glob: "sase-core/**/pyproject.toml"`) so it runs after **both**
   conditional sase-core checkouts (`Check out Rust core` and
   `Check out published Rust core tag`). It must still run before the first uv use:
   `uvx maturin` in source mode, and `uv run` in
   `Fetch and verify the published core wheel`. That restores the pre-`9edbacd4ef`
   ordering for source mode and keeps published mode working. Do not weaken the glob or
   set `ignore-nothing-to-cache`.
2. Extend `tests/test_github_actions_build_core.py`. Every setup-uv step whose
   `cache-dependency-glob` names a path under a checkout's `path:` must come after that
   checkout. Assert it for the source-mode and published-mode step sequences. The test
   must fail on today's `build-core.yml`.
3. In `.github/workflows/publish.yml`, give `sync-release-metadata`'s
   `Reconcile release metadata` step `GH_TOKEN: ${{ secrets.SASE_RELEASE_TOKEN }}`. That
   is the token the job already uses for its branch check and checkout.
   `tools/_release_core.py` reads tag workflows through `gh api` (or urllib with
   `GH_TOKEN`), so they stop running unauthenticated. Pin this env in the existing
   publish-workflow tests, or add one in the style of the other
   `tests/test_github_actions_*.py` files.
4. Check whether any other workflow runs `ratchet_core_pin` or `_release_core` reads
   without a token: `core-pin-ratchet.yml`, `release-steward.yml`. Fix only the
   identical omission.
5. Run `sase tool run check`.

## Phase modal-sizing

1. Reproduce under load. Run `tests/ace/tui/test_custom_gate_modal_sizing.py` together
   with the rest of `tests/ace/tui/` under `-n 8` several times. Isolated runs pass, and
   Master Gate fails it in shard 8 every time. Read `_custom_gate_modal_helpers.py` and
   the modal's resize and layout code in `src/sase/ace/tui/modals/custom_gate_modal.py`
   (and its sizing helpers). Learn exactly which refresh or callback applies the
   responsive width, scroll max-height, and rail breakpoint.
2. After each `resize_terminal`, wait for the semantic state the next assertion reads:
   the shell width for that breakpoint, the refit scroll height, or the rail width or
   stacking. Keep every asserted value exact. If the modal applies its layout through a
   path that can leave a stale geometry after a resize (for example, a deferred
   `call_after_refresh` chain or a missed `on_resize`), fix that in the modal and keep
   the test strict.
3. Audit the sibling custom-gate test files (`tests/ace/tui/test_custom_gate_*`) for the
   same resize-then-pause-then-assert pattern, and fix those too.
4. If a product change alters rendered output, regenerate only the custom-gate goldens
   it changes with a targeted `just fix-tui-screenshots -- <files>`. Inspect every
   update. visual-lane owns everything else.
5. Proof:
   - Run each repaired node 20 times serially.
   - Run the file 3 times under `-n 8` alongside `tests/ace/tui/`.
   - Run `sase tool run check`.

   Record the pass counts.

## Phase visual-lane

Goal: `just fix-tui-screenshots --check` is clean on GitHub's UTC runners. Visual-test
has not run in CI since 2026-10-20. Its last result was `updated=16` at `03c7231e11`,
including `agents_live_reply_*_partial_120x40`.

1. Rebase on current master, which includes modal-sizing. Run the full
   `just fix-tui-screenshots --check` through `/sase_monitor`, with `TZ=UTC` exported
   for the whole command. If a newer CI `visual-test` run exists by then (full-ci may
   trigger one), download its `ace-visual-artifacts` (`report/summary.md`,
   `manifest.json`, and `report/images/updated-*/{expected,actual,diff}.png`) and
   compare.
2. Classify every drifted golden:
   - **Intended.** Trace it to the landed commit that changed the UI (for example,
     `50ef6ba498` widened custom gate modal layouts).
   - **Race.** Turn counts, deck positions, loading placeholders, or a capture before
     settle.
   - **Host-dependent.** Clock, timezone, hostname, `/home/<user>`, terminal size, or
     fonts.

   Inspect the images, not just the counts.

3. Make every race or host-dependent capture deterministic first: wait for the settled
   state, or pin the input. Then regenerate.
   - Use targeted `just fix-tui-screenshots -- <files>` first.
   - Then run the full `just fix-tui-screenshots` through `/sase_monitor`. It must end
     `applied` or `clean`, never `partial`, with zero failed tests.
   - Accept only changes you traced to an intended commit or a deliberate fix.
4. Proof:
   - Run the full `just fix-tui-screenshots --check` again with `TZ=UTC`. It must report
     `clean` with `created=0 updated=0 stale=0` and no failed tests.
   - Run each repaired test's targeted check 5 times.
   - Run `sase tool run check`.

## Phase full-ci

Goal: a Full CI run on a tip that contains the build-core fix, with every non-visual red
triaged and every deterministic one fixed.

1. Confirm the ci-workflows commit is on `origin/master`; re-fetch for up to about 5
   minutes. If ci-workflows closed `canceled`, close this phase `-R canceled` with the
   evidence and run `/sase_final`.
2. Find a Full CI run on master whose `headSha` contains that commit
   (`gh run list -R sase-org/sase --workflow full.yml --branch master`). `ci_watch` may
   already have dispatched one. If none is in flight or done, dispatch it with
   `gh workflow run full.yml -R sase-org/sase --ref master`. Wait through
   `/sase_monitor`, with `--next` and a timeout of about 3h, for example on
   `gh run watch <id> -R sase-org/sase --interval 120`. A Full CI run takes about 2h.
3. Triage every job that did not succeed, using
   `gh run view --job <id> --log-failed -R sase-org/sase`, or
   `gh api repos/sase-org/sase/actions/jobs/<id>/logs`, stripping ANSI.
   - **Owned elsewhere:** custom gate modal sizing (modal-sizing), `visual-test` golden
     drift (visual-lane), and build-core or Publish wiring (ci-workflows). Note them; do
     not fix them here.
   - **Everything else is yours:** the `test (3.12|3.13|3.14)` lanes, `lint`,
     `docs-build`, `perf-floors`, `tutorial-tests`, and `ace-page-group-isolation`. Fix
     every deterministic failure.
   - For a flake, look for a pause-then-assert or wall-clock race and fix it if the
     cause is identifiable. Otherwise run `sase bead search` for an existing owner bead,
     and record the failing output plus a hypothesis as `PROPOSED FOLLOW-UP` naming that
     bead.
4. Also list Master Gate runs on master newer than `7552ebe590`, and treat any red other
   than the modal tests the same way.
5. Proof:
   - Run each repaired node 20 times serially.
   - Run its file 3 times under `-n 8`.
   - Run `sase tool run check`.

   You need not wait for a second Full CI: `ci_watch` re-dispatches every 2h, and the
   watch phase observes the result.

## Phase watch

This phase makes **no repo edits**. It ends by starting one monitor with **no
`--next`**, so no LLM turn runs after it and nothing can hang. The watcher script closes
this bead itself, and the monitored command always exits 0. A monitor member that
settles `completed` or `stopped` with no follow-up satisfies the land agent's
agent-session wait.

1. **Idempotent start.** `sase bead read` this bead. If it is already closed (a relaunch
   after recovery), run `/sase_final` and stop.
2. **Already released?** If `curl -fsS https://pypi.org/pypi/sase/0.18.1/json` succeeds,
   close this bead `--note` with that evidence and run `/sase_final`.
3. **Prerequisites.** Fetch. For each sibling phase, read its bead.
   - A phase closed `canceled` means a prerequisite failed. Close this bead
     `-R canceled -r "release watch not started: <phase> canceled"` with its close note,
     then run `/sase_final`.
   - Confirm each `done` sibling's commit is on `origin/master`
     (`git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`). A phase whose note
     says no commit was needed is fine. Re-fetch for up to about 5 minutes before
     declaring a commit missing.
   - Confirm master's `pyproject.toml` pins a `sase-core-rs` that is on PyPI, and that
     `tools/check_sase_core_rs_bindings` passes against it. Reuse core-pin's
     temporary-venv recipe outside the repo.
   - Confirm `gh pr view 321 -R sase-org/sase --json title,state` is still the open
     `chore(master): release 0.18.1`. If release-please retargeted the version or
     replaced the PR, close
     `-R canceled -r "release watch not started: release target changed"` with the
     evidence.

   Any failure closes this bead `-R canceled` with the evidence, followed by
   `/sase_final`. Do not repair code in this phase.

4. **Epic-symbol guard.** `sase bead epic-symbols <this-phase-id>` must list nothing.
   Otherwise, close `-R canceled -r "release watch not started: epic-symbol entries"`
   and run `/sase_final`.
5. **Refresh the release PR.**

   > [!decision] publish_dispatch
   >
   > If #321's head commit predates master's pin commit from core-pin, or its
   > `release-core-floor-smoke` check is red, run
   > `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=false`
   > once. Release-please then rebuilds the release branch from master, and
   > `sync-release-metadata` reconciles it. Do not wait for the result here; the watcher
   > observes it.

   > [!decision] publish_dispatch = no
   >
   > Skip this step, and delete the `gh workflow run publish.yml` line from the script
   > below. Publish's own cron refreshes the PR and publishes after the merge.

6. **Write the watcher outside the repo.** Use
   `W=/tmp/sase-release-watch-<this-phase-id>` and `mkdir -p "$W"`. Write `$W/watch.sh`
   with a quoted heredoc (`<<'EOF'`), so nothing expands at write time:

   ```sh
   #!/bin/sh
   # sase v0.18.1 release watcher: sh watch.sh BEAD LOG [preflight]
   # Never edits files. In watch mode it closes BEAD itself and always exits 0.
   BEAD=$1 LOG=$2 MODE=${3:-watch}
   REPO=sase-org/sase VERSION=0.18.1
   TITLE="chore(master): release $VERSION"
   unset GH_FORCE_TTY
   export NO_COLOR=1 CLICOLOR=0 CLICOLOR_FORCE=0 FORCE_COLOR=0
   START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   END=$(( $(date +%s) + 36000 ))
   outcome=DEADLINE detail= dispatched= reruns=
   say() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" | tee -a "$LOG"; }
   while :; do
     if curl -fsS -o /dev/null "https://pypi.org/pypi/sase/$VERSION/json" 2>/dev/null; then
       outcome=RELEASED; break
     fi
     merged=$(gh pr list -R "$REPO" --base master --state merged \
       --search "in:title \"release $VERSION\"" --json title,mergedAt \
       --jq "[.[] | select(.title == \"$TITLE\")][0].mergedAt // \"\"" 2>>"$LOG")
     pub=none
     if [ -n "$merged" ]; then
       pub=$(gh run list -R "$REPO" --workflow publish.yml --limit 10 \
         --json databaseId,createdAt,status,conclusion \
         --jq "[.[] | select(.createdAt > \"$merged\")] | sort_by(.createdAt) | last
               | if . == null then \"none\" else \"\(.databaseId):\(.status):\(.conclusion)\" end" 2>>"$LOG")
       if [ "$pub" = none ] && [ -z "$dispatched" ] && [ "$MODE" = watch ]; then
         gh workflow run publish.yml -R "$REPO" --ref master -f publish_existing=false >>"$LOG" 2>&1 && dispatched=yes
       fi
       case "$pub" in
         *:completed:failure|*:completed:cancelled|*:completed:timed_out|*:completed:startup_failure)
           outcome=PUBLISH_RED detail="Publish run $pub"; break ;;
       esac
     fi
     head=$(gh api "repos/$REPO/commits/master" --jq .sha 2>>"$LOG")
     gate=$(gh run list -R "$REPO" --workflow master-gate.yml --commit "$head" --limit 1 \
       --json databaseId,status,conclusion,attempt \
       --jq '.[0] | "\(.databaseId):\(.status):\(.conclusion):\(.attempt)"' 2>>"$LOG")
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
     say "state merged=${merged:-no} publish=$pub head=${head:-?} gate=${gate:-?} full_since_start=${full:-none}"
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

   > [!decision] gate_rerun = no
   >
   > Delete the `case "$gate" in … esac` block. The watcher then only reports Master
   > Gate state.

   The script has four outcomes:
   - `RELEASED`: PyPI serves 0.18.1. The bead closes `done`.
   - `PUBLISH_RED`: a post-merge Publish run failed.
   - `FULL_CI_RED`: two consecutive Full CI runs since the start failed. That releases
     the land agent after about 4h instead of 10h.
   - `DEADLINE`: 10h passed.

   Every outcome except `RELEASED` closes the bead `canceled`, with the last 60 log
   lines as the note.

7. **Preflight inline.** Run `sh -n "$W/watch.sh"`, then
   `sh "$W/watch.sh" <this-phase-id> "$W/watch.log" preflight`. It must exit 0 and print
   one fully populated `state` line (or `outcome=RELEASED`). Preflight never dispatches,
   reruns, or closes. If the line has `?` fields or the command fails, fix the script's
   gh or jq invocation only, without touching the repo, and retry once. If it still
   fails, close `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"`
   with the output, and run `/sase_final`.
8. **Record, then hand off.** Append a phase note with:
   - the master HEAD;
   - the core pin;
   - the preflight line;
   - whether Publish was dispatched in step 5;
   - the exact monitor command.

   Then, from the workspace root, run this in the foreground until it exits:

   ```sh
   sase monitor start \
     -C "$PWD" \
     -r 'Watch sase v0.18.1 through ci_watch, Publish, and PyPI; the watcher closes <this-phase-id>' \
     -t 11h \
     -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -L 'sase 0.18.1 watch' \
     -- 'timeout -k 5m 10h30m sh /tmp/sase-release-watch-<this-phase-id>/watch.sh <this-phase-id> /tmp/sase-release-watch-<this-phase-id>/watch.log; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 60; sase bead close <this-phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Pass the command as one quoted string with no `bash -c` wrapper. Do **not** pass
   `--next`, `-m`, `-p`, or `--idle-timeout`. A successful start ends this turn, and
   that handoff is exempt from `/sase_final`. If `sase monitor start` exits nonzero
   twice, close `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` with both
   errors, then run `/sase_final`.

### Why the land agent always runs

Each layer ends inside the next one out:

1. The watcher stops at 10h.
2. The outer `timeout` stops at 10.5h, plus 5 minutes of kill grace.
3. The fallback close runs after a 60s pause.
4. The monitor's own 11h timeout never fires on a live host.

A fallback close after a successful close is refused as a conflicting re-close, and
`|| true` absorbs that.

| Path                                                   | Who closes the bead                               | What settles the session             |
| ------------------------------------------------------ | ------------------------------------------------- | ------------------------------------ |
| `RELEASED`, `PUBLISH_RED`, `FULL_CI_RED`, `DEADLINE`   | the watcher (3 tries)                             | wrapper exits 0, monitor `completed` |
| watcher hang, or outer `timeout` fires                 | the wrapper's fallback close                      | wrapper exits 0                      |
| watcher crash, or its close fails                      | the wrapper's fallback close                      | wrapper exits 0                      |
| prerequisite, epic-symbol, preflight, or start failure | this agent, inline                                | this agent's `/sase_final`           |
| a human stops the monitor                              | nobody automatically; the land agent can close it | monitor `stopped` counts as success  |
| reboot (`lost`) or an external SIGKILL                 | nobody                                            | a human: `sase bead work <epic-id>`  |

Step 1 makes recovery safe. `sase bead work <epic-id>` relaunches only the land agent if
this bead is closed, and relaunches the watch if it is open.

## Landing

The land agent confirms the release before it lands anything as done.

1. **If sase 0.18.1 is on PyPI,** verify each item and record it:
   - PyPI (`https://pypi.org/pypi/sase/0.18.1/json`) lists a wheel and an sdist; record
     the filenames, SHA-256 values, and upload times.
   - `gh release view v0.18.1 -R sase-org/sase` exists, along with its tag commit.
   - The merged release PR shows the ci_watch merge
     (`gh pr view <n> --json mergedBy,mergedAt,mergeCommit`).
   - The green Full CI run (ID and SHA) and the green Master Gate run at the merged
     HEAD.
   - The Publish run ID, with every job's conclusion.
   - **Clean install proof.** Outside any repo, with an isolated `HOME`, create
     `uv venv --python 3.12` and run `uv pip install sase==0.18.1` with `UV_NO_CACHE=1`.
     The installed `sase` must report 0.18.1. The `sase-core-rs` version must equal the
     pin. `sase core health --json` must pass.

   Write these into `sase_v0_18_1_release_evidence.md` in a temporary directory outside
   the repo. Register it with
   `sase artifact create -p <file> -l "sase v0.18.1 PyPI release evidence" --bead <this-epic-id>`.
   If this epic is nested, also add a `sase bead note` on the root epic bead that names
   the artifact ref. Close the task beads that phases named as resolved, each with its
   evidence. Then close this epic.

2. **If 0.18.1 is not on PyPI,** read every phase's close note and the watcher log
   (`/tmp/sase-release-watch-<watch-id>/watch.log`, if present), and fix the blockers:
   - Fix small, clearly scoped blockers yourself, under the phase rules above.
   - For anything that needs CI round-trips, a core release, or waiting, author a
     **nested epic** with `/sase_plan`. It covers only the remaining blockers, plus a
     final self-closing watch phase built exactly like the watch phase above:
     prerequisites, preflight, a no-`--next` monitor, and a wrapper fallback close.
   - The nested epic's Landing section must repeat this whole Landing section verbatim,
     so every lander keeps creating nested epics until 0.18.1 is observed on PyPI.
   - Only the lander that actually observes 0.18.1 on PyPI creates the evidence
     artifact.
   - Turn the phases' `PROPOSED FOLLOW-UP` notes into task beads with `/sase_new_task`
     only when they are not release blockers.
