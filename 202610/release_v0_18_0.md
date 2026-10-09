---
tier: epic
title: Turn CI green and ship sase v0.18.0 to PyPI
goal: 'Every sase and sase-core CI lane is green, a sase-core-rs release carrying
  every binding sase needs is on PyPI, the release-please PR merges, and `pip install
  sase==0.18.0` works from PyPI by morning.

  '
phases:
- id: core-ci
  title: Fix the red sase-core master CI
  depends_on: []
  size: medium
  description: 'core-ci: fix the macOS replay-golden test failure (and any other red
    job) on sase-core master so its release PR can merge.'
- id: core-release
  title: Cut and publish the sase-core-rs release
  depends_on:
  - core-ci
  size: medium
  description: 'core-release: confirm sase-core CI is green, dispatch the urgent release-plz
    cut, and wait until the new sase-core-rs is fully on PyPI.'
- id: gate-fixes
  title: Fix the sase Master Gate failures
  depends_on: []
  size: medium
  description: 'gate-fixes: fix the lint and eight fast-suite failures that turn every
    Master Gate run red, including the memory README drift.'
- id: full-ci-fixes
  title: Fix the Full CI-only failures
  depends_on: []
  size: medium
  description: 'full-ci-fixes: fix the two coverage-leg-only test failures and the
    two drifted visual PNG goldens that keep Full CI red.'
- id: release-gates
  title: Prove every release gate green
  depends_on:
  - core-release
  - gate-fixes
  - full-ci-fixes
  size: medium
  description: 'release-gates: regenerate the release PR onto the new core floor,
    then drive Master Gate, Full CI, and the release PR checks to green.'
- id: ship
  title: Merge the release PR and publish v0.18.0
  depends_on:
  - release-gates
  size: medium
  description: 'ship: merge the 0.18.0 release PR once its gates hold, run the publish
    workflow, and verify the release installs from PyPI.'
proposed_by: bbugyi200.athena.0ys
create_time: 2026-10-09 03:55:07
status: wip
bead_id: sase-1io
---

- **PROMPT:** [prompts/202610/release_v0_18_0.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/release_v0_18_0.md)
- **BEAD:** [sase-1io](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1io/README.md)

# Plan: Turn CI green and ship sase v0.18.0 to PyPI

The user is asleep and expects sase **v0.18.0** on PyPI when they wake up. Their
request, quoted exactly: "Can you help me do whatever needs to be done to get all tests
green and release version v0.18.0 of the sase package to PyPI?" Every phase serves that
outcome. Nobody is around to answer questions overnight. Make reasonable calls yourself
and record them in bead notes. Do not stop to ask.

## Escalation Rule: Hand Off To `opus/opus@xhigh`

The user explicitly asked for this rule. If you get stuck, or for any reason think you
cannot close your assigned phase bead, do not end your turn with the bead open. Hand off
with the `/sase_handoff` skill to the model `opus/opus@xhigh`. The user's instruction is
your explicit authorization to use that skill. Write a self-contained successor prompt
that includes the phase id, what you found, what you tried, the current evidence (run
ids, PR numbers, failing tests), and what is left. For example:

```bash
sase pipe '<self-contained successor prompt>' \
  --reason 'stuck on <phase-id>: <one-line why>' --model 'opus/opus@xhigh'
```

By default the successor inherits this chat. Add `--fresh` when your context window is
nearly spent.

- If `sase pipe` rejects that model spec, retry with `--model 'claude/opus@xhigh'`. A
  non-zero exit means no handoff happened, so you are still running.
- Fence any literal `%` or `#` text in the prompt. Both are live syntax for the
  successor.
- A successor that is already on opus at xhigh hands off again only when its context
  window is spent, using `--fresh`. Otherwise it keeps working.
- Waiting on CI is **not** being stuck. Wait with `/sase_monitor` instead (see below).

## Facts Established While Planning (2026-10-09 ~07:45 UTC)

Re-verify these at the current `origin/master` before you act. Other agents push to
master all night, so a failure may already be fixed or a new one may appear. Run
`git fetch` and `gh run list --branch master` first.

**Release mechanics**

- `sase` releases through release-please. PR **#299** `chore(master): release 0.18.0`
  sits on the `release-please--branches--master` branch. `publish.yml` runs on cron
  (`17 */3 * * *`) or by `workflow_dispatch`. With `publish_existing=false` it
  regenerates the release PR, and its `sync-release-metadata` job ratchets the
  `sase-core-rs` window in `pyproject.toml` and `uv.lock` to the newest published core.
  After the release PR merges, the next generation run creates the tag and GitHub
  release, then runs build, `install-smoke`, `install-smoke-core-floor`, and publishes
  to PyPI.
- The `ci_watch` AXE routine is running on this host (scheduler is up). It merges a
  fully green release-please PR with merge method `merge` when three things hold:
  `Master Gate` is green for the master tip, `Full CI` was green within the last **6
  hours**, and the PR's own checks are green.
- PR #299's CI currently fails `release-core-floor-smoke`. The published floor,
  `sase-core-rs 0.37.0`, lacks 5 of 807 required bindings: `bead_probe_target_owner`,
  `classify_auto_directive`, `instruction_manifest_wire_schema_version`,
  `normalize_instruction_manifest`, and `wait_epic_follow_reduce`. All five exist on
  sase-core `master`. This blocker is cleared by publishing a new core and letting the
  ratchet move the floor. Never hand-edit the window.

**sase-core** (open with `sase repo open sase-core -r "<why>"` and read its `AGENTS.md`)

- Release PR **#323** `chore: release v0.37.1` is open. Release-plz merges it only on
  the daily `41 7 * * *` cut or on
  `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`, which is
  the documented urgent-cut path in `docs/pypi-retention.md`. Either path waits for the
  PR's checks. The merge push then tags the version and builds and publishes five
  distributions: linux x86_64, linux aarch64, macOS universal2, win_amd64, and sdist.
- Master CI and the release PR's CI are both red on **macOS only**:
  `bead::mutation::tests::replay_goldens::cached_golden_bytes_match_replay` panics at
  `crates/sase_core/src/bead/mutation/tests/replay_goldens.rs:1516`. It came in with the
  recent bead-mutation and replay-golden commits. One earlier master run also failed
  with `Failed to parse a valid channel from rust-toolchain.toml`. Later runs got past
  that step, but confirm it is not recurring.

**sase Master Gate** (latest completed failing run 37898192260, at `3b3d876911`)

- lint, `_lint-test-waits`: `tests/ace/tui/test_plan_decision_ace_stale.py:183` and
  `:310` use retired `inline-pause-wait` waits.
- `tests/ace/tui/widgets/test_directive_completion_candidates.py::test_directive_completion_includes_representative_descriptions`.
  The `%auto` description changed: expected text ends "...the gate kind", actual ends
  "...ils at launch".
- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`. A
  new `xprompt` string literal or resource appeared.
- `tests/ace/tui/test_plan_decision_ace_render.py::test_first_frame_tint_keeps_syntax`.
  The chosen-branch tint drops the syntax foreground.
- `tests/instructions/test_verify_cli.py::test_verify_help_documents_flags` and
  `tests/main/test_parser_command_help.py::test_memory_help_marks_primary_command_and_init_alias`.
  The `sase instructions verify|render` help no longer contains `-a, --agent`.
- `tests/ace/tui/command_line/test_completion_fixes.py::test_tab_through_every_subcommand_reaches_the_last_with_a_highlight`.
  It reaches only 19 to 26 of 32 subcommands, and it fails on every leg, so it is not a
  flake.
- `tests/main/test_init_memory_committed_drift.py::test_repo_project_memory_notes_match_generator_output`.
  The generator wants to update `sase/memory/README.md`.

**sase Full CI** (latest completed failing run 37864697731)

- 3.13 and 3.14 fail only on tests in the Master Gate list above.
- The 3.12 coverage leg also fails
  `tests/ace/tui/test_launch_context_bar.py::test_agents_row_fit_shares_density_between_gauge_and_bar`
  (`wait_for` timed out after 5.0s) and
  `tests/ace/tui/test_link_follow_bounded_panes.py::test_archived_plan_reveals_through_path_context_after_scan`
  (got `None` where it expected an `ArtifactEntryTarget`).
- `visual-test`: `agents_final_receipt_failed_120x40.png` (0.16% of pixels changed) and
  `tool_runs_card_failed_120x40.png` (7.0% of pixels changed) drifted.

## Guardrails For Every Phase

- Fix root causes. Never weaken an assertion, skip a test, or add a `xfail`/retry to get
  green. Update an expectation only after you confirm that the product change behind it
  was intentional (check `git log -S` and the responsible commit). Then make the stale
  side match current truth.
- Never hand-edit release-owned files: `CHANGELOG.md`, any `version` field,
  `.release-please-manifest.json`, the `sase-core-rs` requirement window in
  `pyproject.toml`, or any sase-core `version`, pin, or `CHANGELOG.md`. Release-please,
  release-plz, and `sync-release-metadata` own them.
- Do not run `just install` or `just install-dev`; they are human-only. Use
  `just install-venv` for this checkout's `.venv`.
- Verify with `sase tool run check`. Run it in sase-core too, from the opened checkout.
  Run `just fix` first. Only `full-ci-fixes` is explicitly authorized to run
  `just check-full`.
- Wait for CI, releases, and PyPI only through `/sase_monitor`, for example a
  `gh run watch <id> --exit-status` or a bounded polling script with `--timeout 3h` and
  a `--next` that says exactly what to do with the result. Never end a turn promising to
  come back.
- The only PRs you may merge are sase PR #299 (in `ship`) and the sase-core release-plz
  PR. Merge the sase-core PR only through the documented `release-plz.yml` dispatch.
  Leave every other open PR alone (#300, #301, #319).
- Before you start a fix, look for an existing bead with `sase bead list -T ci` and
  `sase bead list -T flake`. Note on that bead that you are fixing it rather than
  duplicating the work.
- Land code only through the host finalizer at the end of your turn. A phase cannot
  watch CI for its own commit, so the phases are split into fix rounds and verify
  rounds. Close your bead when its done criterion below holds.

## Phase core-ci: Fix the red sase-core master CI

Open sase-core with `/sase_repo` and work in the printed path. Reproduce or reason out
the macOS-only `cached_golden_bytes_match_replay` failure from the CI log, which shows
the full panic message and the golden diff. Likely suspects are platform differences
that leak into the golden bytes: `/tmp` → `/private/tmp` symlinks (sase-core's
`AGENTS.md` requires canonicalizing both sides or neither), filesystem or `readdir`
ordering, line endings, or timestamps and permissions. Fix it so the golden is
deterministic on every platform. Then check the other recent red runs on sase-core
master for any second failure, such as the `rust-toolchain.toml` channel parse, and fix
anything still live. Pass `sase tool run check` in sase-core.

**Done when** the fix is ready to land in the sase-core repository's commit. You cannot
run macOS locally, so record in a bead note how the fix removes the platform dependence.
`core-release` verifies it on CI.

## Phase core-release: Cut and publish the sase-core-rs release

1. Wait for sase-core master CI on the commit that includes the `core-ci` fix, on both
   ubuntu and macOS. Then wait for the release-plz PR's own CI after release-plz
   refreshes it. If CI is still red, apply the `core-ci` approach again and land the
   fix. Close this bead with a note that says `RELEASE NOT CUT: <reason>`, and
   `release-gates` picks up the cut.
2. When CI is green, run
   `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. Do not
   run it if the daily cut has already merged the PR. Monitor the dispatch run. Then
   monitor the push-triggered `Release-plz` run that tags the version and publishes the
   wheels.
3. Verify on `https://pypi.org/pypi/sase-core-rs/json` that the new version (expected
   `0.37.1`, though release-plz decides) lists all five unyanked distributions. Also
   verify that a fresh `uv venv` with `uv pip install sase-core-rs==<version>` imports
   all five bindings named above.
4. If the publish job refuses because of PyPI storage quota, follow
   `docs/pypi-retention.md` in sase-core. If recovery needs a human, such as a PyPI web
   login for deletion, record exactly what is needed, send it as a notification through
   `sase notify create`, and hand off per the escalation rule.

**Done when** the new `sase-core-rs` is complete on PyPI with all five bindings.

## Phase gate-fixes: Fix the sase Master Gate failures

Run `just install-venv`; this workspace's `.venv` was stale at planning time. Then
reproduce each Master Gate failure listed above at `origin/master` and fix it:

- Replace the two retired inline pause waits with `sase.ace.testing.wait.wait_for`,
  following the lint message's guidance.
- For the `%auto` description, help-flag, tint, and terminology failures, find the
  commit that changed the behavior (likely candidates are `docs(sase-1id.6)`, the
  `sase instructions` group, the chosen-branch tint, and the plan-decision splits). Fix
  whichever side is wrong. If a CLI help regression dropped the `-a, --agent` long form,
  fix the parser or help, not the test.
- For the tab-through-subcommands completion test, find why highlighting stops after
  roughly 20 of 32 entries, and fix the product bug or the test's assumption. It fails
  deterministically.

- For the committed-drift test: `sase/memory/README.md` is output that
  `sase memory init` generates, not an authored memory note, so no note content changes.
  The user's request quoted above authorizes regenerating it. First confirm the drift
  comes from a generator change (`git log -S` on the new README text in `src/`). Then
  regenerate it with `sase memory init`. Commit only the project-repo files the drift
  test names. Do not commit or deploy changes to the user's chezmoi home or live `~`
  files. If `sase memory init` touches them, restore them. If the drift instead requires
  editing an authored note under `sase/memory/`, do not edit it. Record a
  `PROPOSED FOLLOW-UP:` note on this bead and tell the user through
  `sase notify create`.

Run all the tests named above, plus `sase tool run check`.

**Done when** every listed Master Gate failure, plus any new fast-gate failure present
at `origin/master` when you start, passes locally and is ready to land.

## Phase full-ci-fixes: Fix the Full CI-only failures

This phase owns the two 3.12-coverage-leg failures and the visual goldens. Failures that
`gate-fixes` owns are out of scope here and count as KNOWN.

- Reproduce the two tests both under coverage (`just test-cov`-style invocation of only
  those tests) and under load. If the 5s `wait_for` timeout is too tight only under
  coverage, fix the underlying slowness or the readiness predicate. Do not just raise
  the timeout. Fix the archived-plan link-follow lookup that returns `None`.
- For the two drifted PNGs, use the Full CI `visual-test` artifacts and the commits that
  changed agents final-receipt and tool-run cards to decide whether the new rendering is
  intended. If it is, regenerate with `just fix-tui-screenshots`, run through
  `/sase_monitor` because it is long. If it is not, fix the rendering regression. Commit
  every dirty golden per `/sase_final`'s screenshot rule.
- This bead explicitly authorizes `just check-full` through `/sase_monitor` with the
  `verify` profile as final verification.

**Done when** both tests and both goldens pass locally and the fixes are ready to land.

## Phase release-gates: Prove every release gate green

1. Precondition: the required `sase-core-rs` is complete on PyPI. If `core-release`
   closed with `RELEASE NOT CUT`, do the `core-release` procedure first.
2. `git fetch` and confirm that the `gate-fixes` and `full-ci-fixes` commits are on
   `origin/master`. Then start these in parallel:
   - `gh workflow run full.yml --repo sase-org/sase` on the current tip. Full CI runs on
     cron every two hours, so check whether a run already covers a tip containing the
     fixes.
   - `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false`. This
     regenerates PR #299 from current master and ratchets its `sase-core-rs` floor.
     Confirm that PR #299 is still titled `chore(master): release 0.18.0`, that its
     `pyproject.toml` floor is now the new core, and that `release-core-floor-smoke`
     passes.
3. Monitor Master Gate for the tip, the Full CI run, and PR #299's checks until they
   settle.
4. If anything is red, reproduce it at `origin/master`. Fix every failure, whether
   residual or newly pushed by another agent. Land the fixes and close this bead with a
   note that says `GATES NOT YET GREEN: <failures fixed, runs to re-dispatch>`. `ship`
   re-runs the gates. Hand off only if you cannot produce a fix.

**Done when** Master Gate for the tip, a Full CI run newer than the fixes, and every
check on PR #299 are green, or when every remaining red has a landed fix and the note
above.

## Phase ship: Merge the release PR and publish v0.18.0

1. Re-establish the three `ci_watch` conditions: Master Gate is green on the current
   master tip, Full CI was green within 6h, and PR #299 is green and still titled
   `chore(master): release 0.18.0`. If `release-gates` left a `GATES NOT YET GREEN`
   note, re-dispatch Full CI and `publish.yml` as described there and wait. A new red
   that needs code goes through the `release-gates` fix procedure: land the fix, close
   the bead with a `RELEASE NOT SHIPPED: <state>` note, and the epic land agent finishes
   the release.
2. Let `ci_watch` merge #299. It runs one merge per tick. If all conditions hold and it
   has not merged within about 30 minutes, merge it yourself with
   `gh pr merge 299 --repo sase-org/sase --merge`. The user explicitly asked for this
   release, and that command uses the same method `ci_watch` would.
3. Run `gh workflow run publish.yml --repo sase-org/sase -f publish_existing=false` so
   release-please creates the `v0.18.0` tag and GitHub release. Then monitor `build`,
   `install-smoke`, `install-smoke-core-floor`, and `publish`. If the tag exists but
   publishing failed, fix the cause. Then use the `-f publish_existing=true` dispatch,
   whose upload uses `skip-existing`.
4. Verify that `https://pypi.org/pypi/sase/json` reports `0.18.0` with a wheel and an
   sdist. In a fresh venv, run `uv pip install sase==0.18.0` followed by `sase version`
   and `sase core health --json`. Both must succeed.
5. Send a `sase notify create` summary for the user to read in the morning. Include the
   PyPI URL, the core version, and which failures were fixed.

**Done when** `sase==0.18.0` installs from PyPI and passes the health check.

## Landing

The land agent confirms that `sase==0.18.0` is live on PyPI and that the final Master
Gate and Full CI runs are green. If any phase closed with a `RELEASE NOT CUT`,
`GATES NOT YET GREEN`, or `RELEASE NOT SHIPPED` note and the release is still not on
PyPI, finishing the release **is** the landing work. Follow the `ship` procedure and the
escalation rule. Triage the phases' `PROPOSED FOLLOW-UP:` notes into task beads as
usual.
