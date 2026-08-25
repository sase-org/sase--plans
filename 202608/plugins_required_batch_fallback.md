---
tier: tale
title: Complete required-plugin batch fallback integration
goal:
  PluginsRequired installs share the epic's bounded source resolution and one
  receipt-preserving uv mutation.
size: medium
proposed_by: bbugyi200.athena.sase-to.land
bead: sase-to
create_time: 2026-08-25 14:48:05
status: wip
---

- **PARENT:**
  [202608/git_fallback_and_bugyi_chops_release.md](git_fallback_and_bugyi_chops_release.md)
- **BEAD:**
  [sase-to](https://github.com/sase-org/sase--beads/blob/main/pages/sase-to/README.md)

# Plan: Complete required-plugin batch fallback integration

## Context

Epic `sase-to` added a typed public-PyPI availability probe and a bounded
`plan_install_many()` path. The approved epic plan explicitly required that path to
cover both marked ACE installs and `PluginsRequired` notification gates so a required
set shares one availability deadline and one reconstructed `uv` operation.

The ACE marked-set path uses the batch planner, but
`src/sase/plugins/_required_gate_spec.py::_execute_install()` still calls
`plan_install()` and `execute_install()` once per baked plugin name. Besides multiplying
the public-index timeout by the number of required plugins, it plans every command from
the same pre-execution receipt. With two missing plugins, executing those independently
can let the second reconstructed `uv tool install` omit the first plugin.

This tale is only the remaining epic-owned integration work. The pre-existing behavior
where a `version_mismatch` row is accepted as `AlreadyInstalled` without satisfying its
specifier is tracked separately by task `sase-tr` and is not part of this repair.

## Implementation

1. Replace the per-name planning/execution loop in the trusted `PluginsRequired` install
   command with one call to `plan_install_many()` and, when ready, exactly one call to
   `execute_install_many()`.
   - Preserve fail-closed behavior for catalog errors, non-uv-tool environments, unknown
     plugins, unsafe/skipped inputs, planning exceptions, and execution errors.
   - Treat only `already installed` skips as a successful no-op so a gate answered after
     another process installed a missing plugin remains idempotent. Continue rejecting
     every other skip reason.
   - Preserve the typed result contract: report the complete baked plugin-name set in
     stable input order and derive `changed` from the single batch outcome. An all-no-op
     batch reports `changed: false`.
   - Rename the injectable test seams to batch-planner and batch-executor semantics; do
     not add a second availability or source-resolution implementation to the gate.

2. Update the human-facing and generated descriptions that currently promise one
   `sase plugin install` command per missing requirement. Describe one combined install
   operation with a bounded index probe and per-plugin index/definitive-404 git source
   resolution in:
   - `src/sase/plugins/_required_gate_preview.py` and relevant command docstrings;
   - `src/sase/default_config.yml` and its generated `docs/configuration.md` copy;
   - the `PluginsRequired` sections of `docs/axe.md` and `docs/notifications.md`.

3. Extend `tests/test_plugins_required_gate.py` and nearby gate contract tests to prove:
   - two baked missing names are passed to one batch planner and executed through one
     reconstructed argv/outcome, including a plan containing both catalog and git
     sources;
   - `NotUvTool`, planning errors, non-benign skipped inputs, and execution errors keep
     the gate pending with a nonzero result;
   - an all-`already installed` plan and a ready plan with benign already-installed
     skips succeed idempotently with the correct ordered `installed` list and `changed`
     value;
   - rebuilt gate previews/specs and checked-in configuration documentation match the
     new batch wording.

## Verification

- Run the focused PyPI probe, install-planning, required-gate construction/validation,
  response, preview, and action tests.
- Exercise a disposable receipt with two missing catalog plugins whose injected probe
  map resolves one from the index and one from git; assert the gate plans one `uv`
  command containing both requirements and preserves the receipt's existing injected
  requirements.
- Run `just check`, as required for changes in the SASE repository.

## Acceptance criteria

- A `PluginsRequired` install response plans every baked missing name through one
  bounded availability batch and performs at most one `uv` mutation.
- Only a definitive public-PyPI 404 selects git for a required plugin; available and
  unavailable probe results remain index-based, using the shared planner unchanged.
- Multi-plugin gate installs cannot drop an earlier plugin because every requested and
  existing receipt requirement is present in the single reconstructed argv.
- Gate failure/idempotency semantics and the persisted typed result schema remain
  intact, and all user-facing descriptions match the shipped batch behavior.
- Focused tests and `just check` pass.
