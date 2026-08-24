---
tier: tale
title: Preserve legacy bead event prefixes during mutations
goal:
  Bead mutations append new events without rewriting any historical JSONL event bytes,
  including legacy note encodings.
size: medium
proposed_by: bbugyi200.athena.0cy
---

- **AGENTS:**
  - [bbugyi200.athena.0cy](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0cy.md)
- **COMMITS:**
  - [b5d7f3c](https://github.com/sase-org/sase-core/commit/b5d7f3cb317a49968f6e18add6c5378e94061635)
    — fix(bead): preserve legacy event-stream byte prefixes when appending JSONL writes

# Preserve legacy bead event prefixes during mutations

## Problem

The structured-note rollout made legacy note data readable but not mutation-safe.
`IssueWire` now deserializes an `issue_created` payload's legacy `notes` string into
`Vec<BeadNoteWire>` and serializes the current structured form. Meanwhile,
`MutableStore::save` calls `write_event_store_changed`, which serializes every event in
a changed stream, including the already-published prefix.

As a result, the first mutation of a legacy stream rewrites historical JSON: an empty
`payload.issue.notes` field is omitted, and a non-empty string becomes structured note
records. The Python append-only guard correctly rejects that as an ancestor rewrite.
This is what failed agent `sase-t2.2`: its source commits landed, commit-time bead
auto-close attempted to append a close and note, the guard reported that ancestor event
1 had removed `payload.issue.notes`, and the commit finalizer ultimately found a dirty
`issues.jsonl` projection in the beads sidecar.

Historical event bytes are immutable. Compatibility belongs at the read/reduce boundary;
an ordinary mutation must append new events without normalizing old ones.

## Implementation

### Rust core: append without reserializing the prefix

In `sase-core`, change the mutation-specific event-store writer in
`crates/sase_core/src/bead/jsonl.rs` so an existing changed stream is updated as an
append-only raw JSONL prefix:

- Read and retain the existing stream bytes before writing.
- Parse the existing events and require them to equal the same-length prefix of the
  desired typed stream. Refuse a shorter stream or any semantic prefix change with a
  clear `BeadError`; do not modify the file on refusal.
- Build the replacement atomically from the untouched existing bytes plus serialized
  JSONL for only the new tail events. Handle a missing trailing newline without changing
  any existing event object.
- Continue fully serializing a brand-new/imported stream for which no stream file
  exists, and continue updating the manifest from the complete typed stream set.
- Keep unrelated stream files byte- and mtime-stable.

Keep full-store rewrite behavior separate for explicit migration/repair operations if
they need it; the normal `MutableStore` save path must use the append-preserving
contract. Do not weaken the Python append-only integrity guard to accept equivalent
rewrites—the guard exposed real corruption of the event-log contract.

### Rust regression coverage

Extend the bead JSONL/mutation tests to cover both the compatibility case and the writer
contract:

- Seed an existing stream whose `issue_created` event contains legacy `notes` text
  (cover at least the empty-string form that triggered `sase-t2.2`, plus non-empty text
  if practical), perform a public mutation such as append-note or close, and assert the
  original raw line is an exact byte prefix of the resulting file.
- Reload and reduce the stream to prove the legacy note still projects into structured
  records and the newly appended mutation is present.
- Assert a changed or shortened ancestor prefix is rejected and leaves the original
  stream untouched.
- Update the existing selected-stream writer test to append a new event rather than
  relying on replacement of an existing event; retain its assertions that unselected
  streams are unchanged and new streams are created.

### SASE end-to-end regression

Add a focused test in the main `sase` repository using a temporary Git-backed bead store
with the legacy event shape. Exercise the real Rust-backed bead mutation and the Python
commit-integrity preparation path used by `sase bead close`/commit-time auto-close.
Assert that:

- the close or note mutation succeeds;
- the original event prefix is unchanged and only valid tail events are added;
- `prepare_event_streams_for_commit` accepts the result;
- the resulting commit includes canonical event state and the derived projection; and
- the repository is clean afterward, with replay reporting the expected final bead
  status and structured notes.

The fixture must be isolated through the supported bead-store test helpers rather than
relying on a workspace project configuration or touching a live sidecar.

## Verification

In `sase-core`, run the focused bead JSONL/mutation tests and the repository's normal
check target. In `sase`, run `just install`, the focused stream-integrity and new
end-to-end regression tests, then `just check`. Confirm the regression fails on the
pre-fix writer with the same `removed payload.issue.notes` diagnosis and passes with the
append-preserving writer.

Do not repair or close the live `sase-t2.2` bead as part of the code change. Once the
fix is installed, recover that epic through the normal failed-agent/bead-work workflow
so its canonical close event is appended by the fixed writer.
