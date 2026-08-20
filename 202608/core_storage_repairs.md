---
tier: tale
title: Complete the core-storage phase
goal:
  Repair hidden-row retention, nullable bead updates, collision-safe created IDs, and
  conservative workspace claims with cross-language verification.
size: medium
proposed_by: bbugyi200.athena.sase-rm.1
bead: sase-rm.1
create_time: 2026-08-20 14:59:12
status: wip
---

- **PROMPT:**
  [prompts/202608/core_storage_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/core_storage_repairs.md)
- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.1.md)

# Complete the core-storage phase

## Outcome

Finish the four contracts assigned to phase bead `sase-rm.1` across `sase-core` and the
primary `sase` repository, with focused cross-language regressions and each assigned
task left close-ready for the epic land agent. Close only `sase-rm.1` after the
repository gates pass and its epic-symbol ownership is clean.

## Constraints and current design

- Work in the linked `sase-core` checkout opened through `sase repo open`; honor its
  `AGENTS.md`, do not edit release-managed Cargo versions, and run its full `just check`
  rather than relying on a core-crate-only test.
- Treat the artifact tree as the source of truth. The SQLite agent-artifact index is a
  rebuildable materialized view, so old terminal hidden rows may leave the hot index,
  but active/visible rows and lookup-critical lineage or context must never be lost.
- Preserve the existing duplicate-event relocation winner semantics. This phase fixes
  the stale caller identity after relocation; it must not weaken append-only stream
  validation or allow a create caller to continue with an unverified ID.
- Do not create or close the four assigned task beads (`sase-kh`, `sase-mu`, `sase-oi`,
  `sase-qa`). Record one evidence block per task on `sase-rm.1`; the epic land agent
  owns task closure. Record unrelated discoveries only as `PROPOSED FOLLOW-UP:` notes on
  the phase.

## Implementation

1. Add a Rust-core hidden-row retention operation for the agent-artifact index. Keep all
   visible or active rows and a deterministic newest-first hot window of terminal hidden
   rows; prune the remaining derived rows and their dependent projections in one
   transaction. Preserve lookup-critical retry/parent/clan context (either by exempting
   the minimal anchors from eviction or by keeping a compact cold lookup projection),
   expose retained/pruned counts in the update/status wire, and document that the
   artifact tree plus an explicit rebuild is the recovery path for evicted historical
   payloads. Apply the policy after rebuild and from deferred lifecycle/GC maintenance
   so repeated upserts and rebuilds converge to a bounded hot index. Cover legacy-schema
   migration, idempotence, active/visible safety, exact and related artifact lookup
   behavior, dependent-row cleanup, and a representative fixture with at least 4,700
   hidden rows. Thread the Rust binding through the Python facade, lifecycle report, CLI
   status/GC output, and contract tests.

2. Give `BeadUpdateFieldsWire.resolution` deliberate presence-preserving semantics.
   Reuse `deserialize_present_option` and the same omit-versus-explicit-null serde
   attributes as the event wire so omitted means leave unchanged while JSON `null` means
   clear. Add serialization and mutation tests proving omitted, clear, and set values
   round-trip without collapsing to a silent no-op, including the PyO3-facing payload
   path.

3. Make malformed workspace-claim suffixes fail closed. Parse a valid RUNNING row even
   when it contains unknown trailing pipe fields, retain those fields when a transfer
   rewrites the row, and recognize the existing `YYYYMMDD_HHMMSS` timestamp shape. Keep
   allocation conservative: such a row must appear in listing, reject an explicit
   duplicate claim, and occupy its workspace during automatic allocation. Add Rust
   parser, transfer, duplicate-guard, and allocator regressions.

4. Promote bead relocation data from a diagnostic string to typed publication state.
   Carry relocation pairs from the Rust merge result through Python conflict resolution,
   repository integration, managed-sync, and push outcomes, composing chains if a
   bounded push retry relocates the same create more than once. When the rebased local
   create/checkpoint commit is relocated, rewrite its bead IDs before the successful
   push so the durable commit subject describes the stored IDs.

5. Add a shared post-publication created-ID resolver and use it at every production
   create surface: `sase bead create`, typed flag creation, the ACE Beads action, and
   epic/phase graph creation followed by `sase bead work`. Re-open the committed store
   after publication, map the epic plus descendants and dependencies, update the plan
   link and launch inputs to the relocated IDs, and fail safely if the published issue
   cannot be resolved. Extend the two-clone collision integration test to assert both
   stored beads, corrected stdout/result payloads, corrected commit subjects, and a
   follow-up note/update on each returned ID affecting only its creator's bead. Add
   focused CLI, ACE, flag, and epic-launch tests for the non-collision and relocation
   paths.

## Verification and closeout

1. Run `just install` before checks, then iterate with focused Rust tests for
   `agent_scan::index`, bead mutation serde, workspace claims, and event relocation; run
   focused Python tests for the scan facade/lifecycle/CLI and every bead-create surface
   plus sync-conflict recovery.
2. Run `just check` in `sase-core`. Run `just check` in the primary `sase` repository;
   if either lane expands into a long full-repository check, hand it to `/sase_monitor`
   with `TESTING`/`TESTED` statuses and a concrete continuation prompt.
3. Re-read the four task beads for concurrent state changes. Append four separate
   close-ready evidence blocks to `sase-rm.1`, naming the cause, changed files, and
   exact passing tests for `sase-kh`, `sase-mu`, `sase-oi`, and `sase-qa`.
4. Run `sase bead epic-symbols sase-rm.1`. Resolve every leftover symbol or re-key its
   Justfile ownership to the parent epic or an open later phase. Then close only the
   phase with `sase bead close sase-rm.1 --note "<what was verified>"`; do not close
   `sase-rm` or any assigned task bead.
