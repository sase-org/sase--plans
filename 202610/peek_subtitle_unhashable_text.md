---
tier: tale
title: Fix the Escape crash from hashing the unhashable peek subtitle Text
goal:
  Pressing Escape (or hiding any completion panel) while the mid-sentence next-word peek
  is visible no longer crashes sase tui with an unhashable rich Text TypeError. The peek
  clears normally, and the model-shortcut subtitle restore works as before.
size: small
proposed_by: bbugyi200.athena.0uu
create_time: 2026-10-01 11:34:14
status: wip
---

# Fix the `sase tui` crash when you press Escape while a next-word peek is visible

## Symptom

`sase tui` crashes when you press `Escape` in an ACE prompt while the mid-sentence
next-word peek (`⇢ review it  [^T] word  [^L] all`) is showing in the prompt border:

```
_prompt_text_area_actions.py  _enter_normal_mode        -> self._clear_file_completion()
_file_completion_base_panel.py _clear_file_completion    -> self._update_file_completion_panel("")
_file_completion_base_panel.py _update_file_completion_panel -> bar.hide_file_completions()
_prompt_input_bar_completion_panel.py hide_file_completions ->
    if self._subtitle_base in {MODEL_ALIAS_MODE_SUBTITLE, MODEL_EXPLICIT_MODE_SUBTITLE}:
TypeError: cannot use 'rich.text.Text' as a set element (unhashable type: 'Text')
```

The `-vim-normal -read-only` classes in the traceback locals don't mean the prompt was
already in NORMAL mode. `PromptTextArea._enter_normal_mode` calls
`super()._enter_normal_mode()` first, and that call flips the classes before it reaches
the prompt-specific teardown that crashes.

## Root cause

Commit `7255e8cd06` ("mid-sentence next-word peek in the prompt border", sase-1dq.5)
widened `PromptInputBar._subtitle_base` from `str` to `str | Text`.
`PromptInputBarCompletionMixin.show_next_word_hint` now stores the styled peek
`rich.text.Text` there, so the violet glyph and word spans survive subtitle composition.

`rich.text.Text` defines `__eq__` but not `__hash__`, so its instances are unhashable.
In `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`, two places still
test membership against a **set literal**. A set membership test hashes the left
operand:

1. `hide_file_completions` (around line 293). This is the crashing site.
2. `show_file_completions`, in the `elif self._subtitle_base in {...}` branch (around
   line 271). It has the same latent bug. Today the next-word menu calls
   `_hide_next_word_hint()` before it opens, and every other menu clears the next-word
   chain first, so this branch normally sees a `str`. It is still one ordering change
   away from the same crash.

On Escape, `PromptTextArea._enter_normal_mode` calls `_clear_file_completion()` (→
`hide_file_completions`) _before_ `_clear_next_word_chain()`. That second call is the
one that would clear the peek. So whenever the peek is visible, `_subtitle_base` is
still the peek `Text` at the moment of the set-membership test. Any other caller of
`hide_file_completions` while a peek is visible crashes the same way.

### Reproduction (verified while planning)

A pilot test using the existing `NextWordTestApp` harness from
`tests/ace/tui/widgets/test_prompt_next_word.py` and `_patch_confident` from
`tests/ace/tui/widgets/test_next_word_peek.py` reproduces the crash both ways:

- Load `"Can you help me the plan"`, put the cursor after `"Can you help me"`, patch a
  confident `["review", "it", "now"]` prediction, call `ta._arm_next_word_chain()`, and
  `await pilot.pause()`. `_next_word_peek_visible()` is now `True` and
  `bar._subtitle_base` is a `Text`. Then `await pilot.press("escape")` raises the
  `TypeError` above.
- With the same setup, a direct `bar.hide_file_completions()` raises the same
  `TypeError`.

## Fix

Fix the membership test at its root rather than reorder `_enter_normal_mode`. Reordering
would only hide one path and leave `hide_file_completions` and `show_file_completions`
unsafe for any other caller.

### `src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`

1. Add a module-level constant beside the other module constants:
   `_MODEL_SHORTCUT_SUBTITLES = frozenset({MODEL_ALIAS_MODE_SUBTITLE, MODEL_EXPLICIT_MODE_SUBTITLE})`.
   Add a small private predicate:

   ```python
   def _is_model_shortcut_subtitle(base: str | Text) -> bool:
       """Whether *base* is a model-completion shortcut subtitle.

       ``base`` may be a styled peek ``Text``, which is unhashable, so only
       plain strings are looked up in the set.
       """
       return isinstance(base, str) and base in _MODEL_SHORTCUT_SUBTITLES
   ```

2. Replace both set-literal membership tests with
   `_is_model_shortcut_subtitle(self._subtitle_base)`:
   - the `elif self._subtitle_base in {...}:` branch in `show_file_completions`
   - the `if self._subtitle_base in {...}:` block in `hide_file_completions`

   Keep both branch bodies as they are: restore `self._mode_subtitle` and re-render.

Behavior after the fix:

- With a peek `Text` as the base, `hide_file_completions` leaves the subtitle alone,
  because a peek is not a model shortcut subtitle. In the Escape path,
  `_enter_normal_mode` then calls `_clear_next_word_chain()`, which clears the peek and
  restores the mode subtitle through `hide_next_word_hint`. Escape therefore ends in
  NORMAL mode, with no peek and the normal mode subtitle.
- The model alias and model explicit subtitle restore flows don't change. Those
  subtitles are always plain `str`.

Before editing, grep `src/sase` once more for any other hashing of `_subtitle_base`,
`_next_word_hint`, or `_next_word_hint_text`: set or dict membership, dict keys,
`lru_cache` or `cache` arguments, `hash(...)`. Fix any hit the same way. During planning
the only hits were the two sites above. No `lru_cache` or `cache` decorators exist in
the next-word or prompt-input-bar modules, and the gate note surface
(`src/sase/ace/tui/modals/gate_input_panel_note.py`) assigns the hint straight to
`border_subtitle` without hashing it.

This is presentation-only Textual glue, so nothing crosses the Rust core boundary. No
keymap or config changes are needed.

## Tests

Add regression tests to `tests/ace/tui/widgets/test_next_word_peek.py`. It already has
the `_chain_app()` and `_patch_confident` helpers, and it is far below the 700-line
toobig threshold.

1. `test_escape_with_visible_peek_enters_normal_mode` (pilot, chain mode):
   - Arm a visible peek as in the existing `test_ctrl_t_after_reveal_*` tests and assert
     `ta._next_word_peek_visible() is True`.
   - Assert `isinstance(bar._subtitle_base, Text)`. This pins the precondition that
     caused the crash.
   - `await pilot.press("escape")`, then `await pilot.pause()`.
   - Assert no exception, `ta._vim_mode` is NORMAL (use the same accessor or enum other
     vim tests use), `ta._next_word_peek_visible() is False`, and
     `bar._subtitle_base == bar._mode_subtitle`.
2. `test_hide_file_completions_preserves_visible_peek` (pilot):
   - With a visible peek, call `bar.hide_file_completions()` directly.
   - Assert it does not raise and `bar._subtitle_base` is still the same peek `Text`
     object (`is`). Hiding an unrelated completion panel must not clobber the peek.
3. A focused unit test of the predicate, `_is_model_shortcut_subtitle`. It returns
   `True` for both model subtitle constants, `False` for an arbitrary `str`, and `False`
   (without raising) for `Text(MODEL_ALIAS_MODE_SUBTITLE)` and an arbitrary `Text`.
   Symvision may flag a test that imports a private helper. If it does, assert the
   behavior through `bar.show_file_completions` / `hide_file_completions` with a `Text`
   base instead, and keep the predicate private.

Before you fix the code, confirm that tests 1 and 2 fail with the `TypeError`. After the
fix, confirm they pass. The existing model alias and explicit tests must still pass
unchanged: `tests/ace/tui/widgets/test_model_alias_completion_interactions.py`,
`test_model_explicit_completion_interactions.py`,
`test_model_alias_completion_catalog.py`, `test_model_explicit_completion_catalog.py`.

## Verification

- Targeted run:
  `pytest tests/ace/tui/widgets/test_next_word_peek.py tests/ace/tui/widgets/test_model_alias_completion_interactions.py tests/ace/tui/widgets/test_model_explicit_completion_interactions.py tests/ace/tui/widgets/test_model_alias_completion_catalog.py tests/ace/tui/widgets/test_model_explicit_completion_catalog.py tests/ace/tui/widgets/test_next_word_menu.py tests/ace/tui/widgets/test_prompt_next_word.py`
- `just check`, run as the lint-and-test memory note directs. Do not run
  `just check-full`.
- No PNG golden changes are expected. The peek's rendering doesn't change; only the
  guard that hashed it does.
