---
tier: tale
title: Demote the owning-xprompt preview inside shorthand argument text
goal:
  "In prompt NORMAL mode, `K` (and `Ctrl+]`) prefer a nested reference, file path,
  glossary term, or plain word under the cursor when the cursor sits inside `#name: ` /
  `#name:: ` argument text, falling back to the xprompt that owns that text only when
  nothing else matches."
size: medium
proposed_by: bbugyi200.athena.01v
create_time: 2026-08-14 18:04:53
status: done
---

- **PROMPT:**
  [prompts/202608/k_preview_shorthand_arg_text.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/k_preview_shorthand_arg_text.md)

# `K` should stop previewing the owning xprompt from inside `#name: ` / `#name:: ` argument text

## Problem

In prompt NORMAL mode, `K` (`_preview_token_under_cursor`) resolves what is under the
cursor through `detect_preview_target_at_cursor()` in
`src/sase/ace/tui/widgets/_prompt_preview_target.py`. That detector walks
`iter_xprompt_references(text)` and returns the first reference whose
`ref.start <= offset < ref.end`.

For the shorthand argument forms, `ref.end` is not the end of the reference head — it is
the end of the _argument text_:

- `#name: text` (single colon + space) runs to the next blank line
  (`find_shorthand_text_end`).
- `#name:: text` (double colon + space) runs to the next line-initial xprompt directive
  or EOF (`find_double_colon_text_end`).
- `#name(args): text` / `#name(args):: text` behave the same after the closing paren.

Observed with the current parser (`iter_xprompt_references`):

```text
'run #foo: fix the bug in src/main.py now'   -> foo colon_shorthand        (4, 40)
'run #foo:: line one\n\nline two\n#bar do it' -> foo double_colon_shorthand (4, 40)
                                                bar none                   (30, 34)
```

So every offset in the argument prose belongs to the reference span, and `K` previews
`#foo` no matter where the cursor sits inside it. Everything else `K` can do is
unreachable there, because those paths only run when detection returns `None`:

- a nested `#bar` inside the argument text (the outer reference matches first),
- a file path in the argument text (`iter_file_path_matches` runs after the reference
  loop),
- a glossary term (`_preview_glossary_under_cursor`),
- plain-word definition / spellcheck (`_lookup_word_under_cursor`).

This is worst with `#name:: `, where the argument text can swallow the entire rest of
the prompt, making `K` return the same xprompt preview everywhere.

## Desired behavior

When the cursor is inside the _free text_ of a `: ` / `:: ` shorthand argument, `K` must
prefer every other lookup it knows before falling back to the owning xprompt:

1. known `/skill` span (already wins today — the skill branch runs first),
2. a nested `#name` reference inside the argument text,
3. a file path inside the argument text,
4. a glossary term,
5. plain-word definition / spellcheck,
6. **last resort only:** the enclosing shorthand reference — i.e. exactly the preview
   `K` gives today.

Step 6 is the reading of "prefer all other functionalities of the `K` keymap before
defaulting to this behavior": the owning-xprompt preview is demoted to the default of
last resort rather than deleted, so `K` on whitespace, punctuation, or an
identifier-shaped token (word lookup rejects runs containing digits/underscores) inside
a long `#name:: ` block still reaches the xprompt that owns the block. **If the intent
was instead to suppress the owning-xprompt preview entirely inside argument text, drop
step 6 — that is deleting one fallback call in each of the two mixins below and
inverting the two tests that assert it.**

Unchanged by this plan:

- Cursor on the reference head (`#foo`, `#!foo`, `#foo!!`), on the `:` / `::`, or on the
  separating space still previews the xprompt.
- Cursor inside parenthesized args (`#foo(x=1)`) still previews the xprompt.
- The inline colon-arg form `#foo:bar` (no space) still previews the xprompt from
  anywhere in `bar`; the existing test
  `test_detects_xprompt_reference_name_and_arg_region` must stay green.

### Decision: `Ctrl+]` moves with `K`

`detect_jump_target_at_cursor()` in `src/sase/ace/tui/widgets/_prompt_jump_target.py`
delegates to `detect_preview_target_at_cursor()`, so jump-to-definition inherits the
detector change whether or not we want it. Rather than parameterize the detector to keep
the two keys asymmetric, apply the same precedence to `Ctrl+]`: glossary jump first,
then the enclosing shorthand reference as the last resort. With step 6 in place,
`Ctrl+]` loses nothing — it only gains glossary/file/nested-reference targets inside
argument text.

### Note: no Rust core change is needed

Per the `rust_core_backend_boundary` memory, check the sibling core before changing
shared behavior. The core's editor/LSP hover path
(`crates/sase_core/src/editor/token.rs::extract_token_at_position`) is delimiter-based:
`:` is a token delimiter, so hovering inside `#foo: some text` already extracts the
plain word, not the xprompt. This change makes the ACE prompt widget converge on the
behavior the LSP already has; the canonical grammar is untouched. Open the core repo
with `sase repo open sase-core -r "<reason>"` if you want to confirm this before
starting — do not clone or web-fetch it.

## Implementation

### 1. Teach the reference parser where shorthand text starts

`src/sase/xprompt/_parsing_references.py`

The detector must not re-derive the shorthand boundary rules (they involve paren
matching, HITL suffixes, and two different terminator scans). Have the parser that
already computes them report the offset.

- Add a field to `XPromptReference` (frozen dataclass; append it last, after
  `hitl_override`, with a default so the keyword construction in
  `src/sase/agent/xprompt_swarm.py` keeps working):

  ```python
  shorthand_text_start: int | None = None
  """Offset where ``: ``/``:: `` free-text argument content begins, if any."""
  ```

  It is `None` for `NONE`, `PLUS`, `COLON` (`#foo:bar`), plain `PAREN`, and for a
  `#foo(a):tok` paren-plus-bare-token tail; it is set for `COLON_SHORTHAND`,
  `DOUBLE_COLON_SHORTHAND`, and for `PAREN` followed by `): ` / `):: `.

- Replace private `_reference_end(prompt, match) -> int` with
  `_reference_span(prompt, match) -> tuple[int, int | None]` returning
  `(end, shorthand_text_start)`. Its branch structure already matches one-to-one with
  the text starts (`match.end() + 2` for `": "`, `match.end() + 3` for `":: "`,
  `paren_end + 3` / `paren_end + 4` for the post-paren forms). `_reference_end` has no
  other callers (`rg '_reference_end\b'`), so this is a local rename.

- `xprompt_reference_from_match()` unpacks the tuple and passes both through. Leave
  `argument_source`, `arg_kind`, `raw`, and `end` semantics exactly as they are — other
  consumers (`_xprompt_arg_assist_detection.py`, `xprompt_inspect.tokenize`,
  `used_xprompts`, the launch path) must see no change.

- Guard the degenerate case: when the argument text is empty (`#foo: ` immediately
  followed by a blank line) the computed start can equal `end`; that yields an empty
  region, which the "cursor inside text" test below already treats as "not inside".

### 2. Skip shorthand text in preview detection, and expose the owner

`src/sase/ace/tui/widgets/_prompt_preview_target.py`

- Add a module-private predicate:

  ```python
  def _cursor_in_shorthand_argument_text(ref: XPromptReference, offset: int) -> bool:
      start = ref.shorthand_text_start
      return start is not None and start <= offset < ref.end
  ```

- In `detect_preview_target_at_cursor()`, inside the `iter_xprompt_references` loop,
  `continue` instead of returning when the predicate is true. Because references are
  yielded in match order and a nested reference starts later, `continue` lets a nested
  `#bar` inside the argument text match on the next iteration, and lets the file-path
  branch below the loop match a path inside the argument text.

- Add a public last-resort helper next to it, exported in `__all__`:

  ```python
  def detect_shorthand_argument_owner_at_cursor(
      text: str, cursor_offset: int
  ) -> PreviewToken | None:
      """Return the xprompt whose ``: ``/``:: `` argument text holds the cursor."""
  ```

  Build the identical `PreviewToken` today's code would return (same `kind`, `raw`,
  `target`, `start`, `end`), so the fallback reproduces current behavior byte for byte.
  When shorthand blocks nest (`#foo: hi #bar: yo` — both own offset of `yo`), return the
  innermost owner, i.e. the matching reference with the greatest `start`. Do not take
  `known_skills`; this only inspects `#` references.

### 3. Wire the fallback into the `K` flow

`src/sase/ace/tui/widgets/_prompt_preview.py`, `_preview_token_under_cursor()`

Current order in the `token is None` branch is glossary → word lookup → warning toast.
Insert the owner fallback between word lookup and the warning:

```python
if token is None:
    preview_glossary = getattr(self, "_preview_glossary_under_cursor", None)
    if callable(preview_glossary) and preview_glossary():
        return
    if self._lookup_word_under_cursor():
        return
    token = detect_shorthand_argument_owner_at_cursor(self.text, offset)
    if token is None:
        self.notify(
            "Move the cursor onto an xprompt, skill, file path, glossary term, "
            "or word to look it up",
            severity="warning",
        )
        return
```

then fall through to the existing `run_worker(self._resolve_preview_async(...))` call
(restructure so the worker launch is shared by both paths rather than duplicated).

### 4. Mirror it in the `Ctrl+]` flow

- `src/sase/ace/tui/widgets/_prompt_jump_target.py`: add

  ```python
  def detect_shorthand_argument_owner_jump_target(
      text: str, cursor_offset: int
  ) -> JumpToken | None:
  ```

  which calls `detect_shorthand_argument_owner_at_cursor()` and reuses the existing
  private `_jump_token_from_preview_token()`.

- `src/sase/ace/tui/widgets/_prompt_jump.py`, `_jump_to_definition_under_cursor()`: in
  the `token is None` branch, after the glossary jump attempt and before the warning
  toast, try the new helper and continue with it when it resolves.

## Tests

Add to `tests/test_xprompt_references.py` (parser-level, no TUI):

- `shorthand_text_start` offsets for `#foo: text`, `#foo:: text`, `#foo(x=1): text`,
  `#foo(x=1):: text`, `#foo!!: text`, `#!foo:: text`, and a multi-line `#foo:: ` block
  that contains a blank line.
- `shorthand_text_start is None` for `#foo`, `#foo:bar`, `#foo+`, `#foo(x=1)`, and
  `#foo(x=1):bar`.
- The reported start points at the first character of the text (not at the colon or the
  separating space): `text[ref.shorthand_text_start]` is the first text character.

Add to `tests/ace/tui/widgets/test_prompt_preview_target.py` (uses the existing
`_detect(text, needle)` helper):

- Cursor on `#foo` / on the `:` / on the space of `#foo: some text` still returns the
  xprompt token; cursor inside `some text` returns `None`.
- Same for `#foo:: `, including a cursor on a later line of a multi-line block.
- `run #foo:: line one\n\nline two\n#bar do it`: cursor on `#bar` returns the `#bar`
  xprompt token (today it returns `#foo`).
- `#foo: fix src/main.py now`: cursor on `src/main.py` returns the file token.
- `#foo(x=1): text`: cursor inside `x=1` still returns the xprompt token; cursor inside
  `text` returns `None`.
- `detect_shorthand_argument_owner_at_cursor` returns the owning token for cursors
  inside argument text, `None` for cursors on the head/`#foo:bar`/plain prose, and the
  innermost owner for `a #foo: hi #bar: yo` at `yo`.
- Regression: `test_detects_xprompt_reference_name_and_arg_region` (the `#foo:bar`
  inline form) is untouched and still passes.

Add widget-level tests next to the existing `K` tests, following the `PromptPage`
harness used in `tests/ace/tui/widgets/test_prompt_glossary_navigation.py`
(`async with PromptPage(text, cursor=(row, col)) as page: await page.press("K")`):

- `K` on a plain word inside a `#foo:: ` block runs word lookup instead of pushing the
  xprompt preview (monkeypatch `_lookup_word_under_cursor` to record the call, mirroring
  how the glossary test asserts the inverse).
- `K` on whitespace/punctuation inside the same block still previews `#foo` (the
  last-resort path), asserted via the scheduled preview worker/token rather than a live
  xprompt load.
- One `Ctrl+]` counterpart: jump from a glossary term inside argument text resolves the
  glossary definition rather than the enclosing xprompt.

## Docs

`docs/ace.md`:

- In the prompt NORMAL-mode paragraph (search for "In prompt NORMAL mode, `K` previews
  the xprompt, slash skill, or file under the cursor."), add a sentence: inside
  `#name: ` / `#name:: ` argument text, `K` and `Ctrl+]` prefer a nested reference, file
  path, glossary term, or plain word under the cursor, and fall back to the xprompt that
  owns the argument text only when nothing else matches.
- Keymap-table rows for `K` and `Ctrl+]` are still accurate; only extend them if the
  wording stays within the table's existing column formatting.

No `sase/memory/*.md`, `AGENTS.md`, or provider-shim edits are in scope. The help-modal
label `("K", "Preview xprompt/skill/file/word")` in
`src/sase/ace/tui/modals/help_modal/binding_common.py` stays as is (it is capped at 32
characters). `CHANGELOG.md` is release-please generated — do not hand-edit it.

## Verification

```bash
just install            # ephemeral workspaces may have stale deps
just check              # whole-repo lint gates + diff-scoped tests
```

Targeted while iterating:

```bash
.venv/bin/python -m pytest tests/test_xprompt_references.py \
  tests/ace/tui/widgets/test_prompt_preview_target.py \
  tests/ace/tui/widgets/test_prompt_glossary_navigation.py
```

Because the change touches the shared xprompt parser used by the launch path, run
`just check-full` before landing, through `/sase_monitor`
(`sase monitor start --command 'just check-full' …` with a `--next` action) — never
inline.

## Out of scope

- The `#foo:bar` inline colon-arg form keeps previewing the xprompt from its argument.
- Fenced-code/inline-code protection: `detect_preview_target_at_cursor` filters literal
  zones for `/skill` spans (via `xprompt_inspect.tokenize`) but not for `#` references,
  so `` `#foo` `` in backticks still previews. That asymmetry predates this change;
  leave it alone here.
- The preview error toast interpolates `token.raw`, which for a shorthand reference is
  the whole multi-line span. That is today's behavior and stays unchanged.
- No changes to `../sase-core`, to xprompt argument-assist completion
  (`_xprompt_arg_assist_detection.py`), or to prompt syntax highlighting.
