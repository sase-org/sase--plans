---
tier: tale
title: Fix the failing CI test jobs
goal:
  Every Master Gate test shard and Full CI test leg passes on master again, with the two
  real product bugs fixed rather than masked.
size: medium
decisions:
  docs_revive_wording:
    ask: Also fix the two stale "revive" jump-legend passages in docs/ace.md?
    default: true
    why: The fixed test pins Archive wording; the guide sentences are now visibly wrong
    answer: true
  glossary_jump_target:
    ask:
      Correct the agent-relation-jump-target glossary strand's stale "revived" wording?
    memory:
      - glossary:agent-relation-jump-target
    default: true
    corrects: dismissed target is revived instead.
    answer: true
proposed_by: bbugyi200.athena.0zw
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-13 06:26:38
status: wip
---

# Fix the failing `test` jobs in CI (Master Gate shards + Full CI test legs)

## Context

Every `Master Gate` push run on master fails its `test (N)` shards, and every `Full CI`
`test (3.12/3.13/3.14)` leg fails too. I pulled the failure summaries from the last 11
Master Gate runs and 6 Full CI runs and re-ran each failing node locally at master HEAD.
Seven tests fail on every run and reproduce locally. One more test fails intermittently
on a busy store. Every other failure in that history already passes at HEAD and is out
of scope. That covers presentation-audit stale exemptions,
`test_neighbors_map_shows_dismissed_revive_cells`,
`test_new_project_apply`/`test_new_project_source`, parser help, and completion kind
coverage.

The `lint` and `visual-test` jobs also fail, but the user asked for the `test` jobs
only. Leave them alone.

| #   | Failing node                                                                                                                      | Root cause                                                                                                                                                                                                                                                                                                                                  | Verdict               |
| --- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 1   | `tests/ace/tui/widgets/test_agent_jump_legend.py::test_dismissed_marker`                                                          | `101c853939` (retire Artifacts Agent pane) made a dismissed-target digit open the agent read-only in the Archive. It renamed the narrowed legend note from `revive` to `archive` and updated `test_agent_jump_panel_numbers.py`, but missed this file.                                                                                      | Stale test            |
| 2   | `tests/ace/tui/test_proc_producer_inventory.py::test_inventory_matches_live_production_source`                                    | `35e9768444` (split new project apply flow) moved `_submit_project_add` and `_submit_project_create` into `_new_project_apply_flow.py`. The proc-producer inventory still names `new_project_apply.py`.                                                                                                                                     | Stale inventory       |
| 3   | `tests/test_agent_display_meta_priority.py::TestMetaPatchPriority::test_project_shown_when_no_patch`                              | **Product regression** from `df61663204`. See "Project label regression" below.                                                                                                                                                                                                                                                             | Source bug            |
| 4   | `tests/ace/tui/widgets/test_agent_display_workflow_async.py::test_workflow_snapshot_render_reads_state_once_and_uses_parsed_data` | Same regression as #3.                                                                                                                                                                                                                                                                                                                      | Source bug            |
| 5   | `tests/ace/tui/modals/test_node_finder_modal.py::test_rapid_keystrokes_coalesce_to_final_query` (`assert 2 == 1`)                 | **Product perf bug.** Every refilter rebuilds the option list twice. See "Node Finder double rebuild" below.                                                                                                                                                                                                                                | Source bug            |
| 6   | `tests/ace/tui/modals/test_node_finder_modal.py::test_dismissed_modal_releases_snapshot_rows_without_gc` (`assert N == N+1`)      | `ba89d2a6db` made the Archive projection unconditional. The modal now holds two views (`_live_view` plus an Archive-composed `_view` copy), and the release swaps both for one shared empty view. The count therefore ends one below a baseline taken after construction. Nothing leaks.                                                    | Stale test arithmetic |
| 7   | `tests/ace/tui/test_agent_archive_parking.py::test_quitting_from_archive_persists_grouping_only` (`'day' == 'project'`)           | `ba89d2a6db` (lazy-load grouping) stopped startup from reading the saved grouping. `init_agents_archive_view_state` now leaves grouping at `ARCHIVE_DEFAULT_GROUPING`, and the saved value loads in the background on the next Archive open. The sibling `test_startup_does_not_read_archive_grouping_preference` asserts this.             | Stale test            |
| 8   | `tests/tool/test_detach.py::test_watchdog_stops_unjoined_run_when_starter_dies` (also `test_watchdog_reports_ended_join_monitor`) | Intermittent: 3 of the last 11 Master Gate runs. It made shard 8 red on the latest run. `_settle_run_id` polls `tool_run_show` with its default 250 ms `busy_timeout_ms` while the detached runner/watchdog is writing the same SQLite store. On a loaded CI runner that raises `RuntimeError: tool run store is busy: database is locked`. | Test-harness flake    |

## Changes

### 1. Jump legend test (stale wording)

`tests/ace/tui/widgets/test_agent_jump_legend.py`, in `test_dismissed_marker`: change
`assert "revive" in narrowed` to `assert "archive" in narrowed`. Do not change the
source; `_agent_jump_legend.py` is correct.

> [!decision] docs_revive_wording In `docs/ace.md` (Agents tab jump-panel section,
> around the "`Ctrl+O` jump-back history" paragraph), replace "A digit on a dismissed
> neighbor revives that agent instead of jumping." with wording saying the digit opens
> that agent read-only in the Archive instead of jumping. In the jump-legend description
> further down (around the "`⊘` prefix (and a `revive` note when narrowed)" sentence),
> change it to an `archive` note and say the digit opens the agent read-only in the
> Archive. Only touch these two passages; other `revive`/`revivable` mentions describe
> real revive features and stay.

> [!decision] glossary_jump_target Use `/sase_memory_write` (this decision authorizes
> it). In `sase/memory/glossary/agent-relation-jump-target.md`, reword "a dismissed sase
> agent that the keymap revives" and "a dismissed target is revived instead." so they
> say the number opens the dismissed agent read-only in the Archive. Then run
> `sase memory init`. If the decision was declined and no human reviewed the plan, file
> a `memory` task bead through `/sase_new_task` instead.

### 2. Proc-producer inventory paths

`src/sase/ace/tui/_proc_producer_sites_actions.py` has two path strings,
`"src/sase/ace/tui/modals/new_project_apply.py"`: one in the `projects.add` site and one
in the `projects.create` site. Change both to
`"src/sase/ace/tui/modals/_new_project_apply_flow.py"`. The owner strings
(`NewProjectApplyMixin._submit_project_add` / `_submit_project_create`) are still
correct. This clears both the stale entries and the unlisted entries the next assertion
would report. No other inventory, allowlist, or exemption table references the moved
modules.

### 3. Project label regression (source fix)

`df61663204` ("show PROJECT*NAME, never directory key") changed `project_display_label`
in `src/sase/ace/tui/widgets/prompt_panel/_helpers.py` from
`agent.project_display_name or str(fallback)` to `agent.project_label or str(fallback)`.
`Agent.project_label` is never `None` for an agent with a `project_file`, so the
`fallback` argument became dead. A workflow's `meta_project` (callers in
`_agent_display_header_metadata_identity.py` `elif meta_project:` and
`_workflow_render.py` "Project/Patch with meta*\* priority") is therefore silently
replaced by the **agent's own directory key** (`fake`, `tmp`). That violates both
docs/ace.md ("`meta_project` … promoted into the normal header fields") and the
`project-labels-never-keys` decision, because a path-derived key shows where a project
name belongs.

Do not restore the old `project_display_name or …` expression.
`tests/_project_display_presentation_audit_fallbacks.py::test_project_display_name_fallback_lives_only_in_chokepoint`
forbids that fallback anywhere outside `Agent.project_label`. Do not add any new
`project_file`-derived expression either, because
`test_project_labels_never_derive_from_paths` would flag it. Rewrite the helper so the
argument is treated as a project key:

```python
from sase.project_display_names import project_display_name_for


def project_display_label(agent: Agent, project_key: object) -> str:
    """Return the display label for *project_key* in an agent's project field.

    A project agent's own key resolves through ``Agent.project_label`` (so an
    attached ``project_display_name`` wins); any other key, such as a workflow's
    ``meta_project``, projects through the display map instead of being replaced
    by the agent's directory key.
    """
    key = str(project_key)
    if agent.is_project_agent and key == agent.cl_name:
        return agent.project_label or key
    return project_display_name_for(key)
```

Keep the import at module top if that creates no import cycle; otherwise make it a local
import, as `Agent.project_label` does. Rename the parameter at all four call sites only
if that reads better; the positional calls need no change. I verified this exact body in
memory. With it, the two target tests,
`tests/ace/tui/widgets/test_agent_display_project_names.py` (including the
`meta_project == own key` cases that require the attached display name to win),
`tests/ace/tui/test_agent_project_label_freshness.py`,
`tests/test_project_display_names.py`, and
`tests/test_project_display_presentation_audit.py` all pass (57 passed). So do the 23
test files that mention `meta_project`/`Project: ` (246 passed). Add one focused
regression test next to `test_project_shown_when_no_patch`. It uses a project agent
whose `meta_project` names a _different_ key and asserts that the header shows that key
and not the agent's own directory key.

### 4. Node Finder double rebuild (source fix)

In `src/sase/ace/tui/modals/_node_finder_modal_filtering.py`, `_apply_refilter` calls
`self._schedule_archive_search(query)`. In
`src/sase/ace/tui/modals/_node_finder_modal_archive.py`, both branches of that method
call `self._refresh_archive_projection()`: the "Archive warming…" early return and the
trailing "Searching Archive…". That call composes a view and runs `_rebuild_options`,
`_paint_chrome`, and `_paint_preview`. `_apply_refilter` then composes a second view and
repeats all three. Every keystroke therefore pays two full option-list rebuilds, which
breaks the module's own promise that rapid keystrokes "collapse to a single list
rebuild". The bug was latent since `db32983ba2` and exposed when `ba89d2a6db` made the
Archive projection unconditional.

Fix:

- `_schedule_archive_search(self, query: str, *, refresh: bool = True)`. Guard both
  `self._refresh_archive_projection()` calls with `if refresh:`. The message and matches
  state updates still happen unconditionally.
- `_apply_refilter` calls `self._schedule_archive_search(query, refresh=False)`, since
  it composes, rebuilds, and paints itself right after.
- `_finish_archive_name_load` keeps the default `refresh=True`.

The final screen is unchanged: `_apply_refilter`'s own compose runs after the message
and matches are set. Do not loosen the test's `== 1` expectation.

### 5. Node Finder release test (stale arithmetic)

In
`tests/ace/tui/modals/test_node_finder_modal.py::test_dismissed_modal_releases_snapshot_rows_without_gc`,
make the view-count baseline independent of how many views the modal holds internally:

- Call `gc.collect()` before taking the baselines (`base_rows` and the view baseline).
  This keeps earlier tests' cyclic garbage from being collected mid-measurement.
- Take `base_views = _live(NodeFinderView)` **before** `NodeFinderModal(...)` is
  constructed.
- After the pop, assert `_live(NodeFinderView) == base_views + 1`: only the single
  shared empty replacement survives.
- Update the comment. The modal holds its live view plus an Archive-composed copy, and
  release leaves one shared empty view. Keep the snapshot and row assertions as they
  are.

### 6. Archive grouping parking test (stale after lazy load)

`tests/ace/tui/test_agent_archive_parking.py::test_quitting_from_archive_persists_grouping_only`:
convert it to `async def` (the repo runs `asyncio_mode = "auto"`; match the file's
existing async style or markers, if any), and import `ARCHIVE_DEFAULT_GROUPING` from
`sase.ace.tui.models.agent_archive_view`. After `init_agents_archive_view_state(app)`:

- Replace `assert state.archive.grouping == "project"` with
  `assert state.archive.grouping == ARCHIVE_DEFAULT_GROUPING`. Startup must not read the
  preference.
- Keep the other resets and `load_agent_archive_grouping() == "project"`.
- Then prove the preference still survives a relaunch: call
  `open_agents_archive_view(app)`, await the archive controller tasks the open spawned
  (`app._archive_controller_tasks`, which holds the `sase-archive-grouping-preference`
  task), and assert `app._agents_archive_view.archive.grouping == "project"`.

Use the repo's `wait_for`/`sase-test-wait` helpers if a bare `asyncio.gather` over the
task set is not enough. If `_ParkingApp` cannot drive the lazy load under an event loop,
fall back to the synchronous default-grouping assertion and state why in the test
docstring.

### 7. Detached-run poll flake

`tests/tool/test_detach.py::_settle_run_id`: a contended store must not fail the poll.
Pass a generous busy timeout to both `tool_run_show` calls (`busy_timeout_ms=5_000`).
Production code already does this for its own deadline-bound reads in
`src/sase/tool/executor_triage.py`. Also treat a `RuntimeError` whose message contains
`"tool run store is busy"` as a missed poll: sleep and retry until the existing
deadline, and re-raise only if the deadline passes. Leave the assertions alone.

## Out of scope / follow-ups

- Full CI's 3.13 leg also failed once each on
  `tests/llm_provider/test_subprocess_utf8_decode.py::test_stream_json_lines_dispatches_stderr_during_a_large_stdout_burst`
  (`assert 1200 < 1200`) and
  `tests/test_models_panel_runner_limit.py::test_title_footer_and_chooser_show_effective_and_configured_limits`
  (5 s `wait_for` timeout). Both pass locally and appeared only once. Do not fix them
  here. Run `/sase_new_task` for each and file a `flake` task bead, or corroborate an
  existing one.
- The `lint` and `visual-test` job failures are not part of this plan.

## Verification

1. Run each target node directly and confirm it passes:
   `.venv/bin/python -m pytest -p no:cacheprovider -q -n 0 <node>` for the seven
   deterministic nodes above, plus both `test_watchdog_*` detach tests.
2. Run the neighbor suites the fixes touch:
   - `tests/ace/tui/modals/test_node_finder_modal.py` and
     `tests/ace/tui/actions/test_node_finder_archive.py`
   - `tests/ace/tui/test_agent_archive_parking.py`,
     `tests/ace/tui/test_agent_archive_view_foundation.py`, and
     `tests/ace/tui/test_agent_archive_remaining_proof.py`
   - `tests/ace/tui/widgets/test_agent_jump_*.py`
   - `tests/ace/tui/test_proc_producer_inventory.py`
   - the project-label set listed in §3
   - `tests/tool/test_detach.py`
3. Run the two `test_node_finder_modal.py` tests a second time inside the full module
   run, so they pass with other tests' garbage present and not only in isolation.
4. Run `sase tool run check`, per the lint_and_test memory note. Do not run
   `check-full`.
