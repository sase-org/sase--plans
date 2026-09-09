---
tier: epic
title:
  Drive the master gate green — fix the fast-suite failures it attributes and realign
  the drifted visual lane
goal: "The Master Gate reaches a green conclusion on the tip of master and stays there,
  and the exhaustive visual lane is green again, so the release gate's default-branch
  and heavy-lane-freshness conditions can both open.

  "
phases:
  - id: fastlane
    title: Fix the three fast-suite failures the gate reports
    depends_on: []
    size: medium
    description:
      "fastlane: stop the gate's own SASE_TEST_SHARD leaking into nested run_pytest
      subprocesses, give chat-path naming an in-process fallback so it no longer needs a
      developer dotfile, and rewrite the two relation-panel tests that still describe
      pre-Link-Rail behavior."
  - id: visual
    title: Realign the ACE visual lane with the shipped Artifacts and Link Rail UI
    depends_on: []
    size: medium
    description:
      "visual: replace every hard-coded Artifacts digit press in the visual suite with
      the AcePage.artifacts_digit seam, regenerate the PNG golden corpus for the
      agents-first sub-tab strip and the app-level Link Rail, and review the diffs
      before accepting them."
  - id: converge
    title: Land, sample the gate on the tip, and record the flakes
    depends_on:
      - fastlane
      - visual
    size: medium
    description:
      "converge: land both lanes, sample Master Gate on the moving tip until it is
      durably green, confirm the exhaustive lane is green, and record every
      fail-then-pass test as a PROPOSED FOLLOW-UP note rather than muting it."
proposed_by: bbugyi200.athena.sase-um.5
parent_bead: sase-um.5
bead_id: sase-um.5.1
create_time: 2026-09-09 19:50:50
status: wip
---

- **PROMPT:**
  [prompts/202608/master_gate_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/master_gate_green.md)
- **PARENT:** [202608/release_gate_liveness.md](release_gate_liveness.md)
- **BEAD:**
  [sase-um.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.5.1.md)

# Plan: Drive the master gate green

## 1. Problem

Phase `gate` of the parent epic landed `.github/workflows/master-gate.yml`. Its first
per-SHA run on master was red, and because the gate is per-SHA and never cancelled, the
red set is now exactly attributable for the first time — which is the whole point of
that phase.

Three independent causes account for every fast-suite failure the gate reports. A
fourth, unrelated cause accounts for 359 failures in the exhaustive lane's `visual-test`
job. None of the four is a flake; all four reproduce.

## 2. Evidence

Measured 2026-08-27 against `sase-org/sase`:

- **Master Gate** run `33069216824` (the tip at the time): `core-wheel`, `lint`,
  `test (3)` and `test (4)` green; `test (1)`, `test (2)`, `test (5)` and `test (6)`
  red, for 19 distinct failing tests.
- **CI** run `33043793529` (the last `ci.yml` run to reach a terminal conclusion):
  `visual-test` reported `359 failed, 473 passed, 1 skipped in 1107.56s`. `lint`,
  `perf-floors` and `build-core` were green.
- The gate's `lint` job is green, so the lint failures the parent epic's research
  recorded are already gone.

Do not work from this failure list alone. Master takes a commit roughly every ten
minutes, so the implementing agent must re-read the newest completed `Master Gate` run
first and reconcile it against these four causes before changing anything.

## 3. Cause A — the gate's shard variable leaks into nested runner subprocesses

`master-gate.yml` exports `SASE_TEST_SHARD=<n>/6` for the whole `just test` step.
`tests/test_suite_gate_integration.py` builds a miniature repo and spawns a real
`tools/run_pytest <mode>` child through `_miniature_pool_environment`, which copies
`os.environ` and scrubs a list of ambient `SASE_TEST_*` switches — but not
`SASE_TEST_SHARD`. The child inherits the parent shard spec and exits 4 with:

```
pytest runner configuration error: SASE_TEST_SHARD is only supported in fast mode; got mode='scoped'
```

Three tests fail:

- `test_scoped_run_takes_no_token_while_the_pool_is_exhausted`
- `test_over_budget_selection_runs_at_a_leased_width_and_releases_it`
- `test_scaled_suite_runs_share_capacity_and_release_after_sigkill` (this one reports
  `TimeoutError` because the child it waits on never starts)

The runner's rejection of a shard spec outside fast mode is correct and is covered by
`tests/test_run_pytest_shards.py`; the defect is that the spawn site is not hermetic.
`tests/_run_pytest_fixtures.AMBIENT_MODE_ENV_VARS` already scrubs `SASE_TEST_SHARD` for
the in-process `main()` tests and its comment states exactly this reasoning — apply the
same reasoning at the subprocess spawn site.

**Work:** add `SASE_TEST_SHARD` to the scrub tuple in `_miniature_pool_environment`,
keeping the tuple's sorted order. Add a regression test that sets `SASE_TEST_SHARD` in
the parent environment and asserts the miniature runner still exits 0, so the leak
cannot return silently. Then audit the other test modules that spawn `tools/run_pytest`
as a subprocess for the same inherited-environment gap and fix any that share it; only
this module failed in the gate run, but the gate is the first caller that ever sets the
variable, so absence of a failure is weak evidence.

## 4. Cause B — chat-path naming hard-depends on a developer dotfile

`sase.history.chat._get_branch_or_workspace_name` shells out to
`branch_or_workspace_name` and raises `RuntimeError` on any non-zero exit. That command
is a script in the maintainer's personal `~/bin`; on a CI runner `/bin/sh` reports
`command not found` and the helper raises.
`sase.gate_shell.settlement._write_settlement_chat` reaches it whenever the member's
`cl_name` metadata is absent, so every gate-shell settlement path dies.

This is a product bug the gate surfaced, not test noise. `sase gate answer` crashes the
same way on any machine that does not have that dotfile installed, which includes every
CI runner and every contributor who is not the maintainer.

Fourteen tests fail on it:

- `tests/gate_conformance/test_gate_shell_conformance.py::test_shell_gate_settles_identically_across_every_surface`
- `tests/gate_shell/test_settlement_chat.py` — both tests
- `tests/gate_shell/test_settlement_followup.py` — all six tests
- `tests/main/test_gate_shell_handler_cancel.py` — all three tests
- `tests/test_gate_cli_answer_detach.py::test_gate_shell_no_detach_runs_inline_and_settles`
- `tests/test_user_question_gates.py::test_shell_backed_question_settles_its_gate_shell_and_streams_output`

**Work:** keep preferring the helper when it is on `PATH`, and fall back in-process when
it is not — the current git branch name, and failing that the working directory's name —
still passing the result through `strip_reverted_suffix`. Deleting the shell-out
entirely is out of scope: the helper encodes the maintainer's own
branch-versus-workspace preference and is the documented source of these names when it
is present.

`tests/history/test_chat_paths.py::test_get_branch_or_workspace_name_failure` currently
pins the crash and must be rewritten to describe the fallback. Add a test that a missing
helper still yields a usable, sanitized name. Check that `sase.history.chat_resume` and
`sase.history.chat_fork` lookups tolerate the fallback value, since chat basenames feed
resume resolution.

## 5. Cause C — two tests still describe the pre-Link-Rail relation panel

Commit `a7b702863` moved typed links to the app-level Link Rail, and
`sase.core.artifact_relation_layout.build_relation_view` now skips every
`RelationKind.LINK` declaration — its docstring says so. Two tests in
`tests/ace/tui/test_artifacts_relation_collapse.py` still describe the old behavior and
fail deterministically, locally as well as in CI:

- `test_expanded_link_row_renders_edge_metadata` indexes `view.sections[0].rows[0]` for
  a view built from a single `LINK` declaration. That view now has no sections, so it
  raises `IndexError`.
- `test_dot_collapses_and_expands_on_each_relations_pane` includes `ref:plan` in its
  pane matrix and asserts every pane's relation panel is displayed. The plans pane's
  declarations are typed links, so its panel is now legitimately hidden, and the test
  fails with `ref:plan relation panel stayed hidden`.

**Work:** before editing either test, establish why the plans pane's panel is empty.
Provider panes carry a `FAMILY`-kind bundle relation as well as the link relations, so
"no sections" can mean either that only link declarations survive or that the test
fixture supplies no non-link edges. If a hierarchy, family, or sibling declaration is
being dropped by `build_relation_view` itself, the defect is in the layout code and the
tests are right — fix the code instead. If the panel is correctly empty, rewrite both
tests against the shipped contract: the first should assert what the panel does render
for a link-only view (or move to whatever now renders link rows), and the second should
stop asserting a panel for a pane that has none.

## 6. Cause D — the visual lane has drifted two intended UI changes behind

Not a gate failure: the gate deliberately runs only the non-visual fast suite. It is in
scope for this phase anyway, because the parent epic's release gate requires a recent
green run of the exhaustive lane, and a red visual job keeps that condition shut.

`just check` and `just check-full` both exclude the visual suite, so it only ever runs
in CI's `visual-test` job and in `just test-visual`. Two intended UI changes therefore
landed without it:

**The Artifacts digit shift (288 failures).** Commit `4dd299502` put the Agents pane
first in the Artifacts sub-tab strip, shifting every digit shortcut by one. 258 visual
tests still press `"2"` and then assert `expect_state("artifacts_subtab", "patches")`,
which is now `stitches`. `AcePage.artifacts_digit(subtab)` exists for exactly this case
and its docstring already says tests must not hard-code digits; the `config_center_*`
visual tests were hand-updated to `press("3")` and pass today.

**Work:** replace every hard-coded digit press that is followed by an Artifacts sub-tab
assertion with `await page.press(page.artifacts_digit("<subtab>"))`. Include the nine
already-correct `press("3")` sites, so the next reorder cannot reintroduce this. Leave
digit presses that drive other panes alone — verify each one's target before touching it
rather than pattern-matching on the digit.

**Stale PNG goldens (70 failures now, more once the presses are fixed).** The 572
goldens under `tests/ace/tui/visual/snapshots/png/` predate both the agents-first strip
and the app-level Link Rail (`d8e8b5ab8`, `a7b702863`). Most of the 288 tests above
abort before reaching their PNG assertion, so the true mismatch count is only knowable
after the press fix.

**Work:** fix the presses first, run `just test-visual` to get the real mismatch set,
then regenerate with `just update-visual-snapshots`. Review before accepting: inspect
the actual/expected/diff artifacts under `.pytest_cache/sase-visual/` for a
representative sample across the affected areas and confirm each diff is the
agents-first strip or the Link Rail and not a rendering fault. Accepting 572 regenerated
goldens without reading any of them would silently bless a real regression.

**One flake (1 failure).** `test_axe_constrained_width_no_wrap_png_snapshot` timed out
after 15s waiting for the AXE output scroll to pin to top. It passes locally and is a
convergence timeout under the visual lane's oversubscribed worker pool, not a golden
problem. Record it; do not rebaseline it.

## 7. Phase converge — land and prove it

1. Rebase onto the current tip before landing; master moves every ~10 minutes.
2. Verify each lane before landing it: `just check` inline, and `just check-full`
   through `/sase_monitor` with the `TESTING` / `TESTED` status pair, because it
   routinely outruns a single agent turn. Run `just test-visual` for the visual lane,
   which `check-full` does not cover.
3. Land the fast-lane fix and the visual rebaseline as separate pushes. The golden
   regeneration is a several-hundred-file diff that will conflict with any other agent
   touching the visual suite; keeping it out of the release-blocking fast-lane fix means
   a conflict costs one redo, not both.
4. After landing, read the `Master Gate` run for each new tip. The exit condition from
   the parent epic is: `Master Gate` green on the tip for a majority of samples taken
   ten minutes apart over an hour, and the newest exhaustive-lane run green.
5. `Full CI` (`full.yml`) is phase `heavy`'s deliverable and may not exist yet. If it
   does not, satisfy the second half of the exit condition through `ci.yml`'s
   `visual-test`, `test` and `coverage-contexts` jobs on a master push, and record which
   lane was used so the epic's verify phase can reconcile it.
6. Record every fail-then-pass test as a `PROPOSED FOLLOW-UP:` note on the phase bead. A
   phase worker must not create beads; the epic's land agent triages these into `flake`
   task beads. Do not mute, skip, or xfail any of them.

Known flake candidates to record, all of which pass locally and failed exactly once in
the runs measured while planning:

- `tests/test_ace_testing.py::test_ace_page_fast_startup_is_structurally_quiet`
- `tests/ace/tui/test_plugins_browser_pane_sase_update.py::test_updates_pane_sase_update_confirm_executes_and_refreshes`
- `tests/ace/tui/visual/test_ace_png_snapshots_axe_layout.py::test_axe_constrained_width_no_wrap_png_snapshot`

## 8. Risks

| Risk                                                              | Safeguard                                                                                                                                |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| The failure list in this plan is stale by the time work starts    | Re-read the newest completed `Master Gate` run first and reconcile it against these four causes before editing anything.                 |
| The golden regeneration bakes in a real UI regression             | Review a representative sample of `.pytest_cache/sase-visual/` diffs against the two intended UI changes rather than trusting the count. |
| The several-hundred-file golden diff conflicts with another agent | Land it as its own push, separate from the fast-lane fix, and rebase immediately before pushing.                                         |
| The chat-name fallback changes resume lookups                     | Only the previously-fatal path changes behavior. Check `chat_resume` and `chat_fork` resolution against a fallback-derived basename.     |
| Fixing the visual presses uncovers many more mismatches           | Expected: most of the 288 abort before their PNG assertion. Measure with `just test-visual` after the press fix, before regenerating.    |
| A fixed cause reappears once the gate is green                    | Each cause gets a regression test in the same push, not just a fix.                                                                      |

## 9. Deferred

- **A lint that rejects hard-coded Artifacts digit presses in tests.** It would have
  caught 258 of the 359 visual failures at `just check` time.
  `tools/check_test_wait_helpers` is the nearest existing AST lint over `tests/`, but
  its charter is bounded-wait idioms and this rule does not belong bolted onto it.
  Record it as a follow-up rather than growing this phase.
- **The structural cause — no cadence runs the visual suite.** Phase `heavy` of the
  parent epic adds `full.yml` and puts the exhaustive matrix on a schedule; that is the
  durable guard against this drift recurring, and it is not this phase's work.
- **Moving chat-path naming behind the Rust core boundary.** The naming rule is shared
  backend behavior by the project's own litmus test, but relocating it is a separate
  change with its own wire and binding work.
