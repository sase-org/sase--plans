---
tier: tale
title: Finish and land epic sase-1df (Jinja2 variable completion)
goal:
  "The Jinja completion engine, the xprompt LSP, and the TUI prompt input agree on every
  in-tag position: no phantom closers or phantom tags from inert regions, correct
  statement and test documentation, and no non-Jinja completion surface claiming an
  in-tag cursor. The missing parity and regression tests exist, and epic sase-1df is
  closed with its plan marked done."
size: medium
proposed_by: bbugyi200.apollo.sase-1df.land
bead: sase-1df
create_time: 2026-09-30 15:31:46
status: wip
---

- **PARENT:**
  [202609/jinja_variable_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/jinja_variable_completion.md)
- **BEAD:**
  [sase-1df](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1df/README.md)

# Plan: Finish and land epic sase-1df (Jinja2 variable completion)

## Why

Epic `sase-1df` ("Jinja2 variable completion in the prompt input and the xprompt LSP",
plan `plan:202609/jinja_variable_completion.md`) has all nine phases closed. Its land
agent then checked every phase against the plan and the source. The LSP phase, the
bindings, the adapter, the lint, input inference, the menu redesign, and the auto-open
setting are all in place, and the targeted suites pass (65 Rust jinja unit tests, the
`sase_xprompt_lsp` suite, and 116 Python tests).

The land check did find epic-caused defects and missing coverage, listed below. Each one
is reproducible on the current tree. This tale fixes them and then closes the epic.

Nothing needs a new design. Every item fixes behavior that the epic plan already
specifies. Keep this work inside that plan's design principles, especially "one engine,
every surface" and "inside a tag, Jinja owns completion".

## Part A: sase-core engine fixes

Open the linked core with `sase repo open sase-core -r "<why>"` and work only in the
printed path. Read its `AGENTS.md` first. The engine lives in
`crates/sase_core/src/editor/jinja/`, and the LSP lives in `crates/sase_xprompt_lsp/`.

### A1. Phantom `endraw` closer

`scan.rs` `scan_document` handles a closed `{% raw %}…{% endraw %}` block through
`find_raw_end` and then jumps past the `{% endraw %}` tag without recording it as a
statement tag. Meanwhile, `scope.rs` `jinja_template_scope` pushes a `raw` frame for the
opener, and nothing ever pops it.

- **Repro:** with prompt scope, `{% raw %}x{% endraw %} {% ` at the end gives the
  statement items `['endraw', 'if', 'for', …]`.
- **Fix:** a closed raw block must leave no open frame. For example, record the `endraw`
  statement tag, or skip the frame when the raw body is closed. An unclosed `{% raw %}`
  should still suggest `endraw`.
- **Tests:** add scope and assist tests for both the closed and the unclosed case.

### A2. Tag scanning must respect inert literal zones

`jinja_tag_at_cursor` checks only the cursor against the frontmatter and
`jinja_inert_literal_zones`. `scan_document` still enumerates tags inside fenced code,
inline code spans, `%xprompts_enabled:false` zones, and the leading frontmatter.
`jinja_template_scope` and the statement stack consume those phantom tags.

- **Repro 1:** ``Use `{{` to open. hello`` with the cursor at the end returns
  `Some(slot=variable)`. It must return `None`, because plain text is not inside a tag.
- **Repro 2:** a fenced block containing `{% for a in b %}`, followed after the fence by
  `{% `, offers `endfor`/`else` first. It also leaks `loop` and `a` as locals.
- **Fix:** make the scan ignore openers that start inside those zones, so that every
  consumer agrees: `jinja_tag_at_cursor`, `scan_jinja_tags`, `scan_statement_tags`, and
  scope analysis. Reuse the existing `exclusion.rs` helpers; do not duplicate them. The
  `%{`/`{%{`/`{%(`/`{%alt(` alternation carve-outs from `next_jinja_tag` must keep
  working.
- **Tests:** add tests for both repros, plus inline code and `%xprompts_enabled:false`.

### A3. Statement documentation renders `{%% … %%}`

`docs.rs` `statement_markdown` uses `format!("{{%% {name} %%}}")`, and Rust has no `%%`
escape. Every statement item's Example and `Closes` bullet therefore renders as
`` `{%% endfor %%}` ``.

- **Fix:** use `format!("{{% {name} %}}")`, and fix the `Closes` bullet the same way.
- **Tests:** add a docs test that asserts `` `{% endfor %}` `` and
  ``Closes `{% for %}` ``. Grep the LSP tests for any assertion that encodes the broken
  string.

### A4. A shadowed builtin must not be both known and unavailable

In `scope_vars.rs` `jinja_scope_variables`, a declared input that has the same name as a
run-time builtin appears in both `known` and `unavailable`. For example, an
input-declaring prompt that declares `n: int` does this.

- **Fix:** the shadowing rule says the input wins, so a name that is a declared input
  (or otherwise known) must not be listed in `unavailable`.
- **Tests:** add a test.

### A5. Wrong test summaries in the catalog

In `catalog.rs`, `gt` and `greaterthan` are summarized as "Alias of `ge`", and `lt` and
`lessthan` as "Alias of `le`". Change them to:

- `gt`: "True when greater than."
- `greaterthan`: "Alias of `gt`."
- `lt`: "True when less than."
- `lessthan`: "Alias of `lt`."

Scan the remaining comparison aliases (`eq`/`equalto`/`==`-style names, `ne`, `ge`,
`le`) for the same mistake.

### A6. Test hardening

- **`assist.rs` legacy-ordering test (around line 1310):** it is wrapped in `if let`, so
  it passes when either name is missing. Assert that both `patch_name` and `cl_name` are
  present, and that `cl_name` sorts directly after `patch_name`.
- **`crates/sase_xprompt_lsp/src/server/tests/jinja.rs` deprecated-alias check (around
  lines 372-383):** it uses cursor offset 7 in `{{ cl_ }}` and is wrapped in `if let`.
  Use offset 6 (the end of `cl_`), and assert that the item exists and carries the
  `DEPRECATED` tag.
- **The same file, around line 111:** `documentation.value.contains('n')` asserts almost
  nothing. Assert the `%repeat` hint text instead.

Run `sase tool run check` in sase-core. Its clippy gate is already red on the clean base
tree: 9 pre-existing denies in untouched `sase_core` files, tracked by task `sase-1an`.
Confirm that the jinja module and `sase_xprompt_lsp` add no new clippy findings, and
that the jinja, bindings (`sase_core_py` `editor_completion`), and LSP tests pass. Use
the recipes `AGENTS.md` prescribes rather than bare cargo where it says so.

## Part B: sase fixes

Rebuild the local core first, per `docs/rust_backend.md`, so that the Python tests see
the Part A fixes. The turn's declaration commits both repos, and the host moves
`sase-core-revision.txt` past the new sase-core commit automatically.

### B1. Soft (ghost-text) completion ignores Jinja precedence

In `src/sase/ace/tui/widgets/prompt_completion.py` (around lines 264-280), when
`build_jinja_completion_result(...)` returns a result, the cursor is inside a tag. But
when no Jinja candidate changes the text, the code falls through to the xprompt-arg,
directive, and file branches.

- **Repro:** `{{ foo %mo` and `{% set x = "%mod` both soft-suggest `%model`.
- **Fix:** when the Jinja result is non-`None`, return its suggestion or `None`, and
  never fall through.
- **Tests:** add a unit test for both repros.

### B2. Auto-open precedence breaks when `auto_jinja_menu` is off

`_open_auto_reference_completion_after_change` in
`src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` (around line 204) consults
Jinja only when `settings.auto_jinja_menu` is on. `_try_auto_jinja_completion` in
`_file_completion_open.py` also returns `False` when the setting is off.

- **Repro:** with the setting off, typing `%mo` inside `{{ x ` opens the directive menu.
- **Fix:** the setting must control only whether the Jinja menu opens. An in-tag cursor
  must still claim the auto path, opening no menu, so that the placeholder, directive,
  `@`, `#`, and other auto menus never open inside a tag. Ctrl+T must keep working with
  the setting off.
- **Tests:** add a pilot test with the setting off.

### B3. The next-word ghost appears inside Jinja tags

`_next_word_ghost_allowed` in `src/sase/ace/tui/widgets/_prompt_next_word.py` blocks the
history next-word ghost only while the Jinja panel is open. The ghost can still appear
at the end of a line inside an open tag.

- **Fix:** suppress the ghost whenever the cursor is inside a Jinja tag. Ask the engine
  through the existing adapter or widget helper with the pane scope; a non-`None`
  completion means the cursor is in a tag. Keep the path read-only, since `tui_perf.md`
  rule 11 applies.
- **Tests:** add a test.

### B4. Delete the legacy Python Jinja completion APIs

The epic plan says to delete these once nothing uses them, and nothing in `src/` does
anymore. Remove the following from `src/sase/xprompt/jinja_inspect.py`, along with their
re-exports in `src/sase/xprompt/__init__.py`:

- `JinjaCompletionContext`
- `builtin_runtime_member_names`
- `completion_context`
- the helpers that only they use (`_namespace_before_token`, `_name_char`)

In `tests/test_xprompt_jinja_inspect.py`, delete the tests that exercise these, or port
them to the engine adapter where the engine lacks equivalent coverage.

Then handle `known_toplevel_context` and `builtin_runtime_names`:

- If nothing in `src/` uses them besides the exports, delete them. Keep
  `inspect_template(known=None)` defaulting to the engine's prompt-scope known set.
- Otherwise, fix their stale docstrings, which still say "kept for the TUI Jinja menu
  until it moves to the engine".

Keep `just symvision` free of new names.

### B5. Remove the duplicate scope helper

`src/sase/ace/tui/widgets/_jinja_diagnostics.py` has a module-level
`jinja_scope_for_text_area(bar, text_area)` that re-implements
`PromptInputBar.jinja_scope_for_text_area` (`_prompt_input_bar_frontmatter.py`).

- **Fix:** have the lint call the bar's method, falling back to prompt scope without
  frontmatter when the bar lacks it, the way `jinja_scope_for_editor` in
  `src/sase/ace/tui/widgets/jinja_completion.py` does.

### B6. Add the missing save-as-xprompt regression test

`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt.py` now infers inputs
against the xprompt-scope known set, but no test covers it.

- **Test:** add one next to the existing
  `tests/ace/tui/actions/test_prompt_save_xprompt*` tests. It should prove that `wait`,
  `patch_name`, and `n` are not inferred as inputs, while a genuinely unknown name is.

### B7. Add the TUI menu-rows parity pilot

The epic's parity phase required this pilot. `sase-1df.9` deferred it until the
engine-backed widget landed, and it never came back.

- **Test:** add one pilot that opens the Jinja menu in the real widget, for example with
  Ctrl+T after `{{ ` in a stack whose frontmatter declares inputs. Assert that the
  candidate names in the rendered menu, in order, equal the
  `jinja_assist.jinja_completion(...)` item order for the same text, cursor, and
  `PromptInputBar.jinja_scope_for_text_area` scope.
- Keep the file small, either a new file or an addition to
  `tests/ace/tui/widgets/test_prompt_jinja_menu.py`.

### B8. Cover run-time names in the parity suite

Every prompt fixture in `tests/test_xprompt_jinja_lsp_parity.py` declares inputs, and
declared inputs hide the run-time names. As a result, parity never covers `wait.`
members, `patch_name`, the conditional `needs %repeat`/`needs %wait` labels, or the
legacy `cl_name` label.

- **Fix:** add a prompt fixture with no inputs that uses `%repeat` and `%wait`, with
  cursors after `{{ `, `{{ pa`, `{{ cl`, and `wait.`. Assert the same parity fields the
  suite already asserts: ordered names, kinds, source/availability labels, and
  documentation.

## Verification

- **sase-core:** see the end of Part A.
- **sase:**
  1. Run `just fmt`, then `just check` (never `just check-full`).
  2. `just symvision` currently fails on two pre-existing private imports that the
     active epic `sase-1d5` owns: `_scanner_rules_version` in
     `src/sase/bead/attachments/audience.py`, and `_store_growth_lines` in
     `src/sase/bead/attachment_doctor.py`. Treat those as baseline. Confirm this work
     adds nothing new, and compare against the clean tree if in doubt.
  3. Run the targeted suites:
     - `tests/test_xprompt_jinja_catalog_parity.py`
     - `tests/test_xprompt_jinja_lsp_parity.py`
     - `tests/test_xprompt_jinja_inspect.py`
     - `tests/ace/tui/widgets/test_prompt_jinja.py`
     - `tests/ace/tui/widgets/test_prompt_jinja_menu.py`
     - `tests/ace/tui/widgets/test_prompt_jinja_auto_menu.py`
     - `tests/ace/tui/widgets/test_local_xprompt_conversion.py`
     - the save-xprompt tests
     - the next-word tests
     - `tests/test_config_schema_ace.py`
  4. Rerun the Jinja PNG goldens
     (`tests/ace/tui/visual/test_ace_png_snapshots_jinja_completion.py`) to confirm they
     are unchanged. If a golden legitimately changes (for example because of the A3 docs
     text), refresh it with `just fix-tui-screenshots -- <selectors>` through
     `/sase_monitor`, and inspect it.

## Final step: close out epic sase-1df

This tale is the landing of epic `sase-1df`. No other agent resumes the landing, so do
all of the following in this same turn, after the code above passes verification:

1. **Epic symbols.** Run `sase bead epic-symbols sase-1df`. For every listed
   `--epic-symbol` entry, either resolve the symbol (wire it up, privatize it, add a
   non-test pragma, or delete it per the Symvision epic-whitelist policy) or re-key its
   Justfile line to a still-open bead that needs it. The landing check found none, but
   recheck.
2. **Close the epic.** Run `sase bead close sase-1df --note "<verification>"`. The note
   should summarize:
   - all 9 phases were verified against the plan and the source;
   - which of this tale's fixes A1-A6 and B1-B8 landed, and the test evidence;
   - the integration check: sase-1co's `{%{`/`{%(`/`{%alt(` carve-outs (ff39548590) are
     inherited by the engine, the next-word ghost is now suppressed in tags, and the
     prompt-prediction validator passes at the current pin;
   - that follow-up triage is already recorded in the epic's notes.

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   epic symbols, finish that cleanup and close again.

3. **Symvision.** Run `just symvision` to confirm the whitelist is clean. Only the
   pre-existing `sase-1d5` private-import errors named above may remain.
4. **Plan status.** In the epic's plan file frontmatter, change `status: wip` to
   `status: done`. The file is `plan:202609/jinja_variable_completion.md`; edit it at
   the PLAN path printed by `sase bead read sase-1df -r "<why>"`.
5. **Parent.** `sase-1df` has no `parent_bead`, so nothing further is needed after the
   close.
