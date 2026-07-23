---
tier: tale
title: Integrate the published core floor into SASE
goal: Declare and validate the first published core wheel that fully satisfies SASE,
  synchronize dependency metadata, and close only sase-8u.4.2.
bead: sase-8u.4.2
parent: sase/repos/plans/202607/finish_capitalized_snippet_aliases.md
create_time: 2026-07-23 10:16:33
status: wip
---

- **PROMPT:** [202607/prompts/published_core_integration.md](prompts/published_core_integration.md)

# Plan: Integrate the published core floor into SASE

## Goal

Complete bead `sase-8u.4.2` by declaring the first actually published `sase-core-rs` release that contains core commit
`f6f6a83111128cd27e3c85ec4ac84d2a367e12bb` and exposes every Rust binding required by the current SASE source tree.
Synchronize the host dependency and lock metadata, preserve capitalized-snippet behavior across every host surface, pass
the Rust and Python gates, and close only `sase-8u.4.2`.

## Current state and constraints

- The SASE checkout is clean and already matches `origin/master` at `cf8832b7e`. The only commits after the
  capitalized-alias documentation commit `ec229ad32` are `58c6641a8` (agent-name registry extraction) and `cf8832b7e`
  (optional machine-identity configuration); their changed paths do not overlap the snippet catalog, editor helper,
  prompt-save, dependency, or snippet documentation paths.
- `pyproject.toml` currently declares the unavailable `sase-core-rs>=0.12.0,<0.13.0` window, while `uv.lock` still
  declares `>=0.5.0,<0.6.0` and resolves `0.5.0`.
- PyPI currently publishes through `sase-core-rs 0.8.0`. An isolated install of that exact wheel is missing 27 of the
  163 bindings statically required by SASE, including `compose_snippet_catalog`; it is not an acceptable floor.
- The linked core `master` contains both the composer commit `f6f6a83` and the gateway readiness fix `7d55028`, but its
  Cargo workspace remains at `0.8.0` until release-plz publishes the next release. Local source builds must not be used
  as evidence for the compatibility floor, and core Cargo versions must not be edited by this work.
- The parent beads `sase-8u.4` and `sase-8u` must remain open. Do not create any beads.

## Implementation

1. Reconfirm a clean, current host base before editing.
   - Fetch `origin/master`, fast-forward the local `master` only if necessary, and inspect every newly arrived commit
     after `ec229ad32`.
   - Recheck any new snippet catalog, editor-helper, prompt-save, dependency, or documentation changes against the
     shared composer. Record the audit in the bead notes; unrelated commits need only a concise no-overlap explanation.
   - If the worktree is no longer clean or a new overlapping change cannot be safely reconciled, stop rather than
     overwrite unrelated work.

2. Select a real published compatibility floor.
   - Query the package index for candidate `sase-core-rs` releases newer than `0.8.0`, beginning with the first
     published release whose release tag contains `f6f6a83`.
   - In a fresh isolated Python 3.12 environment with no linked checkout, editable install, or dependency override,
     install each candidate by exact version.
   - Run `tools/smoke_sase_core_rs_telemetry` and `tools/check_sase_core_rs_bindings` in that environment. Accept the
     first candidate that imports successfully, passes telemetry, and exposes all statically collected bindings,
     specifically `compose_snippet_catalog`.
   - Treat absence of such a published wheel as a hard external blocker: leave all dependency files unchanged and keep
     `sase-8u.4.2` in progress. Never lower the floor to `0.8.0`, substitute a local source build, or manually change
     release-plz-managed core versions.

3. Synchronize host dependency metadata to the passing release.
   - Change the `sase-core-rs` requirement in `pyproject.toml` to use the exact passing release as the inclusive minimum
     and the matching next-minor version as the exclusive upper bound.
   - Run the normal `uv lock` workflow so `uv.lock` regenerates from `pyproject.toml`. Verify the SASE `requires-dist`
     entry and resolved `sase-core-rs` package use the same window and exact minimum, with current hashes/files from the
     package index. Ensure the stale `0.5.0` lock entry and unavailable `0.12.0` floor are both gone.
   - Keep the diff limited to `pyproject.toml` and mechanically refreshed `uv.lock` metadata unless validation exposes a
     genuine integration defect.

4. Validate behavior and release compatibility.
   - Run `just install` first so the workspace dependencies and linked Rust extension are current.
   - Run the focused host tests covering the validated facade, ACE catalog and pending-save recomposition, editor-helper
     catalog metadata/counts, and snippet-reference composition:

     ```bash
     .venv/bin/pytest -q \
       tests/test_core_snippet_catalog_facade.py \
       tests/ace/tui/test_prompt_catalog.py \
       tests/ace/tui/actions/test_prompt_save_xprompt.py \
       tests/test_editor_helpers.py \
       tests/test_xprompt_snippet_bridge.py
     ```

   - Repeat the exact-minimum isolated telemetry and complete binding smoke after locking, deriving the version from
     `pyproject.toml` with `tools/smoke_sase_core_rs_telemetry --print-minimum`.
   - Run `just rust-check`, then the repository-mandated `just check`. Fix only failures caused by this integration and
     rerun affected focused and full gates until all pass.
   - Confirm the focused assertions retain the intended contract: generated capitalized aliases are runtime-only;
     explicit capitalized triggers win; pending saves recompose from explicit state off the UI thread; editor metadata
     and counts include both spellings; and snippet references resolve through lowercase and generated-capitalized
     triggers.

5. Record completion and close only the claimed phase.
   - Inspect the final diff and rerun the exact-minimum smoke once more from a clean isolated environment.
   - Update `sase-8u.4.2` notes with the selected published version, release/tag provenance, latest-base audit,
     exact-minimum binding count, focused test result, `just rust-check` result, and `just check` result.
   - Set only `sase-8u.4.2` to `closed`, then show it and its parent `sase-8u.4` to verify the phase is closed while the
     parent epic remains open. Do not close `sase-8u.4` or `sase-8u`, edit the canonical epic plan, run post-close epic
     cleanup, or create another bead.

## Acceptance criteria

- The declared minimum is an exact, non-yanked, published wheel whose release contains `f6f6a83`; an isolated install
  passes telemetry and exposes all 163 bindings required by the audited host source, including
  `compose_snippet_catalog`.
- `pyproject.toml` and `uv.lock` agree on the selected minimum and matching next-minor upper bound, and neither stale
  `0.5.0` metadata nor the unavailable `0.12.0` floor remains.
- The latest base is audited with no missed snippet-related overlap, the focused host suite passes, and both
  `just rust-check` and `just check` are green.
- `sase-8u.4.2` contains concrete validation notes and is closed. `sase-8u.4` and `sase-8u` remain open, and no new bead
  exists.
