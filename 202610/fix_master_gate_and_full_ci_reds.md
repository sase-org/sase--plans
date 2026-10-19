---
tier: tale
title: Fix the Master Gate and Full CI reds on master
goal:
  Master Gate and every Full CI test leg go green again, with each test race proven
  fixed against a deterministic reproduction.
size: medium
decisions:
  contract_admission:
    ask:
      Should the two new workflow-YAML guards join the contract set, raising its entry
      cap from 75 to 77?
    choices:
      admit:
        Cap becomes 77; workflow edits to install-smoke and testpypi-drill keep scoped
        check coverage
      unmark:
        Cap stays 75; those two workflows lose scoped coverage and rely on CI's
        exhaustive lane
    default: admit
    why:
      .github edits seed no tests, and every other workflow-YAML guard is already
      admitted
    answer: admit
proposed_by: bbugyi200.athena.127.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 17:37:00
status: wip
---

# Fix the Master Gate and Full CI reds on master

## Context

`actstat` shows the sase repo red on master (HEAD `4e5b7d2eb6`) in **Master Gate**
(shards 7 and 8) and **Full CI** (`test (3.12)`, `test (3.13)`, `test (3.14)`). I pulled
every failed-job log for Master Gate runs 39290035777, 39295873153, and 39313695968, and
for Full CI runs 39294492670 and 39311934512. They show four independent defects. Lint,
visual-test, docs, and perf-floors are green. All four defects are in tests, and each
one was reproduced or proven during planning:

| #   | Failing node(s)                                                                                                                                                                                                                                                   | Where                                                              | Root cause                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`                                                                                                                                                                                | every Master Gate shard 7, every Full CI leg                       | `tests/test_github_actions_install_smoke.py` (6a0090f0d9) and `tests/test_github_actions_testpypi_drill.py` (2a83edf996) landed with `pytestmark = pytest.mark.contract` but without `just refresh-contract-manifest`. The marker now selects 77 files and the manifest lists 75; no files were removed.                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| 2   | `tests/ace/tui/test_link_follow_bounded_panes.py::test_archived_plan_reveals_through_path_context_after_scan`, `tests/ace/tui/test_link_follow_entry_points.py::test_matrix_archived_plan_via_links_panel` (`assert 'kind:active' == 'path:…/202607/archive.md'`) | Master Gate shard 8, intermittent (2 of the last 3 runs)           | Test race. Both tests submit `kind:active` and then only wait for `pane.host_limit_query() == "kind:active"`. That value becomes true synchronously when the filter is committed. Query evaluation runs on a thread worker (`ArtifactQuerySession.result()` → `run_worker(thread=True)`), and `_refresh_options()` returns early on `pending_query` without rebuilding the option list. So the archived row is still on screen when the follow starts. `request_entry_target()` → `select_entry_target()` then finds it and completes `SELECTED` without ever reaching the `path:` context rung. **Reproduced deterministically**: wrapping `query_session.evaluate_artifact_query_many` with a 0.5 s sleep makes both nodes fail with exactly the CI assertion. |
| 3   | `tests/main/test_parser_stats.py::test_project_help_golden`, `::test_project_help_includes_definitions_and_examples`                                                                                                                                              | Full CI `test (3.13)` and `test (3.14)` only (Master Gate is 3.12) | `sase stats project -h` (a53f10f6a6) has a single golden captured with Python 3.12 argparse, which renders `-f {json,markdown,rich}, --format {json,markdown,rich}`. Python 3.13+ renders `-f, --format {json,markdown,rich}`, and that also moves short invocations such as `-r, --range RANGE` onto the help column. The substring test hard-codes the 3.12 spelling. **Proven**: the real 3.14 help and a stdlib re-render of the same parser on 3.13.13 and 3.14.7 are byte-identical (sha256 `2256dc39445272a8c2e00caf6146079cbd37cd5026445ad431f22f5443a2be00`). A 3.12.13 re-render is byte-identical to the committed golden (sha256 `4891b34bcf403a3a337b40c8675ef631bf4399c8de8839d97e2ded93360b7d81`).                                                |
| 4   | `tests/ace/tui/test_statistics_pane_interactions.py::test_description_rail_cannot_steal_focus_or_bindings` (`assert 2 == 1`)                                                                                                                                      | Full CI `test (3.12)` coverage leg (slow lane)                     | Test race introduced by 0f2052be6a (sase-1m3.11). That commit made Projects its own report, so switching to view 3 now schedules a debounced `load_statistics_view("projects", …)` (`_set_view` → `needs_reload` → `_schedule_load`). The test still asserts `len(calls) == 1` right after `wait_for(pane._view == "projects")`. On a slow host the debounce fires first and records a second, `projects` call. The elided CI repr hid that view name. **Reproduced**: making `_schedule_load` start immediately fails the test with the same `assert 2 == 1`, and `-vv` shows calls `('overview', …), ('projects', …)`.                                                                                                                                         |

**Not actionable here (record in the final report only):**

- `RuntimeError: tool run store is busy: database is locked` in
  `tests/tool/test_settlement_retention.py::test_handoff_end_to_end_publishes_one_notification`
  appeared once (Master Gate 39291685051, shard 8). This is the open flake bead
  **sase-18t**, which already carries evidence for this exact node. Corroborate it in
  step 6; do not fix it here.
- **Install smoke**: the push-triggered `0s` failures were the "Unrecognized
  named-value: runner" workflow-file error, already fixed by a9046529ca. The workflow is
  now schedule/dispatch only. The later red dispatch 39298650862 is an intentional
  red-path drill (`force-expect sase==0.0.0`), and the dispatch next to it is green.
- `sase-org/sase-core` CI (`release workflow scripts: Set up job`) belongs to a
  different repo and already has an in-flight fix
  (`cbc08c5 fix(ci): pin setup-uv to published release tags`).

All changes are test-only. No product code, Rust core, memory, or docs change. Do
**not** run `just check-full` (it is explicit-only); verify with `sase tool run check`
plus the targeted runs below.

## Steps

### 1. Contract manifest (defect 1)

> [!decision] contract_admission = admit

1. Run `just refresh-contract-manifest` and confirm the only diff to
   `tests/contract_manifest.txt` adds `tests/test_github_actions_install_smoke.py` and
   `tests/test_github_actions_testpypi_drill.py`, giving 77 entries.
2. Measure the refreshed set with the documented command:
   `.venv/bin/python -m pytest -m contract $(cat tests/contract_manifest.txt) -p no:randomly --durations=0`.
   Record the wall time, the test count, and the host load average (`uptime`). For
   reference, the two added files ran 22 tests in 4.10 s standalone during planning.
3. In `tests/test_contract_manifest.py`, set `_MANIFEST_ENTRY_BUDGET = 77` and update
   `_MEASURED_SERIAL_COST` to the measured `"<N> serial seconds across 77 entries"`.
   Append a dated paragraph in the established style to the re-curation comment block
   above them, for example: "Re-curated to 77 on 2026-10-19 for
   `test_github_actions_install_smoke.py` (sase-1my.2) and
   `test_github_actions_testpypi_drill.py` (sase-1my.5), which landed marked `contract`
   without a manifest refresh …". The paragraph must say:
   - they are checked-in workflow-YAML guards, a data asset no import edge reaches, so
     the contract set is their only scoped coverage, the same basis on which
     `test_github_actions_ci_master_gate.py` and its siblings were admitted;
   - the standalone cost and the measured whole-set figure, with any host-load caveat;
   - the existing closing line that the next candidate should displace an entry.

> [!decision] contract_admission = unmark

Remove the `pytestmark = pytest.mark.contract` line (and the now-unused `import pytest`,
if nothing else uses it) from both files. Add a short comment in each saying why it is
deliberately unmarked, mirroring the one in `tests/test_run_pytest_shards.py`. Leave the
manifest, the cap, and the cost comment unchanged. Confirm that
`.venv/bin/python tools/refresh_contract_manifest` leaves `tests/contract_manifest.txt`
unchanged.

### 2. Archived-plan link-follow race (defect 2)

1. In `tests/ace/tui/test_link_follow_bounded_panes.py`, add a helper next to
   `_open_plans`. It commits the filter and waits until the query result has actually
   hidden the target, not merely until the filter is committed:

   ```python
   async def _commit_kind_active_hiding(
       page: AcePage, pane: Any, target: ArtifactEntryTarget
   ) -> None:
       """Commit ``kind:active`` and wait until its query result hides *target*.

       ``host_limit_query()`` reports the committed query at once, but the Rust
       query runs on a worker and the option list keeps the pre-query rows until
       it lands. Following before then selects the still-rendered archive row
       and never needs the ``path:`` context these tests exercise.
       """
       pane.query_one(PlanFilterBar).post_message(PlanFilterBar.Submitted("kind:active"))
       await page.wait_for(
           lambda _state: pane.host_limit_query() == "kind:active"
           and all(plan_row_target(row) != target for row in pane._rows.values())
       )
   ```

   Adjust the imports and annotations to match the module (it already imports
   `PlanFilterBar`, `plan_row_target`, and `ArtifactEntryTarget` for its tests).

2. In `test_archived_plan_reveals_through_path_context_after_scan`, replace the
   `bar = …` / `bar.post_message(...)` / `wait_for(host_limit_query == "kind:active")`
   lines with `await _commit_kind_active_hiding(page, pane, target)`. Keep the following
   `assert pane.selected_entry_target() != target`.
3. In
   `tests/ace/tui/test_link_follow_entry_points.py::test_matrix_archived_plan_via_links_panel`,
   import the helper together with `_install_truncated_archived_plans` and `_open_plans`
   (from `tests.ace.tui.test_link_follow_bounded_panes`) and make the same replacement.
   Drop the local `PlanFilterBar` import if it becomes unused.
4. Prove the fix against the reproduction. Keep the probe **outside the repo**, e.g. in
   `/tmp/ci_probe/_probe_slow_query.py`:

   ```python
   import time
   import pytest
   from sase.ace.tui.widgets.artifacts import query_session as qs

   @pytest.fixture(autouse=True)
   def _slow_query(monkeypatch):
       orig = qs.evaluate_artifact_query_many
       def slow(*args, **kwargs):
           time.sleep(0.5)
           return orig(*args, **kwargs)
       monkeypatch.setattr(qs, "evaluate_artifact_query_many", slow)
   ```

   Run
   `PYTHONPATH=/tmp/ci_probe .venv/bin/python -m pytest -p _probe_slow_query -p no:randomly`
   on both nodes. Both must fail on the pre-fix tree (as they did in planning) and pass
   after the fix. Then run both modules normally a few times, including under
   `-n 8 --count`-style repetition if `pytest-repeat` is available, or a short shell
   loop otherwise.

### 3. Python-version-specific `sase stats project -h` golden (defect 3)

1. In `tests/main/test_parser_stats.py`, select the golden by interpreter:

   ```python
   # Python 3.13 argparse stopped repeating an option's metavar after its short
   # alias ("-f, --format {…}" rather than "-f {…}, --format {…}"), which also
   # moves short invocations onto the help column. 3.13 and 3.14 render this
   # parser identically, so one golden per spelling covers requires-python.
   _HELP_GOLDEN = (
       "project_help_py313.txt" if sys.version_info >= (3, 13) else "project_help.txt"
   )
   ```

   Use `GOLDEN / _HELP_GOLDEN` in `test_project_help_golden`, including the
   `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1` write path. Add a one-line note there that
   update mode rewrites only the running interpreter's golden.

2. Create `tests/stats/golden/cli/project_help_py313.txt` by running on a Python 3.13+
   venv (check `.venv/bin/python --version`):
   `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1 .venv/bin/python -m pytest tests/main/test_parser_stats.py::test_project_help_golden -p no:randomly`.
   Its sha256 must be
   `2256dc39445272a8c2e00caf6146079cbd37cd5026445ad431f22f5443a2be00`.
   `tests/stats/golden/cli/project_help.txt` must stay byte-identical (sha256
   `4891b34bcf403a3a337b40c8675ef631bf4399c8de8839d97e2ded93360b7d81`). Inspect the new
   file: only the option invocations and their wrapping should differ from the 3.12
   file.
3. In `test_project_help_includes_definitions_and_examples`, replace the four
   metavar-bearing substring assertions (`-f`, `-g`, `-p`, `-r`) with the existing
   portable helper `assert_metavar_option_documented` from
   `tests/main/parser_help_helpers.py`, which already accepts both argparse spellings.
   Keep the bare-flag assertions (`-D, --include-disabled`, `-F, --fetch`, `-j, --json`)
   as they are:

   ```python
   assert_metavar_option_documented(help_text, "-f", "--format", "{json,markdown,rich}")
   assert_metavar_option_documented(
       help_text, "-g", "--group-by", "{model,patch,project,repo}"
   )
   assert_metavar_option_documented(help_text, "-p", "--project", "PROJECT")
   assert_metavar_option_documented(help_text, "-r", "--range", "RANGE")
   ```

4. Run `tests/main/test_parser_stats.py` on the local 3.13+ venv. Master Gate (3.12)
   exercises the 3.12 golden.

### 4. Statistics description-rail test race (defect 4)

In
`tests/ace/tui/test_statistics_pane_interactions.py::test_description_rail_cannot_steal_focus_or_bindings`,
move the no-reload assertion before the view switch, then settle and pin the one
expected Projects load. Import `_wait_statistics_settled` from
`tests.ace.tui._statistics_pane_helpers`, where it already exists:

```python
        await page.press("escape")
        await page.pause()
        # Neither the rail click nor the cancelled custom-range editor reloads.
        assert len(calls) == 1
        await page.press("0", "3")
        await page.wait_for(lambda _state: pane._view == "projects")
        _assert_statistics_chrome(pane)
        # Projects is its own report (sase-1m3.11), so the switch schedules
        # exactly one debounced Projects load.
        await _wait_statistics_settled(page, pane)
        assert [call[0] for call in calls[1:]] == ["projects"]
```

Prove it against the reproduction with an out-of-repo probe (e.g.
`/tmp/ci_probe/_probe_fast_debounce.py`) that patches
`sase.ace.tui.modals.statistics_pane.StatisticsPane._schedule_load`. The patch returns
if `not self._is_active_tab()`, otherwise calls `self._start_load()` and then
`time.sleep(0.3)`. Run the node with
`PYTHONPATH=/tmp/ci_probe … -p _probe_fast_debounce`. It must fail before the fix and
pass after. Then run the whole module normally.

### 5. Verify

1. Run `just fix` (or at least `just fmt`) inline.
2. Run `sase tool run check`. Use a `verify` monitor via `/sase_monitor` if it will not
   fit the synchronous limit, and treat any UNKNOWN triage as yours. `.github/**` is not
   touched, so the scoped lane will select the edited tests and the contract set.
3. Also run these directly:
   `.venv/bin/python -m pytest tests/test_contract_manifest.py tests/main/test_parser_stats.py tests/ace/tui/test_link_follow_bounded_panes.py tests/ace/tui/test_link_follow_entry_points.py tests/ace/tui/test_statistics_pane_interactions.py -p no:randomly`.
   `test_contract_manifest.py` is deliberately unmarked and runs only in the exhaustive
   lane, so run it explicitly.
4. Do not run `just check-full`.

### 6. Discovered work (use `/sase_new_task` for each; it dup-checks and routes corroboration)

1. **Corroborate sase-18t**: add the Master Gate 39291685051 shard-8 occurrence of
   `test_handoff_end_to_end_publishes_one_notification` hitting
   `tool run store is busy: database is locked`. Do not create a new bead.
2. **Bug (product race)**: a link follow issued while the Plans pane's committed query
   is still evaluating can select a row that the pending result is about to hide.
   `request_entry_target()` → `select_entry_target()` trusts the pre-query option list.
   When the result lands, `_refresh_options()` drops the row and the selection moves,
   even though the follow already reported `SELECTED`. Evidence: the slow-query probe
   from step 2.4 on the unfixed test path. Beads and other `ArtifactQuerySession` panes
   likely share the shape. Size it per `/sase_new_task`; the narrow root cause is known.
   Do not fix it in this tale. Earlier fixes in this family (sase-1mk.7, sase-1mk.9.3)
   show how easily link-follow changes regress.

## Acceptance

- `tests/contract_manifest.txt` matches the `-m contract` selection, and the cap and
  cost comment match the chosen `contract_admission` branch.
- Both archived-plan link-follow nodes pass normally and under the slow-query probe.
- `tests/main/test_parser_stats.py` passes on 3.13+ locally. The 3.12 golden is
  unchanged, and the new 3.13+ golden has the sha256 given above.
- The statistics rail test passes normally and under the fast-debounce probe.
- `sase tool run check` is green, apart from labelled KNOWN/FLAKY items owned by
  existing beads.
- The final report names the Master Gate and Full CI runs that should go green on the
  next push and lists the non-actionable items (sase-18t, Install smoke, sase-core).
