---
tier: tale
title: Stop directive completion from replacing prose to the right of the cursor
goal:
  Accepting a `%wait` completion in the prompt input widget replaces only the value
  under the cursor and leaves the rest of the line intact, for every unterminated
  directive argument body and for both the ACE prompt widget and the xprompt LSP.
size: medium
proposed_by: bbugyi200.athena.0g2
---

# Plan: Bound unterminated directive argument bodies at the value, not the line

## 1. The Reported Bug

Accepting a completion for the `%wait` directive in the ACE prompt input widget deletes
everything to the right of the cursor on that line.

Reproduction (no TUI needed — the defect is visible in the classifier the accept path
uses):

```bash
.venv/bin/python -c "
from sase.ace.tui.widgets._directive_completion_tokens import classify_directive_completion
line = '%wait:co and then do the thing'
c = classify_directive_completion(line, len('%wait:co'))
print(c.kind, repr(c.token), (c.start, c.end), repr(line[c.start:c.end]))
"
```

Today this prints a replacement range of `(6, 30)` covering `'co and then do the thing'`
— the whole rest of the line. Accepting the `coder` row therefore rewrites the line to
`%wait:coder`, destroying the user's prose. The expected range is `(6, 8)` covering
`'co'`, so that accepting yields `%wait:coder and then do the thing`.

## 2. Root Cause

The replacement range is produced by the shared Rust core, not by ACE.
`detect_directive_context_at_position` in `crates/sase_core/src/editor/directive.rs`
(repo `sase-core`) returns `replacement_range` for a directive-argument context as
`arg_start .. arg_end`, and `arg_end` is derived from the end of the argument **body**.
When the body has no closing delimiter, the body end is the end of the line, so the
range swallows all following prose.

Two code paths produce an unterminated body:

1. **`colon_arg_context` (line 1069)** special-cases `%wait`:

   ```rust
   let body_end = if metadata.name == "wait" {
       line.len()
   } else {
       line[cursor..]
           .find(char::is_whitespace)
           .map(|offset| cursor + offset)
           .unwrap_or(line.len())
   };
   ```

   The special case exists so `%wait:planner, co` can treat the space after the comma as
   part of a target list. Its cost is that every colon-form `%wait` body runs to end of
   line.

2. **`parenthesized_arg_context` (line 1119)** uses
   `let body_end = close.unwrap_or(line.len());`, so a paren form the user has not
   closed yet also runs to end of line. This is **not** wait-specific — `%id(`,
   `%model(`, `%clan(` and `%final(` all reproduce it.

`comma_clause_context` then sets the active clause's `content_end` from that body end,
and `arg_end = content_end.max(arg_start)` becomes the end of the line.

Confirmed ranges on today's build (cursor marked by the token length in each call):

| line                             | cursor after | current replaced text      | correct                 |
| -------------------------------- | ------------ | -------------------------- | ----------------------- |
| `%wait:co and then do the thing` | `co`         | `co and then do the thing` | `co`                    |
| `%wait: do the thing`            | `:`          | `do the thing`             | `` (empty insert point) |
| `%wait(co and more prose`        | `co`         | `co and more prose`        | `co`                    |
| `%id(foo and more prose`         | `fo`         | `foo and more prose`       | `foo`                   |
| `%model(son and more prose`      | `son`        | `son and more prose`       | `son`                   |
| `%clan(rev and more prose`       | `rev`        | `rev and more prose`       | `rev`                   |
| `%final(sase and more prose`     | `sase`       | `sase and more prose`      | `sase`                  |

ACE's accept path does exactly what the range tells it to:
`FileCompletionAcceptMixin._accept_file_completion`
(`src/sase/ace/tui/widgets/_file_completion_accept.py:389`) reads the range through
`_get_token_context()` and calls
`_replace_token_text(row, start, end, selected.insertion)`
(`src/sase/ace/tui/widgets/_file_completion_context.py:184`), which replaces
`start..end` wholesale. The accept path is correct; the range is wrong.

### Why the Python guards do not catch it

`src/sase/ace/tui/widgets/_directive_completion_tokens.py` has two heuristics that
reject prose contexts: `_wait_fragments_are_structured` and
`_colon_argument_chars_are_valid`. Neither helps here:

- `_wait_fragments_are_structured` inspects `clause.token` and `clause.selected_values`.
  `clause.token` is only the text from the clause start **up to the cursor**, so prose
  sitting between the cursor and the (line-end) clause end is invisible to it.
- `_colon_argument_chars_are_valid` returns early for `%wait` and only runs for the
  colon form at all, so the paren-form cases are unguarded.

They are also Python-only. The xprompt LSP server consumes the same core context and has
no equivalent guard, so the identical destructive `textEdit` reaches Neovim. That is the
decisive reason the fix belongs in `sase-core` rather than in ACE: per
`sase memory read rust_core_backend_boundary`, behavior another frontend must match the
TUI on is core backend logic.

## 3. The Fix

Add one quote-aware helper to `crates/sase_core/src/editor/directive.rs` and use it
wherever a body would otherwise fall back to `line.len()`.

### 3.1 The rule

> An **unterminated** directive argument body ends at the first whitespace byte, outside
> quotes and `[[ ]]` text blocks, that is not immediately preceded by a comma.

Whitespace immediately after a comma stays inside the body, which is what keeps the
`%wait:planner, co` list affordance working. Every other whitespace byte ends the body,
which is what stops the range at the value under the cursor.

`comma_clause_context` already returns `None` when `cursor > body_end`, so this rule
also makes a cursor parked in prose past the directive yield **no** completion context
at all — in the core, so the LSP gets the protection ACE currently open-codes in Python.

### 3.2 Helper

Add near `split_top_level_clauses` / `QuoteState` (around line 1286-1379):

```rust
/// Byte offset where an unterminated directive argument body ends.
///
/// Colon-form `%wait` and a paren form the user has not closed yet have no
/// closing delimiter. Letting the body run to the end of the line makes the
/// completion replacement range swallow the prose after the cursor, so the
/// body ends at the first whitespace outside quotes and `[[ ]]` blocks that
/// does not immediately follow a comma separator.
fn unterminated_body_end(line: &str, body_start: usize) -> usize {
    let bytes = line.as_bytes();
    let mut index = body_start;
    let mut state = QuoteState::default();
    while index < bytes.len() {
        let consumed = state.consume(bytes, index);
        if consumed == 1
            && !state.in_quotes()
            && bytes[index].is_ascii_whitespace()
            && !(index > body_start && bytes[index - 1] == b',')
        {
            return index;
        }
        index += consumed;
    }
    bytes.len()
}
```

Reusing `QuoteState` is what makes the scan skip whitespace inside `"..."`, `'...'`,
`` `...` `` and `[[ ... ]]`, matching how `split_top_level_clauses` and
`find_matching_paren_quoted` already read a body.

### 3.3 Call sites

1. `colon_arg_context` (line 1069): delete the `metadata.name == "wait"` branch and
   compute `let body_end = unterminated_body_end(line, body_start);` for every colon and
   brace-shorthand body. Keep the existing `if metadata.name == "wait"` dispatch to
   `comma_clause_context` immediately below it — only the `body_end` computation
   changes.

   Applying the rule to non-wait colon bodies too is deliberate: it replaces the current
   "first whitespace at or after the cursor" scan, which lets a body contain whitespace
   the user typed before the cursor (`%model:opus and| text` yields a token of
   `opus and`). ACE rejects that through `_colon_argument_chars_are_valid`; the LSP does
   not. One rule in core removes the divergence.

2. `parenthesized_arg_context` (line 1119): change to
   `let body_end = close.unwrap_or_else(|| unterminated_body_end(line, body_start));`.
   Leave the `close.is_some_and(|close| cursor > close)` early return above it alone — a
   closed paren is a terminated body and stays authoritative, including for legitimately
   spaced keyword values such as `%clan(research, summary=my summary text)`.

3. No change to `comma_clause_context`, `classify_active_clause`, or any ACE Python
   file.

### 3.4 Worked examples the rule must produce

| line                                | cursor         | body_end | replacement range covers |
| ----------------------------------- | -------------- | -------- | ------------------------ |
| `%wait:co and then do the thing`    | after `co`     | 8        | `co`                     |
| `%wait: do the thing`               | after `:`      | 6        | empty range at 6         |
| `%wait:planner, co`                 | end of line    | 17       | `co`                     |
| `%wait:planner,`                    | end of line    | 14       | empty range at 14        |
| `%wait(planner, co and more prose`  | after `co`     | 17       | `co`                     |
| `%wait(planner, co) and more prose` | after `co`     | 17 (`)`) | `co`                     |
| `%id(foo and more prose`            | after `fo`     | 7        | `foo`                    |
| ``%wait:`my agent` ``               | after `` `my`` | 16       | `` `my agent` ``         |
| `%w:sase-59 Can you help me ... ,`  | end of line    | 10       | no context (`None`)      |

## 4. What This Deliberately Does Not Change

- **A closed paren body still wins.** `%clan(research, summary=my summary text)` keeps
  its whole-clause range.
- **The `, ` list affordance survives.** `%wait:planner, co` and `%wait(planner, co`
  still complete, because comma-adjacent whitespace stays inside the body.
- **The Python guards stay.** `_wait_fragments_are_structured` and
  `_colon_argument_chars_are_valid` still cover closed-paren prose
  (`%wait(sase-59 fix the bug ,)`) that a terminated body legitimately spans. Do not
  delete them as "now redundant".

## 5. Known Residual, and a Separate Defect to File

**Residual (accepted).** With the cursor immediately after a `, ` separator that is
followed by prose — `%wait:planner, |do the thing` — the body still reaches the space
after `do`, so accepting replaces `do`. That is the same "replace the word under the
cursor" contract every other directive already has, and it is a one-word loss rather
than a rest-of-line loss. Do not add a prose-detection heuristic to chase it.

**Separate defect — file a bead, do not fix here.** The completion grammar treats
`%wait:a, b` as a two-target list, but the runtime colon-argument grammar
(`_DIRECTIVE_PATTERN` in `src/sase/xprompt/_directive_types.py`) admits no whitespace,
so the runtime silently waits on `a` only:

```bash
.venv/bin/python -c "
from sase.xprompt.directives import extract_prompt_directives
print(extract_prompt_directives('%wait:planner, coder')[1].wait)
"   # -> ['planner']   (the menu implied both)
```

Before creating it, use `/sase_new_task`. Describe it as: colon-form `%wait` completion
offers comma-space target lists that the runtime parser truncates at the first
whitespace; either the runtime grammar should accept `, ` inside a colon-form wait list
or the completion should stop offering it. Do not change that behavior inside this plan
— it changes what the menu offers, which is a product decision separate from the range
bug.

## 6. Tests

### 6.1 Rust — `crates/sase_core/src/editor/directive.rs` (`mod tests`)

The existing `classify(text, character)` helper returns the `CompletionContext`; assert
on `context.replacement_range` (and on `detect_directive_context_at_position` returning
`None` where the rule says there is no context). Add:

- `unterminated_wait_colon_body_stops_at_prose` — `%wait:co and then do the thing` with
  the cursor after `co` replaces `co` only; `%wait: do the thing` with the cursor after
  the colon yields an empty range at the colon.
- `unclosed_paren_body_stops_at_prose` — `%wait(co and more prose`,
  `%id(foo and more prose` and `%model(son and more prose` each replace only the value
  under the cursor.
- `comma_adjacent_space_keeps_the_wait_list_body` — `%wait:planner, co` and
  `%wait(planner, co` still classify with a range covering `co` only.
- `quoted_wait_value_keeps_its_inner_space` — ``%wait:`my agent` `` with the cursor
  after `` `my`` spans the whole quoted value.
- `cursor_in_prose_past_a_directive_has_no_context` —
  `%w:sase-59 Can you help me get rid of the ,` at end of line returns `None` from
  `detect_directive_context_at_position`.

### 6.2 Python — `sase` repo

- `tests/ace/tui/widgets/test_directive_arg_extraction.py`: add range assertions using
  `classify_directive_completion`, mirroring the section 3.4 table. The existing
  `extract_directive_arg_token_around_cursor` tests only assert the partial token, so
  they cannot catch a range regression.
- `tests/ace/tui/widgets/test_wait_directive_completion_interactions.py`: add an
  end-to-end accept test in the style of
  `test_wait_arg_completion_replaces_only_active_fragment` — load
  `%wait:co and then do the thing`, put the cursor after `co`, drive
  `_try_file_completion_tab()` and the accept, and assert the resulting text is
  `%wait:coder and then do the thing`. This is the test that would have caught the
  reported bug.
- Consider one parity assertion in `tests/test_xprompt_directive_completion_parity.py`
  comparing the LSP `textEdit` range against the ACE clause range for
  `%wait:co and then do the thing`. The current parity harness
  (`tests/_xprompt_directive_completion_parity_lsp.py`) compares candidate rows and
  reads only `textEdit["newText"]`, so covering the range needs a small extension to
  expose `textEdit["range"]`. Skip it if the harness change grows beyond a few lines and
  note why in the patch description.

### 6.3 Existing tests to re-validate (expected to still pass)

These pin behavior the rule must preserve — check them explicitly rather than trusting
the suite:

- Rust: `wait_paren_keywords_are_not_offered_in_colon_form`,
  `wait_bead_value_is_a_keyword_value_clause`,
  `id_and_clan_keyword_values_and_conflicts_classify`,
  `quoted_and_text_block_commas_do_not_split_clauses` (the
  `summary=[[note: use ]] here, and more]]` case exercises the quote-aware scan),
  `utf16_positions_classify_the_active_wait_clause`,
  `incomplete_and_malformed_calls_still_classify`.
- Python: `tests/ace/tui/widgets/test_directive_arg_extraction.py`
  (`test_wait_arg_extraction_keeps_valid_comma_fragments`,
  `test_wait_arg_extraction_rejects_prose_comma_after_directive`,
  `test_wait_arg_extraction_tracks_keyword_and_target_values_to_the_right`,
  `test_directive_arg_extraction_redirects_model_at_suffix_to_effort`),
  `tests/ace/tui/widgets/test_wait_directive_completion_interactions.py`,
  `tests/test_xprompt_directive_completion_parity.py`.

## 7. Working Across The Two Repos

The change spans `sase-core` (the fix) and `sase` (the tests). Both are needed for a
green run.

1. Open the core checkout with your `/sase_repo` skill —
   `sase repo open sase-core -r "<reason>"` — and use the path it prints. Do not locate
   or clone it another way.
2. Make the Rust change and its tests there, then run `just check` from that checkout
   (it runs `./scripts/check.sh all`: fmt, clippy, and the Rust test suite).
3. Back in the `sase` workspace, run `just install`. That rebuilds `sase_core_rs`
   **and** the xprompt LSP server from the local core checkout, so the Python and parity
   tests exercise the fix. Skipping it leaves the old core in the venv and the new
   Python tests will fail for the wrong reason.
4. Add the Python tests, then run `just check` (see
   `sase memory read lint_and_test.md`). Hand it to `/sase_monitor` if it runs long; use
   `just check-full` through a monitor only if the scoped run escalates.

### The core revision pin

`sase-core-revision.txt` pins the core SHA that `sase` CI builds from, so `sase` CI
keeps running the **old** core until that pin moves. Do not hand-edit it to an unmerged
SHA.

- If the core fix is already on `sase-core`'s remote master when you finish, run
  `just ratchet-core-revision` and include the resulting pin bump in the `sase` patch.
- Otherwise leave the pin alone. `.github/workflows/core-pin-ratchet.yml` opens a bump
  PR on a schedule once the core commit lands, and the new Python tests go green with
  it. Say so plainly in the patch description so the transient window is not mistaken
  for a flake.

Both repositories become obligations of your turn's final declaration; `/sase_final`
will surface each one for a `commit` decision.

## 8. Done When

- `classify_directive_completion('%wait:co and then do the thing', 8)` reports a
  replacement range of `(6, 8)`.
- Accepting a `%wait` completion in the prompt input widget leaves every character to
  the right of the cursor untouched, in the colon form and in an unclosed paren form.
- The same holds for `%id(`, `%model(`, `%clan(` and `%final(` with an unclosed paren.
- `just check` passes in `sase-core`, and `just check` passes in `sase` after
  `just install`.
- A task bead exists for the `%wait:a, b` runtime-truncation defect from section 5.
