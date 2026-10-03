---
tier: tale
title: Finish and land the non-TUI macro syntax cutover (sase-1eq.4.1)
goal:
  Repair every regression and stale contract the sase-1eq.4.1 macro cutover left on
  master, then close sase-1eq.4.1 and its parent phase sase-1eq.4.
size: medium
proposed_by: bbugyi200.athena.sase-1eq.4.1.land
bead: sase-1eq.4.1
create_time: 2026-10-03 12:26:56
status: wip
---

- **PARENT:**
  [202610/macro_syntax_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_syntax_cutover.md)
- **BEAD:**
  [sase-1eq.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.4.1.md)

# Plan: Finish and land the non-TUI macro syntax cutover

## Context

Epic `sase-1eq.4.1` ("Complete the non-TUI macro syntax cutover", plan
`plan:202610/macro_syntax_cutover.md`) has all five phases closed (`sase-1eq.4.1.1`
through `.5`, commits `3c1f5c313e`, `6eaa8df521`, `6d0d8a0a2d`, `4f90695659`,
`29c14710fb`). Its land agent verified the work and found that the phases left master
red. Each phase worker dismissed failures as "reproduces on clean base", but that base
already contained the earlier phases' commits. The land agent recorded the findings on
the epic bead (notes `LAND VERIFICATION` and `LAND FOLLOW-UP TRIAGE`); read them with
`sase bead read sase-1eq.4.1 -r "<why>"` before you start.

Evidence: `sase tool run check` ToolRun `cb133f344288ad1e9dada5423ff1f722` (escalated
full lane: 28 failed) compared with the same tests on the pre-epic base `e847b082c2`
(only 4 failed there). Master Gate run `37134560139` shows the same failures. A
check-only visual run (`just test-visual -- <macro-related visual files>`) found 21
drifted goldens. 8 of them drift only after this epic. The other 13 or so were already
stale from the `sase-1eq.3` rename.

Everything below is caused by this epic, or is integration with what it added. Fix it
here. Read `tui.md` (via `/sase_memory_read`) and `src/sase/ace/AGENTS.md` (or the TUI's
local `AGENTS.md`) before touching TUI files. TUI work stays limited to behavior, path
literals, and layout adapters. Widget/module renames, keymap migration, and general TUI
terminology stay with `sase-1eq.5`. Do not touch docs (owned by `sase-1eq.6`), plugin
repos, chezmoi, or memory files. Keep permanent durable readers (`legacy_xprompt_names`,
the `%xprompts_enabled` parser) and realistic legacy-input fixtures intact. Never
rewrite old-input evidence just to make an assertion pass.

## 1. Product regressions

1. **TUI Admin Center XPrompts load is dead.**
   `src/sase/ace/tui/modals/xprompt_browser_actions.py` `action_edit_xprompt` still
   gates on `item.kind != "xprompt"`. Since `4f90695659`,
   `sase.macro.reference_display.workflow_kind_value` returns `"macro"` for simple
   definitions, so Enter/Ctrl+I always warns "Workflow graphs use E / $EDITOR". Compare
   against the canonical value (prefer one exported constant next to
   `workflow_kind_value` over a new literal). Update the `BrowserItem.kind` comment in
   `xprompt_browser_helpers.py`. Make the harmless `("", "xprompt")` default in
   `xprompt_select_modal._create_styled_label` canonical too. Then grep every consumer
   of `workflow_kind_value` / catalog `kind` for remaining `"xprompt"` comparisons.
   Leave TUI-internal completion kinds (`_completion_kind`, Rust token kinds,
   `MiniXPromptWorkflowKind`, etc.) alone; those belong to `sase-1eq.5`/`sase-1eq.10`.
   This fixes
   `tests/ace/tui/test_xprompt_browser_load_keymap.py::test_enter_loads_raw_definition_and_binds_source`
   and `::test_enter_returns_while_xprompt_file_read_is_blocked`.
2. **TUI save destinations are duplicated.**
   `src/sase/ace/tui/modals/unified_xprompt_save_support.py`
   `_with_missing_standard_directories` still adds
   `resolve_project_layout(...).xprompts.write_path` and the home/chezmoi `.xprompts`
   write paths. `xprompt_location_modal.get_all_xprompt_locations` already emits
   `.macros.write_path`, so the mini-xprompt location picker now shows both
   `./sase/macros` and `./sase/xprompts` (and both `~/sase/...` dirs) with duplicate `p`
   and `h` hotkeys, and offers a retired directory for new content. Use the
   `.macros.write_path` layouts so new content only targets canonical paths. Add a unit
   test asserting exactly one project and one home directory row with unique hotkeys.
3. **TUI browser misclassifies canonical definitions.**
   `src/sase/ace/tui/modals/xprompt_browser_helpers.py` `classify_source` walks
   `project_layout.xprompts.candidates` and `home_layout.xprompts.candidates`, so files
   under `sase/macros/` and `~/sase/macros/` (including the tracked
   `sase/macros/reads.md`/`sync.md` moved by this epic) land in "Other". Classify with
   the `macros` layout: `macros.write_path` is the canonical project/home category. The
   Rust layout's `macros.candidates` (`.xprompts`, `xprompts`) plus the retired
   `xprompts.write_path` (`sase/xprompts/`) are the legacy categories. Keep the
   project-home subdirectory branch working for `~/sase/macros/<project>/`. The
   canonical category labels embed path literals
   (`XPROMPT_PROJECT_DIR_LABEL = "Project sase/xprompts/"`,
   `XPROMPT_HOME_DIR_LABEL = "Home ~/sase/xprompts/"` in `xprompt_location_modal.py`).
   Change their path text to `sase/macros/` and `~/sase/macros/`. Replace the duplicated
   literal copies with the constants (`xprompt_browser_catalog.group_browser_items`
   known order, `unified_xprompt_save_support._precedence` and
   `_with_missing_standard_directories`, any others found by grep). Do not rename the
   constants or modules.
4. **Run-policy table names a command the spec no longer has.**
   `src/sase/completion/run_policy.py` `_STDIN_PATHS` still contains
   `("xprompt", "expand")`. Remove it (fixes
   `tests/completion/test_spec_contract.py::test_run_policy_tables_match_live_parser`).
5. **mypy.** `src/sase/doctor/checks_config_retired.py` `_scan_plugin_groups` uses the
   pre-3.10 `entry_points().get(...)` branch (mypy `attr-defined`). Use
   `importlib.metadata.entry_points(group=RETIRED_PLUGIN_GROUP)` directly. Keep the
   no-traceback behavior.
6. **Symvision.** `discover_macro_plugin_entry_points` in
   `src/sase/main/plugin_discovery.py` has no non-test consumer. Make it private (it is
   only used by `discover_macro_plugin_modules` in the same file) and update
   `tests/macro/test_macro_discovery_policy.py`. Do not add an `--epic-symbol` entry.
7. **Legacy CLI spellings still show in help.** The plan required hidden legacy commands
   and targets, normalized in the compatibility home. Today `sase --full-help` lists
   `macro (xprompt)` and includes `xprompt` in the root choices, and `sase path --help`
   advertises `xprompts-dir`, `xprompts-schema`, and `xprompts-collection-schema`.
   Implement root-position normalization:
   - Add a helper to `src/sase/legacy_xprompt_syntax.py` that takes argv (after
     `consume_global_options`). It locates the root command with
     `sase.main.parser_root_args.root_command_index`, the same way `parser_only_hint`
     does. It rewrites a root `xprompt` to `macro`, and rewrites the first positional
     after a root `path` from `xprompts-dir|xprompts-schema|xprompts-collection-schema`
     to the matching `macros-*` target. With `legacy_xprompt_syntax` off it prints
     `<old> is retired; use <new>` to stderr and exits 2, matching today's messages.
     Never rewrite any other token (for example `sase run "xprompt ..."` or
     `sase macro show xprompt`).
   - Call it in `src/sase/main/entry.py` after the global-option block and before the
     fast paths and `create_parser(only=parser_only_hint(sys.argv))`, so -F/-f flag
     overrides already apply and the narrow parser sees `macro`.
   - Remove `aliases=["xprompt"]`, `set_completion_compat_aliases(..., "xprompt")`, and
     the `xprompt_subcommand` narrow-parser compat from `main/parser_macro.py` and
     `main/macro_handler.py`. Also remove the `"xprompt"` registrar from
     `main/parser_registry.py` (and `parser_full_registrars.py` if present), the
     `(("xprompt", "show"), "name")` override in `completion/kinds.py`, the legacy
     `path` choices plus `set_completion_compat_choices` in
     `main/parser_commands.py:register_path_parser`, and the dead legacy branches in
     `entry.py` (`args.command in {"macro", "xprompt"}` and the `xprompts-*` path arms).
     Keep `_LEGACY_KIND_ALIASES` in `completion/candidates/providers.py`: stale
     installed shell scripts still call `completion candidates xprompt`. Keep the
     editor/mobile `xprompt-catalog` helper selector as-is (already hidden).
   - Update the roughly 8 tests that parse `["xprompt", ...]` or use
     `create_parser(only="xprompt")` (`tests/main/test_parser_macro_show.py`,
     `test_macro_handler.py`, `test_parser_command_defaults.py`, grep for the rest) to
     exercise normalization instead. Add both-flag-state tests:
     - flag on: `sase xprompt list`/`show`, `sase -F <other> xprompt ...`, and
       `sase path xprompts-dir` behave like the canonical forms, and
       `sase xprompt list --help` prints `usage: sase macro list`.
     - flag off: exit 2 with the retirement message.
     - `--full-help` and `sase path --help` contain no retired spelling.
     - a prompt argument containing `xprompt` is untouched.
   - Run `just sync-completion-spec`. `tests/completion/snapshots/cli_spec.json` should
     not change; inspect any diff.

## 2. Stale test expectations

Update each one to the canonical behavior the epic intentionally shipped. Keep any
legacy authored input that tests a reader:

- `tests/prompt_command/test_export_save.py` (7 tests): saves now go to `sase/macros/`,
  `~/sase/macros/`, and project-specific `~/sase/macros/<project>/` (stdout already
  shows the canonical paths).
- `%macros_enabled` writer output: `tests/gate_turn/test_followup_prompt.py` (2),
  `tests/question_gate_turn/test_followup_prompt.py` (1),
  `tests/test_axe_run_agent_exec_plan_followup_questions.py` (1),
  `tests/test_fork_workflow.py` (5). In
  `test_deferred_launch_ignores_bare_fork_prose_inside_disabled_region` the authored
  prompt deliberately keeps `%xprompts_enabled`, a legacy reader input. Assert one
  writer-emitted `%macros_enabled:false` plus the preserved authored legacy region; do
  not rewrite the input.
- `tests/test_macro_skill_sources.py::test_non_skill_in_a_skill_source_is_rejected`: the
  placement hint now names `home/sase/macros`.
- `tests/test_macro_swarm_local_helpers.py::test_checked_in_reads_macro_uses_direct_local_helper`:
  the file moved to `sase/macros/reads.md`.
- `tests/main/test_parser_root_help.py::test_root_help_renders_compact_help`: "prompt,
  macro, workflow, or history."
- `tests/macro/test_cli_show_resolve.py::test_workflow_wins_over_shadowed_macro`:
  "shadows macro".
- `tests/ace/tui/widgets/test_xprompt_arg_assist.py::test_assist_adapter_preserves_structured_catalog_fields`:
  kind `"macro"`.
- `tests/completion/test_bash_smoke.py` `_write_marker_fixture_sase`: the fake `sase`
  must answer the `macro` kind, since `ValueKind.MACRO == "macro"`. Rename the fixture
  candidate to `zzz-fixture-macro` and update the parametrized expectation.
- `tests/completion/test_candidates_providers.py::test_snippet_candidates_use_rust_loader`:
  the fake `load_editor_snippet_catalog` must accept the third
  `accept_legacy_xprompt_names` argument (added by `6d0d8a0a2d`). Assert it receives the
  current flag value.

## 3. Integration

- The CI-only parity suites (`tests/test_macro_directive_completion_parity.py`,
  `test_macro_model_alias_shortcut_parity.py`,
  `test_macro_finalizer_completion_parity.py`, `test_macro_jinja_lsp_parity.py`,
  `tests/macro/test_argument_surface_parity.py`) fail on CI with "sase-xprompt-lsp
  binary is missing". `tests/_macro_directive_completion_parity_lsp_session.py`
  hard-codes `Path(sys.executable).with_name("sase-xprompt-lsp")`, but CI installs only
  `sase-macro-lsp`. Follow the epic's canonical-first binary policy: prefer
  `sase-macro-lsp` next to the interpreter and fall back to the legacy name. Fail with a
  message naming the canonical binary. Remove the now-unneeded classified exception pair
  for that line from `tests/_macro_terminology_string_pairs_a.py`, and keep
  `tests/test_macro_terminology.py` green.
- PNG goldens: after the TUI fixes, refresh the macro-related visual files with
  `just fix-tui-screenshots -- <files>`, where `<files>` is every
  `tests/ace/tui/visual/test_*.py` / `tests/pager/visual/test_*.py` matching
  `xprompt|macro` (in zsh, build the list as an array).
  - Drift was seen in: `agents_auto_approve_workflow_child_alignment`,
    `agents_tribe_panel_prompts_{glance,inspect}`,
    `config_center_xprompts_{tab,filter}`,
    `frontmatter_panel_{cell_edit,empty,ghost_row,populated,saved_feedback}`,
    `jump_action_modal`,
    `mini_xprompt_{location_flow_picker,save_diff,scoped_frontmatter}`,
    `preview_panel_{long_markdown,xprompt}`, `save_location_picker_xprompt`,
    `snippet_name_collision`, `xprompt_save_{collision_armed_diff,create}` (all
    `_120x40`).
  - Inspect every update group in the report. Accept only drift explained by canonical
    macro spellings, paths, or kind labels, or by this tale's fixes (for example, the
    location picker losing its duplicate rows).
  - The plugin install/uninstall preview goldens drift intermittently and are not
    related to this work; do not accept them.
  - The `mini_xprompt_location_flow_picker` golden already embeds host `~/.config/sase`
    paths; that non-hermeticity predates this work, so do not try to fix it here.

## 4. Verification

- Run `just fix`, the focused tests for every file above, and the new tests. Then run
  `sase tool run check` (through `/sase_monitor` if it may exceed your synchronous
  limit). Do not run `just check-full`.
- These pre-existing failures do not belong to this work and must not block the close:
  - `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`,
    plus symvision `PublicationPayloadFile`/`plan_publication_payload_batches`. Owned by
    active epic `sase-1ex` (commit `5c7e7514ae`).
  - `tests/doctor/test_checks_beads.py::test_project_beads_skips_when_store_is_absent`
    (task `sase-14o`).
  - `tests/test_vcs_macro_mru_pruning.py::test_load_launchable_prunes_provider_mismatched_prefix`
    (task `sase-172`).
  - The load flake
    `tests/ace/tui/test_launch_context_source.py::test_every_tick_rebroadcasts_to_mounted_views`.
  - The `toobig` violation in
    `tests/ace/tui/visual/test_ace_png_snapshots_memory_pane_history_states.py`.

  Any other failure is yours. Independently confirm that each remaining one is on this
  list.

- Live checks:
  - `sase macro list` emits `"type": "macro"`, and `sase path macros-dir`,
    `macros-schema`, and `macros-collection-schema` print existing files.
  - `sase doctor -C config.retired_xprompt_names` runs without a traceback with and
    without `-F legacy_xprompt_syntax`.
  - `sase -F legacy_xprompt_syntax xprompt list` and
    `sase -F legacy_xprompt_syntax path xprompts-dir` exit 2 with the retirement
    message.
  - `sase --full-help` and `sase path --help` show no retired spelling.

## 5. Close out epic sase-1eq.4.1 and parent phase sase-1eq.4 (final step)

Do this in the same turn as the code. Do not wait for, or order it after, this work's
own commit, push, or CI.

1. Run `sase bead epic-symbols sase-1eq.4.1`. For every listed `--epic-symbol` entry,
   either resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete
   it) or, only when a still-open later bead needs the exemption, re-key its Justfile
   line to that bead. The list was empty at landing review; make sure this tale added
   none.
2. Run
   `sase bead close sase-1eq.4.1 --note "<what you verified: the fixes above, focused tests, sase tool run check result with the remaining failures mapped to their owners, golden refresh inspected, CLI both-state checks>"`.
   Never use `--force` to make the close succeed. If the close is rejected for leftover
   epic-symbol entries, clean them up and close again.
3. Run `just symvision`. The epic whitelist must be clean. The only acceptable remaining
   items are the `sase-1ex`-owned
   `PublicationPayloadFile`/`plan_publication_payload_batches`.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path for
   `plan:202610/macro_syntax_cutover.md` shown by
   `sase bead read sase-1eq.4.1 -r "<why>"`; it currently says `status: wip`).
5. The parent `sase-1eq.4` is a phase bead of epic `sase-1eq`. Its scope (the
   `sase-syntax` phase: sunset flag plus macro spellings with flag-gated aliases for
   CLI, config/frontmatter keys, directories, plugin groups, env vars, doctor ids, and
   skill sources) is exactly what `sase-1eq.4.1` delivered.
   - Verify that, then run `sase bead epic-symbols sase-1eq.4` and resolve any entry as
     in step 1.
   - Close only the phase:
     `sase bead close sase-1eq.4 --note "<child plan sase-1eq.4.1 closed; all five sections (compatibility, config-frontmatter, discovery, cli-doctor, strings-guard) verified; evidence summary>"`.
   - Do not close `sase-1eq`; its own land agent is already waiting.
   - If the phase cannot be closed cleanly, record a blocker note on `sase-1eq.4` with
     `sase bead note` and report it instead of forcing.
