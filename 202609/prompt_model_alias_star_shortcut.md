---
tier: tale
title: Add a prompt model-alias star shortcut
goal: "Typing * at the start of a prompt line or immediately after a literal space
  expands to %m:@ and immediately opens model-alias completion without disturbing
  literal-star or vim search behavior in other contexts.

  "
size: small
proposed_by: bbugyi200.athena.07c
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.chop.refresh_docs.sase.9_721159.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.chop.refresh_docs.sase.9_721159.1/README.md)
  - [bbugyi200.athena.chop.refresh_docs.sase.9_721159.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.chop.refresh_docs.sase.9_721159.2/README.md)
  - [bbugyi200.athena.sase-xe.16.10](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.16.10.md)
  - [bbugyi200.athena.sase-xe.16.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.2/README.md)
  - [bbugyi200.athena.sase-xe.16.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.3/README.md)
  - [bbugyi200.athena.sase-xe.16.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.4/README.md)
  - [bbugyi200.athena.sase-xe.16.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.5/README.md)
  - [bbugyi200.athena.sase-xe.16.7](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.7/README.md)
  - [bbugyi200.athena.sase-xe.16.8](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.8/README.md)
  - [bbugyi200.athena.sase-yf.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yf.2/README.md)
  - [bbugyi200.athena.sase-yf.3.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yf.3.1/README.md)
  - [bbugyi200.athena.toobig-50.artifact_ref_models.0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.toobig-50.artifact_ref_models.0.md)
  - [bbugyi200.athena.toobig-50.processing.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.processing.0/README.md)
  - [bbugyi200.athena.toobig-50.resolve.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.resolve.0/README.md)
  - [bbugyi200.athena.toobig-50.store.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.store.0/README.md)
  - [bbugyi200.athena.toobig-50.test_app.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.test_app.0/README.md)
  - [bbugyi200.athena.toobig-50.test_notify_handler.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.test_notify_handler.0/README.md)
  - [bbugyi200.athena.toobig-50.test_resolve.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.test_resolve.0/README.md)
  - [bbugyi200.athena.toobig-50.test_view_files_pager.0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.toobig-50.test_view_files_pager.0/README.md)
- **COMMITS:**
  - [8c4f8fd](https://github.com/sase-org/sase/commit/8c4f8fd22ae92ddb1e56e67c13778673476aa79d)
    — test(tui): add offline fleet refresh fixture
  - [f081f23](https://github.com/sase-org/sase/commit/f081f23038f4bb21170cc7860c24e5894ab36616)
    — feat(dispatch): isolate third-party provider hooks
  - [a95d7c1](https://github.com/sase-org/sase/commit/a95d7c1ddfcd34d95b56de75d3ad2265f8e3b1a0)
    — feat(ace): add prompt model alias shortcut
  - [18b0a91](https://github.com/sase-org/sase/commit/18b0a91a264ddd3e9b55a609c9a62c209b87a06a)
    — feat(machine): add target bootstrap CLI
  - [6ae983d](https://github.com/sase-org/sase/commit/6ae983ddc2b607513cf5cebf1c6e9ea5ea2318f6)
    — test(tui): add Fleet and Focus PNG coverage
  - [5015d76](https://github.com/sase-org/sase/commit/5015d76e9561cc68e0526627473aef9e159bc647)
    — chore(deps): ratchet sase-core pin and floor
  - [ace9e2c](https://github.com/sase-org/sase/commit/ace9e2cd468ff998b1a3b849ea3f7adf12820231)
    — feat(dispatch): discover tailnet machines
  - [5620ac0](https://github.com/sase-org/sase/commit/5620ac028d1a56439705849330f5937946b1fb7b)
    — fix(model-alias): harden shortcut completion behavior
  - [938d927](https://github.com/sase-org/sase/commit/938d9276b0b9930e410205f6ed323e025c725c55)
    — refactor: split hint input processing
  - [7678e4f](https://github.com/sase-org/sase/commit/7678e4f04278f441eebef300cc63d5fdf3adb20d)
    — refactor(artifact-ref): split artifact reference models
  - [890660e](https://github.com/sase-org/sase/commit/890660e257526d3c8fd1d78ec3e0ab53a062321c)
    — docs(dispatch): add remote setup runbook
  - [d7e6ca1](https://github.com/sase-org/sase/commit/d7e6ca1ff70ba4e11f95a73d4199bd6e845edd54)
    — refactor(usage): split provider usage store
  - [67f2ca6](https://github.com/sase-org/sase/commit/67f2ca6040cf914f89862be5cf7948c56a09002a)
    — refactor(pager): split resolver implementation modules
  - [2601211](https://github.com/sase-org/sase/commit/2601211d109198921d0fe2a527589af93cd70bc6)
    — test(ace): split view files pager tests
  - [1ea2582](https://github.com/sase-org/sase/commit/1ea2582f70b0b01bf2403135c4fead119d87422b)
    — docs: refresh user reference for current behavior
  - [40124f3](https://github.com/sase-org/sase/commit/40124f34a27a805abffea7f1fd5d56edfa2d8174)
    — test: split notify handler tests
  - [c5e8d4e](https://github.com/sase-org/sase/commit/c5e8d4e96e06efe771aaa161ce34931c463a1528)
    — test(pager): split app tests by behavior
  - [990a108](https://github.com/sase-org/sase/commit/990a108f8e687ebf66e7bd3f1fe2047e9f87ea57)
    — docs: align refreshed reference with runtime behavior
  - [8746c44](https://github.com/sase-org/sase/commit/8746c4424e20d644a51a6f30190e8500ccce0a2f)
    — test(pager): split resolve tests by behavior

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
