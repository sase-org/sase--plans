---
tier: tale
title: Identity-relative local persistence and registry compatibility
goal: New locally owned agent state is stored with bare semantic names, imported provenance
  remains explicit and collision-safe, and legacy qualified history continues to resolve
  without mass renaming.
bead: sase-8v.3
parent: sase/repos/plans/202607/global_agent_hoods.md
create_time: 2026-07-23 15:02:52
status: done
---

- **PROMPT:** [202607/prompts/identity_relative_local_persistence.md](prompts/identity_relative_local_persistence.md)

# Plan: Identity-relative local persistence and registry compatibility

## Scope and invariants

Implement Phase 3 (`local-persistence`) of the `sase-8v` epic in the primary SASE repository, using the owner-aware Rust
facade and nested identity configuration delivered by `sase-8v.1` and `sase-8v.2`.

The implementation must preserve these boundaries:

- New locally owned agent, family, clan, wait/reference, chat, and registry writes use the bare semantic name.
- A legacy current-machine spelling such as `athena.foo` and an early full spelling such as `bbugyi200.athena.foo`
  remain exact-first lookup aliases for the same local identity when current ownership is known. Existing artifact/chat
  directories are not renamed.
- Imported v2 data is localized only from explicit source-owner fields: exact-current-owner refreshes do not duplicate,
  same-user/other-machine names retain the machine hood, and other-user names retain username plus machine.
- Legacy v1 imports remain username-unknown foreign data. They are never guessed to be local or republished as local.
- The existing v1 sidecar writer may continue to emit its required machine-qualified compatibility form until the v2
  publishing phase replaces it. Commit/sidecar transport identity must not be confused with local storage identity.
- No memory file, generated agent-instruction shim, historical commit, or historical artifact directory is rewritten.

## Intent-specific identity policy

Replace application use of the ambiguous machine-qualification policy with an owner-aware application facade built on
`sase.core.agent_identity_facade` and the selected `AgentOwnerIdentity`. Expose explicit operations for:

- normalizing a newly locally owned storage name to its bare semantic form;
- producing transport/global provenance for a locally owned name;
- localizing an imported global name from an explicit `AgentSourceOwnerIdentity`;
- classifying source ownership;
- presenting current-owned legacy spellings without hiding conditional foreign hoods;
- generating exact-first lookup candidates for bare, legacy machine-qualified, and early fully-qualified current-owned
  records.

Resolve one immutable identity snapshot per batch or presentation refresh. Include configured sibling machine names for
foreign-local launch guards, but do not infer an arbitrary dotted name's owner from its text. Retain deprecated
machine-hood wrappers only where an explicitly documented v1 compatibility path still needs them.

## Reverse every locally owned write path

Migrate all current local qualification choke points so the stored value is bare:

- launch preflight, planned-name allocation/reservation, template allocation, generated-name environment, runner
  identity, metadata, done markers, and workflow names;
- family attachment/promotion, family member/base comparison, clan declaration/joining, and their container registry
  entries;
- waits, forks, output-variable namespaces, retry/repeat/resume references, directive edits, and persisted waiting
  markers;
- rename and revival reconstruction, including family/clan/reference fields;
- chat catalog/presentation and Agents-tab identity, hood, family, clan, neighbor, completion, and grouping projections.

Keep comparisons and lookups exact-first, then consult current-owner compatibility aliases. New writes must never select
the legacy alias merely because an old row exists; they must collide with that row and report the visible bare name.
Foreign-qualified launch requests must remain rejected, while imported historical rows remain loadable.

At explicit external boundaries, replace accidental local qualification with the correct intent: globalize owned commit
provenance through the configured owner, and keep a narrowly named machine-qualified v1 sidecar adapter until Phase 4
retires the writer. Imported v1 bundle creation/refresh must retain username-unknown provenance.

## Versioned registry provenance and namespace safety

Upgrade the durable agent-name registry schema and entry model without dropping planned reservations or existing
artifact ownership:

- record origin (`local`, explicit v2 import, or username-unknown v1 import), canonical global name when known, source
  owner `{username, machine_name}` when known, legacy source machine when username is unknown, and imported content
  digest;
- make exact same-source/digest refresh idempotent and reject a different owner, global identity, or artifact owner with
  a typed collision;
- preserve explicit provenance during registry rebuild by reading artifact/bundle metadata instead of deriving ownership
  from dotted name segments;
- mark unannotated historical local artifacts as local, recognize current-machine and early current-global spellings as
  current-owned aliases, and retain their exact stored keys;
- reserve configured sibling-machine roots and observed foreign-username roots as owner-namespace containers, so local
  allocation beneath those roots cannot silently alias a future import;
- quarantine/reject an incoming owner hood when any destination spelling or namespace collides, without overwriting,
  suffixing, or partially updating registry state.

Provide a v2-ready imported-claim API that accepts explicit source owner, canonical global name, destination-localized
name, digest, and artifact owner. Keep a compatibility wrapper for the current v1 importer, and update registry queries,
rebuild/staleness checks, template namespace allocation, forced reuse/wipe, and collision messages to respect the new
provenance and container kinds.

## Compatibility-aware loading and presentation

Update loaders and presentation helpers so current-owned bare, legacy current-machine, and early current-global names
share one visible local hierarchy, while `zeus.foo` and `alice.athena.foo` keep their required foreign hierarchy. Apply
the same policy to family/clan containers, wait targets, output-variable keys, retry/revive prompts, chat listings, and
completion candidates. Preserve full global/source provenance in registry and imported artifact metadata for later
detail views and v2 sidecar phases.

Do not add filesystem or configuration reads to Textual render paths. Resolve one identity snapshot during the existing
agent refresh/status-normalization pass and let row rendering consume the precomputed names.

## Verification

Add focused unit and integration matrices covering:

- new launches, templates, multi-prompt plans, waits/forks, rename, revival, family promotion/attachment, clans,
  retries/output variables, chat metadata, and registry rebuild all persisting bare local names;
- configured owner `bbugyi200.athena` resolving exact-first bare `foo`, legacy `athena.foo`, and early
  `bbugyi200.athena.foo` records without rewriting their artifact directories;
- explicit v2 localization for exact owner, same user on `zeus`, another user on `athena`, and different users sharing a
  machine name;
- v1 username-unknown imports, same-source refresh, different-owner and local/import collisions, sibling-machine and
  observed-username namespace reservations, and whole-claim rollback on collision;
- TUI/chat/family/clan grouping and display hiding only the current owner prefixes while preserving foreign hierarchy;
- v1 sidecar export/backfill continuing to produce its legacy transport spelling from bare local artifacts.

Run focused tests while iterating. Then run `just install` followed by `just check`, as required for changes in this
repository. Run `just test-visual` if the Agents-tree presentation changes or snapshots are affected. Re-run the
qualification audit to confirm no ambiguous local-write call sites remain, inspect the final diff for accidental memory
or generated-instruction edits, and close only `sase-8v.3` after all validation passes.

## Acceptance criteria

- Every newly written locally owned name or relationship is bare, while transport/provenance remains explicit.
- Legacy current-owned qualified records resolve exact-first and display locally without any mass migration.
- Imported provenance is explicit, same-source refresh is idempotent, and foreign/local or foreign/foreign collisions
  fail safely before partial registry mutation.
- Same-user/other-machine and other-user/same-machine identities remain distinct in storage, lookup, grouping, and
  presentation.
- Registry namespace reservations prevent later foreign hood imports from aliasing pre-existing local descendants.
- Existing v1 sidecar compatibility remains functional and never promotes username-unknown imports to local ownership.
- Required checks pass, `sase-8v.3` is closed, and parent epic `sase-8v` remains open.
