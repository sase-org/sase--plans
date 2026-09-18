---
tier: tale
title: Move prompt completion acceptance from Enter to Ctrl+G
goal:
  Prompt completion selection uses Ctrl+G without risking accidental prompt submission,
  while existing prompt-prefix and compatibility behavior remains intact.
size: medium
proposed_by: bbugyi200.athena.0n8
create_time: 2026-09-18 15:19:20
status: wip
---

# Move prompt completion acceptance from Enter to Ctrl+G

## Goal

Make prompt-input completion selection explicit and safe: while a completion menu is
active, `Ctrl+G` accepts the highlighted row (including the untouched first row), while
`Enter` always follows the prompt's normal submit or submit-chooser path. Preserve all
non-completion `Ctrl+G` prefix behavior and the existing `Ctrl+L` compatibility path.

## Behavior contract

| Context                                         | `Ctrl+G`                                                                           | `Enter`                                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Completion menu active in prompt INSERT mode    | Accept the highlighted candidate and do not open the `Ctrl+G` prefix hints         | Submit the prompt as typed, or open the existing stack/target submit chooser |
| No completion menu active in prompt INSERT mode | Start the existing prompt-local `Ctrl+G ...` continuation prefix                   | Submit through the existing path                                             |
| Prompt NORMAL mode                              | Preserve the existing prompt-local `Ctrl+G ...` prefix                             | Preserve normal `Enter` and `g<Enter>` behavior                              |
| Soft subtitle completion only                   | Preserve `Ctrl+L` acceptance; do not turn `Ctrl+G` into a soft-completion shortcut | Submit the prompt as typed                                                   |

`Ctrl+L` remains an accepted compatibility key for an open manual menu and remains the
soft-completion key. This change does not alter configurable app keymaps: the contextual
prompt handler already shadows the app-level `Ctrl+G` binding while the prompt has
focus, so `src/sase/default_config.yml` should remain unchanged.

## Implementation

1. Update prompt key-event priority in
   `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`.
   - Before the INSERT/NORMAL `Ctrl+G` prefix dispatchers run, consume `Ctrl+G` when a
     manual completion menu is active, prevent the default event, and route it through
     the existing `_accept_file_completion()` path. This must accept the initially
     highlighted row without requiring `Ctrl+N`, `Down`, or another ownership signal.
   - Remove completion acceptance and the bare-`@` / untouched automatic-xprompt-arg
     exceptions from the `Enter` branch. After preserving the existing NORMAL-mode
     `g<Enter>` check, let `Enter` use only the current single-prompt submit or
     stack/target submit-chooser logic; those paths already clear transient completion
     state.
   - Keep the active-menu `Ctrl+L` branch and soft-completion behavior intact. Keep an
     inactive `Ctrl+G` routed to `_handle_insert_g_prefix_key()` /
     `_handle_normal_g_prefix_key()` so editor entry and every continuation remain
     unchanged.

2. Align code comments and visible completion hints with the new primary key.
   - Update stale acceptance descriptions such as the shared word-completion comment in
     `src/sase/ace/tui/widgets/_file_completion_base.py`.
   - Replace completion-specific `Enter` affordances in `model_alias_completion.py`,
     `model_explicit_completion.py`, and `_prompt_input_bar_completion_panel_labels.py`
     with `Ctrl+G` / `^G` wording, including provider drill-down and insertion previews.
     Continue to distinguish these menu hints from the ordinary `[Enter] send` prompt
     subtitle and from the `[^L]` soft-completion subtitle.
   - Update affected hint assertions and deterministic PNG fixtures; regenerate only
     goldens whose rendered key text intentionally changes, and inspect their actual and
     diff PNGs before accepting them.

3. Convert and extend interaction coverage across the shared completion surface.
   - Change existing menu-acceptance interactions from `Enter` to `Ctrl+G` in the
     focused prompt widget tests for file, xprompt, xprompt-argument, directive/model,
     placeholder, prompt-local word, and history-word providers. Retain
     provider-specific assertions for replacement bounds, cursor placement, chained
     menus, directory or provider drill-down, xprompt spacers/hints, and non-selectable
     loading rows.
   - Add explicit regressions showing that `Ctrl+G` accepts the first untouched row for
     automatically opened menus (especially bare `@` and an empty xprompt argument),
     accepts a row selected with `Ctrl+N`/`Down`, and never reveals the `Ctrl+G` prefix
     panel in those cases.
   - Add regressions showing that `Enter` with a regular manual or automatic completion
     menu open submits the exact unexpanded text (or opens the existing stack chooser),
     clears the menu, and never accepts a candidate even after the highlight moved.
   - Keep or add coverage that `Ctrl+G` without an active menu still opens the prefix
     hints and dispatches its existing second-key actions, and that `Ctrl+L` still
     accepts both manual-menu and soft subtitle completions.

4. Update `docs/ace.md` so the completion key table and the bare-`@` / automatic-menu
   explanations say that `Ctrl+G` accepts a highlighted candidate and `Enter` submits.
   Preserve the documented `Ctrl+L` soft-completion behavior and mention it as the
   retained manual-menu alias where the manual key table is authoritative. No Rust-core,
   schema, or configuration change is needed because this is prompt-widget presentation
   and event routing only.

## Verification

1. Run the focused prompt-widget tests covering the key dispatcher, `Ctrl+G` prefix,
   submit chooser, and every completion provider touched by the interaction migration.
2. Run the affected model/completion visual modules once to produce intentional
   failures, inspect `.pytest_cache/sase-visual/` actual/expected/diff artifacts, update
   the selected snapshots with
   `just test-visual -- --sase-update-visual-snapshots <test paths>`, then rerun the
   same visual selection without update mode.
3. Run `just fix`, review that formatting or generated-doc work did not introduce
   unrelated changes, and finish with `just check`. If dependency drift prevents the
   gates from starting, run `just install` and retry.
4. Manually exercise a prompt with an automatically opened first-row completion and a
   manually navigated completion: observe `Ctrl+G` insertion, `Enter` submission without
   insertion, and inactive-menu `Ctrl+G ...` prefix continuity. For any captured visual
   evidence, inspect the rendered PNG itself rather than only its existence.
