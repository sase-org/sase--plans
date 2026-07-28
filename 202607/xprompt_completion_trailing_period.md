---
tier: tale
title: Trigger xprompt completion when a `#` reference is followed by a period
goal:
  Typing a `#` xprompt reference immediately before a `.` (or any other character that cannot continue an xprompt name)
  opens the xprompt completion menu and soft suggestion for the reference name, and accepting a candidate replaces only
  the reference — leaving the trailing punctuation intact.
---

- **AGENTS:**
  - [bbugyi200.athena.n3--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.n3.md#member-code)
- **COMMITS:**
  - [ad3c751](https://github.com/sase-org/sase/commit/ad3c75151077382cc7f77fe67556b77bb875aadb) — fix(xprompt): preserve trailing punctuation in completion

# Plan: Trigger xprompt completion when a `#` reference is followed by a period

## Problem

In the `sase ace` prompt input widget, typing an xprompt reference directly before a period does not trigger xprompt
completion. Concretely, with prompt text

```
... but not the phase title (see #ss.
```

and the cursor sitting between `#ss` and the trailing `.`, no completion menu opens and no soft (ghost) suggestion
appears, even though `#screenshot`-style entries are in the warm catalog. The same prompt without the trailing period
(`(see #ss`) completes normally, and the same prompt with a trailing `)` (`(see #ss)`) also completes normally.

This is a real workflow paper cut: the user typed `#` in front of an already-written sentence-ending period, so every
keystroke of the reference name was typed in the "broken" state and completion never came up.

The syntax highlighter already gets this right — `#ss` renders in the xprompt style and the `.` does not — which makes
the missing completion look like a bug rather than a rule.

## Reproduction (verified before writing this plan)

```python
from sase.ace.tui.widgets.file_completion import extract_token_around_cursor
from sase.ace.tui.widgets.directive_completion import extract_directive_token_around_cursor
from sase.ace.tui.widgets.xprompt_completion import build_xprompt_completion_candidates

extract_token_around_cursor("(see #ss.", 8)
# (5, 9, '#ss.')          <-- trailing '.' swallowed into the token

extract_token_around_cursor("(see #ss", 8)
# (5, 8, '#ss')           <-- fine without the period

build_xprompt_completion_candidates("#ss.", entries=[<entry named "ss">])
# ([], '')                <-- zero candidates, so nothing opens

extract_directive_token_around_cursor("(see %wa.", 8)
# (5, 8, '%wa')           <-- '%' directives already stop at the period
```

## Root cause

`extract_token_around_cursor()` in `src/sase/ace/tui/widgets/file_completion.py` is a _path-oriented_ token scanner. It
walks outward from the cursor until it hits `_TOKEN_DELIMITERS`:

```python
_TOKEN_DELIMITERS: frozenset[str] = frozenset("'\"`?!;,()[]{}<>|&=+*^%$:\\")
```

`.` is deliberately **not** a delimiter, because file paths need it (`./foo`, `../bar`, `main.py`). `-` is likewise not
a delimiter. So for a `#` token the scanner produces `#ss.` instead of `#ss`, and
`build_xprompt_completion_candidates()` then filters entries with `entry.name.lower().startswith("ss.")`, which matches
nothing.

That is a grammar mismatch. An inline xprompt reference name cannot contain `.` or `-` at all —
`src/sase/xprompt/_parsing_references.py`:

```python
XPROMPT_REFERENCE_NAME_FRAGMENT = (
    r"(?P<name>[a-zA-Z_][a-zA-Z0-9_]*(?:/[a-zA-Z_][a-zA-Z0-9_]*)*)"
)
```

So `.` unambiguously ends an xprompt reference name, exactly like the already-delimiting `)`, `!`, and `:`. The `%`
directive path already models this correctly with its own grammar-aware scanner (`_is_directive_identifier` in
`src/sase/ace/tui/widgets/_directive_completion_tokens.py`); the `#` path is the one that still borrows the path
scanner. This plan closes that gap.

## Affected surfaces

All three prompt completion surfaces funnel through the same extractor, so all three are broken and all three are fixed
by one change:

1. **Automatic menu while typing** — `_try_auto_xprompt_completion()` in
   `src/sase/ace/tui/widgets/_file_completion_open.py` → `_get_xprompt_token_context()`.
2. **Explicit `<ctrl+t>` menu** — `src/sase/ace/tui/widgets/_file_completion_tab.py` (dispatches on the raw token, then
   calls `_get_xprompt_token_context()`).
3. **Soft / ghost suggestion (`<ctrl+l>`)** — `build_prompt_soft_completion()` in
   `src/sase/ace/tui/widgets/prompt_completion.py`.

Plus the live re-narrowing path while a menu is open: `src/sase/ace/tui/widgets/_file_completion_refresh.py` →
`_get_token_context()` → `_get_xprompt_token_context()` in `src/sase/ace/tui/widgets/_file_completion_context.py`.

## Design

Introduce a dedicated, pure, grammar-aware extractor for `#` reference tokens and route the xprompt completion surfaces
through it. Leave the path-oriented `extract_token_around_cursor()` untouched so file-path completion, the `<ctrl+r>`
recursive finder, and prompt-word completion keep their current behavior.

### New extractor semantics

`extract_xprompt_token_around_cursor(line, col) -> XPromptTokenSpan | None`

1. Start from the existing `extract_token_around_cursor(line, col)` result so the _start_ boundary rules (including the
   `#!` standalone-marker special cases at `src/sase/ace/tui/widgets/file_completion.py:65-73`) are inherited unchanged.
   `None` in → `None` out.
2. If the raw token does **not** start with `#`, preserve today's behavior exactly: return the raw span when
   `is_xprompt_like_token(raw_token)` is true (this is the `/skill` case), else `None`. **Do not clamp `/` tokens** —
   `/usr/share/foo.txt` is a legitimate absolute-path token and clamping it at `.` would break path completion.
3. For a `#` / `#!` token: after the marker, extend forward only over characters that can continue an inline reference
   name — `[A-Za-z0-9_/]` — producing `name_end`.
4. If `col > name_end`, return `None`. The cursor has moved past the reference (e.g. `#ss.|` while the user types the
   next sentence), so xprompt completion must stay quiet and the caller falls through to its normal path/word handling.
5. Otherwise return
   `XPromptTokenSpan(start=start, end=name_end, token=line[start:name_end], clamped=name_end < raw_end)`.

Worked examples (`|` marks the cursor):

| Input         | Result                           | Note                                        |
| ------------- | -------------------------------- | ------------------------------------------- |
| `(see #ss\|.` | `(5, 8, "#ss", clamped=True)`    | the reported bug; completion now opens      |
| `(see #ss.\|` | `None`                           | cursor past the reference; stays quiet      |
| `(see #ss\|`  | `(5, 8, "#ss", clamped=False)`   | unchanged from today                        |
| `(#n\|)`      | `(1, 3, "#n", clamped=False)`    | unchanged; `)` is already a delimiter       |
| `#bd/wo\|.`   | `(0, 6, "#bd/wo", clamped=True)` | `/` stays part of the name                  |
| `#!sy\|.`     | `(0, 4, "#!sy", clamped=True)`   | standalone marker preserved                 |
| `#\|`         | `(0, 1, "#", clamped=False)`     | bare `#`; callers still keep the menu shut  |
| `~/foo\|.py`  | `None`                           | not a `#` token; path completion unaffected |
| `.#ss\|`      | `None`                           | see non-goals                               |

### Accept semantics

Because the returned span ends at `name_end`, every accept path already does the right thing:
`_replace_token_text(row, start, end, insertion)` rewrites only `#ss`, so `#ss.` becomes `#screenshot.` with the cursor
parked before the period. The skeleton accept path (`_accept_xprompt_completion_candidate()` in
`src/sase/ace/tui/widgets/_file_completion_accept.py`) already reads `next_char = line[end]` and
`append_text_arg_space = end == len(line)`, so it will see `.` as the next character and suppress the trailing spacer —
the same handling the existing `)` tests exercise. No changes are needed in the accept code; add tests to lock the
behavior in.

### Dotted-name guard (`clamped`)

`validate_xprompt_name()` in `src/sase/xprompt/naming.py` permits `.` and `-` in a saved xprompt name, even though
`XPROMPT_REFERENCE_NAME_FRAGMENT` means such a name can never be expanded from an inline `#` reference. Without a guard,
a clamped token would let a dotted candidate be inserted at the clamped range and mangle the text: with an entry named
`foo.bar` and text `#foo|.b`, accepting would yield `#foo.bar.b`.

Guard: when `clamped` is true, restrict candidates to names that are valid inline reference names. Thread the flag as a
keyword argument into `build_xprompt_completion_candidates()` (`inline_reference_only: bool = False`). Non-clamped
tokens keep today's candidate set byte for byte, so this cannot regress ordinary completion. The only behavior given up
is offering a never-expandable dotted name from a clamped token, which is the correct trade.

## Implementation steps

1. **Expose the reference-name grammar (`src/sase/xprompt/naming.py`).** Add two small public helpers built on
   `XPROMPT_REFERENCE_NAME_FRAGMENT` from `._parsing_references`, so the grammar has a single source of truth and the
   TUI layer does not reach into another package's private module:
   - `is_inline_reference_name_char(character: str) -> bool` — true for ASCII letters, digits, `_`, and `/`. Use
     explicit ASCII range checks, not `str.isalnum()`, so non-ASCII input cannot widen the grammar.
   - `is_inline_reference_name(name: str) -> bool` — `re.fullmatch` against the fragment.

   Add both to `naming.py`'s `__all__`. `naming.py` is already the public naming home and is already imported by the TUI
   (`src/sase/ace/tui/modals/unified_xprompt_save_modal.py`). If importing `._parsing_references` from `naming.py`
   creates an import cycle, put the two helpers in a new `src/sase/xprompt/reference_names.py` instead and import that
   from both places.

2. **Add the extractor (`src/sase/ace/tui/widgets/xprompt_completion.py`).** Add a frozen, slotted `XPromptTokenSpan`
   dataclass (`start`, `end`, `token`, `clamped`) and `extract_xprompt_token_around_cursor(line, col)` implementing the
   semantics above. It may import `extract_token_around_cursor` from `file_completion` (this module already imports
   `CompletionCandidate` from there, so no new dependency direction).

3. **Add the candidate guard (same file).** Give `build_xprompt_completion_candidates()` a keyword-only
   `inline_reference_only: bool = False`. When true, skip entries whose `name` fails `is_inline_reference_name()`. Apply
   the filter before the shared-prefix computation so `shared_extension` stays consistent with the returned candidates.

4. **Route the widget context helpers (`src/sase/ace/tui/widgets/_file_completion_context.py`).**
   - `_get_xprompt_token_context()` → use `extract_xprompt_token_around_cursor()` and return
     `tuple[int, XPromptTokenSpan] | None` (row plus span).
   - `_get_token_context()` → adapt its `"xprompt"` branch to unpack the span back into the `(row, start, end, token)`
     shape its generic callers expect.
   - `_build_xprompt_completion_candidates()` and `_build_warm_xprompt_completion_candidates()` → accept keyword-only
     `inline_reference_only: bool = False` and pass it through.

5. **Automatic menu (`src/sase/ace/tui/widgets/_file_completion_open.py`, `_try_auto_xprompt_completion`).** Consume the
   span; keep the existing `len(token) < 2` quiet-on-bare-marker guard (so typing `#` in front of a period still opens
   nothing until an identifier character follows); pass `inline_reference_only=span.clamped` into the warm candidate
   builder.

6. **`<ctrl+t>` menu (`src/sase/ace/tui/widgets/_file_completion_tab.py`).** Replace the
   `is_xprompt_like_token(raw_token)` dispatch with a single `self._get_xprompt_token_context()` call: when it returns a
   span, set `_completion_kind = "xprompt"` and build candidates from the span (passing
   `inline_reference_only=span.clamped`); otherwise fall through to the existing `is_path_like_token(raw_token)` branch
   and then prompt-word completion. This also removes the current double extraction.

7. **Soft completion (`src/sase/ace/tui/widgets/prompt_completion.py`, `build_prompt_soft_completion`).** Before the
   generic `extract_token_around_cursor()` call, try `extract_xprompt_token_around_cursor(line, col)`; when it returns a
   span and `xprompt_entries` is not `None`, build candidates with `inline_reference_only=span.clamped` and return a
   `_line_suggestion(...)` over `span.start`/`span.end`. Keep the existing generic extraction for the file-path branch
   that follows, and keep the current ordering (jinja → xprompt args → directive → xprompt → file).

8. **Live re-narrowing (`src/sase/ace/tui/widgets/_file_completion_refresh.py`).** In the branch that rebuilds
   candidates for an open menu, call `_get_xprompt_token_context()` directly for `_completion_kind == "xprompt"` so the
   `clamped` flag reaches `_build_xprompt_completion_candidates()`. Leave the other kinds on `_get_token_context()`.

9. **Sanity-check the neighbors.**
   - `_soft_completion_may_need_xprompt_entries()` in `src/sase/ace/tui/widgets/_prompt_soft_completion.py` already
     short-circuits to `True` when `#` is anywhere in the text, so the warm catalog is requested for the fixed case with
     no change.
   - `_structured_completion_claims_cursor()` in `_file_completion_refresh.py` already returns `True` for `#ss.`; no
     change required. Do not expand its behavior in this plan.

## Tests

Follow the existing conventions in `tests/ace/tui/widgets/`; `_entry()` / `_seed_entries()` helpers and the
`CompletionTestApp` harness already exist there. Mirror the existing `..._skips_space_before_punctuation` tests, which
cover the `)` case, and add `.` coverage.

1. **Pure extractor** — new tests in `tests/ace/tui/widgets/test_xprompt_completion.py` covering every row of the
   worked-examples table above, including the `None` cases (`#ss.|`, `~/foo.py`, `.#ss`) and the `clamped` flag values.
2. **Path scanner untouched** — assert in `tests/ace/tui/widgets/test_file_completion_module.py` that
   `extract_token_around_cursor("(see #ss.", 8)` still returns `(5, 9, "#ss.")`, documenting that the path scanner is
   intentionally unchanged.
3. **Automatic menu** — in `tests/ace/tui/widgets/test_auto_xprompt_completion.py`: load `(see #s.` with the cursor
   before the `.`, assert the menu opens with the expected candidate and `_completion_kind == "xprompt"`; and assert
   that with the cursor _after_ the `.` no menu opens.
4. **`<ctrl+t>` accept** — in `tests/ace/tui/widgets/test_xprompt_completion.py`: single-candidate accept on `(see #n.`
   yields `(see #none.` with the cursor immediately before the `.` and no trailing spacer inserted; add a required-input
   entry variant asserting the skeleton lands before the period.
5. **Soft suggestion accept** — in `tests/ace/tui/widgets/test_prompt_live_completion.py`: `<ctrl+l>` on `(see #r.`
   yields `(see #review.`.
6. **Dotted-name guard** — with an entry named `foo.bar` and text `#foo|.b`, assert no candidate is offered from the
   clamped token (so the text can never become `#foo.bar.b`), while the same entry still completes normally from the
   unclamped token `#foo|`.

## Non-goals

Call these out in the final report rather than fixing them here:

- **`.` immediately _before_ the `#`** (`.#ss`) stays quiet. `XPROMPT_REFERENCE_LEADING_CONTEXT`
  (`(?:^|(?<=\s)|(?<=[(\[{"']))`) does not accept `.` as a leading context, so the launcher would not expand such a
  reference; offering completion there would promise an expansion that never happens.
- **Slash-skill tokens followed by `.`** (`/sase_plan.`) remain unhandled, because a `/`-leading token is ambiguous with
  an absolute file path.
- **Leading-context strictness.** The generic delimiter set is broader than the reference grammar's leading context
  (e.g. `foo;#ss` yields a `#ss` token that the launcher would not expand). This pre-existing mismatch is unchanged.
- Nothing here crosses the Rust core backend boundary: this is prompt-widget token scanning and presentation only. No
  `sase-core` changes.

## Verification

- `just install` first (workspace virtualenvs are ephemeral and may be stale), then `just check`.
- Targeted runs while iterating:
  `just test tests/ace/tui/widgets/test_xprompt_completion.py tests/ace/tui/widgets/test_auto_xprompt_completion.py tests/ace/tui/widgets/test_prompt_live_completion.py tests/ace/tui/widgets/test_file_completion_module.py`
- Manual smoke in `sase ace`: open the prompt bar, type a sentence ending in `.`, move the cursor before the period,
  type `#` then a couple of identifier characters, and confirm the menu opens and that accepting leaves the period in
  place.
