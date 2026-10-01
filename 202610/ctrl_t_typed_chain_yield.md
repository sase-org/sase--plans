---
tier: tale
title: Restore prompt Ctrl+T completion under next_word auto
goal:
  Under the default next_word auto mode, Ctrl+T right after typing completes paths,
  xprompts, model aliases, and prompt-local or history words again (e.g. `+bob-cli
  bob-m` -> `bob-mac-capture`). Visible or pending next-word guesses keep owning Ctrl+T,
  and next-word requests stay as boundary or word-end fallbacks.
size: small
proposed_by: bbugyi200.apollo.3v
create_time: 2026-10-01 10:06:06
status: wip
---

# Restore prompt `Ctrl+T` completion under `next_word: auto`

## Diagnosis (confirmed and bisected)

**Suspicion confirmed.** The next-word autosuggest epic (sase-1dq) broke `Ctrl+T` manual
completion in the prompt input widget for every user on the default config.

Reproduction: with default settings (`ace.prompt_completion.next_word: auto`), type
`+bob-cli bob-m` into the prompt and press `Ctrl+T`. History contains `bob-mac-capture`,
but the text is unchanged and the border shows `warming next words…` or
`no next-word guess`. The same steps with `next_word: chain` or `off` complete
`bob-mac-capture`. Loading the same text with `load_text` (which is what the test suite
does) also completes, because no keystroke arms a chain. A bisect harness run on
`e2cec539ef^` (pre-mid-word) completes the word in `auto`; `e2cec539ef` does not.

**Root cause.** The prompt's `Ctrl+T` ladder (`_prompt_text_area_key_handling.py`,
INSERT branch) runs row 3, `self._explicit_next_word_ctrl_t()`, **before** row 4, the
manual dispatcher `_try_file_completion_tab()`. Row 3 delegates to
`_explicit_next_word_request()`, which consumes the key whenever _any_ next-word chain
is armed at the cursor. It shows a hint even when it has no guess. The ladder was
designed when a chain was armed only by intent (a word commit, a ghost/peek accept, or
an explicit `Ctrl+T`), so "armed" meant "the user is chaining next words". sase-1dq
broke that assumption:

1. `e2cec539ef` (mid-word autosuggest) makes `_maybe_auto_next_word_midword` arm a
   `midword=True` chain after **every typed word character**
   (`_arm_next_word_chain(reveal="delayed", complete_current_word=True)`, or
   `_anchor_next_word_chain(midword=True)` on the deferred long-draft path). It arms
   **before** predicting, so the chain stays armed when the core has no confident guess
   or the model is cold.
2. `0abe894140` (sase-1dq.8) flipped the default `next_word` from `chain` to `auto`.
   That exposed (1) to everyone, and also exposed the older sase-1cj.11 boundary trigger
   (`_maybe_auto_next_word_ghost` arms a boundary chain after a typed non-word character
   that follows a word, e.g. `/`, `:`, `(`). The same commit pinned the failing `Ctrl+T`
   interaction tests to `next_word="chain"`, which hid the regression.

Net effect in `auto`: right after typing, `Ctrl+T` never reaches structured or word
completion. Verified broken: prompt-local and history words (`bob-m`), paths
(`see srcdir/al` and `see srcdir/`), xprompts (`#f`, `/s`), and model aliases (`=l`,
`==`). Directive menus that auto-open on `%` are unaffected, because an open menu blocks
arming.

## Intended contract (already documented in `docs/ace.md` → Next-word prediction)

"At the end of a prose word, `Ctrl+T` first tries ordinary current-word completion; it
requests next words only when that completion has no candidate. Structured tokens, such
as paths and Jinja tags, keep their own completion behavior." Also: "`Ctrl+T` never
inserts an unseen guess."

Fix rule: **a chain armed by typing is not a request.** It owns `Ctrl+T` only while it
holds a guess the user can see or is about to see:

- a visible ghost or revealed peek: ladder row 2, unchanged;
- a pending (unrevealed) peek: row 3 reveals it, unchanged documented behavior.

Otherwise `Ctrl+T` falls through to row 4. Row 4 already handles the next-word cases
itself: `_try_next_word_boundary_request` at a whitespace boundary (sase-1dq.3 behavior,
including the `[^G r] recent files` hint), and `_try_next_word_word_end_fallback` (row
4b) at a prose word end after prompt-local/history completion misses. Chains armed by
commits, accepts, or explicit presses keep today's row-3 behavior, so `chain` mode and
the next-word continuation flow do not change.

## Implementation

### 1. Record chain provenance — `src/sase/ace/tui/widgets/next_word_completion.py`

Add a field to the frozen `NextWordChain` dataclass:

```python
#: Whether ``auto`` typing armed the chain (a typed word character or a
#: typed trigger after a word) rather than a word commit, an accept, or an
#: explicit ``Ctrl+T``. A typed chain is not a request: the prompt's
#: ``Ctrl+T`` yields it to manual completion unless a guess is pending.
typed: bool = False
```

Update the class docstring if it still says the snapshot is always "at the last word
commit".

### 2. Thread `typed` through both chain hosts

Add a keyword-only `typed: bool = False` parameter to
`_anchor_next_word_chain(self, *, midword: bool, typed: bool = False)` and to
`_arm_next_word_chain(..., typed: bool = False)`. Pass it into `NextWordChain(...)`.
`_arm_next_word_chain` forwards it to `_anchor_next_word_chain`. Do this in both
implementations:

- `src/sase/ace/tui/widgets/_prompt_next_word.py` (`PromptNextWordMixin`)
- `src/sase/ace/tui/modals/gate_input_panel_note.py` (the note editor host). It only
  needs to store the flag; its `Ctrl+T` behavior does not change.

Update the matching `TYPE_CHECKING` stubs in
`src/sase/ace/tui/widgets/_next_word_midword.py` (both methods) and
`src/sase/ace/tui/widgets/_next_word_ghost_peek.py` (`_arm_next_word_chain`).

Pass `typed=True` at exactly the three typing-triggered arm sites in the shared mixins:

- `_next_word_ghost_peek.py` `_maybe_auto_next_word_ghost`:
  `self._arm_next_word_chain(reveal="delayed", typed=True)`
- `_next_word_midword.py` `_maybe_auto_next_word_midword`:
  `self._arm_next_word_chain(reveal="delayed", complete_current_word=True, typed=True)`
- `_next_word_midword.py` `_defer_next_word_midword_request`:
  `self._anchor_next_word_chain(midword=True, typed=True)`

All other arm sites (accepts, commits, `_try_next_word_word_end_fallback`,
`_try_next_word_boundary_request`) keep the default `typed=False`.

### 3. Let typed chains yield prompt `Ctrl+T` — `src/sase/ace/tui/widgets/_prompt_next_word.py`

Add a small helper to `PromptNextWordMixin` and call it at the top of
`_explicit_next_word_ctrl_t`:

```python
def _typed_next_word_chain_yields_ctrl_t(self) -> bool:
    """Return whether ``Ctrl+T`` should skip an ``auto``-typed chain.

    ``auto`` arms a chain after almost every keystroke, so an armed chain
    alone is not a request. A typed chain owns the press only while its
    guess waits for the reveal beat (row 3 reveals it); otherwise the
    manual dispatcher (row 4) completes the token and requests next words
    itself at a whitespace boundary or a prose word end.
    """
    chain = getattr(self, "_next_word_chain", None)
    if chain is None or not chain.typed or not self._next_word_chain_is_armed():
        return False
    try:
        if self._next_word_peek_pending():  # type: ignore[attr-defined]
            return False
    except Exception:
        pass
    return True
```

```python
def _explicit_next_word_ctrl_t(self) -> bool:
    if self._typed_next_word_chain_yields_ctrl_t():
        return False
    consumed = self._explicit_next_word_request()
    ...  # unchanged
```

Update `_explicit_next_word_ctrl_t`'s docstring to state the typed-chain yield.

- `_try_next_word_boundary_request` calls `_arm_next_word_chain()` (untyped) before
  `_explicit_next_word_ctrl_t`, so the yield never fires there.
- Do **not** put the yield into the shared `_explicit_next_word_request`. The gate/plan
  note editor (`gate_input_panel_note.py`) has no completion menu, so `Ctrl+T` on a
  typed chain there must keep re-requesting and showing the hint.
- No explicit cancel of a pending deferred mid-word request is needed. A manual accept
  re-anchors (which cancels it), an open menu makes `_next_word_ghost_allowed()` false,
  and stale snapshots are already discarded.

### 4. Key-handler comment — `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`

Update the "Ctrl+T ladder rows 2-3" comment above the row-3 check. It should say that
row 3 handles chains armed by a commit, an accept, or an explicit press, plus pending
peeks, and that an `auto`-typed chain with nothing to reveal falls through to the manual
dispatcher. Leave the logic unchanged.

### 5. Docs — `docs/ace.md` (Next-word prediction section)

- After "At the end of a prose word, `Ctrl+T` first tries ordinary current-word
  completion; …", add that this also holds in `auto`. A guess requested while typing
  owns `Ctrl+T` only while its ghost or peek is showing or waiting to be revealed;
  otherwise `Ctrl+T` completes structured tokens, paths, and prompt-local and history
  words exactly as in `chain` mode.
- Replace "For a current-word request already started by `auto`, `Ctrl+T` shows that
  suffix and continuation directly instead of opening a whole-word menu. If the core
  supplies no current-word completion, that request shows no suggestion." with wording
  that matches the new behavior. A current-word request that found a guess shows it as a
  ghost or peek, and `Ctrl+T` takes it. When that request shows nothing (including when
  the core supplies no current-word completion), `Ctrl+T` runs ordinary completion
  instead of repeating the request.
- In the sentence about the gate/plan feedback note editor, note that it has no
  completion menu, so `Ctrl+T` there still re-requests a typing-armed guess and shows
  the hint.

Check whether `docs/configuration.md`'s `next_word` row and the keymap table row for
`Ctrl+T` in `docs/ace.md` still read correctly. They should need no change. No keymap or
`src/sase/default_config.yml` change is involved.

## Tests

Root testing gap: existing `Ctrl+T` tests use `load_text`, which never fires the `auto`
triggers. New regression tests must **type the input with `pilot.press`** in the default
`auto` mode.

### New file `tests/ace/tui/widgets/test_prompt_next_word_ctrl_t_yield.py`

Reuse the harnesses `HistoryCompletionTestApp` / `skip_unrelated_vcs_catalog_warm` from
`tests/ace/tui/widgets/_history_word_completion_helpers.py`. Use
`PromptCompletionSettings(word_ranking="recent")`, which keeps the default
`next_word="auto"`, and `NextWordTestApp` from `test_prompt_next_word.py` where a warm
model is needed (import or mirror it). Map punctuation keys for `pilot.press` (`space`,
`minus`, `plus`, `slash`, `colon`, `number_sign`, …). Each test should assert the typed
chain is armed before `Ctrl+T`, so it keeps proving the regression path. Cover:

1. The user's example: history `["bob-mac-capture"]`, type `+bob-cli bob-m`. Assert
   `ta._next_word_chain.typed and ta._next_word_chain.midword`, press `ctrl+t`, assert
   the text is `+bob-cli bob-mac-capture`. Run it with a cold model (harness default)
   and with a warm model whose result is not confident (monkeypatch
   `_predict_next_words` as `test_prompt_next_word_midword.py`'s `_patch_result` does),
   so both the `warming…` and `no next-word guess` consumption paths are covered.
2. Path mid-token: in a `tmp_path` (chdir via `monkeypatch`) with `srcdir/alpha.py` and
   `srcdir/beta.py`, type `see srcdir/al`, press `ctrl+t`, assert the text is
   `see srcdir/alpha.py`.
3. Path separator (typed boundary chain): type `see srcdir/`, assert the chain is typed
   and not midword, press `ctrl+t`, assert `_file_completion_active` with
   `_completion_kind == "file"`.
4. Prose boundary still requests next words: type `zzz qqq ` with `NextWordTestApp`,
   press `ctrl+t`, assert no menu and `NEXT_WORD_NO_GUESS_RECENT_FILES_HINT` in the
   border hint (sase-1dq.3 contract via row 4).
5. Prose word end with no completion candidate still falls back to next words (row 4b):
   type a word with no prompt-local or history match and assert the press is consumed by
   a next-word hint. Do not assert a menu.

### Unpin the masking tests (sase-1dq.8 pinned them to `chain`)

Remove `next_word="chain"` so these run in default `auto` and exercise the fix:

- `tests/ace/tui/widgets/test_auto_xprompt_completion.py`:
  `test_auto_xprompt_menu_toggle_disables_slash_skill_auto_open` and
  `test_auto_xprompt_menu_toggle_disables_auto_open_only`
- `tests/ace/tui/widgets/test_model_alias_completion_interactions.py`:
  `test_equals_alias_ctrl_t_opens_when_auto_directive_menu_is_disabled`
- `tests/ace/tui/widgets/test_model_explicit_completion_interactions.py`:
  `test_double_equals_ctrl_t_opens_when_auto_directive_menu_disabled`

Keep the intentional chain-mode pins (`test_auto_space_only_fires_in_auto_mode`,
`test_chain_mode_shows_no_midword_ghost`, `test_gate_note_chain_mode_ignores_typing`).

### Update tests that encode the old contract

- `tests/ace/tui/widgets/test_prompt_next_word_midword.py::test_explicit_midword_never_opens_a_menu`
  asserts the buggy behavior (typed mid-word chain + completionless result → `Ctrl+T`
  shows `no next-word guess`). Rename it (e.g.
  `test_typed_midword_miss_yields_ctrl_t_to_word_completion`) and make it assert the
  yield. Load text that contains an earlier complete word (e.g.
  `"implement it. Can you help me impl"`), press `e`, assert a typed midword chain,
  press `ctrl+t`, and assert the prompt-local word completed (text ends with
  `implement`) with no next-word menu (`_completion_kind` is not the next-word kind).
- `tests/ace/tui/widgets/test_prompt_next_word.py::test_auto_armed_boundary_ctrl_t_without_guess_teaches_recent_files`:
  its assertions should still pass. Update the inline comment, which says `Ctrl+T`
  "takes the armed-chain row rather than the unarmed boundary dispatch". It now takes
  the boundary dispatch, which re-arms and teaches `Ctrl+G r`.
- Existing pending-peek / visible-ghost tests
  (`test_midword_peek_reveals_then_finishes_word`,
  `test_ctrl_t_finishes_word_then_continues`, `test_real_midword_peek_after_prose`, and
  the `tests/ace/tui/test_gate_note_next_word.py` suite) must pass unchanged. They pin
  rows 2–3 and the note editor contract.

## Verification

- Targeted (the named `test` tool accepts extra pytest args):
  `sase tool run test tests/ace/tui/widgets/test_prompt_next_word_ctrl_t_yield.py tests/ace/tui/widgets/test_prompt_next_word.py tests/ace/tui/widgets/test_prompt_next_word_midword.py tests/ace/tui/widgets/test_auto_xprompt_completion.py tests/ace/tui/widgets/test_model_alias_completion_interactions.py tests/ace/tui/widgets/test_model_explicit_completion_interactions.py tests/ace/tui/test_gate_note_next_word.py tests/ace/tui/widgets/test_history_word_completion_accept.py tests/ace/tui/widgets/test_prompt_word_completion.py`
- Then `sase tool run check` (the agent default verification; do not run `check-full`).
- No PNG goldens should change, because there is no rendering change. If a next-word
  visual snapshot fails, investigate it rather than regenerating it.

## Out of scope

- Changing the Rust core's prediction gating or adding core-side current-word completion
  to manual completion.
- Gate/plan note editor `Ctrl+T` semantics, which stay as they are.
