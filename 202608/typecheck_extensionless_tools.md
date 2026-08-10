---
tier: tale
title: Type-check extensionless Python tools
goal:
  Every extensionless Python tool is covered by the normal mypy gate, and direct typing
  defects fail verification.
size: medium
proposed_by: bbugyi200.athena.sase-iw
bead: sase-iw
create_time: 2026-08-10 10:54:30
status: wip
---

- **BEAD:**
  [sase-iw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-iw/README.md)

# Type-check extensionless Python tools

## Goal

Make the normal mypy lint stage cover every extensionless Python executable under
`tools/`, including newly created files that are not yet tracked, while preserving the
existing `src/` coverage and avoiding unrelated imported test modules or optional-only
dependency stubs. Remediate every direct tool diagnostic exposed by that gate.

## Implementation

1. Add a small extensionless Python lint helper under `tools/` that recursively
   discovers files whose first line is a Python shebang, excludes transient/cache
   directories, and invokes the configured mypy executable once with
   `--scripts-are-modules`, `--follow-imports=skip`, and `--ignore-missing-imports`. The
   helper must return mypy's exit status so any tool typing defect fails the lint gate.
2. Wire the helper into `Justfile`'s private `_lint-mypy` recipe after the existing
   project mypy invocation. Because `lint`, `check`, and `check-full` already share that
   private recipe, all normal verification paths will receive the new coverage without
   duplicating gate definitions.
3. Fix the direct diagnostics currently exposed in `tools/validate_sase_core_rs`,
   `tools/check_test_wait_helpers`, `tools/smoke_sase_core_rs_at_reference_file_gate`,
   `tools/render_model_alias_docs`, and `tools/test_image_notification` using explicit
   narrowing or annotations that retain their runtime behavior.
4. Add focused tests for discovery and invocation/exit propagation, including a
   temporary extensionless Python tool with an intentional assignment error that must
   make the helper fail. Extend the Justfile contract test to assert `_lint-mypy` runs
   the helper, protecting the normal-gate wiring from regression.

## Verification

- Run the focused helper and Justfile contract tests.
- Run the helper against the repository's real `tools/` tree and confirm all selected
  scripts pass mypy.
- Run `just install`, then the repository-mandated `just check` gate.
- Re-run `just _lint-mypy` after the full check to explicitly revalidate the completed
  normal type-check path before closing `sase-iw` with the verified evidence.
