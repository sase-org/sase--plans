---
tier: tale
title: Wire owner presentation facts through fleet rows
goal:
  Remote fleet rows carry the status, lineage, tribe, and runtime facts needed to
  reproduce the owner's presentation while remaining compatible with older payloads.
size: medium
proposed_by: bbugyi200.athena.sase-133.2
bead: sase-133.2
status: done
---

- **PARENT:**
  [202609/remote_dispatch_agents_tab_parity.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_dispatch_agents_tab_parity.md)
- **BEAD:**
  [sase-133.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-133.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-133.2.md)
- **COMMITS:**
  - [11b8e06](https://github.com/sase-org/sase-core/commit/11b8e060fe88d0a147b8449050d61a5f29038b9c)
    — feat(fleet): expose owner-resolved presentation facts

# Plan: Wire presentation facts

## Objective

Extend the fleet catalog contract additively so each remotely served agent row carries
the owner-resolved facts needed to reproduce the owner's status, family topology, tribe
grouping, and two-part runtime display. Populate those facts in the gateway from the
indexed record set, keep older payloads readable, and lock the shape down with contract
and gateway fixtures.

## Context

- `ResolvedAgentSummaryWire` currently exposes coarse `status`, `family_role`,
  `parent_timestamp`, direct-record tribe values, and one `started_at_unix` value.
- `resolve_record()` currently projects records independently. Historical records that
  omit family or tribe metadata cannot inherit it from a related root record even when
  that root is present in the same indexed snapshot.
- `started_at_unix_for_record()` prefers the run timestamp, so the wire cannot represent
  both the agent/family start and the current run start.
- New fields must be optional/defaulted and the fleet schema version must advance
  without breaking readers of v1-v3 payloads.

## Implementation

1. Extend the Rust fleet contract with bounded, optional owner presentation facts: an
   explicit display status, a distinct run-start timestamp alongside the agent/family
   start, and owner-resolved lineage/tribe inputs sufficient to populate `family_id`,
   `parent_timestamp`, `family_role`, `tribe`, and `clan_tribe` for historical rows.
   Bump the current fleet schema version while retaining the existing minimum readable
   version and serde defaults for older rows.
2. Make the pure projection prefer the explicit owner display status while preserving
   the existing record-derived fallback. Validate the new labels and timestamps,
   normalize nested locator schemas to the current version, and include the new facts in
   stable catalog hashing/comparison wherever presentation-visible summary fields
   participate.
3. In `sase_gateway`, derive a presentation context from the full served record set
   before projecting rows. Resolve missing family identity and tribe values from tracked
   parents, family/clan roots, and terminal marker metadata; compute separate
   agent/family and run-start timestamps; then pass the resolved facts into
   `project_resolved_agent_detail()` without exposing local paths or process
   identifiers.
4. Update the Rust binding/JSON contract surface and the Python fleet adapter only where
   required to preserve the new optional fields through serialization. Do not change
   viewer grouping or rendering policy assigned to the later viewer-parity phase.
5. Add focused Rust unit tests for projection validation/fallbacks and legacy schema
   decoding, plus gateway fixtures proving a done family's historical shells are fully
   linked, tribe-tagged, status-labelled, and carry both timestamps. Update golden JSON
   expectations for the bumped schema.

## Verification

- Run targeted `cargo test` suites for `sase_core::fleet_contract` and
  `sase_gateway::fleet_reads` while iterating.
- Run the linked core repository's standard fast verification command.
- If the primary Python repository changes, read its required lint/test memory and run
  the prescribed targeted and fast checks.
- Inspect `sase bead epic-symbols sase-133.2`, resolve or re-key every remaining symbol,
  then close only `sase-133.2` with a note naming the contract, compatibility,
  gateway-fixture, and verification evidence.

## Non-goals

- Do not implement remote TUI grouping, banner counts, shell-count chips, or visual
  rendering; those belong to `sase-133.3`.
- Do not deploy to another machine or collect the epic's final live screenshot evidence;
  that belongs to the epic lander.
- Do not close `sase-133` or any other phase bead.
