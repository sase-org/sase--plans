---
tier: tale
title: Recover sase-core CI after final completion drift
goal:
  The public final directive completion contract and its tests agree, and sase-core CI
  is green.
size: small
proposed_by: bbugyi200.athena.0ah
create_time: 2026-08-22 11:19:33
status: wip
---

# Recover sase-core CI after `%final` completion drift

## Objective

Restore the `sase-org/sase-core` `cargo test --workspace` GitHub Actions gate and
confirm that the dependent Release-plz workflow is no longer blocked. Preserve the
intended public `%final` editor completion behavior while keeping the core completion
contract and LSP regression tests aligned.

## Evidence and root cause

- `actstat --repo sase-org/sase-core --limit 5` reports the primary failures in CI runs
  1237, 1238, and 1244 at `cargo test --workspace`; the Release-plz failures are
  downstream waits on those red checks.
- Runs 1237 and 1238 fail two `sase_xprompt_lsp` tests because commit `eca4d68` exposed
  `%final` name/snippet completion while the tests still required it to be hidden.
- A later successful commit restored the hidden behavior, but test-only commit `82a5e4a`
  then changed three LSP expectations to require public `%final` completion without
  changing the core metadata. Run 1244 consequently reports a missing canonical `%final`
  row and missing `%final` snippets.
- Current `master` commit `6c4f8a2` removes `final` from the core hidden-completion list
  and replaces the stale core test with assertions for the public completion contract.
  This appears to be the correct narrowly scoped repair and was already present before
  this plan was authored, so it must be validated rather than duplicated.

## Implementation

1. Confirm the checkout is clean, still points at the intended current `master`, and
   that no newer commit has changed the failing tests or `%final` completion contract.
2. Run the three previously failing `sase_xprompt_lsp` tests, plus the core regression
   test for public `%final` name completion. Verify canonical-name, colon-snippet, and
   parenthesized-snippet rows use the clause-local replacement ranges asserted by the
   LSP tests.
3. Run the repository-required `just check` gate, which mirrors CI across formatting,
   clippy, and the entire workspace test suite, including `sase_core_py`.
4. If any related gate still fails, make the smallest correction in the shared core
   directive metadata or its directly coupled tests, then repeat the targeted tests and
   `just check`. Do not weaken or delete the new public-completion assertions.
5. Re-run `actstat` for `sase-org/sase-core` and inspect the current commit's CI and
   Release-plz conclusions. Distinguish an in-progress run from a settled failure and
   inspect any new failed job before declaring recovery.

## Acceptance criteria

- The three historical LSP regressions and the core `%final` completion regression test
  pass locally.
- `just check` passes from the `sase-core` repository root.
- The current GitHub Actions CI run settles successfully, and Release-plz is no longer
  failing because it is waiting on a red CI check.
- The repository has no unintended local modifications; if no additional correction was
  needed beyond `6c4f8a2`, leave the clean tree untouched and report that the fix landed
  concurrently.
