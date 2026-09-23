---
tier: tale
title: Hide the idle dispatch Target/Source line above the prompt input
goal:
  The prompt input bar no longer shows the "Target here local  Source <project>" line
  for ordinary local prompts; it still appears when a %dispatch selector is present or
  invalid.
size: small
proposed_by: bbugyi200.athena.0pt
create_time: 2026-09-23 09:34:00
status: wip
---

# Hide the idle `Target here local  Source <project>` prompt context line

## Problem

Whenever at least one remote machine is enrolled for remote dispatch, the ACE prompt
input bar shows a one-line strip above the prompt text area that reads, for a plain
local launch:

```
Target here local  Source home
```

It is rendered by `PromptInputBarDispatchMixin._dispatch_context_text()` in
`src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py` into the
`#prompt-dispatch-context` `Static` (composed in `_prompt_input_bar_lifecycle.py`). The
visibility rule at the end of `_dispatch_context_text` is:

```python
visible = bool(
    scan is not None
    or self._dispatch_target_rows
    or self._dispatch_preflight_override is not None
)
```

The `or self._dispatch_target_rows` clause keeps the line visible on every prompt once
the machine catalog loads a non-empty set of enrolled aliases, even when the prompt has
no `%dispatch` directive. In that state the line only restates the default (launch here,
from the current project), so it adds no information, distracts, and takes a row of
vertical space from the prompt.

## Goal

Show the context line only when it carries dispatch-specific information:

- the prompt contains a `%dispatch` selector (`scan is not None`): keep today's full
  `Target <alias> <status>  Source <project>` line, plus any preflight override or the
  `proof checked on submit` hint; or
- the dispatch directive fails to parse (`DirectiveError`): keep today's
  `Target error  <message>` branch, which already returns `visible=True`.

For an ordinary local prompt with no `%dispatch` selector, hide the line entirely, even
when remote targets are enrolled. Launch behavior, the `gD` / `Ctrl+G D` Launch Target
picker, the catalog warm-up, and the submit-time preflight do not change.

## Implementation

1. **`src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py`**, in
   `_dispatch_context_text`:
   - Replace the three-clause `visible = bool(...)` expression with
     `visible = scan is not None`. The `DirectiveError` early return stays as it is
     (visible, error severity).
   - Better still, return early right after the successful scan when `scan is None` (for
     example `return Text(), "ok", False`), before computing target/source text, so the
     idle path does no styling work. Keep the rest of the function unchanged for the
     `%dispatch` case.
   - Effect on the catalog-unavailable warning: `_apply_dispatch_target_catalog` stores
     it as a prompt-agnostic override (`prompt == ""`). It now shows only while a
     `%dispatch` selector is present, which is the only time the catalog matters. Every
     code path that sets a prompt-scoped override
     (`_maybe_preflight_dispatch_submission` and `_complete_dispatch_preflight`) runs
     only when a directive exists or failed to parse, so those remain visible. Do not
     change how overrides are stored.
   - Update the `_refresh_dispatch_context_line` docstring if needed so it says the line
     is shown only for prompts that route through `%dispatch`.
   - No change is needed to `_hide_dispatch_context_line`, the height accounting in
     `_prompt_input_bar_stack_lifecycle.py` (it already reads
     `_dispatch_context_visible`), or `styles.tcss`.

2. **Tests**: add focused coverage in `tests/ace/tui/widgets/` (either extend
   `test_dispatch_target_picker_focus.py`, reusing its `DispatchPickerFocusApp`,
   `_seed_remote_targets`, and `no_catalog_worker` fixture, or add a sibling module such
   as `test_prompt_dispatch_context_line.py` that follows the same pattern):
   - With remote targets seeded and a prompt that has no `%dispatch` (such as
     `#gh:sase`), after `bar._refresh_dispatch_context_line()` the
     `#prompt-dispatch-context` Static has the `hidden` class and
     `bar._dispatch_context_visible is False`. This is the regression test for the
     reported text.
   - Same as above, but also with a prompt-agnostic catalog-unavailable override set
     (`bar._dispatch_preflight_override = ("target catalog unavailable: boom", "warning", "")`):
     the line stays hidden.
   - With `%dispatch:apollo` in the prompt and seeded targets, the line is visible and
     its rendered text contains `Target`, `apollo`, and `Source`.
   - With a malformed or duplicate dispatch directive (for example two `%dispatch:`
     selectors), the line is visible, has the `error` class, and contains
     `Target error`.
   - Optionally, a direct unit test of `_dispatch_context_text` returning
     `visible=False` for a directive-free prompt.

3. **Docs** (keep wording short, no other doc restructuring):
   - `docs/ace.md` ("Launch Target Picker" section, the sentence starting "The prompt
     context line shows the cached Target and Source"): say the context line appears
     only while the pane has a `%dispatch` selector (or an invalid one) and is hidden
     for ordinary local launches.
   - `docs/remote_dispatch.md` (the paragraph ending "The prompt's Target/Source context
     line makes the selected owner and portable source explicit before submission"):
     same clarification, stating the line appears once a remote is selected.

## Non-goals

- Do not remove the `#prompt-dispatch-context` widget, the dispatch mixin, the Launch
  Target picker, or submit-time source preflight. They remain useful when a remote
  target is actually selected.
- No keymap, `default_config.yml`, CLI, or Rust-core changes. This is presentation-only
  TUI behavior and stays in this repo.
- No new config toggle or feature flag. The idle line has no value, so no user needs it
  back.

## Verification

- Run the new or updated tests plus the existing
  `tests/ace/tui/widgets/test_dispatch_target_picker_focus.py` and the
  `tests/ace/tui/widgets/test_prompt_input_bar_*.py` modules.
- Run any prompt-bar PNG visual snapshot tests under `tests/ace/tui/visual/` that
  exercise the prompt input bar. They should be unaffected, because snapshots do not
  seed enrolled dispatch targets. If one changes, inspect it and regenerate it only if
  the only difference is the removed idle line.
- Follow the repo's lint-and-test memory note (`just check` through the sase tool
  control plane) before finishing.
