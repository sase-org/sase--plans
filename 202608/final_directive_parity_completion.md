---
tier: tale
title: Complete final directive LSP exposure and ACE/LSP parity
goal:
  The public final directive is positively covered and behaviorally identical across ACE
  and LSP completion.
size: medium
proposed_by: bbugyi200.athena.sase-s0.land
bead: sase-s0
create_time: 2026-08-21 22:26:47
status: done
---

- **PARENT:** [202608/final_directive_completion.md](final_directive_completion.md)
- **BEAD:**
  [sase-s0](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s0/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-s0.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-s0.land.md)
- **COMMITS:**
  - [6ee4e1d](https://github.com/sase-org/sase/commit/6ee4e1d3d26c35d3641de2e267f9297d94b236e1)
    — test(completion): cover ACE and LSP %final completion parity

# Complete `%final` LSP exposure and ACE/LSP parity verification

## Context

Epic `sase-s0` implemented the shared finalizer catalog, selector-aware Rust completion,
ACE inventory loading/rendering, LSP catalog caching/rendering, public directive-name
exposure, documentation, and normal/narrow PNG coverage. Its landing audit found that
the final integration phase did not update the existing LSP snippet tests or the
repository's ACE-versus-LSP parity harness.

The production Rust contract now correctly advertises `%final`, so these two current
tests fail because they still assert the former hidden behavior:

- `sase-core/crates/sase_xprompt_lsp/src/server.rs::snippet_clients_receive_identity_and_clan_forms`
- `sase-core/crates/sase_xprompt_lsp/src/server.rs::directive_snippet_for_alt_uses_brace_shorthand`

The broader parity suite at `tests/test_xprompt_directive_completion_parity.py` also
still excludes `final` from public names, does not serve `finalizer-catalog` from its
fake helper, does not give ACE a warm finalizer inventory, and has none of the finalizer
argument cases required by the approved epic plan. The committed ACE PNG fixtures were
visually reviewed during landing and already cover a legible required/default/optional/
remove/clear grid at normal and narrow widths; do not rebaseline them unless a real
behavior correction changes their output.

## Implementation

1. In the linked `sase-core` repository, update the stale LSP tests to assert the now
   public contract. For both `%f` and `%final`, require the canonical `%final` name row
   and, when snippet support is enabled, the `%final:${1:instance}` and
   `%final(${1:instance}, ${2:instance})` snippet forms with clause-local text edits.
   Replace the old empty-snippet assertions with positive assertions while retaining the
   negative coverage for genuinely retired directives and aliases. Do not restore a
   hidden-name guard or weaken production visibility to satisfy the tests.

2. Complete `tests/test_xprompt_directive_completion_parity.py` in the main repository:
   - Include `%final` in the public directive-name comparison.
   - Add a deterministic structured finalizer inventory containing required, default,
     and optional instances with provider, dependency, retry, and documentation fields.
   - Pass that same inventory to the ACE builder with `finalizers_state="warm"`, and
     teach the surface-row adapter to preserve finalizer operation/policy/provider
     metadata in a form comparable with standard LSP fields.
   - Extend the fake helper bridge with a schema-v1 `finalizer-catalog` response, and
     make the fixture configurable enough to cover catalogs with and without required
     instances plus helper failure without consulting host configuration.
   - Add ACE/LSP assertions for public name rows; configured add ordering; `!` removal
     labels and insertion while omitting required instances; `none` suppression when a
     required instance exists and availability when clearing is legal; provider/policy
     detail and Markdown documentation; repeated directives; parenthesized active-
     clause replacement; and UTF-16 replacement ranges next to non-ASCII text.
   - Cover failure degradation explicitly: the LSP returns no invented dynamic rows,
     while ACE retains its non-selectable unavailable placeholder and manual selectors
     remain typable. Compare the shared selectable behavior rather than erasing this
     deliberate presentation difference.

3. If the completed parity cases reveal a production mismatch, fix the smallest shared
   Rust contract or surface adapter responsible. Keep Python as the trusted configured
   catalog authority and Rust as the owner of filtering, legality, ordering,
   replacement, and LSP conversion; do not duplicate those rules in the parity test.

## Verification

1. In `sase-core`, run the two previously failing tests, the finalizer completion/LSP
   test filters, then the repository's `just check`.
2. In the main repository, run `just install` before tests. Run
   `tests/test_xprompt_directive_completion_parity.py`, the finalizer catalog/helper/ACE
   suites, and the two finalizer PNG snapshot nodes at both widths.
3. Run `just check`. Because this is the completion of an epic combined tree spanning
   Rust core, LSP, and ACE, finish with `just check-full` through `/sase_monitor` using
   `TESTING`/`TESTED` and a follow-up that diagnoses any failure. Run the full visual
   suite as required by the original acceptance criteria; distinguish already-filed
   unrelated baseline drift from any `%final` regression rather than accepting new
   goldens blindly.

## Acceptance criteria

- `%final` name and snippet completion is positively tested for snippet-capable LSP
  clients, while ordinary canonical name completion remains available without snippet
  support.
- The real ACE/LSP parity harness exercises the same deterministic finalizer catalog and
  agrees on selector ordering, legality, insertions, documentation, provider/policy
  metadata, and UTF-16 clause-local edits.
- Required selectors are never offered for removal, `none` is never offered when clear
  is invalid, and helper failure invents no dynamic LSP rows or blocks manual entry.
- Both repositories pass their relevant targeted tests and required checks, with any
  unrelated pre-existing repository-wide failure routed to its existing bead rather than
  hidden or folded into this tale.
