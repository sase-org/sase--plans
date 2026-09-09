---
tier: epic
status: done
title: External-editor star model alias completion
goal:
  Give xprompt LSP clients the same safe, canonical star-triggered model alias expansion
  as the ACE prompt input widget.
phases:
  - id: core_lsp_star_alias
    title: Share the star alias contract with the xprompt LSP
    depends_on: []
    description:
      "core_lsp_star_alias: make the Rust shortcut filter and edit plan directly
      consumable by LSP completion, then add the trigger, response projection, and core
      protocol coverage."
    size: medium
  - id: sase_star_alias_parity
    title: Pin the core and prove ACE/LSP parity
    depends_on:
      - core_lsp_star_alias
    description:
      "sase_star_alias_parity: ratchet to the landed core, make ACE consume the shared
      alias-only filter, add installed-binary parity tests, update editor documentation,
      and run repository verification."
    size: medium
proposed_by: bbugyi200.athena.sase-yf.land.w3
bead_id: sase-ys
create_time: 2026-09-09 19:52:27
---

- **PROMPT:**
  [prompts/202609/lsp_star_model_alias_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/lsp_star_model_alias_completion.md)
- **BEAD:**
  [sase-ys](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ys/README.md)

# External-editor star model alias completion

## Outcome and tier

Typing `*` in an xprompt-aware external editor should open model-alias completion just
as it does in the ACE prompt input. A selected row such as `@large` replaces the whole
live `*query` token with `%m:@large`, using the same literal-zone exclusions, catalog
order, canonical alias validation, whitespace behavior, and UTF-16 ranges as ACE.

This is an epic because the implementation crosses the `sase-core` release boundary. The
shared Rust contract and `sase-xprompt-lsp` must land together first; only then can the
SASE repository pin that revision, consume any added binding, install the matching
binary, and prove cross-surface parity. The second phase therefore depends on the first.

## User-visible contract

- Advertise `*` as an LSP completion trigger. Manual editor completion also works while
  the caret is in a valid `*alias` token. The ACE-only
  `ace.prompt_completion.auto_directive_menu` setting does not govern an editor client's
  trigger policy and gains no new meaning.
- Recognize a star token only through the existing Rust
  `detect_model_alias_shortcut_context` contract: the star is at absolute prompt offset
  zero, the start of a logical line, or immediately after an ASCII space, and the caret
  is after the star. Preserve all current exclusions for escaped/embedded stars,
  Markdown emphasis, paths, inline and fenced code, disabled prompt regions,
  frontmatter, Jinja, placeholders, and directive-owned input.
- Read the same launcher-materialized model catalog already used by LSP `%model:`
  completion. Filter it through the shared Rust model-alias shortcut filter so only
  effective `implicit_alias` and `user_alias` rows appear, in canonical catalog order,
  with case-insensitive prefix matching. Do not add another config loader, provider
  probe, routing resolver, helper subprocess, or LSP-only alias vocabulary.
- Keep the visible alias label and the existing `%model:` target/provenance
  documentation, while making the expansion to `%m:@<alias>` clear in LSP label details.
  Mark the first row preselected where the client honors LSP `preselect`. Preserve
  server order with `sortText`.
- Return a server-filtered incomplete completion list and a `filterText` based on the
  star prefix actually typed. This makes clients re-request as the query changes and
  prevents their local prefix filter from dropping `@large` merely because the source
  text is `*la`.
- Acceptance uses the existing Rust `plan_model_alias_shortcut_edit` result as the LSP
  `textEdit`: replace the complete token, including text to the right of a mid-token
  caret, and insert the canonical `%m:@alias`. At prompt/line end append one ASCII
  space; before a tab append none; before an existing ASCII space preserve the entire
  whitespace run and leave the caret after its first space.
- A valid star context with no aliases or no matches returns an empty shortcut response
  instead of falling through to unrelated prose/file/snippet completion. Invalid star
  contexts leave the existing LSP completion classifier untouched. Missing or malformed
  model catalogs degrade to no alias rows and never rewrite literal text.
- This adds completion only. Unaccepted `*query` text retains no launch-time meaning; no
  editor-specific plugin, diagnostic, semantic-token rule, hover rule, keybinding, or
  new configuration option is required.

## Shared design

Keep frontend-independent behavior in `sase-core`:

1. A public alias-shortcut filter accepts model catalog wire rows plus the bare query,
   internally reuses `filter_model_completion_entries`, restricts results to the two
   alias kinds, and preserves canonical order. The existing edit planner uses this same
   helper when validating a selected alias.
2. The edit planner remains the sole owner of `%m:@alias` construction, full-token
   replacement, trailing-space policy, and resulting caret. Make its edit itself
   LSP-complete: when reusing an existing ASCII space, include exactly that first space
   in the edit range and reinsert one space in `new_text`. The final document is
   byte-for-byte unchanged apart from the star expansion, and the desired caret is now
   always the end of the returned text edit. ACE can continue applying the same plan;
   the LSP no longer needs a frontend-specific caret command or whitespace workaround.
3. Export the alias-only filter from the Rust crate and through the thin PyO3 surface
   using the existing model-entry wire schema. ACE uses that binding to obtain its alias
   rows and then performs only Textual metadata/rendering conversion. The Rust LSP calls
   the same Rust helper directly and performs only `CompletionItem` conversion.

Do not fold the shortcut into the general directive classifier or build a second
Markdown/context parser. The distinct shortcut entry point is needed because its source
token, replacement range, insertion text, and empty-result ownership differ from an
ordinary `%model:` value, while the catalog filtering and edit semantics are shared.

## Phase 1: Share the star alias contract with the xprompt LSP

Work in the linked `sase-core` repository opened through `/sase_repo`, and follow its
`AGENTS.md`.

### Core contract

- Refactor `crates/sase_core/src/editor/model_alias_shortcut.rs` to expose the focused
  alias-only filter described above. Reuse it from selected-alias validation instead of
  repeating `@` query normalization or alias-kind checks.
- Normalize `ModelAliasShortcutEditWire.edit` so its range/new text encode the desired
  final caret for every whitespace case. In particular, consume and reinsert exactly one
  already-present ASCII space; keep tabs/newlines and the rest of a whitespace run
  untouched. Preserve `replacement_range` on the detected context as the star token's
  range and document that a planned edit may deliberately extend one character beyond
  it.
- Export the helper through `crates/sase_core/src/editor/mod.rs` and
  `crates/sase_core/src/lib.rs`, then add the matching rectangular-list PyO3 binding and
  binding documentation/tests in `crates/sase_core_py/src/lib.rs`. Do not create a new
  model catalog schema or Python fallback.
- Extend core tests for bare and mixed-case queries, alias-only filtering, stable order,
  alternate/non-alias rejection, a mid-token caret, Unicode before and within the token,
  CRLF, line/prompt end, tabs, one and multiple existing spaces, and the invariant that
  the planned caret equals the end of the applied edit.

### LSP integration

- In `crates/sase_xprompt_lsp/src/server.rs`, detect a shortcut through the shared core
  function before the generic completion classifier. On a valid context, load the
  existing materialized model catalog, call the shared alias-only filter, validate each
  candidate through the shared edit planner, and return that shortcut response even when
  it is empty.
- Reuse the existing model-catalog candidate conversion and `%model:` metadata helpers
  for labels, target detail, documentation, alias kind, and stable ordering. Add only a
  focused conversion in `lsp_convert.rs` for the star-specific `textEdit`, typed
  `filterText`, expansion label detail, incomplete-list response, and first-row
  `preselect`; do not fork model metadata formatting.
- Add `*` to `CompletionOptions.trigger_characters`. Do not change document eligibility
  or require a client command after acceptance.
- Add server and conversion tests that cover trigger advertisement; bare, partial,
  case-insensitive, later-line, and after-space completion; alias-only rows; no-match
  ownership; first-row preselection; `filterText`/`sortText`/documentation; whole-token
  UTF-16 edits with a mid-token caret; every trailing-whitespace case; missing and bad
  catalogs; and all protected or ordinary-writing contexts from the shared detector. Add
  a JSON-RPC stdio case that initializes the real service, opens an eligible document,
  requests `*alias` completion, and asserts the wire-level completion-list and text-edit
  shape.

Run `just check` (or `./scripts/check.sh`) from the `sase-core` root; crate-only tests
are not an acceptable substitute because the PyO3 and LSP contracts must ship together.
Finish through host-owned finalization so phase 2 can depend on a durable core revision.

## Phase 2: Pin the core and prove ACE/LSP parity

First open `sase-core` through `/sase_repo` and confirm the phase-one exports and LSP
behavior exist at its landed revision. Ratchet `sase-core-revision.txt` through the
established tool, and update the published `sase-core-rs` dependency floor/lock only to
the real released version containing the new binding and edit semantics. Never invent a
SHA/version or land a Python caller ahead of its core.

- Add a thin wrapper in the existing model-completion wire adapter for the new
  alias-only binding, reusing the current `ModelCompletionEntry` serialization and
  deserialization. Change ACE's `build_model_alias_shortcut_candidates` path to consume
  that wrapper. Keep `ModelCompletionMetadata` projection and Textual rendering in
  Python, and keep acceptance-time revalidation through `plan_model_alias_shortcut_edit`
  so a stale selected row cannot edit changed text.
- Adjust ACE shortcut tests for the cursor-complete shared edit range, proving identical
  final text, caret placement, atomic undo/redo, and whitespace preservation. Do not
  alter menu presentation or regenerate PNG goldens unless an actual intentional visual
  change is discovered.
- Extend the existing real-binary LSP harness so tests can inspect
  `CompletionList.isIncomplete`, `preselect`, label details, multiline UTF-16 positions,
  and raw text edits. Add focused ACE/LSP parity coverage using one deterministic model
  catalog: compare the alias names and ordering produced from bare and filtered stars;
  apply each LSP edit and compare its text and resulting caret with ACE's shared edit
  plan; cover mid-token suffixes, Unicode/CRLF, existing-space/tab/newline/end spacing,
  no matches, and protected contexts. Assert concrete models and providers never appear
  in the shortcut response while ordinary `%model:` completion remains unchanged.
- Keep using `src/sase/integrations/xprompt_lsp.py`'s existing launch-time model catalog
  materialization. Add or tighten its tests only where needed to prove the alias rows
  handed to the LSP are the same effective built-in/plugin/user catalog consumed by ACE;
  do not add per-keystroke bridge calls.
- Update `docs/editor.md` and the editor-LSP/model-completion sections of
  `docs/xprompt.md` to document `*alias`, canonical expansion, trigger boundaries,
  literal exclusions, whole-token/spacing behavior, catalog snapshot/restart behavior,
  and manual completion. Cross-link the existing ACE description rather than copying a
  second divergent contract. Clarify that `auto_directive_menu` remains an ACE prompt
  setting. Update `docs/ace.md` only as needed to point to the now-shared editor
  behavior; `docs/configuration.md` and `src/sase/default_config.yml` need no semantic
  change.

Run `just install` first so `sase_core_rs` and `sase-xprompt-lsp` come from the same
opened core revision. Run the focused model-alias widget, model-catalog, LSP parity, and
launcher-environment tests, then `just check` under the repository's two-speed rules.
The epic lander runs `just check-full` only through `/sase_monitor`, with the required
`TESTING`/`TESTED` status pair, and records any unrelated baseline failure separately.

## Completion criteria

The feature is complete when an eligible external editor automatically requests
completion after `*`, shows only canonical effective aliases with existing model
metadata, re-filters them through the shared Rust contract, and accepts one into the
same final prompt text and caret location as ACE for all token/whitespace/Unicode cases.
Literal and protected stars stay untouched, empty or unavailable catalogs do not fall
through to unrelated rows, `%model:` completion is unchanged, both installed Rust
artifacts come from the pinned compatible core, focused parity tests pass, `sase-core`
passes its full `just check`, and the combined SASE tree passes its required
verification.
