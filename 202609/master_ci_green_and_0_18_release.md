---
tier: epic
title: Green sase master CI and ship sase 0.18.0 to PyPI
goal: "sase-core-rs 0.36.0 and sase 0.18.0 are live on PyPI and install cleanly from a
  fresh venv. They get there through the normal release path: Master Gate green on
  master HEAD, a green Full CI no older than 6 hours, the core-floor smoke on PR #299,
  and ci_watch merging #299. Full CI is restructured so that measurement and soak lanes
  can no longer block a release.

  "
phases:
  - id: core-macos
    title: Fix macOS path canonicalization in sase-core
    depends_on: []
    size: small
    description:
      "core-macos: in the linked sase-core checkout, canonicalize the deepest existing
      ancestor in launch_scratch_liveness normalize_path, make the managed_tmp_roots
      test expect canonical storage, and add a symlink regression test that reproduces
      the macOS /var vs /private/var bug on Linux."
  - id: core-release
    title: Cut and verify sase-core-rs 0.36.0 on PyPI
    depends_on:
      - core-macos
    size: small
    description:
      "core-release: drive sase-core release PR #315 to all-green CI including macOS,
      dispatch release-plz with dry_run=false, and verify that PyPI has every 0.36.0
      wheel plus the sdist."
  - id: core-pin
    title: Move the sase core source pin and stop ratchet PR pileup
    depends_on: []
    size: small
    description:
      "core-pin: bump sase-core-revision.txt to sase-core master so the 3 goal bindings
      and 16 tests clear, close the superseded core-pin-ratchet PRs, and make the
      ratchet workflow keep at most one open PR."
  - id: contract-drift
    title: Repair whole-repo contract and guard drift
    depends_on: []
    size: medium
    description:
      "contract-drift: fix the 15 mechanical failures. Add completion kinds and sync the
      spec, add the two schema properties, fix the AgentExecContext mock, and update the
      trash-limit and getting-started docs assertions. Review the two marker-path sites,
      and route the 7 system-clock sites through sase.core.time without touching the
      allowlist."
  - id: tab-completion
    title: Settle %tab completion fallout
    depends_on: []
    size: medium
    description:
      "tab-completion: decide whether the default main tab group is a leak when the
      agent_tabs flag is off, fix the product or the 11 agent/directive completion tests
      to match, and assert removed directive names explicitly."
  - id: admin-center-tabs
    title: Repair Admin Center tab-model fallout from the Tools pane
    depends_on: []
    size: medium
    description:
      "admin-center-tabs: reconcile the 6 Admin Center and Config Center tests (tab
      order, count, digit shortcuts, resume, cached open, blocked write) with the Tools
      tab added by sase-1bt.10, fixing the product wherever a documented invariant
      broke."
  - id: tui-scroll-settle
    title: Fix header half-page scroll and files Ctrl-J settle races
    depends_on: []
    size: medium
    description:
      "tui-scroll-settle: root-cause the asynchronous pin settle behind the header
      half-page scroll test (sase-1b8) and the files deck Ctrl-J timeout (sase-1a7), and
      fix them without blindly longer waits."
  - id: tui-resize-layout
    title: Fix chrome layout resize failures
    depends_on: []
    size: medium
    description:
      "tui-resize-layout: root-cause why a terminal resize sometimes has no effect in
      the two test_chrome_layout nodes (sase-1a6) and make recompose and popup reclamp
      settle deterministically."
  - id: toobig-lint-tail
    title: Split the two oversized modules and clear the masked lint tail
    depends_on: []
    size: medium
    description:
      "toobig-lint-tail: split src/sase/core/tool_run.py and src/sase/tool/executor.py
      below 1000 lines along existing seams, then run every CI lint-job step in order
      and fix what validate, validate-committed-plans, and build-check report now that
      they finally run."
  - id: ci-telemetry-split
    title: Move measurement lanes out of the release-gating Full CI
    depends_on: []
    size: medium
    description:
      "ci-telemetry-split: move test-cost, coverage-contexts, and contention into a new
      scheduled CI Telemetry workflow with realistic timeouts, run plain just test on
      3.13 in Full CI, give the 3.12 coverage leg headroom, and update the
      coverage-contexts fetcher, workflow tests, and docs."
  - id: perf-floors
    title: Fix the real perf-floors regressions
    depends_on: []
    size: medium
    description:
      "perf-floors: shorten the tmux socket directory (sase-18w), bring the
      smoke_sase_tool_runs harness cases up to the current tool-run contract, and fix
      the cached prompt-panel hint render regression behind the view-hints floor."
  - id: visual-lane
    title: Make the visual-test lane green without bulk acceptance
    depends_on:
      - tab-completion
      - admin-center-tabs
      - tui-scroll-settle
      - tui-resize-layout
    size: medium
    description:
      "visual-lane: reproduce just test-visual at HEAD after the TUI phases land, fix
      the semantic timeouts (the Config Center flags SUNSET wait, the narrow top-bar
      startup), and rebaseline only goldens whose drift traces to a deliberate UI
      commit."
  - id: green-master
    title: Integrate and observe green Master Gate and Full CI
    depends_on:
      - core-pin
      - contract-drift
      - tab-completion
      - admin-center-tabs
      - tui-scroll-settle
      - tui-resize-layout
      - toobig-lint-tail
      - ci-telemetry-split
      - perf-floors
      - visual-lane
    size: medium
    description:
      "green-master: re-inventory CI at HEAD, fix any failure that landed since the
      repair phases, watch Master Gate until it is green on HEAD, dispatch Full CI until
      it is green, and take one measured CI Telemetry run."
  - id: release
    title: Release sase 0.18.0 to PyPI through ci_watch
    depends_on:
      - core-release
      - green-master
    size: small
    description:
      "release: dispatch Publish so #299 ratchets to sase-core-rs>=0.36.0,<0.37.0, get
      the floor smoke green, let ci_watch merge #299 (or take the bounded one-time
      bypass), cut the release, and verify 0.18.0 from a clean venv."
proposed_by: bbugyi200.athena.0ti
create_time: 2026-09-28 07:09:20
status: wip
---

- **PROMPT:**
  [prompts/202609/master_ci_green_and_0_18_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/master_ci_green_and_0_18_release.md)

# Plan: Green sase master CI and ship sase 0.18.0 to PyPI

## Why this shape

Read the governing research first. Its evidence is dated 2026-09-28, and every phase
below builds on it:

```sh
sase artifact read research:202609/sase_master_ci_repair_and_pypi_0_18_release/sase_master_ci_repair_and_pypi_0_18_release.md "Governing evidence for the master CI repair and 0.18.0 release epic"
```

PyPI has served `sase` 0.17.1 since 2026-08-29. Release-please PR **#299
(`chore(master): release 0.18.0`)** has been open since 08-30. Three gates stand between
master and PyPI:

1. **`sase-core-rs` 0.36.0 must exist on PyPI.** #299's `release-core-floor-smoke`
   installs the exact published floor. Published 0.35.1 lacks 13 of the 741 bindings
   master requires. sase-core release PR #315 is blocked only by two macOS tests, and
   both fail deterministically on every run from one `/var` vs `/private/var`
   canonicalization bug.
2. **Master Gate must be green on HEAD.** At planning time (master `43cd823af`, Master
   Gate run 36408553789), lint and all 8 shards are red with **52 failing tests**. The
   count grows with each feature landing. Almost every cluster traces back to one
   feature commit from the last 72h that left its tests, allowlists, or schema stale.
3. **Full CI must be green and at most 6h old** for `ci_watch` to auto-merge #299. It
   has been red on every run since 08-29, and **three of its lanes cannot pass as
   configured**:
   - `test (3.13)` = `just test-cost` is cancelled at exactly 90 minutes;
   - `coverage-contexts` is cancelled at exactly 60 minutes;
   - `contention-test` kills its 2-CPU runner.

   `ci_watch` checks the workflow conclusion, so a telemetry timeout blocks a release
   exactly as a real regression would.

Three earlier epics repaired red lanes (`sase-th`, `sase-126`, `sase-10w`; `sase-10w`
aimed at a v0.17.2 release). Each one got master green, only for it to go red again
before a release could happen. The two causes were the Full CI structural timeouts above
and the landing rate, about 85 commits per day by direct push. This epic fixes the
structure (`ci-telemetry-split`) and ends with an integration phase (`green-master`)
that sweeps up whatever landed in the meantime. The landing rate itself is an operator
decision; see the next section.

The phases split into three tracks that run in parallel:

- **Core release track:** `core-macos` → `core-release`.
- **Master Gate repair track:** `core-pin`, `contract-drift`, `tab-completion`,
  `admin-center-tabs`, `tui-scroll-settle`, `tui-resize-layout`, `toobig-lint-tail`.
- **Full CI track:** `ci-telemetry-split`, `perf-floors`, then `visual-lane` once the
  TUI phases have landed.

Everything converges on `green-master`, and `release` follows.

## Operator actions and authorization (read before approving)

**Stop the line (user action, strongly recommended).** Pause launches and landings of
feature epics until `green-master` closes. The epics that caused most of today's
breakage are `sase-1bt` (tool runs, Tools pane), `sase-1bc` (agent tabs), and `sase-1bu`
(goals). At the current landing rate, about 20 new failures appeared in the hour the
research took. Without a pause, `green-master` is chasing a moving target.

**Approving this plan authorizes these outward-facing actions, and only these:**

1. Landing the `core-macos` fix on sase-core master, through the normal host finalizers.
2. Closing superseded bot PRs named `core-pin-ratchet-*` in the **sase** repo (#302–#318
   at planning time; sase's #315 is one of them, unrelated to sase-core's release PR
   #315) and deleting their branches. **Do not touch** #299 (the release PR), #300 (a
   stale shard-timings ratchet), or #301 (a contributor PR). The user decides about #300
   and #301.
3. Dispatching sase-core's `release-plz.yml` with `dry_run=false` once PR #315 is fully
   green.
4. Dispatching sase's `full.yml`, the new CI Telemetry workflow, and `publish.yml` with
   `publish_existing=false`, and rerunning failed CI jobs that died in setup.
5. **One-time only:** a manual merge of #299, strictly under the bypass criteria in the
   `release` section. If the user strikes this item during review, the `release` phase
   records status and stops instead of merging.

**Never:**

- run `publish.yml` with `publish_existing=true`, except to heal a partial upload of
  0.18.0 itself;
- hand-edit the `sase-core-rs` window in master's `pyproject.toml` (it belongs to
  `sync-release-metadata`; see `docs/rust_backend.md` "Who owns the published version
  window");
- weaken, skip, or widen `release-core-floor-smoke`;
- change chezmoi's `ci_watch` config.

## Ground rules for every phase

- **Re-inventory at your own HEAD before fixing anything.** The failure set moves with
  every landing, and some Master Gate shards die in setup on PyPI 503s
  (`tree-sitter-html`). A test that drops out of a run's list is not necessarily fixed.
  Use `gh run list --workflow "Master Gate" -L 5` and `gh run view <id> --log-failed`,
  then reproduce locally.
- **Tell deterministic failures from flakes** by rerunning on an unchanged tree,
  serially and under xdist load, before you change code. Fix root causes. Never go green
  by any of these:
  - `skip`, `xfail`, or longer timeouts without a diagnosis;
  - extending a guard allowlist (the timezone guard especially);
  - bulk-accepting goldens;
  - loosening a perf floor without a provenance comment backed by measurement.
- **Coordinate with active owners.** Several failures are already `DISCOVERED ISSUE`
  notes on active epics: `sase-1bc` owns the `%tab` fallout and the `agent_meta` mock
  skew, `sase-1bt` the Tools pane and tool-run drift, `sase-1bu` goals. Check the
  owner's bead and recent `git log` first. If the owner already landed a fix, consume
  it. If not, make the minimal correct repair, since the release is waiting on it, and
  record it on the owner with `sase bead note <owner> "..."`. Phase workers **never
  create beads**; record `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own
  phase bead instead.
- **Use the failure ownership map below.** Your `just check` shows failures that sibling
  phases own until they land. Treat those as known, and repair only your own slice plus
  anything you introduce.
- **Verification.** Run `sase tool run check` before finishing, using the
  `lint_and_test` reference memory (`sase memory read lint_and_test.md -r "<why>"`).
  **`just check-full` is explicitly authorized for this epic** (per
  `decisions:check-full-is-explicit`), but only to reproduce a failure that shows up
  only in CI's full lane, and only as `sase tool run check-full` inside a
  `/sase_monitor`. Every long wait (GitHub runs, PyPI polling, `just test-visual`, perf
  checks) also goes through `/sase_monitor`, never inline.
- **TUI work.** Every phase that touches ACE TUI code or tests reads the `tui` reference
  memory first (`sase memory read tui.md -r "<why>"`). `just check` does not run PNG
  snapshots. If your change alters rendered output, run a targeted
  `just fix-tui-screenshots -- <selectors>` and attribute each refreshed golden in your
  commit message.
- **Rust core boundary.** Shared backend logic belongs in sase-core. Only `core-macos`
  changes sase-core in this epic. If another phase finds that its fix needs core
  changes, record a `PROPOSED FOLLOW-UP` instead of reimplementing the logic in Python.
- **Resolved task beads.** When your change fixes a failure that a task bead tracks
  (listed in the map), say so in your phase-bead note. The land agent closes those beads
  after `green-master` proves the fix in CI.

## Failure ownership map (master `43cd823af`, Master Gate run 36408553789)

| Cluster                                                                                                              | Failures                                                                                                                                                                                                                                                                                              | Tracking bead                 | Phase                |
| -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | -------------------- |
| Lint `Check pinned core bindings`: pin `cbe70f66` lacks `goal_card_view`, `goal_card_markdown`, `goal_citation_line` | lint job                                                                                                                                                                                                                                                                                              | —                             | `core-pin`           |
| Stale core pin                                                                                                       | `tests/goals/test_goal_artifact_kind.py` (10), `tests/artifact_refs/test_{aliases,parsing,context}.py` (5), `tests/test_check_sase_core_rs_bindings_tool.py` (1)                                                                                                                                      | —                             | `core-pin`           |
| CLI completion kinds + snapshot                                                                                      | `tests/completion/test_kind_coverage.py` (1), `tests/completion/test_snapshot.py` (2)                                                                                                                                                                                                                 | sase-18s                      | `contract-drift`     |
| Public config schema                                                                                                 | `tests/test_config_schema.py`, `tests/test_config_schema_tools.py`                                                                                                                                                                                                                                    | —                             | `contract-drift`     |
| Mock skew (`agent_meta`)                                                                                             | `tests/test_axe_run_agent_exec_repeat_env.py` (6)                                                                                                                                                                                                                                                     | sase-1bc note #3              | `contract-drift`     |
| Trash limit 20 → 100                                                                                                 | `tests/ace/tui/actions/test_prompts_overlay_entry_points.py` (1)                                                                                                                                                                                                                                      | sase-1bq                      | `contract-drift`     |
| Getting-started wording                                                                                              | `tests/test_docs_getting_started_providers.py` (1)                                                                                                                                                                                                                                                    | sase-1as                      | `contract-drift`     |
| Marker-path audit                                                                                                    | `tests/test_agent_artifact_marker_path_passing_audit.py` (1)                                                                                                                                                                                                                                          | sase-1by                      | `contract-drift`     |
| System-clock guard                                                                                                   | `tests/test_timezone_display_guard.py` (1)                                                                                                                                                                                                                                                            | sase-1bp                      | `contract-drift`     |
| `%tab` completion                                                                                                    | `tests/ace/tui/test_agent_completion.py` (9), `tests/ace/tui/widgets/test_directive_completion_candidates.py` (2)                                                                                                                                                                                     | sase-1bc notes #1/#2          | `tab-completion`     |
| Admin Center tab model                                                                                               | `test_admin_center_selection_resume.py[updates]`, `test_log_panel_keymap.py` (alphabetical), `test_xprompt_browser_load_keymap.py` (digits), `test_plugins_browser_pane_loading.py` (seven tabs), `test_plugins_browser_pane_cached_open.py` (3 == 2), `test_config_center_resume.py` (blocked write) | sase-1bt                      | `admin-center-tabs`  |
| Scroll/anchor settle                                                                                                 | `test_agent_header_panel.py` half-page scroll, `test_deck_spread_pilot.py` files Ctrl-J                                                                                                                                                                                                               | sase-1b8, sase-1a7            | `tui-scroll-settle`  |
| Resize recompose                                                                                                     | `test_chrome_layout.py` (2)                                                                                                                                                                                                                                                                           | sase-1a6                      | `tui-resize-layout`  |
| `toobig` and the masked lint tail                                                                                    | lint job                                                                                                                                                                                                                                                                                              | sase-1a9                      | `toobig-lint-tail`   |
| Full CI structural timeouts                                                                                          | `test (3.13)`, `coverage-contexts`, `contention-test`                                                                                                                                                                                                                                                 | —                             | `ci-telemetry-split` |
| Perf floors                                                                                                          | tmux socket, `test_sase_tool_runs_smoke.py` (2), view-hints floor                                                                                                                                                                                                                                     | sase-18w                      | `perf-floors`        |
| Visual                                                                                                               | `test_config_center_flags_narrow_png_snapshot` (SUNSET) plus the unverified set at HEAD                                                                                                                                                                                                               | sase-18n, sase-x5 and backlog | `visual-lane`        |

Anything that fails at your HEAD and is missing from this map landed after planning. It
belongs to `green-master`, unless it is clearly inside your own cluster.

## core-macos

Work in the linked checkout: `sase repo open sase-core -r "<why>"`. Read its `AGENTS.md`
before editing.

The same two tests failed on all six recent sase-core CI runs, on both master and the
release branch. Both are deterministic, and both come from macOS resolving `/var` to
`/private/var`:

1. **`launch_scratch_liveness::tests::live_holder_matches_environ_path`**
   (`crates/sase_core/src/launch_scratch_liveness.rs`, `normalize_path` near line 387).
   `normalize_path` canonicalizes a path only when the path exists.
   - The test writes `TMPDIR=<candidate>/nested` into a fake procfs `environ`, and
     `nested` does not exist.
   - So the candidate canonicalizes to `/private/var/...` while the missing descendant
     stays `/var/.../nested`. `is_at_or_under` then returns false, and a live holder is
     reported dead.
   - This is a **real product bug on macOS**. Fix it in the product: canonicalize the
     deepest existing ancestor, then re-append the missing suffix, so that existing and
     future paths share one prefix. Clean the `.` components lexically. Keep the current
     handling of relative paths (join with the cwd first).
   - Search the crate for the same "canonicalize only if it exists" pattern in
     containment checks, and fix identical instances. Leave unrelated helpers alone.
   - Do **not** gate the test to Linux.
2. **`managed_tmp_roots::tests::missing_roots_are_pruned_on_next_write`**
   (`crates/sase_core/src/managed_tmp_roots.rs`, near line 339).
   `validate_root_candidate` stores the canonical path, and canonical storage is the
   right contract: dedup, broad-root refusal, and reaping all work on one identity.
   - Make the test expect `kept.canonicalize()`.
   - Correct the misleading comment ("Canonicalize for comparison only") so it says that
     storage is canonical.
   - Check the module's sibling tests for the same logical-vs-canonical assumption.

**Make the bug reproducible on Linux CI.** Add a regression test that creates a real
directory plus a symlink to it inside the tempdir. It passes the candidate and the
`environ` `TMPDIR` (with a missing `nested` suffix) through the symlinked spelling, and
asserts the holder is live. Add a direct unit test of `normalize_path` for an existing
prefix plus a missing suffix. Both tests must fail without the fix.

Run `sase tool run check` inside the sase-core checkout.

Exit: both macOS tests fixed at their root causes; the symlink regression tests pass on
Linux; sase-core check green.

## core-release

Runs after `core-macos` is on sase-core master. Read sase-core's
`.github/workflows/release-plz.yml` header and `docs/pypi-retention.md` ("Release
cadence", "Healing a partial release") first.

1. Confirm that the `core-macos` commit is on `origin/master`. Confirm that release-plz
   has refreshed release PR #315 (`chore: release v0.36.0`, branch `release-plz-*`) so
   its head contains the fix. Release-plz runs on every push to master. Record the
   version the PR proposes: 0.36.0 at planning time, because of two `feat!` wire breaks
   since 0.35.1.
2. Watch the PR's CI through `/sase_monitor` until **every** job is green, macOS
   included.
   - If an unrelated job fails, rerun it once to separate a flake from a real failure.
     One flake seen on 09-27 was Linux
     `sase_gateway ... python_hosted_launcher_preserves_isolated_module_prefix`.
   - Record flake evidence as a `PROPOSED FOLLOW-UP`.
   - A deterministic new failure on the PR is in scope. Fix it through a sase-core
     change, which follows the same landing path as `core-macos`.
3. Dispatch the release now instead of waiting for the daily cut:
   `gh workflow run release-plz.yml -f dry_run=false`, run from the sase-core checkout.
   Watch that run and the publish jobs it triggers until they finish.
4. Verify the upload. `https://pypi.org/pypi/sase-core-rs/<version>/json` must list the
   same platform wheel set as 0.35.1 plus the sdist. If files are missing, follow
   "Healing a partial release".

Exit: `sase-core-rs` 0.36.0 (or the version release-plz cut) is complete on PyPI. Its
version and file list are recorded on the phase bead.

## core-pin

1. `sase repo open sase-core -r "<why>"` and `git fetch`. Then bump
   `sase-core-revision.txt` to sase-core's current `origin/master` with
   `python3 tools/ratchet_core_revision`, the same tool the ratchet workflow uses. The
   new pin must be at or past `33b0250f91b81ebe9913796574e5faf787f741cf`; if
   `core-macos` has landed by then, the pin naturally includes it.
2. Check out the linked sase-core at the new pin, run `just install`, and prove the bump
   clears these:
   - `tools/check_sase_core_rs_bindings` (the lint job's "Check pinned core bindings"
     step): no missing bindings;
   - the 16 tests in the map: `tests/goals/test_goal_artifact_kind.py`,
     `tests/artifact_refs/test_aliases.py`, `tests/artifact_refs/test_parsing.py`,
     `tests/artifact_refs/test_context.py`,
     `tests/test_check_sase_core_rs_bindings_tool.py`.

   If any of them still fails with the new core, it is a real defect. Diagnose it. Don't
   assume the pin will fix it.

3. **Clean up the ratchet PRs.** In the sase repo, close every open `core-pin-ratchet-*`
   PR whose target SHA is an ancestor of, or equal to, the new pin (all of #302–#318 at
   planning time). Match PRs by head-branch prefix, never by number alone. Leave a short
   comment naming the new pin, and delete each PR's branch. Only close PRs that the new
   pin supersedes.
4. **Fix `.github/workflows/core-pin-ratchet.yml`** so there is at most one open ratchet
   PR at a time. Each ratchet PR runs the whole `ci.yml`, about 5 runner-hours, and 17
   of them are competing with Master Gate and Full CI. Either approach is acceptable:
   - reuse one fixed branch and update the existing PR in place; or
   - close superseded ratchet PRs and delete their branches before opening a new one.

   Update the workflow assertions in `tests/test_github_actions_ci_master_gate.py` and
   `tests/_github_actions_ci_helpers.py` to match.

Exit: pin moved; lint's bindings step passes locally; the 16 tests pass; at most one
ratchet PR can exist from now on.

## contract-drift

Each item below is small. Fix the contract, not only the assertion.

1. **CLI completion** (sase-18s). `sase tool receipt` and `sase tool receipts` (commit
   `281147666b`) added uncaptioned value slots, `tool/receipt:tool_receipt_tool` and
   `tool/receipts:tool_receipts_days`.
   - Give each one a `ValueKind`, `choices=`, or a free-form value hint in
     `src/sase/completion/kinds.py`, whichever fits the value's real domain.
   - Then run `just sync-completion-spec` so that `tests/completion/test_snapshot.py`
     (2) and `test_kind_coverage.py` pass.
2. **Public schema.** `src/sase/config/sase.schema.json` is missing two properties.
   Mirror the real config shape in each case: the loader and
   `src/sase/default_config.yml`, not a permissive `object`.
   - the tool receipt policy, `tools.<name>.receipt`
     (`tests/test_config_schema_tools.py`);
   - the Tools-pane keymaps, `ace.keymaps.tool_runs` (`tests/test_config_schema.py`).
3. **Mock skew** (6 tests, sase-1bc note #3). In
   `tests/test_axe_run_agent_exec_repeat_env.py`, `_export_exec_agent_tab` now reads
   `ctx.agent_meta`, which `Mock(spec=AgentExecContext)` lacks. Set `ctx.agent_meta` in
   the fixture to the real default shape. Also check that production callers always
   populate `agent_meta`, so a missing attribute there fails loudly and correctly.
4. **Trash limit** (sase-1bq). Commit `919ba740e` raised the stash trash limit from 20
   to 100. Make `test_open_action_opens_overlay_on_stash_with_trash_count` assert
   against the production constant instead of a literal.
5. **Getting-started wording** (sase-1as). Commit `90d6138504` rewrote
   `docs/getting_started.md`. Check the doc's provider and alias statements against the
   current `default_config.yml` alias pools. Then make
   `tests/test_docs_getting_started_providers.py` assert the current contract, not the
   old sentence. If the doc misstates the pools, fix the doc.
6. **Marker-path audit** (sase-1by). Review the two new sites, `finalizers/cli.py`
   `_target_for_dir` and `axe/run_agent_directive_metadata.py` `session_root_tab`,
   against the audit's criteria (read its test docstring). Add a site to the reviewed
   set only if it is correct. If a site passes marker paths incorrectly, fix the site.
7. **System-clock guard** (sase-1bp). Route the 7 sites through the `sase.core.time`
   helpers the guard expects. **Do not extend the allowlist.** Line numbers may have
   drifted:
   - `tool_runs/blocks.py` (~164);
   - `update_gear.py` (~80, 81, 111);
   - `update_panel_state.py` (~142, 143);
   - `tool/view_vocabulary.py` (~131).

   New sites may have landed by now. Fix every site the guard reports at your HEAD.

Exit: the 15 mapped failures pass; `sase tool run check` is green apart from sibling
phases' mapped failures.

## tab-completion

Cause: commit `372ecc97c3` (`%tab`, sase-1bc.4). `_agent_completion_candidates.py` now
always adds a default `('tab', 'main')` group, and the `%t`/`%ta` prefixes now
legitimately match `%tab`.

1. **Decide leak vs. stale expectation first.** `agent_tabs` is a `beta` feature flag.
   The sase-1bc agent-tabs contract says that with the flag off, the TUI stays exactly
   as it was before agent tabs. Find out which flag state these tests run under.
   - Flag **off**, and completion still offers a `main` tab candidate or group: that is
     a product leak. Fix the candidate builder so it offers tab candidates only when the
     flag is on (and, if the design says so, only when there are real tabs).
   - Flag **on**: rewrite the expectations to treat `%tab`/`main` as first-class
     candidates, in the order the builder documents.
2. The 9 nodes in `tests/ace/tui/test_agent_completion.py` include plan-preview
   attachment, ordered groups, empty-clan omission, named-proc exact ids, and bare local
   names. For each node, confirm that the only behavior difference is the tab group
   before you change an expectation. Anything else is a separate regression to fix.
3. `tests/ace/tui/widgets/test_directive_completion_candidates.py`:
   - The two removed-spelling tests should assert that each removed name (`%tale`,
     `%tribe`, and the removed auto-approval directives) is **absent**, instead of
     asserting that the prefix results are empty.
   - If a removed directive still appears as a snippet candidate, that is a product
     leak. Fix it.
4. Record the fix on `sase-1bc` with `sase bead note`.

Exit: the 11 mapped failures pass; any flag-off leak is fixed in the product.

## admin-center-tabs

Commit `8c134b816` (sase-1bt.10) added a Tools tab to the Admin Center. The six mapped
tests encode tab count, order, index, digit-shortcut, and resume assumptions.

1. For each test, work out whether it pins a **documented invariant** or an incidental
   count.
   - "Admin Center tabs are alphabetical by label" is an invariant. If the new tab broke
     it, fix the product order, not the test.
   - An incidental count such as "seven tabs" should be derived from the tab registry
     instead of hard-coding eight.
2. `test_admin_center_selection_resume[updates]` fails with
   `NoMatches: PluginsBrowserPane`, and `test_xprompt_browser_load_keymap` with
   `'updates' == 'config'`. Both look like index shifts. Check that the digit shortcuts
   and resume targets still map to the tabs their labels promise; if not, that is a
   product defect.
3. The two `wait_for` timeouts (`test_plugins_browser_pane_loading` cycling tabs, and
   `test_config_center_resume` blocked write) and
   `test_plugins_browser_pane_cached_open` (`3 == 2`) need to be **reproduced before any
   change**. Run them serially and under xdist load. Find the causal commit with
   `git log`/`git bisect` over the last 72h; do not assume it is the Tools tab.
4. Record any product defect on `sase-1bt` with `sase bead note`.

Exit: the 6 mapped failures pass, each with its cause stated in the commit message.

## tui-scroll-settle

1. **Header half-page scroll** (sase-1b8, `assert 2.0 == 0.0` in
   `tests/ace/tui/widgets/test_agent_header_panel.py::test_expanded_overflowing_header_claims_half_page_scroll`).
   The deck's `pin_to_bottom` settles asynchronously after the synchronous read.
   - Decide which side is wrong. If the product's claimed scroll offset is stale for a
     frame, users see it too, and it should be fixed in the product. If only the test
     reads too early, make it wait on the settle signal the product exposes (or add
     one).
   - Do not sleep.
2. **Files deck Ctrl-J** (sase-1a7,
   `tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top`,
   a 5s `wait_for` timeout). It passes when run alone but fails under full-lane load.
   - Reproduce it under xdist load first.
   - Then find what the predicate waits on and why load starves it: a missed refresh, a
     timer, or an unawaited message. Fix that cause.
   - Raising the timeout is acceptable only with evidence that the operation is correct
     and merely slow. In that case, record the measurement in a comment.

Exit: both nodes pass across repeated runs under xdist load (for example, 20
iterations), not only in isolation.

## tui-resize-layout

`tests/ace/tui/command_line/test_chrome_layout.py` has two failing nodes, and the
symptom is that a terminal resize has no effect (sase-1a6):

- `test_chrome_recomposes_and_stays_aligned_on_resize[wide0-narrow0]` (`134 > 134`);
- `test_popup_reclamps_when_the_terminal_shrinks` (`34 < 34`).

They failed at `89e4882`, passed at `5b409a5`, and failed again at `43cd823`, so they
are load-sensitive rather than fixed.

1. Reproduce under xdist load.
2. Establish whether the pilot's resize reaches the app, and whether the recompose and
   popup reclamp run before the assertion. Fix the missing settle in whichever layer
   drops it:
   - a production handler that doesn't react to every resize is a product bug;
   - a test that asserts before the resize is processed needs a real settle signal.

Exit: both nodes pass across repeated loaded runs.

## toobig-lint-tail

`just lint` stops at its first failure, which has masked everything behind it for days.
`just check` omits `toobig`, so agents never saw it.

1. **Split the two modules over the limit**, keeping the public import paths stable
   (re-export from the original module or update every caller) and symvision clean:
   - `src/sase/core/tool_run.py` (1,154 lines), for example ledger projections versus
     settlement and reporting helpers;
   - `src/sase/tool/executor.py` (1,175 lines, sase-1a9).

   Before splitting, check two things:
   - Is a `toobig_split` routine agent already working on sase-1a9? If one landed a
     split, consume it.
   - Which in-flight `sase-1bt` phases edit these files? Keep the split mechanical (pure
     moves) so their rebases stay trivial.

   Leave the warning-band files (`plan_direct_approval.py`, `tool_runs/view.py`,
   `disk_footprint_inventory.py`, `_agent_tabs.py`) alone.

2. **Run every CI `lint` job step locally, in CI order**, and fix what surfaces:
   `just fmt-py-check`, `just fmt-md-check`, `just fmt-docs-check`,
   `just model-policy-check`, `just lint` (including `toobig`), `just validate`,
   `just validate-committed-plans`, `just build-check`. The last three have not run on
   master for days, so expect findings. When a finding belongs to an active epic's
   recent commit, make the minimal fix and record it on that epic. Findings caused only
   by the stale pin belong to `core-pin`.

Exit: every lint-job step passes locally, apart from the pinned-bindings step, which
`core-pin` owns.

## ci-telemetry-split

Full CI stays the correctness gate that `ci_watch` reads
(`heavy_workflows: ["Full CI"]`, unchanged). Measurement and soak lanes move out of it,
so a timeout in one of them can no longer block a release. This keeps the intent of
`decisions:ci-two-speed-split`. Read that decision, `docs/development.md`, and
`docs/perf_runbook.md` first.

1. **Add a scheduled `CI Telemetry` workflow** (for example
   `.github/workflows/telemetry.yml`) with `workflow_dispatch`, on a schedule that keeps
   the coverage baseline fresh. Every 6 hours, offset from Full CI's `17 */2`, is
   reasonable.
   - Move three lanes into it with realistic ceilings:
     - `test-cost` on 3.13 (`just test-cost` plus the existing budget and report steps),
       about 180 minutes;
     - `coverage-contexts` (master-only upload of the contexts artifact), about 120
       minutes;
     - `contention-test` at `SASE_CONTENTION_REPEAT=1` with a worker count the 2-CPU
       runner survives, or a curated subset, about 90 minutes.
   - Keep the build-core artifact shared, either through a `workflow_call` input on
     `ci.yml` that selects a lane profile or by factoring build-core out. Choose the
     shape that keeps the structural tests simple.
   - Keep the existing `github.event_name == 'schedule'` semantics where they still make
     sense, and document any change.
2. **Full CI (`ci.yml` through `full.yml`):**
   - 3.13 runs plain `just test`;
   - 3.12 keeps `just test-cov`, which already takes 74–79 of its 90 minutes. Give it
     headroom by raising the `test` job's `timeout-minutes` (about 120) or by sharding
     the coverage leg, so it does not become the fourth permanent timeout;
   - the shard-timings artifact stays on the 3.14 leg, because
     `shard-timings-ratchet.yml` reads it from `full.yml`;
   - lint, visual-test, perf-floors, ace-page-group-isolation, and
     release-core-floor-smoke are unchanged.

   `ci.yml` also runs on pull requests, so check that the PR behavior still makes sense.
   The PR lane should lose test-cost too.

3. **Update the consumers:**
   - the `WORKFLOW` constant in `tools/fetch_coverage_contexts`, and
     `just refresh-contexts-baseline`;
   - every structural assertion in `tests/test_github_actions_ci_workflow.py` (the 3.12
     timeout, the 3.13 cost-leg step, the contexts job and fetcher assertions, the
     no-scoped-lane loop over workflow names) and in
     `tests/test_github_actions_ci_master_gate.py` and `tests/test_contract_manifest.py`
     where they apply;
   - the Full CI and CI Telemetry descriptions in `docs/development.md`,
     `docs/perf_runbook.md`, and `docs/rust_backend.md`.

   Add a structural test that `ci_watch`-gated Full CI no longer contains the three
   telemetry lanes.

4. Do **not** edit memory files or the chezmoi `ci_watch` config. The land agent files
   the decision-record follow-up.

Exit: workflow YAML is valid (`actionlint` if available, and the structural tests); the
docs are accurate; `sase tool run check` is green for this slice. The first real runs
are observed in `green-master`.

## perf-floors

Latest Full CI (run 36382114111) `perf-floors` failures:

1. **tmux socket path too long** (sase-18w,
   `tests/ace/tui/terminal_smoke/test_ace_terminal_smoke.py::test_sase_screenshot_cli_captures_png_with_tmux`).
   `TMUX_TMPDIR` under pytest's tmp path exceeds the Unix socket path limit.
   - If `sase screenshot` derives its socket directory from `TMPDIR`, fix that in the
     product: use a short `mkdtemp` under `/tmp`, so users with long `TMPDIR`s are
     covered too.
   - Otherwise, give the test a short socket directory.
2. **Tool-run smoke harness drift** (`tests/test_sase_tool_runs_smoke.py`, the hermetic
   cases and the live-owner cases). `tools/smoke_sase_tool_runs` cases drifted from
   sase-1bt landings:
   - `test-visual` is now in the catalog;
   - the replay stream-order diagnostic changed;
   - the DoD-8 live monitor changed.

   Confirm that each drift is intended, from the causing commit, before updating a case.
   Any unintended behavior change is a regression to fix, recorded on `sase-1bt`.

3. **View-hints floor.** In `just view-hints-perf-check`, "repeat press uses the cached
   hint render" measured `widget.prompt_panel.update_display_with_hints.p50_ms` = 15.0
   ms against a 12 ms absolute ceiling.
   - The metric name points to a cache miss on the repeat-press path. Find the commit
     that regressed it and restore the cached render.
   - Adjust the ceiling only if a measurement on the final SHA proves the old ceiling's
     provenance stale. In that case, add a provenance comment in the same style as the
     neighboring floors.
4. Run the whole `perf-floors` job locally through `/sase_monitor` and confirm it is
   green: `just test-slow`, `just phase7-perf-check`, `just launch-perf-check`,
   `just view-hints-perf-check`, `just agent-disk-load-ops-check`,
   `just plugin-catalog-scale-check`, plus any other steps the job lists.

Exit: every perf-floors step passes locally.

## visual-lane

Runs after the four TUI phases land, because their fixes may change rendering.

1. Run `just test-visual` (check-only) at HEAD through `/sase_monitor`. The latest Full
   CI visual failure was `test_config_center_flags_narrow_png_snapshot`, which timed out
   waiting for the SVG sentinel `SUNSET`. The full set at HEAD is unverified.
2. Treat semantic timeouts as bugs: the SUNSET wait, and the narrow top-bar usage
   `wait_for_startup` timeouts (sase-18n). Reproduce each one and find why the expected
   state never renders.
3. Handle golden drift one snapshot at a time:
   - attribute each diff to the deliberate UI commit that caused it (`git log` on the
     owning widget);
   - check that changed pixels stay on that surface;
   - then rebaseline only that snapshot with `just fix-tui-screenshots -- <selector>`.

   **Never bulk-accept.** Drift you cannot attribute is a bug. Check each backlog bead
   for whether it still reproduces: sase-x5, sase-15t, sase-16o, sase-176, sase-19z,
   sase-1ba, sase-18n.

4. Rerun `just test-visual` and confirm exact-equality green.

Exit: `just test-visual` green locally; each refreshed golden attributed in the commit
message.

## green-master

Runs after every repair phase is on master. All waits go through `/sase_monitor`.

1. **Re-inventory.** Pull the newest Master Gate run on master HEAD and its failing
   tests. For anything still red:
   - Failed in setup, for example PyPI 503s: `gh run rerun <id> --failed`.
   - Landed after planning: make the minimal root-cause repair and record it on the
     causing epic.
   - Deterministic and inside a phase's cluster: fix it here, and name the phase in a
     note.
2. **Watch Master Gate** until a run on the current HEAD is green. If new commits keep
   breaking it faster than one turn can repair, stop. Record the inflow, with commit and
   owner for each break, on this phase bead so the land agent and the user can see that
   stop-the-line is needed.
3. **Full CI:** `gh workflow run full.yml --ref master`, then watch it (roughly 90–100
   minutes once the telemetry lanes are gone) until every job is green: lint, test
   3.12/3.13/3.14, visual-test, perf-floors, ace-page-group-isolation. Fix, then
   re-dispatch. Record each job's duration next to its ceiling.
4. **CI Telemetry:** dispatch it once and record each lane's duration against its new
   timeout. A red or timed-out telemetry lane does not block this phase or the release;
   record it as a `PROPOSED FOLLOW-UP`. When the contexts artifact uploads, run
   `just refresh-contexts-baseline` and confirm that a scoped `just check` consults a
   fresh baseline (no `context-baseline-stale`).
5. Record the exact green run IDs and SHAs on this phase bead.

Exit: Master Gate green on HEAD; a fully green Full CI run on master; telemetry
measurements recorded.

## release

Runs once `sase-core-rs` 0.36.0 is on PyPI (`core-release`) and CI is green
(`green-master`). The host job `ci_watch` owns the merge. It is a bugyi-chops job on
athena that runs every 5 minutes with `merge_enabled: true`, and it merges #299 only
when all four hold:

- (a) Master Gate is green on HEAD;
- (b) the newest completed Full CI is green and at most 6h old;
- (c) #299 is MERGEABLE and CLEAN with every check green;
- (d) no Publish or release-please run is in flight.

Steps:

1. **Refresh #299:** `gh workflow run publish.yml -f publish_existing=false`.
   `sync-release-metadata` (`tools/ratchet_core_window`) then moves #299's window to
   `sase-core-rs>=0.36.0,<0.37.0` and refreshes `uv.lock`. Confirm the diff on the
   release branch. Confirm the version #299 proposes (0.18.0 at planning time).
2. Watch #299's `release-core-floor-smoke` until it is green against the 0.36.0 floor.
   Never weaken it. A failure here means a binding is still missing from the published
   core, and that goes back to the core track.
3. **Let ci_watch merge.**
   - If the newest green Full CI is older than about 4h, dispatch Full CI again first so
     condition (b) holds.
   - Inspect `sase axe job run ci_watch -n -V` (it skips itself when the live job is
     running; retry then) and read its blocking reason.
   - Monitor `gh pr view 299 --json state,mergedAt` until the state is MERGED.
4. **Bounded one-time bypass** (pre-authorized by plan approval unless the user struck
   it). Merge #299 by hand with `gh pr merge 299 --merge`, the merge method `ci_watch`
   uses, **only when every one of these holds**, and record the evidence on this phase
   bead:
   - Master Gate is green on HEAD;
   - #299's floor smoke is green against the published 0.36.0 floor;
   - a Full CI run dispatched on current master shows `lint` and every `test` leg green,
     and its only reds are items with an existing tracking bead;
   - `ci_watch` has reported only the heavy-lane condition as blocking for more than
     about 3 hours.

   Never bypass the core track. A 0.18.0 wheel built against 0.35.1 crashes at runtime
   on the missing bindings.

5. **Cut the release.** After the merge, dispatch
   `gh workflow run publish.yml -f publish_existing=false` again rather than waiting up
   to 3h for the cron. `release_created=true` should yield the tag, the GitHub release,
   the build, `install-smoke` (latest core), `install-smoke-core-floor` (exact floor),
   and the trusted PyPI upload. Watch it through to the end. Use `publish_existing=true`
   only to heal a partial upload of this same version.
6. **Verify what the user asked for:**
   - `https://pypi.org/pypi/sase/json` reports `info.version == "0.18.0"`, and its
     `requires_dist` carries `sase-core-rs>=0.36.0,<0.37.0`;
   - in a clean venv
     (`uv venv <tmp> && uv pip install --python <tmp>/bin/python sase==0.18.0`),
     `sase version` prints 0.18.0 and `sase core health --json` is healthy.

Exit: #299 merged; tag `v0.18.0` and its GitHub release exist; PyPI serves sase 0.18.0;
the clean-venv smoke passes; all evidence recorded on the phase bead.

## Land agent

1. Close the task beads that `green-master` proved fixed in CI, with
   `sase bead close <id> --note "<run id / sha>"`. Candidates: sase-1a9, sase-18w,
   sase-1bp, sase-1by, sase-1as, sase-1bq, sase-1b8, sase-1a7, sase-1a6, sase-18n,
   sase-18s, and any visual backlog beads `visual-lane` resolved. Leave a bead open if
   CI did not prove its fix.
2. Add notes to the stale CI epics `sase-10w` (its v0.17.2 target is superseded by
   0.18.0), `sase-126`, and `sase-th`, pointing at this epic's green runs and the
   release. Do not close them; they have their own land agents, and the user decides.
3. File follow-ups through `/sase_new_task`, checking for duplicates first. These are
   the research report's §7.5 keep-green items, which matter because detection alone has
   not kept master green:
   - **Stop-the-line automation.** When Master Gate on HEAD stays red longer than N
     hours, hold non-repair launches and launch one build-cop repair agent seeded with
     the failing jobs and matching `ci` beads. This reverses `ci_watch`'s "never
     launches repair agents" contract, so it needs a new decision record. It is the most
     important follow-up.
   - Add the cheap whole-repo invariant tests to `tests/contract_manifest.txt`:
     completion snapshot and kinds, config schemas, the marker audit, and the docs
     guards.
   - Make `toobig` visible to `just check` for touched files, at least as a warning.
   - Key guard-test failure signatures on the violation payload, so KNOWN stops masking
     new violations.
   - Enforce cross-repo landing order, so a sase commit that needs unpinned core
     bindings waits for the pin.
   - Add a non-blocking Master Gate job that runs `check_sase_core_rs_bindings` against
     the newest published core, so core-floor skew shows up the day it starts.
   - Alert when a CI job passes 80% of its timeout.
   - Harden the CI install step against PyPI 503s (uv retries, cache, or a pre-resolved
     lock artifact).
   - A `memory` task for a decision record stating that measurement and soak lanes (CI
     Telemetry) do not gate releases, related to `decisions:ci-two-speed-split`. Nothing
     in this epic edits memory.
4. Triage every `PROPOSED FOLLOW-UP:` note the phases recorded.
