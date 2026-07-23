---
tier: epic
title: Finish and land capitalized snippet aliases
goal: 'Publish and consume a core wheel that contains the shared capitalized-snippet
  composer, restore every promised validation gate, integrate the latest base, and
  close sase-8u with clean post-close metadata.

  '
phases:
- id: core-release-readiness
  title: Restore core release readiness
  depends_on: []
  size: medium
  description: '''Restore core release readiness'' section: fix the current gateway
    Clippy failure, preserve the snippet implementation, and verify the release-plz
    release containing the new binding.'
- id: published-core-integration
  title: Integrate the published core floor into SASE
  depends_on:
  - core-release-readiness
  size: medium
  description: '''Integrate the published core floor into SASE'' section: audit the
    latest base, set a real exact-minimum wheel floor, refresh lock metadata, and
    pass all Rust and host gates.'
- id: land-epic
  title: Close and clean up the epic
  depends_on:
  - published-core-integration
  size: small
  description: '''Close and clean up the epic'' section: close sase-8u, run post-close
    symvision cleanup, and mark the canonical epic plan done.'
parent_bead: sase-8u
parent: sase/repos/plans/202607/capitalized_snippet_aliases.md
create_time: 2026-07-23 09:53:04
status: wip
---

# Plan: Finish and land capitalized snippet aliases

## Context

Epic bead `sase-8u` implements generated initial-capital aliases for every effective SASE snippet across ACE, the editor
helper, and the native Rust LSP fallback. Its three child beads are closed, but the land audit found two
release-blocking gaps that make the epic incomplete:

- `sase-core` commit `f6f6a83111128cd27e3c85ec4ac84d2a367e12bb` contains the shared two-pass composer, alias provenance,
  native metadata propagation, PyO3 binding, and LSP fallback coverage. The focused Rust tests pass, and the host
  commits `6e6b8d85c3c4314d84ba5167c22a955bacf623fe` and `ec229ad32c315ccd7b0754dd1140fe9cf46610eb` correctly consume
  and document that contract. The phase-note hashes for host phases 2 and 3 are stale pre-integration hashes; the
  bead-tagged commits above are the landed commits.
- The feature is not available at SASE's published compatibility floor. PyPI still serves `sase-core-rs 0.8.0`, which
  lacks 27 of the 163 Rust bindings currently required by SASE, including `compose_snippet_catalog`. Meanwhile,
  `pyproject.toml` declares the unavailable range `sase-core-rs>=0.12.0,<0.13.0`, and `uv.lock` still records the
  obsolete `>=0.5.0,<0.6.0` / `0.5.0` metadata. Local development hides this because `just install` deliberately
  overrides the published constraint and builds the linked core source.
- `just rust-check` is not currently green: Rust 1.97 Clippy reports 14 `result_large_err` failures in
  `crates/sase_gateway/src/routes.rs` because `ApiError` stores a large `ApiErrorWire` inline. This is outside the
  snippet composer but blocks the epic's promised full Rust validation.

The focused host suite passed 89 tests, the focused core composer/binding/LSP tests passed, the local linked-core
binding inventory exposes all 163 required bindings, and the full host `just check` passed. The integration audit
covered every host commit after the first epic core commit, including the later AXE, agent-name, plugin-browser, and
multi-prompt allocator refactors. None duplicates or conflicts with snippet composition. Before implementation,
fast-forward the clean host checkout to the latest `origin/master` and repeat the audit for any still-newer commits.

Do not close `sase-8u` until an exact, actually published core wheel passes the binding inventory and the full Rust and
host gates are green. Follow the `sase-core` release policy: release-plz owns Cargo/workspace versions; do not manually
edit them or substitute an unpublished source build for the published minimum smoke.

## Phase 1: Restore core release readiness

**ID:** `core-release-readiness`

**Depends on:** none

1. Use the repository-opening workflow for the linked `sase-core` checkout. Confirm that `f6f6a83` remains on
   `origin/master` and is an ancestor of the release candidate that will publish the new binding. Re-read the current
   composer, binding registration, native catalog metadata propagation, and LSP fallback test before changing anything.
2. Fix the current `result_large_err` failures without changing gateway wire responses. Prefer reducing `ApiError`
   itself, for example by boxing its `ApiErrorWire` payload and adapting constructors/`IntoResponse`, over suppressing
   the lint on every returning function. Preserve status codes, JSON bodies, and existing gateway tests.
3. Run focused gateway tests, the focused snippet composer/PyO3/LSP tests, and the full `just rust-check` gate from the
   SASE repository. Fix and rerun until formatting, Clippy, and all workspace tests pass.
4. Verify release-plz's published release rather than editing Cargo versions. The existing remote `v0.9.0` release
   candidate contains `f6f6a83`, but use the first version that is actually visible from the package index and whose
   wheel installs successfully. If publication is still pending, leave the epic open and do not weaken the host
   dependency to `0.8.0`.

## Phase 2: Integrate the published core floor into SASE

**ID:** `published-core-integration`

**Depends on:** `core-release-readiness`

1. Fast-forward the clean SASE `master` checkout to `origin/master`. Inspect every new base commit since `ec229ad32`;
   integrate any new snippet catalog, editor helper, prompt-save, dependency, or documentation path with the shared
   composer. Record why unrelated refactors need no changes.
2. In an isolated environment with no linked-source override, install each candidate core release starting with the
   first published release containing `f6f6a83`. Run `tools/check_sase_core_rs_bindings` against that exact wheel. The
   selected minimum must expose all statically collected bindings, specifically including `compose_snippet_catalog`; a
   local source build is not evidence for this step.
3. Change the `sase-core-rs` range in `pyproject.toml` to start at that exact passing published version and use the
   matching next-minor upper bound. Refresh `uv.lock` with the project dependency workflow so both `requires-dist` and
   the resolved `sase-core-rs` package agree with `pyproject.toml`. Do not retain the stale `0.5.0` lock entry or the
   unavailable `0.12.0` floor.
4. Run `just install`, the focused host snippet facade/catalog/live-save/editor helper tests, the exact-minimum isolated
   binding smoke, `just rust-check`, and the mandatory `just check`. Fix and rerun until all gates pass. Confirm
   generated aliases remain runtime-only, explicit capitalized triggers still win, pending saves recompose from explicit
   state off the UI thread, metadata and counts agree, and both spellings work in snippet references.

## Phase 3: Close and clean up the epic

**ID:** `land-epic`

**Depends on:** `published-core-integration`

1. Re-run `sase bead show sase-8u` and show each child. Confirm the phase notes, landed commits, published-wheel smoke,
   latest-base integration audit, and all validation results before changing epic state.
2. Close the epic with `sase bead close sase-8u`.
3. Only after closure, review the SASE `symvision.md` long-term memory through the required memory-read workflow, then
   run `just symvision`. Remove any expired `sase-8u` epic-symbol entries and genuinely unused code it reports; rerun
   symvision until clean. If host source files change, rerun the proportionate focused tests and mandatory `just check`.
4. Use the repository-opening workflow for the plans sidecar and change only the canonical epic plan frontmatter at
   `202607/capitalized_snippet_aliases.md` from `status: wip` to `status: done`. Validate the plan document and verify
   the epic remains closed.

## Acceptance criteria

- `foo: "foo bar baz"` yields exactly `foo -> "foo bar baz"` and `Foo -> "Foo bar baz"` across ACE, editor-helper
  completion, normal LSP completion, and native Rust fallback, while explicit `Foo` definitions, metadata, reference
  composition, and live-save behavior retain the authored precedence and threading guarantees.
- The linked core passes focused tests and full `just rust-check` without Clippy suppressions scattered across gateway
  result functions.
- The exact declared minimum published `sase-core-rs` wheel installs without a linked checkout and passes the complete
  static binding inventory, including `compose_snippet_catalog`.
- `pyproject.toml` and `uv.lock` agree on a real published compatibility window; neither the unavailable `0.12.0` floor
  nor stale `0.5.0` lock state remains.
- The latest base-branch commits have been audited and any relevant overlap is integrated. The focused suites,
  `just rust-check`, and `just check` all pass.
- `sase-8u` and every child bead are closed, post-close `just symvision` is clean, and the canonical epic plan has
  `status: done`.
