---
tier: tale
size: medium
title: Finish and land epic sase-1ca (Never lose stashed prompts)
goal:
  The pytest session HOME/SASE_HOME sandbox holds for every test, not just each worker's
  first; the remaining sase-1ca restore, stop-axe-quit, logging, and archive-hint gaps
  are fixed with regression tests; and epic sase-1ca is closed, symvision is clean, and
  its plan file is marked done.
proposed_by: bbugyi200.athena.sase-1ca.land
bead: sase-1ca
status: done
---

- **PARENT:**
  [202609/never_lose_stashed_prompts.md](https://github.com/sase-org/sase--plans/blob/main/202609/never_lose_stashed_prompts.md)
- **BEAD:**
  [sase-1ca](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ca/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1ca.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1ca.land.md)
- **COMMITS:**
  - [119f97d](https://github.com/sase-org/sase/commit/119f97da8078cc265d6f2219cd94c91b8cbda43e)
    — feat(prompt-stash): land sase-1ca hardening gaps 1-6 with regression tests

# Finish and land epic sase-1ca: seal the session HOME sandbox, fix small hardening gaps, close the epic

## Context

Epic `sase-1ca` ("Never lose stashed prompts", plan
`plan:202609/never_lose_stashed_prompts.md`) has all six phase beads closed
(`sase-1ca.1`–`sase-1ca.6`). Its land agent reviewed every phase against the code and
commits:

- sase: `ee75c62d81`, `aba5d035c2`, `703b042c23`, `67f4a1d1ec`, `4f4764b42d`.
- sase-core: `df23cce`, which is pinned in `sase-core-revision.txt`.

Phases 2, 3 and 6 match the plan: the prompt-store pytest guards, the fsynced
fail-closed Rust archive, and the `sase prompt stash-archive` CLI. Phases 1, 4, 5 and 6
each left a gap. **Gap 1 below is a real hole in the epic's main safety goal.** The rest
are small correctness and copy fixes in code the epic added.

The land agent already triaged follow-ups and recorded the outcome in a note on
`sase-1ca` ("LAND FOLLOW-UP TRIAGE"). Tasks `sase-1cd`, `sase-1ce`, `sase-1cf`,
`sase-1cg`, `sase-1ch` and `sase-1ci` exist, and a DISCOVERED ISSUE note is on
`sase-1bc`. **Do not** fix those items here, and do not create beads for them again.

Before you finish, read `lint_and_test.md` with `/sase_memory_read`. Also read `tui.md`
before touching TUI text that may appear in visual snapshots.

## Gap 1 (main fix): the session HOME/SASE_HOME sandbox only covers each worker's first test

Phase 1 added the session-scoped autouse fixture `_sandbox_session_home`
(`tests/_conftest_environment.py`). It points `HOME` and `SASE_HOME` at a per-worker tmp
dir, so module-scoped fixtures and any stray function-level `monkeypatch.undo()` fall
back to a sandbox instead of the account home.

The problem is the per-test env snapshot in `tests/conftest.py::pytest_runtest_protocol`
(`snapshot_sase_environment` / `restore_sase_environment` in
`tests/_sase_global_state_isolation.py`):

- It takes its snapshot _before_ the session fixtures run on the first test.
- After that first test, it restores `HOME` to the real account home and removes
  `SASE_HOME`.
- From then on, every module-scoped fixture and every undo fallback sees the real home
  again.

The land agent reproduced this with two throwaway test modules, each with a
module-scoped fixture recording `os.environ["HOME"]`:

- Module A (first test) saw `.../pytest-0/session-home0`.
- Module B saw `/home/bryan` with `SASE_HOME=None`.

`tests/conftest.py::_disable_detach_scope_by_default` documents this exact trap for its
own env keys: they "must stay in `_ENV_KEYS_TO_IGNORE`".

Fix:

1. In `tests/_sase_global_state_isolation.py`, add `"HOME"` and `"SASE_HOME"` to
   `_ENV_KEYS_TO_IGNORE`, keeping the set alphabetized. The leak-detector fingerprints
   (`tests/_global_state_leaks/fingerprints.py`) import the same set, so they stay in
   sync.
   - This is safe. The autouse `_isolate_sase_home` fixture re-sets both keys through
     the function-level `monkeypatch` on every test. Its teardown restores them to the
     session-sandbox values, so a per-test change still cannot leak.
   - The land agent prototyped this change: module B then saw the sandbox, and
     `tests/test_global_state_leak_detector.py`,
     `tests/test_sase_global_state_isolation.py` and
     `tests/test_pytest_isolation_guards.py` all passed.
2. Extend the `_sandbox_session_home` docstring to say `HOME`/`SASE_HOME` must stay in
   `_ENV_KEYS_TO_IGNORE` and why, mirroring `_disable_detach_scope_by_default`.
3. In
   `tests/test_global_state_leak_detector.py::test_process_snapshot_uses_isolation_env_ignore_list`,
   also assert that `"HOME"` and `"SASE_HOME"` are in the ignore set.

## Gap 2: the seal regression test is a tautology

`tests/test_pytest_isolation_guards.py::test_session_sandbox_seals_home_after_function_patches_lost`
simulates losing the function-level patches by calling
`scoped.setenv("HOME", str(sandbox))` itself. It would pass even if
`_sandbox_session_home` were deleted, and it did not catch Gap 1. Replace it with real
coverage:

1. Add a module-scoped fixture to that module that captures `os.environ.get("HOME")` and
   `os.environ.get("SASE_HOME")`.
   - A module-scoped fixture runs after the session fixtures and before the
     function-level isolation, so it sees exactly what an undo of the function-level
     patches reverts to.
2. Rewrite the seal test to use that captured baseline:
   - Assert the baseline `HOME` and `SASE_HOME` resolve at or below
     `SASE_PYTEST_SANDBOX_DIR`.
   - Inside `monkeypatch.context()`, set `HOME`/`SASE_HOME` back to the baseline values
     (deleting a key whose baseline is `None`).
   - Assert that `sase.core.paths.sase_home()` resolves inside the sandbox and is not
     `sase.core.state_write_guard._account_home() / ".sase"`.
   - Delete the self-fulfilling `setenv("HOME", str(sandbox))`.
3. Add an ordering regression test that guarantees the seal test is never a worker's
   first test.
   - Model it on
     `tests/test_workspace_metadata_cache_teardown.py::test_metadata_patch_teardown_does_not_poison_a_later_test`:
     `pytester.runpytest_subprocess` with `-p no:randomly`, `-c <repo>/pyproject.toml`,
     `--rootdir <repo>`, and `timeout=60`.
   - Pass exactly two node IDs from this file:
     `test_no_undo_on_shared_monkeypatch_fixture` first, then the seal test. Assert
     `passed=2`.
   - Never select the ordering test itself, so it cannot recurse.
4. Prove the new tests catch Gap 1. Temporarily remove `HOME`/`SASE_HOME` from the
   ignore set and confirm the ordering test fails. Restore the fix and confirm it
   passes.

## Gap 3: the stash-task failure log has no traceback

In
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_store.py::_spawn_prompt_stash_task`,
the done-callback calls `logging.getLogger("sase").exception(...)` outside an `except`
block. `sys.exc_info()` is empty there, so `tui.log` gets `NoneType: None` instead of
the task's traceback.

- Change it to
  `logging.getLogger("sase").error("Prompt stash background task failed", exc_info=exc)`,
  or equivalent.
- Extend
  `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py::test_spawned_task_exception_is_logged_and_toasted`
  to assert that the log record's `exc_info` carries the raised exception type.

## Gap 4: the restore rollback claims success even when it failed

In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore_apply.py`,
`_rollback_stash_restore` swallows every `append_prompt_stash` failure (`continue`). The
caller then toasts "draft put back in the stash" anyway.

1. Make `_rollback_stash_restore` return whether every popped row was re-appended.
2. When it did not, toast an error saying the draft is still recoverable from the
   archive. Pop archives every removed row with reason `popped` (sase-1ca.3). Reuse
   `STASH_ARCHIVE_RECOVERY_HINT` from `src/sase/ace/tui/modals/_stash_trash_commit.py`.
3. Drop the `# pragma: no cover` markers on branches that are now tested.
4. Add a test next to `test_confirm_load_failure_rolls_row_back_into_stash`, using tmp
   stores only:
   - both the bar load and the rollback append fail;
   - the toast names `sase prompt stash-archive`;
   - the popped row is present in `read_prompt_stash_archive(...)` with reason `popped`.

## Gap 5: stop-axe-and-quit stops the scheduler before a stash failure cancels the exit

In `src/sase/ace/tui/actions/axe.py::_stop_axe_and_quit`, the method stops the stall
watchdog and then the scheduler (`stop_service_proc`). Only after that does it reach
`_begin_controlled_exit`, where a failed quit-draft stash cancels the exit. The result
is a TUI that stays open with its scheduler and watchdog already stopped.

1. Call the quit-draft stash first: `getattr(self, "_stash_quit_draft_or_cancel", None)`
   from `lifecycle.py`.
2. If it returns `False`, return before touching the watchdog or the scheduler.
3. A successful stash sets `_quit_draft_stash_attempted`, so the later
   `_begin_controlled_exit` will not stash twice.
4. Add a test in `tests/ace/tui/test_quit_prompt_stash.py`: with a failing stash,
   `_stop_axe_and_quit` does not call `stop_service_proc` (patch
   `sase.service.actions.stop_service_proc`), does not stop the watchdog, does not start
   the controlled exit, and toasts the error.

Do **not** change `_restart_tui`: restart's stash-failure behavior is task `sase-1cg`.

## Gap 6: the archive-hint copy contradicts itself or is missing

1. `src/sase/ace/tui/modals/trash_pane.py::purge_confirm_text`:
   - The first line says "Permanently delete N drafts? This cannot be undone." but the
     text then says drafts stay recoverable with `sase prompt stash-archive`.
   - Change the first line to "Permanently delete N drafts from Trash?" and drop the
     "cannot be undone" sentence.
2. `src/sase/ace/tui/modals/_stash_trash_commit.py`:
   - In `trash_outcome_text`, append `. {STASH_ARCHIVE_RECOVERY_HINT}` when `evicted` is
     non-empty. This is the Stash→Trash commit toast that reports trash-limit evictions.
   - In `trash_commit_confirm_text`, append the same hint on the expected-evictions
     line.
3. Update the tests that pin the old strings. Search for `cannot be undone`,
   `purge_confirm_text`, `trash_outcome_text` and `trash_commit_confirm_text`; the
   likely files are `tests/ace/tui/modals/test_trash_pane.py`,
   `tests/ace/tui/modals/test_prompts_modal_trash.py` and
   `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py`.
4. If a PNG golden renders one of these strings, follow `tui.md` / `lint_and_test.md` to
   regenerate it.

## Verification

- Run the directly affected modules first:
  - `tests/test_pytest_isolation_guards.py`
  - `tests/test_global_state_leak_detector.py`
  - `tests/test_sase_global_state_isolation.py`
  - `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py`
  - `tests/ace/tui/actions/test_prompt_stash_handler.py`
  - `tests/ace/tui/test_quit_prompt_stash.py`
  - `tests/ace/tui/modals/test_trash_pane.py`
  - `tests/ace/tui/modals/test_prompts_modal_trash.py`
  - `tests/prompt_command/test_stash_archive.py`
- Then run the full file-change gate, `sase tool run check`. Do not run
  `just check-full`.
- Two known failures do not belong to this work. Report them, but do not fix them or
  file them again:
  - `lint (feature flags)` rule 8 about flag bead `sase-1be` / key `agent_tabs`, owned
    by epic `sase-1bc`;
  - the three agent-loader tests tracked by `sase-1cd`.

## Final step: close out epic sase-1ca

After everything above passes:

1. Run `sase bead epic-symbols sase-1ca`. It listed nothing at planning time.
   - For every entry now listed, resolve the symbol: wire it up, privatize it, add a
     non-test pragma, or delete it, per the Symvision epic-whitelist policy.
   - Do not add new `--epic-symbol` entries keyed to `sase-1ca` or its phases.
2. Close the epic with a verification note:

   ```bash
   sase bead close sase-1ca --note "<verification>"
   ```

   The note must state:
   - Phases 1-6 were verified against sase commits `ee75c62d81`, `aba5d035c2`,
     `703b042c23`, `67f4a1d1ec`, `4f4764b42d` and sase-core `df23cce` (pinned).
   - The land fixes in this tale (Gaps 1-6), with the new regression tests and the
     `sase tool run check` result.
   - The integration review: concurrent commits `f379c64179`, `b390be0780`,
     `d1063d161c`, `a3327e02bf` and `96c03436e0` do not conflict with the epic. The
     sase-1bc availability edits touch a different branch of `check_app_action`, and no
     code calls the prompt-stash Rust bindings outside the guarded facade.
   - A pointer to the "LAND FOLLOW-UP TRIAGE" note already on `sase-1ca`, which covers
     tasks `sase-1cd` through `sase-1ci` and the declined items.

   Never pass `--force`.
   - If the close is refused because `--epic-symbol` entries remain, finish that cleanup
     and close again.
   - If it is refused for any other reason, record the blocker with
     `sase bead note sase-1ca "..."` and report it; do not force it.

3. Run `just symvision` and confirm it is clean.
4. Set `status: done` (it is currently `wip`) in the YAML frontmatter of the epic's plan
   file, at the path on the `PLAN` line of
   `sase bead read sase-1ca -r "Need the plan path for closeout"`. That path is
   `202609/never_lose_stashed_prompts.md` in the plans repo.
5. `sase-1ca` has no `parent_bead`, so no parent needs closing. Finish normally.
