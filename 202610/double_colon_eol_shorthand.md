---
tier: tale
title: Support `::` at end of line for double-colon text shorthand
goal:
  "`#name::`, `#name(args)::`, and `%clan...::` followed by a line break bind the
  following lines as their free-text payload in expansion, launch planning, and TUI/LSP
  highlighting, exactly like the same-line `:: text` form."
size: medium
proposed_by: bbugyi200.athena.0uz
create_time: 2026-10-01 13:11:34
status: wip
---

# Plan: Support `::` at end of line for double-colon text shorthand

## Problem

The user typed this in the ACE prompt:

```
+sase
#research_swarm(gemini=true,grok=true,muse=true,image=true,image_model=gpt-6-astra)::
Can you do some research with the goal of ...
End your analysis with a ranked list of improvements ...
```

The documented `#name:: text` / `#name(args):: text` shorthand should bind the following
lines as the free-text argument. Instead the text after `::` is ignored. The TUI
highlights `::` as a delimiter plus a one-character argument and leaves the body below
it uncolored.

## Root cause

Every recognizer of the double-colon text delimiter hard-codes `":: "`, a double colon
followed by a space. When `::` ends the line, the next character is `\n`, so none of
them match, and each layer falls back to a different wrong interpretation. I reproduced
all of these against the current tree:

| Input                                       | Layer                                     | Current (wrong) result                                                                                                                                                                                |
| ------------------------------------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `#rs::\nhello\nworld`                       | `process_xprompt_references`              | `#rs` expands with **no** payload, and `::\nhello\nworld` is left as literal prompt text                                                                                                              |
| `#rs(g=yes)::\nhello`                       | `process_xprompt_references`              | same: the paren args bind, the payload doesn't, and `::\nhello` leaks                                                                                                                                 |
| `#foo(a=1)::\none\ntwo`                     | shared lexer `iter_xprompt_references`    | raw=`#foo(a=1)::`, arg_kind=PAREN, `parse_arguments()` → `([':'], {'a': '1'})`. The bare `):value` branch takes the second colon as an argument, so swarm detection (`_sole_xprompt_reference`) fails |
| `#foo::\none`                               | shared lexer                              | arg_kind=NONE, raw=`#foo`                                                                                                                                                                             |
| `foo::\none`                                | `parse_workflow_reference`                | `('foo', [':\none'], {})`                                                                                                                                                                             |
| `#a:: x\n#b::\ny`                           | `find_double_colon_text_end`              | the `#b::` line is not a boundary, so `#a` swallows `#b` and its body                                                                                                                                 |
| `#foo::\none` / `#foo(a=1)::\none`          | Rust `xprompt_argument_spans` (TUI + LSP) | `:` delimiter plus `arg_value` `:`, with no span over the body. This is the screenshot's highlighting                                                                                                 |
| `%clan:research::\nSummary\n%model:opus\nX` | Python directive extraction               | `clan_summary=None`, and the cleaned prompt is `::\nSummary line\n\nX`, so the `::` leaks into the agent prompt                                                                                       |

The `":: "` literal shows up in these places:

- **sase (Python)**
  - `src/sase/xprompt/_parsing_shorthand.py`: `DOUBLE_COLON_SHORTHAND_PATTERN`,
    `_preprocess_paren_shorthand`, and `_NEXT_DIRECTIVE_PATTERN`. The boundary lookahead
    `(?:\(|::? )` also requires the space.
  - `src/sase/xprompt/processor.py`: `_consume_trailing_shorthand_text`.
  - `src/sase/xprompt/_parsing_references.py`: `_reference_arg_kind_from_match`,
    `_reference_span` (both the bare branch and the paren branch), and
    `XPromptReference.parse_arguments`. `parse_arguments` slices `argument_source[3:]`,
    which assumes a 3-character `":: "` delimiter.
  - `src/sase/xprompt/_parsing_args.py`: `parse_workflow_reference`, in both the paren
    branch (`rest.startswith(":: ")`) and the colon branch (`rest.startswith(": ")`).
  - `src/sase/xprompt/_directive_shorthand.py`: `%clan` rewrite
    (`prompt.startswith(":: ", suffix_start)`, `text_start = suffix_start + 3`).
  - `src/sase/xprompt/_directive_edit_core.py`: `%clan` shorthand removal
    (`prompt.startswith(":: ", match_end)`, `match_end + 3`).
- **sase-core (Rust)**
  - `crates/sase_core/src/editor/xprompt_args.rs`: `parse_parenthesized`
    (`after.starts_with(":: ")`), `parse_colon` (`after_colon.starts_with(": ")`), and
    `starts_with_xprompt_directive`, the boundary check.
  - `crates/sase_core/src/editor/argument_spans.rs` (~line 350): the post-`)` `::`
    delimiter span check.
  - `crates/sase_core/src/agent_launch/identity.rs` (~line 441): the `%clan...::`
    summary shorthand in typed launch planning.

## Contract (the new rule)

A **double-colon text delimiter** is `::` that is not part of a longer colon run. The
same rule applies wherever the shorthand is accepted: `#name::`, `#!name::`, `#name!!::`
/ `#name??::`, `#name(args)::`, and `%clan...::` / `%c...::`. The delimiter must be
immediately followed by one of:

1. **A space plus text on the same line.** The payload starts after exactly one space.
   This case is unchanged.
2. **Zero or more ASCII spaces/tabs, then a line break (`\n` or `\r\n`).** The payload
   starts at the beginning of the next line. The delimiter, the trailing horizontal
   whitespace, and the line break are not part of the payload.
3. **Zero or more ASCII spaces/tabs, then end of input.** The delimiter is recognized
   and the payload is empty. This matches today's `#name:: ` at EOF, so empty-payload
   handling stays exactly as it is (the processor still leaves an empty-payload
   reference unconsumed, the editor still marks the call open, and the `%clan`
   empty-summary diagnostics are unchanged).

The payload's end rules don't change. An xprompt payload runs to the next line that
starts with an xprompt reference followed by `(`, `: `, or `:: `, and that boundary set
now also includes **`#name::` followed by optional spaces/tabs and end of line/input**.
A `%clan` payload runs to the next top-level `%`/`#` line item, or to EOF.

There is one small intentional behavior change: `#name:: \nbody` (a space, then a line
break) now binds `body` instead of `\nbody`.

Explicitly **out of scope / unchanged**:

- Single-colon `#name:` at end of line stays as it is. It would collide with ordinary
  prose such as "see #note:".
- Directives other than `%clan` keep their current behavior. `%if::` / `%proc::` already
  own their fence through the separate code-directive path. Non-allowlisted directives
  such as `%id:worker::` stay literal.
- The TUI skeleton insertion in `_xprompt_arg_assist_skeletons.py` still inserts `:: `.
- The existing `(`-pairing edit (`plan_argument_double_colon_to_parentheses_edit`)
  already accepts `#foo::<cursor>`.
- sase-nvim has no syntax regex for `::`; it gets the fix through the LSP.

## Implementation

Change both repositories in one turn. Open the linked Rust repo with
`sase repo open sase-core` and edit only at the path it prints. Read its `AGENTS.md`
before editing. One declaration covers both repos, and the host commits `sase-core`
first and writes the new SHA into `sase-core-revision.txt` automatically
(`docs/rust_backend.md`). Do **not** hand-edit the pin.

### 1. Rust: `sase-core`

1. `crates/sase_core/src/editor/xprompt_args.rs`
   - Add one private helper that implements the contract, for example
     `double_colon_payload_start(text, colon_idx) -> Option<usize>`. It returns the
     payload start offset (cases 1–3), or `None` when the delimiter does not apply.
   - `parse_parenthesized`: replace `after.starts_with(":: ")` / `after_close + 3` with
     the helper. Check it **before** the `": "` and bare `):value` branches.
   - `parse_colon`: replace `after_colon.starts_with(": ")` / `colon_idx + 3` with the
     helper so that `#foo::\nbody` becomes `XpromptArgSyntax::DoubleColonText`.
     `value_start >= value_end` still sets `is_open`. Check `#foo:\nbar` (single colon)
     afterwards and make sure it parses exactly as it does today.
   - `starts_with_xprompt_directive`: also treat `#name::` + `[ \t]*` + (`\n` | `\r\n` |
     end) as a boundary.
2. `crates/sase_core/src/editor/argument_spans.rs`: the post-`)` check (~line 350) must
   use the same helper, so that `#foo(a=1)::\nbody` emits a 2-byte `::` `ArgDelimiter`
   span. Confirm that the bare `DoubleColonText` arm
   (`push_colon_delimiter(..., double_colon=true, ...)`) gives a 2-byte `::` delimiter
   plus an `arg_value` span over the next-line payload, and that no span covers the
   `\n`.
   - If the helper is needed in both files, make it `pub(crate)` in `xprompt_args.rs`.
     Do not duplicate it.
3. `crates/sase_core/src/agent_launch/identity.rs` (~line 441): make the `%clan` summary
   shorthand use the same contract. Compute `text_start` from the rule instead of
   `directive.end + 3`. `clan_double_colon_text_end` / `is_prompt_item_start` already
   accept `:` after a name, so verify that a following `%`/`#name::` line still acts as
   a boundary. A small agent_launch-local helper is fine if the editor helper is not
   reachable. Keep both implementations byte-identical in behavior.
4. Tests, placed next to the existing ones (`xprompt_args.rs` / `argument_spans.rs`
   `mod tests`, the clan-shorthand tests under `agent_launch/tests/`, and an
   `argument_syntax_edit.rs` case `#foo::<cursor>\nbody` → `#foo()::\nbody`):
   - `#foo::\none\ntwo` → DoubleColonText, payload `one\ntwo`
   - `#foo(a=1)::\none` → payload appended after `a=1`; no `:` value
   - `#foo::  \none`, `#foo::\r\none`, `#foo::\t\none` → payload `one`
   - `#foo::` at EOF → DoubleColonText, open, no value
   - `#a:: x\n#b::\ny` → two calls; `#a`'s payload is `x`
   - `#foo:\nbar` → unchanged from today
   - `%clan:research::\nSummary\n%model:opus\nDo work` → summary `Summary`, region ends
     before `%model`
5. Verify with `sase tool run check` in the sase-core checkout. It is the guarded
   `just check` and takes about 5 minutes, so give it a timeout of 10 minutes or more.

### 2. Python: `sase`

1. Add **one** shared helper to `src/sase/xprompt/_parsing_args.py`, for example
   `double_colon_text_start(text: str, colon_idx: int) -> int | None`, with the same
   semantics as the Rust helper. It belongs in this module, the lowest-level one,
   because `_parsing_shorthand` already imports from `_parsing_args`, and putting the
   helper here avoids an import cycle. Re-export it from `_parsing.py` only if another
   package needs it.
2. `_parsing_shorthand.py`
   - `DOUBLE_COLON_SHORTHAND_PATTERN`: match `::` followed by ` ` or by
     `[ \t]*(?:\r?\n|\Z)`. In pass 2, compute `text_start` with the helper instead of
     `match.end()`.
   - `_preprocess_paren_shorthand`: replace `after_paren.startswith(":: ")` /
     `paren_close + 4` with the helper.
   - `_NEXT_DIRECTIVE_PATTERN`: extend the lookahead alternation with
     `::[ \t]*(?:\r?\n|\Z)`. Keep `#name(`, `#name: `, and `#name:: ` as they are.
3. `processor.py::_consume_trailing_shorthand_text`: use the helper for the double-colon
   case. Keep the single-colon `": "` path and the empty-payload early return unchanged.
4. `_parsing_references.py`
   - `_reference_span`: in the paren branch, check double-colon with the helper
     **before** the `": "` and bare `after_paren.startswith(":")` branches. That bare
     branch is what currently eats the second `:`. Use the helper in the bare branch
     too.
   - `_reference_arg_kind_from_match`: classify via the helper, or from
     `shorthand_text_start`, instead of `startswith(":: ")`.
   - `XPromptReference.parse_arguments`: stop slicing `argument_source[3:]`. Slice the
     payload from `shorthand_text_start` (the argument source begins at
     `end - len(argument_source)`). Leave the `COLON_SHORTHAND` `[2:]` path alone.
5. `_parsing_args.py::parse_workflow_reference`: in the paren branch, use the helper on
   `rest` before the `": "` and bare `":value"` checks. In the colon branch, recognize
   `workflow::` followed by EOL/EOF via the helper, so that `foo::\none` →
   `('foo', ['one'], {})`.
6. `%clan`: `_directive_shorthand.py` and `_directive_edit_core.py` must accept the
   end-of-line delimiter through the same helper, with `text_start` coming from the
   helper. `_NEXT_PROMPT_ITEM_PATTERN` already treats `:` after a name as a boundary, so
   only add a test for that.
7. Afterwards, `grep -rn '":: "' src/sase/xprompt` should show no recognizer left
   outside the helper. The TUI skeleton string in `_xprompt_arg_assist_skeletons.py` is
   intentionally excluded.
8. No code change is expected in the TUI consumers (`_prompt_preview_target.py`,
   `_xprompt_arg_assist_detection.py`, `xprompt_inspect.py`, `highlight.py`). They read
   `XPromptReference.end` / `shorthand_text_start` and the Rust spans. Cover them with
   tests anyway.

### 3. Python tests

Add cases next to the existing double-colon tests:

- `tests/test_xprompt_references.py`: `#foo::\none\ntwo` (DOUBLE_COLON_SHORTHAND,
  `shorthand_text_start` at `o`, `parse_arguments() == (["one\ntwo"], {})`).
  `#foo(a=1)::\none` (payload `one` appended, no `':'` positional). `#foo::  \none`,
  `#foo::\r\none`, `#foo::` at EOF. The `#a:: x\n#b::\ny` boundary yields two refs.
  `#foo:\nbar` is unchanged.
- `tests/test_xprompt_parsing.py`: `parse_workflow_reference("foo::\none")` and the
  paren variant; `preprocess_shorthand_syntax("#foo::\none", {"foo"})`; the boundary
  case in `find_double_colon_text_end`.
- `tests/test_xprompt_processor_args.py`: `#rs::\nhello\nworld` and
  `#rs(g=yes)::\nhello` bind the payload, and the expansion contains no `::`.
- `tests/test_xprompt_swarm_expansion.py`: the screenshot shape
  `#three(x=...)::\nlogin flow` (or `#research_swarm::\nfind foo (bar)`) expands
  identically to its `:: ` same-line form.
- `tests/test_directives_agent_session.py` / `tests/test_directive_edit.py`:
  `%clan:research::\nSummary\n%model:opus\nDo work` → summary `Summary`, and the cleaned
  prompt has no `::`. Also cover the parenthesized `%clan(research, tribe=study)::\n...`
  form.
- TUI: `tests/xprompt/test_highlight.py` (or
  `tests/ace/tui/widgets/test_prompt_xprompt_highlight.py`): `#foo(a=1)::\nbody`
  highlights `::` as an argument delimiter and `body` as an argument value. Arg-assist
  detection returns `None` with the cursor inside the next-line payload, and the preview
  target skips the payload. These tests need the new Rust core. `just test` /
  `just check` rebuilds `sase_core_rs` from the linked checkout automatically when its
  source changed.

### 4. Docs

- `docs/xprompt.md` → "Double-Colon Shorthand": state that `::` may end its line, in
  which case the payload starts on the next line. Add an example in the shape of the
  user's prompt (`#template(style=formal)::` then body lines), and add `#name::` at end
  of line to the boundary list.
- `docs/xprompt.md` (`%clan` summary paragraph, ~line 2233) and `docs/agent_sessions.md`
  (~line 103) currently say "The `::` form requires a following space". Change them to
  "a following space or end of line".
- Leave the editor and ace docs as they are unless a sentence there becomes false.

## Verification

1. In `sase-core`, run `sase tool run check` and confirm it is green. Then, in sase,
   follow the repo's lint/test memory (`sase memory read lint_and_test.md`) and run its
   `just check` gate through `sase tool run`.
2. Spot-check through the real binding from the sase checkout:
   `xprompt_argument_spans("#foo(a=1)::\none\ntwo")` should return `::` as one
   `arg_delimiter` and an `arg_value` covering `one\ntwo`.
3. Rerun the diagnosis probes from the table above. Each must produce the fixed result,
   and every pre-existing `:: ` same-line test must stay green without edits.
