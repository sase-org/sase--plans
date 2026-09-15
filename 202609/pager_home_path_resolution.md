---
tier: tale
title: Fix pager home paths entering repository search
goal:
  Make plain home-directory links such as ~/.ssh/config resolve correctly from
  document-owned pager views without searching repositories for a literal tilde
  directory.
size: small
proposed_by: bbugyi200.athena.0l2
create_time: 2026-09-15 07:34:17
status: wip
---

# Fix pager home paths entering repository search

## Outcome and scope

Following `~/.ssh/config` from a file-backed pager document opens the existing local
file. Copy and edit use the same resolved path, including line/column locations. A
missing home path produces an ordinary missing-path result.

This is a small tale: one coding agent can make a localized Rust validation fix and add
regression coverage across the existing binding and pager adapters. The root cause has
been reproduced. No separate implementation phases are needed.

## Diagnosis and evidence

The supplied screenshot, `.sase/artifacts/pool/7974d27d04f8-file-ref.png`, shows the
tailnet reference note in the pager with this notification:

> bounded repository search budget was exhausted

The screenshot's original reference was recovered from local staging metadata and read
with `sase artifact read`: `file:/home/bryan/tmp/screenshots/20260915_070042.png`. The
tailnet note was read through `sase memory read tailnet.md`; its SSH configuration
instructions are context, not files requiring edits.

Read-only checks against the installed Rust binding confirmed that the home config
exists and the scanner produces the correct `file_path` target `~/.ssh/config`.
Resolution varies with document ownership:

| Input and context                           | Observed result                       |
| ------------------------------------------- | ------------------------------------- |
| Home path, no document owner                | Opens successfully                    |
| Absolute spelling, with or without an owner | Opens successfully                    |
| Home path, document owner present           | Search-budget error; marked retryable |
| Copy home path, document owner present      | Copies unresolved `~/.ssh/config`     |

The call chain is:

1. `src/sase/pager/adapters.py:path_section` attaches document ownership.
2. `src/sase/pager/_resolve_file_paths.py` attempts `_owned_file_path_resolution` before
   `search_existing_path`, for both follow and copy.
3. `src/sase/pager/source_resolve.py:lookup_owned_source_path` invokes the Rust
   `resolve_document_source_target` operation through
   `src/sase/artifact_ref_operations.py`.
4. In the `sase-core` repository,
   `crates/sase_core/src/artifact_ref/repository_resolution.rs:normalize_payload`
   accepts `~/.ssh/config` as a relative source path. Rust path components treat `~`
   literally; this operation does not perform home expansion.
5. The core probes `<checkout>/~/.ssh/config`, then recursively searches eligible
   repositories for that suffix. The shared 20,000-entry budget is exhausted in the
   reproduced context. A smaller inventory can instead report missing; a literal
   matching directory could even produce a wrong target.
6. This returned failure is terminal, so the pager never reaches its existing
   `Path.expanduser()` handling in `_resolve_path_search.py`.

Absolute source paths already fail core validation with
`ValueError: validation: source path must be relative`. The adapter handles that
exception by returning `None`, allowing ordinary filesystem resolution. A read-only
simulation of the same validation rejection for home-prefixed paths made follow, edit
targeting, and copy succeed. It also made a missing home path return a non-retryable
"not found (searched 1 locations)" diagnostic. This simulated behavior is evidence for
the proposed approach, not a completed implementation.

## Implementation

### 1. Correct the shared source-path input contract

Open `sase-core` with `/sase_repo` using
`sase repo open sase-core -r "Implement approved pager home-path validation fix"`; use
only the returned checkout path and read its `AGENTS.md` before editing.

In `crates/sase_core/src/artifact_ref/repository_resolution.rs`, reject raw `~` and
paths starting with `~/` as non-repository-relative input at the existing
`normalize_payload` boundary. Return an `ArtifactRefError` of kind `validation`,
consistent with existing absolute-path rejection, before owner filtering, candidate
probing, or suffix enumeration. State in the function documentation that home paths
belong to the caller's filesystem resolver.

Keep this rule local to document source resolution. The shared relative-payload
validator also serves typed references with different contracts. Apply the check to the
authored prefix so explicitly relative literal paths such as `./~/config` and nested
names such as `docs/~/config` retain their current meaning. The required scope is
current-user `~`/`~/` syntax; new named-user syntax is not part of this change.

Use the existing PyO3 error mapping and Python adapter fallback. No new wire fields,
schema version, or alternate Python resolver are needed. Clarify the docstrings in
`src/sase/artifact_ref_operations.py` and `src/sase/pager/source_resolve.py` so future
callers understand the input contract and why these validation errors hand control to
filesystem resolution.

### 2. Add regression coverage at the boundaries

In `sase-core`:

- Extend the tests in `repository_resolution.rs` to reject `~` and `~/.ssh/config` with
  validation errors even when the repository contains a literal `~/.ssh/config` decoy.
  Use the existing internal budget seam with a zero suffix budget to show home paths
  never reach suffix search. Cover a missing home target as the same lexical rejection.
- Cover an explicitly relative literal-tilde path and a normal source path as controls.
  Keep existing traversal, owner-policy denial, ambiguity, stale checkout, and revision
  tests passing.
- Add a PyO3 regression in `crates/sase_core_py/src/lib.rs` confirming the public
  `artifact_ref_resolve_document_source_target` binding raises `ValueError` for
  home-prefixed input, just as it does for absolute input.

In `sase`:

- Add adapter coverage in `tests/artifact_refs/test_document_source_resolution.py` for
  that real binding validation contract.
- Extend `tests/pager/test_resolve_paths.py` and/or `tests/pager/test_copy_owned.py`
  with a temporary home fixture and an owned document context. Inject fixture home
  expansion rather than depending on the developer's SSH configuration. Keep the real
  Rust source resolver in this integration path; only substitute fixture inventory and
  filesystem context.
- Verify existing and missing home paths; copy, scroll, and edit location metadata;
  absolute-path parity; and a repository literal-tilde decoy. An existing home directory
  should retain the pager's directory landing behavior.
- Add one Textual pilot regression using the rendered-link helpers: display a
  file-backed note containing a backticked `~/.ssh/config`, follow its generated hint,
  verify the fixture file content and navigation history, then return to the original
  document. The test must exercise attached document ownership, which is the missing
  condition in the failure. PNG golden updates are unnecessary for this behavioral fix.

### 3. Preserve the existing resolution rules

Returned owner-scoped outcomes for repository-relative links remain terminal. Do not
convert denied, ambiguous, unavailable-revision, missing-checkout, or temporary-error
results into generic filesystem retries. Retain the existing decoy protection in
`tests/pager/test_copy_owned.py` and the rendered-link failure tests. The 20,000-entry
budget remains useful for actual repository-relative suffix searches.

Plain pager paths continue to use their existing filesystem semantics. Typed `file:`
references retain their configured root filters. This fix requires no changes to SSH
files, chezmoi configuration, memory notes, artifact root settings, keymaps, or feature
flags. Resolution stays in the existing background worker; do not add filesystem work to
rendering or key handlers.

## Verification and completion

1. Show that the new regression fails against the old binding/core behavior.
2. Run the `sase-core` repository's `just check` from its opened checkout. Its required
   script includes formatting, clippy, and the whole workspace's tests, including PyO3;
   `cargo test -p sase_core` alone is insufficient.
3. Build/install the edited core into the current SASE workspace's virtualenv with
   `just --set sase_core_dir <opened-core-path> rust-install`. Ensure the pager
   integration tests load this rebuilt extension, rather than a previously installed
   wheel. Leave release version management to the existing host flow.
4. Run the focused adapter and pager regression suites, including the existing owner
   failure/decoy tests. Read `lint_and_test.md` and run SASE's required `just check`
   after the Python/test changes. Use `/sase_monitor` for checks or builds that need a
   long-running handoff; follow each repository's instructions.
5. Confirm the fixture note opens its home config through a rendered hint, copy and edit
   target the same file, line/column metadata survives, and a missing home file yields a
   normal missing-path diagnostic without a repository-budget error. Confirm owned
   relative-link denials still cannot select cwd decoys.

Completion requires the actual Rust change and rebuilt-binding regressions to pass.
Report both repositories' changes and verification through the normal host-owned
completion flow. This planning turn only authors and submits this plan; implementation
starts after its approval handoff.
