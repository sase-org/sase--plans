---
tier: tale
title: Fix the ten deterministic just check failures on master
goal:
  The ten full-suite failures on a clean master tree pass. The emptied Agents-tab panel
  retires within one visual frame. The tracking task beads are closed.
size: medium
proposed_by: bbugyi200.athena.sase-165.6.f0
create_time: 2026-09-22 11:12:29
status: wip
---

# Fix the ten deterministic `just check` failures on master

## Problem

Every `just check` run whose scoped lane escalates to the full suite ends with the same
10 failures on a clean master tree. Re-running only those nodes at master `7c763a2e7`
gives `10 failed, 14 passed`. That turns unrelated agents' verification red and has
already blocked several phase/land closes (sase-165.4, sase-165.5, sase-165.6,
sase-14n.15, sase-15p). Four of the five root causes are tests that were not updated
after an intentional behavior change. The fifth, the epic-panel arrival frame, is a real
one-frame visual regression in the Agents tab.

Failing nodes:

1. `tests/test_bead/test_cli_at_path_values.py::test_every_bead_free_text_option_is_classified`
2. `tests/llm_provider/test_usage_config.py::test_usage_indicator_defaults_and_overrides`
3. `tests/llm_provider/test_usage_config.py::test_usage_indicator_real_bundled_default_and_user_override_projection[empty-provider-map-keeps-bundled-exact-window]`
4. `...[unrelated-provider-override-keeps-claude-defaults]`
5. `...[broader-provider-default-loses-to-exact-window]`
6. `tests/test_commit_bead_hooks.py::TestHandleBeads::test_assigned_bead_syncs_without_reading_or_closing`
7. `tests/test_commit_bead_hooks.py::TestHandleBeads::test_bead_sync_runs_when_bead_dir_exists`
8. `tests/test_commit_bead_hooks.py::TestHandleBeads::test_bead_sync_runs_when_split_sidecar_exists`
9. `tests/ace/tui/test_epic_panel_arrival_frames.py::test_a_removal_that_collapses_a_panel_settles_the_column_in_frame`
10. `tests/ace/tui/test_session_proc_reporter.py::test_session_reporter_uv_runner_streams_through_stderr_adapter`

Existing tracking to settle when done:

- **sase-15z** (ready task): items 1 (the `read` options) and 6-8.
- **sase-14u and sase-14v** (ready tasks, duplicates of each other): items 2-5.
- **sase-14j** (in-progress epic): a DISCOVERED ISSUE note for the `('touched', 'verb')`
  part of item 1.
- **sase-158.6** (in-progress epic): a DISCOVERED ISSUE note for item 10.
- Item 9 has no bead.

Those epics' land agents may fix their piece first. Before editing each test, re-run its
node on current master. If it already passes, skip that step and do not duplicate the
fix.

## Changes

### 1. Classify the new bead free-text options (item 1)

Root cause: `0b3061047` added `sase bead read -P/--project` and `-r/--reason`
(`src/sase/main/parser_bead_queries.py`). `319fe6b24` added
`sase bead touched -v/--verb` (`src/sase/main/parser_bead_touched.py`). Neither updated
the classification sets in `tests/test_bead/test_cli_at_path_values.py`.

Add all three to `_DELIBERATELY_LITERAL_FREE_TEXT`:

- `("read", "project")` and `("touched", "verb")` belong in the "Selectors, times,
  paths, and filter names" group, next to the existing `("show", "project")`.
- `("read", "reason")` stays literal. `src/sase/bead/bead_reads.py` records the audit
  reason verbatim and never calls `read_at_path_value`, matching the one-line `-r`
  reasons of `sase artifact read` and `sase memory read`. Add a short comment saying
  audit reasons are recorded verbatim. Do not add `@path` expansion.

### 2. Commit bead-hook mock expectations (items 6-8)

Root cause: `0b3061047` made
`src/sase/workflows/commit/bead_hooks.py::_run_bead_command` always forward `env=env` to
`subprocess.run`. `env` defaults to `None`, and only the `sase bead show` lookup passes
a real env (`SASE_BEAD_SKIP_VIEW_LOG=1`). Production behavior is correct.

In `tests/test_commit_bead_hooks.py::TestHandleBeads`, add `env=None` to the three
`assert_called_once_with(["sase", "bead", "sync"], ...)` expectations. Leave production
code unchanged.

### 3. Usage-indicator tests follow the generic Fable threshold (items 2-5)

Root cause: `c00964773` ("govern Claude Fable usage indicator by generic threshold")
intentionally removed the shipped
`providers.claude.windows["weekly:claude-fable-5"]: always` entry from
`src/sase/default_config.yml`. It added
`tests/llm_provider/test_claude_fable_usage_indicator_default.py` for the new shipped
behavior but left `tests/llm_provider/test_usage_config.py` asserting the old pin. The
shipped config is authoritative. Do not restore the pin.

Update `tests/llm_provider/test_usage_config.py`:

- `test_usage_indicator_defaults_and_overrides`:
  - Replace the `settings.config["providers"]["claude"]...` assertion with
    `"claude" not in settings.config["providers"]`. Muse is the only bundled provider
    entry.
  - Expect `_projected_window_keys() == ("weekly", "session-low")`. The snapshot's Fable
    window defaults to 100% remaining, which the generic 20% threshold hides.
  - Keep the override half of the test unchanged.
- In the parametrized projection test:
  - `empty-provider-map...` expects `("weekly", "session-low")`. Rename its id to
    describe the new meaning, for example `empty-provider-map-keeps-bundled-defaults`.
  - `unrelated-provider-override-keeps-claude-defaults` expects
    `("weekly", "session-low")`.
  - `broader-provider-default-loses-to-exact-window` still has to test precedence, but
    the bundled config no longer has an exact key. Give the user config both
    `claude.default: never` and `claude.windows["weekly:claude-fable-5"]: always`, and
    keep the expectation `("weekly:claude-fable-5",)`.
  - The `exact-key-never-wins`, `exact-key-threshold-restores-generic-boundary`, and
    `indicator-disabled` cases already pass. Leave them alone.
- Do not duplicate what `test_claude_fable_usage_indicator_default.py` already covers.

### 4. Session reporter uv-runner test double (item 10)

Root cause: the `sase update` live-progress work (`94ccd1917`/`5a89392fe`) made
`SessionProcReporter.uv_runner()` (`src/sase/ace/tui/session_proc_reporter.py`) forward
`on_output=` to `sase.uv_tool.runner.run_uv`. The test's `fake_run_uv` does not accept
that keyword.

In `tests/ace/tui/test_session_proc_reporter.py`:

- Give `fake_run_uv` an `on_output: object = None` keyword and record it.
- Assert the default call forwards `None`.
- Add a second call, or a small sibling test, that passes a sink through `uv_runner()`
  and asserts `run_uv` receives that same object.

### 5. Retire an emptied panel within one visual frame (item 9: product fix plus test update)

Root cause: `2bf6f3d87` ("retire emptied Agents-tab tribe panels whatever way their
nodes leave") made an authoritative complete-history apply retire an emptied sticky
panel, in `_reconcile_session_mounted_for_apply` (`_display_panel_collection.py`, called
from `_loading_apply.py`). The arrival scenario's `default_removed` window
(`tests/ace/tui/_epic_arrival_frames.py`) now retires `@default` instead of collapsing
it to a title strip.

The retirement is not atomic. `_sync_mounted_panel_widgets`
(`src/sase/ace/tui/actions/agents/_display_panel_widgets.py`) calls
`_unmount_agent_list`, which calls Textual's `widget.remove()`. That only schedules a
prune, so the widget stays a child of `#agent-list-container` until the prune lands.
Meanwhile `_reorder_agent_list_widgets` moves the kept panels ahead of it and
`_settle_agent_list_container_width` shrinks the column. Recorded frames:

| Frame        | State                                                                                                                         |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| 28 (before)  | container width 127; `@default` has 3 rows                                                                                    |
| 29 (refresh) | container width 74; the doomed `@default` is still visible with 3 rows at requested width 127, now at the bottom of the stack |
| 30           | `@default` is finally gone                                                                                                    |

`visual_transition_violations` reports `visible state changed in 2 frames: [29, 30]`.
This is the rows-then-geometry flicker the sase-142 invariants forbid, so it is a real
regression and not only a stale assertion.

Product fix. Make retirement land in the same refresh frame:

- In `_unmount_agent_list`, before calling `remove()`, hide the widget synchronously
  (`widget.display = False`) and set a module-owned retiring marker on it. Guard both
  for test fakes, the way the existing `remove`/`children` fallback does. Do not rely on
  Textual's private `_pruning`.
- Make `agent_list_widgets_in` (`src/sase/ace/tui/actions/agents/_display_helpers.py`)
  skip retiring widgets by default. Reorder, width settle, `first_agent_list_widget`,
  focus, and the paint log (`_paint_log.py`) then all see the post-retirement panel set.
  Check each of the 7 call sites.
- Hazard: `_sync_mounted_panel_widgets` builds its `existing` id map from
  `agent_list_widgets_in`. If a key is re-added before the prune lands, excluding the
  retiring widget there would mount a second `AgentList` with the same id and raise
  Textual `DuplicateIds`. Today's code instead silently reuses the doomed widget, which
  then vanishes. Handle this explicitly. For example, add an opt-in
  `include_retiring=True` for the id map, and when a kept key maps to a retiring widget,
  mount its replacement once the removal completes (for instance by scheduling a
  follow-up panel sync from the `AwaitRemove`), so the panel neither duplicates nor
  disappears. Add a focused unit test for this re-add-while-retiring case next to the
  existing sticky-panel tests (`tests/ace/tui/test_agent_panels_display_sticky.py` or
  the display-widgets tests).
- Follow the TUI perf rules: keep this synchronous and cheap on the apply path, and add
  no new refresh paths.

Test update for the new contract in `tests/ace/tui/test_epic_panel_arrival_frames.py`:

- Rename `test_a_removal_that_collapses_a_panel_settles_the_column_in_frame` to
  `test_a_removal_that_retires_a_panel_settles_the_column_in_frame`. Assert:
  - `@default` is present with rows in the `before` frame.
  - It is absent from both the `refresh` frame and the `after` frame.
  - The window records no `update_list` or `render_collapsed` paint call.
  - `refresh.container_width < before.container_width` and
    `after.container_width == refresh.container_width`.
  - No `container_width` frame follows.
  - `visual_transition_violations(run.window_frames(label, with_previous=True)) == []`.
- Rename `test_a_removal_that_collapses_a_panel_leaves_the_other_panels_alone`
  accordingly.
- Update the "collapse" wording in the module docstring. Also update the
  `default_removed` / `starting*` scenario comments in `_epic_arrival_frames.py` to
  describe retirement. Keep the invariants strict; do not add `xfail` or loosen them.
- Run the whole arrival-frames module and the sticky/cleanup panel suites:
  - `tests/ace/tui/test_agent_panels_display_sticky.py`
  - `tests/ace/tui/test_agent_cleanup_panel_clan_sticky_e2e.py`
  - `tests/ace/tui/test_agent_cleanup_panel_clan_members_e2e.py`
  - any `tests/ace/tui` module that exercises `_sync_mounted_panel_widgets` or
    `agent_list_widgets_in`

No PNG golden change is expected because settled states do not change. If an Agents-tab
visual snapshot does change, run a targeted `just fix-tui-screenshots -- <selector>` and
inspect the report before accepting it.

## Verification

1. Re-run the 10 nodes above plus the full modules they live in, and
   `tests/llm_provider/test_claude_fable_usage_indicator_default.py`. All must pass.
2. Run `just fmt` (or `just fix`), then `sase tool run check`. Hand it to
   `/sase_monitor` if it runs long. The run must be green, or any remaining failure must
   be shown to reproduce on a clean master tree and be unrelated. Record any such
   failure as a follow-up instead of fixing it here.
3. Do not run `just check-full`.

## Bead bookkeeping (after verification passes)

- Close each piece's tracking task with the command below. Check each bead's status
  first; skip a bead that is already closed.
  - `sase bead close sase-15z --note "<nodes fixed + check run id>"`
  - `sase bead close sase-14u --note "..."`
  - `sase bead close sase-14v --note "..."`
- Do not close epics sase-14j or sase-158.6, which belong to their land agents. Append a
  `sase bead note` to each saying the DISCOVERED ISSUE test is now fixed on master and
  naming the fix.
