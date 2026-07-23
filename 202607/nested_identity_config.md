---
tier: tale
title: Nested identity config and initializer migration
goal: Make the selected machine overlay the sole source of a validated per-user owner
  identity, migrate legacy overlays safely, and require complete identity for new
  provenance.
bead: sase-8v.2
parent: sase/repos/plans/202607/global_agent_hoods.md
create_time: 2026-07-23 14:13:21
status: done
---

- **PROMPT:** [202607/prompts/nested_identity_config.md](prompts/nested_identity_config.md)

# Nested identity config and initializer migration

## Goal

Complete bead `sase-8v.2` by making the selected machine overlay the sole authority for an explicit
`id.username`/`id.machine_name` owner identity, migrating legacy top-level `machine_name` overlays without losing
unrelated YAML or deployment behavior, and making commands that create new agent provenance fail with an actionable
initializer diagnostic until the full identity is configured.

This tale is intentionally limited to the identity-config phase. It preserves the legacy machine-name compatibility
facade for existing readers and later phases, and does not implement identity-relative local persistence, v2 sidecar
serialization/import, global commit links, or the linked chezmoi migration assigned to dependent beads.

## Implementation

1. Add the nested identity schema and a raw selected-overlay identity model in `sase.config`.
   - Define `id.username` and `id.machine_name`, keeping top-level `machine_name` only as deprecated migration input and
     adding no bundled identity default.
   - Discover machine overlays by nested `id.machine_name` first and legacy top-level `machine_name` second.
   - Resolve and cache one immutable identity snapshot from the selector plus the selected raw overlay, validate it
     through the Rust-backed `AgentOwnerIdentity` facade, and key it from the existing selector/overlay freshness token
     so repeated/render-path reads do not reparse YAML.
   - Export optional and required owner accessors while retaining `get_machine_name()`/`require_machine_name()` as
     compatibility projections of a complete configured owner.
   - Prevent identity values from defaults, plugins, the user base, ordinary overlays, or project-local config from
     overriding provenance; preserve the selected `id` object in the merged config for inspection and surface actionable
     source/conflict diagnostics.

2. Rework `sase config init` and its shared `sase init config` planner around migration-aware overlay state.
   - Classify missing selector/overlay, legacy, partial nested, complete, selector mismatch, duplicate, and conflicting
     identity cases before prompting or writing.
   - Explain the stable per-user username contract, validate explicit usernames with the Rust identity domain, offer a
     single existing username only behind clear confirmation, and never choose between conflicting usernames.
   - On creation or migration, minimally set `id.username` and `id.machine_name` in the same machine overlay, remove its
     legacy top-level key, preserve comments/unrelated content, and then write the existing machine selector.
   - Keep non-TTY failure, collision confirmation, chezmoi source remapping, direct/deferred deployment, no-commit,
     no-push, no-apply, cache invalidation, and idempotent reruns intact.
   - Make `--check`, bare onboarding, and doctor planner output distinguish missing username, legacy migration, selector
     mismatch, invalid values, and identity conflicts without mutating files.

3. Enforce the complete identity at provenance-creating choke points without blocking repair and history inspection.
   - Require the configured owner before actual agent process creation.
   - Make new commit runtime provenance use the configured identity rather than falling back to the hostname.
   - Route existing agents-sidecar mutation requirements through the compatibility projection of the complete owner.
   - Keep config/help/init/doctor and read-oriented legacy history paths available when identity is missing or only
     partially migrated.

4. Update public documentation and command help for the stable identity contract.
   - Document the selector-versus-overlay distinction, username uniqueness and cross-machine reuse, authoritative source
     restrictions, legacy migration behavior, initialization failure modes, and complete-identity requirement.
   - Update initializer and agents-sidecar references while avoiding promises belonging to later sidecar/persistence
     phases.

## Tests and validation

- Add focused config/schema/cache tests for clean nested selection, nested-over-legacy discrimination, source authority,
  selector and overlay invalidation, compatibility projections, invalid/reserved usernames, and conflicting identities.
- Expand initializer/onboarding/doctor tests for clean creation, source-preserving legacy and partial migration,
  one-user multi-machine reuse confirmation, conflict refusal, non-TTY behavior, idempotence, and direct/deferred
  chezmoi deployment.
- Add choke-point tests proving incomplete identity blocks actual launches, commit provenance, and sidecar mutations
  with the `sase config init` diagnostic while read-only config/history paths remain usable.
- Update schema-inventory and documentation assertions for `id.username`, `id.machine_name`, and deprecated top-level
  `machine_name`.
- Run focused tests while iterating, then run `just install` followed by the repository-mandated `just check`.

## Completion

After validation succeeds, update only `sase-8v.2` to `closed`. Do not close parent epic `sase-8v` and do not create
additional beads.
