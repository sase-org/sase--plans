---
tier: epic
title: Legacy note bytes survive bead conflict resolution
goal: "Resolving a bead event-stream conflict never rewrites a historical event's legacy
  `notes` encoding, so a store that predates the structured note log can be rebased and
  published without tripping the append-only stream guard, and the memory note and
  templates describing bead notes match the shipped append-only behavior.

  "
phases:
  - id: resolver
    title: Preserve ancestor event bytes through the conflict resolver
    depends_on: []
    size: medium
    description:
      "resolver: stop the bead conflict resolver from re-encoding a conflicted stream's
      already-published events, so a legacy `notes` field survives a merge byte-for-byte
      and the append-only publish guard stays satisfied."
  - id: memory
    title: Correct the bead-notes documentation that still promises replacement
    depends_on: []
    size: small
    description:
      "memory: rewrite the generated bead memory note's Notes And History section and
      its packaged template for the append-only log, and fix the one stale code comment
      that still justifies itself with `--notes` overwrite semantics."
proposed_by: bbugyi200.athena.sase-t2.land
parent_bead: sase-t2
bead_id: sase-t2.7
create_time: 2026-09-09 19:50:43
status: wip
---

- **PROMPT:**
  [prompts/202608/legacy_note_bytes_in_conflict_resolution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/legacy_note_bytes_in_conflict_resolution.md)
- **PARENT:** [202608/timestamped_bead_notes.md](timestamped_bead_notes.md)
- **BEAD:**
  [sase-t2.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-t2/sase-t2.7.md)

# Plan: Legacy note bytes survive bead conflict resolution

## Goal

Epic `sase-t2` ("Timestamped bead notes") replaced the free-text `notes` blob with a
list of structured note records. Its Migration section promised that the note log is
_derived_, not migrated: "There is no data conversion step and no store rewrite."

That promise holds for reads and, since the `202608/preserve_legacy_bead_event_prefixes`
tale, for ordinary mutations. It does **not** hold for bead conflict resolution, which
still round-trips every conflicted stream through the typed Rust wire and writes the
re-encoded result back. This epic closes that last gap, and finishes the one
documentation change `sase-t2`'s own `docs` phase left undone.

## Phase `resolver`: Preserve ancestor event bytes through the conflict resolver

### The defect

`src/sase/bead/conflict_resolver.py` resolves a conflicted
`sdd/beads/events/streams/ <id>.jsonl` by reading the three merge stages, handing them
to `merge_event_streams_with_relocation()` (the
`bead_merge_event_streams_with_relocation` Rust binding), and writing
`outcome["merged"]` back through `_write_resolved_store()`.

The binding parses each stage into a typed `BeadEventStreamWire` and serializes the
merged result. Since `sase-t2`, `IssueWire.notes` is a `Vec<BeadNoteWire>` with a custom
deserializer that also accepts the legacy JSON string — so that round-trip is **lossy at
the byte level** for any `issue_created` event written before the epic:

- `"notes":""` — the field is dropped entirely on re-serialization
  (`skip_serializing_if = "Vec::is_empty"`).
- `"notes":"[<ts> · <actor>] first note"` — the string becomes
  `"notes":[{"id":"__legacy_note__#1","timestamp":"<ts>","author":"<actor>","text":"first note"}]`.

Reproduce it directly against the binding (both cases print `byte-stable=False`):

```python
import json
from sase.core.bead_conflict_facade import merge_event_streams_with_relocation

# Any pre-sase-t2 issue_created event will do; ~/sdd/beads has many with "notes":"".
ev = json.loads(open("<some>/events/streams/<id>.jsonl").readline())
sid = ev["issue_id"]
ev["payload"]["issue"]["notes"] = ""          # then repeat with a non-empty string
stream = {"stream_id": sid, "root_issue_id": sid, "events": [ev]}
base = {"stream_id": sid, "root_issue_id": sid, "events": []}

def enc(s):
    return "".join(
        json.dumps(e, separators=(",", ":"), ensure_ascii=False) + "\n"
        for e in s["events"]
    )

merged = merge_event_streams_with_relocation(base, stream, stream, None)["merged"]
print("byte-stable=", enc(stream) == enc(merged))
```

### Why it matters

Bead event bytes are immutable by contract, and the repo enforces that.
`src/sase/bead/_stream_integrity.py` compares each stream against its ancestor revision
and refuses to publish one whose ancestor events changed —
`_stream_integrity_messages.py` phrases it as
`cannot publish non-append-only bead event stream <id>: worktree rewrote ancestor event <n>`.
A conflict resolution that re-encodes event 1's `notes` field produces exactly that
refusal on the next publish, wedging the rebase it was supposed to unblock.

This is the same class of corruption that broke agent `sase-t2.2`'s commit and prompted
the `202608/preserve_legacy_bead_event_prefixes` tale. That tale deliberately scoped
itself to the `MutableStore::save` writer in `crates/sase_core/src/bead/jsonl.rs`; the
conflict resolver is a second, independent writer of the same files and was never
covered. Phase bead `sase-t2.3` recorded the gap as a `PROPOSED FOLLOW-UP` (note 3) and
proposed the fix taken here: preserve ancestor bytes rather than replaying them through
a re-encoder.

Non-conflicted streams are already safe: `_load_worktree_streams()` parses them with
`json.loads` and never sends them through the wire, so `_write_resolved_store()`'s
byte-identity skip still matches and leaves them untouched. Only the conflicted streams
regress, which is why this has stayed latent.

### Implementation

Work in `src/sase/bead/conflict_resolver.py`.

1. While reading the three merge stages, keep the **raw parsed dict** for every input
   event, keyed by `event_id`, from all three stages (`base`, local, upstream). Python's
   `json.loads` preserves key order, and `_write_resolved_store()` re-encodes with
   `json.dumps(..., separators=(",", ":"), ensure_ascii=False)`, so a raw dict written
   straight back reproduces the original line byte-for-byte.

2. After the merge returns, rebuild `outcome["merged"]["events"]` (and the relocated
   stream's events, if any): for each merged event, substitute the raw input dict when
   one exists for that `event_id` **and** the merge did not change the event's meaning.

   Compare on meaning, not on bytes — the raw dict is precisely what differs. Round-trip
   the candidate raw dict back through the same binding path the merge used (or compare
   the merged event against the normalized form of the raw dict) and substitute only on
   equality. Any event the merge genuinely rewrote must keep the merged form.

3. Relocation is the case that must **not** substitute. When a duplicate bead id is
   relocated, the merge rewrites `stream_id`, `issue_id`, and ids inside the payload
   while `event_id` may still look familiar; the equality check in step 2 is what
   protects this, so cover it with a test rather than special-casing it.

4. Extend `_write_resolved_store()`'s docstring: it already explains why the encoding
   must stay byte-identical to the Rust writer's, and should now also say that
   conflicted streams reuse their input events' raw dicts for the same reason.

Prefer this Python-side fix over teaching `sase-core` to remember a legacy `notes`
encoding. Round-trip fidelity would require a provenance marker on `IssueWire` that
exists only to reproduce a historical byte layout, and it would put migration
compatibility back into the wire type that `sase-t2` deliberately kept clean.

### Tests

Extend the conflict-resolver tests (`tests/` sibling of the module; find them with
`tools/select_tests --explain src/sase/bead/conflict_resolver.py`). Build the fixtures
from real pre-`sase-t2` event JSON so they keep testing the legacy shape after this
lands:

- A conflicted stream whose `issue_created` payload carries `"notes":""` resolves with
  that line byte-identical to its input.
- The same for a non-empty legacy `notes` string, and assert the resolved store still
  reduces to the recovered structured records (the compatibility path must survive; only
  the rewrite goes away).
- A stream with genuinely divergent events on both sides still merges: new tail events
  appear in the merged form, ancestor lines are unchanged.
- A relocated duplicate bead id keeps the merged (rewritten) events and is **not**
  substituted back to its input dict.
- The resolved store passes the append-only check in
  `src/sase/bead/_stream_integrity.py` for the legacy-notes case — the regression that
  actually motivates this phase.

**Done when** `just check` passes and resolving a conflict on a stream with a legacy
`notes` field leaves that event's bytes untouched.

## Phase `memory`: Correct the bead-notes documentation that still promises replacement

Two places still tell readers that `sase bead update --notes` replaces the note field.
It does not: `src/sase/bead/cli_crud_update.py` makes it a hard error that names
`sase bead note`. `docs/beads.md` and `docs/configuration.md` were already corrected by
`sase-t2`'s `docs` phase; these were missed.

1. **`sase/memory/sase_beads.md`**, "Notes And History" — currently reads
   "`sase bead note <id> \"<text>\"` appends an attributed entry atomically, while
   `update --notes` replaces the whole field". Rewrite it for the append-only
   timestamped log: notes are records with a stored timestamp and author,
   `sase bead note` and `sase bead update --note` both append, `--notes` is a removed
   tombstone that errors, `sase bead note --edit <n>` / `--remove <n>` are the repair
   path addressed by the ordinal `sase bead show` renders, and
   `sase bead history --lost-notes` is a historical repair for stores predating the log.

   **This edit needs the user's explicit permission in the implementing agent's own
   conversation.** `sase/memory/gotchas.md` is explicit that a plan file is not
   permission — asking is part of this phase, not a formality to skip. When it is
   granted, make the edit and then run `sase memory init` to regenerate `AGENTS.md`, the
   provider shims, and the memory README; that regeneration needs no separate
   permission. If permission is refused, land step 2 and file the memory update as a
   `task(memory)` bead through `/sase_new_task`, then say so on the phase bead.

2. **`src/sase/main/init_memory/templates/memory-sase-beads.template.md`** (line ~108)
   carries the same stale sentence, and `sase/memory/sase_beads.md` is generated from it
   — `sase init memory --check` compares the two. It is ordinary packaged source, not a
   memory file, but the two must move together or that check goes red. Make the same
   rewrite here and confirm `sase init memory --check` is clean afterward.

3. **`src/sase/vcs_provider/plugins/_git_commit_dispatch.py`**, `_amend_bead_changes`'s
   docstring, currently justifies not writing a commit SHA into bead notes with "and
   `--notes` overwrites existing notes rather than appending to them". The conclusion is
   still right but the reason is now backwards. Replace the second clause with the
   append-only rationale: the note log only appends, so a stale SHA record could not be
   taken back.

**Done when** `just check` passes, `sase init memory --check` is clean, and no file in
the repo still describes `sase bead update --notes` as replacing the note field.
