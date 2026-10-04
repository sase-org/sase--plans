---
tier: tale
title: Fix sase-core CI clippy failures in sase_xprompt_lsp
goal:
  The sase-core CI workflow is green again on master (and on the pending release-plz PR)
  because the three clippy -D warnings errors introduced by f50782f in sase_xprompt_lsp
  are fixed without changing behavior.
size: small
proposed_by: bbugyi200.athena.0w3
create_time: 2026-10-04 05:35:37
status: wip
---

# Plan: Fix sase-core CI clippy failures in `sase_xprompt_lsp`

## Scope

Get the sase-core `CI` workflow green again on `master` (and on the pending release-plz
PR `chore: release v0.36.5`, which is blocked by the same failure). All work happens in
the **linked `sase-core` repo**: open it with `sase repo open sase-core -r "<reason>"`
and use the printed path for every read and write. Read that repo's `AGENTS.md` before
editing.

## Diagnosis (already done — do not redo)

`actstat --repo sase-org/sase-core` shows the last three `master` commits red:

| Commit    | Failing step                                         | Status now                                                                                                                                                              |
| --------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `f50782f` | `cargo clippy --workspace --all-targets -D warnings` | still failing (root cause, see below)                                                                                                                                   |
| `d7f2dbf` | `cargo fmt --all -- --check`                         | **already fixed** — `3d406d4` reformatted `agent_session_manifest.rs`, `sase_core_py/src/prelude.rs`, `sase_core_py/src/vcs/{mod,tests}.rs`; `fmt-check` passes on HEAD |
| `3d406d4` | clippy (same 3 errors as `f50782f`)                  | still failing                                                                                                                                                           |

Root cause: commit `f50782f` ("feat(core): add xprompt completion-spacer planner,
binding, and LSP paren handling") added `crates/sase_xprompt_lsp/src/server/spacer.rs`
and extended `crates/sase_xprompt_lsp/src/server/documents.rs` with code that trips
three clippy lints. `master` is unprotected, so it landed red. It is **not** toolchain
drift: CI runs rustc 1.99.0, but local stable 1.98.1 reproduces exactly the same three
errors.

Exhaustive error set (verified locally with `--keep-going`, and with the three lints
allowed every remaining `sase_xprompt_lsp` target — both bins and all integration tests
— is clippy-clean; every other workspace crate is already clean):

1. `clippy::if_same_then_else` — `crates/sase_xprompt_lsp/src/server/documents.rs`
   (~line 197): the confirmed-spacer fold has
   `if A { confirmed_spacer = None; } else if B { confirmed_spacer = None; }`.
2. `clippy::match_like_matches_macro` — `crates/sase_xprompt_lsp/src/server/spacer.rs`
   (~line 151), tail of `is_normalized_after_deletion`:
   `match text.as_bytes().get(reference_byte) { Some(b'(') => true, _ => false }`.
3. `clippy::too_many_arguments` (8/7) — `crates/sase_xprompt_lsp/src/server/spacer.rs`
   (~line 311),
   `transition_argument_completion(&self, uri, document, confirmed, spacer_byte, position, trigger_kind, trigger_character)`.

The other CI gates were verified green locally on HEAD: `fmt-check`, `features`
(workspace-hack drift), the full hermetic `test` suite (CI never reached it for these
three commits, but it passes), `script-test`, and the `wheel-smoke` job passed on CI.

## Changes

All three edits are behavior-preserving.

### 1. `crates/sase_xprompt_lsp/src/server/documents.rs` — merge identical branches

Replace the `if … else if …` pair inside
`if let Some(confirmed) = confirmed_spacer.clone() { … }` with one condition:

```rust
if let Some(confirmed) = confirmed_spacer.clone() {
    if super::spacer::is_normalized_after_deletion(&text, &confirmed)
        || super::spacer::owned_spacer_byte(&text, &confirmed).is_none()
    {
        confirmed_spacer = None;
    }
}
```

`||` short-circuits in the same order as the old `else if`, so evaluation is unchanged.
Keep the existing comment block above it (it already explains both clearing cases). The
workspace is edition 2021, so clippy's `collapsible_if` does not ask to fold this into a
let-chain.

### 2. `crates/sase_xprompt_lsp/src/server/spacer.rs` — `is_normalized_after_deletion` tail

Replace the two-arm `match` with a direct comparison (the same idiom this file already
uses elsewhere):

```rust
text.as_bytes().get(reference_byte) == Some(&b'(')
```

Keep the preceding three-line comment.

### 3. `crates/sase_xprompt_lsp/src/server/spacer.rs` — `transition_argument_completion`

Add `#[allow(clippy::too_many_arguments)]` directly above
`pub(super) async fn transition_argument_completion(` (after its doc comment). This
matches the established convention in this repo (17+ existing uses in `sase_core` and
`sase_core_py`, e.g. `crates/sase_core/src/agent_hold.rs`). Do not restructure the
signature or its single caller in `server/mod.rs`.

## Verification (run in the sase-core checkout)

1. `just fmt` — rustfmt (`max_width = 80`) may rewrap the new `||` condition.
2. `just clippy` — must exit 0 with no `error:` lines. (A pre-existing, non-fatal cargo
   manifest _warning_ that `src/main.rs` backs two `[[bin]]` targets is expected and out
   of scope.)
3. `just test -p sase_xprompt_lsp` — the spacer unit tests (`server/tests/spacer.rs`)
   and `tests/jsonrpc_stdio_spacer_paren.rs` must still pass.
4. `sase tool run check` — the full gate, identical to CI (fmt-check, features, clippy,
   all tests, script tests). It takes ~5 minutes; give it a tool timeout of at least 10
   minutes. A test that fails under the full gate but passes alone is likely a known
   load flake — check `sase bead list -T flake` before treating it as yours, and never
   weaken an assertion.

## Commit

One Conventional Commit in sase-core, e.g.
`fix(xprompt_lsp): clear clippy -D warnings errors from spacer paren handling`.
Completion is host-owned: declare the sase-core repo obligation as a `commit` via the
final declaration; do not commit or push by hand. Do not touch any `version` field,
path-dependency version pin, or `CHANGELOG.md` (release-plz owns them). No sase-side
change is needed: no binding, wire schema, or `sase-core-revision.txt` pin is involved.

## Non-goals

- The pre-existing `src/main.rs` multiple-build-targets manifest warning (not a failure;
  introduced earlier by `be86aa9f`, whose CI passed).
- The already-fixed `d7f2dbf` fmt failure.
- Process changes to stop red commits landing on unprotected `master`.
