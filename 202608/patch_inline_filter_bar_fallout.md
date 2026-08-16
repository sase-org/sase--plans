---
tier: tale
size: medium
title: Repair the Patch inline-filter-bar fallout left by the Artifacts query epic
goal:
  The 23 test failures and 45 errors that epic sase-m6.6.1 left on master are gone.
  Refreshing the Patch display against a not-yet-composed inline filter bar no longer
  raises, the Patch-pane suites assert the inline bar instead of the removed
  QueryEditModal, and the Agents footer and help expectations follow the keymap after
  edit_hooks moved from f to F.
proposed_by: bbugyi200.athena.sase-m6.6.1.land
bead: sase-m6.6.1
create_time: 2026-08-16 02:11:32
status: wip
---

- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.md)

# Plan: Repair the Patch inline-filter-bar fallout

## Why

Epic `sase-m6.6.1` replaced Patch's `QueryEditModal` with a persistent inline
`PatchFilterBar` and rebound `edit_hooks` from `f` to `F` so `f` could open the new bar.
Phase `sase-m6.6.1.7` recorded the resulting suite breakage as a `PROPOSED FOLLOW-UP`
("Stabilize full Python suite after Artifacts inline-filter migration") instead of
fixing it, so master still carries it.

Reproduced by the epic's land agent on master `d22622365`:

- Full `just test`: **83 failed, 30733 passed, 11 skipped, 45 errors**.
- 55 of those 83 failures are unrelated host-environment contamination already tracked
  by task bead `sase-ml` (the durable-proc `SASE_PROC_*` env vars leak into pytest).
  Re-running the same files with those six variables unset turns all 55 green. They are
  **out of scope here**.
- The remaining **23 failures + 45 errors** are all caused by this epic and are the
  entire scope of this plan. Re-verified in isolation on `d22622365` with the
  `SASE_PROC_*` variables unset: `23 failed, 181 passed, 45 errors`.

Reproduce the in-scope set with:

```bash
env -u SASE_PROC_REQUEST_PATH -u SASE_PROC_RESULT_PATH -u SASE_PROC_ID \
    -u SASE_PROC_OPERATION -u SASE_PROC_LOG_PATH -u SASE_PROC_SESSION_ID \
  .venv/bin/python -m pytest \
    tests/ace/tui/widgets/test_vim_normal_key_containment.py \
    tests/ace/tui/test_changespecs_onboarding.py \
    tests/test_ace_testing.py \
    tests/test_keybinding_footer_agent.py \
    tests/test_ace_tui_app.py \
    tests/ace/tui/widgets/test_keybinding_footer_idempotent.py \
    tests/ace/tui/test_changespec_detail_only_refresh.py \
    tests/test_keymaps_display_help.py \
    tests/ace/tui/test_jump_to_changespec.py -q
```

## Scope

Three groups, in dependency order. Group 1 is a real product defect; groups 2 and 3 are
stale expectations left behind by intentional behavior changes.

### 1. Patch filter bar crashes a display refresh before its children mount

`FilterBar.set_query()` (`src/sase/ace/tui/widgets/filter_bar.py:216`) calls `_editor()`
(`:315`), which does `query_one("#patch-filter-input", _FilterBarInput)` unguarded. When
the `PatchFilterBar` node is in the DOM but its composed children are not mounted yet,
that raises
`textual.css.query.NoMatches: No nodes match '#patch-filter-input' on PatchFilterBar(id='patch-filter-bar')`.

The reachable path is an ordinary tab refresh:

```
ace_page_group.checkout            src/sase/ace/testing/ace_page_group.py:144
  _reset_shared_page               src/sase/ace/testing/ace_page_group.py:173
    _refresh_current_tab           src/sase/ace/tui/actions/_event_base.py:69
      _refresh_display             src/sase/ace/tui/actions/patch/_display.py:384
        _refresh_display_impl      src/sase/ace/tui/actions/patch/_display.py:394
          FilterBar.set_query      src/sase/ace/tui/widgets/filter_bar.py:220
            FilterBar._editor      src/sase/ace/tui/widgets/filter_bar.py:315
```

`_get_search_query_widget()` (`src/sase/ace/tui/actions/patch/_display.py:369`) already
returns `None` when the bar itself is absent, and `_closed_display()`
(`filter_bar.py:317`) already guards on `is_mounted`; the editor lookup is the one
unguarded path.

Fix the widget so a refresh against a not-yet-composed bar is a no-op rather than an
exception, and keep the guard where every caller benefits (`set_query`, `set_status`,
and any other public entry point that resolves the editor) rather than only at the Patch
call site. Do not paper over it by swallowing exceptions in `_refresh_display_impl`.

This single defect produces all **45 errors** in
`tests/ace/tui/widgets/test_vim_normal_key_containment.py` — the tests themselves pass;
the shared-page checkout that follows them raises — and 3 of the 4 failures in
`tests/test_ace_testing.py`
(`test_ace_page_group_reuses_page_and_resets_prompt_without_history`,
`test_ace_page_group_rejects_overlapping_checkouts`,
`test_ace_page_group_reports_reset_hook_leaks`).

Add regression coverage that refreshes the Patch display while the bar is unmounted or
uncomposed, so the guard cannot silently regress.

### 2. Patch-pane tests still assume `QueryEditModal` and `#search-query-panel`

`QueryEditModal` is intentionally retained — the Agents tab still uses it via
`src/sase/ace/tui/actions/agents/_filter_actions.py`, and converting Agents off
`ace/agent_query` is explicitly deferred by the epic plan. Only the **Patch** pane moved
off it. `action_edit_query` (`src/sase/ace/tui/actions/base.py:440`) now delegates
`slash` on the Patches pane to `pane.show_filters()`, so no modal ever appears.

Update these to assert the inline bar (`#patch-filter-bar` / `#patch-filter-input`) and
the new open/commit/rollback flow instead:

- `tests/test_ace_tui_app.py::test_query_edit_modal_cancel`,
  `::test_query_edit_modal_apply`, `::test_query_edit_modal_invalid_query` (3) —
  currently `expect_modal('QueryEditModal') timed out after 5.0s — last value was None`.
  Escape rollback, submit-applies, and invalid-query error surfacing all still exist on
  the inline bar (`PatchesFilterSessionMixin` in
  `src/sase/ace/tui/widgets/artifacts/patches_filter_session.py`), so keep the coverage
  and retarget it; do not delete these cases.
- `tests/test_ace_testing.py::test_expect_modal` (1) — uses the Patch query modal purely
  as a sample modal for the `expect_modal` harness assertion. Point it at a modal the
  harness can still open.
- `tests/ace/tui/test_changespecs_onboarding.py` (7) — `_search_query_plain()` and
  `_assert_patches_onboarding_layout()` query `#search-query-panel`, which the Patches
  pane no longer mounts (`NoMatches ... on Screen(id='_default')`). Retarget to the
  filter bar and keep asserting the onboarding chrome contract.
- `tests/ace/tui/test_changespec_detail_only_refresh.py::test_full_refresh_still_calls_update_list`
  and `::test_mark_toggle_falls_back_to_full_refresh_on_patch_failure` (2) — the
  `_FakeApp` double records `update_list_calls == 0` where 1 is expected. Determine
  whether `_refresh_display_impl` genuinely stopped rebuilding the list for this shape
  or the double is simply missing state the new implementation reads; fix the product
  code if the former, the double if the latter.
- `tests/ace/tui/test_jump_to_changespec.py::TestNavigateToPatchExactFirst::test_exact_target_wins_after_switching_to_project_query`
  (1) — `navigate_to_patch_tab` returns `False`.
  `src/sase/ace/tui/actions/agents/_notification_navigation.py:249-286` now needs
  `app._parse_patch_query` and a pane-keyed `app._query_history` dict, and its blanket
  `except Exception` swallows whatever the `FakeNavigationApp` double is missing.
  Confirm the production path still works, then bring the double up to the current
  contract. Consider narrowing that `except Exception` so a missing collaborator is not
  reported to users as a generic "Navigation error".

### 3. Agents footer and help still expect `f` for fork

Commit `3c3909c31` changed `src/sase/default_config.yml` so `patches_filters: "f"` and
`edit_hooks: "F"`. The Agents-tab clan/tribe fork bindings render from the `edit_hooks`
keymap (`src/sase/ace/tui/widgets/_keybinding_bindings.py:302`), so fork moved to `F` as
an intended consequence. The rendering is already consistent; only the expectations are
stale:

- `tests/test_keybinding_footer_agent.py::test_keybinding_footer_clan_advertises_clan_fork`,
  `::test_keybinding_footer_named_panel_advertises_tribe_fork`,
  `::test_keybinding_footer_agent_bindings_tale_done_with_chat` (3) — e.g.
  `assert ('f', 'fork clan') in [..., ('F', 'fork clan'), ...]`.
- `tests/ace/tui/widgets/test_keybinding_footer_idempotent.py::test_clan_footer_keeps_row_cleanup_and_panel_chooser_labels`,
  `::test_named_tribe_footer_advertises_fork_and_wait` (2).
- `tests/test_keymaps_display_help.py::test_agents_help_uses_f_for_fork_not_r_for_resume`
  (1) — the assertion and the test name both encode `f`; rename it so the name matches
  what it now guards.

Prefer deriving the expected key from the keymap registry over hard-coding `F`, so a
future rebinding does not break these again.

While here, confirm the `?` help modal and the footer describe `f` as the Patch filter
bar on the Patches pane (`src/sase/ace/CLAUDE.md` requires help-popup updates whenever a
`sase ace` option changes) and fix them if phase `sase-m6.6.1.6` missed a surface.

## Out of scope

- The `SASE_PROC_*` host-environment contamination (55 failures) — tracked by `sase-ml`,
  noted on epic `sase-m9.3.1`.
- The repo-wide PNG golden drift from the model-badge fixture change (421 of 690 visual
  snapshots) — noted on epic `sase-mf`; caused by commit `981106799`, not by this work.
  Do **not** accept visual goldens as part of this plan.
- Any further Artifacts query-engine behavior change. The engine, profiles, persistence,
  and conformance goldens from `sase-m6.6.1` are verified and must not be reworked here.

## Verification

1. `just install` first — this workspace may be stale.
2. Reproduce the in-scope set with the command above; it must reach
   `0 failed, 0 errors`.
3. `just check` (whole-repo lint gates + the diff-scoped test lane). `just symvision`
   must stay clean, and no new `--epic-symbol` entries may be added to the `Justfile`.
4. `just check-full` through `/sase_monitor` with a `--next` action before landing,
   because this diff touches broad ACE TUI surfaces. Expect the 55 `SASE_PROC_*`
   failures if the monitor runs inside an agent proc; confirm against the `sase-ml`
   signature and do not attribute them to this diff.
5. Do not run `just test-visual` as a gate — it is red repo-wide for an unrelated reason
   (see Out of scope). If a Patch snapshot must be regenerated, inspect the diff
   artifact in `.pytest_cache/sase-visual/` and only accept a golden whose diff is
   outside the top-right model-badge region.
