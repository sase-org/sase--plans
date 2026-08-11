---
tier: tale
title: Vim `*` / `#` word-under-cursor search in the prompt input widget
goal:
  Prompt NORMAL mode gains vim's `*` / `#` (whole-word search for the word under the
  cursor), `g*` / `g#` (the substring variants), and VISUAL-mode `*` / `#` (search the
  selection), all sharing the existing prompt-stack search register so `n` / `N` repeat
  them with identical semantics.
size: medium
proposed_by: bbugyi200.athena.xw
create_time: 2026-08-11 07:27:26
status: wip
---

# Plan: Vim `*` / `#` word-under-cursor search in the prompt input widget

## Goal

Add vim's "search for the thing under the cursor" family to the ACE prompt input widget
(`PromptTextArea` inside `PromptInputBar`):

| Key       | Mode          | Behavior                                                       |
| --------- | ------------- | -------------------------------------------------------------- |
| `*`       | NORMAL        | Search **forward** for the **whole word** under the cursor     |
| `#`       | NORMAL        | Search **backward** for the **whole word** under the cursor    |
| `g*`      | NORMAL        | Like `*`, but matches the word as a **substring** too          |
| `g#`      | NORMAL        | Like `#`, but matches the word as a **substring** too          |
| `*` / `#` | VISUAL/V-LINE | Search forward / backward for the **selected text**, literally |

All four NORMAL-mode keys accept a count (`3*` jumps to the third following match).
Every variant records the shared prompt-stack search register, so a subsequent `n` / `N`
repeats the _same_ match semantics (whole-word or not, case-sensitive) rather than
degrading to a plain substring search.

## Why this shape

`*` in vim is not "type the word into `/`". It (a) resolves a keyword under/after the
cursor, (b) searches for `\<word\>`, and (c) is explicitly exempt from `smartcase`. All
three details are what make it feel right, and all three need new plumbing here, because
today's prompt search is literal-substring-only with smartcase always on.

## Current architecture (verified)

Two independent vim-search stacks exist. **Only the first is in scope.**

1. **Prompt stack** (in scope):
   - `src/sase/ace/tui/widgets/_vim_search.py` — pure helpers
     `find_search_matches(text, query, *, smartcase=True)` (literal, overlap-aware,
     smartcase) and `select_search_match(...)`.
   - `src/sase/ace/tui/widgets/_prompt_search.py` (`PromptSearchMixin`) — the `/` `?`
     incremental command line, `_apply_prompt_search_result`,
     `_clear_prompt_search_result`, `_repeat_prompt_search`,
     `_show_prompt_search_feedback`.
   - `src/sase/ace/tui/widgets/_prompt_input_bar_search.py`
     (`PromptInputBarSearchMixin`) — owns the **shared register**
     (`_prompt_search_register: tuple[str, SearchDirection] | None`), the whole-stack
     pane snapshot (`_prompt_search_pane_snapshot`), and the counted cross-pane resolver
     (`_resolve_prompt_search_destination`) used by `n` / `N`.
   - `src/sase/ace/tui/widgets/_search_highlight.py` (`SearchHighlightMixin`) —
     `search.match` / `search.current` overlay spans.
   - Key dispatch: `_vim_normal.py::_handle_normal_mode_key` (early block for `/` `?`
     `K` `Ctrl+]`), `_vim_normal_motions.py::_handle_normal_motion_key` (where `n` / `N`
     live, **after** count parsing and pending-key routing),
     `_vim_normal_pending.py::_handle_normal_pending_key` (the `g` prefix; consults
     `_dispatch_host_g_prefix_key` first, then vim's own `gg` / `ge` / `gu` branches),
     `_vim_visual_keys.py::_handle_visual_mode_key`.
   - Host hooks: `vim_text_area.py::VimTextArea` declares inert defaults
     (`_start_prompt_search`, `_repeat_prompt_search`, ...) that `PromptTextArea`
     overrides; other `VimTextArea` hosts (config editor, AXE cells, modals) inherit the
     inert defaults.

2. **`VimSearchController`** (`vim_search_controller.py`), used by the zoom panel
   (`modals/zoom_panel_search.py`) and agent metadata search
   (`actions/agents/_metadata_search.py`). **Out of scope** — see Non-goals.

Useful existing pieces to reuse rather than reinvent:

- `_vim_motions.py::_char_class` already encodes vim's keyword class (`isalnum() or "_"`
  → `"word"`). The new word resolver must use it so `*` and `w` / `iw` agree.
- `_vim_visual_ops.py::_visual_selected_text()` returns `(text, VisualKind)` for the
  current selection — exactly what VISUAL `*` needs.
- `find_a_word` / `find_inner_word` are line-local and cursor-anchored; they classify a
  _group_ (which may be punctuation or whitespace), so they are **not** a drop-in for
  `*`, which must skip forward past non-keyword characters. Write a dedicated resolver.

Verified non-conflicts:

- `*` is currently a **silent no-op** in prompt NORMAL mode: `VimTextArea._on_key`
  swallows unhandled printable NORMAL/VISUAL keys via `_swallow_unhandled_vim_key`, so
  it never reaches the app-level `open_saved_query_picker: "asterisk"` binding in
  `src/sase/default_config.yml`. Nothing regresses, and there is no config keymap to
  update — prompt-widget vim keys are hardcoded, not configurable.
- `*` is not in `_PROMPT_G_PREFIX_BINDINGS`, so `g*` falls straight through
  `_dispatch_host_g_prefix_key` into vim's own `g` branches, and the `g` hint panel
  (which renders only the prompt-local table) is unaffected — same as `gg` today.

## Design decisions

**D1 — Whole-word matching is a matcher flag, not a regex query.** Extend
`find_search_matches` with `whole_word: bool = False`. Keep the existing overlap-aware
lookahead construction and `re.escape` on the query; when `whole_word` is set, prepend
`\b` only if the query's first character is a keyword character and append `\b` only if
its last character is one. (Unconditional `\b` would silently match nothing for a query
edged by punctuation.) The function stays pure and independently unit-testable.

**D2 — `*` is case-sensitive; typed `/` searches keep smartcase.** Vim documents `*` /
`#` as exempt from `smartcase`, and "find instances of _this_ word" is the least
surprising reading of the user's request. So `*` passes `smartcase=False`, giving exact
case matching in both directions (`*` on `Foo` will not match `foo`, and `*` on `foo`
will not match `Foo`). `/` and `?` are untouched.

**D3 — The shared register must carry the match semantics, not just the string.** This
is the crux. Today the register is `tuple[str, SearchDirection]`, and both
`repeat_prompt_search` and `_prompt_search_pane_snapshot` re-derive matches from the
bare string. If `*` only stored the word, `n` immediately after `*` would silently widen
to a smartcase substring search and land on different matches.

Replace the register payload with a small frozen record carrying `query`, `whole_word`,
and `smartcase` (a frozen dataclass in `_vim_search.py` is the natural home, alongside
`SearchSelection`), keeping `direction` as the second element or a field of the same
record — implementer's choice, but it must be one value that flows unchanged through
`record_prompt_search` → `prompt_search_register` → `repeat_prompt_search` →
`_prompt_search_pane_snapshot` → `find_search_matches`. `_confirm_prompt_search` records
a `/`-typed query with `whole_word=False, smartcase=True`, preserving today's behavior
exactly.

This is a deliberate breaking change to an internal API;
`tests/ace/tui/widgets/test_prompt_search_interactive.py` asserts
`bar.prompt_search_register() == ("alpha", "forward")` and must be updated to the new
shape.

**D4 — Word resolution follows vim, and is line-local.** New pure helper (in
`_vim_motions.py`, next to `_char_class`), roughly:

```python
def find_search_word(doc, row, col) -> tuple[int, int, str] | None:
    """Return ``(row, start_col, word)`` for vim ``*``, or ``None``."""
```

Rules: if the character at `col` is a keyword character, expand left and right over the
keyword run. Otherwise scan **forward on the current line only** for the first keyword
character and expand from there. Never cross a line boundary (vim does not). Return
`None` when the rest of the line has no keyword character.

**D5 — Delegate the jump to the existing cross-pane resolver.** `*` must behave like `n`
across a `---` prompt stack, so after resolving the word it should:

1. Move the cursor to the **start of the resolved word** — this is both vim's behavior
   and what makes the origin correct, since the resolver excludes matches at
   `start == origin_offset` in both directions, so the word under the cursor is skipped
   and a mid-word cursor cannot re-select its own match.
2. Record the register (D3).
3. Call the existing `repeat_prompt_search(origin, reverse=..., count=...)` path, which
   already handles cross-pane focus, per-pane highlight clearing, `wrapped` feedback,
   and "pattern not found".

A single occurrence in the whole stack therefore wraps back onto itself and emits
`search hit BOTTOM, continuing at TOP` — matching vim.

**D6 — No search command line for `*`.** The `/` panel (`#prompt-search-command`) is an
interactive, 4-line, height-managed box owned by an active `_search_active` session.
Showing it non-interactively for `*` would leave a panel nobody owns and perturb prompt
bar height. `*` gives feedback through the match highlights it sets plus the existing
wrap / not-found notifications. `_search_active` stays `False`, so `*` never intercepts
subsequent keys.

**D7 — Failure is a notification, not a silent no-op.** When no word can be resolved
(cursor on a blank line, or only punctuation/whitespace to the end of the line), emit
vim's meaning through `_show_prompt_search_feedback` (e.g. `no string under cursor`),
leave the cursor untouched, leave existing highlights and the register untouched, and
still report the key as handled so it cannot leak to an app-level binding. Same
treatment for a VISUAL selection that is empty.

**D8 — Dispatch placement mirrors `n` / `N`.** Put NORMAL `*` / `#` in
`_vim_normal_motions.py::_handle_normal_motion_key` beside the `n` / `N` branch, not in
the early `/` `?` block. That placement is what makes counts (`3*`) work at all, since
the early block runs before the count prefix is parsed, and it inherits three behaviors
for free:

- `_pending_keys` routing runs first, so `f*`, `t*`, `dt*`, `r*`, and surround targets
  still consume `*` as a **literal character**.
- A dangling `_pending_operator` is cleared and the key acts as a plain motion, exactly
  as `n` / `N` already do (`d*` moves rather than deleting; matching `n`'s precedent is
  preferred over inventing operator-motion support).
- `*` lands in `_mutation_key_buffer` but never commits to `_last_mutation_keys` (only
  `_vim_normal_state.py` mutation paths do that), so `.` is unaffected — again the same
  as `n`.

`g*` / `g#` go in `_vim_normal_pending.py` as a new `pending == "g"` branch alongside
the existing `g`+`uU~`, `g`+`g`, `g`+`eE` branches, which gives them `pending_count` (so
`3g*` works).

**D9 — VISUAL `*` searches the selection literally.** Take `_visual_selected_text()`,
return to NORMAL mode, record the register with `whole_word=False, smartcase=False`
(word boundaries are meaningless for an arbitrary selection), and delegate to the same
resolver. A multi-line or V-LINE selection is matched literally including its newlines,
which works because matching runs over `text_area.text`; it simply cannot match across a
pane boundary. Cursor origin is the selection start, so the selection's own occurrence
is skipped.

**D10 — This stays in Python, not `sase-core`.** The whole vim tower, its search
matchers, and its keymaps already live in this repo as presentation-layer TUI behavior,
and no other frontend consumes them. Adding `*` introduces no new shared backend
concept, so per the Rust core boundary litmus test there is no `../sase-core` change.

## Implementation steps

1. **`src/sase/ace/tui/widgets/_vim_search.py`** — add `whole_word: bool = False` to
   `find_search_matches` (D1) and add the frozen register record (D3). Export it.

2. **`src/sase/ace/tui/widgets/_vim_motions.py`** — add the `find_search_word` resolver
   (D4), built on `_char_class`.

3. **`src/sase/ace/tui/widgets/_prompt_input_bar_search.py`** — thread the new register
   record through `_prompt_search_register`, `record_prompt_search`,
   `prompt_search_register`, and `repeat_prompt_search`; make
   `_prompt_search_pane_snapshot` pass `whole_word` / `smartcase` to
   `find_search_matches`. No change to `_resolve_prompt_search_destination`.

4. **`src/sase/ace/tui/widgets/_prompt_search.py`** — update `_confirm_prompt_search` to
   record the new shape with today's flags, and add the entry point used by every new
   key, e.g.:

   ```python
   def _search_word_under_cursor(
       self, *, reverse: bool = False, whole_word: bool = True, count: int = 1
   ) -> bool: ...

   def _search_visual_selection(self, *, reverse: bool = False, count: int = 1) -> bool: ...
   ```

   Both implement D5 / D7 / D9 and return `True` (handled) in every branch.

5. **`src/sase/ace/tui/widgets/vim_text_area.py`** — add matching inert host-hook
   defaults next to `_start_prompt_search` / `_repeat_prompt_search`, with docstrings in
   the same "Default: inert" style, so non-prompt `VimTextArea` hosts keep working
   unchanged and `mypy` sees the methods on the base.

6. **`src/sase/ace/tui/widgets/_vim_normal_motions.py`** — dispatch `*` / `#` (D8).

7. **`src/sase/ace/tui/widgets/_vim_normal_pending.py`** — dispatch `g*` / `g#` (D8).

8. **`src/sase/ace/tui/widgets/_vim_visual_keys.py`** — dispatch VISUAL / V-LINE `*` /
   `#` (D9).

9. **Help popup** — add rows to `PROMPT_INPUT_SECTION` in
   `src/sase/ace/tui/modals/help_modal/binding_common.py`, immediately after the
   existing `("n / N", "Repeat prompt search fwd/rev")` row. Keep descriptions within
   the 32-char limit and the 57-char box width mandated by `src/sase/ace/CLAUDE.md`; two
   rows such as `("* / #", "Search word under cursor")` and
   `("g* / g#", "... as substring")` fit.

10. **Docs — `docs/ace.md`** — add rows to the prompt NORMAL-mode "Other Commands" table
    (currently ending with the `/ / ?` and `n / N` rows near line 4664), extend the
    paragraph below it that explains search preview / register sharing to state the
    whole-word and case-sensitivity rules and the "no string under cursor" feedback, and
    note VISUAL `*` / `#` in the "Visual Mode" section.

## Testing

Add `tests/ace/tui/widgets/test_prompt_star_search.py`, modeled on the existing
`test_prompt_search_interactive.py` (Textual `App` host + `pilot.press`, with app-level
bindings registered so key leakage is observable). Cover:

- `*` jumps to the next occurrence, sets `search.match` / `search.current` highlights,
  and records the register.
- **Whole-word**: `*` on `log` in `log login catalog log` reaches only the second `log`.
- `g*` on the same text does reach `login` / `catalog`.
- `#` searches backward; `3*` honors the count.
- `n` / `N` **after** `*` keep whole-word + case-sensitive semantics (the D3 regression
  guard — this test fails if the register drops the flags).
- Case sensitivity both ways (`*` on `Foo` skips `foo`; `*` on `foo` skips `Foo`).
- Cursor mid-word and on the word's first character produce the same destination.
- Cursor on whitespace/punctuation scans forward on the line; a line with no keyword
  character after the cursor notifies, leaves the cursor put, and leaves the register
  and highlights untouched.
- Single occurrence in the stack wraps and emits the `hit BOTTOM` feedback.
- Cross-pane jump in a `---` stack focuses the destination pane in NORMAL mode.
- `f*` and `dt*` still treat `*` as a literal target; INSERT-mode `*` inserts `*`.
- `*` in NORMAL mode does **not** fire an app-level `asterisk` binding (assert the host
  app's action counter stays 0, mirroring the `edit_query_count` assertions in the
  existing search test).
- VISUAL `*` on a charwise selection, on a V-LINE selection, and on an empty selection.

Add pure unit tests next to the existing vim helper tests (e.g. alongside
`tests/test_prompt_normal_mode_char_search.py`) for
`find_search_matches(..., whole_word=True)` — including a query edged by punctuation,
which must not be `\b`-wrapped — and a table-driven test for `find_search_word`.

Update `tests/ace/tui/widgets/test_prompt_search_interactive.py` for the register shape
(D3).

## Verification

```bash
just install     # workspace venvs are ephemeral; required before anything else
just check       # whole-repo lint gates + diff-scoped tests
```

Run `just check-full` before landing, since this touches the shared vim tower that many
prompt tests import. `just test-visual` is not expected to be needed — no PNG snapshot
exercises `*` — but if
`tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py` moves, re-check
rather than force-updating goldens. Follow `sase/memory/symvision.md` for any
unused-symbol findings on the new helpers.

## Non-goals

- `VimSearchController` hosts (zoom panel, agent metadata search) and the other
  `VimTextArea` hosts (`SingleLineVimTextArea`, config editor, AXE entry cells, modals):
  the base-class hook defaults stay inert, so their behavior is unchanged. Porting `*`
  there is a reasonable follow-up, not part of this plan.
- Regex search patterns, a user-facing `ignorecase` / `smartcase` setting, search
  history, or a `:` command line.
- Making prompt vim keys configurable through `default_config.yml`.
- Any change under `../sase-core` (D10).

## Risks

- **Register shape change (D3)** ripples through the bar mixin, the text-area mixin, and
  an existing test. It is small but must be done in one pass; a partial change silently
  degrades `n` after `*` rather than failing loudly — hence the explicit regression
  test.
- **Dispatch ordering.** Placing `*` before the pending-key routing would break `f*` /
  `dt*`; placing it in the early `/` `?` block would break counts. The tests for both
  are listed above deliberately.
