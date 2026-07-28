---
tier: tale
title: Transactional agents-sidecar import and family revival
goal: Import validated v2 hoods as recoverable local history, preserve conservative
  v1 compatibility, and expose complete synced families through the existing R revival
  flow.
bead: sase-8v.5
create_time: 2026-07-24 14:30:06
status: done
---

- **PROMPT:** [202607/prompts/transactional_agent_import.md](prompts/transactional_agent_import.md)
- **PARENT:** [202607/global_agent_hoods.md](https://github.com/sase-org/sase--plans/blob/main/202607/global_agent_hoods.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-8v.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8v.5/README.md)
  - [bbugyi200.athena.sase-8v.5--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-8v.5.md#member-code)
- **COMMITS:**
  - [2409ed2](https://github.com/sase-org/sase/commit/2409ed2e37e454f712f44651534516d04517ef4f) — feat: import agent packages transactionally (sase-8v.5)

# Transactional agents-sidecar import and family revival

## Goal

Complete bead `sase-8v.5` by turning owner-sharded v2 hood snapshots into coherent, loader-compatible local history and
directly revivable saved families. Imports must validate an entire untrusted hood before mutation, localize every name
from explicit owner data, allocate and rewrite all run relationships as one batch, recover cleanly after interruption,
remain idempotent across refreshes, and retain conservative read-only support for legacy v1 bundles without guessing
usernames or republishing foreign data.

## Current state and constraints

- `src/sase/agents_sync/v2_models.py`, `v2_io.py`, `v2_run_io.py`, and `publication.py` already define and publish
  strict owner-sharded v2 manifests, hood snapshots, per-run payloads, digests, containers, and relationships.
- `sase.core.agent_identity_facade` already exposes owner classification/localization plus Rust-backed whole-batch
  relationship validation and destination-ID rewriting. Shared identity and graph rules must stay behind that boundary
  rather than being reimplemented in Python.
- `sase.agent.names.claim_imported_registered_name_v2` already enforces explicit source-owner provenance,
  exact-current-owner reuse, conditional local spellings, and owner-namespace collision checks. The importer must
  preflight a whole hood and use this path only when the batch can commit.
- `src/sase/agents_sync/bundles.py` imports v1 entries one agent at a time. It treats a matching machine token as local,
  lacks explicit unknown-username provenance, and cannot provide batch recovery or family revival.
- Full sync currently calls only that v1 importer before v2 publication. The integration seam in
  `src/sase/agents_sync/git_sync.py` must invoke dual v1/v2 reading while preserving the existing shared bounded sidecar
  lock and per-project failure isolation.
- Existing artifact loaders consume `agent_meta.json`, terminal marker/state, raw xprompt, workflow/prompt-step markers,
  and optional transcript paths. The `R` flow resolves saved-group refs through dismissed bundle suffixes, then restores
  artifacts and supports the existing relaunch actions.
- Imported data is historical only: never restore remote PIDs, running markers, workspace numbers, absolute host paths,
  or other live execution state. Do not delete or rewrite existing v1 sidecar payloads, mass-rename history, edit SASE
  memory/generated instruction files, close the parent epic, or create beads.

## Implementation

### 1. Build a fully validated import package before acquiring the mutation lock

- Add focused v2 import models/helpers under `src/sase/agents_sync/` that read an owner manifest entry, its hood
  snapshot, and every referenced per-run file into an immutable in-memory package.
- Enforce manifest/snapshot owner, project, hood, count, schema, path-containment, UTF-8/JSON, declared size, file
  digest, snapshot digest, and aggregate payload limits. Reject missing/extra required references, duplicate source
  runs/global names, unsafe names, and owner/name disagreement.
- Decode `meta.json`, `state.json`, and `commits.json` with the existing strict readers, cross-check them against the
  snapshot record, and validate relationships through the Rust facade before any local mutation or lock acquisition.
- Extend the strict optional per-run file allowlist and publisher/inventory only where needed to carry loader-compatible
  raw prompt, embedded-workflow, and prompt-step restart data that Phase 4 does not yet expose. Keep every added payload
  bounded, relative, deterministic, and free of machine-local execution fields.
- Add a manifest-discovery entry point that returns validated v2 hood packages separately from legacy v1 candidates, so
  a malformed hood is quarantined/reported as one unit and cannot partially block unrelated valid owner hoods.

### 2. Preflight owner localization, existing imports, collisions, and destination IDs as one hood

- Resolve one target `AgentIdentitySnapshot`, classify the explicit source owner, and derive every destination
  run/container name with `localize_imported_agent_name`. Preserve bare names for exact-current-owner round trips,
  `<machine>.<local>` for the same user on another machine, and `<username>.<machine>.<local>` for another user.
- Identify exact-current-owner source runs against matching local project artifacts/commit evidence and treat them as
  observations/refreshes rather than duplicate historical artifacts.
- Detect an already imported v2 hood/run by canonical global identity, source owner/run ID, and snapshot digest. Make
  the same digest a no-op and plan an in-place portable refresh for a later digest without changing its allocated
  destination ID.
- Preflight all registry and owner-container claims before writes. Reject/quarantine the complete hood on any
  local/imported owner collision; do not claim a prefix or materialize a subset while checking.
- Allocate all new artifact timestamps/run IDs in deterministic source order. Preserve a safe free source timestamp when
  available and probe forward deterministically on collisions, considering both disk state and every destination
  reserved earlier in the same batch.
- Pass the complete source-to-destination map to `rewrite_agent_relationship_batch`; project rewritten parent,
  workflow-parent, retry, wait, family, and clan relationships into the loader marker fields only after the Rust rewrite
  succeeds.

### 3. Materialize a recoverable whole-hood transaction

- Add a project-scoped import journal/store beneath SASE state with a stable transaction key derived from project,
  source owner, top-level hood, and snapshot digest. Journal only portable planned paths/IDs, digests, and state; never
  serialize untrusted absolute source paths.
- Use explicit `prepare`, `apply`, and `finalize` states. During prepare, render every destination artifact, chat,
  dismissed bundle, saved-group record, and provenance update into transaction-owned staging paths and durably write the
  journal.
- Render loader-compatible historical artifacts: allowlisted `agent_meta.json`, a terminal historical `done.json`/state
  even when the source snapshot was active, raw xprompt, optional embedded workflow/prompt-step markers, optional
  transcript, commit metadata, localized family/clan/relationship fields, canonical global name, source owner/run ID,
  and imported snapshot/file digests. Omit PID/running/workspace/host state.
- Under the shared project mutation lock, recover any unfinished journal first, then atomically rename the complete
  staged artifact/chat/bundle batch into place. Update registry provenance only after all durable files exist; if a
  later finalize step fails, recovery must be able to finish deterministically from the journal.
- Finalize artifact-index rows, dismissed-bundle index rows, saved-group metadata, and the journal completion marker in
  an idempotent order. On startup/next import, a prepared-but-unapplied transaction rolls back its stage, while an
  applied-but-unfinalized transaction finishes derived indexes/registry/group state. Never expose a partially
  materialized family to normal loaders.
- Return per-hood imported/refreshed/unchanged/quarantined diagnostics and aggregate them into the existing sync
  outcomes without changing established v1 count meanings.

### 4. Create one idempotent `R` saved group for each imported family

- For every v2 family container, write dismissed bundles compatible with `Agent.from_bundle_dict` and one saved-group
  archive record keyed deterministically by canonical global family plus snapshot digest. Preserve container member
  order, rewritten parent/workflow relationships, localized display names, source run IDs,
  model/provider/reasoning/tribe fields, raw prompt preview, and concrete bundle refs.
- Label these groups `Agents sidecar` in the existing saved-group modal through the existing free-form archive `source`
  field and rendering path; keep marked/recent group behavior unchanged.
- Ensure a later digest refreshes/replaces the family’s current sidecar group and bundle contents idempotently when
  transcript/final state arrives, without multiplying equivalent groups. Keep previously revived metadata semantics
  intact.
- Exercise the real `R` loader/resolution/restoration path, then the existing relaunch path, so a synced family can be
  revived directly without first dismissing imported rows individually.

### 5. Make legacy v1 conservative and dual-stack

- Keep `manifest.json` and `agents/<machine-qualified-name>` read-only. Validate all selected v1 bundles before
  importing any selected candidate and continue to use explicit v1 provenance fields so registry rebuilds preserve
  `username_unknown_v1`.
- Promote a v1 entry that names the current configured machine only when matching local artifact/commit evidence proves
  it is the current owner; in that case observe/refresh the local record rather than duplicating it.
- Treat every other v1 machine as foreign unknown-username provenance, including a token equal to a configured machine
  without ownership proof. Localize it only through the legacy compatibility spelling, never merge it with v2 owner
  data, and never export it as locally owned.
- Route full sync through the v2 hood importer plus the conservative v1 compatibility importer before local publication.
  Preserve v1 files in the sidecar and keep failures/quarantines scoped so one invalid foreign hood or legacy bundle
  does not leave partial local state.

### 6. Verify safety, recovery, loader behavior, and compatibility

- Add focused unit tests for strict referenced-file verification, unsupported schemas, invalid UTF-8/JSON, declared
  size/digest mismatch, path traversal, aggregate limits, owner/name mismatch, duplicate/dangling/cyclic relationship
  rejection, and no-write-before-validation.
- Add importer tests for the full owner matrix, exact-current round-trip/no duplicate, deterministic timestamp
  collisions, whole-batch relationship rewrite, family/clan containers, active-source historical markers, optional
  transcript/restart files, registry/container collision quarantine, refresh/no-op idempotence, and absence of live host
  state.
- Inject failures/restarts at prepare, each apply boundary, and finalize/index/group boundaries. Assert recovery
  produces either no visible imported family or one complete family with consistent artifact, chat, registry, index,
  dismissed-bundle, and saved-group state.
- Add dual-stack tests for proven-current v1 promotion, unknown-username ambiguity (including same machine token),
  coexistence with v2 payload, no deletion, and no foreign re-export.
- Add an integration test covering v2 publish/import/load, opening `R`, selecting an `Agents sidecar` family, restoring
  every member in order with rewritten parents, and relaunching from preserved raw prompts. Update saved-group
  rendering/visual snapshots only if the new source label changes the rendered modal.
- Run focused agents-sync, registry, loader, saved-group revival, and visual tests while iterating. Because project
  files will change, run `just install` and finish with `just check`; also run `just test-visual` if any rendered
  snapshot changes.

## Completion

- Record concise implementation/recovery notes on bead `sase-8v.5`.
- Close only `sase-8v.5` after all required checks pass and verify `sase-8v` remains open/in progress.
