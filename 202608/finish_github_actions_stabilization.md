---
tier: tale
size: medium
title: Finish stabilizing GitHub Actions and land epic sase-m4
goal:
  Repair the two CI failures epic sase-m4 left red, unblock the flake-baseline gate,
  drive a green Actions run for the landed commit, and close the epic
proposed_by: bbugyi200.athena.sase-m4.land
bead: sase-m4
create_time: 2026-08-14 18:08:28
status: wip
---

- **PARENT:** [202608/stabilize_github_actions.md](stabilize_github_actions.md)
- **BEAD:**
  [sase-m4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m4/README.md)

# Plan

Epic `sase-m4` (Stabilize GitHub Actions, plan `202608/stabilize_github_actions.md`)
closed all six phases, but the landing review found that two of its phases did not
actually reach their acceptance criteria: the `sase` default branch is still red. This
tale finishes that work, lands it, verifies the resulting Actions run, and closes the
epic.

Do not edit any file under `sase/memory/`, `AGENTS.md`, or the generated provider
instruction shims. Run `just install` first in a fresh workspace.

## What the landing review already verified as genuinely done

Do not redo these:

- **sase-m4.1 (core release floor).** `pyproject.toml` pins
  `sase-core-rs>=0.27.2,<0.28.0`, `tools/ratchet_core_window` validates lockfile diffs
  semantically, and `tests/test_ratchet_core_window_tool.py` covers it. The `Publish`
  workflow has succeeded on every master push since (357c45c72, d3c5254ca, 8338a320a,
  191e9f219).
- **sase-m4.2 (strict PDF export).** `mkdocs-pdf.yml` sets `theme.font: false`,
  `tests/test_docs_pdf_tools.py` exists, and `Deploy Docs` has succeeded on every master
  push since.
- **sase-m4.3 items 2-4.** The commit-finalizer baseline fixture (ee6f3c7d3), the
  metavar-aware monitor-help assertion (`tests/main/parser_help_helpers.py`), and the
  `TabQuickStart` `NoMatches` guard plus its regression test are all in place. The
  Python 3.13 "stall" was correctly diagnosed as a slow-but-terminating `just test-cost`
  leg; the 3.13 leg of run 31840230310 again ran ~50+ minutes without hanging.
- **sase-m4.4 (visual baselines).** `render_svg_to_png` passes `font_files`, the goldens
  were regenerated, the CI `visual-test` job passed on runs 31838558537 and 31840230310,
  and `just test-visual` passes locally at 191e9f219 (676 passed, 1 skipped) — so the
  goldens still hold against the commits that landed after the epic started.
- **Integration.** The four commits that landed after the epic's first commit
  (e701dba68, d3c5254ca, 8338a320a, 191e9f219) touch none of the epic's files and
  neither duplicate nor conflict with its changes; the visual run above confirms the
  regenerated goldens survive 191e9f219's new artifact provider tabs.

## Defect 1: the clipboard delivery race in the commit-hint tests is NOT fixed

`sase-m4.3` recorded this as "already fixed at HEAD by a633a29ed/bba5aa19d/4dc323117".
Those three commits changed `tests/ace/tui/modals/test_artifact_files_modal_copy.py` — a
different file. The failing file was never touched, and CI still fails on it:

- `tests/ace/tui/actions/test_view_files_commits.py::test_commit_hint_copy_suffix_copies_short_sha`
- `tests/ace/tui/actions/test_view_files_commits.py::test_multiple_commit_hint_copy_suffix_copies_all_short_shas`

Both fail with `AssertionError: Expected 'mock' to be called once. Called 0 times.` (the
`app.notify` mock), on the `test (3.12)` coverage leg and the `coverage-contexts` job,
in two consecutive master runs: 31838558537 (357c45c72, the epic's own last commit) and
31840230310 (d3c5254ca). They pass locally because the machine is fast; ten consecutive
local runs of the file were green.

Root cause: each test sets `copied_event` from inside the patched
`sase.ace.tui.actions.clipboard._delivery.copy_to_system_clipboard`, which
`deliver_copy()` invokes through `asyncio.to_thread`. The test then does
`await asyncio.wait_for(copied_event.wait(), timeout=1.0)` and immediately asserts on
`app.notify`. Awaiting the worker-thread event only proves the copy callback ran; the
delivery coroutine has not necessarily resumed to reach `_notify(...)`, so under load
the notification assertion runs first. This is exactly the failure the epic plan's Phase
3 item 1 described.

Fix it the way `tests/ace/tui/modals/test_artifact_files_modal_copy.py` (see its
`_wait_for_copy` helper) already does: after the copy lands, drain the pump-free
clipboard task before asserting on the notification. In this harness
`schedule_copy_delivery` spawns the task on `_app_for(owner)`, which for the `_ViewApp`
in `tests/ace/tui/actions/_view_files_helpers.py` is its `app` attribute (the
`SimpleNamespace`), under registry attribute `_pump_free_clipboard_tasks`:

```python
while tasks := tuple(getattr(app.app, "_pump_free_clipboard_tasks", ())):
    await asyncio.gather(*tasks)
    await asyncio.sleep(0)
```

Prefer a shared helper in `tests/ace/tui/actions/_view_files_helpers.py` over copying
the loop into both tests. Keep the existing `copied_event` wait or replace it with the
drain, whichever leaves the strongest assertion; do not weaken the notification
assertions and do not add sleeps or retries.
`tests/ace/tui/actions/test_view_files_reports.py` and `test_view_files_image.py` use a
`MagicMock` for `_copy_files_to_clipboard` and are not affected; a repo-wide grep found
no other test with this pattern.

Acceptance criteria:

- Both node IDs pass, and still pass when the file is run repeatedly and under load (for
  example with the rest of `tests/ace/tui/actions/` running concurrently).
- The tests still assert the exact copied payload and the exact notification.

## Defect 2: the agent-scan performance floor is still too tight for CI

`sase-m4.5` raised the per-anchor `rust_slowdown_factor` for
`scan_agent_artifacts.synthetic_6p_200pp.scan_facade` to `2.15` in
`tests/perf/baselines/phase7_regression_floor.json` (ceiling 257975.73us), calibrated
from three local 12-sample medians of 241.38, 247.51 and 245.47 ms — about 4% headroom.
Local medians do not describe GitHub-hosted runners. CI run 31840230310 measured
**266261.62us** and failed the `perf-floors` job.

Harvested medians for that anchor from the `perf-floors` job of the last 17 master CI
runs that produced one (2026-08-13T22:45Z through 2026-08-14T20:56Z), in ms:

```
146.81  153.05  153.32  175.51  204.96  236.42  237.63  239.29  240.35
243.30  248.75  254.64  254.85  256.23  262.64  266.26  269.78
```

The code did not meaningfully change across most of those runs, so the 146.8-269.8 ms
spread is GitHub-hosted runner heterogeneity, not a regression. The Phase 7B baseline is
119.99 ms, so the worst observed CI median is already 2.25x baseline.

Reproduce and extend this evidence, then choose one of:

1. Recalibrate the per-anchor factor so the ceiling covers the observed CI distribution
   with documented headroom above the worst observed median, and rewrite the anchor's
   `comment` to cite the CI dataset (run IDs and medians) rather than local-host
   samples. Keep the factor as small as the evidence allows and keep the global 1.4x
   gate intact.
2. If profiling shows a tractable win, reduce the measurement instead. `sase-m4.5`'s
   cProfile split a warm scan into roughly 128 ms Rust/dict construction and 156 ms
   Python wire hydration dominated by constructing 1,200 expanded `AgentMetaWire`
   records. Note the Rust core boundary: shared backend behavior belongs in
   `../sase-core`, so prefer recalibration for this tale and file a task bead for a real
   optimization if one looks worthwhile.

Do not calibrate from local-host measurements alone again — the floor gates CI, so CI
measurements are the evidence that matters.

Acceptance criteria:

- The recorded rationale cites repeatable CI-observed medians, not just local samples.
- `just phase7-perf-check` passes locally, and the reasoning explains why the chosen
  ceiling still detects a real facade regression.
- The `perf-floors` job passes on the landed commit's CI run.

## Defect 3: the flake-baseline gate blocks `just check-full`

`just check-full` ends in `tools/selection_health --fail-on-new-flake`, which currently
fails with 16 reproducible flakes beyond `tests/reproducible_flake_baseline.txt`
(records after the 2026-08-10T23:36:35Z cutoff). `sase-m4.6` reported this twice and was
forbidden from filing the required bead; the landing review reconfirmed the identical 16
at 191e9f219:

```
tests/ace/tui/modals/test_snippet_name_modal.py::test_matches_filter_order_and_tab_completion
tests/ace/tui/test_config_center_state.py::test_save_atomically_replaces_existing_state
tests/ace/tui/widgets/test_prompt_panel_header.py::test_family_header_renders_followup_role_attribution
tests/ace/tui/widgets/test_prompt_panel_header.py::test_header_renders_skill_uses_without_memory_reads
tests/main/test_project_handler_list_show.py::TestListAndShow::test_project_handler_imports_in_fresh_interpreter
tests/monitor/test_monitor_start.py::test_start_monitor_promotes_a_bare_lane_and_runs_to_completion
tests/monitor/test_monitor_supervise.py::test_run_supervisor_escalates_term_ignoring_chatty_child
tests/monitor/test_monitor_supervise.py::test_run_supervisor_kills_the_whole_process_group_on_timeout
tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed
tests/test_axe_run_agent_runner_deferred_workspace_outcomes.py::TestDeferredWorkspaceOutcomes::test_repeat_stop_exits_before_workspace_claim_and_run_loop
tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection
tests/test_core_vcs_log.py::test_classify_origin_matches_python_golden[fix: automatic-Details\n\nSASE_TYPE=sase init]
tests/test_core_vcs_log.py::test_classify_origin_matches_python_golden[fix: legacy-Details\n\nSASE_AGENT=sase-1]
tests/test_core_vcs_log.py::test_classify_origin_matches_python_golden[fix: tracked-Details\n\nSASE_TYPE=stitch\nSASE_BEAD=sase-1]
tests/test_core_vcs_log.py::test_parse_computes_auto_origin_from_footer
tests/test_core_vcs_log.py::test_parse_computes_origin_from_footer
```

`tests/test_external_mirror_issues.py::test_creation_budget_defers_then_converges_next_pass`
is reported separately as stale (renamed or deleted) and is already excluded by the
tool; it needs no baseline entry.

None of these live in files epic sase-m4 changed, so they are pre-existing debt, not
epic-caused. The baseline file's own header states the project's rule: "Adding a node
requires a filed SASE bead that names the flake, evidence, and owner." Follow it — file
the bead with `/sase_new_task` (it may corroborate a duplicate instead of creating a new
task; record whatever it decides), then add the 16 node IDs to
`tests/reproducible_flake_baseline.txt` under a comment naming the returned bead ID.
`tests/ace/tui/test_logs_pane.py::test_logs_tab_g_and_shift_g_scroll_detail_extremes` is
already baselined under `sase-jb`; do not duplicate it.

Acceptance criteria:

- A bead exists (new or corroborated) that records these flakes with the evidence above.
- `tools/selection_health --fail-on-new-flake` exits clean, with every added node ID
  attributed to that bead in the baseline file.
- No test is skipped, deselected, or otherwise weakened to pass the gate.

## Also file these follow-ups

Use `/sase_new_task` for each and record the outcome (new task, corroborated duplicate,
or attached to an active epic) so the epic close note can cite it:

- **FORCE_COLOR Rich substring failures** (proposed by `sase-m4.2`): with `FORCE_COLOR`
  set by the agent/CI environment, roughly 118 CLI/TUI tests assert plain substrings
  that Rich emits with ANSI escapes (for example `test_cli_work_from_plan_preview` and
  the plugins pane). They pass with `NO_COLOR=1`. Unrelated to this epic's changes.
- `sase-m4.5`'s two full-suite-only TUI flakes
  (`tests/ace/tui/modals/test_snippet_name_modal.py::test_matches_filter_order_and_tab_completion`
  and
  `tests/ace/tui/test_logs_pane.py::test_logs_tab_g_and_shift_g_scroll_detail_extremes`)
  are a subset of the Defect 3 set; fold them into that bead instead of filing
  separately.

## Verify, land, and observe Actions

1. Run `just install`, then the focused tests for each fix, then `just check`.
2. Run `just check-full` **only** through `/sase_monitor`
   (`sase monitor start --command 'just check-full' ...`), never inline, with a `--next`
   instruction that makes the follow-up agent inspect the result, fix any attributable
   failure, and repeat the monitored full check until it passes. Resolve any unrelated
   pre-existing failure through the task-bead workflow rather than masking it.
3. Commit through `/sase_git_commit`.
4. After the commit lands and triggers Actions, obtain the **exact** CI run ID for that
   commit — do not watch an older queued run. Use `/sase_monitor` to wait for it, for
   example `gh run watch <exact-run-id> --repo sase-org/sase --exit-status`, with an
   appropriate timeout and a `--next` instruction. Note that the `test (3.13)` leg alone
   runs about an hour, so size the timeout accordingly.
5. The next agent must run `actstat` and confirm that the `sase` project reports its
   latest GitHub Actions run as passed, and that every workflow triggered for the landed
   commit — CI, Docs, and Publish — is terminal and successful rather than queued behind
   an older run. `docs-build` and `release-core-floor-smoke` are legitimately `skipped`
   on master pushes (they are pull-request-only jobs); that is not a failure.
6. If `actstat` does not show a stable passing last run, or any workflow is failed,
   cancelled, stalled, or still pending, inspect the new job logs and create a new plan
   with `/sase_plan` before making further file changes. That plan must run
   `sase plan validate --explain`, revalidate after every edit until clean, submit with
   `sase plan propose`, and repeat this same monitored observe-and-replan loop until CI
   is green. Carry the closing step below into that plan's final phase.

## Close out epic sase-m4

Only after a CI run for the landed commit is confirmed green:

1. `sase bead close sase-m4 --note "<what was verified across the epic and this tale>"`.
   The note must record: which phases were verified against real source and CI evidence,
   the two phase acceptance criteria this tale repaired (the clipboard race and the
   agent-scan floor), the integration check against the four post-epic commits, and the
   outcome of every `PROPOSED FOLLOW-UP` — including any that were declined and why. If
   the close is rejected, the named phases were never completed: finish or reopen them.
   Never force merely to make the command succeed.
2. Run `just symvision` and remove the stale entries and unused code it reports —
   epic-symbol whitelist entries for `sase-m4` expire at close. (A repo-wide grep found
   no `sase-m4` pragmas, so this is expected to be a no-op; confirm rather than assume.)
3. Set `status: done` in the frontmatter of the epic's plan file at
   `plan:202608/stabilize_github_actions.md`
   (`/home/bryan/.sase/plans/202608/stabilize_github_actions.md`).

Acceptance criteria:

- `actstat` shows the latest `sase` run passing, and CI, Docs, and Publish are all
  terminal and successful for the landed commit.
- Epic `sase-m4` is closed with a note covering verification, integration, and every
  follow-up outcome.
- `just symvision` is clean and the epic's plan file is marked `status: done`.
