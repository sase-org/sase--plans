---
tier: epic
title: 'Every sase workflow green, release PR #322 merged, and sase 0.19.0 proven
  on PyPI with evidence artifacts'
goal: Master Gate, Full CI, CI Telemetry, and the Release steward are green on a master
  tip that holds every fix, and sase-core's master CI is green. The release steward
  reports the open release-please PR as releasable, the PR is merged, the steward's
  release.yml train publishes the version to PyPI, and the published files are verified
  against the sealed manifest, attested, and installable. The proof is registered
  as SASE artifacts.
decisions:
  merge_prompt:
    ask: 'Once #322 is releasable, raise a gate whose approval runs the steward''s
      own merge command?'
    choices:
      gate: One gate runs the steward's exact merge argv; a hand merge or enabling
        steward merges also works
      poll: No gate; only poll up to 72h for you or the steward to merge, else the
        phase ends canceled
    default: gate
    why: Merge authority is off, so a human must act; the gate hands you that one
      exact command
    answer: gate
  rehearsal:
    ask: Rehearse release.yml on the refreshed release PR branch before the version
      is tagged?
    choices:
      release_branch: One rehearse-mode run (about 2h of Actions) proves the train
        before a break can burn 0.19.0
      skip: Go straight to the real release; faster, but a train break at the tag
        burns 0.19.0
    default: release_branch
    why: The train has never finished a run; its last rehearsal (2026-10-25) was red
    answer: release_branch
phases:
- id: master-reds
  title: The deterministic reds on master are fixed
  depends_on: []
  size: medium
  description: 'master-reds: fix the test_core_rust monkeypatch ordering, the coverage-contexts
    crash from a global os.stat patch, and the Archive golden drift from the focus
    change, plus any other deterministic red at the tip.'
- id: steward-retry
  title: The release steward survives transient GitHub 5xx reads
  depends_on: []
  size: small
  description: 'steward-retry: give tools/_release_watch_transport.py GithubTransport
    a bounded retry for transient HTTP 5xx and timeouts, with tests and a docs sentence.'
- id: core-ci
  title: sase-core's master CI is green and the steward stops reporting core_ci_red
  depends_on: []
  size: small
  description: 'core-ci: rerun or repair sase-core''s red master CI run 42673707335
    so tools/release_watch no longer reports core_ci_red.'
- id: rehearse
  title: An accepted release.yml rehearsal on the refreshed release PR branch
  depends_on:
  - master-reds
  size: small
  description: 'rehearse: refresh the release-please branch onto the fixed master,
    dispatch one rehearse-mode release.yml run there, and accept it job by job (or
    skip per the rehearsal decision).'
- id: ci-green
  title: Every workflow that runs on master is green on a tip holding every fix
  depends_on:
  - master-reds
  - steward-retry
  - core-ci
  size: medium
  description: 'ci-green: drive Master Gate, Full CI, CI Telemetry, and the Release
    steward green on a tip holding every phase commit, repair what is still red, and
    confirm the steward reports the release PR releasable.'
- id: release
  title: sase 0.19.0 is merged, published, verified on PyPI, and recorded as artifacts
  depends_on:
  - ci-green
  - rehearse
  size: small
  description: 'release: get the release PR merged per the merge_prompt decision,
    watch the steward''s train to PyPI, verify the published files, and register the
    evidence as SASE artifacts.'
proposed_by: bbugyi200.athena.1b0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 14:38:12
status: wip
bead_id: sase-1s7
---

- **PROMPT:** [prompts/202611/ci_green_v0_19_0_pypi_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/ci_green_v0_19_0_pypi_release.md)
- **BEAD:** [sase-1s7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1s7/README.md)

# Every sase workflow green, release PR #322 merged, and sase 0.19.0 proven on PyPI

The user wants four outcomes, in order:

1. Every CI workflow of the sase project is green.
2. The next release-please PR is merged.
3. The new version is verified to be on PyPI.
4. That proof is registered as one or more SASE artifacts.

The next release PR is **#322**, `chore(master): release 0.19.0`. If release-please
retitles it before the merge, follow the version the open PR carries.

Read `docs/contribute/release-train.md` before any phase that touches the release path.
In short:

- `release.yml` is an exact-candidate train. It builds once at the tag, seals a
  manifest, qualifies against the published core by calling `ci.yml`, exercises Linux
  and macOS, publishes the retained bytes, confirms them from PyPI, and un-drafts the
  release.
- `tools/release_watch` (the steward) is the single decider. Every 15 minutes the
  scheduled `release-steward.yml` ticks live. It refreshes release-please metadata and
  dispatches `release.yml mode=release` at a fresh tag by itself. It only **merges** the
  release PR when the repository variable `RELEASE_STEWARD_MERGE_ENABLED` is the literal
  `true`, and that variable is unset.

## Evidence (observed 2026-11-10 around 19:00 UTC, master `c7f6cdce75`)

| #   | Red                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Phase         |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| 1   | **Master Gate is red on every push since 2026-11-08 09:31 UTC (run 42467573771 onward).** It always fails on `tests/test_core_rust.py::test_require_rust_extension_raises_when_missing` with `ImportError: No module named 'sase'`. The test first replaces `importlib.import_module` with a raising fake. It then calls `monkeypatch.setattr("sase.core.rust.install_context", ...)` with a **string** target, which `dde1ada139` added. CI resolves the dev dependencies fresh and gets **pytest 9.2.0**, whose `_pytest.monkeypatch.resolve` calls `importlib.import_module`, so the string target hits the fake. A workspace venv on pytest 9.0.2 resolves through `__import__` and passes, which hides the bug locally. The sibling `test_require_rust_extension_uses_uv_tool_repair_hint` patches in the safe order. Tracked by task **sase-1s6** (READY, with a +1). | master-reds   |
| 2   | **Full CI is red since 2026-11-08.** `test (3.12                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | 3.13          | 3.14)`all fail on #1. Since run 42754008243 (head`bd97664e07`), `visual-test`also reports`drift`on at least 12`agents_archive*\*_120x40`and`agents_record*__120x40`goldens: about 47% of pixels changed, and about 22% for the two`agents_archive_chooser_`goldens. Another run also listed`agents_archive_help`(0.6%) and`agents_archive_rebuilding`, so the full set must come from a complete check, not these annotations. Cause: `bd97664e07`(bead sase-1rq.2.1.4.1.5.3.1.2) added`_focus_archive_section_after_refresh`in`src/sase/ace/tui/actions/agents/_archive_view_lifecycle.py`. It moves focus into `#archive-list`or`#archive-pulse`on open, and`tests/ace/tui/test_agent_archive_column_app.py`now asserts`app.focused is archive_list`, but no golden was regenerated. Last all-green Full CI: 42434370775 (`4a7ce0f330`). | master-reds |
| 3   | **Single-lane flakes, seen once each in recent Full CI runs, with no bead found:** `tests/test_models_panel_provider_routing.py::test_models_panel_title_shows_disabled_provider_line` (3.12, run 42818081238); `tests/ace/tui/test_notification_plan_gate.py::test_tale_plan_modal_renders_no_raw_editor_for_host_collected_properties` (3.13, 42696271732); `tests/ace/tui/modals/test_snippets_panel.py::test_jk_navigation_does_not_stat_or_parse_config` (3.14, 42596662223).                                                                                                                                                                                                                                                                                                                                                                                          | ci-green      |
| 4   | **CI Telemetry is red.** `test-cost` and `contention-test` fail only on #1. `coverage-contexts` also dies in a pytest-xdist `INTERNALERROR` (`assert not crashitem` for `tests/ace/tui/test_plan_decision_ace_stale.py::test_plan_section_render_path_no_stat_no_validate`). That test globally replaces `os.stat` with a raiser. coverage's tracer calls `os.path.exists` from `coverage/inorout.py::_analyze_directory` while the patch is active, so the worker crashes. This is the class of bug closed sase-1rq.2.1.2 fixed elsewhere. `tests/_sync_io_guard.py::forbid_sync_file_io` exists for exactly this case: it only counts frames from the code under test, and `tests/test_sync_io_guard.py::test_harness_frames_nearer_than_sase_pass_through` proves coverage passes through.                                                                               | master-reds   |
| 5   | **Release steward tick 42853044142 is red** (`watcher_broken`): `gh api` returned HTTP 502 while reading master-gate runs. 5 of the last 100 ticks failed, all on transient GitHub 5xx or 504 errors (for example 42659268406, 42067261253, 40712807867). `tools/_release_watch_transport.py` retries PyPI reads on 408, 429 and 5xx (`PYPI_ATTEMPTS = 3`), but `GithubTransport._call` makes exactly one `gh api` attempt.                                                                                                                                                                                                                                                                                                                                                                                                                                                 | steward-retry |
| 6   | **sase-core's master CI is red, which blocks the steward.** A read-only `tools/release_watch --once --json` returns primary `core_ci_red` with evidence sase-core run 42673707335: CI on the push of `chore: release v0.44.2 (#345)`, head `1ba79d2f`. That run lists a single job (`release workflow scripts`, success) yet concluded `failure` after 50 s. Its `Runner starvation retry` run 42673888372 failed in 0 s with no log. Every earlier sase-core master CI run succeeded, so this looks like infrastructure, not code.                                                                                                                                                                                                                                                                                                                                         | core-ci       |
| 7   | **The train has never completed.** The newest `release.yml` run is rehearsal 40234446645 (2026-10-25, red: exercise `UPGRADE_NO_INSTALLABLE_PREVIOUS`, plus qualify). 0.18.1 shipped before the train cutover. Since then `72a1e3a18e` resolved the previous release in a throwaway venv, and `173a548ee1` sealed core-pin containment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | rehearse      |

### Facts the release phases rely on

**Release PR #322**

- Head `739799da59`, based on master `c7f6cdce75`; open and mergeable.
- The merge gate's required checks (`.github/release-qualification.json`
  `merge_gate.required_checks`: `Conventional PR title`, `prepare`,
  `release-core-floor-smoke`) are green.
- It bumps `pyproject.toml` to 0.19.0 and the exact core pin to `sase-core-rs==0.44.2`.
- Tag `v0.44.2` contains `sase-core-revision.txt` pin `5c0e49cf` (GitHub compare: ahead
  by 1).

**PyPI**

- `sase` latest is 0.18.1.
- `sase-core-rs` latest is 0.44.2, and 0.43.2 (0.18.1's core) is still listed.

**Merge authority**

- `RELEASE_STEWARD_MERGE_ENABLED` is unset and no cutover gate is pending, so the
  steward will report `releasable` with its merge action suppressed.
- The owner merged #321 and #299 by hand.
- The steward's own merge argv is
  `gh pr merge <n> --merge --match-head-commit <head_sha>`
  (`tools/_release_watch_actions.py`).

**The `pypi` environment**

- It allows the `master` branch and `v*` tags and has no required reviewers, so the
  tag-run publish needs no human approval.

**Release-please**

- `release-please-config.json` cuts draft releases with `force-tag-creation`.
- `publish.yml` (every 3h, or dispatched bare by the steward) refreshes #322 and, after
  a merge, cuts the tag and draft release that the steward then trains.

**Already green on master:** Deploy Docs, Install smoke, Publish, Core Pin Ratchet, and
Shard Timings Ratchet.

**Out of scope:** the stale bot PRs (#300, #319, #320, #323, #326) and the external PR
#301. #319's CI is red only on #1 and #2. Merging any of them is the owner's call.
Mention their status in the final summary; do not merge or close them.

## Overlapping in-flight work: respect it, do not take it over

Three older epics chase the same release, and none has a live worker. At planning time,
`sase agent list` showed only WAITING land agents for them and one live monitor.
`sase gate list` had no pending cutover gate.

| Bead(s)                                                  | State and owner                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| sase-1s6 (task, READY)                                   | Exactly Evidence #1. **This plan resolves it:** master-reds names it in its close note, and the land agent closes it.                                                                                                                                                                      |
| sase-1q2.9.2 / .3 / .4 (phases of epic sase-1q2.9)       | IN_PROGRESS since 2026-10-31: core pin, master rehearsal, and release watch for 0.19.0. No live agent.                                                                                                                                                                                     |
| sase-1r2.6 (phase of epic sase-1r2)                      | IN_PROGRESS: "One Full CI run on master is green in every job." No live agent. Epic sase-1r2 installed an hourly reboot-watchdog crontab entry (marker `# sase-epic-watchdog:sase-1r2`). It relaunches that epic's agents only after a reboot, and expires about 14 days after 2026-11-03. |
| sase-1rq.2.1.7, sase-1rq.2.1.4.1.5.3.1.3 (epic sase-1rq) | WAITING or IN_PROGRESS: "confirm every in-scope workflow green", and "confirm hosted contention-test on landed master" (blocked on Telemetry run 42711704334).                                                                                                                             |
| sase-1n6.11.4, sase-1oe.4 (E2, epic sase-1n6)            | IN_PROGRESS: "first steward release, acceptance evidence, and the E2 decision record", and the human cutover gate. Monitor `sase-1n6.11.4 settings` is live and polls for the human cutover settings.                                                                                      |

Rules for these beads, binding on every phase:

- **Do not touch their lifecycle.** Never close, reopen, edit, reassign, relaunch, stop,
  or resume these beads, their agents, or their monitors. Never touch the sase-1r2
  watchdog crontab entry.
- **Notes are allowed.** Where a phase below says so, `sase bead note <id> "<text>"` may
  add evidence pointers (run URLs, artifact refs). So may
  `sase artifact link add <ref> related bead:<id> "<reason>"`.
- **Check ownership before fixing.** Before you fix any red, run
  `sase bead search '<node id or symptom>'` and `sase agent list -j`. If a **RUNNING**
  agent is visibly fixing that exact red, leave it, and say so in your close note.
  Otherwise fix it, and name every task bead your evidence resolves.
- **Stop on a revived owner.** If one of these epics comes back to life mid-plan (for
  example, the sase-1r2 watchdog relaunches it after a reboot), stop before duplicating
  its in-flight action. Record that as a `PROPOSED FOLLOW-UP:`.

## Rules for every phase

- **Read memory first** through `/sase_memory_read`:
  - always `lint_and_test.md` and `sase_beads.md`;
  - `tui.md` and `tui_screenshot.md` for golden work;
  - `ace.md` before any change under `src/sase/ace/`;
  - `tools.md` before any change under `tools/`;
  - `docs_writing.md` before editing `docs/`;
  - `sase_artifacts.md` before creating artifacts;
  - `symvision.md` if that gate fails.
- **Re-inventory first.** `git fetch` and re-inventory on the current `origin/master`
  tip before editing. Other epics land on master continuously, so the Evidence table is
  a starting point, not a whitelist.
- **Fix waits, fixtures, and products, never tolerance.**
  - Do not raise timeouts, budgets, caps, or perf bounds.
  - Do not add retries around assertions, and do not skip or xfail.
  - Do not loosen pixel comparison or delete assertions.
  - Wait on the semantic state an assertion is about, using
    `sase.ace.testing.wait.wait_for` or the file's existing helper.
- **Goldens.**
  - Regenerate a golden only after its capture is deterministic.
  - Never copy CI candidate PNGs into the tree.
  - Inspect every created, removed, and updated group before accepting it.
  - Every dirty golden belongs in your commit (`/sase_final` rule).
- **Rust boundary.** None of these fixes is expected to need a sase-core change. If one
  does, work through `sase repo open sase-core` and run that repo's own
  `sase tool run check` there.
- **Verify.**
  - Run `just fix`, then `sase tool run check`, in every checkout you changed.
  - Never run `just check-full` (`decisions:check-full-is-explicit`): Full CI and the
    rehearsal are this epic's exhaustive signal.
  - The CI `lint` job enforces `toobig`, so never grow a file past its line limit.
  - If the workspace `.venv` is stale, run `just install-venv` first.
- **Hand waits to monitors.** Every long command goes to `/sase_monitor` with an
  explicit deadline and a `--next` follow-up: verification, visual runs, CI waits, PyPI
  waits. Pure waits (sleeps, `gh run watch`, polls) use `-w 0`. Use
  `gh run watch --interval 120 || true`, because watch exits on API rate limits; then
  judge the run with `gh run view --json jobs`. Never end a turn waiting.
- **Outward actions you may take, and nothing else.**
  - `gh run rerun` exactly where a phase says so.
  - `gh workflow run` for `full.yml`, `telemetry.yml`, and `release-steward.yml`, the
    last only in its default observe-only form with no `live` input.
  - `gh workflow run publish.yml`, bare: this is the steward's own metadata refresh.
  - `gh workflow run release.yml -f mode=rehearse`, only in the rehearse phase.
- **Forbidden outward actions.**
  - Never dispatch `release.yml` with `mode=release` or `mode=resume`; the steward does
    that.
  - Never run `gh variable set` or edit any environment or branch protection.
  - Never dispatch sase-core's `release-plz.yml`.
  - Never merge any PR, except through the release phase's human-approved gate.
- **No new beads; always close.** Phase workers create no beads. Record follow-ups as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead. To close:
  1. `sase bead epic-symbols <your-phase-id>` must list nothing.
  2. Run `sase bead close <id> -R done -n '<evidence>'`, then finish through
     `/sase_final`.

  If you cannot finish within the phase's bounds, still close, with `-R canceled` and
  the evidence, so the dependent phases and the land agent always run.

## Phase master-reds

**Repo:** sase. Read `tui.md`, `tui_screenshot.md`, and `ace.md` before the golden work.

1. **Re-inventory.**
   1. Fetch.
   2. Read the newest completed Master Gate at `origin/master` HEAD, the newest Full CI,
      and the newest CI Telemetry:
      `gh run list -R sase-org/sase --workflow <file> --limit 3`, then
      `gh run view <id> --log-failed`.
   3. List every failing node and visual drift name.

   Fix every deterministic red in Master Gate, Full CI, or Telemetry that is not owned
   by a live agent. Flakes belong to ci-green.

2. **Match CI's pytest.** Run
   `.venv/bin/python -c 'import pytest; print(pytest.__version__)'`. CI resolves pytest
   fresh (9.2.0 at planning); grep a recent CI log for `platform linux -- Python` to
   confirm. If the local version differs, temporarily run
   `uv pip install --python .venv/bin/python 'pytest==<CI version>'`. Prove the
   test_core_rust and stale-plan fixes under that version and under the venv's own
   version, then restore with `just install-venv`.
3. **test_core_rust (Evidence #1, resolves sase-1s6).**
   - Reproduce the failure under CI's pytest first.
   - Then make `test_require_rust_extension_raises_when_missing` patch `install_context`
     before replacing `importlib.import_module`, matching the sibling test. Passing the
     module object instead of a dotted string also works; prefer whichever form keeps
     `evict_rust_extension` semantics intact.
   - `rg -n 'setattr\(importlib, "import_module"' tests/` found no other test with this
     ordering at planning; re-check.
4. **coverage-contexts crash (Evidence #4).**
   - Replace the global `os.stat` lambda in
     `test_plan_section_render_path_no_stat_no_validate` with
     `with forbid_sync_file_io("os.stat", reason="plan render path must not stat"):`
     around the two `section._load_plan_sheet(...)` calls. Keep the `validate_plan`
     guard and every assertion.
   - Reproduce the lane's shape: read the `test-contexts` recipe in the `Justfile` and
     run the file with its coverage-context flags under `-n 2`. Prove there is no
     INTERNALERROR and the node passes.
   - Prove that the guard still bites: temporarily call `os.stat(plan)` inside
     `_load_plan_sheet` and confirm the test fails, then revert.
   - Re-check for other tests that replace `os.stat`, `os.lstat` or `os.path.exists`
     globally with a raiser that is not scoped to code under test. Planning found only
     this one.
5. **Archive golden drift (Evidence #2).**
   1. Get the complete drift set. Either run `just test-visual`, the check-only form,
      for the Archive and record families (selectors after `--`, through a monitor), or
      download the newest Full CI `visual-test` report into a temp dir outside the
      repository. Use the report only for names; never copy its PNGs.
   2. Inspect the expected PNG next to a fresh local capture for one representative per
      group. Accept only if the change is the intended focus move from `bd97664e07`:
      focus-dependent styling of the Archive list or pulse, and the corresponding
      unfocused styling elsewhere. If it shows anything else (lost rows, broken layout,
      wrong content), fix the product in `_archive_view_lifecycle.py` instead and keep
      the goldens.
   3. Prove the capture is deterministic. Focus is applied through `call_after_refresh`,
      so capture the affected nodes 3 times and require identical bytes. If they differ,
      make the capture wait on the semantic state (`app.focused` is the expected section
      widget) before regenerating.
   4. Run `just fix-tui-screenshots -- <selectors>`. Read its report (status, the
      `skipped` list, each update group) as `lint_and_test.md` requires.
6. Run `just fix` and `sase tool run check`, then land.

**Close note.** Name sase-1s6 as resolved. Give the pytest versions you proved under,
the golden families updated with counts, and the determinism proof.

**Done when:**

- Both nodes pass under CI's pytest and under the coverage-context run.
- The Archive and record visual families check clean locally and deterministically.
- Every other deterministic red found in step 1 is fixed with the same proof.
- `sase tool run check` passes.

## Phase steward-retry

**Repo:** sase. Read `tools.md` and `docs_writing.md`.

1. **Add a bounded retry to `GithubTransport._call`** in
   `tools/_release_watch_transport.py`:
   - At most 3 attempts total, with short backoff (about 2 s, then 5 s). Use an
     injectable `sleep`, as `PypiTransport` already does.
   - Retry only when `gh api` exits nonzero and its stderr or stdout names a transient
     server error: HTTP 500, 502, 503 or 504 (for example `HTTP 502`, `Server Error`,
     `Bad Gateway`, `Service Unavailable`, `Gateway Timeout`), or when the call raises
     `subprocess.TimeoutExpired`.
   - Never retry 4xx (401, 403, 404, 422) or rate-limit responses.
   - After the last attempt, raise the same `ReleaseTransportError(path, ...)` shape as
     today, carrying the last attempt's message, so `watcher_broken` reporting and its
     existing tests stay unchanged.
   - A paginated call retries the whole `--paginate --slurp` invocation.
   - Confirm the worst case still fits the steward job's 25-minute timeout.
2. **Tests.** Find the existing transport tests with
   `rg -l GithubTransport tests tools`. Add cases:
   - 502 then success returns the data and sleeps once;
   - 404 fails at once with no sleep;
   - three 502s raise with the last message;
   - a timeout followed by success.
3. **Docs.** In `docs/contribute/release-train.md`, "Inspecting the release steward",
   add one sentence: transient GitHub 5xx reads retry up to three attempts before a tick
   reports `watcher_broken`.
4. Run `just fix` and `sase tool run check`, then land.

**Done when** the new tests pass, the existing release_watch tests are unchanged and
green, and the repo checks pass.

## Phase core-ci

**Repos:** sase-core, with read-only CI inspection first. Use
`sase repo open sase-core -r '<why>'` before reading its files.

1. **Re-check.** Run
   `gh run list -R sase-org/sase-core --workflow ci.yml --branch master --limit 5`. If
   the newest master CI run is already green, go to step 4.
2. **Diagnose run 42673707335**, or the newest red master run.
   - Read its jobs with
     `gh api repos/sase-org/sase-core/actions/runs/<id>/attempts/<n>/jobs` and its
     check-suite.
   - Read sase-core's `.github/workflows/ci.yml` in the opened checkout to see which
     jobs a release-commit push should run.
   - Decide whether the failure is infrastructure (jobs never created or never got a
     runner, no failed step) or a real job failure.
3. **Repair.**
   - **Infrastructure:** rerun the whole run once with
     `gh run rerun <id> -R sase-org/sase-core`. There is no failed job, so `--failed`
     has nothing to rerun. Watch it with a zero-weight monitor (deadline about 2h,
     `--next`).
   - **Real failure, or the rerun fails again:** fix it in the sase-core checkout. Run
     that repo's `just fix` and `sase tool run check`, land through host completion, and
     watch the resulting push CI run green.
   - Never dispatch `release-plz.yml` and never merge sase-core PRs.
4. **Verify.** From the sase checkout, run
   `.venv/bin/python tools/release_watch --once --json`. It needs `sase_core_rs` built
   from the pin; run `just install-venv` if the import fails. The primary must no longer
   be `core_ci_red`. A remaining `gate_red` from Master Gate is expected here and
   belongs to ci-green.

**Done when** sase-core's newest master CI run is green and the steward's primary state
is no longer `core_ci_red`. Record both run URLs.

## Phase rehearse

**Repo:** sase. This phase normally changes no files. Read `/sase_monitor` first.

> [!decision] rehearsal = skip Do not dispatch anything. Close
> `-R done -n "rehearsal skipped per the rehearsal decision"` and finish.

> [!decision] rehearsal = release_branch Follow the steps below.

1. **Prerequisite.** master-reds' commit is on `origin/master`. Check with
   `git log origin/master --grep 'SASE_BEAD=\[<master-reds phase id>\]'`, re-fetching
   for up to about 5 minutes. If master-reds closed canceled, close
   `-R canceled -r "rehearsal not started: master-reds canceled"`.
2. **Refresh the release branch.**
   - If #322 is already merged, close `-R done` noting that the real train supersedes
     the rehearsal.
   - Otherwise check
     `git merge-base --is-ancestor <master-reds sha> origin/release-please--branches--master`.
   - If the branch does not contain the fix, dispatch
     `gh workflow run publish.yml -R sase-org/sase`, bare. This is the steward's own
     metadata refresh; with #322 open it only regenerates the PR. Poll with a
     zero-weight monitor (deadline about 1h) until the branch contains the fix.
3. **One release.yml run at a time.** Run
   `gh run list -R sase-org/sase --workflow release.yml --limit 5`. If one is queued or
   in progress, wait for it.
4. **Rehearse.**
   1. Dispatch
      `gh workflow run release.yml -R sase-org/sase --ref release-please--branches--master -f mode=rehearse`.
      Use no `core_version` override: the branch pins `sase-core-rs==0.44.2`, whose tag
      contains the source pin.
   2. Record the run ID and the branch head SHA.
   3. Run `tools/release_watch --once --json` immediately. It must not report
      `ambiguous_trains`; a branch rehearsal is not a tag train. If it does, cancel the
      rehearsal with `gh run cancel` and record why.
   4. Hand `gh run watch <id> -R sase-org/sase --interval 120 || true` to a zero-weight
      monitor (deadline about 4h, `--next`). Long queues are not failures: never cancel,
      just wait.
5. **Accept job by job.**
   - `identity`, `build` (including containment), and `qualify-prepare` are green.
   - `qualify` is green, which requires the blocking `lint` and `test (3.12|3.13|3.14)`.
     An advisory red (`visual-test`, `perf-floors`, `tutorial-tests`,
     `ace-page-group-isolation`) becomes a `PROPOSED FOLLOW-UP:`; it never fails
     acceptance.
   - All four `exercise-legs` and the merged `exercise` verdict are green. Both wheel
     legs resolve 0.18.1 as the previous release and pass the N-1 upgrade.
   - `publish` (verify-only) is green with an `upload` plan for every sealed file,
     because the version is not on PyPI. Any refusal is red.
   - `stash`, `confirm` and `confirm-install` are skipped in rehearse mode.
6. **Bounded repair.**
   - If qualify is red at attempt 1 only on blocking `tests` failures, run
     `gh run rerun <id> --failed` once, mirroring the train's own rule. Record the flaky
     nodes as `PROPOSED FOLLOW-UP:`.
   - Any other red is a defect to fix on master before the tag exists, because a tag
     runs its own workflow files. Fix it under the global rules, land, refresh the
     branch, and rehearse once more. At most 2 rehearsal dispatches in total.
   - If it is still red, close `-R canceled` with the run URL, failing job and step.
7. **Notes.** Add the run URL and per-criterion verdict to sase-1q2.9.3 and sase-1oe.4.
   Change no status.

**Done when** one rehearsal meets every acceptance criterion. Close with its URL and the
verdicts.

## Phase ci-green

**Repo:** sase, plus read-only checks.

1. **Prerequisites.** The commits of master-reds and steward-retry are on
   `origin/master` (the same grep as above), and core-ci closed done. If one closed
   canceled, continue anyway and carry its evidence into the close note.
2. **Master Gate.**
   1. Judge the newest completed Master Gate whose head contains every phase commit. If
      none exists yet, wait for the push run with a zero-weight monitor.
   2. If it is red, run the failing node locally (20 times serially for a suspected
      flake):
      - **Flake:** `gh run rerun <id> --failed` once, plus a `PROPOSED FOLLOW-UP:`
        naming the node and run.
      - **Deterministic:** fix it, land, and judge the new push.
3. **Full CI and Telemetry.**
   1. Unless a run at a SHA containing every phase commit is already queued or running,
      dispatch both:
      - `gh workflow run full.yml -R sase-org/sase --ref master` (about 1h50m);
      - `gh workflow run telemetry.yml -R sase-org/sase --ref master` (about 2h30m).
   2. Watch both in one zero-weight monitor (deadline about 4h, `--next`).
   3. Judge job by job.
4. **Repair rounds** (at most 3).
   - **Deterministic red:** fix it and land, then redispatch only the red workflow.
   - **Flake** (for example Evidence #3 recurring):
     1. Search beads.
     2. Rerun the failed jobs once with `gh run rerun <id> --failed`, and record a
        `PROPOSED FOLLOW-UP:` with the node, run URL, and any existing owner bead.
     3. If it fails again, treat it as deterministic: find the race, wait on semantic
        state, land, and redispatch.
   - Telemetry's `contention-test` is the domain of the sase-1rq epic. Fix a red there
     only after the ownership check finds no live owner. A contention failure that needs
     more than a bounded fix becomes a `PROPOSED FOLLOW-UP:`.
5. **Release steward.** Once Master Gate and sase-core CI are green, require one green
   steward run whose reported state is `releasable` (merge suppressed) or a non-red
   `waiting` (metadata refresh or cadence). Either wait for a scheduled tick or dispatch
   the observe-only `gh workflow run release-steward.yml -R sase-org/sase`. Read the
   state from the run log or step summary. Scheduled live ticks may dispatch
   `publish.yml` to refresh #322 themselves; that is expected.
6. **Release PR.** #322 is refreshed onto a base containing every phase commit, and its
   PR checks (CI `prepare`, `release-core-floor-smoke`, PR Title) are green. If the
   steward has not refreshed it within about 3h, dispatch `publish.yml` bare yourself.
7. **Other workflows.** Confirm that the newest master runs of Deploy Docs, Install
   smoke, Publish, Core Pin Ratchet, and Shard Timings Ratchet are still green.
8. **Notes.** Add the Full CI run URL and verdict to sase-1r2.6. Add the Telemetry
   `contention-test` result (job URL, conclusion, duration) to sase-1rq.2.1.4.1.5.3.1.3.
   Add the workflow table to sase-1rq.2.1.7. Change no status.

**Close note:** a table of workflow → newest run URL → head SHA → conclusion, plus these
two lines:

- `RELEASE_PATH_GREEN=yes|no`: Master Gate, Full CI's blocking jobs, sase-core CI, the
  steward state, and #322's checks.
- `ALL_GREEN=yes|no`: every workflow above, including Full CI's advisory jobs and
  Telemetry.

Close `done` only when `ALL_GREEN=yes`. If the 3 rounds end with only off-release-path
reds (Telemetry, or Full CI advisory jobs), close `-R canceled` with
`RELEASE_PATH_GREEN=yes`, the remaining reds, and their `PROPOSED FOLLOW-UP:` notes.

## Phase release

**Repo:** sase. Read `sase_artifacts.md`. Read `/sase_gate` and `/sase_monitor` before
using them.

1. **Idempotent start.** Determine the target version from the open or just-merged
   release-please PR, `chore(master): release X.Y.Z` (expected 0.19.0, #322). If
   `curl -fsS https://pypi.org/pypi/sase/<version>/json` already succeeds, go to step 6.
2. **Prerequisites.**
   - ci-green's close note must say `RELEASE_PATH_GREEN=yes`.
   - rehearse must be closed done; skipped per its decision counts.
   - Otherwise close `-R canceled -r "release not attempted: <phase> not green"` with
     its evidence.
   - Run `.venv/bin/python tools/release_watch --once --json`. Expect `releasable`. For
     `waiting`, poll `--once` every 5 minutes with a zero-weight monitor (deadline about
     3h).
   - For a red state, record it and stop with `-R canceled`. Do not repair code here;
     the earlier phases own repairs.
3. **Merge.**

   > [!decision] merge_prompt = gate
   >
   > 1. Read the steward's current merge action from `--once --json`: PR number and head
   >    SHA.
   > 2. Create one gate through `/sase_gate`, titled like
   >    `Merge sase release PR #<n> (<version>)`.
   > 3. Its approve action runs exactly
   >    `gh pr merge <n> -R sase-org/sase --merge --match-head-commit <head_sha>`.
   > 4. Its body links the steward state, the Master Gate and Full CI runs, and the
   >    rehearsal.
   > 5. A second branch lets the owner answer that they merged by hand or set
   >    `RELEASE_STEWARD_MERGE_ENABLED=true` themselves.
   >
   > Creating the gate ends this turn, so continue in its follow-up. If
   > `--match-head-commit` refuses because release-please moved the head, re-read the
   > action and raise the gate once more. If the PR is already merged when the gate
   > would be raised, skip the gate.

   > [!decision] merge_prompt = poll Raise no gate. Step 4's watch also covers the
   > merge: poll up to 72h for the PR to merge by hand or by the steward. If it never
   > merges, close `-R canceled` noting that the merge never happened.

   Never merge outside the approved gate. Never set the variable.

4. **Watch the train.** Hand
   `.venv/bin/python tools/release_watch --until-released <version> --poll 5m --deadline 24h`
   to a zero-weight monitor: timeout 25h (73h with `merge_prompt = poll`, deadline 72h),
   with `--next`.
   - The live steward cuts the tag and draft release (via `publish.yml`), dispatches
     `release.yml mode=release` exactly once, and handles its own reruns and resumes.
     This phase never dispatches `release.yml` itself.
   - If the train ends `burned` or `abandoned`, `publish_conflict`, `vetoed`, or stays
     red past the deadline: record the steward report, run URLs and failing jobs as
     `PROPOSED FOLLOW-UP:`, and close `-R canceled`. Do not start a second release
     cycle.
5. **Confirm completion.** The release.yml run at `v<version>` concluded success with
   `confirm` green, and `tools/release_watch --once --json` reports `released`.
6. **Verify PyPI.** Every check must hold:
   - `https://pypi.org/pypi/sase/<version>/json` lists exactly the sealed wheel and
     sdist, with upload times, sizes, and SHA-256 digests.
   - Those digests equal the sealed candidate manifest on the GitHub release
     (`tools/release_candidate show`/`download`, or `confirm`). The sets match exactly,
     with nothing extra.
   - Each file has PyPI provenance from the GitHub trusted publisher
     (`https://pypi.org/integrity/sase/<version>/<filename>/provenance`) naming
     `sase-org/sase` and `release.yml`.
   - GitHub release `v<version>` is published (not a draft) and marked Latest.
   - **Clean install,** in a fresh temp dir outside every repository, with isolated
     `HOME`, `SASE_HOME`, `XDG_*`, `UV_TOOL_DIR` and `UV_TOOL_BIN_DIR`, `UV_NO_CACHE=1`,
     and caller `SASE_*` variables unset:
     1. Run `uv tool install --python 3.12 sase==<version>`.
     2. `sase version -j` reports `<version>` and the manifest's exact `sase-core-rs`
        (expected 0.44.2).
     3. `sase core health --json` passes.
7. **Evidence artifacts.** Write the files in a fresh temp dir outside every repository:
   - `sase-<version>-pypi-release-evidence.md`, the human-readable proof:
     - the version, PyPI URL, and release URL;
     - the merge commit, tag, and peeled SHA;
     - the release.yml run URL with each job's conclusion;
     - a table of each file's PyPI SHA-256 against the manifest SHA-256;
     - the provenance summary;
     - the clean-install transcript excerpt;
     - the steward's `released` verdict;
     - ci-green's workflow table;
     - the rehearsal URL.
   - `pypi-sase-<version>.json`: the raw PyPI JSON response.
   - `install-check-sase-<version>.txt`: the raw clean-install transcript.

   For each file:
   1. Run `sase artifact create -p <file> -l "<label>" --bead <your phase id>`.
   2. Confirm with `sase artifact show <ref>`.
   3. Add
      `sase artifact link add <ref> related bead:sase-1n6.11.4 "first release through the steward train"`.
   4. Add
      `sase artifact link add <ref> related bead:sase-1r2 "sase 0.19.0 confirmed on PyPI"`.

   Then note the artifact refs on sase-1n6.11.4, sase-1q2.9.4, and sase-1r2.6. Change no
   status.

**Done when** every step 6 check holds and every artifact is registered. Close with the
artifact refs, the PyPI URL, and the release.yml run URL.

## Landing

- Close sase-1s6 and any other task bead a phase's close note names as resolved.
- Turn the `PROPOSED FOLLOW-UP:` notes into task beads through `/sase_new_task`.
  Expected candidates:
  - any recurring flake from Evidence #3;
  - advisory-lane reds;
  - stale overlapping beads (sase-1q2.9.x, sase-1r2.6, sase-1rq.2.1.7,
    sase-1rq.2.1.4.1.5.3.1.3) that the owner may now close with this epic's evidence.
- The final summary must list:
  - the green workflow table;
  - the merged PR, tag, and release.yml run;
  - the PyPI URL;
  - every evidence artifact ref;
  - the out-of-scope bot PRs and their CI state.
