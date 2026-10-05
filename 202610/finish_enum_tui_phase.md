---
tier: tale
title: Finish and close sase-1g4.3 (enum TUI) to unblock the sase-1g4 epic
goal:
  Inline enum inputs are authorable losslessly in the frontmatter panel, the choices
  parser and picker are correct, phase-3 acceptance coverage exists, check is green, and
  sase-1g4.3 is closed so sase-1g4.6, sase-1g4.7, and the lander resume.
size: medium
proposed_by: bbugyi200.athena.0x4
create_time: 2026-10-05 16:57:32
status: wip
---

# Plan: Finish and close sase-1g4.3 so the sase-1g4 epic can resume

## Why the epic is stuck

Epic **sase-1g4** ("Named macro input types") has phases 1, 2, 4, and 5 closed. Phase
**sase-1g4.3** (`tui-enum`) is still `in_progress`. Phase **sase-1g4.6** (`model-tui`)
waits on it, phase **sase-1g4.7** (`adopt`) waits on 6, and the `sase-1g4.land` agent
waits on all seven. All three runners are parked in `WAITING` and will start on their
own when their bead dependencies close. Do not relaunch or touch them, and never close
the epic bead.

The phase's code landed on master in `a958ba4e87` ("feat(tui-enum): route enum and bool
args through Rust choice builder with picker and modal editors"). Its last worker,
`sase-1g4.3--2`, ended with `bead_action: keep`. Its joined check had all lint gates
green, but the escalated full test lane reported 2 NEW and 2 KNOWN failures:

- `test_prompt_tab_focus_steal.py`
- `test_deck_block_spread_pilot.py::test_block_spread_bracket_top_aligns`
- `test_launch_context_source.py::test_every_tick_rebroadcasts_to_mounted_views`

All three pass in isolation on current master (35 passed). `3569571a73` has since
hardened `launch_context_source` and AcePage teardown. Nobody was assigned to resolve
the red check, so the bead has sat open.

An audit of the landed work against the phase plan (`plan:202610/macro_enum_tui.md`)
found that the prompt-bar value menus, routing, PNG goldens, and golden-corpus parity
are done. However, the phase cannot honestly be closed yet. It has these real gaps, each
confirmed by reading the code:

1. **Enum choices cannot be authored on the live surface.** No production code opens
   `InputItemModal` or `MacroItemModal`; the frontmatter panel replaced them with inline
   cell editing in `7776f7a857`. The panel's input cell edit (`_begin_cell_edit` in
   `src/sase/ace/tui/widgets/_frontmatter_panel_cell_editing.py`) has only the cells
   `name, type, default, description`. As a result:
   - a new inline `enum` input cannot be created there;
   - `_parse_compact_inputs` rejects `name:enum` in a local macro's compact inputs with
     "edit it in the input modal", a modal the user cannot open;
   - because `_format_compact_inputs` writes `status:enum` for an existing inline-enum
     input, any edit to such a local macro fails;
   - `_format_compact_inputs` also drops input descriptions, repeatability, and roles.
2. **The choices parser drops validator errors.** `_parse_choices_yaml` in
   `src/sase/ace/tui/modals/input_item_modal.py` ignores the `issues` list that the Rust
   `validate_enum_choices` binding returns; it reports errors there rather than raising.
   Probed results:
   - `[a, a]` is de-duplicated silently;
   - `- yes` (a YAML boolean) and `- ""` vanish;
   - `[on, off]` becomes `()`.

   Separately, `_choices_to_yaml` writes values unquoted, so `on`, `off`, and `yes` do
   not survive a round trip.

3. **Frontmatter panel rows show the base type.** In `_frontmatter_panel_rendering.py`
   (the input row renderer and the `type_width` calculation), a named type such as
   `effort` renders as `enum`.
4. **The enum choice picker has bugs**
   (`src/sase/ace/tui/modals/enum_choice_picker_modal.py`):
   - `j`/`k` move the selection, so those letters cannot be typed into the filter;
   - only the first `_VISISBLE_ROWS` (12) rows ever render, so with 13 or more choices
     the selection disappears;
   - a broad `except Exception` fallback reimplements filtering in Python, which
     violates `decisions:rust-core-required`.
5. **The help popup is not updated.** `PROMPT_INPUT_SECTION` in
   `src/sase/ace/tui/modals/help_modal/binding_common.py` has no enum/bool value-menu
   entry.
6. **Acceptance coverage is missing.**
   - `tests/ace/tui/widgets/test_macro_arg_value_completion.py` was not extended: it has
     no `#pr(x, status=` menu, chaining, or enum filtering test.
   - The typed-form six-choice test never opens the picker to accept or cancel.
   - No test covers enum authoring, invalid-choice feedback, named-type preservation, or
     a save/reopen round trip.

This plan closes those gaps, verifies, and closes sase-1g4.3. Phases 6 and 7 then start
automatically.

## Ground rules

- Read beads with `sase bead read <id> --no-links -r "<why>"`. The linked read currently
  fails with
  `validation: bead id segment must contain only letters, digits, '-' and '_'`.
- Read the audited sources with `sase artifact read`:
  - `plan:202610/macro_enum_tui.md` (the phase plan, "Implementation" sections 4–6 and
    "Acceptance and verification");
  - `plan:202610/macro_named_input_types.md` (the epic contract).
- Read these memories with `/sase_memory_read`:
  - `tui.md` and its children, before touching TUI code and visual snapshots;
  - `lint_and_test.md`, before verifying;
  - `sase_beads.md`, before closing or noting beads.
- Read `src/sase/ace/AGENTS.md` for help-popup and keymap conventions.
- Shared behavior stays in sase-core. Use the existing Rust bindings only:
  - `validate_enum_choices`
  - `macro_argument_choice_candidates`
  - `macro_input_type_label`
  - `input_type_schema` / the input type catalog

  YAML parsing and cell text are Python glue. No sase-core change or
  `sase-core-revision.txt` move is expected. If one turns out to be necessary, follow
  `docs/rust_backend.md` and say so explicitly in the final declaration.

- Do not change the prompt-bar value-menu title format or regenerate its goldens. The
  existing Rust-label title (`status · wip | draft | ready`) stays, and the deviation is
  recorded as a follow-up (see Step 8).
- Do not modify phases 6/7, their runners, or the epic bead.

## Step 1 — One shared, lossless choices-text helper

Add a small typed module beside the panel widgets, for example
`src/sase/ace/tui/widgets/_input_choices_text.py`, with:

- `parse_choices_text(text: str, *, name: str) -> tuple[InputChoice, ...]`. It should:
  1. YAML-safe-load the text; it may be a block list or a single-line flow list such as
     `[wip, draft, {value: ready, label: Ready, description: Ship it}]`.
  2. Require a non-empty list.
  3. Call `require_rust_binding("validate_enum_choices")({"items": raw})`.
  4. Raise `MacroValidationError` that joins every `issues` entry with severity `error`.
     Never silently drop, de-duplicate, or coerce. Rust already reports booleans, ints,
     nulls, empty values, whitespace, and duplicates.
  5. Build `InputChoice` values from the returned `choices`.
- `format_choices_text(choices, *, flow: bool) -> str`. Serialize with `yaml.safe_dump`,
  so values like `on`, `off`, `yes`, `null`, and `1` are quoted.
  - Scalars stay bare strings when the choice has no label or description; otherwise use
    `{value, label, description}` mappings.
  - `flow=True` produces one line for the panel cell; `flow=False` produces the
    multiline form for `InputItemModal`.
  - `parse_choices_text(format_choices_text(c, flow=...)) == c` must hold for both
    forms.

Make `InputItemModal` use these helpers in place of `_choices_to_yaml` and
`_parse_choices_yaml`. Delete the private copies; do not keep wrappers.

## Step 2 — Author inline enum choices in the frontmatter panel

In `_frontmatter_panel_cell_editing.py`, the input cell edit gains a `choices` cell, so
the cells become `name, type, choices, default, description`.

- **Prefill.** For an existing inline enum (`type is enum` and `named_type is None`),
  prefill with `format_choices_text(arg.choices, flow=True)`. For every other type,
  start empty.
- **Build (`_build_cell_result`).**
  - When the type resolves to an inline `enum`, the choices text is required and parsed
    with `parse_choices_text`.
  - When it resolves to anything else (including named enums, whose choices come from
    the catalog), non-empty choices text is an error naming the rule.
  - Keep preserving `repeatable` and the resolved `value_role` from the existing input.
  - Remove the current "reuse `existing.choices`" special case; the prefilled cell now
    carries them explicitly.
  - Validate the default against the resulting choices, as today.

  Errors must surface through the existing `_refresh_cell_feedback` /
  `_build_cell_result` path, and nothing may raise out of a keystroke.

- **Rendering.** In `_frontmatter_panel_rendering.py`, the edit row renders the new cell
  consistently with the others. An empty `choices` cell on a non-enum type should render
  as a quiet dim placeholder, not a blank that looks broken. Keep narrow widths and both
  themes readable.
- **Type column.** The non-edit input row shows `arg.named_type or arg.type.value`, and
  `type_width` uses the same text, which fixes gap 3.
- **Local macro compact inputs.** Make `_parse_compact_inputs` preserve richer
  declarations instead of rejecting or flattening them:
  - Pass it the edited macro's existing inputs: from the model for an existing local
    macro, or from the `prefill` macro when `begin_prefilled_macro` started the edit.
    Store whichever applies on `_CellEdit`.
  - For each compact chunk whose name matches an existing input with the same canonical
    type spelling (`named_type` or base; an inline enum spells `enum`), start from a
    copy of that existing `InputArg`. That keeps its choices with labels and
    descriptions, description, repeatability, and role.
  - Apply only the compact default: `=value` sets it, and its absence clears it. This
    matches `_format_compact_inputs`, which always writes an existing default.
  - Keep input order as written.
  - A genuinely new inline `enum` (no matching existing input) is still rejected. Its
    message must point at a real path: declare it as a top-level input, or use the
    panel's raw-YAML editing (look up the actual key or flow in the panel code and docs
    and name it). Never mention "the input modal".
- **Docs.** Update the `docs/ace.md` section that documents the frontmatter panel's
  input/macro sub-item cell editing to describe the `choices` cell and its flow-list
  syntax, and that compact local-macro inputs preserve existing rich declarations.

## Step 3 — Fix the enum choice picker

In `enum_choice_picker_modal.py`:

- Remove `j`/`k` from the movement keys so they become filter text; keep `↑`/`↓` and
  `Ctrl+P`/`Ctrl+N`.
- Window the rendered rows so the selected row is always visible: keep a scroll offset,
  clamp it on move and refilter, and reset it sensibly on reopen-at-current. When more
  rows exist than fit, show a quiet position indicator such as `7/40` in the footer or
  query line.
- Rename `_VISISBLE_ROWS` to `_VISIBLE_ROWS`.
- Replace the `except Exception` Python filtering fallback with the shared adapter in
  `src/sase/ace/tui/widgets/_macro_arg_choice_adapter.py` (or the direct binding call
  through `hint_to_wire`, as today). Rust failures propagate; no Python reimplementation
  of filtering or ordering.
- Run targeted visual capture for `typed_form_enum_choice_picker_*`. Update the golden
  only if the footer intentionally changes, and inspect it.

## Step 4 — Help popup

Add a `PROMPT_INPUT_SECTION` entry in
`src/sase/ace/tui/modals/help_modal/binding_common.py`, for example
`("#m(k= / #m:", "Enum/bool value menu")`, describing the enum/bool value menus. Follow
`src/sase/ace/AGENTS.md`, keep the text within the existing help-box width tests, and
update the help-modal snapshot or golden only if a test requires it. No new configurable
keybinding is introduced, so `src/sase/default_config.yml` should not change. Confirm
that.

## Step 5 — Acceptance tests (behavioral, not implementation-mirroring)

Extend existing suites; add a focused companion file only where a suite would get
unwieldy. Stay under the `toobig` thresholds (700/850/1000).

1. **Value menus** (`tests/ace/tui/widgets/test_macro_arg_value_completion.py`, or the
   detection/live-completion suites it already pairs with). Use a catalog fixture with
   an enum input that has labels and descriptions, or the dogfooded `#pr` status enum if
   the existing fixtures already load it. Prove:
   - `#pr(x, status=` opens `wip`/`draft`/`ready` in declared order with descriptions
     and the default badge;
   - accepting the `status=` name row chains into the same value menu;
   - typing a prefix filters in Rust order;
   - accepting replaces only the active value and preserves a following `, other=1)`
     suffix;
   - bool inputs still offer `true`/`false`.
2. **Typed form and picker** (`tests/ace/tui/test_typed_input_form.py` plus picker
   tests). With six choices, open the picker through the form (not by pushing the modal
   directly) and check:
   - accept updates the field to the canonical value and emits `Changed`;
   - cancel leaves the value intact;
   - reopen highlights the current value.

   In the picker itself:
   - typing `j` filters rather than moving;
   - with 40 choices, moving to the last row keeps it rendered and the position
     indicator updates.

3. **Choices text helper** (unit tests). Cover:
   - every Rust issue class surfaces as `MacroValidationError`: duplicate, YAML boolean,
     int, null, empty, whitespace, and an empty list;
   - lossless round trips in both flow and block form for `on`, `off`, `yes`, labels,
     descriptions, and non-ASCII values.
4. **Panel authoring** (`tests/ace/tui/widgets/test_frontmatter_panel_subeditors.py` and
   its helpers). Cover:
   - creating a new inline enum input via the `choices` cell commits the expected
     `InputArg`;
   - empty, duplicate, and unquoted-boolean choices show feedback and do not commit;
   - a non-member default is rejected;
   - choices on a non-enum type are rejected;
   - a named-type input shows its named type in the row and survives editing its
     description;
   - editing the description of an inline enum input with labels, descriptions, and
     `repeatable` keeps all of them;
   - a local macro with an inline-enum input can have its content edited and keeps its
     choices;
   - adding a new `x:enum` compact input is rejected with the new message;
   - save, then reopen through the model/serializer, round-trips the full choice
     metadata.
5. **Existing modal tests** (`tests/ace/tui/modals/test_frontmatter_item_modals.py`)
   still pass with the shared helper. Add one case proving `InputItemModal` surfaces a
   duplicate-choice error instead of silently de-duplicating.

## Step 6 — Visual snapshots

The panel edit row and possibly the picker footer change. Read `tui.md`'s screenshot
guidance first. Then run a targeted `just fix-tui-screenshots -- <selectors>` covering:

- the frontmatter panel/input snapshots, including `frontmatter_input_item_modal_120x40`
  if affected;
- the gate input panel snapshots touched by `a958ba4e87`;
- `typed_form_enum_choice_picker_*`.

Inspect the retained manifest and warnings, and view every created, removed, and updated
golden. A `partial` result is not approval. Run long captures through `/sase_monitor`,
using a concrete follow-up that inspects the results.

## Step 7 — Verify

1. Run `just fix` inline, then **`sase tool run check`**. Do not run `just check-full`.
   Treat UNKNOWN failures as yours and fix them.
2. If the test lane escalates and the only failures are unrelated full-lane flakes, the
   bead may still close. Unrelated means: in files this plan did not touch, and of the
   same kind as the earlier `test_prompt_tab_focus_steal`,
   `test_deck_block_spread_pilot`, or `test_launch_context_source` failures. For each
   such NEW failure:
   1. Rerun it in isolation twice on this tree, and confirm it passes.
   2. Use `/sase_new_task` to corroborate an existing `flake` task, or file one, with
      the ToolRun id and isolation evidence.
   3. Cite that bead in the close note.

   Do **not** leave sase-1g4.3 open solely because of such flakes; that is exactly what
   stalled this epic.

## Step 8 — Record follow-ups and close the phase

1. Append `PROPOSED FOLLOW-UP:` notes on sase-1g4.3 for the epic lander, each as one
   `sase bead note`:
   - **Title deviation:** the prompt value-menu title uses the Rust type label rather
     than the plan's `<input> · <named_type or enum>`.
   - **Type labels:** type-label wire serialization is split across
     `_macro_arg_choice_adapter.py`, `macro/_catalog_format.py`, and `highlight.py`.
   - **Detection:** the Rust-span overlay in `_macro_arg_assist_detection.py` still
     picks the active input with the old comma splitter and a 64-byte window.
   - **Dead modals:** `InputItemModal` and `MacroItemModal` are unreachable from
     production code; delete them or wire them in.
2. On epic bead sase-1g4, add a note that its `DISCOVERED ISSUE` (an `InputItemModal`
   NameError on `InputType`) was resolved by `a958ba4e87`. Cite the test that constructs
   `InputItemModal()` with no `existing`.
3. Use `/sase_new_task` to corroborate or file the bead linked-read defect: on this
   tree, `sase bead read sase-1g4 -r ...` fails with
   `validation: bead id segment must contain only letters, digits, '-' and '_'`, while
   `--no-links` succeeds. It was hit by both the phase-3 planner and the epic triage.
4. Close sase-1g4.3 once every step above is complete and verified.
   - If your final-declaration context assigns you **sase-1g4.3**, close it through the
     declaration's `bead_action: "close"` on the primary repository.
   - Otherwise, run
     `sase bead close sase-1g4.3 --note "<what was fixed and how it was verified, ToolRun id, flake beads if any>"`
     immediately before `/sase_final`.

   Never close sase-1g4 itself.

## Done when

- Inline enum inputs can be created and edited from the frontmatter panel without losing
  choice metadata. Local macros with enum inputs remain editable, and named types
  display correctly.
- The choices parser surfaces every Rust validation issue and round-trips losslessly.
- The picker filters on every printable key and keeps its selection visible for any
  number of choices.
- The help popup and `docs/ace.md` describe the behavior.
- The new acceptance tests pass, and visual goldens are inspected and current.
- `sase tool run check` is green, or has only flakes documented in task beads as in
  Step 7.
- sase-1g4.3 is closed with an evidence note, so sase-1g4.6 starts on its own.
