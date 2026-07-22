---
tier: tale
title: Complete agents-sidecar integration in sase repo init
goal: Managed projects can explicitly consent to and initialize a privacy-aware hidden
  agents sidecar at its stable machine-level path without disrupting other sidecars
  or the plans/research compatibility store.
bead: sase-8k.5
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 11:58:25
status: wip
---

- **PROMPT:** [202607/prompts/agents_sidecar_repo_init.md](prompts/agents_sidecar_repo_init.md)

# Complete agents-sidecar integration in `sase repo init`

## Goal

Finish bead `sase-8k.5` by making `sase repo init` explicitly declare, preflight, consent to, create/adopt, clone, seed,
and report the hidden `agents` sidecar. The flow must make publication of commit-associated agent chats unmistakable,
honor disabled/private configuration, keep hidden data out of workspace-local `sase/repos`, and leave the plans/research
compatibility store unchanged.

## Implementation

1. Extend repo-init configuration and specification resolution in `src/sase/main/repo_init_handler.py`.
   - Add the canonical agents role and description to the explicit `repos.sidecar` entries written for managed projects,
     while treating any existing agents entry—including `disabled: true`—as authoritative so init is idempotent and
     never resurrects an opt-out.
   - Stop filtering enabled hidden sidecars from `_configured_sidecar_specs`; preserve resolved repository pins, remote
     URLs, descriptions, and `visibility`, while continuing to suppress disabled entries.
   - Resolve each sidecar's clone root through one shared helper: ordinary roles remain under the checkout, while the
     agents role uses `hidden_sidecar_clone_dir` with the owning SASE project key so apply, materialized refresh, and
     read-only plan/diff output agree on the stable machine-level location.

2. Add agents-specific consent without weakening existing sidecar confirmation.
   - Render a dedicated multi-line prompt for a missing agents remote that states the full chat transcripts,
     prompts/responses, agent metadata, and commit associations that will be published across machines; show the
     configured visibility prominently and explain the `private` and `disabled` controls.
   - Keep an explicit interactive `y`/`yes` as the only authorization; default/negative, EOF, interruption, and non-TTY
     paths must refuse agents-repo creation and explain how to rerun interactively.
   - Treat refusal of the agents sidecar as an opt-out for that repo-init invocation rather than a failure: omit it from
     materialization and continue initializing found or authorized plans/research/custom sidecars. Preserve the existing
     failure behavior when the user declines a non-agents sidecar, and ensure no authorization leaks between roles.

3. Adapt sidecar initialization for hidden storage and compatibility boundaries in `src/sase/sdd/_sidecar_init.py` and
   its file-generation support.
   - Choose the agents clone root at `~/.sase/projects/<project_key>/repos/agents` and continue using workspace-local
     roots for visible sidecars; use the same selection in create/adopt and already-materialized refresh paths.
   - Keep the rigid plans/research `SddStoreRecord` compatibility projection restricted to those two roles, so adding
     agents neither creates nor mutates an agents slot and existing plans/research metadata remains intact.
   - Seed agents repos deterministically with a privacy-forward `README.md`, an initial `manifest.json` containing
     schema version 1 and an empty agents mapping, and a tracked placeholder for the empty `agents/` tree. The README
     must describe bundle purpose/layout, publication implications, and `sase agent sync`. Feed all generated paths
     through the existing `commit_sdd_files` and push transaction.

4. Make read-only planning accurately preview the new behavior.
   - Include the explicit agents config declaration in `--check`/`--diff` results.
   - Report pending agents remote creation with configured visibility and the machine-level clone path, and preview all
     deterministic seed files without probing the network or writing local state.

5. Expand focused regression coverage.
   - In repo-init handler/plan tests, cover explicit-config insertion and second-run idempotence, preservation of an
     existing disabled agents entry, enabled/private spec propagation, agents-specific public/private prompt text,
     default-no and non-TTY refusal, refusal continuing other sidecars, and plan/diff paths/details for the
     machine-level clone.
   - In sidecar initialization/file tests, cover hidden-root selection, agents seed contents and drift idempotence,
     provider visibility/authorization forwarding, coexistence with plans/research, and proof that the compatibility
     store remains plans/research-only.
   - Update existing expectations that currently assert only plans/research declarations or deliberately exclude agents.

## Validation

- Run focused tests for `tests/main/test_repo_init_handler.py`, `tests/main/test_repo_init_plan.py`, and
  `tests/sdd_store/test_sidecar_init.py` while iterating.
- Run `just install` as required for this ephemeral workspace, then run the mandatory full `just check` suite.
- Recheck `git diff`, confirm only bead-scoped source/tests changed, and close `sase-8k.5` only after validation passes;
  do not close parent epic `sase-8k` and do not create beads.

## Acceptance criteria

- A managed project's explicit config gains one agents sidecar declaration unless any agents entry already exists.
- Missing agents repos can be created only after loud, role-specific, interactive consent; refusal does not prevent
  other sidecars from completing.
- Public/private visibility is both displayed and passed unchanged to the provider.
- Agents clones and plan output use the stable machine-level hidden path, never workspace-local `sase/repos/agents`.
- The initialized agents repo contains the README, empty manifest, and tracked agents tree, committed and pushed through
  the established transaction.
- The plans/research compatibility record is unchanged in shape and semantics.
- Focused tests and `just check` pass before the bead is closed.
