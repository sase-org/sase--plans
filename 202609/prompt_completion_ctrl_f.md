---
tier: tale
title: Move prompt completion acceptance from Ctrl+E to Ctrl+F
goal:
  Prompt completion menus use Ctrl+F for highlighted-row acceptance while Ctrl+E
  consistently retains its readline end-of-line behavior.
size: medium
proposed_by: bbugyi200.athena.0r5
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0r5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0r5.md)
- **COMMITS:**
  - [307da2d](https://github.com/sase-org/sase/commit/307da2dacc1c5cfaf37c15cc69e4d60f1adf02fc)
    — feat(ace): accept completion menus with ctrl-f

# Plan: Move prompt completion acceptance from Ctrl+E to Ctrl+F

## Outcome and invariants

Migrate the prompt input widget's active manual-completion acceptance chord from
`Ctrl+E` to `Ctrl+F` without changing the rest of the prompt editing contract:

- With a manual completion menu open in INSERT mode, `Ctrl+F` accepts the highlighted
  row, including the initially highlighted row; `Ctrl+L` remains the compatibility
  acceptance alias.
- With no manual completion menu open, `Ctrl+F` keeps the inherited readline-style
  single-character forward motion from `VimTextArea`.
- `Ctrl+E` always keeps the inherited end-of-line motion, including while a manual
  completion menu is open; it must not accept or otherwise mutate the highlighted
  completion.
- Soft completion remains accepted by `Ctrl+L`; `Enter` still submits instead of
  accepting; `Ctrl+G` remains the prompt-local prefix; completion navigation and delete
  chords remain unchanged.
- This is a widget-local behavior change, not a configurable app keymap migration, so
  `src/sase/default_config.yml` and the app-level `Ctrl+F` bindings remain unchanged.

## Implementation

1. Update the active-manual-completion branch in
   `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` to consume `Ctrl+F` or
   `Ctrl+L` for acceptance and let `Ctrl+E` fall through to `VimTextArea`'s
   `cursor_line_end` binding. Rewrite the nearby rationale so it documents the
   menu-open/menu-closed behavior of both readline chords. Update the shared word
   completion contract comment in `src/sase/ace/tui/widgets/_file_completion_base.py` to
   name the new acceptance chord.

2. Replace every completion-menu hint that advertises `Ctrl+E` with `Ctrl+F`, including
   the alias and explicit-model mode subtitles in `model_alias_completion.py` and
   `model_explicit_completion.py`, plus provider drill down and model-expansion previews
   in `_prompt_input_bar_completion_panel_labels.py`. Keep unrelated uses of `Ctrl+E`
   elsewhere in the TUI intact.

3. Update `docs/ace.md` so the INSERT-mode table separates `Ctrl+E` end-of-line motion
   from conditional `Ctrl+F` completion acceptance/forward motion, and migrate all
   prompt-completion prose and the completion key table (xprompt arguments, model
   shortcuts, artifact references, and automatic bare-`@` menus) to `Ctrl+F` while
   continuing to document `Ctrl+L` as the retained alias. Do not rewrite documentation
   for unrelated `Ctrl+E` actions or generic readline controls.

4. Migrate completion-acceptance interactions from `ctrl+e` to `ctrl+f` across the
   focused prompt widget tests for files, artifact references, prompt/history words,
   placeholders, xprompts and argument values, and model alias/explicit-model menus.
   Update test names, comments, and visible-subtitle assertions accordingly. Make the
   replacements semantic rather than global: preserve `ctrl+e` fixture text that is a
   completion candidate, the standalone `VimTextArea` and `SingleLineVimTextArea`
   end-of-line tests, and the soft-completion regression proving `Ctrl+E` moves to line
   end without accepting.

5. Strengthen `test_prompt_file_completion.py` with explicit boundary coverage for the
   conflict being resolved: prove `Ctrl+F` accepts both the initial and a navigated
   manual row without submitting, prove it accepts at a mid-line cursor without moving
   to line end, prove `Ctrl+F` still moves one character when no manual menu is active,
   and prove `Ctrl+E` moves to line end while an active manual menu remains unaccepted.
   Assert text, cursor, menu state, submission state, and prompt-prefix state where
   relevant so app-level `Ctrl+F` bindings cannot silently steal the chord.

6. Update model-completion panel-title tests and the alias/explicit-model visual tests
   to expect `Ctrl+F` labels. Regenerate only the affected PNG goldens under
   `tests/ace/tui/visual/snapshots/png/` with targeted `just fix-tui-screenshots`
   selectors, then inspect the visual report and every changed golden to confirm the
   diff is limited to the intended key label and associated text reflow.

## Verification

1. Run the focused non-visual prompt-completion and panel-label pytest modules covering
   every migrated completion family plus the prompt-file conflict regressions. Include
   the generic and single-line Vim text-area modules to confirm the underlying
   `Ctrl+E`/`Ctrl+F` readline motions remain unchanged outside active manual completion.
2. Run the targeted alias and explicit-model PNG snapshot update/check workflow, inspect
   its retained report and all modified goldens, and rerun the targeted visual tests in
   check mode after accepting the intentional updates.
3. Audit source, tests, and `docs/ace.md` for stale completion-specific `Ctrl+E` labels
   or acceptance presses, distinguishing them from intentional end-of-line and
   unrelated-TUI uses.
4. Run `just fix` if formatting requires it, then run the recorded default repository
   gate with `sase tool run check`. Do not run `just check-full`; it is not requested
   and the targeted visual lane supplies the golden coverage excluded from `just check`.
