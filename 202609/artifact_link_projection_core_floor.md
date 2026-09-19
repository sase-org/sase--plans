---
tier: epic
title: Ratchet the core wheel floor for bulk artifact-link projection
goal:
  Published SASE installations resolve a sase-core-rs release that provides every
  binding used by the current Python tree, including bead_set_link_projections.
phases:
  - id: core_floor_reconciliation
    title: Reconcile the published core wheel floor
    depends_on: []
    size: medium
    description:
      "core_floor_reconciliation: move the declared sase-core-rs floor and lockfile
      through the safe window workflow, including recovery from the invalid old
      baseline, and prove the published floor provides every Python-used binding."
parent_bead: sase-12y
proposed_by: bbugyi200.apollo.sase-12y.land
create_time: 2026-09-18 20:08:24
status: wip
---

- **PROMPT:**
  [prompts/202609/artifact_link_projection_core_floor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_projection_core_floor.md)
- **PARENT:**
  [202609/artifact_link_projection_timeout.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_projection_timeout.md)

# Ratchet the core wheel floor for bulk artifact-link projection

## Problem

Epic `sase-12y` moved artifact-link projection onto the atomic Rust batch API. Its
Python code now calls `sase_core_rs.bead_set_link_projections`, introduced by core
commit `d32591f` and first published in `sase-core-rs` v0.34.53. The main package still
declares `sase-core-rs>=0.34.48,<0.35.0`, so a valid dependency resolution can install a
wheel that lacks the binding and fail at import or first use.

The local linked-core development path masks this packaging defect. During the epic
landing audit, `tools/probe_core_floor` installed the declared floor in an isolated
environment and reported five missing capabilities; the newest missing capability in
that snapshot first appeared in v0.34.54. Core then advanced concurrently through
v0.34.57, so the implementation must derive the live target rather than pinning a
version copied from this plan.

The supported window reconciler currently cannot start from the stale declaration:
`tools/ratchet_core_window --report-only` exits 3 because v0.34.48 is not a complete
non-yanked PyPI release under its policy. That guard must not be bypassed by weakening
release-completeness checks or by allowing an uncontrolled lockfile refresh.

## Phase `core_floor_reconciliation`: reconcile the published wheel floor

1. Re-derive the live compatibility state before editing:
   - Record the current `sase-core` linked revision and the published release that
     contains it. Preserve any newer release or dependency-window changes that landed
     after this plan was written.
   - Run `tools/probe_core_floor --json` against the current `pyproject.toml` and retain
     its missing-capability evidence.
   - Run `tools/ratchet_core_window --report-only` (or the equivalent `just` recipe) to
     determine whether the invalid-starting-floor refusal still reproduces.

2. Move the declared `sase-core-rs` floor and `uv.lock` to the newest complete published
   release accepted by the repository's core-window policy:
   - Prefer the sanctioned `tools/ratchet_core_window` workflow.
   - If it still refuses solely because the existing v0.34.48 floor is no longer a
     complete non-yanked baseline, make the smallest auditable bootstrap of the
     dependency requirement to a complete published release that contains every binding
     used by the current Python tree, then rerun the sanctioned reconciler so it chooses
     the final live target and validates the lockfile diff.
   - Keep the `<0.35.0` ceiling policy intact. Do not select only v0.34.53 if later
     Python-used capabilities require a newer first release, and do not lower or
     overwrite a newer concurrent floor.
   - Refresh only the `sase-core-rs` lock entry and mechanically required lock metadata.
     Do not accept unrelated direct-dependency movement. If unchanged-package-set
     transitive metadata must refresh, use the tool's explicit opt-in only after
     reviewing and documenting that diff.

3. Add or adjust focused regression coverage only if repository behavior changes beyond
   the dependency declaration and lockfile. In particular, do not weaken
   `ratchet_core_window`'s release-completeness or unexpected-diff guardrails merely to
   make this one ratchet pass. If a reusable recovery path is necessary, cover the
   invalid-old-floor case in the ratchet tool's focused tests and retain all existing
   safety properties.

4. Verify the packaging contract without relying on the linked-core development build:
   - `tools/probe_core_floor --json` must report that the declared floor satisfies the
     current Python binding set when it installs the published wheel in isolation.
   - The window reconciler's check/report mode must show no pending ratchet against the
     live published release set.
   - Run the focused core-window/probe tests if their code or fixtures changed.
   - Run the artifact-link facade and projection tests that exercise
     `bead_set_link_projections`.
   - Run `just check` per the repository's two-speed verification policy.

## Scope boundaries

- This child owns only the dependency-floor reconciliation uncovered by the `sase-12y`
  landing audit.
- Do not close `sase-12y`, remove or re-key its `--epic-symbol` entries, run the final
  post-close Symvision pass, or mark `plan:202609/artifact_link_projection_timeout.md`
  done. The parent epic's resumed land agent owns those steps after this child lands.
- Do not change Rust projection behavior: the batch implementation and its Rust/PyO3
  tests are already complete in core commit `d32591f`.

## Done when

- No dependency resolution allowed by `pyproject.toml` can install a `sase-core-rs`
  wheel missing `bead_set_link_projections` or any other binding used by the current
  Python tree.
- `pyproject.toml` and `uv.lock` agree on the reconciled, complete published release
  window with no unrelated dependency churn.
- The isolated floor probe, no-pending-ratchet check, focused artifact-link tests, and
  `just check` pass.
