---
tier: tale
title:
  Fix the red Master Gate CI (core pin, stale tests, Textual drift, import shadowing)
goal:
  Master Gate lint and all eight fast-suite shards pass on master, with no production
  behavior changes beyond the core pin bump and one completion hint.
size: medium
proposed_by: bbugyi200.athena.0u5
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0u5](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0u5.md)
- **COMMITS:**
  - [8c38eb6](https://github.com/sase-org/sase/commit/8c38eb6a9a1c4089ac1d00ade84f15012a5baf3e)
    — fix(master-gate): repair red Master Gate CI per 202609/fix_master_gate_ci

# Fix the red Master Gate (lint + all 8 fast-suite shards)

## Problem

`Master Gate` (`.github/workflows/master-gate.yml`) has failed on **every** master push
for 400+ consecutive runs (back to at least 2026-09-25). The latest run (`79e7107`,
run 36622706287) fails `lint` at "Check pinned core bindings" plus all eight `test (N)`
shards, with ~60 distinct failing test ids. Several unrelated regressions have piled up
on top of each other because nobody could see a green baseline. Agents do not catch them
locally because `just check` is diff-scoped and most workspace venvs carry a stale
`sase_core_rs` wheel and Textual 8.0.1.

The diagnosis below was reproduced locally against a venv rebuilt from sase-core
`1ad57ea` (reproducing ~50 of the failures). The chrome-layout failures were reproduced
in a throwaway Python 3.12 venv built exactly like CI (fresh `.[dev]` resolve).

There are **seven independent root causes**. Every one is a stale test expectation, a
test-infrastructure bug, or a pin that was not moved. No production behavior needs to
change.

## Root causes and fixes

### 1. Core pin is one commit behind the Python code (lint + 3 tests)

- `lint` → `tools/check_sase_core_rs_bindings`: "sase_core_rs 0.36.0 is missing 1 of 753
  required binding(s): `evaluate_prompt_prediction_replay`".
- `tests/core/test_prompt_prediction_facade.py` (2 tests) and
  `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`
  fail for the same reason.
- Cause: sase commit `9d60b97` (calibrate replay harness) calls the new binding, which
  landed in sase-core commit `1ad57ea426fe400da427042294d815259744453a` ("add per-point
  novel coverage and precision to sweep wire"). `sase-core-revision.txt` still pins
  `43f744be33e516aad80059b33e64056c7d156741`, its direct parent. `1ad57ea` is already on
  sase-core `origin/master`.
- **Fix:** write exactly `1ad57ea426fe400da427042294d815259744453a` into
  `sase-core-revision.txt`, followed by a newline. Pin this verified SHA; do **not** use
  `just ratchet-core-revision` to jump to whatever sase-core HEAD is at implementation
  time, because an unverified newer core would expand this change's blast radius. Before
  writing, confirm the SHA is on the sase-core remote. Use `sase repo open sase-core`,
  then `git fetch` followed by `git merge-base --is-ancestor 43f744b 1ad57ea` and
  `git branch -r --contains 1ad57ea`.
- Locally, `just install` must build from a sase-core checkout that contains `1ad57ea`
  so the local wheel exposes the binding. Verify with
  `.venv/bin/python tools/check_sase_core_rs_bindings`, which should report "exposes all
  … bindings".

### 2. `origin=` kwarg threaded through launch/prompt-record seams; ~17 test modules not updated (~40 failures)

- Commit `eaa4aa4aa7` ("feat(prompt-history): record typed vs generated origin on prompt
  rows") added an `origin` keyword at every launch and prompt-record write site.
  Examples include `launch_agent_from_cwd(..., origin="generated")` and
  `launch_planned_bead_work_agents(..., origin="generated")` in
  `src/sase/bead/cli_work_launch.py`, plus
  `record_failed_launch_prompt(query, origin="typed")`,
  `add_or_update_prompt(..., origin="typed")`, and similar. The commit updated some
  tests with the idiom `origin: Any = None` on fake signatures, but missed many others.
- Symptoms:
  - `TypeError: <fake>() got an unexpected keyword argument 'origin'`. Inside
    `sase bead work` this surfaces as `BeadWorkError: agent launch failed …` →
    `SystemExit: 1`. The epic-lifecycle rollback test then also fails with "partially-
    launched child was not killed".
  - `assert_called_with(...)` mismatches where the actual call now carries
    `origin='typed'`.
- Failing modules to fix. A fake may live in a shared helper module next to these files;
  fix it at its definition site.
  - `tests/test_partial_launch_cleanup.py`
  - `tests/test_launch_admission_dispatch_hold_lifecycle.py`
  - `tests/test_force_reuse_launch_seam_consume.py`,
    `tests/test_force_reuse_launch_seam_registry.py`,
    `tests/test_force_reuse_launch_seam_rejection.py` (expected
    `record_failed_launch_prompt(...)` calls need `origin="typed"`)
  - `tests/test_bead/test_epic_from_plan.py`
  - `tests/test_bead/test_cli_work_from_plan_publication_sidecars.py`
    (`inspect_fresh_worker_clone`)
  - `tests/test_bead/test_cli_work_cleanup_confirm.py`,
    `tests/test_bead/test_cli_work_collisions.py`,
    `tests/test_bead/test_cli_work_epic_checkpoint.py`,
    `tests/test_bead/test_cli_work_epic_dry_run.py`,
    `tests/test_bead/test_cli_work_epic_launch_cleanup.py`,
    `tests/test_bead/test_cli_work_epic_launch_cleanup_sessions.py`,
    `tests/test_bead/test_cli_work_epic_lifecycle.py`,
    `tests/test_bead/test_cli_work_epic_summary.py` (`stale_epic_summary_launch` fake)
  - `tests/ace/tui/test_prompt_bar_submit_no_cancel_save.py` (expected
    `add_or_update_prompt(..., cancelled=True)` needs `origin="typed"`)
  - `tests/ace/tui/test_entry_points_vcs_prefix_mru.py` (`_record_cancelled_prompt`
    fake)
- `tests/test_proc_env_isolation.py::test_sase_ml_file_families_ignore_inherited_live_proc_env`
  is **derivative**: it runs `_SASE_ML_FILE_FAMILIES` (which includes a
  `test_partial_launch_cleanup` test) in a subprocess and asserts all pass. It goes
  green once that module is fixed.
- **Fix:** match the idiom `eaa4aa4aa7` already used. Add `origin: Any = None` (or a
  keyword-only `origin=None` on lambdas) to each fake. Add the production value
  (`origin="typed"` for interactive/query launches, `origin="generated"` for bead-work
  and machine launches) to each `assert_called_with` / `assert_called_once_with`
  expectation. Where a bead-work fake already records its kwargs, one assertion that
  `origin == "generated"` is welcome but optional. Do not weaken assertions with `ANY`
  when the concrete value is known. Do not change production code.
- Also sweep for not-yet-failing fakes of the same seams that would break in non-fast
  lanes. Grep `tests/` for fakes/monkeypatches of `launch_agent_from_cwd`,
  `launch_agents_from_cwd`, `launch_planned_bead_work_agents`,
  `record_failed_launch_prompt`, `record_interactive_failed_launch`, and
  `add_or_update_prompt` that neither accept `origin` nor `**kwargs`. Fix any that are
  reachable from the changed call sites.

### 3. `tests/tool/test_settlement*.py` collection error (3 modules + contract manifest)

- CI: `ModuleNotFoundError: No module named 'tool._settlement_helpers'` for
  `tests/tool/test_settlement.py`, `test_settlement_hook.py`, and
  `test_settlement_retention.py`. It reproduces deterministically in a full-suite
  collect (`pytest --collect-only`) but **not** when those files are collected alone.
- Cause: the three files (created by `be9cd4d096`, "split test_settlement into four
  files") import `from tool._settlement_helpers import …` via the bare top-level name
  `tool`. `tests/tool/` has no `__init__.py`, so `tool` is only a namespace portion on
  `pythonpath = ["src", "tests"]`. Several legacy `src/sase` modules run
  `sys.path.append(<src/sase>)` at import time, for example `src/sase/ace/status.py:6`
  and `src/sase/ace/operations.py:9`. Once any earlier-collected test imports them,
  `src/sase` is on `sys.path`, and Python resolves the regular package
  `src/sase/tool/__init__.py` as top-level `tool`, since a regular package beats a
  namespace portion regardless of path order. Probe evidence: `sys.modules['tool']` was
  `src/sase/tool/__init__.py` at the failure point.
- `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
  (`pytest -m contract --collect-only failed (exit 2)`) is **derivative** of this
  collection error. `-m contract --collect-only` succeeds once these three modules are
  ignored.
- **Fix:** in all three files, change `from tool._settlement_helpers import (` to
  `from tests.tool._settlement_helpers import (`. This is the established repo
  convention for namespace test dirs, e.g. `from tests.monitor._fixtures import`,
  `from tests.history._prompt_placeholders_helpers import`, and
  `from tests.gate_turn._cli_fixtures import`. `tests` is a regular package, so this
  path cannot be shadowed. No other test module uses a bare top-level import of a test
  dir that collides with a `src/sase` package; this was checked for all 29 colliding dir
  names. Then confirm the contract manifest is still current.
  `just refresh-contract-manifest` must produce no diff (the settlement files carry no
  `contract` marker today), or commit the refreshed `tests/contract_manifest.txt` if it
  does.
- Do **not** remove the `sys.path.append` hacks in `src/sase/ace/**` in this change;
  that is product-import surgery with its own blast radius (see Follow-ups).

### 4. Textual drift makes two chrome-layout pilot tests fail only in CI

- `tests/ace/tui/command_line/test_chrome_layout.py::test_frame_width_cap_and_full_height_toggle_keep_the_labels`
  (`assert 200 == 96`) and
  `::test_chrome_recomposes_and_stays_aligned_on_resize[wide0-narrow0]`
  (`assert 134 > 134`) failed in 9/9 sampled CI runs but pass in local agent venvs.
- Cause: the fast shards install `.[dev]`, where `textual[syntax]>=0.45.0` is unpinned,
  so CI resolves the latest Textual (currently **8.2.8**). Only the `visual` extra pins
  `textual==8.0.1`, and existing workspace venvs sit on 8.0.1. Under 8.2.x, the autouse
  `_event_driven_bare_pilot_pause` fixture in `tests/ace/tui/conftest.py` (which swaps
  `Pilot.pause` for the settle barrier) no longer covers the frame's post-resize layout
  pass. The test's two waits are therefore vacuous:
  - `_resize_terminal` waits only for `app.size`/`screen.size` to match.
  - `_wait_for_border_recompose` checks that the labels span
    `frame.outer_size.width - 6` using the frame's **stale** width, which is trivially
    true before the frame is re-laid out.

  Verified: with Textual 8.2.8 plus the autouse patch the helpers return with width 200.
  With 8.0.1 they return 96.

- **Fix (test-only):** make the recompose wait require the frame's own layout to reach
  the expected width for the new terminal size before checking the labels. The frame is
  CSS 96% wide, capped at 200. `min(200, terminal_width * 96 // 100)` matches the
  measured widths exactly on both Textual 8.0.1 and 8.2.8 (240→200, 140→134, 100→96,
  80→76). Concretely:
  - Add a small helper, e.g. `_expected_frame_width(terminal_width: int) -> int`, with a
    one-line comment that it mirrors the frame's `96%` / `max 200` CSS.
  - Give `_wait_for_border_recompose` an `expected_width: int` parameter. Its predicate
    must require `frame.outer_size.width == expected_width` **and** both labels'
    `cell_len == expected_width - 6`.
  - Update both call sites to pass `_expected_frame_width(<the width just resized to>)`.
    In the parametrized resize test that is `size[0]` inside the loop.

  A prototype of exactly this predicate converged under both Textual versions with the
  autouse pause patch active (sequence 140→80→140→100 gave 134/76/134/96). Keep the
  existing `wait_for` timeout semantics; do not add sleeps.

- **Verify under CI's Textual, not just the local one.** Build a throwaway venv the way
  CI does, e.g. `uv venv --python 3.12 /tmp/<name>` then
  `SASE_CORE_WHEEL=<a sase_core_rs 0.36.0 cp312-abi3 wheel built from 1ad57ea> just --set venv_dir /tmp/<name> install`.
  A cached wheel lives under `~/.sase/cache/sase-core-artifacts/*/`. Confirm
  `textual.__version__` is ≥ 8.2 there, and run
  `/tmp/<name>/bin/python -m pytest -p no:randomly tests/ace/tui/command_line/test_chrome_layout.py`.
  Also run it in the normal workspace venv (Textual 8.0.1). Both must pass. Delete the
  throwaway venv afterwards.
- Do **not** pin Textual in `dev`/base dependencies here (see Follow-ups).

### 5. ACE agents-loader tests not updated for the baseline-inbox change (3 tests)

Commit `548bbe9284` ("fix startup missing rows with baseline inbox and roster
completion") intentionally changed loader behavior but left these tests stale:

- `tests/ace/tui/actions/test_agent_loader_phase5_index_wiring.py::test_load_agents_from_disk_uses_artifact_index_for_initial_tier`
  and `::test_viewport_window_keeps_tier1_caps` assert
  `query.recent_completed_limit == 200`. Tier-1 queries now use
  `_TIER1_VISIBLE_COMPLETED_LIMIT = 2000`
  (`src/sase/ace/tui/models/_agent_loader_artifacts.py:34`); the 200 cap
  (`_TIER1_RECENT_COMPLETED_LIMIT`) now only bounds O(archive) source scans. A baseline
  Tier-1 cached read with no viewport now also sends `window_limit=2000` (read the
  `is_baseline_read` branch near line 294).
  - **Fix:** update the expectations to the new contract. Assert
    `recent_completed_limit == 2000` (importing `_TIER1_VISIBLE_COMPLETED_LIMIT` is
    acceptable in this white-box wiring test). Keep the viewport test's
    `window_limit == 120` assertion. For the initial-tier test, assert the baseline
    `window_limit` the code now sends, reading the current code path to pick the exact
    expectation. Refresh any docstring/comment that still describes the 200 cap as the
    visible limit.
- `tests/ace/tui/actions/test_agent_search_history_split.py::test_bounded_agents_viewport_expands_near_prefix_end`
  (`assert [] == ['navigation']`). `agents_viewport_for_load`
  (`src/sase/ace/tui/actions/agents/_loading_disk_viewport.py`) now returns `None` (a
  baseline read) unless `app._agents_roster_complete_query_key` equals
  `current_agents_history_query_key(app)`. The fixture never sets it, so the expansion
  never schedules.
  - **Fix:** in the test (or the `_SearchLoadApp` fixture if other tests need the same
    state), set
    `app._agents_roster_complete_query_key = current_agents_history_query_key(app)`
    before calling `_maybe_schedule_agents_viewport_expansion`, importing the function
    from `sase.ace.tui.actions.agents._loading_disk_viewport`. Verified: with that one
    line the test's existing assertions (`["navigation"]`, last requested limit `141`)
    pass unchanged.

### 6. Bead-attachment feature left three inventories stale (3 tests)

The bead note-attachment work (`9cc4570` attach verbs, `c2aec59` attachment surface, the
content-addressed store, and the sase-core `attachment` builtin kind already inside the
current pin) added surfaces that three guard tests enumerate:

- `tests/test_bead/test_cli_command_routing_acceptance.py::test_bead_subcommand_inventory_is_classified_for_routing_acceptance`:
  `attach` and `attachment` are registered but unclassified. Both handlers
  (`src/sase/bead/cli_attach.py`, `src/sase/bead/cli_attachment.py`) resolve their
  existing bead id through `resolve_bead_operation_context([id])`, the same
  cross-project routing path as `note`/`show`. **Fix:** add both to
  `ROUTED_EXISTING_ID_COMMANDS`, keeping the set sorted.
- `tests/test_agent_artifact_directory_operation_audit.py::test_artifact_directory_operation_sites_are_reviewed`:
  new unreviewed site `src/sase/bead/attachments/store.py:remove`. It `rmtree`s only
  `<SASE_HOME>/attachments/views/<sha256[:16]>` (and unlinks the object file), which is
  not an agent artifact directory. **Fix:** add a `_REVIEWED_DIR_OPERATION_CONTEXTS`
  entry with a `DirOpReview(exemption=...)` saying it removes only a content-addressed
  attachment view directory under `SASE_HOME/attachments/views`, not an agent artifact
  directory. Match the neighboring entries' style and ordering.
- `tests/artifact_refs/test_context.py::test_context_assembles_dynamic_document_role_and_namespaces`:
  `context.known_kinds` now contains the builtin `attachment` kind (from sase-core
  `artifact_ref/kinds.rs`), between `file` and `tool`. **Fix:** insert `"attachment"`
  after `"file"` in the expected tuple.

### 7. `prompt stash-archive restore` left completion metadata stale (3 tests)

`4f4764b42d` (recoverable stash archive) added `sase prompt stash-archive restore ID...`
(dest `ids`, `src/sase/main/parser_prompt.py`) without completion metadata.

- `tests/completion/test_kind_coverage.py::test_every_value_slot_is_kinded_choiced_or_hinted`:
  "uncaptioned completion value slots: prompt/stash-archive/restore:ids". **Fix:** add
  `"ids"` to the `"text"` hint set in `_VALUE_HINT_TABLE`
  (`src/sase/completion/kinds.py`), next to the existing `"id"` entry. This mirrors how
  bare `id` is handled: bead `ids` slots already carry a `_BEAD_ID_SLOTS` /
  `PATH_OVERRIDES` kind, which wins during coverage, so their behavior is unchanged.
  Keep the tuple sorted.
- `tests/completion/test_snapshot.py::test_checked_in_snapshot_has_no_drift` and
  `::test_current_structural_view_matches_checked_in_snapshot`: the checked-in CLI
  completion spec is out of sync. **Fix:** after the `kinds.py` change, run
  `just sync-completion-spec` and commit the regenerated snapshot. Inspect the diff:
  expect the stash-archive subtree and any other already-landed parser additions, such
  as the bead `attach`/`attachment` subcommands. Nothing unrelated should change.

## Verification

1. `just install` (rebuilds the core from a sase-core checkout containing `1ad57ea`).
   Then `.venv/bin/python tools/check_sase_core_rs_bindings` must pass.
2. Run every module that failed in CI in one xdist process so cross-module collection
   effects are exercised:

   ```bash
   .venv/bin/python -m pytest -p no:randomly -n 8 \
     tests/core/test_prompt_prediction_facade.py tests/test_check_sase_core_rs_bindings_tool.py \
     tests/test_partial_launch_cleanup.py tests/test_launch_admission_dispatch_hold_lifecycle.py \
     tests/test_force_reuse_launch_seam_consume.py tests/test_force_reuse_launch_seam_registry.py \
     tests/test_force_reuse_launch_seam_rejection.py tests/test_proc_env_isolation.py \
     tests/test_bead/ tests/tool/ tests/test_contract_manifest.py \
     tests/ace/tui/command_line/test_chrome_layout.py \
     tests/ace/tui/actions/test_agent_loader_phase5_index_wiring.py \
     tests/ace/tui/actions/test_agent_search_history_split.py \
     tests/ace/tui/test_prompt_bar_submit_no_cancel_save.py \
     tests/ace/tui/test_entry_points_vcs_prefix_mru.py \
     tests/test_agent_artifact_directory_operation_audit.py \
     tests/artifact_refs/test_context.py tests/completion/
   ```

3. Full-suite collection must be clean (this is what exposed root cause 3):
   `.venv/bin/python -m pytest -p no:randomly -p no:xdist --collect-only -q | tail -3`
   must show no errors.
4. Chrome-layout tests pass under both Textual 8.0.1 (workspace venv) and ≥ 8.2 (the
   throwaway CI-like venv from root cause 4).
5. `just fmt` / `just fix`, then `sase tool run check` (the agent default; do not run
   `just check-full`, per the check-full-is-explicit decision).
6. Optionally reproduce a CI shard locally for extra confidence:
   `SASE_TEST_SHARD=<n>/8 just test`, or run through `sase tool run` if the recipe is
   guarded. Not required when steps 2–5 are green.

## Follow-ups (file with `/sase_new_task`; do not fix here)

- **Legacy `sys.path.append(<src/sase>)` hacks.** About 13 import-time mutations exist
  under `src/sase/ace/**`: `status.py`, `operations.py`, `revert.py`, `pr_status.py`,
  `archive.py`, `restore.py`, `mail_ops.py`,
  `handlers/{reword,mail,workflow_handlers}.py`, `workflows/crs.py`, `tui/app.py`, and
  `tui/modals/status_modal.py`. They make every `src/sase` subpackage importable as a
  top-level name, which caused root cause 3 and can shadow real third-party/top-level
  modules in production. Type `bug`.
- **CI/dev dependency drift.** The fast shards resolve the newest Textual (8.2.8) from
  unpinned `textual[syntax]>=0.45.0`, while agent workspace venvs keep 8.0.1 and the
  `visual` extra pins 8.0.1. The result is CI-only failures agents cannot reproduce. The
  bead should decide between pinning or bounding Textual for `.[dev]` and having
  `just install` upgrade already-installed deps. Type `feature` or `bug`, at the
  implementer's judgment.
- **Stale core-pin ratchet PR #319** (`core-pin-ratchet` branch, "bump pinned sase-core
  revision to f88fb255e1c1") targets a commit **older** than the current pin. Merging it
  would regress the pin and re-break lint. Mention it in the final report so the user
  can close it; do not close or modify the PR from the agent.

## Non-goals

- No production behavior changes. Every fix is a test expectation or test-infra
  correction, the core pin bump, one completion-hint table entry, and the regenerated
  completion snapshot.
- No `just check-full`, no visual-golden regeneration, no shard-timing refresh.
