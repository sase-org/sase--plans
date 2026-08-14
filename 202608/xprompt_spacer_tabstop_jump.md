---
tier: tale
title: Drop the xprompt completion spacer when Tab jumps to the next tabstop
goal:
  Pressing Tab (or Shift+Tab) immediately after accepting a no-required-input xprompt
  completion inside a live snippet session deletes the completion's trailing spacer
  before moving to the neighboring tabstop, so users stop hand-deleting that space.
size: small
proposed_by: bbugyi200.athena.01n
create_time: 2026-08-14 13:56:10
status: done
---

- **PROMPT:**
  [prompts/202608/xprompt_spacer_tabstop_jump.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_spacer_tabstop_jump.md)

# Plan: Drop the xprompt completion spacer when Tab jumps to the next tabstop

## Problem

Xprompts with no required inputs complete to `#name ` — with a deliberate trailing
spacer — from all three acceptance paths (`Ctrl+T` panel accept / `<enter>`, `Ctrl+L`
soft completion, and the `#@` selector). That spacer is right when the user keeps typing
prose, but wrong when the accepted reference sits at a snippet tabstop and the user's
next keystroke is `Tab` to travel to the next tabstop: the reference is finished, the
surrounding template already supplies its own separators, and the spacer becomes a stray
space the user deletes by hand before pressing `Tab`.

Concrete repro (snippet template `Use $1 to fix $2.`):

1. Expand the snippet; the cursor lands on `$1`.
2. Type `#pl`, press `Ctrl+T` (or `<enter>` on the panel) and accept `#plain`. The
   document is now `Use #plain  to fix .` — a double space, because the skeleton appends
   a spacer and the template already had one after `$1`.
3. Press `<backspace>` to kill the spacer, then `<tab>` to reach `$2`.

Step 3's `<backspace>` is what this plan removes.

## Current behavior (code map)

- `src/sase/ace/tui/widgets/_xprompt_arg_assist_skeletons.py:47-57` —
  `xprompt_completion_skeleton()` returns `f"{entry.insertion} "` for an entry with no
  required inputs, unless the next character is punctuation (`_suppresses_no_arg_space`,
  line 13).
- `src/sase/ace/tui/widgets/_xprompt_arg_hints.py:319-345` —
  `_note_xprompt_completion_spacer()` records a `PendingXPromptCompletionSpacer`
  (`spacer_offset`, `reference_start`, `reference_text`, `has_optional_inputs`) right
  after the skeleton expands, while the cursor still sits just past the spacer. Called
  from `_file_completion_accept.py:163`, `_prompt_soft_completion.py:305`, and
  `_prompt_input_bar_actions.py:676`.
- `src/sase/ace/tui/widgets/_xprompt_arg_hints.py:347-375` —
  `_consume_xprompt_completion_spacer()` replaces the spacer in place when the very next
  keystroke is `,` (any eligible entry) or `:` (optional-only entries), after checking
  four invariants: the spacer offset is still in range, the cursor is still exactly at
  `spacer_offset + 1`, `text[spacer_offset]` is still `" "`, and the reference text is
  unchanged.
- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py:179-190` — the top of
  `_on_key` reads the pending spacer into the local `pending_spacer`, unconditionally
  clears the attribute (the one-shot rule: any other key drops it), and consumes the
  keystroke only when `_consume_xprompt_completion_spacer` succeeds. A `tab` keypress
  falls straight through this block today.
- `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py:406-436` — the INSERT-mode
  `tab` / `shift+tab` branch: bullet/ordered shift only when no session is active, then
  `_try_retreat_tabstop()` for `shift+tab`, else `_try_expand_snippet()` and finally
  `_try_advance_tabstop()`.
- `src/sase/ace/tui/widgets/_snippets.py:196-222` — `_try_advance_tabstop()` /
  `_try_retreat_tabstop()` drive the Rust session engine through
  `sase.core.snippet_session_facade` and return whether the cursor moved.

Two facts from the Rust engine (`../sase-core/crates/sase_core/src/snippet_session.rs`)
that this plan depends on, both already true today:

- `expand()` (line 210) with an empty tabstop list leaves an enclosing session untouched
  when the expansion range is contained in the innermost session span. That is why
  accepting `#name ` at a tabstop keeps the snippet session alive — the premise of this
  feature.
- `apply_edit()` (line 303) remaps every stop sticky-right (line 517), so deleting one
  character at `spacer_offset` shifts every later stop back by exactly one and clamps a
  stop sitting on the deleted space to `spacer_offset`. Routing the deletion through the
  widget's normal edit path therefore keeps every tabstop correct with no extra
  bookkeeping.

## Design decisions

1. **Fire on both `tab` and `shift+tab`.** Both are "I am done with this reference, take
   me to another tabstop" gestures; leaving the spacer behind on `Shift+Tab` only would
   be an arbitrary asymmetry.
2. **Only when the jump actually moves the cursor.** `advance()` off the last stop ends
   the session and reports no target; `retreat()` at the first stop reports no target
   and keeps the session. In both cases `Tab` is a consumed no-op with no visible
   effect, and silently eating a character there would be surprising. Probe first,
   delete only when the jump has a target.
3. **Delete, then jump — never the reverse.** `_try_advance_tabstop()` calls
   `_try_auto_placeholder_completion()` at the new stop; running that before the
   deletion would compute a placeholder menu against text that is about to shift by one.
   Deleting first also lets the engine's own sticky-right remap place the target stop,
   instead of this code hand-correcting a cursor offset.
4. **Bypass `_try_expand_snippet()` on this path.** Once the spacer is gone, the word
   immediately before the cursor is the xprompt name (`plain` in `#plain`), so falling
   through to the generic `tab` branch could expand a same-named snippet trigger instead
   of advancing. The new path calls `_try_advance_tabstop()` / `_try_retreat_tabstop()`
   directly.
5. **Both eligible entry shapes qualify.** Unlike the `:` rewrite (optional-only), the
   spacer is equally unwanted before a tabstop jump for no-input and optional-only
   entries, so `has_optional_inputs` is not consulted.
6. **One-shot semantics are unchanged.** The attribute is still cleared at the top of
   `_on_key` for every key; the new path only reads the local the existing block already
   captured, and the same four invariants must still hold.

## Implementation

### 1. `src/sase/ace/tui/widgets/_xprompt_arg_hints.py`

Extract the shared invariant check and add the tabstop-jump consumer.

- Add a private predicate used by both consumers:

  ```python
  def _xprompt_completion_spacer_is_intact(
      self,
      pending: PendingXPromptCompletionSpacer,
  ) -> bool:
      """Return True while a pending spacer is still exactly as it was inserted.

      The cursor must still sit immediately after the spacer, the spacer must
      still be a space, and the reference text before it must be unchanged.
      """
  ```

  Move the four checks out of `_consume_xprompt_completion_spacer` into it
  (`spacer_end > len(text)`, cursor offset `!= spacer_end`,
  `text[pending.spacer_offset] != " "`, reference-text mismatch), and have
  `_consume_xprompt_completion_spacer` keep only its character-eligibility test plus a
  call to the new predicate before `_replace_absolute_range`. Behavior there must not
  change.

- Add the new consumer:

  ```python
  def _consume_xprompt_completion_spacer_for_tabstop(
      self,
      pending: PendingXPromptCompletionSpacer,
      *,
      retreat: bool,
  ) -> bool:
      """Delete a pending completion spacer on the way to a snippet tabstop.

      An xprompt with no required inputs completes to ``#name ``; when the very
      next key jumps to another tabstop the reference is finished and that
      spacer is dead weight the user would otherwise delete by hand. Deleting
      before the jump lets the session engine remap the target stop for the
      one-character deletion, and keeps the placeholder completion that the jump
      opens in sync with the final text. Returns False -- leaving the spacer and
      the keystroke to the normal Tab path -- when no session is live, when the
      spacer is no longer intact, or when the jump would have no target.
      """
  ```

  Body, in order:
  1. `if not self.snippet_session_active: return False`
  2. `if not self._xprompt_completion_spacer_is_intact(pending): return False`
  3. `if not self._snippet_tabstop_jump_moves(retreat=retreat): return False`
  4. `self._replace_absolute_range(pending.spacer_offset, pending.spacer_offset + 1, "")`
     (this leaves the cursor right after the reference, since `_replace_absolute_range`
     parks it at `start_offset + len(replacement)`)
  5. `self._try_retreat_tabstop() if retreat else self._try_advance_tabstop()`, then
     `return True` — the keystroke is consumed either way; step 3 already established
     that the jump has a target.

  Add `_snippet_tabstop_jump_moves`, `_try_advance_tabstop`, and `_try_retreat_tabstop`
  to this module's `TYPE_CHECKING` stub block (lines 38-74) alongside the existing
  cross-mixin stubs; `snippet_session_active` is already declared there (line 48).

### 2. `src/sase/ace/tui/widgets/_snippets.py`

Add the read-only probe to `SnippetExpansionMixin`, next to the two jump methods:

```python
def _snippet_tabstop_jump_moves(self, *, retreat: bool) -> bool:
    """Return True when a tabstop jump would land on another stop.

    The engine's transitions are pure, so this asks it for the answer and
    discards the returned state rather than reimplementing "is there a next
    stop?" in Python -- advancing off the last stop ends the session and
    retreating at the first stop stays put, and both report no target.
    """
    if not self._snippet_session.is_active:
        return False
    move = retreat_snippet_session if retreat else advance_snippet_session
    return move(self._snippet_session).cursor_offset is not None
```

Both facade functions are already imported at the top of the module. The returned state
must be discarded — the real transition happens in `_try_advance_tabstop` /
`_try_retreat_tabstop` after the deletion has remapped the stops.

### 3. `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`

- Extend the comment at lines 174-178 to cover the new key: an immediate `Tab` /
  `Shift+Tab` that jumps to another snippet tabstop deletes the spacer instead of
  rewriting it, and is handled in the `tab` branch rather than here because tabstop
  jumps are INSERT-mode only.
- In the `tab` / `shift+tab` branch (line 406), right after
  `self._clear_soft_completion(cancel_timer=True)` and before the
  `if not self.snippet_session_active:` bullet-shift block, insert:

  ```python
  if pending_spacer is not None and (
      self._consume_xprompt_completion_spacer_for_tabstop(
          pending_spacer,
          retreat=event.key == "shift+tab",
      )
  ):
      return
  ```

  `pending_spacer` is the local already bound at line 179; the attribute was cleared
  there, so the one-shot rule holds whether or not this path fires. Add
  `_consume_xprompt_completion_spacer_for_tabstop` to the module's `TYPE_CHECKING` stub
  block near `_consume_xprompt_completion_spacer` (line 103).

- Nothing else in the branch changes: when the new call returns False the existing
  bullet-shift / retreat / expand / advance order runs exactly as before.

No hint refresh is needed after the jump — `_refresh_xprompt_arg_hint_from_cursor()`
early-returns while a snippet session is active (`_xprompt_arg_hints.py:101`), and the
session is still live by construction (decision 2).

### 4. Tests — `tests/ace/tui/widgets/test_xprompt_completion_spacer.py`

Extend the module docstring to cover the tabstop-jump case and add the cases below.
Build the session without `load_text` (which clears it) — expand a template directly,
then insert the trigger through the widget's own edit path so the stops remap the same
way they do in the app:

```python
ta.load_text("")
ta._expand_snippet_template_at_range(
    "$1 and $2", (0, 0), (0, 0), session_policy="reset"
)
ta._replace_via_keyboard("#p", (0, 0), (0, 0))  # leaves the cursor at (0, 2)
_seed_entries(ta, [_entry("plain")])
await pilot.press("ctrl+t")   # -> "#plain  and ", cursor (0, 7), spacer at 6
```

1. `test_tab_after_spacer_jumps_to_next_tabstop_without_the_space` — the setup above,
   then `await pilot.press("tab")`; assert `ta.text == "#plain and "`,
   `ta.cursor_location == (0, 11)` (the remapped `$2`), and
   `ta._pending_xprompt_completion_spacer is None`.
2. `test_shift_tab_after_spacer_retreats_without_the_space` — same template, but press
   `tab` once first so the session sits on `$2`, insert `#p` there, accept, then
   `shift+tab`; assert `ta.text == " and #plain"` and `ta.cursor_location == (0, 0)`.
3. `test_tab_without_a_snippet_session_keeps_the_spacer` — accept `#plain ` with no
   session (the existing `Ctrl+T` setup), press `tab`; assert `ta.text == "#plain "` is
   unchanged.
4. `test_tab_at_the_last_tabstop_keeps_the_spacer` — expand a single-stop template
   (`"only $1"`), accept there, press `tab`; assert the trailing space survives and the
   session ended (`ta.snippet_session_active is False`).
5. `test_tab_does_not_expand_a_snippet_named_after_the_xprompt` — the decision-4
   regression: build the app as `CompletionTestApp(snippets={"plain": "EXPANDED"})`, run
   the case-1 setup, and press `tab` inside
   `patch.object(type(ta), "_ace_app", new_callable=lambda: property(lambda self: app))`
   (mirroring `_setup()` in `tests/ace/tui/widgets/test_prompt_snippet_expansion.py`);
   assert the text is `"#plain and "` and contains no `EXPANDED`. The patch is required
   because `_ace_app` asserts `isinstance(self.app, AceApp)`; the other cases never
   reach it, since `_try_expand_snippet` bails on the whitespace before the cursor
   before it ever calls `_get_snippets()`.
6. `test_cursor_movement_invalidates_the_tab_spacer_deletion` — case-1 setup, move the
   cursor one column left, press `tab`; assert the space survives.
7. `test_spacer_tab_deletion_is_one_shot` — case 1, then press `tab` again; assert the
   second `Tab` only advances/ends the session and deletes nothing further.

Existing punctuation tests in this module must keep passing untouched — they are the
regression net for the `_xprompt_completion_spacer_is_intact` extraction.

### 5. Docs — `docs/ace.md`

In the smart-insertion paragraph at lines 4625-4628 (which already documents "a selected
xprompt with no required inputs inserts a trailing space"), add one sentence: when the
next keystroke is `Tab` or `Shift+Tab` and it jumps to another snippet tabstop, that
trailing space is removed on the way out; at the final tabstop, where the jump has
nowhere to go, the space is kept.

## Verification

```bash
just install
just check
```

`just check` covers the lint gates plus the diff-scoped test lane. If the scoped run
reports an unusual selection or escalates, run `just check-full` through `/sase_monitor`
instead of inline.

## Out of scope

- **Suppressing the spacer at insertion time when the next character is whitespace.**
  `_suppresses_no_arg_space` only suppresses before punctuation, which is why the repro
  above produces a double space. Widening it to whitespace would fix that one case but
  not the tabstop-at-end-of-line or tabstop-followed-by-newline cases, and it would
  change behavior outside snippets entirely. Worth its own follow-up if the double space
  still annoys after this lands.
- Any change to the `,` / `:` rewrites, to which entry shapes get a spacer, or to the
  skeletons themselves.
- Rust-side changes. The spacer lifecycle is prompt-input editing policy and already
  lives in Python; the new probe only reads existing pure transitions through the
  facade, so `../sase-core` is untouched.

## Risks

- **A same-named snippet trigger hijacking `Tab`** — the reason for decision 4, covered
  by test 5.
- **Deleting a character the user wanted** — bounded by the four intactness invariants
  (one-shot, cursor unmoved, spacer and reference text unchanged) plus decision 2's
  requirement that the jump have a real target.
- **Dot-repeat capture drift** — none expected. `_dot_insert_capture_offset` marks where
  the INSERT session began, which always precedes the spacer (an intervening `Escape`
  would have dropped the pending spacer), and offsets before an edit are not remapped.
  This matches the existing `,` / `:` path, which likewise does not remap the capture.
