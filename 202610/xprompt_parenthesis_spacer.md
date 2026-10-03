---
tier: tale
title: Continue accepted xprompts into parenthesized arguments
size: medium
goal:
  Make an immediate opening parenthesis consume the space inserted by an accepted
  xprompt completion with inputs and expose its argument completions, including through
  standard LSP features where supported.
proposed_by: bbugyi200.apollo.4h
create_time: 2026-10-03 07:57:25
status: wip
---

# Continue accepted xprompts into parenthesized arguments

## Scope and tier

Implement this as one medium tale across `sase` and its linked `sase-core` repo. The
widget already tracks completion-owned spaces, pairing and argument menus already exist,
and the language server already handles `(` on-type formatting. One implementation agent
can extend those seams and verify their integration. No implementation changes were made
while preparing this plan.

Open `sase-core` with `/sase_repo` and `sase repo open sase-core`, then use the printed
checkout and read its `AGENTS.md`. All core paths below are relative to that repo; all
other paths are relative to `sase`. Do not change editor plugins, dotfiles, keymaps,
completion acceptance keys, or xprompt invocation grammar.

## User-visible contract

With an xprompt whose `topic` input is optional:

1. Accept `#optional` from completion, producing `#optional |` (`|` is the caret).
2. Type `(` once.
3. In the prompt widget, obtain `#optional(|)` and the existing argument menu, including
   `topic=`. Accepting that row must produce `#optional(topic=|)`.

Delete only the exact single ASCII space inserted by completion. Apply the new rule only
when that accepted entry has at least one input, the reference and spacer are intact,
the caret is immediately after the spacer, the selection is empty, and the widget is in
insert mode. The existing one-shot invalidation rule remains: another key consumes
eligibility, even a cursor movement followed by a return to the original position.
Pasted or manually typed lookalike text does not acquire completion ownership.

The usual spacer case is an entry with optional inputs only. Completion already inserts
`:` for one required non-text input, `:: ` at end of line for one required text input,
and `($0)` for multiple required inputs. Preserve those paths and their existing
colon/double-colon conversions. A zero-input xprompt keeps its space when `(` is typed,
with ordinary pairing behavior.

Preserve all suffix text and surrounding snippet placeholders. Use the existing pairing
policy: insert `()` at a safe pairing position, or only `(` where an existing following
token makes pairing unsafe. The caret lands immediately after the opener. The
transformation is one keyboard edit for undo, snippet range remapping, and Vim insert
capture. Existing paired backspace, closer skip, comma/colon spacer rewrites, and
Tab/Shift+Tab spacer removal keep working.

The automatic argument menu respects `auto_xprompt_menu`; disabling that menu does not
disable the space rewrite. Reuse the existing argument candidate selection policy,
including specialized value/agent/path menus where applicable.

## Findings and implementation seams

- `src/sase/ace/tui/widgets/_xprompt_arg_hints.py` records
  `PendingXPromptCompletionSpacer` only after an accepted no-required-input skeleton.
  `_consume_xprompt_completion_spacer` currently replaces its space with `,`, or with
  `:` when `has_optional_inputs` is true.
- `_prompt_text_area_key_handling.py` consumes that pending record at the start of
  `_on_key`, before ordinary pairing, then refreshes completion. Simply adding `(` to
  the punctuation tuple would bypass pairing and cursor handling.
- `_prompt_text_area_key_pairing.py` already composes colon deletion and pair insertion
  into a single `TextEdit`, applies it through `_apply_planned_text_edit`, and opens
  argument completion after the edit.
- `_file_completion_accept_kinds.py`, `_prompt_soft_completion.py`, and
  `_prompt_input_bar_target_actions.py` already record the spacer for manual completion,
  soft completion, and selector insertion. Preserve this shared coverage instead of
  adding special acceptance paths.
- Core `crates/sase_core/src/editor/argument_syntax_edit.rs` owns shared argument edits
  and literal-region exclusions. Python's `_argument_syntax_editing.py` adapts those
  edits through Rust bindings.
- Core `crates/sase_xprompt_lsp/src/server/mod.rs` advertises `(` both as a completion
  trigger and an on-type formatting trigger. `actions.rs` adapts the pre-insertion core
  edit to the document after typing. `documents.rs` and `state.rs` track full-document
  changes and recent inserted parentheses.
- LSP `completion_items.rs::macro_completion_skeleton` also inserts a trailing space for
  no-required-input entries. The server currently has no accepted completion spacer
  record. Neither requesting completion nor resolving an item proves the user accepted
  it.

## Implementation

### 1. Shared core edit planning and binding

Add a small pure editor planner for consuming a completion-owned xprompt space before
`(`. Give it the pre-insertion document, caret position, and a typed record of the
accepted reference, reference/spacer positions, and input eligibility. Frontends
establish acceptance; core validates the exact reference, single space, adjacency,
bounds, input eligibility, and existing excluded literal/definition regions. Reuse
editor position and edit wire types. Return only the spacer-deletion edit, leaving
parentheses and caret placement to the frontend. This is a completion edit, not a parser
change that permits whitespace between an invocation and its argument list.

Use cached acceptance metadata, with no catalog reload, filesystem access, subprocess,
or provider resolution on the typing path. Support existing `#` and `#!` references and
namespaced entries; validate the recorded insertion rather than inventing another name
regex. Reject zero-input and unknown or unowned references, tabs/newlines/nonbreaking
spaces, mismatched positions, and stale text. Existing colon/comma behavior need not be
migrated wholesale.

Export the planner from the editor module and add/register its PyO3 binding alongside
the argument-edit bindings in `crates/sase_core_py/src/editor_completion/mod.rs`. Import
directly from the owning core module rather than introducing root exports or prelude
aliases. Add binding round-trip coverage and a thin adapter in
`src/sase/ace/tui/widgets/_argument_syntax_editing.py`, using the existing UTF-16
conversion helpers and validating returned edit ranges.

### 2. Prompt widget integration

Extend the pending-spacer dispatch with an explicit `(` branch guarded by insert mode,
empty selection, and input eligibility. Keep the pending state one-shot on both success
and failure. Reuse the core deletion planner and compose its range with the existing
`plan_pair_insert` result into one edit, analogous to
`_plan_argument_colon_pair_conversion`. Use literal `(` if pairing is unsafe. Apply
through the keyboard edit path with appropriate dot-capture remapping, then call the
existing automatic completion refresh after the final text and caret are installed. Do
not delete the space and then redispatch the key as two independent edits.

Keep argument name and specialized input completion behavior centralized in the current
completion pipeline. In particular, test a real optional word input so that a correctly
rewritten buffer alone cannot hide a missing menu. Update pending-spacer docstrings and
comments to describe `(` alongside `:`.

### 3. Generic LSP integration

Use standard LSP messages, with no editor-specific commands or key mappings. Attach a
server-owned `CompletionItem.command` to eligible xprompt completion items whose actual
insertion includes a spacer. Register and handle that command through the existing
`workspace/executeCommand` route. Its only effect is to record acceptance; the later
on-type response performs the edit. The standard command runs after completion
insertion, unlike completion-item resolution.
[CompletionItem protocol types](https://docs.rs/lsp-types/latest/lsp_types/struct.CompletionItem.html#structfield.command)
also distinguish commit characters, which accept a currently active suggestion and do
not implement this post-acceptance behavior.

Keep the new state scoped and bounded per open document. Associate each served eligible
completion with its document identity, source generation/version, replacement range,
exact inserted reference/spacer, and input eligibility. Thread the URI/version through
the production completion route as needed; direct text-only test helpers need not
manufacture acceptance. Validate the acceptance command against a served item and
expected document transition. Account for command delivery before or after the
acceptance `didChange`, and for a coalesced acceptance plus `(` or `()` change. Reject
stale commands after unrelated changes or document close/reopen. Do not infer acceptance
from merely seeing `#optional ` in the buffer.

On the next qualifying `(` on-type request, use the shared core planner to return a
deletion of the owned spacer in the current document's UTF-16 coordinates. Preserve the
typed opener, any editor-inserted closer, and every suffix character; this branch does
not insert a closer itself. Handle the before/after-opener cursor conventions already
accepted by `actions.rs`. Preserve eligibility across a paired `()` change or an
immediately following autopair `)` change, using the existing recent-parenthesis
tracking. Unrelated text changes invalidate it. Keep enough transition state until the
client applies the deletion so completion and formatting requests cannot consume one
another's evidence; discard it once the normalized change arrives. Repeated requests on
one unchanged snapshot should return consistent results.

Address request ordering explicitly: a `(` completion request may arrive while the
buffer is still `#optional (` or `#optional ()`. For this confirmed pending transition
only, derive argument completion from a temporary document with that owned space removed
and map candidate edit ranges back to the actual document. Put reusable
normalization/position mapping in core. Use standard, nonoverlapping additional edits to
delete the spacer if the user accepts an argument candidate before formatting applies.
After the deletion `didChange`, use the ordinary argument-completion route with no extra
deletion. Verify both request orders and ensure no completion edit uses normalized
coordinates against the unnormalized buffer. Do not relax the general xprompt parser or
offer argument completions for arbitrary prose that resembles this transition.

External-editor support is conditional: the client must execute completion item
commands, send/apply on-type formatting, and request/show automatic completion on the
advertised `(` trigger. LSP does not report every cursor movement or provide a universal
command to force a popup. The server can invalidate on text changes and validate the
current caret, but cannot promise the widget's exact move-away-and-back cancellation
behavior. Document these limits; clients missing these facilities retain ordinary
editing. Do not add editor-specific hooks or generic background document rewrites to
compensate.

### 4. Documentation and integration

Add the accepted-completion example and zero-input exclusion next to the argument typing
conveniences in `docs/xprompt.md`. Explain the optional-input case, normal menu
settings, and the external-editor capability limits. Update the widget spacer test
description. No new configuration or keymap is needed.

Build/install the matching linked Rust extension before Python verification. Both
repositories belong in the implementation turn's host final declaration. Per
`docs/rust_backend.md`, host finalization commits the changed linked core first and
writes its pushed SHA to `sase-core-revision.txt` before committing the primary repo.
Ensure that dependency pin is included in the landing result; do not invent a SHA or
manually create commits. If consuming an already-landed core change instead, advance the
pin using the supported ratchet.

## Verification and acceptance

Use focused regression tests around behavior, not tests that only restate helper
implementation. Extend existing suites or split a focused neighbor if a file would
become oversized.

- **Core and binding:** input-bearing accepted spacer succeeds; zero-input,
  unowned/stale/malformed records and excluded regions do not. Include `#!`, namespaced
  references, nested invocation context, multiline text, and an astral Unicode character
  before the reference. Assert the exact deletion range and UTF-16 round trip.
- **Widget:** extend `tests/ace/tui/widgets/test_xprompt_completion_spacer.py` to cover
  manual multi-candidate acceptance, single-candidate Ctrl+T, soft completion, and
  selector insertion. Assert text, caret, consumed pending state, actual argument
  candidates, and successful acceptance of `topic=`. Test disabled automatic menus,
  zero-input entries, selection and normal-mode guards, intervening keys, manual/pasted
  lookalikes, cursor movement, stale text, and completion before punctuation where no
  spacer was inserted.
- **Editing integrity:** extend `test_prompt_pair_editing.py` and existing snippet
  spacer coverage for preserved prefix/suffix, unsafe-pair fallback, one undo/redo step,
  closer skip, paired backspace, nested snippet remapping, and Tab/Shift+Tab
  progression. Keep required-input colon/double-colon, comma/colon spacer, and existing
  argument-name completion tests passing.
- **LSP:** add focused server tests and a stdio JSON-RPC test alongside
  `crates/sase_xprompt_lsp/tests/jsonrpc_stdio_on_type_formatting.rs`. Exercise real
  completion acceptance through the returned command, document sync, `(` formatting,
  application of its edit, and argument completion. Include command/change ordering,
  plain `(` and combined/separate autopairs, both completion/formatting request orders,
  Unicode ranges, unchanged repeated requests, suffix preservation, multiple documents,
  close/reopen, stale commands, and no acceptance command. Assert no command is attached
  to zero-input or non-spacer items and no deletion targets authored whitespace.

Run the relevant widget regression tests and targeted Rust tests during implementation.
In core use its `just test -p <crate> [filter]` recipes, never bare `cargo`; include
`sase_core`, `sase_core_py`, and `sase_xprompt_lsp`. Finish with formatting and
`sase tool run check` in each changed repository, following their instructions and
`/sase_monitor` for long verification. The targeted tests do not replace either
repository's check gate. Do not run `just check-full` for this task. Read the TUI
screenshot guidance and run targeted visual checks only if rendered output or snapshot
coverage changes.

Done means the user can accept an input-bearing xprompt and immediately type `(` to
enter its arguments in the widget, equivalent standard-LSP protocol flows pass,
zero-input and unrelated text retain their behavior, and both repositories pass their
required checks with the core dependency pin accounted for. Report the supported editor
capabilities without claiming every editor will automatically display the menu.
