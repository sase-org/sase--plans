---
tier: tale
title: Route machine artifact-link writes through hidden sidecar clones
goal:
  Background artifact-link maintenance writes, commits, and publishes only from fresh
  host-owned hidden sidecar clones while primary sidecars remain pull-sync-only.
size: medium
proposed_by: bbugyi200.athena.sase-y3.3
bead: sase-y3.3
create_time: 2026-09-09 20:00:38
status: wip
---

- **PARENT:**
  [202609/machine_link_mutations_off_primary.md](https://github.com/sase-org/sase--plans/blob/main/202609/machine_link_mutations_off_primary.md)
- **BEAD:**
  [sase-y3.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y3/sase-y3.3.md)

# Route machine artifact-link writes through hidden sidecar clones

Implement phase `sase-y3.3` without changing interactive artifact-link resolution or the
primary-sidecar ownership invariant.

## Implementation

1. Add a dedicated artifact-link store resolver for host background work. It will take
   the canonical project key and primary checkout, resolve the recorded split SDD store,
   map every document sidecar role (including plans and custom roles such as research)
   to `hidden_sidecar_clone_dir(project_key, role)`, and materialize or freshly
   integrate those clones from the recorded remote before returning an
   `ArtifactLinkStore`. It will use the corresponding primary clone only as a Git object
   reference when available. Non-split storage retains the existing resolved store
   behavior, so the ownership gate continues to fail closed where no hidden machine lane
   exists. Keep beads and the already-hidden agents sidecar available as read context;
   only document roots move to the machine lane.
2. Extend the workspace ownership contract with an explicit host-owned-sidecar access
   kind. Machine-context inference will recognize a target only when it is inside the
   canonical `~/.sase/projects/<project_key>/repos/<role>` root, the project lifecycle
   record resolves a primary checkout, and that primary's materialized SDD record names
   the role. Authorization for that context is confined to the one hidden clone. All
   unidentified paths, user-directed contexts, foreign canonical stores, and primary
   workspace `#0` paths remain refused.
3. Switch the hourly `artifact_link_backfill` project runner and the agents-sync
   Referenced By drain to the dedicated machine resolver. Pass both the project key and
   recorded primary checkout so all sweep, outbox, aggregate/rename repair, and
   Referenced By work shares the freshly integrated hidden store. Preserve the existing
   machine mutation origins, asynchronous publication paths, diagnostics, and
   per-project failure isolation.
4. Add focused tests for hidden-store path mapping, on-demand clone materialization,
   forced fresh integration, and use of primary clones solely as Git references. Add
   ownership tests proving a configured canonical hidden sidecar is machine-writable,
   sibling/unconfigured lookalikes are refused, confinement to the selected role is
   enforced, and primary `#0` refusal is unchanged. Update caller tests to assert both
   background anchors use the machine resolver with the correct project identity.
5. Add an end-to-end repository test: start with a clean primary clone behind a shared
   sidecar remote, resolve the hidden machine clone, apply and publish an artifact
   rename repair there (rewritten successor plus stale-index deletion), then run the
   existing primary sidecar auto-sync and assert it fast-forwards to the machine commit
   with a clean worktree. Retain explicit coverage that plans and research are included
   in auto-sync role discovery.

## Verification

- Run focused tests for artifact-link machine resolution, ownership authorization,
  backfill caller routing, agents-sync Referenced By routing, and primary auto-sync
  convergence.
- Run `just install` if required for this fresh workspace, then `just check` according
  to the repository verification policy.
- Run `just check-full` through `/sase_monitor` because the approved epic explicitly
  calls for the exhaustive lane for this phase.
- Run `sase bead epic-symbols sase-y3.3`, resolve or re-key every remaining symbol, and
  close only `sase-y3.3` with a note summarizing the verified lanes.
