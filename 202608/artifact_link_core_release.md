---
tier: epic
status: done
title: Publish artifact-link bead mutations and raise the floor
goal:
  The artifact-link bead mutation feature is backed by a published sase-core-rs release
  and sase requires that release.
parent_bead: sase-r8
phases:
  - id: core_release
    title: Publish the bead-link mutation bindings
    depends_on: []
    size: small
    description:
      "core_release: release the linked sase-core commit containing bead_add_link and
      bead_remove_link inside the existing 0.29 compatibility window, and verify the
      published Python package exposes both bindings."
  - id: floor_integration
    title: Require and verify the published core release
    depends_on:
      - core_release
    size: small
    description:
      "floor_integration: raise sase's sase-core-rs floor to the containing published
      release, refresh the lockfile, remove any obsolete unpublished-capability
      accommodation, and run the focused artifact-link plus repository verification
      gates."
proposed_by: bbugyi200.athena.sase-r8.land
bead_id: sase-r8.9
create_time: 2026-09-09 19:49:50
---

- **PROMPT:**
  [prompts/202608/artifact_link_core_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifact_link_core_release.md)
- **PARENT:** [202608/artifact_link_graph.md](artifact_link_graph.md)
- **BEAD:**
  [sase-r8.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-r8/sase-r8.9.md)

# Publish artifact-link bead mutations and raise the floor

Epic `sase-r8` implemented typed bead links through the Rust bindings `bead_add_link`
and `bead_remove_link`. The linked `sase-core` checkout has both at commit `751d60f`,
but its newest release tag, `v0.29.4`, points to the preceding commit `fe6f97d`.
Consequently `tools/validate_sase_core_rs` reports `blocked_unpublished`: the declared
Python floor cannot supply behavior the landed feature requires.

This plan contains only the remaining release integration. It does not repeat the
artifact-link implementation and does not include closing `sase-r8`, its Symvision pass,
or its plan-file status update; the parent link hands those landing steps back to the
waiting `sase-r8` land agent.

## Acceptance criteria

- A tagged and published `sase-core-rs` release in the existing `>=0.29,<0.30` window
  contains commit `751d60f` or an equivalent commit exposing both bead mutation
  bindings.
- `pyproject.toml` and `uv.lock` require that containing release, and
  `tools/validate_sase_core_rs` reports the capabilities as published rather than
  `blocked_unpublished`.
- Focused Rust/Python bead-link tests cover add, remove, event projection, and page
  rendering against the published floor.
- `just check` passes. If its selection escalates or the broadening set is touched, run
  `just check-full` through `/sase_monitor` before handing back to the parent land
  agent.

## Phase guidance

### core_release

Open `sase-core` with `/sase_repo`; do not infer a sibling path. Confirm the release
contains both mutations and the artifact-link wire APIs already shipped in `0.29.3`. Use
the repository's ordinary release workflow, update release metadata as required, and
verify the published wheel/version before closing the phase. Do not introduce a `0.30`
compatibility break for this additive binding release.

### floor_integration

Run `just install` after changing the dependency floor. Exercise the exact Python paths
in `src/sase/core/bead_mutation_facade.py`, bead event persistence, CLI link mutation,
and bead-page projection. Re-run `sase bead epic-symbols sase-r8` as an audit input but
leave the parent epic's final cleanup and close to its land agent.
