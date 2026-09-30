---
tier: tale
size: medium
title: Core attachment audience policy and scanner
goal:
  Give every SASE frontend the same fail-private attachment visibility decision, secret
  scan, and public object path through sase-core.
proposed_by: bbugyi200.athena.sase-1d5.1
bead: sase-1d5.1
create_time: 2026-09-30 02:09:30
status: wip
---

- **PARENT:**
  [202609/public_bead_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/public_bead_attachments.md)
- **BEAD:**
  [sase-1d5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d5/sase-1d5.1.md)

# Core attachment audience policy and scanner

## Context and scope

Implement only phase `sase-1d5.1` of the public bead attachments epic. The accepted
design is `plan:202609/public_bead_attachments.md`, especially its shared Wire, Decision
table, Scanner, Canonical extension, and Phase 1 sections. The cited research is
`research:202609/bead_attachment_audience/bead_attachment_audience.md`, especially
§4.2–4.3. Work in the linked `sase-core` checkout opened with
`sase repo open sase-core`; follow its `AGENTS.md`. Do not change the `sase` Python
repository, its core revision pin, other linked repositories, or memory files in this
phase. Later epic phases gather provenance in Python, route stores, and expose user
flags.

The current descriptor is in `crates/sase_core/src/note_attachment/manifest.rs`. The
existing policy helpers and wire types live there; classification is in `extensions.rs`,
note attachment bindings in `crates/sase_core_py/src/note_attachment/mod.rs`, and bead
queries in `crates/sase_core/src/bead/attachments.rs`. The event schema remains
version 1. Existing descriptors without visibility and unknown future visibility values
must reduce to private.

## Implementation

1. **Additive wire and queries.** Add snake-case `AttachmentVisibilityWire` with a serde
   fallback treated as private, an optional, omitted-when-none `visibility` field on
   `BeadNoteAttachmentWire`, and `effective_visibility()`. Carry effective visibility
   into roster and reference wire results. Expand `bead_attachment_roster` and
   `bead_attachment_references` to include attachments in initial `IssueCreated` notes
   and `TaskPlusOneRecorded` evidence as well as appended/edited notes. References must
   identify whether the source is a note or +1 evidence, preserve
   current-versus-historical pinning semantics, and remain deterministic. Since +1
   evidence lacks a note ID, choose and document a stable identifier/ordinal mapping
   rather than inventing a mutable note. Update all Rust descriptor literals and binding
   round trips as required. Keep descriptor validation, note token matching, and
   `BEAD_EVENT_SCHEMA_VERSION` unchanged. Test an old-shaped descriptor, unknown
   visibility, explicit public visibility, initial notes, edits/removals, and +1
   evidence.

2. **Pure ordered audience decision.** Add `AttachmentAudienceFactsWire` and
   `AttachmentAudienceDecisionWire` in the `note_attachment` domain, with wire
   enums/fields from the accepted plan. Implement `attachment_audience_decision` as
   first-match rules 0–9: private bead store; sensitive path; explicit
   private/local-only; size cap; scanner hit; private provenance; opaque type;
   unverified image/video; positive public evidence; fail private. Implement the SASE
   secret-file and zone tables in core, using `home` and `sase_home` facts. Include
   personal/config and SASE personal zones, plus publishable logs, tool-run logs, perf
   traces, and TUI screenshots. A public remote checkout takes precedence over
   overlapping personal-zone paths, but files outside the workspace need byte identity
   with the remote-tracking default branch for the already-public evidence rule. Treat
   missing/unknown remote facts and no run window as private. Apply the widening rules
   after the policy result: never widen a private bead store, sensitive path,
   known-value hit, or size cap; agent `public` over widenable private refuses with an
   actionable publish hint; human widening requires confirmation unless already
   confirmed. `allow_sensitive` plus `public` and local-only plus `public` refuse. Keep
   reasons generic and local, without any secret value.

3. **Streaming scanner.** Add
   `attachment_scan_file(path, max_bytes, env, home, sase_home)` and
   `ATTACHMENT_SCANNER_RULES_VERSION = 1`. Python will call it on the ingested CAS
   object only for text classes, including SVG. Return `clean`, `hit`, or `skipped`,
   with hit limited to kind, rule ID, and line, plus bytes scanned and rules version.
   Skip oversize, unreadable, and binary content. Match secret-named env values and
   trimmed SASE secret-file contents of at least 12 characters using direct
   `aho-corasick`; exclude numeric, boolean, and absolute-path values. Add
   high-precision credential-pattern and high-entropy assignment detectors, plus
   ten-line-window and structured-object env-dump detectors as specified by the epic.
   Bound memory for long lines and detect values split across read chunks. Never log,
   serialize, or return a matched value or surrounding line. Tests use synthetic
   token-like data assembled at runtime and injected env maps, never real credentials or
   network.

4. **Object naming and bindings.** Add the pinned MIME-to-canonical-extension table,
   `attachment_public_object_relpath(sha256, mime_type)`, and
   `attachment_object_digest_from_relpath` for extensionless and canonical-extension
   forms; validate digest, shard, and shape without changing tombstones or private
   object layout. Expose these and `attachment_audience_decision`,
   `attachment_scan_file`, and `attachment_scanner_rules_version` from
   `crates/sase_core_py/src/note_attachment/mod.rs` and register each binding. Ensure
   bead roster/reference bindings serialize the new visibility and source fields. Add
   Python binding tests for argument parsing, all exported names, and output round
   trips.

## Verification and handoff

- Add table tests for every decision rule, rule precedence, each widening outcome, and
  SASE path/zone membership. Include public checkouts under personal-zone directories,
  stdin, unknown remote, no run window, and skipped scans.
- Add scanner tests for each detector, chunk-straddling, long-line bounds, over-cap and
  binary skipping, and no matched text in output. Pin the extension table and both
  public/old path forms.
- Run targeted Rust and binding tests during development, then the linked core
  repository's required `sase tool run check` gate. Use the SASE monitor workflow if the
  gate exceeds the synchronous limit. Do not run bare cargo or `just check-full`.
- If a full-gate failure reproduces identically on the clean base tree, record
  `PROPOSED FOLLOW-UP:` on `sase-1d5.1` with any existing tracking bead, and close the
  phase anyway. Record any other out-of-scope findings as `PROPOSED FOLLOW-UP:` on this
  phase; do not create beads.
- Before closing, run `sase bead epic-symbols sase-1d5.1`. Resolve each remaining symbol
  or re-key its Justfile line to an open bead. Close only `sase-1d5.1` using
  `sase bead close sase-1d5.1 --note "<specific verification evidence>"`; never close
  its parent or any ancestor.
