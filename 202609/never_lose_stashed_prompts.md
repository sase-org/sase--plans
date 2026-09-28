---
tier: epic
title: Never lose stashed prompts
goal: 'A pytest process can never read or mutate the user''s real prompt stash or
  history, every row that permanently leaves the prompt stash is archived first and
  recoverable with `sase prompt stash-archive`, and the TUI''s restore, stash-capture,
  and quit paths can no longer silently drop a prompt draft.

  '
phases:
- id: seal-pytest-home
  title: Seal pytest home isolation and remove the stash-popping test bug
  depends_on: []
  size: medium
  description: 'seal-pytest-home: fix the monkeypatch.undo() test that popped the
    real stash, remove every undo() on the shared monkeypatch fixture, add an undo-proof
    session-level HOME/SASE_HOME sandbox, an AST guard test, a seal regression test,
    and SASE_HOME forwarding for tmux-launched TUIs.'
- id: guard-prompt-stores
  title: Hard pytest boundary on the prompt stash and prompt history stores
  depends_on: []
  size: small
  description: 'guard-prompt-stores: route every prompt_stash_facade read/mutation
    and every prompt-history write through assert_test_state_write_isolated so pytest
    can never touch the account''s real prompts, with tests.'
- id: core-stash-archive
  title: sase-core append-only archive for every permanent stash removal
  depends_on: []
  size: medium
  description: 'core-stash-archive: in the linked sase-core repo, archive (fsynced,
    fail-closed, same lock) every popped, purged, evicted, and overwritten stash row
    before rewriting the stash; add read_prompt_stash_archive and recover_prompt_stash_archive
    bindings; fsync appends; tests.'
- id: restore-capture-hardening
  title: Restore and capture hardening in the TUI
  depends_on: []
  size: medium
  description: 'restore-capture-hardening: restore loads from the pop outcome with
    fail-closed reads and rollback, background stash-task failures are logged and
    toasted, failed stash appends put the draft back in the bar, and @/q/Q are unavailable
    while a prompt owns keys.'
- id: quit-preserves-draft
  title: Quitting the TUI stashes an open prompt draft
  depends_on:
  - restore-capture-hardening
  size: small
  description: 'quit-preserves-draft: generalize the pre-restart stash helper and
    call it on explicit quit paths (source=quit), cancel the exit if the stash write
    fails, mention the draft in the quit-confirm impact, with tests.'
- id: stash-archive-recovery
  title: Stash-archive recovery surface (CLI, TUI hints, docs) and core pin
  depends_on:
  - guard-prompt-stores
  - core-stash-archive
  - restore-capture-hardening
  size: medium
  description: 'stash-archive-recovery: bump the sase-core pin, add guarded archive
    facade functions, the sase prompt stash-archive list/restore/show CLI, archive
    hints in purge/evict/delete toasts, docs, and tests.'
proposed_by: bbugyi200.athena.0tt.w0
create_time: 2026-09-28 17:30:04
status: done
bead_id: sase-1ca
---

- **PROMPT:** [prompts/202609/never_lose_stashed_prompts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/never_lose_stashed_prompts.md)
- **BEAD:** [sase-1ca](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ca/README.md)

# Never lose stashed prompts: seal the pytest leak that deleted a real stash row, and make every stash removal recoverable

## Why this epic exists

On 2026-09-28 the user stashed a `#research_swarm` prompt draft on athena. At 19:07:58Z,
the main TUI toasted "Stashed prompt". Ten minutes later the row was gone for good. It
was not in Stash, Trash, prompt history, or any log. The user asked that SASE **never**
lose a user prompt like this again.

### Root cause (confirmed)

A pytest test popped the row out of the user's **real** `~/.sase/prompt_stash.jsonl`.

- `tests/ace/tui/actions/test_prompt_stash_restore_open.py::test_failed_read_clears_flags_next_opens`
  calls `monkeypatch.undo()` in the middle of the test (line ~464). It meant to drop
  only its own failing-read patch. But the function-scoped `monkeypatch` fixture is
  shared with the autouse `_isolate_sase_home` fixture
  (`tests/_conftest_environment.py`, `redirect_sase_home`) and with the test's own
  `_point_store_at` patch. So the undo also restored the real `HOME` and `SASE_HOME` and
  the real `prompt_stash_path`.
- The test's next line runs `harness.action_restore_prompt_stash()` (the global `@`)
  against the real store. The real stash held exactly one unpinned row, so
  `auto_restore_single` (`_prompt_bar_stash_restore_overlay.py`) called
  `_apply_stash_restore(pop_ids=[id])`. That permanently deleted the row with
  `pop_prompt_stash` (sase-core `prompt_stash/store.rs`) and loaded the text into the
  throwaway test harness. The harness captures `notify()` in memory, so no toast was
  logged.
- Evidence:
  - ToolRun `4bd69d8aca04` was a `just check` run by another agent that escalated to the
    full suite.
  - Worker gw0 seeded the test's tmp stash at 15:17:24.030 EDT
    (`popen-gw0/test_failed_read_clears_flags_0/prompt_stash.jsonl`, rows `a` and `b`,
    both still present).
  - The real `~/.sase/prompt_stash.jsonl` was rewritten 48 ms later, at 15:17:24.078. It
    has not been written since.
  - The run logged `FAILED ...::test_failed_read_clears_flags_next_opens` with
    `assert len(harness.pushed) == 1` / `E assert 0 == 1`. That failure happens only
    when the store the test read held exactly one row.
- The test is a landmine that detonates only when the developer's real stash holds
  exactly one unpinned row. `919ba740e6` (2026-09-27) introduced the `undo()`. The user
  trashed every older stash row on 2026-09-27, which left a single-row stash.

### Design flaws that made the loss permanent and silent

1. **No pytest boundary on prompt stores.** `src/sase/core/state_write_guard.py`
   (`assert_test_state_write_isolated`) resolves the OS account home through `pwd`, so
   `HOME`/`SASE_HOME` games cannot fool it. It already protects telemetry, axe state,
   notifications and more. It does not protect the prompt stash facade
   (`src/sase/core/prompt_stash_facade.py`) or prompt history
   (`src/sase/history/prompt_store.py`).
2. **Function-level isolation is undo-able.** No session-level sandbox sits underneath
   it:
   - any `monkeypatch.undo()` (test body or fixture teardown) drops the test back onto
     the real home;
   - module-scoped `AcePageGroup` fixtures (for example
     `tests/ace/tui/widgets/test_vim_normal_key_containment.py`,
     `tests/ace/tui/widgets/test_prompt_tab_focus_steal.py`) mount `AceApp` before the
     function-scoped isolation and write `tui_startup` records into the real
     `~/.sase/logs`.

   The same `undo()` pattern exists in `tests/test_notification_modal_sent_at.py` (test
   body), `tests/tool/test_settlement.py` (test body) and the `tz_divergence` fixture in
   `tests/_conftest_runtime.py` (fixture teardown on the shared `monkeypatch`).

3. **Permanent removal with no archive.** These remove stash rows with no recoverable
   copy anywhere:
   - restore pops the row before it loads it into a (possibly doomed) prompt bar;
   - in-place delete pops;
   - Trash purge and trash-limit eviction delete;
   - `rewrite_prompt_stash` overwrites old text (gS update-pinned, directive rewrites).
4. **Restore is fragile after the pop.**
   - `_read_prompt_stash_entries` swallows read errors into `[]`, but the pop still
     commits.
   - An exception while loading the bar dies inside an un-awaited task
     (`_spawn_prompt_stash_task`) and is not logged to `tui.log`.
   - The loaded rows come from a separate snapshot read, not from what the store
     actually removed.
5. **Other ways a draft vanishes:**
   - "Stashed prompt" is toasted and the pane is removed before the append persists; a
     failed append loses the text.
   - Quitting the TUI discards a mounted prompt-bar draft; only restart stashes it.
   - The global `@` (`restore_prompt_stash`) and `q`/`Q` are not blocked while a prompt
     owns the keyboard. Commit `e54fceedd1` fixed the same focus-loss hole for Tab only.
6. **Two isolation gaps elsewhere.**
   - `src/sase/main/ace_tmux_window.py::_tmux_env_args` does not forward `SASE_HOME`. So
     a sandboxed caller of `sase screenshot` or ace tmux gets a TUI on the tmux server's
     real home. This played no part in this incident.
   - The Rust stash append does not fsync.

The apollo side is out of scope for code changes. At 16:29:17 EDT, apollo logged a
DANGER-confirmed "Permanently deleted 1 draft" purge from Trash. It is unclear whether
that was the lost prompt. The archive in this epic makes even confirmed purges
recoverable. The cause of the 2026-09-27 06:01 / 06:04 mass trash on both hosts is
unconfirmed. Those rows are still in Trash, and no test run failed on stash or trash
then.

### Target end state

- A pytest process cannot read or write the account's real prompt stash or prompt
  history. Tests cannot accidentally undo their home isolation.
- Every row that leaves the prompt stash permanently is first appended to an append-only
  archive, in the same store lock, fsynced, fail-closed. That covers restore-pop,
  delete, purge, eviction, and overwrite.
- The archive has a recovery path: `sase prompt stash-archive list|restore|show`.
- Restore loads exactly what the store removed, rolls back on failure, and surfaces
  every background failure.
- Stash capture is rolled back into the bar if it fails to persist. Quitting stashes a
  mounted draft. Typing `@`/`q` while a prompt owns the keyboard never fires a global
  action.

No feature flag: every phase lands a complete, unconditional safety fix. No keymap or
`src/sase/default_config.yml` changes are needed.

## Phases

### Phase 1: Seal pytest home isolation and remove the stash-popping test bug

- **repo:** sase

This is the direct root-cause fix, plus the structure that stops the class recurring.

1. Fix
   `tests/ace/tui/actions/test_prompt_stash_restore_open.py::test_failed_read_clears_flags_next_opens`:
   - Drop the mid-test `monkeypatch.undo()`. Scope the failing-read patch instead: apply
     it inside `with monkeypatch.context() as scoped:` around the first `@`, or remove
     just that instance attribute with `monkeypatch.delattr`. The tmp store redirect
     (`_point_store_at`) and the autouse home isolation must stay active.
   - Assert after the second `@` that the tmp store still holds rows `a` and `b` and
     that `sase.core.paths.prompt_stash_path()` still returns the tmp path.
2. Remove every other `undo()` on the shared, injected `monkeypatch` fixture:
   - `tests/test_notification_modal_sent_at.py` (test body): patch the formatters only
     where needed, or use `monkeypatch.context()`.
   - `tests/tool/test_settlement.py` (test body): use a scoped context for the `busy`
     patch.
   - `tz_divergence` in `tests/_conftest_runtime.py`: use its own `pytest.MonkeyPatch()`
     instance for `TZ`, so its teardown cannot revert other fixtures' patches.

   Grep for any more, including fixtures.

3. Add an undo-proof, session-level sandbox baseline:
   - Add a new session-scoped autouse fixture in `tests/_conftest_environment.py`,
     exported through `tests/conftest.py` like its siblings.
   - It owns a private `pytest.MonkeyPatch()` and points `HOME` at a per-worker
     `tmp_path_factory.mktemp(...)` directory and `SASE_HOME` at its `.sase` child. It
     undoes them at session end.
   - Session scope makes it run before any module-scoped fixture. After it lands:
     - module-scoped `AcePageGroup` fixtures no longer write to the real `~/.sase/logs`;
     - any stray function-level undo falls back to the sandbox, never the account home.
   - Keep `_isolate_sase_home` and `redirect_sase_home` behavior unchanged on top of it.
4. Add a guard test, for example `tests/test_pytest_isolation_guards.py`:
   - It AST-scans `tests/**/*.py` and fails, naming `file:line`, when `.undo()` is
     called on a function or fixture parameter named `monkeypatch`, meaning the injected
     fixture. Locally constructed `pytest.MonkeyPatch()` instances stay allowed.
   - Its message points at `monkeypatch.context()`.
5. Add a regression test proving the seal:
   - It runs under the suite's normal fixtures and simulates losing the function-level
     patches. Either use `pytester`, which the suite already enables, or temporarily
     clear the per-test env keys inside a `monkeypatch.context()`.
   - It asserts that `sase.core.paths.sase_home()` resolves at or below
     `SASE_PYTEST_SANDBOX_DIR`, never under the OS account home
     (`sase.core.state_write_guard._account_home()`).
6. Make `_tmux_env_args` in `src/sase/main/ace_tmux_window.py` forward `SASE_HOME` via
   `-e` when the caller's environment sets it. Add a unit test next to the existing ace
   tmux tests using the fake runner.

Verification: `sase tool run check`. Run the stash and restore test modules,
`tests/ace/tui/widgets/test_prompt_tab_focus_steal.py`, and
`tests/ace/tui/widgets/test_vim_normal_key_containment.py` directly too. Confirm that a
targeted run adds no new record to the developer's real `~/.sase/logs/tui_startup.jsonl`
(compare line counts before and after).

### Phase 2: Hard pytest boundary on the prompt stash and prompt history stores

- **repo:** sase

Defense in depth that holds even if every test fixture is wrong. It uses the existing
account-home guard in `src/sase/core/state_write_guard.py`.

1. In `src/sase/core/prompt_stash_facade.py`, every public function calls
   `assert_test_state_write_isolated(path, category="prompt stash")` before it touches
   the Rust binding:
   - reads: `read_prompt_stash_snapshot`, `read_prompt_stash_lifecycle`;
   - mutations: `append`, `pop`, `set_pinned`, `rewrite`, `trash`, `restore`, `purge`,
     `reconcile`.

   Guard reads as well as mutations. This incident started with a test reading the real
   row count, and no test has a legitimate reason to read the account's real prompts.
   Outside pytest the call is a no-op.

2. In `src/sase/history/prompt_store.py`, guard the prompt-history write paths
   (`save_shard`, `save_prompt_history`, and any other writer found there or in
   `prompt_store_mutations.py`) with `category="prompt history"`.
   `src/sase/agent/failed_launch_prompt_stash.py` goes through the facade, so confirm it
   is covered.
3. Add tests (a new module or next to the existing `state_write_guard` tests):
   - Under pytest, a facade read or mutation aimed at
     `<account_home>/.sase/prompt_stash.jsonl` raises. Point
     `state_write_guard._account_home` at a tmp dir so the test never touches the real
     home.
   - A sandbox path works normally.
   - The same two checks hold for a prompt-history shard write.
4. Run the whole stash, restore, trash and prompt-history suites to prove no legitimate
   test depends on the real home. Fix any that do.

Verification: `sase tool run check`.

### Phase 3: sase-core append-only archive for every permanent stash removal

- **repo:** sase-core, the linked repo (open it with `sase repo open sase-core`; follow
  its `AGENTS.md`)

Makes "never lose" true at the store level, where the lock and the atomic rewrite live.
Work in `crates/sase_core/src/prompt_stash/` (`store.rs`, `wire.rs`, `mod.rs`) and the
existing bindings in `crates/sase_core_py/src/editor_content/`.

1. Archive file:
   - It is a sibling of the stash file, derived from the stash path the same way the
     lock path is: `prompt_stash.jsonl` → `prompt_stash_archive.jsonl` in the same
     directory.
   - Each line is a new `PromptStashArchiveRecordWire`:
     `{ "kind": "archived", "archived_at": <RFC 3339 UTC>, "reason": "popped" | "purged" | "evicted" | "overwritten", "pid": <u32>, "trashed_at": <optional, for rows that were in Trash>, "entry": <full PromptStashEntryWire> }`.
   - Use `chrono`, already a dependency, for `archived_at`.
2. Inside the existing exclusive lock, append the archive lines and fsync the archive
   file **before** `write_parsed_atomic` rewrites the stash. If the archive append
   fails, return an error and leave the stash untouched (fail closed). A crash in
   between can duplicate a row in the archive, but never lose one. Archive:
   - `pop_prompt_stash`: every removed active row, reason `popped`.
   - `purge_prompt_stash`: every purged trash row, reason `purged`.
   - `trash_prompt_stash` and `reconcile_prompt_stash_trash`: every trash-limit
     eviction, reason `evicted`, including limit 0.
   - `rewrite_prompt_stash`: the previous version of every active row whose `text`,
     `frontmatter` or `cursor` the input replaces, reason `overwritten`. Merge-preserved
     rows are not archived.
3. Add read and recover APIs:
   - `read_prompt_stash_archive(path, limit)` returns a
     `PromptStashArchiveSnapshotWire { schema_version, records (newest-first), stats }`.
     It tolerates malformed lines and counts them in stats, like the stash reader. A
     missing file reads as empty. It takes the shared lock.
   - `recover_prompt_stash_archive(path, ids)`, under the exclusive lock:
     - For each id in caller order, it appends the newest archived version of that entry
       back to the active stash.
     - Ids currently active or in Trash, and unknown ids, are skipped and reported.
     - It returns a `PromptStashLifecycleOutcomeWire`-shaped result (recovered ids in
       `changed`) with the authoritative snapshot.
     - The archive stays append-only; recovery does not delete archive lines.
4. Make `append_prompt_stash_unlocked` fsync (`sync_all`) after writing, so an
   acknowledged stash survives a crash.
5. Expose the two new functions to Python next to the existing prompt-stash bindings
   (same naming pattern: `read_prompt_stash_archive`, `recover_prompt_stash_archive`).
   This is a non-breaking, additive `feat:` wire change; existing binding signatures do
   not change. Follow AGENTS.md: never edit versions or CHANGELOG.
6. Tests:
   - in `crates/sase_core/tests/` (extend `prompt_stash_trash_lifecycle.rs` or add
     `prompt_stash_archive.rs`): each removal path archives the full entry with the
     right reason; archive-before-rewrite ordering holds (make the archive path
     unwritable, for example a directory, and assert the stash is unchanged and an error
     is returned); recovery round-trips and skips active or trashed ids; malformed
     archive lines are tolerated;
   - binding tests in `sase_core_py`.

Verification in the sase-core checkout: `sase tool run check`, the full gate, not a
targeted run.

### Phase 4: Restore and capture hardening in the TUI

- **repo:** sase

1. In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore_apply.py`,
   rework `_apply_stash_restore`:
   - Load the popped rows from `outcome.removed`, which is what the store actually
     removed, not from the separately read snapshot.
   - Only `keep_ids` still need a snapshot read, and that read must be fail-closed: add
     a raising variant of `_read_prompt_stash_entries`, or call the facade directly. The
     badge-count readers keep their lenient behavior.
   - If the read fails, do not pop.
   - If building or loading panes raises after a pop, append the removed entries back to
     the stash with their original ids, which is allowed because popped rows are not in
     Trash. Toast an error that says the draft was put back.
   - Keep the existing count-aware success toasts.
2. In `_prompt_bar_stash_store.py`, give `_spawn_prompt_stash_task` a done-callback. It
   logs any unhandled exception through the `sase` logger, so it reaches `tui.log`, and
   toasts a short error. Cancelled tasks stay silent.
3. In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash.py`, fix stash-capture
   failure in `on_prompt_input_bar_stashed` / `_persist_prompt_stash_entry_async`:
   - Keep the optimistic toast and teardown.
   - If the append fails (error or lock timeout), put the captured panes back into the
     prompt bar. Append to a mounted prompt-mode bar, or mount the home bar with the
     text, reusing the restore-loading helpers. Toast an error saying the draft is back
     in the bar.
4. In `src/sase/ace/tui/_app_action_availability.py::check_app_action`, return `False`
   for `restore_prompt_stash` while `_prompt_input_owns_keys(app)`, mirroring the
   `next_tab`/`prev_tab` guard from `e54fceedd1`. Focus can transiently leave the
   prompt's VimTextArea, so a typed `@` (for example in `%m:@xlarge`) must never fire
   the global stash restore. Stash access from an open prompt remains available through
   the prompt-local `Ctrl+G p` and Ctrl+S-on-empty-pane paths. Apply the same guard to
   `quit` and `stop_axe_and_quit`, so a stray `q`/`Q` during focus loss cannot exit with
   a draft open. Check whether help text claims `@` works with an open prompt and update
   it.
5. Tests, extending the existing `tests/ace/tui/actions/test_prompt_stash_restore_*`
   modules and the availability tests:
   - the pop outcome drives loading;
   - a read failure does not pop;
   - a load failure rolls the row back into the stash, and the toast fires;
   - a task exception is logged and toasted;
   - a failed stash append puts the draft back into the bar;
   - `restore_prompt_stash`, `quit` and `stop_axe_and_quit` are unavailable while the
     prompt owns keys, and available otherwise.

Verification: `sase tool run check`. If rendered output changes, follow
`lint_and_test.md` for visual snapshots. Nothing here is expected to change them.

### Phase 5: Quitting the TUI stashes an open prompt draft

- **repo:** sase

1. Generalize `_stash_prompt_bar_before_restart` (`_prompt_bar_stash.py`) into a shared
   synchronous "stash mounted draft before exit" helper with a `source` argument (keep
   `restart` for the existing caller in `src/sase/ace/tui/actions/axe.py`).
2. In `src/sase/ace/tui/actions/lifecycle.py`, call it with `source="quit"` on the
   explicit quit paths before the controlled exit (`action_quit`, and the
   confirm-then-exit path, `_begin_controlled_exit` / `_request_controlled_exit`) and in
   `stop_axe_and_quit`. Then:
   - If a non-empty prompt-mode draft exists and the stash write fails, cancel the exit
     and toast an error. Never exit and drop the draft.
   - Do not stash from signal handlers. Agent- or screenshot-driven TUIs killed by tmux
     must not add junk rows to the real stash.
3. Add one line to the quit-confirm impact summary (`src/sase/ace/tui/quit_impact.py`)
   when a draft is open, for example "Your unsent prompt draft will be stashed".
4. Tests:
   - quit with a mounted non-empty prompt bar appends a `source="quit"` stash row with
     the full panes and frontmatter;
   - quit with an empty bar appends nothing;
   - a failing append cancels the exit and toasts;
   - the restart path still uses `source="restart"`.

Verification: `sase tool run check`.

### Phase 6: Stash-archive recovery surface (CLI, TUI hints, docs) and core pin

- **repo:** sase

1. Move sase's `sase-core-revision.txt` pin past the `core-stash-archive` commit, and
   apply any compatibility-window or wheel steps `docs/rust_backend.md` requires for new
   bindings.
2. In `src/sase/core/prompt_stash_facade.py` and `src/sase/core/prompt_stash_wire.py`,
   add `read_prompt_stash_archive(path, limit)` and
   `recover_prompt_stash_archive(path, ids)`, with wire rehydration. Guard both with the
   `guard-prompt-stores` helper. Add `prompt_stash_archive_path()` to
   `src/sase/core/paths.py` if Python needs the path; otherwise let Rust derive it from
   the stash path.
3. Add the CLI group `sase prompt stash-archive` under the existing `sase prompt` parser
   (`src/sase/main/parser_prompt.py` plus its handler module). Read `cli_rules.md`
   first. Subcommands, in alphabetical order:
   - `list` is the default via the central `_default_list_subcommands()`. Options:
     `-j/--json`, `-n/--limit N` (default 20), `-q/--query TEXT` (case-insensitive
     substring over text and frontmatter),
     `-r/--reason {evicted,overwritten,popped,purged}`. It prints a colored table:
     archived time, reason, short id, project, first line.
   - `restore IDS...` resolves unique id prefixes, calls `recover_prompt_stash_archive`,
     and reports restored and skipped ids with reasons.
   - `show ID` (`-j/--json`) prints the full archived entry.

   Help text must be excellent and explain that the archive keeps every draft that ever
   left the stash (restored, deleted, purged, evicted, or overwritten).

4. TUI hints, with no new keys: in `_prompt_bar_stash_restore_trash.py` and
   `_prompt_bar_stash_restore_overlay.py`, make the purge confirmation, the purge
   success toast, and the trash-limit eviction toast mention that drafts stay
   recoverable with `sase prompt stash-archive`. The in-place delete toast in
   `_prompt_bar_stash_restore_apply.py` does too.
5. Docs: update the Prompts overlay, stash and Trash documentation (`docs/ace.md` and
   any stash page found with `rg -n "Trash" docs`) to describe the archive and the
   recovery command.
6. Tests:
   - CLI tests: list, filter, JSON shape, show, restore round-trip, prefix ambiguity and
     skipped ids, using tmp stores only;
   - facade guard coverage for the two new functions;
   - an end-to-end test: pop a row through the TUI restore path, then recover it with
     the CLI.

Verification: `sase tool run check`.

## Out of scope and follow-up candidates

These are for the implementing agents to record as `PROPOSED FOLLOW-UP:` notes on their
phase beads, not to build here:

- A crash-safe, per-session prompt-draft journal, periodically autosaved and imported
  into Stash as `source="recovered"` on the next start. It would cover SIGKILL, OOM and
  terminal close with a draft open.
- Recording short (<5 word) and frontmatter-only drafts as cancelled history; today
  `prompt_store.py` drops them silently.
- Error toasts when a prompt-history save returns `False`.
- Keeping the `$EDITOR` temp file when the editor exits non-zero.
- Retention or rotation for `prompt_stash_archive.jsonl`. It is append-only and small
  per row; revisit only if growth is measurable.
- Deleting the unreachable standalone `StashedPromptsModal` and its permanent-delete
  path.
