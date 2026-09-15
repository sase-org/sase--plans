---
tier: tale
title: Add double-colon parentheses insertion to prompt editing
goal:
  Typing an opening parenthesis after an invocation's double colon and optional spaces
  inserts an argument pair before the delimiter and places the caret inside it in the
  prompt widget and external editors.
size: medium
proposed_by: bbugyi200.athena.0ll
create_time: 2026-09-15 18:21:22
status: wip
---

# Add parentheses before double-colon prompt arguments when typing `(`

## Goal and scope

When the user types `(` immediately after an invocation's `::` and zero or more ASCII
spaces, insert `()` before the first colon, preserve the double colon and spaces, and
place the caret between the parentheses. Apply the same behavior to the prompt input
widget and external editors through the xprompt LSP. This lets the user immediately
enter an argument such as `foo=bar` before the text block.

This is a `tale` of `medium` implementation size: one coding agent can complete the
shared Rust planner, Python integration, LSP adaptation, and editor tests together.
Multiple repositories are involved, but their changes form one bounded feature and do
not need separate implementation phases.

Author and approve this plan before changing implementation files. During
implementation, open `sase-core` and `sase-nvim` with `/sase_repo` and use only the
returned checkout paths. All paths below are relative to the named repository; do not
assume another agent's workspace location or a sibling checkout path.

## Behavior contract

In these examples `<caret>` marks the caret and is not document text. The left column is
the document immediately before the `(` keystroke.

| Before                 | After typing `(`         |
| ---------------------- | ------------------------ |
| `#foo::<caret>`        | `#foo(<caret>)::`        |
| `#foo:: <caret>`       | `#foo(<caret>):: `       |
| `#foo::   <caret>`     | `#foo(<caret>)::   `     |
| `#foo:: <caret>body`   | `#foo(<caret>):: body`   |
| `#foo:: <caret>  body` | `#foo(<caret>)::   body` |
| `#!ns/foo:: <caret>`   | `#!ns/foo(<caret>):: `   |
| `%clan:: <caret>`      | `%clan(<caret>):: `      |

1. Recognize an argument-opening `::` immediately following a valid invocation name and
   any already-supported workflow/HITL suffix. Reuse the shared xprompt name, namespace,
   escape, and left-boundary recognition. Catalog discovery or name existence is
   unnecessary, as in the existing single-colon conversion.
2. Recognize known directives and aliases that support parenthesized arguments,
   including `%proc`; reject unknown directives and directives without that form, such
   as `%if` and `%xprompts_enabled`. Keep the existing single-colon directive
   eligibility unchanged. This is an editing convenience, not an expansion of the launch
   grammar or a change to feature-flag behavior.
3. Between the second colon and the caret, accept only zero or more U+0020 spaces. Do
   not scan across tabs, newlines, nonbreaking spaces, or existing body text. Reject a
   third colon and a caret between the two colons.
4. Preserve the exact authored spacing and document suffix. The special double-colon
   path always inserts the complete `()` pair, including before existing body text. The
   generic bracket-pairing guard must not suppress its closing parenthesis.
5. Do not add a second argument list to `#foo(...)::` or an invocation with an existing
   colon argument. Preserve ordinary insertion behavior in code fences, inline code,
   disabled xprompt regions, frontmatter, and Jinja, using the same exclusion rules as
   the existing conversion.
6. Only the typed `(` event triggers conversion. Widget selection replacement, paste,
   loading text, and other characters keep their existing behavior.
7. After conversion, typing `foo=bar` inserts inside the new pair, `)` skips the closing
   parenthesis, and deleting the empty pair preserves `::` and its spaces. Existing
   single-colon conversion and its pairing rules remain unchanged.

## Findings and design

### Existing implementation

- **sase-core:** `crates/sase_core/src/editor/argument_syntax_edit.rs` implements
  `plan_argument_colon_to_parentheses_edit`. It takes the pre-insertion document and
  UTF-16 caret position, returns an `EditorTextEdit` deleting one colon, and explicitly
  rejects adjacent colons. Recognition comes from `editor/xprompt_args.rs`,
  `editor/directive.rs`, and `editor/exclusion.rs`.
- **sase-core:** `crates/sase_core_py/src/lib.rs` exposes
  `argument_colon_to_parentheses_edit` as a plain dictionary or `None`.
- **sase:** `_argument_syntax_editing.py` under `src/sase/ace/tui/widgets/` validates
  that payload as a single-colon deletion. `_prompt_text_area_key_pairing.py` combines
  the deletion with normal pairing, then applies one keyboard edit and clears completion
  state. Its edit application also remaps the Vim dot-repeat insertion capture.
- **sase-core:** `crates/sase_xprompt_lsp/src/server.rs` advertises `(` for
  `textDocument/onTypeFormatting`. It reconstructs a pre-insertion snapshot by removing
  the typed opener, accepting request positions either on or immediately after it, and
  invokes the shared planner. Document eligibility is checked by the request handler.
- **sase-nvim:** `lua/sase/lsp.lua` enables native Neovim on-type formatting for the
  SASE client. `tests/lsp_on_type_formatting_smoke.lua` exercises the existing path, but
  its feed helper can leave insert mode before asserting the cursor.

### Shared planner and compatible API

Add a sibling Rust planner, for example
`plan_argument_double_colon_to_parentheses_edit`, rather than changing the meaning of
the existing single-colon deletion API. Return the existing `EditorTextEdit` wire shape:
replace the span from the first colon through the pre-insertion caret with `()` followed
by the original `::` and spaces. The frontend's final caret is one ASCII character after
the edit range's start.

This keeps parsing and eligibility in Rust, requires no general editor wire schema
change, and preserves existing callers. The Python adapter only validates and converts
positions/edits; it must not implement a second invocation parser.

### LSP text edits and cursor placement

On-type formatting returns text edits, without a separate cursor-position result. See
Microsoft's
[on-type formatting protocol API](https://learn.microsoft.com/en-us/dotnet/api/microsoft.visualstudio.languageserver.protocol.methods.textdocumentontypeformatting?view=visualstudiosdk-2022).
Shape the edits around the existing typed opener so the editor keeps the caret attached
to it:

1. Delete the original `::` and spaces immediately before the typed `(`.
2. Immediately after that unchanged `(`, insert `)` plus the saved `::` and spaces,
   reusing an editor-generated closer when one exists.

Both ranges must refer to the same post-insertion document and must not overlap. For
post-insertion `#foo::  (` the zero-based ranges are `[4, 8)` -> empty and `[9, 9)` ->
`)::  `. With an editor-added `)`, the second range is `[9, 10)`. Do all index
arithmetic in Rust byte positions and convert through `DocumentSnapshot` to UTF-16
ranges.

During planning, an in-memory probe through Neovim 0.12.5's native on-type formatting
callback confirmed that these edits leave the actual insert-mode caret at column 5 for
both paired and unpaired input. Subsequent input produced `#foo(foo=bar)::  `. This
probe used a stub response, so integration with the real rebuilt server must still be
tested.

Do not assume every adjacent `)` was generated by an editor pairing plugin. Preserve a
pre-existing body `)` and any enclosing delimiter. Use bounded recent document-change
context to distinguish an inserted `()` from `(` typed before an existing `)` when
necessary; handle both a combined pair insertion and separate opener/closer updates.
Keep this information local to the open document, invalidate it on unrelated changes,
and discard it on close. A missing history must not authorize deleting an existing
suffix character. No custom cursor RPC or Lua invocation parser is needed for the
verified native path.

## Implementation steps

### 1. Implement and expose the Rust edit planner

In **sase-core**:

- Add the double-colon planner and unit tests in
  `crates/sase_core/src/editor/argument_syntax_edit.rs` (split tests into a nearby
  module if required by repository file-size checks).
- Scan backward over ASCII spaces from the supplied caret, require exactly two
  argument-opening colons, and use the existing literal exclusions and invocation
  recognition. Factor directive recognition carefully so the double-colon helper can
  accept parenthesis-capable `%proc` without changing the single-colon helper's
  colon-plus-parentheses requirement.
- Export the planner through `editor/mod.rs` and `crates/sase_core/src/lib.rs`. Add the
  corresponding PyO3 binding and registration in `crates/sase_core_py/src/lib.rs`,
  preserving the old binding contract.
- Add binding tests for the new edit payload, `None` results, UTF-16 positions, and
  malformed position inputs. Use the existing editor wire shape and do not manually
  change Cargo package versions or unrelated dependency pins.

### 2. Integrate the prompt widget

In **sase**:

- Add a thin adapter for the new binding in
  `src/sase/ace/tui/widgets/_argument_syntax_editing.py`. Convert the Rust range to
  Python offsets and set `TextEdit.cursor` to the replacement start plus one. Reject
  malformed or out-of-bounds payloads.
- In `_prompt_text_area_key_pairing.py`, try the double-colon plan on `(` before falling
  back to the existing single-colon and generic pairing paths. Apply the full
  replacement once through `_apply_planned_text_edit` with the appropriate dot-capture
  remapping, then run the existing completion-context refresh.
- Keep selection and input-mode guards. Verify one undo checkpoint restores the exact
  original colons/spaces, redo restores the conversion, and dot-repeat capture does not
  accidentally include or discard the relocated delimiter.
- Add tests alongside `tests/ace/tui/widgets/test_prompt_pair_editing.py`, including
  continuing to type `foo=bar`, completion at the new argument position, close-skip, and
  pair deletion.

### 3. Adapt LSP formatting and verify the external editor

In **sase-core**:

- Extend `on_type_formatting_for_text` and its request adapter to use the new planner
  and emit the two non-overlapping post-insertion edits described above. Keep the
  single-colon response unchanged. Preserve both supported trigger position conventions
  and eligible-document gating.
- Reuse an editor-added closer exactly once while preserving pre-existing suffix text.
  Carry only the recent insertion context needed to establish closer provenance; do not
  add durable state or broaden document synchronization modes.
- Extend `crates/sase_xprompt_lsp/tests/jsonrpc_stdio_on_type_formatting.rs` with
  realistic open/change/format sequences, applying returned edits to assert final text.
  Cover paired/unpaired input, multiple spaces, suffixes, Unicode, multiline positions,
  rejection cases, and repeated/no-op requests. Confirm named-argument completion works
  at the resulting caret.

In **sase-nvim**:

- Extend `tests/lsp_on_type_formatting_smoke.lua` against the actual rebuilt LSP.
  Exercise real insert-mode events with and without the `()<Left>` mapping. Check both
  text and caret before leaving insert mode, and then type `foo=bar` to prove the
  insertion point. Keep the existing single-colon and unattached buffer cases.
- Prefer the existing `lua/sase/lsp.lua` native integration. Production Lua changes
  should only be necessary if the real server/client test reveals an integration issue;
  any such change must remain transport/presentation glue.

### 4. Document and verify

- Add concise examples in **sase** `docs/editor.md`, the prompt input section of
  `docs/ace.md`, and the shorthand section of `docs/xprompt.md`. Explain the ASCII-space
  trigger, preservation of `::`, caret placement, and the full pair inserted by this new
  path. Update **sase-nvim** README's argument-colon section.
- No new CLI options, keymaps, configuration switches, or feature flags are needed.
- Rebuild/install the Rust binding and LSP from the checkout opened for this work, for
  example with `SASE_CORE_DIR` set to that returned path and `just rust-dev-install` in
  **sase**. Ensure Python tests and the editor smoke use these new builds rather than an
  older installed wheel or server binary.
- Run the targeted Python widget tests, then **sase** `just check` as required by
  `lint_and_test.md`. Run **sase-core** `just check` / `./scripts/check.sh`, covering
  formatting, clippy, workspace tests, and PyO3 tests with Python >= 3.12. Do not
  substitute `cargo test -p sase_core` for the required core checks.
- From the opened **sase-nvim** checkout, run its existing headless LSP config test and
  on-type smoke script with `SASE_XPROMPT_LSP_CMD` pointing to the rebuilt binary. Use
  the repository's formatting checks for changed Lua/Markdown.
- Use `/sase_monitor` for long-running checks. If exhaustive verification is required by
  the changed-file selection or landing workflow, run `just check-full` through that
  skill. Submit completion through `/sase_final`; host-owned finalizers handle all
  changed repositories.

## Acceptance and regression matrix

The test suites together must establish:

- Zero, one, and multiple spaces; caret partway through a run of spaces; body text to
  the right; preserved blank lines and following prompt items.
- Inline/start-of-line references, namespaced names, `#!` workflows, supported HITL
  suffixes, known directive aliases, and parenthesis-capable `%proc`.
- Tabs/newlines/nonbreaking spaces between delimiter and caret, triple colons, caret
  between colons, body text before the caret, existing argument lists, escaped/embedded
  markers, URLs/prose, unknown directives, and literal regions do not trigger this
  rewrite.
- Non-BMP characters before the invocation and a later document line produce correct
  UTF-16 LSP ranges and Textual row/column positions.
- Editor pairing on/off yields exactly one argument pair; an existing body or outer `)`
  is preserved; continuing to type enters the new argument list.
- TUI conversion is one undoable keyboard edit, preserves pair operations, and refreshes
  argument completion without subprocess, filesystem, catalog, or network work on the
  typing path.
- Existing single-colon conversion, LSP document eligibility, ordinary selection
  replacement/paste, and no-op requests remain covered and pass.

Completion requires passing the repository checks and the real-editor caret test, with
no implementation changes outside this feature's scope.
