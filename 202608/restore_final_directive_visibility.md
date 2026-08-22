---
tier: tale
title: Restore public final directive completion
goal:
  Typing a percent directive prefix advertises the public final directive in ACE and
  external LSP editors, with positive cross-surface regression coverage.
size: small
proposed_by: bbugyi200.athena.0aa
create_time: 2026-08-22 10:43:18
status: wip
---

# Restore public `%final` directive completion

## Context and root cause

Epic `sase-s0` completed the configured finalizer catalog, selector-aware value
completion, ACE presentation, LSP rendering, documentation, and public name exposure.
Its later parity tale also added positive ACE/LSP assertions for `%final` names and LSP
snippets. A concurrent integrity-acceptance stitch then restored the old hidden policy:

- `sase-core` commit `f7e8247` changed
  `crates/sase_core/src/editor/directive.rs::HIDDEN_COMPLETION_DIRECTIVES` from empty to
  `['final']`.
- Main-repository commit `47830f9` changed the focused ACE unit tests back to expecting
  `%final` to remain hidden, even though the production Python adapters and public docs
  had already been changed to expose it.
- The later `sase-s0` land commits added positive interaction, parity, and LSP tests but
  did not remove the reintroduced Rust guard or reconcile the stale negative tests.

The current shared Rust contract still contains `%final`, so the generic directive
catalog exposes it. Name candidate construction in the Rust core skips the hidden set,
however, and `sase-xprompt-lsp` uses the same predicate to suppress argument snippets.
After rebuilding both local artifacts from the linked core, ACE returns no `%final` row
for `%`, and the LSP returns no `%final` name or snippet rows for `%`, `%f`, or
`%final`. The positive ACE panel test fails, while the obsolete hidden-name unit tests
pass. This is a visibility-policy regression, not a missing catalog, renderer, or
selector-value implementation.

## Implementation

1. In the linked `sase-core` repository, restore the completed epic's public visibility
   policy by removing `final` from `HIDDEN_COMPLETION_DIRECTIVES` while retaining the
   shared predicate as the single authority for any genuinely hidden directives. Do not
   add a Python-side exception, duplicate the directive contract, or bypass the shared
   guard only in the LSP; the canonical name builder and snippet builder must agree.

2. Replace the stale Rust hidden-name unit expectation with positive coverage that both
   `%f` and `%final` resolve to the canonical `%final` completion row. Retain the
   current positive LSP tests for canonical name completion, single-selector and
   parenthesized snippets, snippet-disabled clients, and clause-local replacement
   ranges. Run those tests first so the guard removal is proven to repair both
   external-editor paths without changing selector argument semantics.

3. In the main `sase` repository, reconcile
   `tests/ace/tui/widgets/test_directive_completion_candidates.py` with the public
   contract: require `%final` in the canonical directive set, rename the obsolete
   hidden-name test, and positively assert the canonical row for both `%f` and `%final`.
   Keep the existing generic-catalog, prompt interaction, and ACE/LSP parity tests as
   independent coverage. Production Python completion code and `docs/xprompt.md` already
   describe and delegate to the correct public Rust contract, so change them only if
   focused verification exposes a separate mismatch.

4. Rebuild both development artifacts from the modified linked core before running the
   cross-repository tests: refresh `sase_core_rs` for ACE and install the matching
   `sase-xprompt-lsp` binary for the external-editor harness. This avoids a false pass
   or failure caused by testing one surface against a stale artifact.

## Verification

1. In `sase-core`, run the focused directive-name and LSP completion tests that cover
   `%`, `%f`, `%final`, snippet-capable and snippet-disabled clients, then run the
   repository-required `just check` so the core, binding, and server remain consistent.
2. In `sase`, run `just install` and install the matching local LSP binary. Run:
   - `tests/ace/tui/widgets/test_directive_completion_candidates.py`;
   - `tests/ace/tui/widgets/test_directive_completion_interactions.py::test_ctrl_t_at_percent_opens_directive_panel`;
   - `tests/test_xprompt_directive_completion_parity.py::test_ace_and_lsp_directive_name_rows_match`;
   - `tests/completion/test_candidates_providers.py::test_directive_candidates_use_shared_contract_and_expose_final`;
   - the existing `%final` selector parity suite to confirm name exposure did not alter
     configured add/remove/clear completion.
3. Run the main repository's required `just check`. If its scoped lane escalates or
   reports unusual selection, use the required monitored `just check-full` workflow and
   distinguish unrelated pre-existing failures from this regression rather than
   weakening `%final` assertions.

## Acceptance criteria

- Typing `%` in the ACE prompt opens a directive menu containing `%final` with the
  canonical description; `%f` and `%final` narrow to the canonical row.
- External editors receive the canonical `%final` name without snippet support and the
  existing single-selector and parenthesized snippet rows when snippet support is
  enabled, with clause-local UTF-16 edits.
- ACE, LSP, and the generic directive catalog agree that `%final` is public, while
  genuinely retired or hidden directives remain absent.
- Existing `%final:` and `%final(...)` configured selector completion, legality,
  ordering, documentation, and graceful failure behavior remain unchanged.
- Both repositories pass focused coverage and their required checks; no feature flag,
  memory file, generated instruction shim, documentation rewrite, or visual rebaseline
  is introduced for this completed public behavior.
