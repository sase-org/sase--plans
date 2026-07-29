---
tier: tale
title: Expand %{ before trailing punctuation in the prompt input
goal:
  Typing { after a directive-valid % expands to the padded %{  } alt shorthand even when the cursor sits directly before
  trailing punctuation such as ?, in both the ACE prompt input and the sase-nvim mirror.
create_time: 2026-07-29 18:40:29
status: done
---

- **PROMPT:** [202607/prompts/alt_brace_punctuation.md](prompts/alt_brace_punctuation.md)
- **AGENTS:**
  - [bbugyi200.athena.ox--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ox.md#member-code)
  - [bbugyi200.athena.ox--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ox.md#member-plan)
- **COMMITS:**
  - [a79dad1](https://github.com/sase-org/sase/commit/a79dad1639a7a54406a4c185fdc15be00d8e8628) — feat(ace): allow alt
    braces before punctuation

# Plan: Expand `%{` before trailing punctuation

## Goal

Typing `{` after a directive-valid `%` must expand to the padded alt shorthand `%{  }` even when the cursor sits
directly before a trailing punctuation character (`?`, `!`, `.`, `,`, `;`, `:`), not only at end-of-line, before
whitespace, or before a bracket closer.

Target behavior in the ACE prompt input (verified against a patched build, see "Current state"):

| Buffer / cursor                | Keys         | Today (broken)              | After                          |
| ------------------------------ | ------------ | --------------------------- | ------------------------------ |
| `Which is better ?` @ `(0,16)` | `%` `{`      | `Which is better %{?`       | `Which is better %{  }?`       |
| ...then                        | `A` `\|` `B` | `Which is better %{A \| B?` | `Which is better %{ A \| B }?` |

Cursor lands at `(0, 19)` after `%{` (between the two padding spaces) and at `(0, 24)` after `B`.

## Why

The trailing-`?` case is the natural way to write a fan-out question: the user types the sentence, then goes back to
wrap the varying part. Today the expansion silently declines and — worse — the follow-up `|` still fires, because
`_find_enclosing_alt_span` treats an unclosed `%{` as running to end of text. The result is `%{A | B?`: an **unmatched
`%{`** that renders as an `alt.error` span and never fans out. So the current gate does not merely skip a convenience,
it steers the user into a malformed prompt.

The parse/render side is already punctuation-aware, so nothing downstream needs to change. Verified in this workspace:

```
split_prompt_for_alternatives("Which is better %{ A | B }?")
  -> ['Which is better A?', 'Which is better B?']
split_prompt_for_alternatives("Which is better %{ A | }?")
  -> ['Which is better A?', 'Which is better?']      # empty branch collapses the space before `?`
```

That collapse is deliberate (`collapse_empty_alternative_whitespace` in
`sase-core/crates/sase_core/src/agent_launch/mod.rs`, documented as avoiding "a space stranded against punctuation"),
which is direct evidence that `%{...}` immediately followed by punctuation is an intended authoring shape. Only the
input widget's insertion gate lags behind it.

## Current state (research summary)

- The whole gap is one predicate: `_next_char_allows_alt_brace_pair` in
  `src/sase/ace/tui/widgets/_alt_syntax_editing.py:38`, which accepts only EOF, whitespace, and
  `_PAIR_SAFE_CLOSE_CHARS = frozenset(")]}>")`. It is called from `plan_alt_brace_pair` (line 46) and has exactly one
  call site. Verified today:

  ```
  plan_alt_brace_pair("Which is better %",  17) -> TextEdit(start=17, end=17, text='{  }', cursor=19)
  plan_alt_brace_pair("Which is better %?", 17) -> None      # <-- the bug
  plan_alt_brace_pair("Which is better %)", 17) -> TextEdit(...)
  ```

  With `None`, `_try_prompt_text_pair_edit` falls through to `plan_pair_insert`, whose `_next_char_allows_pair` gate is
  identically strict, so the `{` is inserted as a bare literal.

- **No dispatch change is needed.** `_prompt_text_area_key_handling.py:591` already routes a typed `{` to
  `plan_alt_brace_pair` (after close-skip, before generic pair insert), and `_try_jinja_auto_pair` does not match `%{`
  (its guard requires `line[col - 1] == "{"`).

- **No Rust core change is needed.** The boundary rule points shared backend behavior at `../sase-core`, but the core
  only owns _parsing_ (`alt_directive_re` = `(?m)(^|[\s\(\[\{"':])(%(?:alt)?\(|%\{)`, which constrains only what
  precedes `%{`). A grep of `crates/` for `brace_pair` / `auto_pair` / `autopair` finds nothing: the insert-mode pairing
  planners are deliberately client-side, mirrored per editor. This matches the predecessor plan
  (`plans/202606/alt_brace_two_space_padding.md`), which reached the same conclusion.

- **`sase-nvim` mirrors this predicate and must move with it.** `lua/sase/alt_edit.lua` opens with "This mirrors the ACE
  prompt-input editing helpers in `src/sase/ace/tui/widgets/_alt_syntax_editing.py` so Neovim and the TUI behave
  identically while typing an alt directive", and carries a line-for-line `PAIR_SAFE_CLOSE` table plus
  `next_char_allows_brace_padding`. The predecessor change landed as two commits, one per repo (sase `36810c583`,
  sase-nvim `6279dbb`), driven by a single tale plan — follow that shape.

- `docs/ace.md` is already stale on this feature: the "Alt Brace Syntax (`%{...}`)" section still describes the
  pre-padding expansion `%{|}`, which `36810c583` replaced with `%{ | }`. Fix that while documenting the new rule.

## Design

### 1. Relax the follow-character gate (`_alt_syntax_editing.py`)

Add a second named character class next to `_PAIR_SAFE_CLOSE_CHARS` and accept the union:

```python
# Trailing punctuation can never begin a token, so a padded ``%{  }`` inserted
# directly before it is unambiguous -- this is how a fan-out question is
# normally authored (``Which is better %{ A | B }?``).
_PAIR_SAFE_PUNCTUATION_CHARS = frozenset(".,;:!?")
_PAIR_SAFE_FOLLOW_CHARS = _PAIR_SAFE_CLOSE_CHARS | _PAIR_SAFE_PUNCTUATION_CHARS
```

and have `_next_char_allows_alt_brace_pair` test `following in _PAIR_SAFE_FOLLOW_CHARS`. Keep the two source sets
separate rather than widening `_PAIR_SAFE_CLOSE_CHARS` in place, so the constant names stay honest about _why_ each
character is safe. Update the docstring to name the new class. Nothing else in the module changes.

Deliberately **excluded** from the punctuation class, because each can legally _open_ a token or span and padding before
it would be a guess: `"` `'` (open/close ambiguous), `*` `_` (Markdown emphasis), `-` (flags, list markers, ranges), `%`
`#` `@` `$` (directive / xprompt / reference sigils), and word characters (the existing `%{word` rejection).

### 2. Leave the generic pairing path alone

`plan_pair_insert` / `_next_char_allows_pair` in `_paired_text_editing.py` keep their strict gate. The relaxation is
justified by the explicit `%` opener, which makes intent unambiguous; a bare `{` before `?` has no such signal. So
`word` + `{` before `?` must still produce the literal `word{?`, and a test pins that.

### 3. Mirror in `sase-nvim` (`lua/sase/alt_edit.lua`)

Open the repo with the `/sase_repo` skill (`sase repo open sase-nvim -r "..."`) and use the printed path only. Add the
sibling table and widen the same predicate:

```lua
local PAIR_SAFE_PUNCTUATION = {
  ["."] = true, [","] = true, [";"] = true, [":"] = true, ["!"] = true, ["?"] = true,
}
```

`next_char_allows_brace_padding` then returns true when `following` matches `%s`, or is in `PAIR_SAFE_CLOSE`, or is in
`PAIR_SAFE_PUNCTUATION`. Use table lookups (not Lua patterns) so no character needs escaping. `M.plan_brace_padding`,
`on_insert_char`, and `apply_plan` are unchanged. Mirror the Python comment so the two files still read as one rule.

This is a separate commit in a separate repo; land it after the sase-side commit.

## Tests

### Python — `tests/ace/tui/widgets/test_prompt_alt_syntax_editing.py`

Pure-helper tests (public `plan_alt_brace_pair` only; do not import the new private constants):

1. Parametrize over `.,;:!?`: `plan_alt_brace_pair("%" + punct, 1) == TextEdit(start=1, end=1, text="{  }", cursor=3)`.
2. Extend/duplicate `test_plan_alt_brace_pair_requires_safe_following_character` to pin the exclusions:
   `plan_alt_brace_pair('%"', 1)`, `plan_alt_brace_pair("%*", 1)`, and `plan_alt_brace_pair("%-", 1)` are all `None`,
   alongside the existing `plan_alt_brace_pair("%word", 1) is None`.

Textual integration tests (mirroring the existing `AltEditTestApp` style):

3. `load_text("Which is better ?")`, cursor `(0, 16)`, `pilot.press("%", "{")` → text `"Which is better %{  }?"`, cursor
   `(0, 19)`.
4. Continue from (3) with `pilot.press("A", "|", "B")` → `"Which is better %{ A | B }?"`, cursor `(0, 24)` — proves the
   separator path composes inside a span that is now correctly closed.
5. Regression guard for the untouched generic path: `load_text("word?")`, cursor `(0, 4)`, `pilot.press("{")` →
   `"word{?"` with cursor `(0, 5)` (no pair inserted).

All five expectations above were produced by running the real widget with the gate patched, so they are the actual
post-fix values, not predictions.

### Lua — `sase-nvim` `tests/alt_edit.lua`

6. `same(alt.plan_brace_padding("%{?", 2), { start = 2, stop = 2, text = "  ", cursor = 3 }, "brace padding before trailing punctuation")`
   (same plan shape as the existing `"%{"` and `"%{}"` cases).
7. A nil guard for an excluded character, e.g.
   `same(alt.plan_brace_padding('%{"', 2), nil, "brace padding rejects ambiguous quote")`, keeping the existing
   `"%{word"` nil case.

## Docs

`docs/ace.md`, section "Alt Brace Syntax (`%{...}`)" → **Auto-pair** bullet:

- Correct the stale expansion: typing `{` after a directive-valid `%` inserts `%{  }` and parks the cursor between the
  two padding spaces (currently written as `%{|}`).
- State the new rule: the expansion fires at end of line, before whitespace, before a bracket closer (`)`, `]`, `}`,
  `>`), and before trailing punctuation (`.`, `,`, `;`, `:`, `!`, `?`), so `Which is better %{ A | B }?` can be authored
  by inserting the fan-out before an existing `?`. It is still suppressed before word characters and other token-opening
  characters.
- Leave the generic auto-pair paragraph (~line 2874) as-is; it describes `plan_pair_insert`, which is unchanged. The alt
  section carries the exception.

`sase-nvim` `README.md`, section "Alt Brace Syntax (`%{...}`)" → extend the first **Editing** bullet with the same
follow-character rule (padding now also fires before trailing punctuation), keeping the existing "the plugin does not
insert the closing `}`" contract intact.

`CHANGELOG.md` needs no manual edit — it is generated from conventional commits. Use a `fix(tui):` subject for the sase
commit and `feat:`/`fix:` for the nvim mirror, matching `36810c583` / `6279dbb`.

## Edge cases / notes for the implementer

1. **Neovim has no closing brace of its own.** The plugin contributes only the two spaces and leaves `}` to the user's
   auto-pair plugin — and most such plugins apply the same conservative "not before a token character" rule this plan is
   relaxing. So in nvim, `%{` before `?` will likely yield `%{  ?` (padded, still unclosed) until the user types `}`.
   That is strictly better than today's `%{?` and keeps the two implementations honest about sharing one predicate; do
   not try to make `sase-nvim` insert `}` (an explicit hard requirement of the predecessor plan).
2. **Empty group still does not fan out.** `split_prompt_for_alternatives("Which is better %{  }?")` returns `None`,
   same as the padded-at-EOF case. No parser change; do not "fix" this.
3. **Pre-existing quirk, out of scope.** Typing `}` in `%{ A |}?` inserts a literal `}` rather than skipping over the
   padded closer (close-skip requires the closer to be the immediate next character, and the padding space intervenes).
   That predates this change and is unaffected by it.
4. `%(` / `%alt(` legacy shorthands go through generic paren pairing and are intentionally not touched.
5. Highlighting needs no change: `alt_inspect.tokenize` keys off the same `%{`-position rule and ignores what follows
   the closer. A `%{ A | B }?` span already highlights correctly once the closer exists.

## Files to change

- `src/sase/ace/tui/widgets/_alt_syntax_editing.py` — new punctuation class + widened
  `_next_char_allows_alt_brace_pair`.
- `tests/ace/tui/widgets/test_prompt_alt_syntax_editing.py` — tests 1-5.
- `docs/ace.md` — Alt Brace Syntax auto-pair bullet (new rule + stale-padding fix).
- `sase-nvim` linked repo (opened via `/sase_repo`, committed separately):
  - `lua/sase/alt_edit.lua` — `PAIR_SAFE_PUNCTUATION` + widened `next_char_allows_brace_padding`.
  - `tests/alt_edit.lua` — tests 6-7.
  - `README.md` — Alt Brace Syntax editing bullet.

## Validation

- sase repo: `just install` (ephemeral workspace may have stale deps), then `just check`.
- `sase-nvim`: from the checkout root, `nvim --headless -u NONE -c "set rtp+=." -l tests/alt_edit.lua`.
- Manual smoke in `sase ace`: type `Which is better ?`, move the cursor before the `?`, then type `%{A|B` and confirm
  the buffer reads `Which is better %{ A | B }?` with the alt delimiters highlighted (no `alt.error` span).
