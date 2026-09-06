---
tier: tale
title: Require the released artifact-context core bindings
goal:
  Published SASE installs always receive the Rust artifact-context query contract used
  by the wait.artifacts runtime namespace.
size: medium
proposed_by: bbugyi200.athena.sase-x8.land
bead: sase-x8
create_time: 2026-09-05 22:34:14
status: wip
---

- **PARENT:** [202609/wait_artifacts.md](wait_artifacts.md)
- **BEAD:**
  [sase-x8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x8/README.md)

# Outcome

The SASE package can no longer install or validate successfully against a `sase-core-rs`
release that lacks the `wait.artifacts` query bindings added by sase-x8.
Published-install checks exercise those binding names, and the lockfile selects the
first complete release that contains them.

# Evidence and constraints

- `src/sase/core/artifact_context_query_facade.py` calls `artifact_context_query` and
  `artifact_context_query_wire_schema_version`.
- Those bindings first shipped in the complete PyPI release `sase-core-rs 0.32.25`;
  `tools/ratchet_core_window --report-only` resolves the current safe window to
  `>=0.32.25,<0.33.0`.
- `pyproject.toml` still accepts `>=0.32.19,<0.33.0`, and `uv.lock` currently selects
  `0.32.20`, so a published SASE install can reach the lazy artifact property with an
  incompatible core.
- The post-sase-x8.1 migration work expanded `tools/validate_sase_core_rs` with its new
  binding contract, but that manual inventory does not yet include the two
  artifact-context bindings. The static `tools/check_sase_core_rs_bindings` scan already
  discovers both facade call sites.
- Do not change `sase-research-artifacts`' `sase>=0.17.0` floor in this work. There is
  not yet a published SASE release containing the runtime namespace, and the parent plan
  forbids guessing a future version. The resumed land agent will recheck release
  availability and publication ordering.

# Implementation

1. Use the repository's core-window ratchet path to update `pyproject.toml` and only the
   permitted `sase-core-rs` regions of `uv.lock` to the complete `0.32.25` release.
   Preserve the existing `<0.33.0` compatibility ceiling and do not hand-edit
   release-owned package versions.
2. Add `artifact_context_query` and `artifact_context_query_wire_schema_version` to the
   explicit required-binding inventory in `tools/validate_sase_core_rs`, alongside
   focused tests that prove each missing symbol makes validation fail. Keep the static
   binding scanner as the independent source-tree check; do not duplicate its
   implementation.
3. Verify the facade tests, binding-check/validator tests, dependency-window tests, and
   the exact published-floor validation path against `sase-core-rs 0.32.25`. Run
   `just install` first if the ephemeral workspace's extension is stale, then run the
   repository-default `just check` gate. Record any unrelated full-lane failure as
   evidence rather than weakening the floor checks.

# Acceptance criteria

- The declared and locked `sase-core-rs` minimum is `0.32.25`, with no unrelated
  dependency refresh.
- Both artifact-context bindings are present in the static scan and the explicit
  environment validator contract.
- Validation fails clearly when either binding is absent and succeeds against the
  published `0.32.25` wheel.
- Existing `wait.artifacts` facade behavior and the rest of `just check` remain green.
