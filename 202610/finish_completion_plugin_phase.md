---
tier: tale
title: Finish phase sase-1if.4 (plugin subtrees in completion) and close it
goal:
  The landed completion phase passes the TUI tests and its own Symvision findings, its
  verification check is clean apart from the KNOWN base backlog, and bead sase-1if.4 is
  closed.
size: small
proposed_by: bbugyi200.apollo.5y
create_time: 2026-10-08 19:05:34
status: wip
---

# Plan: Finish phase sase-1if.4 (plugin subtrees in completion) and close it

## Context

Phase bead `sase-1if.4` belongs to epic `sase-1if` (`plan:202610/plugin_commands.md`,
section "Phase: completion"). Its implementation already landed on master in commit
`7922974062` ("feat(completion): merge plugin parsers into runtime spec with
plugin-aware cache identity"):

- `src/sase/completion/plugin_runtime.py` (`build_runtime_spec()`, omissions);
- the plugin walker mode in `src/sase/completion/build.py`;
- the plugin-aware `runtime_identity()` / `source_fingerprint()` and
  `CACHE_FORMAT_REVISION = 2`;
- manifest omissions and the `completion.plugins` doctor check;
- the TUI key recheck in `src/sase/ace/tui/command_line/grammar.py`;
- `docs/completion.md` and `tests/completion/test_plugin_runtime.py`.

An audit against every bullet of the phase scope found nothing missing.

The bead is still open because its verification `sase tool run check` stopped at the
first stage (`fmt (markdown)` on `docs/completion.md`). The lint and test stages after
it never ran. Re-verified on current master:

- prettier, ruff check and format, mypy (`src`), pyscripts, test-waits, changelog, and
  terminology all pass.
- 683 completion, doctor, plugin-command, and TUI command-line tests pass.

That re-verification found the three problems below. This plan fixes them, verifies, and
closes the bead.

### Problem 1: a deterministic TUI regression

`tests/ace/tui/command_line/test_completion_fixes.py::test_tab_through_every_subcommand_reaches_the_last_with_a_highlight`
fails every time (`assert state.index == count - 1` → `17 == 32`). It passes against the
sources of `7922974062^`.

The cause is the new recheck path:

- `ensure_command_line_grammar_loaded` now starts `_recheck_grammar_key` on every `:`
  open once a handle exists.
- Five TUI test harnesses inject `app._command_line_grammar` directly, without a
  recorded key: `_panel` in that file, plus `test_chrome_layout.py`,
  `test_transcript_scroll.py`, `test_policy_io.py`, and `_completion_sources_shared.py`.
- The recheck compares the freshly computed key with `None`. It treats the mismatch as
  "changed", runs the real `sase completion spec -d -j` subprocess, and swaps the
  injected handle for the live one.
- It then fires `_on_grammar_ready_from_worker`, which refreshes the popup partway
  through the Tab walk.

The same over-eager reload happens in production whenever the first load's key capture
failed. The phase rule is "reload only if the key **changed**"; with no recorded key
there is nothing to compare against.

### Problem 2: three phase-owned Symvision findings

Diffing `symvision src/sase` between `7922974062^` and `7922974062` shows exactly three
new unused-public findings:

- `RuntimeCompletionSpec` in `src/sase/completion/plugin_runtime.py`, used only in its
  own file.
- `command_line_grammar_spec_key_for` in `src/sase/ace/tui/command_line/grammar.py`,
  used only in its own file plus tests.
- `current_structural_view` in `src/sase/completion/snapshot.py`. Its only non-test
  `src` consumer was `_handle_completion_spec`, which now uses the runtime spec. Its
  real remaining consumer is the extensionless script `tools/sync_completion_spec`,
  which Symvision cannot see.

The other ~48 Symvision findings reproduce byte-identically on a clean base. They are
owned by `sase-1i5.9.1.2.1.5`, and the `ToolRun` ledger has recorded them since
2026-10-08 03:55 across 27 agents. Sibling phases `sase-1if.1` and `sase-1if.3` closed
over them. Do **not** touch them.

### Problem 3: a load-sensitive CPU-budget flake

`tests/main/test_completion_candidates_contract.py::test_candidates_fast_path_child_cpu_budget[snippet]`
failed once at load average ~26 and passed in isolation. The candidates fast path does
not touch this phase's plugin scan. Treat it as a load flake, not phase work.

## Changes

### 1. Recheck adopts a baseline key instead of reloading (`grammar.py`)

In `src/sase/ace/tui/command_line/grammar.py`:

- In `_recheck_grammar_key`, after computing `key`, handle a missing recorded key. When
  the app has no recorded key:
  - store `key` as the baseline (`_COMMAND_LINE_GRAMMAR_KEY_ATTR`);
  - drain the queued callbacks without invoking them, exactly like the unchanged-key
    branch (`_take_grammar_callbacks(app)`);
  - return without reloading.
- Only a recorded key that differs from the computed key triggers a reload. Keep the
  existing `finally` that clears the recheck flag.
- Rename `command_line_grammar_spec_key_for` to `_command_line_grammar_spec_key_for` and
  remove it from `__all__`. Update its in-file caller.
- Add one clause to the `ensure_command_line_grammar_loaded` and `_recheck_grammar_key`
  docstrings: an unkeyed handle adopts the current key as its baseline.
- Add no new imports. The TUI app import budget is ratcheted.

### 2. Make the runtime-spec carrier private (`plugin_runtime.py`)

In `src/sase/completion/plugin_runtime.py`:

- Rename `RuntimeCompletionSpec` to `_RuntimeCompletionSpec` and drop it from `__all__`.
- Update the `build_runtime_spec()` return annotation and constructor call.
- Callers only use `.spec`, `.omissions` and `.structural_view()`. Grep `src/` and
  `tests/` to confirm nothing imports the old name.

### 3. Pragma for the real `tools/` consumer (`snapshot.py`)

In `src/sase/completion/snapshot.py`, add `# symvision: tools/sync_completion_spec`
directly above `def current_structural_view`. This mirrors the existing pragma on
`completion_spec_drift` in the same file. Do not delete the function: the snapshot tool
and the builtin-snapshot tests still need it, and `build_spec()` must stay builtin-only.

### 4. Tests (`tests/completion/test_plugin_runtime.py`)

- Point the two `grammar_module.command_line_grammar_spec_key_for(app)` assertions at
  the private name.
- Add `test_tui_grammar_recheck_adopts_key_for_unkeyed_handle`. Reuse the file's
  `_fake_app` helper, with `current_command_line_spec_key` returning `"key-1"` and then
  `"key-2"`, and `_load_command_line_grammar_sync` recording calls. Inject a handle
  object directly, with no key, then assert:
  - `ensure_command_line_grammar_loaded(app, on_ready=...)` returns `True`;
  - no sync load ran, and the handle is the same object;
  - `_command_line_grammar_spec_key_for(app) == "key-1"`;
  - the `on_ready` callback was not invoked;
  - the grammar is not pending.
  - A second call with the key now `"key-2"` reloads exactly once and records `"key-2"`,
    which proves baseline adoption does not disable freshness.
- Keep the new test compact. Do not change the five TUI harnesses: the product fix
  restores their hermetic, no-subprocess behavior.

### 5. Docs (`docs/completion.md`)

In the "Plugin Commands" freshness paragraph, add one sentence: an already-running TUI
rechecks the key off-thread each time the `:` command line opens, and reloads the
grammar only when the key changed. Then run prettier on the file (`just fmt` covers it).
Prettier failing on this same file is what stalled the bead the first time.

## Verification

1. Run the targeted tests:

   ```bash
   .venv/bin/python -m pytest -q tests/completion/test_plugin_runtime.py \
     tests/ace/tui/command_line/test_completion_fixes.py tests/ace/tui/command_line \
     tests/main/test_completion_handler.py tests/completion/test_snapshot.py
   ```

   `test_tab_through_every_subcommand_reaches_the_last_with_a_highlight` must pass.

2. Run `just fix` (format plus keep-sorted).

3. Run `just _lint-symvision` and confirm that none of `RuntimeCompletionSpec`,
   `command_line_grammar_spec_key_for` or `current_structural_view` is listed. The
   remaining base backlog is expected.

4. Run `sase tool run check` **in the foreground**, with a generous tool timeout (it has
   taken 16–40 minutes on this host).
   - The expected verdict is `no_new_failures`. The run still exits 1, because the KNOWN
     base Symvision backlog stays red, and triage never changes exit codes.
   - Any NEW or UNKNOWN item is in scope: fix it and rerun.
   - If only the CPU-budget test above fails, rerun that test in isolation to confirm it
     is a flake before treating it as one.

## Close the bead

After verification is clean (only KNOWN or FLAKY items):

- If this run's assigned bead is `sase-1if.4`, set `bead_action: "close"` on the primary
  repository decision in `/sase_final`.
- Otherwise, run
  `sase bead close sase-1if.4 --note "<what was fixed and the check ToolRun id + verdict>"`
  before submitting `/sase_final` for the commit.

Never close the parent epic `sase-1if`; its land agent does that. Do not create task
beads for the base Symvision backlog, which `sase-1i5.9.1.2.1.5` already owns. No
sase-core (Rust) change and no `sase-core-revision.txt` bump are needed.
