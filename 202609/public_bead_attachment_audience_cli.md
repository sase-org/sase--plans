---
tier: tale
title: Add audience decisions to bead attachment authoring
goal:
  Bead attachment CLI authoring safely records and routes public or private intent under
  the beta flag, with provenance, scanning, and confirmation.
size: medium
proposed_by: bbugyi200.athena.sase-1d5.3
bead: sase-1d5.3
create_time: 2026-09-30 09:01:47
status: wip
---

- **PARENT:**
  [202609/public_bead_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/public_bead_attachments.md)
- **BEAD:**
  [sase-1d5.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d5/sase-1d5.3.md)

# Public bead attachment audience CLI

Implement and close phase bead `sase-1d5.3` only. The design of record is
`plan:202609/public_bead_attachments.md`, especially its shared specification and
Phase 3. The research basis is
`research:202609/bead_attachment_audience/bead_attachment_audience.md`. Phase
`sase-1d5.1` has already provided the decision table, scanner, and wire in `sase-core`;
use those bindings from Python rather than duplicating policy. Phase `sase-1d5.4` owns
the public sidecar implementation. Until it lands, public attachments must retain public
intent and queue for that missing store.

## Implementation

1. Create the `public_bead_attachments` beta flag with `sase flag new` and the enabled,
   disabled, and removal text specified in Phase 3. Add its printed registry entry. Gate
   only authoring decisions and `-W/--public`; keep existing read and drain paths usable
   with the flag off. Add both-state tests.
2. Build lazy Python provenance helpers under `src/sase/bead/attachments/` for
   owner-only mode, checkout root and ignore state, a byte-for-byte comparison against
   the remote-tracking default branch, workspace and managed scratch roots, agent
   identity and run window, and configured bead-store visibility. Resolve the origin
   through `parse_hosted_git_remote`; probe anonymous smart-HTTP with no credentials or
   netrc, a three-second timeout, and the specified public/private/unknown cache TTLs.
   Fail private on uncertainty.
3. Extend the existing authoring pipeline after CAS ingestion to classify, scan the CAS
   object through `attachment_scan_file`, gather facts, and call
   `attachment_audience_decision`. Keep the early sensitive-path refusal. Make `-S`
   private only; refuse it with `-W`. For `confirm`, prompt only on a human TTY, and
   allow `-y` for a human without a TTY. An agent's widening request must refuse with
   the reason and exact publish command before a bead event is written. Preserve private
   intent for an existing private descriptor with the same digest, using the extended
   core references query. Write local CAS audience metadata with rule, reason,
   explicitness, scanner version, and time; never put the reason in a descriptor or
   public bead record.
4. Preserve `visibility` in the Python attachment model, note codec, roster, and all
   five authoring handlers. Add `-K/--private`, `-W/--public`, and `-y/--yes` to `note`,
   `close -n`, `update -n`, `+1 -n`, and `attach`, with mutually exclusive and
   incompatible flag validation, sorted CLI help, and a dim ignored-flags warning when
   no attachment is present. Make the Rust bead fast path fall through for these flags.
   With the beta flag off, write no visibility field, accept `-K` harmlessly, and refuse
   `-W` with an enable hint. Include stdin attachments in the same decision and scan
   flow.
5. Add `bead.attachments.public_max_bytes` with a 25 MiB default and a 95 MiB upper
   bound in the default config, schema, and getter. Route by audience without
   cross-audience fallback; a missing public store queues the upload for that store.
   Keep private and local-only placement as specified. Update attachment echoes to show
   🌐/🔒, local-only reasons, destination or pending state, and eliminate the duplicate
   private label. Move the sase-core CI pin past the core audience commit after its
   Python binding is in use.
6. Add a short `Attachment visibility (beta)` section to `docs/beads.md` that explains
   the public meaning, private and local-only choices, the public cap, confirmation, and
   that private protects bytes but not names or prose.

## Verification and completion

- Test every decision rule through the Python CLI, duplicate-digest intent, agent
  refusal before mutation, human TTY and `-y` paths, both beta states, stdin, fast-path
  fallthrough, config bounds, remote HTTP status mapping and TTLs. Use synthetic
  secrets, injected environment and HTTP probes, and no network in tests.
- Open the agents sidecar with `sase repo open agents`. Scan published transcripts and
  cited file objects with the core scanner, outputting only aggregate counts by hit
  kind. Confirm credential-pattern hits cover at least the research report's 13 known
  files and record counts in a note on `sase-1d5.3`; never print values, matching lines,
  or file contents.
- Read `lint_and_test.md` through `sase memory read`, then run `sase tool run check`. If
  a failure reproduces identically on the clean base, record a `PROPOSED FOLLOW-UP:`
  note and continue. Record any other discovered follow-up on this phase bead, without
  creating task beads.
- Run `sase bead epic-symbols sase-1d5.3`; resolve all entries or re-key each Justfile
  line to a still-open bead. Close only `sase-1d5.3` with
  `sase bead close sase-1d5.3 --note "<what was verified>"`. Do not close its parent or
  any ancestor.
