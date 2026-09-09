---
tier: tale
title: Add a prompt model-alias star shortcut
goal: 'Typing * at the start of a prompt line or immediately after a literal space
  expands to %m:@ and immediately opens model-alias completion without disturbing
  literal-star or vim search behavior in other contexts.

  '
size: small
proposed_by: bbugyi200.athena.07c
status: done
---

# Plan: Add a prompt model-alias star shortcut

## Behavior and edit path

- Add a small, pure prompt-edit planner near the existing auto-pair and alternate-
  syntax planners. Given the current document, cursor offset, and typed character, it
  should return one `TextEdit` that inserts `%m:@` and parks the cursor after `@` only
  when the character is `*` and the collapsed cursor is either at document/line start or
  immediately after a literal ASCII space. Preserve the preceding space rather than
  absorbing it. Treat a newline as line-start context, but do not broaden the trigger to
  tabs or other whitespace.
- Intercept the shortcut in `PromptTextArea`'s INSERT-mode key path before Textual's
  default insertion. Apply it through the existing planned keyboard-replacement helper
  so completion state is cleared consistently and the entire expansion is a single
  undoable edit. Stop/prevent the original `*` event only when the planner applies;
  otherwise retain ordinary selection replacement, literal-star insertion, and all
  existing key handling.
- Keep the shortcut scoped to launch-prompt mode, where `%model` directives select an
  agent model. Feedback prompts should continue accepting a literal `*`. NORMAL-mode `*`
  must continue to run the existing whole-word vim search, including at column zero and
  after whitespace.
- After applying `%m:@`, call the existing directive-argument completion opener
  directly. The leading `@` will reuse the model catalog's alias-only filtering and the
  existing model-row presentation; do not add another catalog, resolver, parser, or
  Rust-core behavior. Treat this two-keystroke shorthand as an explicit completion
  request, so it opens the alias menu even when general automatic directive menus are
  disabled. Keep the keystroke path in-memory and lock-free, relying on the already
  warmed/cached model completion data.

## Coverage

- Add focused pure-planner cases for document start, a later line start, and a literal
  preceding space, including an insertion before existing trailing text. Cover the
  non-trigger cases: after a non-space character, after a tab/other whitespace, and for
  any character other than `*`.
- Add Textual integration coverage proving that both `*` at line start and ` *` after
  prose produce the exact `%m:@` text, preserve surrounding text, leave the cursor after
  `@`, open a `directive_arg` panel labeled for model aliases, and expose only alias
  candidates from a deterministic patched model catalog. Accept one candidate with the
  existing completion key to exercise the shortcut through canonical insertion.
- Cover interaction boundaries: a selected range still receives a literal `*`, a
  feedback bar does not expand, the explicit shortcut still opens completion when
  `auto_directive_menu` is false, and one NORMAL-mode `u` reverses the full expansion.
  Retain or extend the existing prompt-star-search regression so NORMAL-mode `*` is
  demonstrably unchanged.

## Discoverability and documentation

- Add the INSERT-mode shortcut to the shared Prompt Input section of ACE's `?` help,
  using a description short enough for the fixed-width help layout and wording that
  distinguishes it from the existing NORMAL-mode `*`/`#` word search. Extend the help
  data test so all ACE tabs are checked for the new row.
- Update the prompt-input key table and completion prose in `docs/ace.md` with the exact
  expansion, its line-start/literal-space trigger, prompt-mode scope, and its reuse of
  alias-only `%model` completion. Update `docs/configuration.md` to state that this
  explicit shorthand remains available when `auto_directive_menu` is false, just like
  manual completion. No default configuration or keymap entry is needed because this is
  a typed text expansion rather than a remappable command binding.

## Verification

- Run `just fmt`, then focused tests for the new shortcut, prompt star search,
  directive-value completion, and shared help rows.
- Run the required agent gate with `just check` (running `just install` first if the
  ephemeral workspace environment needs refresh). Do not use a visual snapshot update
  unless the fixed-width help rendering actually changes a checked-in PNG.
