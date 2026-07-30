---
tier: tale
title: Land editor artifact-reference parity with the correct Rust dependency floor
goal:
  Published SASE installs require the first sase-core-rs release that supports ACE's current artifact-reference menu
  API, and sase-b3.10 closes only after that integration is verified.
bead: sase-b3.10
create_time: 2026-07-30 08:10:11
status: done
---

- **PROMPT:**
  [202607/prompts/editor_artifact_ref_parity_integration.md](prompts/editor_artifact_ref_parity_integration.md)
- **PARENT:**
  [202607/editor_artifact_ref_parity.md](https://github.com/sase-org/sase--plans/blob/main/202607/editor_artifact_ref_parity.md)
- **BEAD:** [sase-b3.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-b3/sase-b3.10.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-b3.10.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-b3.10.land.md#member-code)
  - [bbugyi200.athena.sase-b3.10.land--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-b3.10.land.md#member-plan)
- **COMMITS:**
  - [02de1fd](https://github.com/sase-org/sase/commit/02de1fd2aceb105419a188fa9cd1d46c53782d7c) — build(deps): require
    sase-core-rs 0.12.19

# Plan: Land the editor artifact-reference parity epic with the correct Rust dependency floor

## Context

Epic `sase-b3.10` added fuzzy, titled, cached artifact-reference payload completion to the xprompt LSP and released it
in `sase-core` v0.12.19. All four child beads are closed, their bead-tagged commits are present on the relevant default
branches, and a fresh `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
`cargo test --workspace` run passes in `sase-core`.

One change landed after the epic began and intersects this work:

- `sase-core` commit `4e61ad0` (`sase-b4.1`) added `AtReferenceMenuOptionsWire`, the `include_files` option, and an
  `options` keyword argument to the Python `at_reference_menu` binding.
- `sase` commit `9ba92b0` (`sase-b4.2`) integrated that behavior into ACE and now always calls the binding with
  `options={"include_files": ...}`.
- The Python-visible API exists in `sase-core-rs` 0.12.19 but not 0.12.18. `pyproject.toml` still allows
  `sase-core-rs>=0.12.18,<0.13.0`, and `uv.lock` still selects 0.12.18. A linked-source development install builds
  0.12.19 and masks the published-package incompatibility.

The plan linked from `sase bead show sase-b3.10` is `plans:202607/editor_artifact_ref_parity.md`. Its frontmatter
already says `status: done`, but that status was set before the land agent found this dependency-floor gap. Do not treat
the premature status as evidence that landing is complete.

## Goal

Make the published SASE dependency contract require the first `sase-core-rs` release that supports the API ACE now uses,
verify the editor/ACE integration, and then close `sase-b3.10` through the normal descendant-guarded path.

## Phase 1: Ratchet and verify the dependency contract

1. Update the `sase-core-rs` lower bound in `pyproject.toml` from 0.12.18 to 0.12.19, preserving the `<0.13.0` upper
   bound.
2. Regenerate `uv.lock` with the normal lock command so it selects published `sase-core-rs` 0.12.19 and records the
   release artifacts and hashes. Do not hand-edit generated lock metadata.
3. Confirm the resulting diff is limited to the intended dependency floor and lockfile refresh.
4. Run `just install` before repository checks, as required by this repository.
5. Verify the minimum is published with
   `.venv/bin/python tools/validate_sase_core_rs_version --pyproject pyproject.toml --published-minimum`.
6. Run the artifact-reference integration tests, including `tests/ace/tui/widgets/test_artifact_ref_completion.py` and
   `tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py`. The old 0.12.18 binding must no longer be a valid
   dependency resolution, and the current caller's `options/include_files` path must pass.
7. Run `just check`. If it reports an unrelated pre-existing validation failure, record it precisely and still run the
   relevant format, lint, unit, and artifact-reference checks individually; do not silently broaden this epic to
   unrelated plans. Any failure caused by these changes or by the dependency/API mismatch is in scope and must be fixed
   before landing.

## Phase 2: Land the epic

This is the final phase and must run only after Phase 1 is green.

1. Re-read `sase bead show sase-b3.10` and its four children. Confirm all descendants remain closed.
2. Close the epic without force:

   ```bash
   sase bead close sase-b3.10 --note "<dependency floor raised to 0.12.19; child implementation, later b4.1/b4.2 integration, source, commits, and verification checks confirmed>"
   ```

   If the close is rejected, resolve or reopen the named unfinished work. Never use `--force` merely to bypass the
   descendant guard.

3. After the close, run `just symvision`. Remove only stale `sase-b3.10` epic-symbol whitelist entries or unused code it
   reports, then rerun Symvision until clean. If this creates source changes, rerun `just check`.
4. Open the plans sidecar through `sase repo open plans -r "<reason>"`, then ensure
   `plans:202607/editor_artifact_ref_parity.md` has `status: done` in frontmatter. It is already `done` in the audited
   state, so preserve that value unless the durable plan changed during handoff; the important invariant is that it is
   confirmed after the successful bead close.
5. Finish with clean worktrees for every repository touched and report the close resolution, dependency versions,
   verification commands, and any explicitly unrelated failures.

## Non-goals

- Do not close the parent epic `sase-b3`; it has its own land agent.
- Do not modify `sase-core` or `sase-nvim` unless a new verification failure proves their current default branches are
  broken.
- Do not force-close any bead.
- Do not edit unrelated plan-link failures or unrelated Symvision findings.
