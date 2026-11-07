---
tier: epic
title: Turn bob-cli CI green on master
goal: 'bob-cli''s `CI` workflow on master is green at the tip, with lint, test (ubuntu-latest),
  test (macos-latest), and package all succeeded and none cancelled. Green evidence
  is published as `ci_evidence`, or, after a very large and documented effort, a research
  report says exactly what is still needed. The land agent then closes phase sase-1rq.3
  of epic sase-1rq.

  '
phases:
- id: lint-incremental
  title: Make the install_ui EarnedEta test hermetic to CARGO_INCREMENTAL
  depends_on: []
  size: small
  description: 'lint-incremental: fix the one deterministic lint-job failure. In the
    check-scripts step, EarnedEtaTests.test_seeded_history_shows_bar_range_plan_and_summary
    seeds history rows with incremental=True, but CI''s rust-cache sets CARGO_INCREMENTAL=0,
    so no row matches and the "usually" ETA text never renders. Make the test independent
    of the caller''s CARGO_INCREMENTAL, then prove it with and without the variable.

    '
- id: mac-known-failures
  title: Fix the known macOS-only test failures and the Google sign-in hang
  depends_on: []
  size: medium
  description: 'mac-known-failures: on the mac host, reproduce and fix the macOS-only
    failures already seen in CI. Lib: one case-insensitive-APFS test and three /var
    vs /private/var path tests. CLI: clipboard_status picking Alfred over the command
    provider. gcal_auth: 11 login tests each stall about 600 s because choose_flow
    picks the browser flow on macOS, so the paste-scripted tests wait out CONSENT_WINDOW.
    Make every Google login test family (gcal_auth, gmail_shell, and gkeep_auth if
    affected) select its flow deterministically on every OS.

    '
- id: mac-full-suite
  title: Run the full macOS CI test set to green on the mac host
  depends_on:
  - mac-known-failures
  size: large
  description: 'mac-full-suite: on a tree that contains the mac-known-failures fixes,
    run every command CI''s macOS job runs on the mac host, with a per-binary watchdog.
    Every test binary after gcal_auth (all gmail_* suites, gcal_*, gkeep_*, randomize,
    tasks_parity, tasks_real_vault_parity, doctests) has not run on macOS since about
    2026-11-02. Fix every remaining failure or hang, and show that the macOS test
    step fits well inside its unchanged 45-minute job timeout.

    '
- id: ci-green
  title: Confirm green CI on the master tip and publish evidence
  depends_on:
  - lint-incremental
  - mac-known-failures
  - mac-full-suite
  size: medium
  description: 'ci-green: with every fix landed, dispatch and watch CI on the master
    tip through /sase_monitor, and fix any residual red job (iterating across monitor
    turns). Then publish the ci_evidence variable and the Markdown evidence table.
    Fall back to the research report only under the parent contract''s very-large-effort
    rule.'
proposed_by: bbugyi200.athena.sase-1rq.3
proposed_with: claude/opus
parent_bead: sase-1rq.3
create_time: 2026-11-07 08:10:10
status: wip
bead_id: sase-1rq.3.1
---

- **PROMPT:** [prompts/202611/bob_cli_ci_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/bob_cli_ci_green.md)
- **PARENT:** [202611/red_ci_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/red_ci_repair.md)
- **BEAD:** [sase-1rq.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.3.1.md)

# Plan: Turn bob-cli CI green on master

This is the child epic of phase `sase-1rq.3` ("Turn bob-cli CI green") of epic
`sase-1rq`, `plan:202611/red_ci_repair.md`. Read that parent plan's **Contract Shared By
Every Phase** before starting. Its working rules, landing loop, evidence format, and
research-report fallback all apply here unchanged. The parent epic's DECISIONS are
final:

- `test_quarantine = quarantine_flakes`: a test proven flaky on an unchanged tree may be
  skipped. Timeouts, job lists, and lint levels stay exactly as they are.
- `release_merge = gate`. bob-cli has no release PR, so this decision should never come
  into play. Never dispatch `release.yml`.

Repository: the `bob-cli` SASE project. Open it with `sase repo open bob-cli` and work
only in the path it prints. Read its `AGENTS.md` first. bob-cli's GitHub repo is
`bobs-org/bob-cli`, and its default branch is `master`.

## What The Planner Verified (2026-11-07)

The parent plan's leads for this phase were partly wrong. These facts were checked
against real runs and local reproduction:

- **Last fully green master CI was run `40586019117` on `6b0a30bf` (2026-10-27).** Every
  completed master run since has been red. At planning time the tip was `dbd30ec0`, run
  `42364277532`: `lint` failure, `test (macos-latest)` cancelled at 45m,
  `test (ubuntu-latest)` success, `package` success.
- **Clippy is not what fails the lint job.** `clippy::result_large_err` on
  `CaptureError`, `needless_else` at `src/native/task_cancel/prune.rs:119`, and the rest
  are **warnings**. `docs/develop/ci.md` and the justfile's `clippy_gate` comment list
  `result_large_err` as an advisory class that stays at its default level. Do **not**
  box `CaptureError`, add `allow`s, or change lint levels. That refactor is out of
  scope, and the land agent records the correction in its closing note.
- **The lint job actually fails in `just check-scripts`.** The failing test is
  `install_ui.tests.test_install_all.EarnedEtaTests.test_seeded_history_shows_bar_range_plan_and_summary`
  (`AssertionError: 'usually' not found`). Root cause: `_seed_bobcli_history` in
  `scripts/lib/install_ui/tests/test_install_all.py` seeds rows with
  `"incremental": True`. `history.incremental_for()` returns
  `os.environ.get("CARGO_INCREMENTAL") != "0"`, and Swatinem/rust-cache exports
  `CARGO_INCREMENTAL=0` in CI. So `estimate._row_matches_key` rejects every seeded row,
  and the UI prints "first timed cold build". To reproduce locally:
  `CARGO_INCREMENTAL=0 python3 -B -m unittest discover -s scripts/lib/install_ui/tests -t scripts/lib`
  gives 1 failure out of 146, and without the variable all 146 pass. It is the only
  failing test in that suite.
- **macOS is not one hung test.** In run `42364277532` the `Run tests` step was killed
  by the job timeout after these results:
  - `bob_cli` lib, `5465 passed; 4 failed`:
    - `native::capture_dependency_tasks::tests::interleaved_names_walk_in_sorted_order`:
      `A.md` and `a.md` collide on case-insensitive APFS
      (`assertion failed: paths.contains(&PathBuf::from("A.md"))`).
    - `native::gmail::link::tests::scan_skips_excluded_dirs_mail_folder_and_zorg`: got
      `["//private/var/folders/…/tasks.md"]`, expected `["tasks.md"]`.
    - `native::gmail::worker::tests::gmail_queue_drains_arrivals_mid_run_and_excludes_another_worker`
      and `…::gmail_kick_spawns_scoped_command_and_keeps_job_on_spawn_failure`: got a
      `/private/var/…` path, expected `/var/…`.
  - `tests/cli` (`bob-obsidian --test cli`), `3268 passed; 1 failed`:
    `clipboard_status::healthy_command_provider_reports_only_metadata`, where
    `selected_provider` was `"Alfred"` and `"command"` was expected, even though
    `BOB_CLIPBOARD_HISTORY_CMD` is set.
  - `tests/gcal_auth.rs`: login tests ran more than 60 s and then FAILED about 9.5
    minutes later. The same 11 tests (`login_account_flag_overrides_config`,
    `login_account_picker_accepts_a_number`, `login_dash_c_accepts_legacy_flat_file`,
    `login_dash_c_stores_the_client`, `login_discovers_a_downloaded_client`,
    `login_guides_pasted_client_json_and_adds_json_fields`,
    `login_human_transcript_stays_on_stderr_and_mentions_remote_writer`,
    `login_reuses_legacy_default_client_without_prompt`,
    `login_self_heals_a_stale_saved_grant`,
    `login_suggests_config_when_bob_accounts_remain_ambiguous`,
    `login_without_manual_never_runs_interactive_auth_add`) failed the same way in runs
    `41226270223` (10-31) and `41541096646` (11-02). There `gcal_auth` alone took 1802
    s, and the panics were at `tests/gcal_auth.rs:30`. This started with `00c44ab4`
    "feat(gcal): bob-owned login sign-in with browser and paste flows" (2026-10-30).
  - Strong lead for the hang: in `src/native/google/signin.rs`, `FlowEnv::detect` sets
    `is_macos: cfg!(target_os = "macos")`, and `choose_flow` only picks `Flow::Paste`
    when the run is manual, on SSH, or `!is_macos && !has_display`. So on a macOS runner
    the paste-scripted tests get `Flow::Browser`. `spawn_opener` then runs `open <url>`,
    and the login waits `CONSENT_WINDOW = 600 s` for a loopback callback that never
    comes. On Linux CI there is no `DISPLAY`, so the same tests get the paste flow and
    pass.
  - Every binary after `gcal_auth` never ran: `gcal_cli`, `gcal_diagnostics`,
    `gcal_lifecycle`, `gcal_list`, `gcal_sync`, all `gkeep_*`, every `gmail_*` (those
    suites are new since 2026-11-05), `randomize`, `tasks_parity`,
    `tasks_real_vault_parity`, the doctests, and the follow-up
    `Run archive-move presenter cases` step. In the 11-02 run, the `gcal_*` and
    `gkeep_*` binaries that existed then all passed on macOS.
- Job timing on the 11-07 macOS runner: `Lint macOS-only targets` took about 5 min. The
  test build took about 18 min before the first test output, the lib tests took 90 s,
  and `tests/cli` took 439 s. The last green macOS job (10-27) took about 14 min in
  total. `Swatinem/rust-cache` saves only after a successful job, so the macOS cache is
  stale and the first green run will compile more than usual.
- Many master push runs are `cancelled` by the workflow's concurrency group, because
  other agents push often. `workflow_dispatch` runs use their own concurrency group with
  `cancel-in-progress: false`, so pushes don't cancel them. `ci.yml` has only
  `contents: read` and publishes nothing, so dispatching it is safe.

## Rules For Every Phase

- **Root-cause fixes first.** Adapting a test whose assertion depends on host filesystem
  semantics counts as a root-cause fix: for example, probing case sensitivity of the
  temp dir, or comparing canonicalized paths. Quarantine does not. Any `#[ignore]`,
  `#[cfg_attr(target_os = "macos", ignore)]`, or skip of a deterministically failing
  test is forbidden. A test may be skipped only when it is proven flaky on an unchanged
  tree. Each such test gets a
  `sase bead note <your-bead> 'PROPOSED FOLLOW-UP: flake — <test path> …'` entry so the
  land agent can file its `flake` bead. Phase workers never create beads.
- If a fix shows that production code is wrong on macOS (for example, a path that should
  stay relative to the caller's vault root comes back canonicalized), fix the production
  code and add or adjust a test. Don't paper over it in the test.
- Keep `test (ubuntu-latest)` and `lint` green. Before landing, run `just fmt`,
  `just lint`, `just check-scripts`, `just check-release-tools`, and the `cargo test`
  binaries you touched on Linux (or `just check` for broad changes).
- Never change `.github/workflows/*`, timeouts, job lists, the `clippy_gate`, repo
  variables or secrets, or branch protection. Never force-push, and never dispatch
  `release.yml`.
- Land through host completion only: declare a commit for the `bob-cli` repo in
  `/sase_final`. Agents never commit or push to master themselves.

### Using The mac Host

Read the `tailnet` reference memory with `/sase_memory_read` before the first SSH. The
mac is `ssh mac` (user `bbugyi`, Kelly's MacBook Pro, Apple M1, 8 cores, 8 GB RAM, macOS
26.5). It already has `rustup` with toolchain `1.95.0`.

- The mac is best-effort and offline when its lid is closed. If it is unreachable,
  verify through bob-cli's existing pre-land flow, `scripts/dev/macos-ci-check`. It
  pushes a working-tree snapshot to a throwaway `ci/precommit/*` branch, waits for that
  exact-SHA macOS run, and deletes the branch. Run it under `/sase_monitor`, because a
  full run can take 45+ min.
- **Never touch `~/projects/...` on the mac.** Those are Bryan's live checkouts. Work in
  a fresh directory named for your bead, such as `/tmp/bob-cli-<bead-id>`. Ship your
  working tree with `git archive` or `rsync`, excluding `target/`, `.git/`, and
  `sase/repos/`. Use `CARGO_TARGET_DIR` inside that directory.
- **Disk is tight.** The data volume was 96% full, with about 18 GiB free, at planning
  time. Check `df -h /System/Volumes/Data` before building, and stop if less than about
  12 GiB is free. Delete your directory, including its target dir, before your phase
  closes. Never delete other agents' `/tmp` directories. The parallel `bob-mac-capture`
  phase (`sase-1rq.4`) may be using the mac at the same time, so run builds and tests
  under `nice -n 10` and report wall time.
- **SSH changes the code path under test.** An SSH session exports `SSH_CONNECTION` and
  `SSH_TTY`, which makes `choose_flow` pick the paste flow. The `gcal_auth` hang
  therefore does not reproduce over plain SSH. To mimic the CI runner, run tests with
  `env -u SSH_CONNECTION -u SSH_TTY -u SSH_CLIENT CARGO_INCREMENTAL=0 …`.
- **Don't let tests open real windows on the laptop.** With the SSH variables unset, the
  browser flow runs `open <google-url>` against the logged-in GUI session. For
  reproduction runs, put a shim directory first on `PATH` with a no-op `open` that only
  logs its arguments. It reproduces the CI hang, because the login still waits for the
  callback, without launching a browser. Use the shim only to reproduce; the fix must
  not depend on it.
- libtest has no per-test timeout. Wrap each test binary in a watchdog (for example a
  short Python `subprocess.run(..., timeout=…)` driver, or `gtimeout` if present), and
  use libtest's "has been running for over 60 seconds" lines to name hung tests.
- Local results don't always match the runner. The mac may have Alfred, Clipy, a real
  `~/Downloads`, or `gog` config that the runner lacks, so tests must isolate `HOME` and
  config themselves. When a mac result and the CI log disagree, trust the CI log and fix
  the isolation gap.

## Phase: lint-incremental

Make `test_seeded_history_shows_bar_range_plan_and_summary` hermetic to the caller's
`CARGO_INCREMENTAL`. Preferred fix: the install_ui test sandbox pins or removes
`CARGO_INCREMENTAL` in the environment it gives the child `install_all` process, the way
it already pins other host inputs. That keeps every install_ui test independent of the
runner. Acceptable alternative: seed `incremental` with the same predicate the code
under test uses. Don't change `history.incremental_for` semantics or the estimate
matching rules; production behavior is correct.

Verify:
`CARGO_INCREMENTAL=0 python3 -B -m unittest discover -s scripts/lib/install_ui/tests -t scripts/lib`
and the same without the variable (146/146 both ways). Then run the lint job's full
sequence, `just fmt && just lint && just check-scripts && just check-release-tools`,
with `CARGO_INCREMENTAL=0` exported. Land through `/sase_final`. This phase needs no CI
watch: the `ci-green` phase confirms the lint job.

## Phase: mac-known-failures

Fix, verify on the mac host (see **Using The mac Host**), and land:

1. `capture_dependency_tasks::tests::interleaved_names_walk_in_sorted_order`. The test
   writes both `a.md` and `A.md`, which are one file on case-insensitive APFS. Keep the
   intent (sorted, component-wise depth-first walk). Write and assert the `A.md` sibling
   only when a probe shows the temp dir is case-sensitive, or pick names that cannot
   collide while still exercising uppercase-before-lowercase ordering.
2. `gmail::link::tests::scan_skips_excluded_dirs_mail_folder_and_zorg`, plus the two
   `gmail::worker` tests. Find where the paths get canonicalized (`scan_vault_notes`,
   `enqueue`, or `relative_slashes`). If canonicalized absolute paths are the intended
   contract, canonicalize the test's expected paths and vault root. If callers expect
   paths under the vault root they passed in, fix production so it doesn't leak
   `/private/var/…` (the doubled `//private/…` from `relative_slashes` suggests a real
   strip-prefix bug). State which one you chose and why in your final response.
3. `tests/cli/clipboard_status.rs::healthy_command_provider_reports_only_metadata`. On
   macOS the report selects `"Alfred"` even though `BOB_CLIPBOARD_HISTORY_CMD` is set.
   The documented order in `src/native/capture_clip/history.rs` is environment, then
   config, then `auto`, which prefers Alfred, then Clipy. Work out whether the macOS
   path ignores the environment override (a production precedence bug: fix it) or
   whether the test fails to isolate `HOME` or config (fix the isolation). Check the
   other `clipboard_status` tests for the same macOS gap.
4. The Google login hang. Make every paste-scripted login test pick its flow
   deterministically on every OS and under any ambient `SSH_*` or `DISPLAY` environment.
   Cover `tests/gcal_auth.rs` and its `tests/gcal_support` harness, the `gmail_shell`
   login tests and `tests/gmail_support`, and `gkeep_auth` if it shares the flow. Good
   shapes:
   - The shared test env scrubs `SSH_CONNECTION`, `SSH_TTY`, `SSH_CLIENT`, `DISPLAY`,
     and `WAYLAND_DISPLAY`, then injects exactly what each scenario needs.
   - Tests that mean to exercise the paste flow select it explicitly, for example
     through the existing `-m` / manual path or an injected `FlowEnv`.
   - Tests that mean to exercise the browser flow always use the `FakeBrowser` /
     `BOB_GCAL_BROWSER` override.

   Keep the existing transcript assertions meaningful. Some tests assert reason text
   such as "no desktop session", so update them rather than weakening them. No test may
   reach the real `open` or `xdg-open`. Consider a harness guard that fails fast, for
   example a PATH shim `open` / `xdg-open` that exits non-zero with a clear message, so
   a future regression fails in seconds instead of 600 s. Don't change `CONSENT_WINDOW`
   or production flow selection unless you find a real production bug; macOS desktop
   users should keep the browser flow.

On the mac, under the CI-like environment, run the lib filters for items 1–2,
`--test cli clipboard_status`, `--test gcal_auth`, `--test gmail_shell`, and
`--test gkeep_auth`. All must pass, and `gcal_auth` must finish in under a minute. Then
run the Linux checks from **Rules For Every Phase**. Record in your final response every
test command and its wall time. Land through `/sase_final`. If you also notice failures
in other binaries, fix them only when they share one of these root causes. Leave the
rest to `mac-full-suite` by recording them in a bead note.

## Phase: mac-full-suite

Start from the current `origin/master`, which must contain the `mac-known-failures`
fixes. Run the exact commands from CI's macOS job on the mac host, under the CI-like
environment and a per-binary watchdog:

1. `just lint` (the "Lint macOS-only targets" step).
2. `cargo test --locked --workspace --no-fail-fast`.
3. `cargo test --locked --workspace --lib archive_move_presenter -- --nocapture --test-threads=1`.

Record an inventory of every test binary: test count, result, and wall time. Fix every
failure and hang under **Rules For Every Phase**, re-running only the affected binaries
between fixes and the full set once at the end. Expect the new `gmail_*` suites to be
the main risk. They never ran on macOS, and they combine temp-dir paths, subprocess
fakes (`tests/gmail_support/fake_gog.py`), and Google sign-in.

Budget check: from your measured local times and the runner times above, estimate the
macOS job's total wall time on the runner. If even an all-passing run would come within
about 10 minutes of the unchanged 45-minute `timeout-minutes`, treat the slowness as a
defect: find the slow tests, such as long sleeps or real network or consent waits, and
fix them. Never raise the timeout or drop a step.

Before landing, optionally confirm the whole macOS job on the real runner with
`scripts/dev/macos-ci-check` under `/sase_monitor`. This is the strongest pre-land
signal, and it is required if the mac host became unreachable mid-phase. If this phase
finds more independent failure families than one agent can finish, land the strict
improvements you have and record the remainder in bead notes on this phase (one
`PROPOSED FOLLOW-UP:` per family, with test names and leads), so the `ci-green` phase
and the land agent see it.

## Phase: ci-green

1. Fetch `origin/master` and confirm that every earlier phase's fix commit is on it.
   Read the newest `CI` runs on master:
   `gh run list --repo bobs-org/bob-cli --workflow ci.yml --branch master --limit 10`.
2. Dispatch a verification run on the tip with
   `gh workflow run ci.yml --repo bobs-org/bob-cli --ref master`, so concurrent pushes
   can't cancel it. Then start a zero-weight `/sase_monitor` (`-w 0`, labels
   `WATCHING CI` → `WATCHED CI`) that polls that run to completion. Give it a `--next`
   telling the follow-up turn to read every job's conclusion and either iterate or
   finish.
3. If any job is red, read the failing log, reproduce it (on the mac host for macOS),
   fix it under **Rules For Every Phase**, land it through `/sase_final`, and repeat
   from step 2 in a later turn. Apply the parent contract's scope rule for new
   regressions from other agents' commits: fix quick ones, and otherwise record a bead
   note and cite it in the evidence.
4. Green means one CI run on a master SHA at or after the last fix commit, with all four
   jobs `success` and none cancelled or skipped. A push run qualifies as well as a
   dispatch run. Note first attempt versus rerun. A run that went green only on a rerun
   needs a `PROPOSED FOLLOW-UP: flake — …` bead note per test involved.
5. In the confirming turn, publish `ci_evidence` through `/sase_var` with the parent
   contract's fields: `repo` `bobs-org/bob-cli`, `default_branch` `master`, `tip_sha`,
   `runs` (`workflow`, `run_id`, `url`, `head_sha`, `event`, `attempt`, `conclusion`,
   `completed_at`), `fixes` (landed SHAs with one-line summaries from every phase),
   `quarantined`, and `notes`. The notes must say that the clippy `result_large_err`
   warnings are advisory and were deliberately left alone. Repeat the evidence as a
   Markdown table in the final response, with the reproducing `gh run list` command.
6. Use the research-report fallback only under the parent contract's very-large-effort
   rule. The report goes in the **sase** project's research sidecar
   (`sase-org/sase--research`, opened with `/sase_repo`), and its path and `@research:`
   reference are published as `ci_report`.

## Land Agent

The land agent for this epic also finishes parent phase `sase-1rq.3`:

1. Confirm every phase bead in this epic is closed, and re-read the `ci-green` phase's
   evidence or report.
2. Refresh the evidence with
   `gh run list --repo bobs-org/bob-cli --workflow ci.yml --branch master --limit 5 --json databaseId,headSha,event,attempt,conclusion,createdAt,updatedAt`.
   If a newer master run is red for a reason that is in scope here, do not close
   `sase-1rq.3`. Record what remains as a note on it instead. Otherwise republish
   `ci_evidence` (or `ci_report`) through `/sase_var` in the land turn, so the turn that
   completes `sase-1rq.3` carries it.
3. Triage every `PROPOSED FOLLOW-UP:` note from this epic's phases into task beads
   through `/sase_new_task`. That includes one `flake` bead per quarantined test.
4. Run `sase bead epic-symbols sase-1rq.3` and resolve any leftovers. Then close
   `sase-1rq.3` with
   `sase bead close sase-1rq.3 --note "<tip SHA, run id, evidence summary or report ref>"`.
   The note must also record that the parent plan's clippy lead was mistaken: the
   warnings are advisory, and the lint job actually failed in `check-scripts`.
5. Never close epic `sase-1rq` or any other ancestor.
