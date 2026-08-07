---
tier: tale
title: Route ChangeSpec navigation through the Artifacts PRs sub-tab
goal:
  Pressing `<enter>` on the Agents tab lands the user on the Artifacts tab's PRs sub-tab
  with the target ChangeSpec selected, and every other cross-tab ChangeSpec jump does
  the same.
proposed_by: bbugyi200.athena.uz
create_time: 2026-08-07 14:55:29
status: wip
---

- **PROMPT:**
  [prompts/202608/agents_enter_jumps_to_prs_subtab.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agents_enter_jumps_to_prs_subtab.md)

# Plan: Route ChangeSpec navigation through the Artifacts PRs sub-tab

## Problem

On the Agents tab, `<enter>` is bound to `jump_to_agent_changespec` ("Go to PR"). Since
the PR surface was migrated from a top-level "PRs" tab into a sub-tab of the "Artifacts"
tab, that keymap no longer shows the user anything. It switches to the Artifacts tab but
leaves whatever Artifacts sub-tab was last visible on screen (`commits` by default), so:

- the target ChangeSpec is selected in a list the user cannot see, and
- when the ChangeSpec is not in the current filtered list, the app silently rewrites the
  query to `project:<name>`, pushes the old query onto the back stack, and persists it
  as the last-used query — all behind the Commits pane, with no visible feedback.

## Root cause

`navigate_to_changespec_tab()` in
`src/sase/ace/tui/actions/agents/_notification_navigation.py:172-248` predates the
Artifacts sub-tab migration. At line 195 it does only:

```python
app.current_tab = "changespecs"  # type: ignore[attr-defined]
```

Setting the top-level tab used to be sufficient, because `changespecs` _was_ the PR
surface. It no longer is. `ARTIFACTS_TAB` is still the internal id `"changespecs"`
(`src/sase/ace/tui/tab_order.py:18`), but which leaf pane that tab renders is now owned
by a second reactive, `current_artifacts_subtab`, which defaults to
`DEFAULT_ARTIFACTS_SUBTAB = "commits"` (`src/sase/ace/tui/artifact_tabs.py:23`) and is
not persisted across sessions. Because `navigate_to_changespec_tab()` never touches it,
three things go wrong downstream:

1. `watch_current_tab` (`src/sase/ace/tui/_app_watchers.py:95-100`) calls
   `changespecs_view.activate_current()`, which activates the _currently selected_ leaf
   pane — Commits, not PRs — and then `_sync_active_artifacts_entry_state()` paints the
   non-PR footer for that pane.
2. `watch_current_idx` (`src/sase/ace/tui/_app_watchers.py:18-31`) only schedules
   `_refresh_changespecs_display_debounced()` when `current_artifacts_subtab == "prs"`,
   so the `app.current_idx = idx` assignments at lines 201 and 236 move the PR selection
   without repainting the PR list.
3. The not-found branch (lines 204-237) mutates `parsed_query` / `query_string`, calls
   `_load_changespecs()`, pushes query history, and calls `_save_current_query()` — all
   invisible, because the PRs pane is hidden. This is the "wrong ChangeSpec query" half
   of the reported symptom.

Callers affected by this one function: the Agents `<enter>` keymap
(`actions/agents/_changespec_navigation.py:110`), notification `jump_to_changespec` and
`jump_to_mentor_review` actions (`actions/agents/_notification_handlers.py:143,171`),
and the runners modal's ChangeSpec jump target (`actions/axe.py:449`).

### Two more sites with the identical migration miss

Both assign `current_tab` without selecting the PRs sub-tab, so they have the same
invisible-navigation bug:

- `src/sase/ace/tui/actions/navigation/_modals.py:28` — the cross-tab "Jump to Entry"
  modal (`action_jump_to_all_entries`). Selecting a ChangeSpec entry sets
  `current_tab = result.tab` and `current_idx`, but never the sub-tab.
- `src/sase/ace/tui/actions/changespec/_query.py:41-43` — `_load_saved_query()`, reached
  from the `load_saved_query_<N>` actions. Those actions stay available on the Agents
  tab, so loading a saved query slot from Agents lands on the Commits pane with the new
  query applied but unseen.

## Approach

Introduce one authority for "show this Artifacts sub-tab and the tab that hosts it", and
route all three sites through it.

The correct ordering already exists in
`ArtifactsNavigationActionsMixin._switch_artifacts_subtab()`
(`src/sase/ace/tui/actions/artifacts_navigation.py:345-352`): assign
`current_artifacts_subtab` _first_, while Artifacts may still be hidden, then assign
`current_tab`. That ordering matters — it lets `watch_current_tab` activate only the
requested pane instead of briefly activating the previously selected one. It is also
correct when Artifacts is already the active tab: `watch_current_artifacts_subtab`
performs the deactivate/activate swap, and the subsequent `current_tab` assignment is a
no-op because `watch_current_tab` early-returns when `old_tab == new_tab`.

## Steps

1. **Extract the ordering rule into a reusable helper.** Add a module-level function
   that takes a duck-typed app and a sub-tab, applies the `subtab`-then-`tab` ordering
   described above, and carries a docstring explaining why the order is load-bearing.
   Reduce `ArtifactsNavigationActionsMixin._switch_artifacts_subtab()` to a delegation
   to it so there is exactly one implementation.

   Placement constraint: `_notification_navigation.py` must be able to call it. Prefer a
   home that does not drag the Textual widget tree into that module's import graph —
   either `src/sase/ace/tui/artifact_tabs.py` (already documented as the
   widget-and-keymap-free shared module, and `tab_order.ARTIFACTS_TAB` is equally
   dependency-free) or a function-local import from
   `src/sase/ace/tui/actions/artifacts_navigation.py`. Pick one and note the reason in
   the docstring.

2. **Fix `navigate_to_changespec_tab()`**
   (`src/sase/ace/tui/actions/agents/_notification_navigation.py:195`). Replace the bare
   `app.current_tab = "changespecs"` with a call to the helper requesting the `"prs"`
   sub-tab. Use the `ARTIFACTS_TAB` constant rather than the `"changespecs"` string
   literal (import as `from ...tab_order import ARTIFACTS_TAB`) so the next rename of
   the internal id cannot re-open this bug.

   Keep the existing control flow otherwise: switch up front, exactly where the current
   code switches, so the failure branches still leave the user looking at the PRs pane
   and the query that was applied. Do not reorder resolution ahead of the switch — the
   normal tab-cycle path (`actions/navigation/_basic.py:475-478`) also assigns
   `current_tab` before `current_idx`, so the one-frame stale-row render is pre-existing
   and is corrected by the debounced refresh that `watch_current_idx` now schedules.

3. **Fix `action_jump_to_all_entries`'s dismiss handler**
   (`src/sase/ace/tui/actions/navigation/_modals.py:23-29`). When `result.tab` is the
   Artifacts tab, select the `"prs"` sub-tab through the same helper before assigning
   `current_idx`. The jump-all modal only lists ChangeSpec entries for that tab, so PRs
   is unconditionally the right pane there — confirm that against
   `src/sase/ace/tui/modals/jump_all_modal.py` while implementing.

4. **Fix `_load_saved_query()`**
   (`src/sase/ace/tui/actions/changespec/_query.py:41-43`). Saved queries are a PRs-pane
   concept, so the tab switch there should request the `"prs"` sub-tab too. Note the
   asymmetry worth preserving: `action_open_saved_query_picker` already refuses to open
   off the PRs pane (`src/sase/ace/tui/_app_action_availability.py:136`), while the
   numbered `load_saved_query_<N>` actions are reachable from Agents — that is the path
   this step repairs.

5. **Sweep for any remaining `current_tab = "changespecs"` assignment** that is not one
   of the three sites above, and either fix it or record why it is correct. As of this
   plan the only other writers are `_switch_to_tab`, the tab-bar click handler
   (`actions/_event_widgets.py:231`), and `_state_init.py`'s startup assignment, all of
   which intentionally restore whichever sub-tab was last showing.

## Tests

6. **Unit — `tests/ace/tui/test_jump_to_changespec.py`.** Give `FakeNavigationApp` a
   `current_artifacts_subtab` attribute seeded to a non-PR value (`"commits"`) so the
   fake matches the real default. Assert in the existing
   `TestNavigateToChangespecExactFirst` cases — covering both the found-in-current-list
   path and the `test_exact_target_wins_after_switching_to_project_query` reload path —
   that after `navigate_to_changespec_tab()` the app reports
   `current_tab == "changespecs"` **and** `current_artifacts_subtab == "prs"`. These
   assertions fail against `master`.

7. **End-to-end — the Agents `<enter>` round trip.** Add Pilot coverage (new module
   `tests/ace/tui/test_agents_jump_to_prs_subtab.py`, or an existing agents e2e module
   if one fits) following the established pattern in
   `tests/ace/tui/test_agent_metadata_search.py`: seed agents with
   `patch_startup_loaders(monkeypatch, agents=[...])`, open
   `AcePage(initial_tab="agents")`, `wait_for_startup`, press `enter`, then assert the
   captured state has `tab == "changespecs"`, `artifacts_subtab == "prs"`, and
   `selected.name` equal to the agent's `cl_name`. `AcePage._extract_state` already
   exposes `artifacts_subtab` (`src/sase/ace/testing/ace_page.py:37`), so no harness
   change is needed. Also assert the mounted `ArtifactsView.current_subtab` is `"prs"`
   so the test pins the rendered pane, not just the reactive.

   Cover both branches: one agent whose `cl_name` matches a seeded ChangeSpec (default
   fixtures are `feature_a` / `feature_b` / `feature_c` in
   `src/sase/ace/testing/fixtures.py`), and one whose ChangeSpec is outside the active
   query so the `project:<name>` swap runs — asserting there that the rewritten
   `query_string` is visible on the PRs pane rather than applied behind another pane.

8. **Regression coverage for the two secondary sites.** Add at least one test each for
   the jump-all modal ChangeSpec jump and for a `load_saved_query_<N>` action invoked
   from the Agents tab, asserting the PRs sub-tab ends up active.

9. **Confirm no keymap, config, or docs drift.** The binding
   (`src/sase/ace/tui/bindings.py:255`), its `default_config.yml` entry
   (`jump_to_agent_changespec: "enter"`, line 392), and its "Go to PR" label
   (`src/sase/ace/tui/keymaps/metadata.py:159`) all stay accurate — this is a behavior
   fix, not a rebinding. Per the repo's help-popup rule, re-read the `?` help modal and
   `keybinding_footer` copy for the Agents tab and update only if something there
   describes the old top-level PRs tab.

## Verification

- `just install` first (workspace virtualenvs are ephemeral), then `just check`.
- Run the touched suites directly while iterating:
  `pytest tests/ace/tui/test_jump_to_changespec.py tests/ace/tui/test_agents_jump_to_prs_subtab.py`.
- Confirm each new assertion fails before the fix and passes after — the point of this
  change is behavioral, so a test that passes on `master` is not covering the bug.
- No PNG snapshot goldens should move; if `just test-visual` reports drift, that is a
  signal the fix changed rendering somewhere unintended, not a golden to accept.

## Non-goals

- Renaming the internal `"changespecs"` tab id to `"artifacts"`. The comment at
  `src/sase/ace/tui/tab_order.py:14-17` documents that the stable id is deliberate.
- Persisting `current_artifacts_subtab` across sessions.
- Reworking `navigate_to_changespec_tab()`'s query-rewrite policy (whether it _should_
  swap to `project:<name>` at all). This plan only makes that swap visible.
- Fixing the stale-agent-index round trip: `navigate_to_changespec_tab()` never calls
  `_save_current_tab_position()`, so tabbing back to Agents restores `_agents_last_idx`
  rather than the agent the user jumped from. That is a real but separate defect that
  predates the Artifacts migration. If it is still present after this change, file it
  via `/sase_new_task` rather than folding it in here.
