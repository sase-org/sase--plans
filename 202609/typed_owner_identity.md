---
tier: tale
title: Typed owner-aware agent identity
goal:
  Owner provenance is parsed and projected explicitly across Rust core, Python bindings,
  registries, sidecars, and import graphs without becoming agent topology.
size: medium
proposed_by: bbugyi200.apollo.sase-w2.6
bead: sase-w2.6
create_time: 2026-09-03 14:24:00
status: wip
---

- **PARENT:** [202609/athena_agent_sync_repair.md](athena_agent_sync_repair.md)
- **BEAD:**
  [sase-w2.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w2/sase-w2.6.md)

# Plan: Typed owner-aware agent identity

## Goal

Complete phase `sase-w2.6` by making owner provenance an explicit, typed part of
agent-name parsing and projection. A localized foreign name such as `athena.7n--code`
must parse as owner root `athena`, hood `7n`, family `7n`, and role `code`; it must
never be interpreted as hood `athena` or be globalized under the destination owner.
Preserve the existing localization behavior: exact-owner names become bare,
same-user/other-machine names retain the machine prefix, other-user names retain
username plus machine, and username-unknown v1 names retain their machine prefix.

## Implementation

1. In the linked `sase-core` repository, extend `crates/sase_core/src/agent_identity`
   with a validated `OwnerRoot` domain type and an owned-name parse result that
   separates `owner_root`, `hood`, `family`, and terminal member `role`. Accept a
   caller-supplied set of known owner roots, normalize archive prefixes, prefer the
   longest matching typed root, and retain the historical family-name rules for the
   semantic remainder. Add explicit errors for malformed roots and for attempts to treat
   a known foreign-root name as destination-owned.

2. Make the core name operations owner-aware. Route hood extraction, ancestry, hood
   membership, link-target construction, current-owner globalization, and new name
   validation through the typed parser when known roots are supplied. Keep compatibility
   behavior for callers with no known roots, but ensure the owner-aware entry points
   produce `7n`-scoped topology for `athena.7n--code`, reject names born inside a
   foreign root, and refuse to globalize a foreign-root name.

3. Add a single Rust-owned graph projection operation alongside the existing
   relationship validation/rewrite API. It must take canonical run/container/
   relationship identity, the explicit source and destination owners, and known roots,
   then return a consistent localized projection for run names, family/clan container
   names, global-name relationship targets, and the registry namespace root. Validate
   the graph before projection and fail atomically on inconsistent ownership or
   collisions so Python import code cannot independently rebuild prefixes.

4. Expose the new root parsing, owner-aware helpers, guarded globalization/ validation,
   and graph projection through `crates/sase_core_py`. Update export coverage and
   binding-level shape tests. Keep Python as typed dataclass/mapping conversion only; do
   not duplicate parsing or projection rules in the facade.

5. In the primary SASE repository, replace `AgentIdentitySnapshot.sibling_machines` as
   the sole parse context with known owner roots derived from data. Build the union of
   valid overlay discriminators, typed `owner_namespace` reservations in the durable
   name registry, and owner directories/manifests found beneath each configured agents
   sidecar's `users/<username>/machines/<machine>/` tree. Keep discovery read-only,
   deterministic, tolerant of absent or malformed optional sources, and structured to
   avoid registry-scan/identity-snapshot recursion.

6. Update `src/sase/core/agent_identity_facade.py` and its Python callers to pass the
   immutable known-root snapshot into Rust for parsing, lookup keys, launch validation,
   links, and globalization. Update v2 import rendering/planning to use the one-shot
   core graph projection rather than recomputing localized run, family, relationship,
   and namespace names independently. Preserve exact-first compatibility lookup
   candidates for genuine current-owner spellings.

## Verification

- Add Rust unit tests for typed roots and longest-prefix parsing, historical family
  spellings, owner-aware hood/ancestor/membership/link behavior, foreign-root
  globalization and creation rejection, and the full exact-owner / same-user-other-
  machine / other-user / username-unknown-v1 projection matrix.
- Add PyO3 binding tests for every new exported operation and wire shape, including
  graph projection and error propagation.
- Add Python facade/config/registry/sidecar discovery tests proving roots are found from
  each source independently and as a deduplicated union, without requiring an overlay
  file. Include the regressions: `agent_local_hood("athena.7n--code") == "7n"`,
  `foreign_agent_owner_root("athena.7n--code") == "athena"`, and
  `globalize_owned_agent_name("athena.7n--code")` raises when `athena` is known foreign
  provenance.
- Add importer tests proving a graph projects all related names consistently and no
  Python prefix reconstruction remains on that path.
- Run focused Rust and Python tests while iterating. Then run the linked core
  repository's required `just check`. Because the primary SASE repository is also
  changed, read `lint_and_test.md`, install the linked core binding as required for an
  ephemeral workspace, and run the prescribed primary `just check` lane.
- Before closing `sase-w2.6`, run `sase bead epic-symbols sase-w2.6`, resolve or re-key
  every remaining symbol to an open bead, and close only this phase with a note naming
  the verification performed.

## Boundaries

- Do not change the established username-stripping localization rule.
- Do not implement archive visibility/capabilities or ACE family/badge rendering; those
  belong to later phases `sase-w2.7` and `sase-w2.8`.
- Do not close the parent epic or any ancestor bead. Record out-of-scope discoveries
  only as `PROPOSED FOLLOW-UP:` notes on `sase-w2.6`.
