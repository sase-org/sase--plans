---
tier: epic
title: Replace star model shortcuts with equals shortcuts
goal:
  ACE and xprompt-aware external editors use =alias and ==model for model completion
  while retiring the former star syntax and preserving their shared completion contract
phases:
  - id: core_equals_shortcuts
    title: Migrate the shared core and LSP shortcut grammar
    depends_on: []
    description:
      "core_equals_shortcuts: change the Rust-owned model-shortcut detector, edit
      planner, Python bindings coverage, and xprompt LSP trigger/rendering/tests from
      star markers to equals markers, then pass the sase-core repository checks."
    size: medium
  - id: ace_equals_shortcuts
    title: Adopt equals shortcuts throughout ACE
    depends_on:
      - core_equals_shortcuts
    description:
      "ace_equals_shortcuts: pin the completed core revision, migrate ACE prompt
      completion behavior, tests, help, configuration reference, user documentation, and
      affected visual goldens to =alias and ==model without duplicating the Rust
      grammar."
    size: medium
  - id: integrated_verification
    title: Verify the cross-repository migration
    depends_on:
      - ace_equals_shortcuts
    description:
      "integrated_verification: rebuild ACE against the linked core revision, prove
      ACE/LSP parity and the hard cutover from star syntax, inspect the updated visuals,
      and run both repositories' required verification gates."
    size: small
proposed_by: bbugyi200.athena.0i4
bead_id: sase-z3
create_time: 2026-09-09 19:52:20
status: wip
---

- **PROMPT:**
  [prompts/202609/equals_model_shortcuts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/equals_model_shortcuts.md)
- **BEAD:**
  [sase-z3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z3/README.md)

# Plan

## Outcome and decisions

Replace the recently introduced model shortcuts as a hard syntax cutover:

- `=alias` opens the alias-only catalog and accepts a row as `%m:@alias`.
- `==model` opens the concrete-model catalog, including provider-qualified filtering,
  and accepts a row as `%m:model`.
- `*alias` and `**model` no longer create model-shortcut contexts, no longer auto-open
  ACE model menus, and no longer receive shortcut completions from the LSP. They remain
  ordinary prompt text. This deliberately avoids a compatibility period that would
  retain the shift-requiring syntax the change is intended to replace.
- The accepted `%m:` expansions, alias/model catalog filtering, catalog order, selection
  validation, whitespace insertion, full-token replacement, UTF-16/CRLF handling,
  cold-catalog behavior, and ACE/LSP parity remain unchanged.
- Keep the existing marker-neutral Rust/Python wire function names and schema version:
  the payload shape is unchanged, only the recognized source syntax changes. Rename
  star-specific private helpers, test names, comments, and documentation so they
  describe equals shortcuts accurately.
- Do not alter unrelated uses of `*`, especially prompt NORMAL-mode star search, and do
  not alter the existing `g=` frontmatter-panel command. Do not edit the published
  `sase-core-rs` dependency floor or `uv.lock`; release automation owns that version
  window. The source revision pin is the integration mechanism for this change.

## Phase 1: `core_equals_shortcuts`

Work in the linked `sase-core` repository opened through `/sase_repo`.

1. Update `crates/sase_core/src/editor/model_alias_shortcut.rs` so the shared detector
   recognizes exactly one leading `=` as `ModelShortcutKind::Alias` and exactly two as
   `ModelShortcutKind::Model`. Replace star-specific scanning with an equals/marker-
   neutral helper, reject three or more leading equals signs and equals signs later in
   the same token, and make the former `*`/`**` inputs return no shortcut context.
2. Preserve the existing eligibility boundaries and exclusions: prompt/document start,
   logical line start, or an immediately preceding ASCII space; no tab boundary; nothing
   embedded or escaped; and nothing inside inline/fenced/disabled literal zones,
   frontmatter, Jinja, placeholders, or directive-owned regions. Adapt the marker-pair
   protection to equals-delimited tokens such as `=text=` and `==text==`.
3. Keep catalog filtering and edit semantics intact. Alias rows remain limited to
   effective `implicit_alias`/`user_alias` entries; explicit shortcuts remain limited to
   concrete model rows while retaining provider-scope filtering; stale or unsafe
   selections still fail closed; accepting a row still replaces the full live token and
   preserves the established space/tab/newline behavior.
4. In `crates/sase_xprompt_lsp/src/server.rs` and
   `crates/sase_xprompt_lsp/src/lsp_convert.rs`, advertise `=` instead of `*` as the LSP
   completion trigger, render `filterText` as `=<query>` or `==<query>`, and update the
   shortcut ownership/comments without changing completion item kinds, ordering,
   preselection, documentation, or text edits.
5. Update the Rust unit tests, PyO3 binding shape tests in
   `crates/sase_core_py/src/lib.rs`, LSP server tests, and both stdio JSON-RPC shortcut
   suites. Cover bare and filtered aliases/models, provider-qualified models,
   single/double marker transitions in both directions, mid-token replacement,
   Unicode/CRLF offsets, malformed or missing catalogs, protected regions, unsafe
   catalog values, literal old-star inputs, and `=` present/`*` absent in advertised
   trigger characters.
6. Do not manually change Cargo package versions. Format the repository and run its
   mandatory `just check` gate, which includes the complete workspace and PyO3 binding
   coverage. Submit the phase through the host finalizer so the resulting core commit is
   available to the dependent SASE phase.

## Phase 2: `ace_equals_shortcuts`

Work in the primary `sase` repository only after Phase 1 has produced a committed core
revision.

1. Advance `sase-core-revision.txt` to the exact committed core revision containing the
   equals grammar. Rebuild/install both `sase_core_rs` and `sase-xprompt-lsp` from the
   workspace-matched linked checkout before running Python parity tests. Leave
   `pyproject.toml` and `uv.lock` unchanged because the release metadata reconciler owns
   the published dependency window.
2. Update the ACE model alias/explicit completion adapters and completion mixin comments
   to describe `=alias` and `==model`. Continue calling the shared Rust context and edit
   bindings; do not introduce a Python parser or a second marker grammar. Preserve the
   off-thread warmed catalog path, stale-result rejection, loading/unavailable rows,
   manual `Ctrl+T`, `Enter`/`Ctrl+L` acceptance, undo/redo, and the automatic
   `auto_directive_menu` behavior.
3. Change the prompt-widget tests so typing one equals sign opens alias completion,
   typing the second switches the live menu to concrete models even when the alias
   catalog is empty/loading, and backspacing returns to the alias menu. Retain coverage
   for selection stability, full-token edits, provider scopes, spacing, Unicode,
   failure/retry paths, non-selectable placeholders, and unknown literal text. Add
   explicit regression assertions that insert-mode `*alias`/`**model` no longer open a
   model shortcut while NORMAL-mode `*` search and `g=` frontmatter navigation remain
   unaffected.
4. Migrate the installed-binary ACE/LSP parity suite to equals terminology and inputs.
   Assert `=` is advertised and `*` is not, ACE and LSP return the same alias/model rows
   and edits, filter text uses the new prefixes, protected equals contexts do not own an
   empty shortcut list, and old star inputs cannot yield a `%m:` shortcut edit. Rename
   star-specific test functions or the parity module where doing so removes stale public
   vocabulary without changing generic model-shortcut APIs.
5. Update the model shortcut examples and terminology in `docs/ace.md`,
   `docs/editor.md`, `docs/xprompt.md`, `docs/configuration.md`, the default-config
   comment, and the ACE `?` help row. Document the hard cutover, the `=` LSP trigger,
   retained context exclusions, and literal behavior for unaccepted queries. Sweep the
   relevant source/tests/docs for stale `*alias`, `**model`, “star shortcut,” and
   “double-star” language while excluding unrelated wildcard, Markdown, and Vim-search
   uses.
6. Change the visual test inputs/titles to equals syntax, regenerate only the affected
   model-shortcut PNG goldens, inspect the actual/expected/diff artifacts or the new
   images, and rerun the targeted visual module without update mode to prove exact
   convergence. Run the focused alias, explicit-model, installed-binary parity, model
   filtering, and prompt-keybinding tests, followed by the primary repository's required
   `just check` gate.

## Phase 3: `integrated_verification`

1. Open the linked core through `/sase_repo`, confirm the SASE revision pin names the
   committed Phase 1 revision (or a descendant containing it), and configure the SASE
   build with the linked path returned by that command. Reinstall so the Python
   extension and LSP binary are built from the same checkout rather than a previously
   published wheel.
2. Rerun `just check` in `sase-core`. In `sase`, rerun the focused prompt alias,
   explicit-model, ACE/LSP parity, NORMAL-mode star-search, `g=` frontmatter, and model
   completion visual tests. The visual run must be without snapshot-update mode.
3. Exercise both automatic and manual completion at prompt/line start and after an ASCII
   space: `=la` must expand to `%m:@large`, `==gpt` and a provider-qualified query must
   expand to their canonical models, the second equals sign and backspace must switch
   menus, and old star text must remain literal. Confirm the LSP initialization payload
   advertises `=` but not `*` and its completion edits/filter text match ACE.
4. Run the SASE epic's final `just check-full` through `/sase_monitor` as required for a
   long exhaustive gate. If verification exposes a defect, repair only the owning
   phase's implementation, repeat its focused checks, and then rerun the complete final
   sequence until every gate passes.

## Acceptance criteria

- ACE and the xprompt LSP expose only `=alias` and `==model` as model shortcuts.
- The former `*alias` and `**model` syntax is not advertised, detected, or expanded.
- Alias/model filtering, provider scoping, edit/caret behavior, asynchronous catalog
  loading, and ACE/LSP installed-binary parity remain covered and passing.
- Help, configuration reference, user docs, test names, and affected visual snapshots
  consistently teach equals syntax, with unrelated star and equals keybindings intact.
- `sase-core` passes `just check`; `sase` passes focused/visual checks, `just check`,
  and the monitored `just check-full` run against the pinned linked core revision.
