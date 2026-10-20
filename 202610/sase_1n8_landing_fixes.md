---
tier: tale
size: small
title: Finish landing epic sase-1n8 (controller test fixtures, TUI import budget,
  closeout)
goal: The two regressions epic sase-1n8 introduced are fixed, `just check` shows no
  epic-caused failures, and epic sase-1n8 is closed with its plan file marked done.
proposed_by: bbugyi200.athena.sase-1n8.land
proposed_with: claude/opus
bead: sase-1n8
status: done
---

- **PARENT:**
  [202610/host_completion_loop_and_disk_churn.md](https://github.com/sase-org/sase--plans/blob/main/202610/host_completion_loop_and_disk_churn.md)
- **BEAD:**
  [sase-1n8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n8/README.md)

# Finish landing epic sase-1n8

Epic `sase-1n8` ("Stop unsatisfiable host-completion loops and the disk churn they
caused") has all six phases closed and landed: sase commits `25d14fae6c`, `e77280c873`,
`321d881250`, `379040e699`, `2648c7cddf`, and sase-core commits `b664b15b` and
`2f433851` (sase's `sase-core-revision.txt` pins `b664b15b`). The land agent verified
every phase against the plan and found two regressions the epic caused. Both still
reproduce on master `2648c7cddf`. Fix them, then close the epic in this same turn.

The land agent already finished follow-up triage and recorded it on the epic (its
`LAND TRIAGE` note). Do not create any beads.

## Background

1. **Host-completion controller tests fail (7 nodes).** Phase `sase-1n8.2` made
   `prepare_conditional_completion` (`src/sase/finalizers/_prepare_completion.py`)
   resolve `stage_source` from the project tool catalog. It passes `run_silent` only
   when a catalog tool's `argv` equals the verification command and declares
   `stages: run_silent`; anything else seals `none`, and the core refuses `none`.
   `tests/monitor/test_monitor_host_completion_controller.py` prepares a
   `["just", "check-full"]` completion in a tmp project (`CODEX_PROJECT_DIR` points at a
   tmp repo with no catalog). Every test that goes through `_bind_prepared` now fails
   with
   `FinalizerDeclarationError: ... bound 'just check-full' reports no stage evidence`.
   These are:
   - `test_real_declaration_submit_is_not_stubbed_on_success_path`
   - `test_real_controller_success_uses_zero_model_calls`
   - both `test_stale_or_missing_checks_use_one_recovery` params
   - `test_unsupported_finalizer_uses_one_recovery`
   - `test_tree_drift_after_finalizers_uses_one_recovery`
   - `test_two_repository_partial_crash_retains_receipts_and_does_not_complete`

   `tests/test_final_prepare.py` already solves this hermetically. It has an autouse
   fixture `_run_silent_check_catalog` built on private helpers `_catalog_entry` and
   `_stub_tool_catalog`, which monkeypatch
   `sase.config.tools.load_project_tool_catalog_at`. The fixture also clears
   `DELIVERY_ARTIFACTS_ENV`, so `_collect_prior_refusals` stays hermetic.

2. **The TUI import budget is exceeded.**
   `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
   fails with 3524 modules against the 3522 cap. Measured with
   `.venv/bin/python tools/tui_import_closure -d 3dbe570462`: pre-epic was 3521, and the
   epic added three modules:
   - `sase.axe.run_agent_runner_scratch` and `sase.core.managed_tmp_reaper`, through the
     top-level
     `from sase.axe.run_agent_runner_scratch import cleanup_launch_scratch_key` in
     `src/sase/monitor/proc_adapter.py`;
   - `sase.procs.supervisor_storage`, through the top-level
     `from .supervisor_storage import last_supervisor_error` in
     `src/sase/procs/submission.py`.

   The land agent prototyped deferring both imports into their only callers. The closure
   measured 3521, and the budget test, `tests/test_procs_supervisor_storage.py`, and
   `tests/monitor/test_monitor_proc_settlement.py` all passed. No test monkeypatches
   either name on `proc_adapter` or `submission`.

## Steps

### 1. Defer the epic's new import edges (`src/sase`)

- `src/sase/monitor/proc_adapter.py`: remove the module-level
  `from sase.axe.run_agent_runner_scratch import cleanup_launch_scratch_key`. Import it
  at the top of `_cleanup_starter_cargo_target` (its only user). Add a short comment
  that the import is deferred to keep the scratch reaper out of the TUI import closure
  (import budget), matching the existing `# ... (import budget)` comments in `src/sase`.
  Keep `read_recorded_scratch_key` at module level; its module was already in the
  closure.
- `src/sase/procs/submission.py`: remove the module-level
  `from .supervisor_storage import last_supervisor_error`. Import it inside
  `_reconcile_named_proc`'s `supervisor-loss` branch, right before the
  `last_supervisor_error(current.proc_id)` call, with the same import-budget comment.
- Confirm with `.venv/bin/python tools/tui_import_closure`: it must report no more than
  3521 modules, and none of the three modules above may appear in `-d 3dbe570462`'s
  added list.

### 2. Share the run_silent catalog stub and use it in the controller tests (`tests/`)

- Move `_catalog_entry` and `_stub_tool_catalog` out of `tests/test_final_prepare.py`
  into `tests/finalizer_declaration_channel_test_helpers.py`. Both test modules already
  import from it. Make them public test helpers (`catalog_entry`, `stub_tool_catalog`).
  Add `stub_run_silent_check_catalog(monkeypatch)`, which does what the current fixture
  body does: clear `DELIVERY_ARTIFACTS_ENV` and stub a catalog with `check` →
  `[just, check]` and `check-full` → `[just, check-full]` (both `run_silent`), plus
  `install-venv` → `[just, install-venv]` (`none`).
- `tests/test_final_prepare.py`: make `_run_silent_check_catalog` call
  `stub_run_silent_check_catalog`. Update the tests that call `_stub_tool_catalog(...)`
  directly to call `stub_tool_catalog(...)`. Change no assertions.
- `tests/monitor/test_monitor_host_completion_controller.py`: add an autouse fixture
  that calls `stub_run_silent_check_catalog(monkeypatch)`. Its docstring should say that
  it mirrors the sase checkout's stage-reporting check tools, so prepare seals
  `stage_source: run_silent`. Change no test assertions.
- Run
  `.venv/bin/python -m pytest tests/monitor/test_monitor_host_completion_controller.py tests/test_final_prepare.py tests/monitor/test_no_new_receipt_prepare.py tests/ace/tui/test_app_import_budget.py tests/test_procs_supervisor_storage.py tests/monitor/test_monitor_proc_settlement.py -q`.
  Everything must pass.

### 3. Verify

- Run `just fmt`, then `sase tool run check` (the wrapped `just check`; never
  `just check-full`).
- Known failure that is not this epic's:
  `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
  (`tests/contract_manifest.txt` is missing
  `tests/test_github_actions_release_steward.py`). Commit `812cdd9e21`
  (`sase-1n6.7.1.5`) caused it, and the `sase-1n6.7.1` land agent's tale owns the fix.
  Do not fix it here, and do not file a bead for it.
- Treat any other failure as yours unless triage labels it KNOWN/FLAKY or you show it
  reproduces on clean master `2648c7cddf`. Fix it if this tale's changes or the epic
  caused it.

### 4. Close out epic sase-1n8 (this turn, after steps 1-3 pass)

1. Run `sase bead epic-symbols sase-1n8`. At planning time it printed
   `No --epic-symbol entries for sase-1n8.` If entries have appeared, resolve each one
   per the Symvision epic-whitelist policy (wire it up, privatize it, add a non-test
   pragma, or delete it) before closing.
2. Run `sase bead read sase-1n8 -r "Need the PLAN path and parent link for closeout"`.
   It has no `parent_bead`, so no parent bead needs handling after the close.
3. Close the epic. Do not use `--force`. Adjust the note if your verification differed:

   ```bash
   sase bead close sase-1n8 --note "Verified all six phases against plan:202610/host_completion_loop_and_disk_churn.md and the epic commits (sase 25d14fae6c, e77280c873, 321d881250, 379040e699, 2648c7cddf; sase-core b664b15b, 2f433851; pin b664b15b). Every DECISION was honored (no_stage_source=refuse, repeat_guard=guard). Land fixes for two epic-caused regressions: deferred proc_adapter's scratch-cleanup import and submission's supervisor_storage import, which restores the TUI import closure to 3521 under the 3522 cap; shared the run_silent catalog stub and applied it to the 7 failing host-completion controller tests. Integration: post-epic master commits (3dbe570462, 3cb4c57ca3, b79549090b, d9f711bbca, 043d67a919) do not touch the epic's surfaces. Generated skills were deployed from landed master. Follow-up triage is in the LAND TRIAGE note: +1 sase-xo, +1 sase-1gz, new feature sase-1n9; the stale contract manifest is owned by sase-1n6.7.1; the models-panel flake was declined for insufficient evidence. just check passed apart from the sase-1n6.7.1-owned contract manifest."
   ```

4. Run `just symvision` and confirm it is clean.
5. Set `status: done` (it is currently `status: wip`) in the YAML frontmatter of the
   epic's plan file: the `PLAN` path that `sase bead read sase-1n8` prints for
   `plan:202610/host_completion_loop_and_disk_churn.md`. Change nothing else in that
   file.

You do not commit. The host commits your changes after the turn ends. Closing the epic
in this same turn is the normal landing; do not wait for a commit, push, or CI.
