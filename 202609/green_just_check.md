---
tier: epic
title: Return just check to green and remove its recurring failure causes
goal: "A clean checkout of latest master passes `sase tool run check` and the full
  non-visual `just test` suite. The recurring causes found in the ToolRun ledger are
  removed: premature flag-bead closes, split-file agents that break symvision, stale
  `__pycache__`-only directories that fail pyscripts, and the uncached per-launch Rust
  LSP rebuild that pushes checks past agent command timeouts.

  "
phases:
  - id: lint-green
    title: Restore every lint gate except toobig on master
    depends_on: []
    size: medium
    description:
      "lint-green: re-derive master's lint failures stage by stage, including
      symvision's masked categories. Fix the mypy errors in the agent-detail mixins,
      launch-prompt inputs, and command-line input/screen. Replace the fixed sleep in
      the command-line completion test. Resolve the ~41 unused public symbols using the
      symvision decision hierarchy."
  - id: toobig-splits
    title: Split the two oversized ACE modules
    depends_on:
      - lint-green
    size: medium
    description:
      "toobig-splits: split command_line/screen.py and widgets/decks/panel.py under the
      toobig limit with purely mechanical moves. Import no `_private` names across
      modules, keep the public import paths stable, and keep symvision and mypy green."
  - id: tests-core
    title: Repair non-UI tests that fail on clean master
    depends_on:
      - lint-green
    size: medium
    description:
      "tests-core: fix the deterministic non-UI failures. These are the wait-check
      summary and scan-once tests (sase-186), the system-clock guard (sase-188 plus
      command-line block_render), the config-schema keymap, fakey help color, the
      agent-session rename fallout in kill-and-edit and launch approval, and the
      marker-mutation audit. Classify the load-sensitive tests by rerunning them."
  - id: tests-ace-ui
    title: Repair ACE TUI tests that fail on clean master
    depends_on:
      - lint-green
      - toobig-splits
    size: medium
    description:
      "tests-ace-ui: fix the ACE/TUI failures from the legacy-agents-UI removal and the
      deck cutover. This includes the prompt-panel monitor tests (sase-184), the
      completion-accept key move to ctrl-f, the command-line visual fixture host path,
      the TUI import budget, and the artifacts ref-prefix contract. Fix the code or
      update the tests of intentionally removed behavior."
  - id: pyscripts-stale-dirs
    title: Ignore cache-only script directories in the pyscripts lint
    depends_on: []
    size: xsmall
    description:
      "pyscripts-stale-dirs: make tools/pyscripts-260801 ignore scripts/ and tools/
      directories with no collectable files, such as a `__pycache__`-only leftover from
      a rename, so reused workspaces stop failing Rule 2. Add a regression test."
  - id: flag-close-guard
    title: Refuse closing a flag bead while its registry definition survives
    depends_on: []
    size: small
    description:
      "flag-close-guard: mirror the leftover --epic-symbol refusal in `sase bead close`.
      Refuse to close a flag task bead while the working tree's feature-flag registry
      still defines a flag naming it, because check_feature_flags rule 7 would redden
      every workspace."
  - id: split-file-prompt
    title: Make the split_file xprompt keep symvision and mypy green
    depends_on: []
    size: small
    description:
      "split-file-prompt: expand the built-in split_file xprompt that toobig routine
      agents run. It must forbid cross-module private imports, keep the public import
      paths, and require the split agent to run and fix symvision, mypy, and toobig
      before finishing. Close sase-180."
  - id: lsp-build-cache
    title: Cache sase-xprompt-lsp builds and dedupe concurrent core builds
    depends_on: []
    size: medium
    description:
      "lsp-build-cache: add a host-wide content-addressed cache for the sase-xprompt-lsp
      binary, keyed like the sase_core_rs wheel cache, and use it in rust-lsp-install.
      Add a per-identity build lock so concurrent workspaces build each new sase-core
      source identity once."
  - id: verify-green
    title: Verify green check and full test suite on clean master
    depends_on:
      - lint-green
      - toobig-splits
      - tests-core
      - tests-ace-ui
      - pyscripts-stale-dirs
      - flag-close-guard
      - split-file-prompt
      - lsp-build-cache
    size: small
    description:
      "verify-green: on a clean checkout of latest master, prove that `sase tool run
      check` exits 0 and that the full non-visual `just test` passes except for baseline
      flakes. Fix only small stragglers from concurrent landings and record the rest as
      follow-ups."
proposed_by: bbugyi200.athena.0rh
create_time: 2026-09-24 17:18:47
status: wip
---

- **PROMPT:**
  [prompts/202609/green_just_check.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/green_just_check.md)

# Plan: Return `just check` to green and remove its recurring failure causes

## Why

`just check` has been red for nearly every agent for days. The machine-local ToolRun
ledger (`sase tool runs -t check -n 1000 -j`, retained logs under
`~/.sase/tools/logs/<run>/`) recorded 572 `check` runs from 2026-09-20 07:26 to
2026-09-24 16:33. Only 55 succeeded. First failing stage of the rest:

| First failing stage                          | Runs | Nature                                                                                        |
| -------------------------------------------- | ---- | --------------------------------------------------------------------------------------------- |
| `lint (symvision)`                           | 226  | Mostly master-red: the same symbols were hit by 25–30 different agents                        |
| `lint (mypy)`                                | 85   | Mostly master-red (four distinct breakages, 09-20 → 09-24)                                    |
| `test (scoped)`                              | 44   | Mostly master-red nodes, each hit by 17–22 agents                                             |
| setup/timeout (no failing stage recorded)    | ~60  | Rust rebuild in `_setup`; 12 runs killed at ~539 s by the provider's command timeout, 22 lost |
| `fmt (python)`                               | 19   | Agents' own diffs                                                                             |
| `lint (pyscripts)`                           | 15   | 100% environmental: stale `tests/ace/tui/tools/__pycache__`                                   |
| `lint (feature flags)`                       | 12   | Flag beads closed before their registry definitions were removed                              |
| other (fmt md, toobig, validation, waits, …) | ~20  | Mixed                                                                                         |

Master has needed a manual "back to green" sweep roughly once a day: `86ff62c08` and
`47e281b7a` (09-20), `d3002aba1` (09-22), `c87c9d1aa` (09-23), and `064830632` (09-24
13:30). It turned red again within two hours of the last one. `just check` is fail-fast,
so one master-red stage hides every later stage. Epics then land new violations unseen.
For example, `742c1df38` added mypy errors while mypy was already red from `9bd351b67`.
Command-line phases also added about 40 unused public symbols while mypy hid symvision.

### State of master at `d5aa86dfc` (2026-09-24 16:31), clean tree

- `lint (mypy)`: 16 errors in 5 files.
- `lint (test waits)`: 1 site.
- `lint (symvision)`: 41 unused public symbols.
- `lint (toobig)`: 2 files over 1000 lines.
- Every other lint stage, `validate`, and `validate-committed-plans` pass.
- Full `just test`: 43 failed, 46,716 passed.

Each phase below lists its items.

### Causes this epic removes

1. **Current master-red items.** Phases `lint-green`, `toobig-splits`, `tests-core`, and
   `tests-ace-ui`.
2. **Split-file agents break symvision.** Toobig routine agents run the built-in
   `#split_file` xprompt (`src/sase/xprompts/split_file.md`), which is two sentences
   long. They repeatedly moved `_private` helpers into sibling modules and imported them
   across files. That happened six times (`sase-180`: sase-13l, sase-10n, sase-zk,
   sase-17c, sase-17j, sase-17l). Symvision matches private names by name, so every
   agent's check went red. Phase `split-file-prompt` fixes this.
3. **Premature flag-bead closes.** Phase sase-17d.3 closed flag bead `sase-17k`
   (`agent_decks`) at 11:41 on 09-24. The registry entry survived until the removal
   phase about three hours later, and `tools/check_feature_flags` rule 7 failed every
   workspace in between. The same thing happened with `sase-12m`/`service_host` on
   09-20. `sase bead close` already refuses leftover `--epic-symbol` entries
   (`src/sase/bead/epic_symbols.py`). Stale epic-symbol failures stopped after 09-20
   because of that guard. Flags have no equivalent guard. Phase `flag-close-guard` adds
   one.
4. **Stale cache-only directories.** `868abb246` renamed `tests/ace/tui/tools/`, but
   workspaces reused since then keep a `__pycache__`-only `tests/ace/tui/tools/`. Git
   does not delete it. `tools/pyscripts-260801` walks the filesystem, treats it as a
   placement target, and fails Rule 2. This accounts for all 15 pyscripts failures, 2 of
   them on clean trees. Phase `pyscripts-stale-dirs` fixes this.
5. **Uncached per-launch Rust LSP rebuild.** 89 of 504 sase `check` runs rebuilt
   `sase_core_rs` in `_setup`. 75 of them compiled `sase-xprompt-lsp` and all its crates
   from scratch (median 146 s, max 594 s). 49 of those did so even though the Python
   extension was a wheel-cache hit. Two things combine here. `clear_workspace_repos`
   deletes the launch's `sase/repos/` (the linked sase-core clone and its `target/`) at
   every agent launch. `rust-lsp-install` also has no host cache, unlike `rust-install`
   (`tools/sase_core_wheel_cache`). Concurrent workspaces that see the same new
   sase-core commit also build it in parallel. This pushed checks past the ~9-minute
   agent command limit (`signaled/143` at ~539 s). Phase `lsp-build-cache` fixes this.

### Out of scope, with owners

- **Seeing past known failures.** Labeling KNOWN/NEW failures and continuing past
  master-red stages is the planned `sase tool` E3 epic, researched in
  `research:202609/sase_tool_e3_e4_readiness__cld.md`, with prerequisite bug `sase-182`.
  That is the structural defense against the next red master. This epic removes causes
  and restores green; it does not change `tools/run_silent`'s fail-fast behavior or exit
  codes. The research's open question 1 (continuation default) is for Bryan. Recommend
  planning E3 next.
- **Landing policy.** Host completion committing only on "no new failures" belongs to
  E4.
- **CI Master Gate.** Master Gate is red on every commit for an independent reason: the
  CI core pin lags the Python code, so the binding check and core-dependent tests fail.
  That is tracked under `sase-126` and the core-pin ratchet, not here.
- **Feature work of in-flight epics.** sase-17x (command line; final phase sase-17x.12
  in flight), sase-17d (decks; sase-17d.10.1 land in flight), sase-17m (agent-session
  rename), and sase-185 (pending launches) caused most of the current items. This epic
  only restores gates. It must not change those features' intended behavior.

## Rules for every phase

- **Re-derive before fixing.** Master moves every few minutes and in-flight land agents
  may fix items first. Start from the latest `origin/master`. Run `just install` if
  `_setup` reports drift. Reproduce each listed item on a clean tree. Skip items already
  fixed upstream; check with `git log origin/master -- <file>`. An item that is new
  since this plan is in scope when it is in the phase's area and small. Otherwise record
  it as a `PROPOSED FOLLOW-UP:` note on the phase bead, with the commit that introduced
  it.
- **Run lint stages individually.** `just check` stops at the first failure. Use
  `just fmt-py-check`, `just fmt-md-check`, `just lint-keep-sorted`, `just _lint-ruff`,
  `just _lint-mypy`, `just _lint-flags`, `just _lint-pyscripts`,
  `just _lint-test-waits`, `just _lint-changelog`,
  `just _lint-patch-stitch-terminology`, `just _lint-symvision`, `just _lint-toobig`,
  `just validate`, and `just validate-committed-plans`.
- **Unmask symvision.** Symvision stops at its first failing category. Reveal the hidden
  categories with the throwaway probe in `plan:202609/symvision_green_master.md` Step 0.
  Never commit the probe.
- **Find intent before changing a test.** Use `git log -S`/`git blame` to find the
  causing commit and its `SASE_PLAN`/`SASE_BEAD` trailers. Update or delete a test only
  when it encodes behavior that the causing plan intentionally removed or changed.
  Otherwise fix the code.
- **Never weaken a guard test to pass.** This covers system-clock sites, host paths,
  import budget, audits, and schema. Fix the code. Raise a budget only with a measured
  cause stated in the commit message.
- **Record fixes for owning epics.** When a fix lands for an item that an in-flight
  epic's `DISCOVERED ISSUE` note names (see sase-17x notes 1–3), record it in a note on
  this phase's bead so the epic's land agent does not redo it.
- **Verify, then finish.** Run `sase tool run check` before finishing. The full
  `just test` exceeds the agent command limit, so run it only through `/sase_monitor`.
  Prefer targeted `pytest` node ids.

## Restore every lint gate except toobig on master

Items at `d5aa86dfc`:

**mypy.**

- `src/sase/ace/tui/widgets/_agent_detail_display.py` (lines 69, 91, 98, 156, 176,
  181, 199) and `_agent_detail_state.py` (50, 62, 267) report `query_one`,
  `_sync_header_visibility`, and `update_display` undefined. `742c1df38` removed their
  `AgentDetailPanelMixin(Static)` base, which used to supply them. Declare them for type
  checking only, using the class-level `if TYPE_CHECKING:` host-method pattern in
  `src/sase/ace/tui/modals/models_panel_display_options.py`. Do not add runtime stubs
  that could shadow Textual's `Widget.query_one` or `AgentDetail`'s real methods in the
  MRO.
- `src/sase/ace/tui/actions/agent_workflow/_launch_prompt_inputs.py:178`:
  `LaunchPromptInputMixin` lacks `_accept_resolved_launch` (from `93d8d4140`). Declare
  it the same way.
- `src/sase/ace/tui/command_line/input.py:158`: assigns `TextAreaTheme | None` to a
  `TextAreaTheme` variable. Handle `None` explicitly.
- `src/sase/ace/tui/command_line/screen.py` (about 1300–1317): passes `LineContext` /
  `LineContext | None` where `dict[str, Any]` is declared (`set_resolve_context`,
  `_complete_line`, `_maybe_fetch_providers`). Retype those parameters to the typed
  `LineContext` from `sase.completion.command_line_grammar`, and do not cast.

**test waits.** `tests/ace/tui/command_line/test_completion_popup.py:181` has a fixed
sleep without a pragma. Replace it with an observable wait (`sase.ace.testing.wait`).

**symvision.** 41 unused public symbols. Read `sase memory read symvision.md` and apply
its hierarchy: delete dead code, then privatize single-file helpers. Use a pragma only
for real external consumers. Use `--epic-symbol` only when a named, still-open epic
phase will consume the symbol, which after sase-17x.12 is unlikely. Test references do
not keep a public symbol alive.

- `src/sase/completion/command_line_grammar.py`: `CommandHelp`, `CommandLineCompletion`,
  `CommandLineGrammar`, `CompletionItem`, `DynamicCandidate`, `HelpChild`, `HelpOption`,
  `HelpPositional`, `LineDiagnostic`, `LineSignature`, `LineSlot`, `LineToken`,
  `RunPolicyOutcome`, `SignatureSegment`. These are wire types of a Rust adapter. If
  `LineContext` becomes a used annotation (see mypy), check whether the others are
  genuinely unused or only referenced in strings.
- `src/sase/ace/tui/command_line/`:
  - `transcript.py`: `CommandLineBlockWidget`
  - `sources.py`: `SourceCandidate`, `in_memory_candidates`
  - `signature.py`: `build_chips`, `build_signature`, `first_diagnostic_message`,
    `option_summary_text`, `role_style`
  - `restore.py`: `command_line_from_proc`, `select_restore_rows`
  - `grammar.py`: `command_line_grammar_error`, `load_command_line_grammar_sync`
  - `exits.py`: `completion_toast_text`
  - `popup.py`: `longest_common_prefix`, `render_popup_row`
- `src/sase/ace/tui/keymaps/bindings.py`: `build_command_line_bindings`.
- `src/sase/history/command_line.py`: `command_line_history_file`,
  `save_command_line_history`, `set_command_line_history_file`.
- `src/sase/ace/tui/modals/procs_pane_agent_jump.py`: `is_command_line_row`,
  `open_command_line_on_block`.
- `src/sase/ace/tui/widgets/vim_search_controller.py`: `invert_search_direction`,
  `offset_for_row`, `wrap_feedback_message`.
- `src/sase/ace/tui/widgets/file_panel/_file_list.py`: `FileSourceLabel`.
- `status_text` in both `src/sase/main/monitor_render.py` and
  `src/sase/main/proc_render.py`.

Also fix anything Step 0 reveals: masked symvision categories, feature-flag rules,
pyscripts on a clean tree, and so on.

**Done when:** on a clean tree at the phase's final HEAD, every lint stage above exits 0
except `just _lint-toobig`, and the symvision probe shows no hidden category.

## Split the two oversized ACE modules

At `d5aa86dfc`, `src/sase/ace/tui/command_line/screen.py` has 1,718 lines and
`src/sase/ace/tui/widgets/decks/panel.py` has 1,067. The toobig limit is 1000, with a
warning at 850. Split each one below 850 lines.

- Keep the split mechanical: move code without rewriting behavior. sase-17x's final
  phase is still editing the command-line screen, so rebase immediately before
  finishing.
- Keep public import paths stable, e.g. `sase.ace.tui.command_line.screen`. Put moved
  code in `_`-prefixed sibling modules or mixins, following the existing ACE mixin
  conventions.
- Never import a `_private` name across modules. A helper that more than one new module
  needs gets a public name inside an already-private module.
- Keep monkeypatch targets working, or retarget the tests. Search `tests/` for string
  patch targets that name the moved functions.

**Done when:** `just _lint-toobig`, `just _lint-symvision`, and `just _lint-mypy` pass.
`tests/ace/tui/command_line/` and the deck-panel tests pass. `sase tool run check` on a
clean tree passes every lint stage and reaches `test (scoped)`.

## Repair non-UI tests that fail on clean master

Failing at `d5aa86dfc` in a full `just test`:

**Wait checks (`sase-186`, from `9bd351b67`).**

- `tests/test_axe_chop_wait_checks.py::test_wait_checks_no_projects_dir_emits_noop_summary`
  fails on the new `deferred_unconfirmed` summary field.
- `tests/test_axe_chop_wait_checks.py::test_multiple_waiting_dependencies_scan_artifacts_once`
  sees 10 meta reads against a contract of 5.
- `tests/test_axe_chop_incremental_scans.py::test_wait_checks_index_resolution_skips_filesystem_meta_reads`
  sees 2 reads against a contract of 0.
- Follow the bead's analysis. Prefer making the confirmation pass reuse the index or
  cached meta reads so the scan-once contract holds. Update the summary-text test either
  way.

**System-clock guard (`sase-188` plus the command line).**
`tests/test_timezone_display_guard.py::test_no_system_clock_display_sites` flags:

- `src/sase/completion/candidates/catalog_plans.py` (around lines 380, 403, 410)
- `src/sase/ace/tui/command_line/block_render.py` (around line 125,
  `datetime.fromtimestamp(...).strftime("%H:%M")`)

Route both through the `sase.core.time` helpers. Do not extend the allowlist.

**Config schema.**
`tests/test_config_schema.py::test_default_config_matches_public_schema` fails because
`ace.keymaps.command_line` is in `default_config.yml` but missing from
`src/sase/config/sase.schema.json`. Add the schema entry, using the schema generator if
one exists.

**fakey help color (from `5a7955161`, sase-17x.1).**
`tests/fakey/test_cli.py::test_help_is_colored_sorted_and_all_long_options_have_aliases`
expects colored help while stdout is not a TTY. Align the test with the shared stdout
color contract, `sase.core.term_color` (set `FORCE_COLOR=1`), unless that contract says
otherwise.

**Agent-session rename fallout (sase-17m.4.1.x).**

- `tests/ace/tui/test_kill_and_edit_prompt_name.py`: 7 tests fail because
  `prepare_kill_and_edit_prompt()` no longer accepts `family_name`.
- `tests/test_launch_approval.py::test_agent_launch_request_rejects_competing_family_successor`
  fails because its expected message `requester's family lane` changed.
- Migrate these tests to the renamed API and messages. Change code only if the rename
  broke behavior.

**Audit.** `tests/test_agent_artifact_marker_mutation_audit.py` fails in 2 tests. Review
each new marker-mutation site, then update the reviewed manifest.

**Load-sensitive tests.**

- `tests/test_provider_disable.py::test_facade_try_disable_one_winner_under_process_contention`
  hit a subprocess timeout.
- `tests/tool/test_lifecycle_controls.py::test_stop_live_inline_run_settles_stop_requested`
  hit `tool run store is busy: database is locked`.
- Rerun each at least 3 times in isolation on an unchanged tree. Fix a deterministic
  failure. For a flaky one, record a `PROPOSED FOLLOW-UP:` flake note with the run
  evidence.

When done, close `sase-186` and `sase-188` with notes citing the fixing commit. Use
`sase bead close` and read `sase memory read sase_beads.md` first.

## Repair ACE TUI tests that fail on clean master

Failing at `d5aa86dfc`. Most come from `742c1df38` (legacy agents UI removed) and the
deck cutover (sase-17d). Detail panes now render deck `CardPart`s instead of the old
scroll widgets and `Text` renderables.

**Detail pane and deck cutover.**

- `tests/ace/tui/widgets/test_agent_jump_panel_visibility.py::test_search_overlay_keeps_jump_panel_visible`:
  `NoMatches '#agent-prompt-scroll'`.
- `tests/ace/tui/widgets/test_agent_prompt_panel_steps.py::test_parallel_step_does_not_show_agent_prompt`.
- `tests/ace/tui/widgets/test_agent_prompt_panel_xprompts.py::test_update_display_renders_xprompts_after_detail_settles`.
- `tests/ace/tui/widgets/test_agent_prompt_semantic.py`: 3 tests.
- `tests/ace/tui/widgets/test_agent_tribe_prompts.py::test_same_body_in_different_projects_does_not_coalesce`:
  1 group instead of 2. Decide whether this is a coalescing regression.
- `tests/ace/tui/test_agents_tab_current_project_seed.py::test_update_state_rebuilds_when_seeded_flag_changes`:
  `update_state()` no longer takes `view_mode`.
- `tests/test_file_panel.py::test_zoom_file_cap_subtitle_points_to_editor`: imports the
  deleted `sase.ace.tui.modals.zoom_panel_events`.
- `tests/ace/tui/test_agent_fold_transitions_llm_calls.py`: 2 tests.
  `_arm_panel_fold_hint_mode` is missing, and the detail clamp no longer expands.
- `tests/test_keymaps_defaults_modes.py::test_zoom_and_agents_fold_defaults_are_in_sync_with_help`
  and `tests/test_keymaps_display_help_agents.py` (2 tests): the `Z` zoom binding and
  its help rows were removed. Keep the default keymap config in
  `src/sase/default_config.yml` consistent.
- `tests/test_timezone_display_tui.py::test_llm_calls_panel_fallbacks_use_configured_wall_time`:
  its fake agent lacks `identity`.

**Prompt-panel monitor tests (`sase-184`).**
`tests/ace/tui/widgets/test_agent_prompt_panel_monitor.py` fails in 3 tests: the monitor
section renders empty or without `TESTING`/`FAILED`. The bead says this predates
`7e1b05964`. Bisect as the bead suggests.

**Completion accept key moved to `ctrl-f` (`307da2dac`).**

- `tests/ace/tui/test_model_completion_panel_titles.py`: 2 tests still expect `Ctrl+E`.
- `tests/ace/tui/widgets/test_prompt_file_completion.py::TestPromptFileCompletion::test_ctrl_e_moves_to_line_end_while_completion_remains_open`.

Confirm the intended behavior from that commit's plan before updating the tests.

**Command line (sase-17x).**

- `tests/ace/tui/test_visual_fixture_host_paths.py::test_visual_fixtures_embed_no_host_home_paths`:
  `test_ace_png_snapshots_command_line.py` lines 36 and 143 embed
  `cwd="/home/test/projects/sase"`. Use an allowed synthetic owner or the renderer's
  `_HOME` snapshot. If that changes a golden, regenerate only the affected goldens with
  `just fix-tui-screenshots -- <selector>` and inspect them. Read
  `sase memory read tui.md` first.
- `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`:
  the module count is 3,349 against a budget of 3,290, which is deterministic. Find the
  new eager import edges into the app import graph, likely the command-line panel. Defer
  them. Do not raise `_MAX_MODULE_COUNT` without a measured reason.

**Artifacts contract.**
`tests/ace/tui/artifacts_contract/test_no_ref_prefix_dispatch.py::test_behavioral_modules_do_not_dispatch_on_ref_prefix`
names an `actions/artifacts…view_targets.py` module that dispatches on a ref prefix.
Route it through the artifact provider contract.

When done, close `sase-184` with a note citing the fixing commit.

## Ignore cache-only script directories in the pyscripts lint

In `tools/pyscripts-260801`, `_find_target_dirs()` adds every `scripts/` and `tools/`
directory it walks, even when the directory contains nothing but excluded subdirectories
such as `__pycache__`. `_has_closer_target_dir()` then reports false Rule 2 violations.

- Change: a `scripts/`/`tools/` directory counts as a placement target only when
  `_collect_scripts(d)` returns at least one file.
- Add a regression test that builds a temporary git repo with a real `tools/` script
  referenced from a nested file, plus a sibling `nested/tools/__pycache__/x.pyc`-only
  directory. Assert that there is no violation, and that there still is one once
  `nested/tools/` holds a real file.
- Optionally, also delete `__pycache__`-only directories during numbered-workspace
  preparation. The lint-side fix is required either way.

**Done when:** `just _lint-pyscripts` passes in a checkout containing a stale
`tests/ace/tui/tools/__pycache__/`.

## Refuse closing a flag bead while its registry definition survives

`handle_bead_close` in `src/sase/bead/cli_crud_lifecycle.py` already calls
`_refuse_leftover_epic_symbols` before `mutation.project.close`. The host finalizer's
`-B close` runs that same CLI (`src/sase/workflows/commit/bead_hooks.py`
`close_assigned_bead_after_commit`), so a guard added there covers both paths. Add a
sibling refusal:

- For each bead being closed that is a flag bead (task type `flag`, or the legacy flag
  issue type), find the working tree's `src/sase/feature_flags/registry.py`. Resolve it
  from the same start directory `_owner_symbol_start(bead_context)` gives the Justfile
  lookup.
- Parse the registry statically: find `bead="<id>"` keyword literals with `ast`. Do not
  import it; the running `sase` may be a different install than the working tree.
- If a definition names the bead, refuse with an error naming the flag key and the fix:
  delete the Off branch, make the On branch unconditional, and remove the registry entry
  in the same change, or keep the bead open. Mirror the epic-symbol wording.
- Skip silently when the registry file does not exist (non-sase projects, trees without
  a checkout).
- Leave the `FlagTriage` gate's deliberate "Close" (abandon) action alone.
- Add no new CLI option. Put the helper beside `src/sase/bead/epic_symbols.py` (for
  example `src/sase/bead/flag_definitions.py`).

Tests:

- Refusal when a definition survives.
- Success once the entry is removed in the working tree.
- Non-flag beads are unaffected.
- A missing registry file skips the check.
- The finalizer close path surfaces the refusal as a failed close, as it does for
  epic-symbol refusals today.
- Update `docs/` where the epic-symbol refusal is documented.

## Make the split_file xprompt keep symvision and mypy green

`src/sase/xprompts/split_file.md` is the prompt every toobig routine split agent runs
(`%auto #split_file:<path>`). Today it says only "split into files <=500 lines". Expand
it; read `sase memory read xprompts.md` first. It should require:

- Keep the original module's public import path working. Re-export only public names;
  never re-export `_private` names from a facade.
- Never import a `_`-prefixed name across the new modules. A helper that more than one
  module calls gets a public name inside an already-private (`_`-prefixed) module. Move
  a helper that only one other module uses into that module.
- Keep monkeypatch targets that tests use working, or retarget the tests.
- Before finishing, run `just _lint-symvision`, `just _lint-mypy`, and
  `just _lint-toobig` individually. Fix every item in a file the split touched, even
  when an earlier stage of `just check` is already red. Then run `sase tool run check`.

Update any xprompt tests that pin its content. Close `sase-180` with a note: this phase
removes the split-agent cause, and the "agents must compare masked stages" half belongs
to the planned E3 continuation work (`research:202609/sase_tool_e3_e4_readiness__cld.md`
§4.5, §4.9).

## Cache sase-xprompt-lsp builds and dedupe concurrent core builds

**Facts.**

- `rust-install` (Justfile) looks up and stores the `sase_core_rs` wheel in
  `tools/sase_core_wheel_cache`, keyed by the sase-core source identity
  (`tools/_sase_core_source_identity.py`).
- `rust-lsp-install` always runs `cargo build --profile dev-update -p sase_xprompt_lsp`
  with `CARGO_TARGET_DIR=<linked clone>/target/uv-tool-lsp`.
- `clear_workspace_repos` (`src/sase/_linked_repo_workspaces.py`) moves that clone to
  trash at each agent launch. The launch env also gives every agent a fresh per-launch
  `CARGO_TARGET_DIR` (`src/sase/agent/launch_spawn.py`). Every rebuild therefore starts
  cold.
- Keep that isolation. Fix the cost with caching.

**Changes.**

1. Add a host-wide content-addressed cache for the `sase-xprompt-lsp` binary. Key it by
   the same source-identity digest the wheel cache uses, plus the Cargo profile, target
   triple, and `rustc -V`. Prefer extending `tools/sase_core_wheel_cache` with an
   artifact kind so both artifacts share root discovery, eviction (max bytes/entries),
   and the "clean checkout only" rule (`CacheUnavailable`). A sibling tool that reuses
   its helpers is also acceptable.
2. In `rust-lsp-install`, look up the cache first. On a hit, install with the existing
   atomic `tmp` + `mv` copy and print
   `[rust-lsp-install] Installing cached sase-xprompt-lsp from <path>.` On a miss, build
   as today, then store the result best-effort.
3. Dedupe concurrent builds. Around each "lookup → miss → build → store" sequence, for
   both the wheel and the LSP binary, take an exclusive per-identity lock (`flock`)
   under the cache root. Re-check the cache after acquiring the lock. Bound the wait
   (about 15 minutes), then build anyway rather than deadlock. Print one line while
   waiting so ToolRun logs show why setup paused.
4. Leave source-identity stamping, `purge_sase_core_rs_extensions`, and the stale-core
   floors unchanged.

**Tests.** Extend `tests/test_sase_core_wheel_cache_tool.py` (and
`tests/test_rust_install_cleanup.py` or the Justfile contract tests, if they pin the
recipe) to cover:

- LSP store/lookup/eviction
- key separation by profile and toolchain
- dirty-checkout refusal
- lock re-check after wait
- lock timeout fallback

**Done when:** in two numbered workspaces at the same sase-core HEAD, the second
`_setup` after a sase-core bump installs both the wheel and the LSP from cache, with no
`Compiling` lines, in seconds. Report the measured setup time from the ToolRun
(`UNATTRIB`) before and after in the phase's final note.

## Verify green check and full test suite on clean master

On a clean checkout of the latest `origin/master`, with `just install` first if needed:

1. `sase tool run check` exits 0. Record the run id.
2. The full non-visual `just test` passes, apart from nodes listed in
   `tests/reproducible_flake_baseline.txt`. Run it through `/sase_monitor` with the
   `TESTING`/`TESTED` status pair, because it takes about 15 minutes.
3. `just _lint-pyscripts` passes with a stale `__pycache__`-only `tools/` directory
   present.

If concurrent landings introduced new master-red items, fix up to about five small,
clearly attributable ones in place. Record the rest as `PROPOSED FOLLOW-UP:` notes
naming the introducing commit and its `SASE_BEAD` trailer.

Final note on the phase bead:

- the check and test run ids
- the before/after ledger comparison, using `sase tool runs -t check -j` over the days
  after landing: pass rate, and the share of runs that reached `test (scoped)`
- the LSP-cache setup timings
