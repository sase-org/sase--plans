---
tier: epic
title: TUI macro surfaces and goldens
goal: Rename the SASE TUI's xprompt modules, identifiers, CSS, copy, keymap actions,
  and Admin Center ids to macro spellings. The agent prompt tab and headings say raw
  prompt. Pre-rename resume state still opens, retired keymap actions stay flag-gated
  aliases, and the PNG goldens match the new pixels.
parent_bead: sase-1eq.5
phases:
- id: tui-contracts
  title: Keymap, resume ids, and stats request contracts
  size: medium
  depends_on: []
  description: 'tui-contracts: make focus_macro, clear_macro_focus, and start_last_vcs_macro_in_editor
    the keymap actions, with flag-gated aliases in legacy_xprompt_syntax.py. Resume
    Admin Center sub-tab and Statistics view id xprompts as macros, and send macro
    stats request keys.'
- id: tui-browser
  title: Macro browser, save flows, and location labels
  size: medium
  depends_on:
  - tui-contracts
  description: 'tui-browser: move the browser, unified-save, mini-macro modal, and
    agent-workflow save modules to macro names. Rename their identifiers, CSS selectors,
    and location labels with their tests.'
- id: tui-completion
  title: Completion, argument assist, and highlight roles
  size: medium
  depends_on:
  - tui-browser
  description: 'tui-completion: move completion, argument-assist, and syntax modules
    to macro names. Rename in-process highlight roles to macro.* in the TUI and the
    CLI render mirror together.'
- id: tui-prompt-copy
  title: Prompt panel, raw-prompt headings, and remaining visible copy
  size: medium
  depends_on:
  - tui-completion
  description: 'tui-prompt-copy: rename prompt-panel and mini-bar modules, apply the
    raw-prompt tab and AGENT RAW PROMPT headings, and update help, keymap descriptions,
    and command-palette copy.'
- id: tui-sweep
  title: Remaining TUI identifiers and assigned mirror files
  size: medium
  depends_on:
  - tui-prompt-copy
  description: 'tui-sweep: finish every remaining in-scope xprompt hit, including
    statistics identifiers, scattered widgets, and the non-TUI mirror files the terminology
    guard already assigns to sase-1eq.5.'
- id: tui-goldens
  title: PNG goldens, terminology guard, and navigation benchmark
  size: medium
  depends_on:
  - tui-sweep
  description: 'tui-goldens: rename the xprompt PNG goldens, re-baseline changed pixels,
    widen the terminology guard over the TUI scope, and run the j/k navigation benchmark.'
proposed_by: bbugyi200.athena.sase-1eq.5
create_time: 2026-10-03 13:29:38
status: done
bead_id: sase-1eq.5.1
---

- **PROMPT:** [prompts/202610/tui_macro_surfaces.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/tui_macro_surfaces.md)
- **PARENT:** [202610/xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/xprompts_to_macros.md)
- **BEAD:** [sase-1eq.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.5.1.md)

# TUI macro surfaces and goldens

## Scope and evidence

This implements the entire `tui` phase of bead `sase-1eq.5`, defined in
`plan:202610/xprompts_to_macros.md`. Read that artifact with `sase artifact read` and
the assigned bead with `sase bead read` before implementing. Its Shared Policy binds
every phase here. Prerequisite `sase-1eq.4` is closed. `sase-1eq.10` (`core-flip`) is in
progress and must keep receiving the pre-flip wire spellings this plan does not own.

The tree still uses xprompt vocabulary throughout the TUI. A case-insensitive count
found about 7,600 hits in 525 files under the phase scope, 165 matching lines in
`src/sase/ace/tui/styles.tcss`, and 34 PNG goldens whose names contain `xprompt`. That
volume, plus a breaking keymap contract and a screenshot re-baseline, is why this is an
epic. A tale cannot be `large`, and one medium agent cannot own the contract, the module
moves, the copy pass, and the golden run together.

In-scope trees:

- `src/sase/ace/tui/**`
- `src/sase/ace/testing/**`
- `tests/ace/**`
- `tests/perf/**`
- the TUI parts of `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`

Also in scope, because the current terminology guard already labels them
`owned by sase-1eq.5` and they share TUI vocabulary:

- `src/sase/macro/cli_show_render.py`
- `src/sase/macro/highlight.py`
- `src/sase/macro/highlight_theme.py`
- `src/sase/macro/jinja_assist.py`
- `src/sase/macro/save_index.py`
- `src/sase/macro/write_targets.py`
- `src/sase/stats/query.py` (the stats request keys named in the parent phase)
- the keymap tests `tests/test_keymaps_defaults_panels.py`,
  `tests/test_keymaps_registry_loading_panes.py`, and `tests/test_keymaps_validation.py`
- `tests/macro/test_argument_surface_parity.py`, `tests/macro/test_cli_show_body.py`,
  `tests/macro/test_cli_show_render.py`, `tests/macro/test_highlight.py`, and
  `tests/macro/test_highlight_theme.py`
- any other file that imports or quotes a symbol this epic renames

Out of scope: docs, README, blog, memory, skill deploy, chezmoi, plugins, nvim,
telegram, the `src/sase/xprompt/` import shim, generated changelogs, accepted decision
records, `sdd/` archives, and every core emitted wire that `core-flip` still owns. Do
not edit sase-core. Do not close `sase-1eq`, `sase-1eq.5`, or `sase-1eq.10`. A phase
description is not authorization to close an ancestor. Record discovered follow-up work
as `PROPOSED FOLLOW-UP:` on that phase's own bead. Do not create beads.

## Shared rules

Read `tui.md` and the child note it links before the matching work: `tui_screenshot.md`
before any golden or live capture, `tui_perf.md` before touching navigation, render,
refresh, or startup. Read `lint_and_test.md` before finishing a phase that changes the
sase tree, and `symvision.md` before fixing symvision findings. Read `sase_flags.md`
before changing flag behavior.

Vocabulary follows the parent plan.

- Rule 1 is the default. `xprompt` / `XPrompt` / `XPROMPT` become `macro` / `Macro` /
  `MACRO`, plurals included. Fix prose the replacement leaves ungrammatical. Replace in
  this order: `an xprompt` → `a macro`, then `XPROMPT`, `XPrompt`, `Xprompt`, then
  `xprompt`.
- Rule 2 names the authored prompt as a whole and becomes "prompt", never "macro". Apply
  it only to the names below and to the collisions already decided in this plan. Before
  renaming any other name under Rule 2, check it for collisions and record the decision
  on the phase bead.
  - `PREVIEW_TAB_LABEL` becomes `RAW PROMPT`. `preview_card` crops the tab instead of
    wrapping it, and the agents goldens are 160 columns wide, so use the full label.
    Fall back to `RAW` only if a real golden shows a broken crop, and record why.
  - Every visible heading and entry kind `AGENT XPROMPT` becomes `AGENT RAW PROMPT`.
    Known sites include `agent_run_log_modal.py`, `_identity_header.py`,
    `_agent_display_hint_body.py`, `_agent_display_render.py`,
    `_agent_display_agent_session_render.py`, `_agent_display_clan_sections_text.py`,
    `_agent_clan_member_content.py`, and `_metadata_pager_conversation.py`. Update
    comparisons against the kind in the same change.
  - The pager section key in `_CONVERSATION_SECTIONS` is `"xprompt"` beside an existing
    `"prompt"` key for the expanded prompt. Renaming it to `"prompt"` collides. Rename
    that internal key, its identity suffix, and `_tag_styled_xprompt_body` to
    `raw_prompt`, not `macro` and not `prompt`.
  - Artifact names and accessors stay Rule 2: `raw_xprompt.md` → `raw_prompt.md`,
    `submitted_xprompt` → `submitted_prompt`, `get_raw_xprompt_content` →
    `get_raw_prompt_content`, and the `%proc` origin family `xprompt-proc` /
    `xprompt_proc` → `prompt-proc` / `prompt_proc`. Do not rewrite a longer identifier
    merely because it contains `xprompt_proc` as a substring.
    `_is_standalone_xprompt_row` names a macro definition row and becomes
    `_is_standalone_macro_row`.
- Jinja `{% macro %}` / `endmacro`, `JinjaLocalKind` macro variants, Rust
  `macro_rules!`, query-language shorthands, and the LSP semantic token type `macro`
  stay as they are. User-facing Jinja copy says "Jinja macro".
- `macro` is a Jinja statement keyword. Do not expose a template variable named `macro`.
- Codemods skip `src/sase/legacy_xprompt_names.py`, `src/sase/legacy_xprompt_syntax.py`
  except for the keymap table this plan adds, every `LEGACY_*` and `legacy_*`
  identifier, fixtures named `*legacy_xprompt*`, and the history listed in the parent
  plan. After each codemod, review the diff of every legacy constant.
- Keep the codemod as a re-runnable scratch script. Sync to latest master and re-run it
  rather than hand-merging a conflict.
- Move paths with `git mv`. Never rewrite PNG bytes by search-and-replace.
- Re-sort `keep-sorted` blocks, `__all__` lists, and schema properties whose existing
  order is alphabetical. The Statistics keymap block is logical order aligned with the
  dataclass, not alphabetical. Keep that alignment when the field is renamed. Do not
  alphabetize it.
- Default-keymap gotcha: `AppKeymaps` and `StatisticsPaneKeymaps` fields must all be
  present under `ace.keymaps.<scope>` in `default_config.yml`, or startup raises
  `ValueError`. The YAML file is the source of truth. A rename updates the dataclass
  field, the YAML key, and the schema property together.
- A non-TUI test that imports or quotes a renamed symbol belongs to the phase that
  renames it. Update the terminology pair rows in the same change so
  `tests/test_macro_terminology.py` stays green. Do not leave that failure for a later
  phase.
- Renames only. Add no filesystem work, awaits, subprocesses, or full rebuilds to
  navigation or render handlers.
- Writers emit the new spelling. Durable readers prefer the new spelling and fall back
  to the old one, and they are never flag-gated. Authored keymap aliases are flag-gated.
  Help, completion, examples, and output never show a retired spelling.
- Verify with `sase tool run check` in sase. Run `just install` first if the workspace
  environment is stale. Do not run `just check-full`. `just check` does not run PNG
  snapshots. A phase that changes rendered pixels runs a targeted
  `just fix-tui-screenshots` through `/sase_monitor` and inspects that run's report
  before finishing. `partial` means the skipped goldens are not current. Only
  `tui-goldens` may remove stale goldens, and only after a full run. Generation is not
  approval.
- Each phase's final declaration covers the sase repo. Commit the contract phase with a
  `feat!:` subject and a `BREAKING CHANGE:` footer listing the three keymap actions.
  Later phases do not repeat that footer.

## Phase tui-contracts

Own the breaking user-facing ids, and leave module filenames alone.

Keymap actions become `focus_macro`, `clear_macro_focus`, and
`start_last_vcs_macro_in_editor`.

- Update `StatisticsPaneKeymaps`, `AppKeymaps`, `default_config.yml`, `sase.schema.json`
  descriptions, `keymaps/metadata.py`, bindings, action methods, help-modal action
  checks, and the command catalog entries that name these actions.
- Add the three pairs to `src/sase/legacy_xprompt_syntax.py`. The statistics and app
  keymap loaders consult that table. The flag on accepts a legacy key as its macro
  action. The flag off rejects it with an error that names the replacement. Both
  spellings in one mapping is an error in both flag states. Do not only warn and ignore,
  which is what `migrate_key_aliases` does for older aliases. Model the error on
  `normalize_frontmatter_macros`.
- The doctor table in `checks_config_retired.py` already lists the pairs. Keep it.
- Add tests for both flag states and for both spellings present.
- Schema descriptions that say "xprompt" become "macro", except the
  `legacy_xprompt_syntax` flag prose, which stays.

Resume ids, two different levels:

- `validated_center_tab` in `config_center_catalog.py` maps a persisted top-level Admin
  Center id `"xprompts"` to `"config"`. Leave that mapping and
  `tests/ace/tui/test_config_center_state.py` in place. It is not this rename.
- The Config hub sub-tab id in `config_hub_session.py` and `config_hub_catalog.py`
  becomes `"macros"`. `validated_config_subtab` reads a stored `"xprompts"` as
  `"macros"`, unconditionally, before the membership check. It must not return
  `"config"`.
- The Statistics view id in `statistics_pane_data.py` becomes `"macros"`. The validator
  or loader that accepts a stored view id reads `"xprompts"` as `"macros"`,
  unconditionally. If no disk file stores the view today, put the mapping on that
  validator anyway and unit-test the literal. Do not add a new persistence file.
- Update every comparison, literal type, bookmark field, and widget id that is this
  sub-tab or this view, so the tree stays consistent. The bookmark field
  `AdminCenterSessionState.xprompts` is a Rule 1 identifier and becomes `macros`.
- Visible labels for that Config sub-tab: `Macros`, compact `Macros`, micro `Ma`. `Ma`
  is the 2-letter label. Sibling micros are Flag, Hold, Run, Mem, and Snip. Statistics
  view labels: `Macros`, compact `Macros`, micro `Mac`. Sibling statistics micros are
  Ovr, Rnrs, Proj, Prov, Act, P&Q, and Prf. Descriptions become the macro wording
  already specified in the parent phase: adoption, focus, and breakdowns. The empty
  state title becomes `Macro statistics unavailable`. The focus chrome says
  `Macro focus`.

Stats requests:

- `query_run_stats` in `src/sase/stats/query.py` already takes `macro_top_n`,
  `macro_breakdown_top_n`, and `macro_focus`, then sends `xprompt_top_n`,
  `xprompt_breakdown_top_n`, and `xprompt_focus`.
- Send `macro_top_n`, `macro_breakdown_top_n`, and `macro_focus`. The parent phase says
  `core-expand` accepts the aliases. Confirm the pinned core accepts each key before
  switching it. A key core rejects stays on the legacy request spelling, keeps its guard
  row, and gets a `PROPOSED FOLLOW-UP:` note that `core-flip` owns it.
- Response bodies still use the pre-flip keys until `core-flip`. Keep reading those
  response keys through the existing mirror constants. Do not hard-code a new response
  literal that core does not emit.
- Update `tests/stats/test_query.py` and the terminology rows that pin the old request
  keys.

`default_config.yml` comments that say the collapsed header previews an xprompt, and the
schema sentences about the `XPROMPT` tab row, become macro and `RAW PROMPT` in this
phase because those strings live in the two files this phase already edits.

Exit: keymap tests pass in both flag states, the top-level `xprompts` → `config` resume
test still passes, the new sub-tab and statistics-view reader tests pass, and
`sase tool run check` passes. Targeted screenshot update for the Admin Center and
Statistics goldens whose labels changed.

## Phase tui-browser

Own the browser, save, and mini-macro modal modules, after the contracts phase so
`styles.tcss` has one writer at a time.

Move these with `git mv`, then rename identifiers, classes, CSS ids, and CSS classes
together with their Python selectors:

- `modals/xprompt_*`
- `modals/unified_xprompt_save_*`
- `modals/mini_xprompt_*`
- `modals/add_xprompt_modal.py`, `local_xprompt_name_modal.py`, and
  `xprompt_write_conflict_modal.py`
- `actions/agent_workflow/*xprompt*`
- the matching tests under `tests/ace/**`

Location labels currently hard-code `Project sase/xprompts/`,
`Project xprompts/ (legacy)`, `Home ~/sase/xprompts/`, and `Home xprompts/ (legacy)` in
`xprompt_browser_helpers.py`, `xprompt_browser_catalog.py`,
`unified_xprompt_save_support.py`, and `xprompt_location_modal.py`. Canonical labels use
the macro directories (`sase/macros/`, `macros/`, `~/sase/macros/`). A row whose file
still lives in a retired directory keeps a distinct group, but the visible label names
the macro replacement and does not spell xprompt. Recognition of the retired path goes
through the existing legacy directory constants, not a new TUI literal.

`WrittenFileKind.MACRO` is the string `"xprompt"`, and save helpers default `noun` and
`commit_type` to `"xprompt"` (`src/sase/macro/write_targets.py`). `IndexKind` /
`DefinitionKind` include `"xprompt_config"` (`src/sase/macro/save_index.py`). Writers
emit `macro` and `macro_config`. Before deleting the old spelling, check whether
`commit_type` or the index kind is parsed back from durable history. Where a value is
actually persisted, add an unconditional legacy reader and a legacy-input test. Where it
is only an in-process cache key, rename it.

Browser and picker copy becomes `Macro Browser [n macros]`, `Select Macro`, and
`Save Pane as Local Macro`, including `prompt_submit_choice_modal.py` if that modal's
save strings live next to these flows. If the submit-choice modal is outside the moved
files, leave it for `tui-prompt-copy` rather than editing it twice.

Update every importer, including non-TUI tests such as `tests/test_command_catalog.py`.
Exit with `sase tool run check` and a targeted screenshot update for the browser, save,
and mini-macro goldens.

## Phase tui-completion

Own completion, argument assist, inline expansion, and syntax highlighting.

Move with `git mv` and rename identifiers and CSS together:

- `widgets/xprompt_arg_assist.py` and `widgets/_xprompt_arg_assist_*`
- `widgets/_xprompt_arg_hints.py`
- `widgets/xprompt_completion.py`
- `widgets/xprompt_inline_expansion.py`
- `widgets/_xprompt_syntax_highlight.py`
- `widgets/_file_completion_xprompt_args.py`
- `util/xprompt_syntax.py`
- the matching tests, including
  `tests/ace/tui/widgets/_xprompt_arg_completion_helpers.py`

Highlight roles such as `xprompt.invocation` and `xprompt.arg_value` are in-process
style names shared by `sase.macro.highlight`, `sase.macro.highlight_theme`,
`sase.macro.cli_show_render`, and the TUI highlighter. Rename them to `macro.*` in all
of those files in this phase, plus `tests/macro/test_highlight.py`,
`test_highlight_theme.py`, `test_cli_show_render.py`, `test_cli_show_body.py`, and
`test_argument_surface_parity.py`. Confirm first that the role string is not a
core-emitted or durable value. If it is, keep the legacy reader and do not drop the old
spelling.

`MacroArgumentSource = Literal["xprompt", "directive"]` and the matching `_SOURCES` set
are Rule 1 values. Rename the source to `macro`. Apply the same durable-reader check as
the browser phase before dropping a stored value.

Do not rename `JinjaScopeKind` here. `tui-sweep` owns that wire.

Exit with `sase tool run check` and a targeted screenshot update for the prompt
argument-completion and highlight goldens.

## Phase tui-prompt-copy

Own the prompt panel, the collapsed header preview, and the remaining user-visible
sentences in the TUI modules this phase lists.

Move `widgets/prompt_panel/_agent_xprompts.py` and the other prompt-panel modules whose
names contain xprompt. Rename `widgets/_prompt_input_bar_*xprompt*` and
`widgets/_local_xprompt_conversion.py`. Apply the Rule 2 heading, tab, and pager-key
decisions from Shared rules in the same change, including `agent_header_preview.py`.

Also update, when they still contain xprompt prose:

- `modals/help_modal/` (`binding_common.py`, agents, axe, and patches)
- `keymaps/metadata.py` descriptions that `tui-contracts` did not already change
- `commands/_app_metadata_display.py` command-palette keywords
- `modals/prompt_submit_choice_modal.py`
- mini-macro pane copy that still says xprompt

`frontmatter_panel.py` still understands a legacy `xprompts` schema key. Keep that read
on `normalize_frontmatter_macros`. The visible and canonical key is `macros`. Do not
drop the legacy read.

Exit with `sase tool run check` and a targeted screenshot update for the agents, prompt,
preview, and help goldens whose pixels changed.

## Phase tui-sweep

Finish every remaining case-insensitive `xprompt` hit in the in-scope trees and in the
mirror files listed under Scope. This includes statistics module
`statistics_pane_xprompts.py` and its identifiers, `modals/statistics_pane*.py` beyond
the view id, models, commands, util leftovers, `src/sase/ace/testing/**`, and
`tests/perf/**`.

Classify `tests/perf` before renaming:

- Fixtures that write `raw_xprompt.md` as a pre-rename agent artifact are legacy inputs.
  Keep the old filename, name the fixture so the codemod skip recognizes it, and
  allowlist it. If the test instead asserts that a writer creates `raw_xprompt.md`,
  expect `raw_prompt.md` and keep a separate legacy-read test where the product has that
  reader.
- `xprompts_used` in `bench_detail_header_summary.py` is a pre-flip response field until
  `core-flip`. Leave the wire spelling, allowlist the row, and point the reason at
  `sase-1eq.10`.
- The bench print `xprompt tokenizer` is prose. It becomes `macro tokenizer`.

`JinjaScopeKind = Literal["prompt", "xprompt"]` in `src/sase/macro/jinja_assist.py`
already documents the macro-definition scope, while the literal is still the pre-flip
spelling. Confirm against the pinned core. If core accepts `macro` on input and still
emits `xprompt`, send `macro` and keep accepting `xprompt` until `core-flip`. If core
rejects `macro`, leave the literal, retarget the guard reason to `sase-1eq.10`, and
record a follow-up. Do not drop a value the current core emits.

Do not widen the terminology guard over the TUI trees in this phase. PNG filenames still
contain xprompt until `tui-goldens`. Do remove or rewrite `owned by sase-1eq.5` rows for
files this phase actually cleaned, so the existing non-TUI guard stays accurate.

Exit: `git grep -i xprompt` in the in-scope trees returns only classified leftovers
(legacy readers, the top-level `xprompts` → `config` migration, pre-flip wires, and the
not-yet-renamed PNG filenames). `sase tool run check` passes.

## Phase tui-goldens

Rename the 34 xprompt-named goldens and their tests, including window titles. `git mv`
the PNG filenames to the macro names. Re-baseline pixels only through the screenshot
tool.

Run the full `just fix-tui-screenshots` through `/sase_monitor`. Inspect every creation,
every removal, and every update group in the report. Expand groups with unexpected
differences. Remove stale goldens only after that full run reports complete evidence. A
`partial` result does not authorize removal. Read `tui_screenshot.md` and the PNG
section of `lint_and_test.md` first.

Widen `tests/test_macro_terminology.py` to the TUI scope: drop the `_TUI_SCOPES`
exclusion for `src/sase/ace/tui` and `tests/ace`, and include `src/sase/ace/testing` and
`tests/perf`. Delete the `_tui_module_basenames` exemption once no TUI module stem
contains xprompt. The allowlist may contain only classified lines:

- the legacy homes and the flag module
- the top-level `"xprompts"` → `"config"` migration and its test
- the unconditional sub-tab and statistics-view readers
- legacy-input fixtures
- pre-flip response keys still emitted by core, each pointed at `sase-1eq.10`
- the `legacy_xprompt_syntax` flag prose in the schema

A new hit fails. Do not allowlist a whole file to hide a missed rename.

Run the existing j/k navigation benchmark,
`pytest -s -m slow tests/ace/tui/bench_tui_jk.py`, through `/sase_monitor` if it will
not finish in the turn. This epic adds no navigation work, so a regression is a defect
to fix, not a baseline to accept. Read `tui_perf.md` before touching a handler to chase
a number.

Exit: `sase tool run check` passes, the widened guard passes, and the golden report has
been inspected. Record `PROPOSED FOLLOW-UP:` notes for any image, wire, or machine step
this epic deferred. Do not close the parent phase or the parent epic.

## Done when

- TUI modules, CSS, copy, keymap actions, and Admin Center ids use macro spellings,
  except the classified legacy readers.
- The preview tab reads `RAW PROMPT` and the agent headings read `AGENT RAW PROMPT`.
- Retired keymap actions work only while `legacy_xprompt_syntax` is on, and both
  spellings together are an error.
- A stored top-level Admin Center id `xprompts` still opens Config. A stored sub-tab or
  statistics view id `xprompts` opens Macros.
- Stats requests send the macro keys core accepts, and response reads still accept the
  pre-flip body.
- The 34 goldens have macro names, the screenshot report was inspected, and the
  terminology guard covers the TUI scope.
- The j/k benchmark shows no navigation regression.
- No phase closed `sase-1eq` or `sase-1eq.5`.
