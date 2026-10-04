---
tier: tale
title: Reopen a macro's closed argument list when ( is typed after it
goal:
  "In the prompt input widget and over LSP on-type formatting, typing `(` right after a
  macro's closed argument list (`#foo(bar=1)|` or `#foo(bar=1):: |text`) adds a comma
  before the `)`, puts the caret after that comma, and (in the TUI) opens the macro
  argument completion menu."
size: medium
proposed_by: bbugyi200.apollo.51
create_time: 2026-10-04 10:00:27
status: wip
---

# Plan: Reopen a macro's closed argument list when `(` is typed after it

## Goal

Typing `(` right after a macro reference that already has a closed parenthesized
argument list should reopen that list instead of inserting a literal `(`. The prompt
input widget and the LSP on-type formatting path should both:

1. add a comma after the list's last argument,
2. put the caret right before the list's closing `)`, and
3. (TUI) open the macro argument completion menu at the new caret.

This continues the existing `(` normalizations: colon removal (`#foo:` → `#foo()`),
double-colon delimiter relocation (`#foo:: body` → `#foo():: body`), and the
completion-owned spacer (`#optional ` → `#optional()`).

### Behavior spec

The rule applies only in INSERT mode with an empty selection. Before the `(` is
inserted, the caret must sit immediately after one of these:

- the `)` that closes a macro reference's parenthesized argument list (`#foo(bar=1)|`),
  or
- that `)` followed by a `::` text delimiter and zero or more ASCII spaces
  (`#foo(bar=1)::|`, `#foo(bar=1):: |Some text`).

When it applies, the `(` is never inserted. Instead:

- If the list content, ignoring trailing ASCII whitespace, is non-empty and does not
  already end in `,`, insert `,` right after the last non-whitespace argument character.
- If the content is empty, whitespace-only, or already ends in `,`, change no text and
  only move the caret.
- In both cases the caret lands immediately before the closing `)`. The `)`, the `::`
  delimiter, its spaces, and any suffix text are preserved byte-for-byte.
- TUI: afterwards, run the same post-edit path the colon and double-colon conversions
  use (`_open_auto_reference_completion_after_change("(")`). With the `auto_macro_menu`
  setting on, this opens the `macro_arg_name` menu (for example, the remaining `baz=` /
  `text=` rows). Turning that menu off does not turn off the edit.
- The edit is a single keyboard edit, so one undo restores the previous text.

| Before (`\|` = caret)        | After pressing `(`            |
| ---------------------------- | ----------------------------- |
| `#foo(bar=1)\|`              | `#foo(bar=1,\|)`              |
| `#foo(bar=1):: \|Some text.` | `#foo(bar=1,\|):: Some text.` |
| `#foo(bar=1)::\|`            | `#foo(bar=1,\|)::`            |
| `#foo(bar=1)::   \|body`     | `#foo(bar=1,\|)::   body`     |
| `#foo()\|`                   | `#foo(\|)`                    |
| `#foo(bar=1,)\|`             | `#foo(bar=1,\|)`              |
| `#foo(bar=1, )\|`            | `#foo(bar=1, \|)`             |
| `#foo(bar=1 )\|`             | `#foo(bar=1, \|)`             |
| `#foo(\n  bar=1\n)\|`        | `#foo(\n  bar=1,\n\|)`        |
| `#outer(#inner(a=1)\|)`      | `#outer(#inner(a=1,\|))`      |
| `#foo(a=")", b=[[x)y]])\|`   | `#foo(a=")", b=[[x)y]],\|)`   |

**Eligible references.** Exactly the `#` macro reference forms that the core
`macro_ref_re` in `crates/sase_core/src/editor/macro_args.rs` matches: `#name`,
`#!name`, `#ns/name`, `#ns__name`, `#name!!`, `#name??`. The marker must be at line
start or after whitespace or one of `([{"'`. Paren matching uses the existing
`find_matching_paren_for_args`, which respects quotes and `[[...]]` text blocks.

**Not eligible.** These keep today's behavior (the ordinary `(` pairing/insert path):

- `%` directives of every kind: `%q(capacity=1)`, `%wait(...)`, `%proc(a)::`,
  alternation `%(a,b)` / `%alt(...)`.
- Plain prose parens: `(note)`, `foo(bar)`.
- Escaped or mid-word references, and URLs: `\#foo(a)`, `word#foo(a)`,
  `https://x.test/#foo(a)`.
- A gap after `)` that is not `::` plus ASCII spaces: `#foo(a) |`, `#foo(a):|`,
  `#foo(a): |`, tabs or NBSP after `::`, and `#foo(a)::|:` / `#foo(a):::`.
- A caret that is not adjacent to the list: `#foo(a):: body|`, `#foo(a)x|`.
- A caret inside the list or after an unclosed list: `#foo(a|)`, `#foo(a|`.
- A list in a literal or definition region (inline code, fenced code, disabled macro
  regions, prompt frontmatter, Jinja tags), using the same
  `excluded_literal_and_definition_ranges` check as the existing planners.

## Design

Per the Rust core backend boundary, the eligibility and edit logic is shared backend
behavior. It belongs in `sase_core` in the linked `sase-core` repo. The TUI and the
language server stay thin adapters, the same way
`plan_argument_double_colon_to_parentheses_edit` is used today.

Open the linked repo with `sase repo open sase-core -r "<reason>"` and work only in the
printed path. Read its `AGENTS.md` first. In particular: no `macro_rules!`; import core
items by module path; do NOT add names to the root `pub use` list in
`crates/sase_core/src/lib.rs` or new `core_*` aliases to
`crates/sase_core_py/src/prelude.rs`; never edit versions or `CHANGELOG.md`.

### 1. sase-core — paren-matching helper (`crates/sase_core/src/editor/macro_args.rs`)

Add a crate-private helper next to `macro_argument_open_colon_at`:

```rust
/// Return the `(` index when `close_idx` closes a `#` macro reference's
/// parenthesized argument list.
pub(crate) fn macro_argument_list_open_paren_for_close(
    text: &str,
    close_idx: usize,
) -> Option<usize>
```

It requires `text[close_idx] == b')'`. It then iterates `macro_ref_re().captures_iter`
and computes each match's suffix start the same way `macro_argument_open_colon_at` does
(after the `hitl` group if present, else after `name`). It returns `Some(open_idx)` for
the first match where `text[open_idx] == b'('`, `open_idx < close_idx`, and
`find_matching_paren_for_args(text, open_idx) == Some(close_idx)`. The regex already
rejects escaped, mid-word, and URL markers, and nested references such as
`#outer(#inner(a))` are matched separately because the inner one is preceded by `(`. Add
small unit tests in that file's `tests` module: plain, hitl, namespaced, nested, quoted
`)` and `[[...)...]]` content, a mismatched close, and a directive (`%q(a)`) → `None`.

### 2. sase-core — the planner (`crates/sase_core/src/editor/argument_syntax_edit.rs`)

Add:

```rust
/// Plan reopening a macro's closed parenthesized argument list before `(`.
///
/// The caller supplies the pre-insertion document plus the caret position
/// where `(` is about to be typed. The returned edit's range ends at the
/// list's closing `)`; frontends place the caret at
/// `range.start + new_text.len()` (immediately before that `)`), and the
/// typed `(` is never inserted.
pub fn plan_argument_list_continuation_edit(
    document: &DocumentSnapshot,
    position: EditorPosition,
) -> Option<EditorTextEdit>
```

Algorithm (byte offsets, pre-insertion text):

1. `cursor = document.position_to_byte_offset(position)?`.
2. Walk back over ASCII spaces (`b' '` only) to `delimiter_end`.
3. Delimiter forms:
   - If `text[delimiter_end-2..delimiter_end] == b"::"`, check that
     `text.get(delimiter_end) != Some(&b':')`, which mirrors the double-colon planner's
     `:::` guard. Then `close_idx = delimiter_end - 3`.
   - Else, if no spaces were skipped (`delimiter_end == cursor`), use
     `close_idx = cursor - 1`.
   - Otherwise return `None`.
4. Require `text[close_idx] == b')'` and
   `open_idx = macro_argument_list_open_paren_for_close(text, close_idx)?`.
5. Reject when `open_idx` or `close_idx` falls in
   `excluded_literal_and_definition_ranges(text)` (compute it once).
6. `content_end` = `close_idx` with trailing ASCII whitespace trimmed, but not below
   `open_idx + 1`.
   - If `content_end == open_idx + 1` or `text[content_end - 1] == b','`, return
     `EditorTextEdit { range: [close_idx, close_idx], new_text: "" }`. This is a
     caret-only edit.
   - Otherwise return
     `EditorTextEdit { range: [content_end, close_idx], new_text: format!(",{}", &text[content_end..close_idx]) }`.
     The trailing whitespace is re-emitted after the comma, so the caret target is
     always the `)`.

Export it from `crates/sase_core/src/editor/mod.rs`'s existing
`pub use argument_syntax_edit::{...}` list (module facade). Do not add it to `lib.rs`.

Unit tests in the same file's `tests` module: add an `applied_continuation` helper
modeled on `applied_double` that returns `(new_text, caret_byte)`. Cover every row of
the behavior table above (including UTF-16/astral and multi-line positions, as in
`double_colon_handles_multiline_and_utf16_positions`), every "Not eligible" bullet, and
invalid UTF-16 positions. Keep the existing assertion that
`plan_argument_double_colon_to_parentheses_edit` still returns `None` for
`#foo(a):: <cursor>`; the two planners must stay mutually exclusive.

### 3. sase-core — Python binding (`crates/sase_core_py/src/editor_completion/mod.rs`)

Add `#[pyfunction] #[pyo3(name = "argument_list_continuation_edit")]`
`py_argument_list_continuation_edit(py, text, position)`. Mirror
`py_argument_double_colon_to_parentheses_edit` exactly, but call
`sase_core::editor::plan_argument_list_continuation_edit` by module path (no new prelude
alias). Register it in the module's function list next to the double-colon binding. Add
a binding test in `crates/sase_core_py/src/editor_completion/tests/snippets.rs`, modeled
on `argument_double_colon_to_parentheses_binding_returns_plain_edit_or_none`. It should
check the comma edit dict, a caret-only edit dict for `#foo()`, `None` for `%q(a)`, a
UTF-16 range, and the malformed-position error.

### 4. sase-core — LSP on-type formatting (`crates/sase_xprompt_lsp/src/server/actions.rs`)

In `on_type_formatting_for_text_with_recent`, after the double-colon branch and before
the single-colon fallback, call the new planner on `pre_insert_document` /
`pre_insert_position`. Import it via `sase_core::editor::...`, not a new root alias. On
`Some(edit)`:

- Convert the range to pre-insertion bytes: `start = range.start` and
  `close = range.end`. Check `pre_insert_text[close] == b')'`.
- Set `delimiter = pre_insert_text[close + 1 .. opener_idx]`. It must be empty or `::`
  followed by ASCII spaces; return `None` on anything else, as a defensive check.
- Compute `closer_end` with the same `recent_paren_insertion` rule the double-colon
  branch uses. It consumes the editor-inserted `)` at `after_opener_idx` only when it
  was provably auto-paired.
- Return two LSP edits against the post-insertion document. Offsets below `opener_idx`
  are identical in both documents.
  1. range `[start, after_opener_idx)` → `edit.new_text`. This replaces the trailing
     whitespace, the old `)`, the delimiter, and the typed `(`, and it ends exactly at
     the caret.
  2. range `[after_opener_idx, closer_end)` → `format!("){delimiter}")`. This starts
     exactly at the caret.

  Splitting the edits at the caret boundary follows the double-colon branch's
  convention. Clients that keep the caret at edit boundaries (for example Neovim's
  `nvim_buf_set_text` cursor fix-up) end with the caret after the comma and before `)`.
  Put that rationale in a short comment.

The `spacer_on_type` path in `server/mod.rs` keeps running first. It is mutually
exclusive with this branch.

LSP tests:

- `crates/sase_xprompt_lsp/tests/jsonrpc_stdio_on_type_formatting.rs`: remove
  `(17, "#foo(args):: (", 14)` from the `Value::Null` loop in
  `stdio_jsonrpc_on_type_formatting_moves_double_colon_delimiter`, because it now
  continues the list. Add a new
  `stdio_jsonrpc_on_type_formatting_continues_macro_argument_list` test using the file's
  existing `did_open` / `did_change` / `request_on_type` / `apply_text_edits` helpers.
  - Assert exact edit JSON for one unpaired case, to pin the caret-boundary structure.
  - Check the unpaired forms `#foo(bar=1)(` → `#foo(bar=1,)` and
    `#foo(bar=1):: (Some text` → `#foo(bar=1,):: Some text`.
  - Check the editor-paired forms `#foo(bar=1)()` and `#foo(bar=1):: ()Some text`,
    reached through a `did_change` sequence so recent-paren detection fires. They must
    produce the same results.
  - Check that a pre-existing `)` that was not editor-inserted is not consumed.
  - Check the empty list `#foo()(` → `#foo()`, the trigger-at-cursor variant, and a
    unicode/multi-line case.
  - Check that `%q(a)(`, `foo(bar)(`, and `#foo(a) (` return `Value::Null`.
- Add a server unit test (new `crates/sase_xprompt_lsp/src/server/tests/` module
  registered in `tests/mod.rs`, using `bridge_with_catalog_entries` as `spacer.rs`
  does). Give it a catalog entry `#foo` with inputs `bar` and `baz`. Run on-type on
  `#foo(bar=1):: (Some text`, `did_change` to the formatted text, then request
  completion at the formatted caret, and assert that `baz=` is offered. This guards the
  claim that argument completion works at the new caret. No completion code change is
  expected.

### 5. sase — TUI adapter (`src/sase/ace/tui/widgets/_argument_syntax_editing.py`)

Add `plan_argument_list_continuation_edit(text, cursor_location) -> TextEdit | None`,
modeled on `plan_argument_double_colon_to_parentheses_edit`:

- Call `_editor_position` and then
  `require_rust_binding("argument_list_continuation_edit")`.
- Convert the range with
  `editor_range_to_offsets(text, payload["range"], allow_empty=True)`.
- Validate that `text[end] == ")"` and that `text[start:end]` is ASCII whitespace only.
  `new_text` must be either `""` with `start == end`, or `"," + text[start:end]`. Return
  `None` otherwise.
- Return `TextEdit(start=start, end=end, text=new_text, cursor=start + len(new_text))`.

Add it to `__all__`.

### 6. sase — TUI wiring (`src/sase/ace/tui/widgets/_prompt_text_area_key_pairing.py`)

In `_try_prompt_text_pair_edit`'s `char == "("` branch, after the double-colon and
single-colon conversions and before `plan_pair_close_skip` / `plan_pair_insert`, try the
new planner. If it returns a plan, call
`self._apply_planned_text_edit(plan, remap_dot_capture=True)`, then
`self._open_auto_reference_completion_after_change(char)`, then `return True`, mirroring
the double-colon branch. Update the method docstring's dispatch-order description. No
change to `_prompt_text_area_key_handling.py` should be needed: the active-arg-hint `(`
interception only fires when the caret equals an accepted hint's `reference_end`, and no
hint is detected at `#foo(bar=1)|` (verified while planning). Add a regression test
anyway (below).

### 7. sase — docs

- `docs/macros.md`: after the double-colon `(` paragraph in "Reference Syntax", add a
  paragraph describing the continuation rule, with `#review(path=a)` →
  `#review(path=a,|)` and `#review(path=a):: body` → `#review(path=a,|):: body`
  examples. Cover the caret-only cases, the argument menu (`auto_macro_menu`), the
  eligibility limits (macros only, not directives), and the LSP note from section 4
  (on-type edit only; the menu comes from the client's next completion request).
- `docs/ace.md`: extend the "When `(` is typed immediately after a macro or supported
  directive argument delimiter…" paragraph with the continuation case and the
  auto-opened argument menu.
- `docs/editor.md`: extend the on-type formatting paragraph under the LSP feature table.

No keymap or `default_config.yml` change is needed: `(` is not a configurable binding.

### 8. sase — tests

- New `tests/ace/tui/widgets/test_prompt_argument_list_continuation.py`:
  - Parametrized pilot tests using a small `App` with a `PromptTextArea`, like
    `PairEditTestApp` in `test_prompt_pair_editing.py`. Load the text, set the caret,
    press `(`, and assert the text and caret for every row of the behavior table,
    including the user's example
    `#foo(bar=1):: |Some text for the first positional input.` →
    `#foo(bar=1,|):: Some text for the first positional input.`
  - Ineligible sources from the spec keep the ordinary pair result.
  - Selected text receives a literal replacement.
  - The edit is one undo checkpoint (`PromptPage` + `escape`, `u`).
  - Typing continues inside the list: pressing `(`, `b`, `a`, `z`, `=`, `2` yields
    `#foo(bar=1,baz=2):: …`.
  - Menu tests using `CompletionTestApp` from `_completion_helpers.py` and seeded
    `MacroAssistEntry` rows (the `_seed_entries` pattern in
    `test_macro_completion_spacer.py`). After `(`, `ta._file_completion_active is True`,
    `ta._completion_kind == "macro_arg_name"`, and the remaining rows appear (for
    example `baz=`). Repeat with `#foo(bar=1):: text`. With
    `PromptCompletionSettings(auto_xprompt_menu=False)` patched in, the edit still
    happens and no menu opens.
  - A regression test showing that an accepted-completion arg hint at a different
    reference does not intercept the `(`.
- `tests/ace/tui/widgets/test_prompt_pair_editing.py`: remove `"#foo(args):: "` from
  `test_typing_paren_after_ineligible_double_colon_keeps_literal_pair`, because it is
  now covered by the continuation tests.
- Optional unit tests of the Python adapter's validation
  (`plan_argument_list_continuation_edit`), in the new test file.

## Verification

1. In the opened `sase-core` checkout: iterate with
   `just test -p sase_core argument_syntax_edit`, `just test -p sase_core macro_args`,
   and `just test -p sase_xprompt_lsp`. Finish with `sase tool run check` (≈5 min; give
   it a ≥10 min tool timeout) and get it green.
2. In the sase workspace: run `just rust-install`, which builds `sase_core_rs` from the
   linked `sase-core` checkout so the new binding is importable. Then run the new and
   touched pytest files directly, and finish with `sase tool run check`.
3. Do not hand-edit `sase-core-revision.txt`. When the final declaration commits both
   repos, the host commits `sase-core` first and writes the pushed SHA into the pin.

## Out of scope / known limitations

- **LSP argument menu timing.** The server's on-type edit produces the same text and
  caret, but an LSP server cannot itself open a completion menu. A client's
  `(`-triggered completion request races the formatting edit, so external editors
  typically show the argument rows on the next completion request (typing the first
  argument character or invoking completion manually). Building a pre-format
  "transition" completion view, like the spacer feature has, is deliberately not
  attempted. Its rewrite must move `)` and the delimiter across the caret, which
  collides with the completion item's own insertion point. Document this limitation in
  `docs/macros.md` / `docs/editor.md`.
- **Directive argument lists** (`%q(...)`, `%wait(...)`, `%proc(...)::`, alternation)
  are excluded on purpose. They have heterogeneous paren semantics, and the request was
  about macro inputs.
- **Single-colon tails after a list** (`#foo(a): text`, `#foo(a):x`) are not handled.
- **Pre-existing:** the arg menu still offers the first positional input's `name=` when
  a `::` text tail already supplies it. This change leaves that alone.
