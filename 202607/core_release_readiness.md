---
tier: tale
title: Restore core release readiness
goal: Fix the gateway Clippy blocker without changing wire or snippet behavior, verify
  a release-plz wheel containing the shared snippet binding, and close only phase
  bead sase-8u.4.1.
bead: sase-8u.4.1
parent: sase/repos/plans/202607/finish_capitalized_snippet_aliases.md
create_time: 2026-07-23 09:58:05
status: done
---

- **PROMPT:** [202607/prompts/core_release_readiness.md](prompts/core_release_readiness.md)

# Restore core release readiness for `sase-8u.4.1`

## Goal

Make the linked `sase-core` repository release-ready without changing gateway wire behavior or the capitalized-snippet
implementation, then verify that release-plz publishes an installable `sase-core-rs` release containing
`compose_snippet_catalog`. Close only phase bead `sase-8u.4.1` after every criterion below passes. Do not close parent
bead `sase-8u.4` or epic `sase-8u`, and do not create beads.

## Verified starting state

- The phase design is the `Restore core release readiness` section of `202607/finish_capitalized_snippet_aliases.md` in
  the plans sidecar.
- Linked-core commit `f6f6a83111128cd27e3c85ec4ac84d2a367e12bb` is `origin/master`, so the shared two-pass composer,
  alias provenance, native catalog metadata propagation, PyO3 registration, and native LSP fallback are all ancestors of
  any subsequent release candidate.
- The composer in `crates/sase_core/src/xprompt_catalog.rs` resolves explicit references, creates non-colliding
  initial-capital aliases, records their source triggers, and resolves references again. Native catalog construction
  copies source metadata to generated aliases. The focused Rust tests cover authored-capital precedence, Unicode and
  unchanged prefixes, reference composition, metadata/count parity, and native fallback behavior.
- The binding in `crates/sase_core_py/src/lib.rs` returns plain `templates` and `alias_provenance` dictionaries and is
  registered as `compose_snippet_catalog`.
- release-plz PR `sase-org/sase-core#27` is an open `0.9.0` candidate based on `f6f6a83`. Its wheel build/import smoke
  succeeds, but Rust 1.97.1 CI fails Clippy with 14 `result_large_err` diagnostics because gateway `ApiError` is at
  least 136 bytes. No `v0.9.0` tag exists yet, and PyPI currently exposes only `sase-core-rs 0.8.0`.

Because release state can change while this plan is awaiting approval, refresh the linked checkout through the
repository-opening workflow and recheck `origin/master`, release-plz PR/tag state, and the package index before editing.
If upstream has already fixed the lint, audit that fix against this plan rather than duplicating it.

## Implementation

1. In `crates/sase_gateway/src/routes.rs`, reduce the size of `ApiError` by storing its `ApiErrorWire` payload behind a
   `Box`. Adapt every constructor and `IntoResponse` as required so Axum serializes the same wire object. Preserve every
   existing HTTP status, error code, message, target, details field, schema version, and JSON body. Do not add
   per-function Clippy suppressions.
2. Keep the snippet implementation unchanged unless a validation failure proves an actual regression. In particular,
   preserve explicit-trigger precedence, the two-pass reference resolution, generated alias provenance, copied native
   metadata, binding registration, and the Rust LSP fallback.
3. Format the core workspace. Run `cargo test -p sase_gateway`, then focused tests for:
   - core composer, collision/reference behavior, and native catalog metadata;
   - `compose_snippet_catalog_binding_returns_plain_dict_shape` in `sase_core_py`;
   - `snippet_cache_uses_rust_fallback_when_helper_unavailable` in `sase_xprompt_lsp`.
4. From the SASE repository, ensure the local development environment is current as required by its instructions, then
   run the complete `just rust-check` gate. This must pass formatting, workspace-wide Clippy with warnings denied, and
   all workspace tests. Fix and rerun focused and full checks until green.

## Release verification

1. Do not manually edit workspace/crate versions, path-dependency version pins, release changelogs, or tags. Let
   release-plz refresh or supersede its release candidate after the validated gateway fix reaches `origin/master`.
2. Confirm the eventual release tag and candidate commit both descend from `f6f6a83` and include the gateway fix, and
   confirm release CI is green. Use the first version release-plz actually publishes; do not assume the currently
   proposed `0.9.0` if it is replaced.
3. Wait for that exact `sase-core-rs` version to appear on the package index. In a fresh temporary virtual environment
   with no linked-source override, install the wheel from the package index and verify:
   - `import sase_core_rs` succeeds;
   - `compose_snippet_catalog` exists;
   - composing `{"foo": "foo bar baz"}` returns both `foo -> "foo bar baz"` and `Foo -> "Foo bar baz"` with `Foo -> foo`
     provenance.
4. If publication is pending or failed, keep `sase-8u.4.1` in progress and continue checking the release
   workflow/package index or report the exact external blocker. Never substitute a local source build or the old `0.8.0`
   wheel for published-release evidence.

## Completion

Recheck the linked-core diff and validation results, record concise implementation/release evidence in the bead notes,
and update only `sase-8u.4.1` to `closed`. Show `sase-8u.4.1`, `sase-8u.4`, and `sase-8u` afterward to prove the phase
is closed while both parents remain open.

## Acceptance criteria

- `ApiError` no longer triggers `result_large_err`; no scattered lint suppression is introduced; existing gateway
  response tests prove wire compatibility.
- Focused gateway/composer/PyO3/LSP tests and the full `just rust-check` gate pass.
- An exact published wheel whose release commit contains `f6f6a83` and the gateway fix installs without a linked
  checkout and exposes a working `compose_snippet_catalog` binding.
- Only `sase-8u.4.1` is closed, with validation and published-release evidence; parent `sase-8u.4` and epic `sase-8u`
  remain open.
