---
tier: epic
title: Fix every release blocker and ship sase v0.19.0 through the release steward
goal: "Master Gate, Full CI, and a release.yml rehearsal are green on master; the
  release steward merges release-please PR #322; sase 0.19.0 is on PyPI; and the final
  lander records the release evidence as an artifact.

  "
decisions:
  wake_cutover:
    ask: Wake sase-1oe.4's stale poll early once this epic's master rehearsal is green?
    choices:
      wake:
        Stop and resume its monitor, so the E2 cutover gate can come up to ~30h sooner
      leave: Only note it; sase-1oe.4 wakes at its poll timeout, ~2026-11-01 02:40 UTC
    default: wake
    why:
      Its poll runs a pre-fix matcher that cannot see the fixed core, so waiting burns
      time
    answer: wake
phases:
  - id: core-match
    title: sase recognizes complete sase-core releases with compressed macOS wheel tags
    size: small
    depends_on: []
    description: "core-match: port sase-core's compressed-tag wheel matcher into
      tools/_release_core.py and tools/release_candidate, with tests on the live 0.43.0
      and 0.43.1 macOS filenames.

      "
  - id: core-publish
    title:
      A complete published sase-core-rs contains master's core revision, and master pins
      it
    size: medium
    depends_on:
      - core-match
    description: "core-publish: get a complete sase-core-rs that contains the source pin
      onto PyPI (one rerun of the failed 0.43.2 upload, or a wait for the daily cut),
      then ratchet master's exact pin and prove the bindings.

      "
  - id: cli-reds
    title: Fix the agent sync help, completion snapshot, and agent load CLI reds
    size: small
    depends_on: []
    description: "cli-reds: make the sync help assertion hold on Python 3.12 through
      3.14, caption the two agent sync completion slots and refresh the snapshot, and
      stop the agent load tests from assuming the athena hostname (sase-1ov).

      "
  - id: tui-reds
    title: Fix the pager, footprint, import-budget, and visual-lane reds
    size: medium
    depends_on: []
    description: "tui-reds: fix the pager notify teardown errors, the footprint
      microbench, the TUI import budget, and the tool-runs visual sentinel, then clear
      any other visual drift.

      "
  - id: harness-reds
    title: Fix the suite-gate, nested-preservation, and remaining non-TUI reds
    size: medium
    depends_on: []
    description: "harness-reds: fix the suite-gate scratch repo's import closure and the
      nested-preservation error wrapping, then every other non-TUI red at the tip.

      "
  - id: full-ci
    title: Run Full CI on a tip with every fix and repair what is still red
    size: medium
    depends_on:
      - core-match
      - cli-reds
      - tui-reds
      - harness-reds
    description: "full-ci: run Full CI on a tip containing every sibling fix, then fix
      each remaining deterministic red, including the toobig split if its routine has
      not landed it.

      "
  - id: rehearse
    title: Prove a green release.yml rehearsal at master
    size: small
    depends_on:
      - core-publish
      - full-ci
    description: "rehearse: dispatch a release.yml rehearsal at master, judge it job by
      job, note the result on sase-1oe.4, and wake its cutover per wake_cutover.

      "
  - id: watch
    title:
      Watch sase v0.19.0 through the steward, release.yml, and PyPI with a self-closing
      monitor
    size: small
    depends_on:
      - rehearse
    description:
      "watch: after prerequisite and preflight checks, start a no-follow-up monitor
      whose watcher observes the steward, release.yml, and PyPI for up to 48h and always
      closes this bead."
proposed_by: bbugyi200.athena.16e
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 12:37:04
status: wip
---

- **PROMPT:**
  [prompts/202610/v0_19_0_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_19_0_release.md)

# Fix every release blocker and ship sase v0.19.0 through the release steward

**Target.** The target is sase **0.19.0**. PyPI's newest sase is 0.18.1, published
2026-10-25. Release-please PR **#322** (`chore(master): release 0.19.0`, head branch
`release-please--branches--master`) is the release that is waiting. The request also
mentions "v0.18.0" once, but v0.18.0 has been on PyPI and GitHub Releases since
2026-10-19. Every "observe the release" step in this epic and in any nested epic means
0.19.0.

**ci_watch no longer merges sase releases.** E2's handover phase sase-1n6.9 changed how
the release happens. `ci_watch` (chezmoi `home/dot_config/sase/sase_athena.yml`) now
watches `sase-org/sase` for CI health only. Its `release_repositories` list names only
sase-github and sase-telegram, and its description says sase "is watched for CI health
only and never merged". The release steward is now the single decider (see
`docs/contribute/release-train.md`). The steward is
`.github/workflows/release-steward.yml` plus `tools/release_watch`, and its policy lives
in sase-core.

1. The steward merges the release PR only when all of these hold:
   - merge authority is on (repository variable `RELEASE_STEWARD_MERGE_ENABLED` is the
     literal `true`; today it is unset);
   - Master Gate is green at master HEAD;
   - the release PR's required checks are green;
   - a complete published `sase-core-rs` contains `sase-core-revision.txt`.

   It merges at most once per UTC day, after the daily core cut.

2. Release-please then tags `v0.19.0` as a draft release, and `release.yml`
   (`mode: release`) runs the train:
   - It builds and seals the candidate.
   - It qualifies the candidate by calling `ci.yml` at the tag against the published
     core. The inventory `.github/release-qualification.json` makes `lint`,
     `test (3.12|3.13|3.14)` and `exercise` blocking. `visual-test`, `perf-floors`,
     `tutorial-tests` and `ace-page-group-isolation` are advisory.
   - It runs four exercise legs (Linux and macOS, each with the wheel and the sdist).
   - It publishes to PyPI, confirms the upload, and un-drafts the release.
3. Merge authority turns on only through the E2 cutover gate. Phase **sase-1oe.4**
   raises that gate after a green master rehearsal, and a human approves it. The same
   approval makes the `pypi` environment tag-only and relaunches sase-1n6.10, which
   proves the first steward release. No phase in this epic changes those settings or
   merges a PR by hand.

So "full check green" means three things: Master Gate is green on master, Full CI
(`full.yml` → `ci.yml`) is green on master, and a `release.yml` rehearsal at master is
green. The rehearsal runs the same qualification and exercise jobs the real release will
run.

## Evidence (observed 2026-10-30, about 16:00 UTC, master `72b683eebb`)

| #   | Blocker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Phase                                     |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 1   | **The core-completeness matcher rejects maturin 1.16 macOS wheels.** `tools/_release_core.py:146` `matches_suffix` uses `filename.endswith(suffix)`. The live 0.43.1 macOS file is `sase_core_rs-0.43.1-cp312-abi3-macosx_10_12_universal2.macosx_10_12_x86_64.macosx_11_0_arm64.whl`, so `universal2.whl` never matches and `complete_files_for_version` returns `None`. `tools/release_candidate:152` has the same literal rule. This feeds `ratchet_core_pin`, the steward's core facts, the train's core-ensure, and `build-core.yml` in published mode. sase-core already fixed the producer side in `.github/scripts/pypi_release_files.py` `matches` (commit `7300a471`). sase-1oe.4 note #9 recorded the defect, and nobody owns it. | core-match                                |
| 2   | **No published core contains master's core revision.** sase-core v0.43.2 (tag `0ec67fdc`) contains pin `17e79c8c` and is merged. Its push run 41077488562 built all five dists, but `publish to PyPI` failed on a Sigstore Rekor `HTTP 502` while attesting. PyPI's newest core is 0.43.1, which predates the pin. The steward reports `core_lag` with its cut suppressed (authority off). Master pins `sase-core-rs==0.42.7`, and #322's `release-core-floor-smoke` is red.                                                                                                                                                                                                                                                                 | core-publish                              |
| 3   | **Master Gate is red on every recent push** (runs 41059468172 through 41080913472).<br>- `lint`: `_lint-toobig` reports `src/sase/agents_sync/publication_state_repair.py` at 1790 lines (limit 1000). The toobig routine owns this split (`toobig-8w.publication_state_repair.0`, queued), and so does task sase-1q1.<br>- The shard reds are listed below.                                                                                                                                                                                                                                                                                                                                                                                 | cli-reds, tui-reds, harness-reds, full-ci |
| 4   | **Full CI has been red on every run since at least 2026-10-28.** The last green run was 40189680772, on 2026-10-25. Its extra reds are listed below.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | cli-reds, tui-reds, harness-reds, full-ci |
| 5   | **No rehearsal since the resolver fix.** The last `release.yml` rehearsal, 40234446645 (2026-10-25), was red in qualification and in both wheel legs. sase-1oe.1 has since fixed the wheel-leg resolver, but no rehearsal has run since.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | rehearse                                  |
| 6   | **The cutover is stuck behind a stale poll.** sase-1oe.4 sits in a 36h zero-weight poll: monitor `c6xp53d5fv86` (member `sase-1oe.4--mon-2`, label "sase-1oe.4 core publication"), started about 14:40 UTC on 2026-10-30. The poll runs `tools/ratchet_core_pin --check --fetch --ref origin/master` from its own checkout. `--ref` refreshes the inputs but not the checker code, so the poll keeps the old matcher until it times out. sase-1n6.10's monitor waits for the settings flip.                                                                                                                                                                                                                                                  | rehearse (`wake_cutover`), watch          |

**Master Gate shard reds** (run 41080913472, Python 3.12):

- `tests/agents_sync/test_cli.py::test_sync_help_distinguishes_full_cached_and_refresh_modes`
  asserts the substring `-k, --retry-key`. Only Python ≥3.13 argparse prints that form
  for an option with a metavar; 3.12 prints `-k KEY, --retry-key KEY` (sase-1nv.5.1 note
  #1).
- `tests/pager/test_app_navigation.py::test_past_eof_line_clamps_with_information_toast`
  and `::test_past_eof_end_line_clamps_silently` error at teardown with
  `AttributeError: 'PagerScreen' object has no attribute 'notify'`.
- `tests/workspace_provider/test_rescue/test_nested_preservation.py::test_enumeration_failure_is_refused`
  lets a raw `PermissionError` escape instead of `PreservationError`. This happens on
  3.12 and 3.13 only.
- `tests/test_agent_load_cli.py::test_json_shape_carries_report_provenance_and_metadata`
  and `::test_cli_json_flag_emits_exact_report` hardcode machine `athena`, but the
  runner reports its own hostname. Task sase-1ov tracks this.
- `tests/footprint/test_tui.py::test_bump_microbench_stays_within_two_microseconds`
  measures a mean of 3.1–8.2 µs on runners against its 2 µs bound. It fails
  intermittently.

**Full CI-only reds** (run 41059821743):

- `tests/completion/test_kind_coverage.py::test_every_value_slot_is_kinded_choiced_or_hinted`
  reports `agent/sync:state_after` and `agent/sync:state_limit` as uncaptioned. Two
  tests in `tests/completion/test_snapshot.py` report the completion spec as out of
  sync.
- All five tests in `tests/test_suite_gate_scoped_integration.py` fail. Their scratch
  repo lacks `tests/_agent_artifact_marker_audit_helpers.py`, which
  `tests/_test_selection_rules.py:17` now imports, so `tools/run_pytest` raises
  `ModuleNotFoundError`.
- `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
  reports 2851 `sase.*` modules against the 2851 cap.
- In `visual-test`, `test_tool_runs_header_chips_png_snapshots` timed out waiting for
  its `✓` sentinel. It was also red on 2026-10-28.

Full CI runs from the last three days also showed reds that have since been fixed or may
recur:

- the contract manifest;
- `tests/test_justfile_lint_lint.py`;
- the monitor-start tests;
- `tests/test_agent_artifact_directory_operation_audit.py`;
- `tests/footprint/test_tui.py::test_tui_contract_pilot_records_each_seam_once`;
- `tests/test_sudo_acceptance_credentials.py`;
- `tests/sdd_store/test_sdd_git_streaming.py`.

Not blockers: open PRs #319 (stale core-pin ratchet), #320 and #323 (shard timings), and
#301 (an external contributor fix) gate nothing.

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
  - Do not edit `src/sase/agents_sync/publication_state_repair.py` while a toobig agent
    for it is queued or running (`sase agent list` shows
    `toobig-*.publication_state_repair.*`).
- **Fix waits, fixtures, and products, never tolerance.** Do not raise timeouts,
  budgets, caps or perf bounds. Do not add retries around assertions. Do not skip or
  xfail. Do not loosen pixel comparison or delete assertions.
  - Wait on the semantic state an assertion is about, using
    `sase.ace.testing.wait.wait_for` or the file's existing helper.
  - If the product settles late or inconsistently, fix the product.
  - A perf test may change how it measures (for example, best-of-N batches instead of
    one mean) only when the product did not regress. It never gets a looser bound.
- **Goldens.** Never copy CI candidate PNGs into the tree. Regenerate a golden only
  after its test is deterministic.
- **Rust boundary.** Shared domain behavior belongs in sase-core
  (`rust_core_backend_boundary`). None of these fixes is expected to need a Rust change.
  If one does, work through `sase repo open sase-core` and run that repo's own checks
  there.
- **Verify.** Run `just fix`, then `sase tool run check`, in every checkout you changed.
  Do not run `just check-full` (`decisions:check-full-is-explicit`): Full CI is this
  epic's exhaustive signal. The CI `lint` job enforces `toobig`, so do not grow a file
  past its line limit.
- **Hand waits to monitors.** Hand every long command to `/sase_monitor` with an
  explicit deadline. That covers verification, CI waits, and PyPI waits. Never end a
  turn waiting.
- **No merges, PRs, settings changes, or core cuts.**
  - Never merge #322 or any sase-core PR.
  - Never run `gh variable set`, and never edit the `pypi` environment.
  - Never dispatch a sase-core cut (`release-plz.yml`). Sibling epics sase-1oe and
    sase-1pi run under the approved `core_cut = wait`.

  Allowed: `gh workflow run full.yml` (full-ci only); `gh workflow run release.yml` with
  `mode=rehearse` (rehearse only); and `gh run rerun --failed` where a phase says so.

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

## Phase core-match

**Repo:** sase. **Goal:** sase's consumers count a sase-core-rs release as complete when
its macOS wheel carries a compressed platform-tag set. They use the same rule as
sase-core's producer.

1. Open sase-core (`sase repo open sase-core -r '<why>'`) and read `matches` in
   `.github/scripts/pypi_release_files.py`. It checks the wheel's name and version, then
   accepts the wheel when any dot-separated platform tag, plus `.whl`, ends with the
   expected suffix. The sdist must equal `stem + suffix` exactly.
2. Port that rule into `tools/_release_core.py` `matches_suffix`. Keep these unchanged:
   the `EXPECTED_DIST_SUFFIXES` parsing, the yanked-file handling, and the completeness
   requirement (`universal2.whl` stays required). Do not weaken completeness.
3. Make the missing-suffix check in `tools/release_candidate` (around lines 140–156) use
   the same rule:
   - Import it from `_release_core` if the script's sibling-import layout allows that.
   - Otherwise, mirror the rule there with a comment naming the canonical copy.

   Then audit every other check that matches core dist filenames and fix only the same
   literal-suffix mistake. Check `tools/_release_metrics.py`,
   `tools/_release_watch_health.py`, `tools/check_family_install`,
   `tools/exercise_release`, and the published-core verification in
   `.github/workflows/build-core.yml`.

4. Add tests next to the existing tests for these tools. They must cover:
   - the exact 0.43.0 macOS filename
     (`…macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl`) and the
     exact 0.43.1 filename
     (`…macosx_10_12_universal2.macosx_10_12_x86_64.macosx_11_0_arm64.whl`), both
     matching `universal2.whl`;
   - a wheel whose tag set lacks `universal2` (for example, `macosx_11_0_arm64` only),
     which must not match;
   - a different version or project name, which must not match;
   - an exact sdist match;
   - a yanked compressed-tag wheel, which still does not count.

   Exercise both `complete_files_for_version` and the `release_candidate` path.

5. **Local proof.** Run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master` and record its
   exit code and output. With the fix, 0.43.1 must count as complete: `newest_complete`
   must be at least 0.43.1.
6. Run `just fix` and `sase tool run check`. Then append a note on sase-1oe.4. It says
   that the consumer matcher defect from note #9 is fixed by phase `<this-phase-id>`,
   and that the commit can be found with
   `git log origin/master --grep 'SASE_BEAD=\[<this-phase-id>\]'`.

**Done when** the new tests pass, the proof output is in the close note, and the repo
checks pass.

## Phase core-publish

**Repo:** sase, plus read-only checks in sase-core. **Goal:** a complete sase-core-rs on
PyPI contains master's `sase-core-revision.txt`, and master pins it exactly.

1. **Confirm core-match landed.** Its commit must be on `origin/master`
   (`git log origin/master --grep 'SASE_BEAD=\[<core-match-id>\]'`). Re-fetch for up to
   about 5 minutes if a host push is still landing. Run every command below from a
   checkout that contains that commit. If the commit is missing, or core-match closed
   `canceled`, close this phase `-R canceled` with the evidence.
2. **Check the pin.** Run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: master already pins a complete containing core; skip to step 5;
   - exit 2: a ratchet is pending; go to step 4;
   - exit 3 with `CORE_LAG`: go to step 3.
3. **Get a containing core published.**
   1. In the opened sase-core checkout, find the oldest tag that contains the pin
      (`git tag --contains <pin>`). At planning time that is `v0.43.2` → `0ec67fdc`.
      Find its `Release-plz` push run with
      `gh run list -R sase-org/sase-core --workflow release-plz.yml --event push`. The
      run is titled `chore: release v0.43.2 (#341)`, and at planning time it
      was 41077488562.
   2. Rerun it once if every one of these holds:
      - only the upload or attestation steps failed (`publish to PyPI`);
      - the failure is transient infrastructure (an HTTP 5xx from Sigstore, Rekor or
        PyPI, or a network timeout);
      - the run is still at attempt 1;
      - no newer attempt or run for that tag is queued or running.

      The rerun is `gh run rerun <id> --failed -R sase-org/sase-core`, and you get one.
      If the failure is deterministic (a build, dist-gate or twine error), do not rerun.
      Record a `PROPOSED FOLLOW-UP:` with the job ID and log lines, and close
      `-R canceled`.

   3. If no tag contains the pin yet (another agent advanced it after v0.43.2), wait
      through sase-core's next scheduled daily cut (`41 7 * * *`). Never dispatch one.
   4. Hand the wait to `/sase_monitor` with a follow-up (`--next`). Allow about 6h after
      a rerun, or about 36h for a daily cut. Fetch first, then poll
      `python3 tools/ratchet_core_pin --check --fetch --ref origin/master` every 300 s
      from this phase's own checkout, which has the fixed matcher, until the exit code
      leaves 3. In the follow-up, rerun the check at once.
   5. If the rerun fails again or the wait times out, record the run and job evidence as
      a `PROPOSED FOLLOW-UP:` and close `-R canceled`.

4. **Ratchet master.** Run
   `python3 tools/ratchet_core_pin --allow-transitive-lock-refresh`, then `uv lock`.
   - The diff must touch only `pyproject.toml` and `uv.lock`, and should give
     `sase-core-rs==0.43.2` (or the version step 3 produced).
   - Keep `sase-core-revision.txt` unchanged.

   Then reproduce the `release-core-floor-smoke` job from `.github/workflows/ci.yml`:
   1. Create a fresh `uv venv --python 3.12` outside the repo.
   2. Run `uv pip install -e '.[dev]' "sase-core-rs==<X>"`.
   3. Run `tools/check_sase_core_rs_bindings`, then `tools/validate_sase_core_rs`.

   It must report 0 missing bindings. Paste the output into your close note.

5. **Notes.**
   - On sase-1pi.1, record the publication evidence: version, peeled tag SHA, and all
     five PyPI filenames. Its macos-core criterion ("a core containing the fix
     publishes") is met by 0.43.1 and later.
   - On sase-1oe.4, record the complete containing core and that this phase ratchets
     master's exact pin.
6. Run `just fix` and `sase tool run check`, then land.

## Phase cli-reds

**Repo:** sase.

1. **Sync help.** Make `test_sync_help_distinguishes_full_cached_and_refresh_modes`
   assert a form that every supported Python (3.12–3.14) prints, with the same meaning.
   For example, assert that `-k` and `--retry-key` share one option line, or render the
   expected line with argparse's own formatter. Check the test's sibling assertions for
   the same metavar-dependent form.
2. **Completion.** Give the `sase agent sync` value slots `state_after` and
   `state_limit` a ValueKind, choices, or a hint, following the three ways the
   kind-coverage failure message lists. Then regenerate the checked-in completion
   snapshot with the recipe the snapshot test names, and run `tests/completion/`.
3. **Agent load** (task sase-1ov). Pin `local_machine_label` the way
   `tests/test_machine_capacity_report.py`'s autouse monkeypatch does, or derive the
   expected label from the live one. Name sase-1ov as resolved.
4. **Re-inventory.** Look for any other red in `tests/agents_sync/`,
   `tests/completion/`, or the CLI help and docs tests, using the newest Master Gate and
   Full CI runs. Fix each one that no live agent owns.
5. **Proof.**
   - Run each repaired node 20 times serially.
   - Run each changed file 3 times under `-n 8`.
   - Run the help test explicitly under Python 3.12, which is what Master Gate uses, and
     under one newer Python.
   - Run `sase tool run check`.

## Phase tui-reds

**Repo:** sase. Read `tui.md`, and `ace.md` too if you touch `src/sase/ace/`.

1. **Pager.** Find what calls `.notify` on a `PagerScreen` during `test_past_eof_*`. The
   past-EOF clamp toast probably calls it on an object that has no `notify` after the
   PagerView split (sase-1eg.1). Fix the product so the toast goes through an object
   that owns notifications. Do not make the test tolerate the missing attribute.
2. **Footprint.** Handle every red in `tests/footprint/`, including
   `test_bump_microbench_stays_within_two_microseconds` and, if it recurs,
   `test_tui_contract_pilot_records_each_seam_once`.
   - Check `git log` on the measured code for a cost regression, and bisect locally with
     the bench.
   - If the product regressed, fix the product.
   - If the bench is measuring runner noise, make the measurement robust (warm-up plus
     best-of or median batches, matching the other benches in the file). Never raise the
     2 µs bound.
3. **Import budget** (2851 against a 2851 cap). Follow the failure message: run
   `tools/tui_import_closure --diff <base>` to find the modules that newly entered the
   TUI app's import closure, then defer them with lazy imports at their use sites. Never
   raise the cap.
4. **Visual.**
   - `test_tool_runs_header_chips_png_snapshots` times out waiting for `✓`. sase-1ih
     (the tool-run visibility epic) may own it. If a live sase-1ih agent is working on
     it, leave it. Otherwise find why the sentinel never paints on CI (missing fixture
     state, time dependence, or a wrong sentinel) and fix it.
   - Then download the newest Full CI `ace-visual-artifacts` and handle every other
     drift. Classify each one as intended (traceable to a landed UI commit), a race, or
     host-dependent. Make races and host-dependent captures deterministic first.
     Regenerate only intended changes, with targeted
     `just fix-tui-screenshots -- <files>` first.
5. **Proof.**
   - Run each repaired node 20 times serially.
   - Run each changed file 3 times under `-n 8` alongside `tests/ace/tui/`.
   - Run the full `just fix-tui-screenshots --check` with `TZ=UTC` through
     `/sase_monitor`. It must report clean with `created=0 updated=0 stale=0` and no
     failed tests.
   - Run `sase tool run check`.

## Phase harness-reds

**Repo:** sase. You own every red outside cli-reds' areas (agent sync, completion, agent
load) and tui-reds' areas (pager, footprint, import budget, visual).

1. **Suite gate.** Fix the scratch repo that
   `tests/test_suite_gate_scoped_integration.py` builds so that it contains the
   selection layer's real import closure, including
   `tests/_agent_artifact_marker_audit_helpers.py`.
   - Prefer deriving the file set from the actual imports over extending a hand-written
     list, so the next helper module cannot break it again.
   - If that import is audit-only and does not belong in the selection layer, move it
     instead.

   Keep every assertion.

2. **Nested preservation.** On 3.12 and 3.13, a raw `PermissionError` from pathlib
   escapes enumeration. Find the pathlib behavior that differs from 3.14, then fix the
   product so any `OSError` during enumeration becomes the `PreservationError` that
   `test_enumeration_failure_is_refused` expects.
3. **Re-inventory.** Check the newest Master Gate run at the tip and the newest Full CI
   run. Fix every deterministic red you own. Start with the recent-red list in Evidence:
   - the contract manifest: `just refresh-contract-manifest`, then check the contract
     budget;
   - `tests/test_justfile_lint_lint.py`;
   - the monitor-start tests;
   - the artifact directory-operation audit;
   - sudo acceptance;
   - sdd git streaming.

   For a flake, look for a pause-then-assert or wall-clock race and fix it if you can
   identify it. Otherwise record the failing output and a hypothesis as a
   `PROPOSED FOLLOW-UP:` that names the existing owner bead.

4. **Proof.**
   - Run each repaired node 20 times serially.
   - Run each changed file 3 times under `-n 8`.
   - Run `sase tool run check`.

## Phase full-ci

**Repo:** sase. This phase sweeps up everything still red once every sibling fix has
landed.

1. **Confirm the sibling fixes landed.** Each dependency's commit must be on
   `origin/master`. A phase whose note says it needed no commit is fine. Re-fetch for up
   to about 5 minutes if a push is still landing. If a sibling closed `canceled`, its
   remaining reds become yours.
2. **Toobig.** If `_lint-toobig` still fails at the tip and no toobig agent for that
   file is queued or running, split the file yourself following task sase-1q1, keeping
   behavior and import paths. Then name sase-1q1 as resolved.
3. **Get a Full CI run.** Find a Full CI run on master whose `headSha` contains every
   dependency commit
   (`gh run list -R sase-org/sase --workflow full.yml --branch master`). If none is in
   flight or done, dispatch one with
   `gh workflow run full.yml -R sase-org/sase --ref master`. Hand the wait to
   `/sase_monitor` with `--next` and about a 3h deadline, for example
   `gh run watch <id> -R sase-org/sase --interval 120`.
4. **Triage.** Check every job that did not succeed
   (`gh run view --job <id> --log-failed -R sase-org/sase`), plus the Master Gate run at
   the same head.
   - Fix every deterministic red, in any area: the siblings are done.
   - For a flake, find the race. If you cannot, record a `PROPOSED FOLLOW-UP:` with the
     output.
5. **Proof** for every fix:
   - Run each repaired node 20 times serially.
   - Run each changed file 3 times under `-n 8`.
   - Run `sase tool run check`.

   You need not wait for a second Full CI. The rehearse phase's qualification reruns the
   same `ci.yml` lanes on a tip that includes your commit.

**Done when** the Full CI run that contains every dependency commit is either green, or
every deterministic red in it is fixed locally with the proof above. Close `-R canceled`
only if a deterministic red you own remains unfixed. List it in the close note.

## Phase rehearse

**Repo:** sase. Read the `/sase_monitor` skill first. This phase changes no files.

1. **Prerequisites.** core-publish and full-ci must be closed `done`, and their commits
   must be on `origin/master`. Re-fetch for up to about 5 minutes if needed. If either
   one closed `canceled`, close this phase
   `-R canceled -r "rehearsal not started: <phase> canceled"` with its close note.
2. **Pin.** In a refreshed checkout, run
   `python3 tools/ratchet_core_pin --check --fetch --ref origin/master`:
   - exit 0: no override;
   - exit 2: add `-f core_version=<core-ensure pick>`, as sase-1oe.4's cutover step 1
     does, and record why;
   - exit 3 (`CORE_LAG`, meaning the pin moved past the published core again): close
     `-R canceled` with the evidence.
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
   - the core-match commit;
   - that its running poll uses the pre-fix matcher from its old checkout.
7. **Wake the cutover** (accepted rehearsals only):

   > [!decision] wake_cutover = wake
   >
   > Act only if sase-1oe.4 is still IN_PROGRESS and its active monitor is the
   > core-publication poll. Check with `sase monitor list` and `sase monitor show <id>`:
   > the label is "sase-1oe.4 core publication" and the command runs
   > `ratchet_core_pin --check`.
   >
   > 1. Start step 6's note with this line: "WAKE: fast-forward this phase's checkout to
   >    origin/master before running ratchet_core_pin; the old checkout's matcher
   >    rejects maturin 1.16 macOS filenames."
   > 2. Run `sase monitor stop <monitor-id>`, then `sase monitor resume <monitor-id>`.
   >    Stop launches no follow-up. Resume launches the recorded follow-up from the
   >    frozen result without rerunning the command.
   > 3. Confirm with `sase agent list` that a sase-1oe.4 follow-up is queued or running.
   >    Record the monitor ID and the outcome in your close note.
   >
   > Retry a failed stop or resume at most once. Then record it and move on: sase-1oe.4
   > still wakes at its own timeout. If sase-1oe.4 is closed, or is waiting on something
   > else (its own rehearsal, a steward verdict, a release-PR sync), leave it alone.

   > [!decision] wake_cutover = leave
   >
   > Write only step 6's note. sase-1oe.4 wakes when its poll times out, around
   > 2026-11-01 02:40 UTC.

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
  unpublished sase-core commit brings core lag back. The steward then waits for
  sase-core's daily 07:41 UTC cut. rehearse rechecks the pin, and the watch reports it.
- **Late wheel-leg failures.** The rehearsal may surface failures in wheel-leg steps CI
  has never proven: the N-1 install, the fakey launch, or the upgrade. rehearse stops
  with the evidence, and the lander fixes them.
- **The human cutover gate.** Until someone approves the E2 cutover gate, the steward
  never merges. The watch then ends at its deadline and the lander nests another watch.
  Approve the gate when sase-1oe.4 raises it.
- **Master churn.** Many epics land on master every hour, so new reds can appear after a
  phase closes. full-ci, the watch's `FULL_CI_RED` exit, and the lander absorb that.

## Non-goals

- Making ci_watch merge sase release PRs again. The steward handover is the design.
- E2's memory decisions, and closing or editing any E2 phase.
- `publish.yml`'s `sync-release-metadata` reds during `CORE_LAG`. They heal once a
  containing core publishes.
- The `just check-full` cost budgets (sase-j0).
