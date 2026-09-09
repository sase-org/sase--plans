---
tier: epic
title: Timestamped bead notes
goal: "Every bead note carries a real timestamp and author as structured data, no write
  path can produce an untimestamped or clobbered note, and `sase bead show` renders the
  note log as a dated, attributed, per-entry section.

  "
phases:
  - id: core
    title: Structured note log in sase-core
    depends_on: []
    size: medium
    description: "core: replace the free-text `notes` blob with an event-derived list of
      timestamped note records in sase-core, including the legacy-blob recovery parser
      and the text projection every existing reader keeps using.

      "
  - id: pyapi
    title: Python model, projections, and text consumers
    depends_on:
      - core
    size: medium
    description: "pyapi: carry the structured note list through the Python `Issue`
      model, JSONL codec, SQLite mirror, and every read-only consumer via `notes_text`.

      "
  - id: show
    title: NOTES rendering in `sase bead show`
    depends_on:
      - pyapi
    size: medium
    description: "show: add the shared note presentation module and render one dated,
      attributed, ordinal-addressed block per note in `sase bead show`, the ACE bead
      detail pane, and the detail/query JSON payloads.

      "
  - id: cli
    title: Append-only write surface
    depends_on:
      - pyapi
    size: small
    description: "cli: retire `sase bead update --notes` behind a teaching error and add
      a batch `sase bead update --note` append that mirrors `sase bead close --note`.

      "
  - id: repair
    title: Note edit and retraction
    depends_on:
      - show
      - cli
    size: medium
    description: "repair: add `sase bead note --edit` and `--remove` with their own
      events, an `edited` render marker, and drop the clobbering Notes field from the
      ACE bead editor modal.

      "
  - id: docs
    title: Documentation and memory
    depends_on:
      - repair
    size: small
    description:
      "docs: update the bead docs, the CLI reference tables, and the sase_beads memory
      note to describe the append-only timestamped note log."
proposed_by: bbugyi200.athena.0ct
bead_id: sase-t2
create_time: 2026-09-09 19:51:51
status: wip
---

- **PROMPT:**
  [prompts/202608/timestamped_bead_notes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/timestamped_bead_notes.md)
- **BEAD:**
  [sase-t2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-t2/README.md)

# Plan: Timestamped bead notes

## Goal

Bead notes today are one free-text string. `sase bead note` prepends a
`[<utc> · <actor>] ` marker by string concatenation, and `sase bead update --notes`
replaces the whole field with whatever text it is given — untimestamped, unattributed,
and destroying every earlier note (the reason `sase bead history --lost-notes --restore`
exists at all).

After this epic a bead note is a record, not a substring:

- Every note carries a stored `timestamp` and `author`. There is no write path that can
  produce one without them.
- Nothing silently overwrites the log. Appending is the only ordinary write; editing and
  retracting are explicit, separately-evented operations that keep the original in
  `sase bead history`.
- `sase bead show` renders one dated, attributed, ordinal-addressed block per note.

## Design

### The record

```rust
// crates/sase_core/src/bead/wire.rs
pub struct BeadNoteWire {
    pub id: String,        // stable identity, derived from the creating event
    pub timestamp: String, // RFC-3339 UTC, the instant the note was written
    pub author: String,    // the actor that wrote it
    pub text: String,      // the note body, trimmed, never empty
    pub edited_at: Option<String>,
    pub edited_by: Option<String>,
}
```

`IssueWire.notes` becomes `Vec<BeadNoteWire>`. It follows the shape already proven by
`plus_one_evidence` and `close_history`: a
`#[serde(default, skip_serializing_if = "Vec::is_empty")]` list on the wire, a validated
record with its own `validate()`, and a matching frozen dataclass on the Python side.

`id` is the `event_id` of the `NoteAppended` event that created the note. Entries
recovered from a legacy blob get `<event_id>#<k>` for the k-th entry that event
produced. Ids are therefore a deterministic function of the event stream: the same store
always replays to the same ids. Ordinals shown in `sase bead show` (`#1`, `#2`, …) are
positional labels for humans; the CLI resolves an ordinal to an id before writing any
event, so an edit or retraction can never land on the wrong note.

### The text projection

Every surface that legitimately wants a blob keeps getting one, from a single derived
helper (`notes_text()` in Rust, an `Issue.notes_text` property in Python) that
reproduces exactly today's format:

```
[<timestamp> · <author>] <text>

[<timestamp> · <author>] <text>
```

That is the same string `appended_note_text` builds today, so search indexes, bead
pages, gate previews, `sase bead show --field notes`, and `--lost-notes` keep their
current bytes for every bead whose notes were written by `sase bead note`. This is what
keeps the migration cheap and auditable: the structured list is new, the text consumers
are untouched.

### Migration

`beads/events/**` is canonical and every load reduces it (`MutableStore::load` →
`reduce_event_streams`), so the note log is derived, not migrated in place. There is no
data conversion step and no store rewrite — the new reducer simply produces records
where the old one produced concatenated text.

Two reducer branches feed the list:

- `NoteAppended { entry }` → one record. The timestamp and author come from the event
  envelope, which already carries the true instant and actor. **Every note ever appended
  recovers its real timestamp**, not an approximation.
- `IssueUpdated { fields: { notes: Some(text) } }` (legacy events only) → the blob is
  parsed back into records by
  `parse_legacy_note_blob(text, event_timestamp, event_actor)`:
  - Split the blob into paragraphs on blank lines, matching how `appended_note_text`
    joins them.
  - A paragraph beginning `[<ts> · <actor>] ` where `<ts>` parses as RFC-3339 starts a
    new record with that recovered timestamp and author; its body runs to the next such
    marker.
  - A marker whose timestamp does not parse is _not_ treated as a header. The parser
    never invents structure from ambiguous text.
  - Any text before the first marker becomes one record attributed to the update event's
    own timestamp and actor — the honest answer, since that is when and by whom that
    text was written into the field.

`IssueCreated { issue }` carries a whole `IssueWire`; its `notes` deserializes through a
custom deserializer that accepts either the new array or a legacy JSON string, running
the same parser with the creation event's timestamp and actor. The same deserializer
covers `import_issues_from_jsonl`, so a legacy store with no event directory still
opens.

### Verification of the migration

`core` ships `tools/check_bead_note_migration`, a read-only tool that opens a bead
store, and for every bead classifies `notes_text(new)` against the old `notes` string
recorded in `issues.jsonl` before the change:

- `identical` — byte-for-byte equal (expected for every bead written by
  `sase bead note`).
- `attributed` — differs only by `[<ts> · <author>] ` headers the new code added to
  previously bare prose, and the note bodies concatenate back to the original.
- `differs` — anything else.

**The phase is not done until `differs` is zero on the real bead store and on the
fixtures.** This is the acceptance gate for the whole migration.

### Write paths after this epic

| Command                                    | Behavior                                           |
| ------------------------------------------ | -------------------------------------------------- |
| `sase bead note <id> <text>`               | append one record (unchanged)                      |
| `sase bead update <ids…> -n/--note <text>` | append the same record to every named bead         |
| `sase bead close <ids…> -n/--note <text>`  | append then close (unchanged)                      |
| `sase bead update --notes`                 | **removed**; hard error naming `sase bead note`    |
| `sase bead note <id> -e/--edit <n> <text>` | rewrite record `#n`, stamp `edited_at`/`edited_by` |
| `sase bead note <id> -x/--remove <n>`      | retract record `#n`; `sase bead history` keeps it  |

`--notes` becomes a tombstone rather than silently changing meaning. Silently turning a
replace into an append would be a semantic change a caller could not see; an error that
names the right command teaches instead. Nothing in `src/sase/xprompts/` uses `--notes`,
so only docs and the memory note reference it.

### Rendering

```
NOTES (3)
  #1 · 2026-08-22 09:14:02 EDT · 2d ago · bryan
     Reproduced on a clean tree; the second stream is dropped on replay.

  #2 · 2026-08-23 17:02:44 EDT · 1d ago · sase-a1-phase-2
     PROPOSED FOLLOW-UP: extract the note parser — two callers need it.

  #3 · 2026-08-24 08:30:10 EDT · 6h ago · bryan · edited 09:02:41 EDT
     Verified with `just check-full`; suite green.
```

- Chronological, oldest first: a work log reads top to bottom, and it preserves the
  order the blob already had.
- Ordinals are the addressing scheme `--edit` and `--remove` take, which is why they are
  rendered rather than decorative.
- Bodies keep going through `prose_lines`, so markdown and fenced-code highlighting,
  wrapping, and the no-color style levels all behave exactly as they do today.
- Timestamps render through `bead_instant_label` / `bead_age_label`, so they land in the
  configured timezone and match every other bead time on screen.
- The section is omitted entirely when a bead has no notes, as today.

`bead_note_label()` takes `relative: bool` for the same reason `close_record_label`
renders no relative time: it is shared with `bead_pages/rendering_identity.py`, whose
bytes are persisted and re-derived. The CLI passes `relative=True`; persisted and
gate-preview surfaces pass `relative=False`.

### No feature flag

Checked against `sase/memory/sase_flags.md`. A flag is for behavior that reaches users
before it is ready, or for a deprecation whose old branch must stay reachable. Neither
applies: the note log is unconditional once it lands, and `update --notes`'s old branch
must _not_ stay reachable — `sase bead note --edit`/`--remove` is its replacement, so
keeping the clobbering branch alive would defeat the epic. Land it unflagged.

### Out of scope

- A note-count chip on `sase bead list` rows.
- Any change to `sase bead history` output beyond keeping it correct.
- Reflowing already-generated bead pages; they regenerate on their next write.

---

## Structured note log in sase-core

Work in the linked `sase-core` repo, opened with the `/sase_repo` skill.

1. **`crates/sase_core/src/bead/wire.rs`** — add `BeadNoteWire` with `validate()`
   (non-blank `id`, `timestamp`, `author`, `text`; `edited_at` and `edited_by` are both
   present or both absent). Change `IssueWire.notes` to `Vec<BeadNoteWire>` with a
   custom deserializer accepting the legacy JSON string. Add
   `notes_text(&[BeadNoteWire]) -> String` reproducing the current `appended_note_text`
   join exactly. Call `note.validate()` from `IssueWire::validate`.

2. **`crates/sase_core/src/bead/events.rs`** — retire `appended_note_text` in favor of
   pushing a record. `NoteAppended` builds the record from the event envelope.
   `apply_update_event_fields` runs `parse_legacy_note_blob` for the legacy
   `fields.notes` branch; that branch stays readable forever because old events are
   immutable, but new mutations no longer emit it. Add
   `parse_legacy_note_blob(text, timestamp, actor) -> Vec<BeadNoteWire>` per the Design
   section, with unit tests covering: pure appended blobs, bare prose, mixed
   prose-then-markers, a marker with an unparseable timestamp, a marker-looking line
   mid-paragraph, empty and whitespace-only blobs, and CRLF input.

3. **`crates/sase_core/src/bead/mutation.rs`** — `append_note_to_store` pushes a record
   instead of rewriting a string. Its five call sites (`append_issue_note`, `close` with
   `--note`, `snooze`, the `+1` wake note, and the batch close path) need no behavioral
   change. Reject `notes` as a settable field in the update request path so no new
   `IssueUpdated{notes}` event can ever be written.

4. **Readers** — `search.rs` indexes `notes_text(&issue.notes)` so query semantics are
   byte-identical; `history.rs` `lost_notes_from_streams` compares `notes_text` (the
   finding can no longer occur going forward but the historical repair stays correct);
   `cli.rs` renders the NOTES block per record and `--field notes` returns `notes_text`;
   `jsonl.rs` import/export and `compare_issues_canonically` handle the list.

5. **`crates/sase_core_py/src/lib.rs`** — the `bead_append_note` payload now carries the
   record list. Update the module docstring signature block.

6. **`tools/check_bead_note_migration`** (in the `sase` repo, read-only, no Python API
   change needed beyond reading `issues.jsonl`) implementing the `identical` /
   `attributed` / `differs` classification from the Design section.

**Done when** `cargo test` passes in sase-core, `just check` passes in the sase repo
against the locally-built binding, and `tools/check_bead_note_migration` reports zero
`differs` on the live bead store.

## Python model, projections, and text consumers

1. **`src/sase/bead/model.py`** — add a frozen `BeadNote` dataclass mirroring
   `BeadNoteWire`, with `validate()`. Change `Issue.notes` to `list[BeadNote]` and add a
   `notes_text` property returning the flattened projection. Validate every note from
   `Issue.validate`.

2. **Projections** — `src/sase/bead/jsonl.py` encode/decode (accepting the legacy string
   on read), `src/sase/bead/db.py` and `src/sase/bead/_db_rows.py` for the SQLite
   mirror, and an additive idempotent migration in `src/sase/bead/_db_migrations.py`
   following `_migrate_add_patch_metadata`. Store the list as JSON text in the `notes`
   column, exactly as `plus_one_evidence TEXT NOT NULL DEFAULT '[]'` already does, and
   mirror the DDL change in `src/sase/bead/_db_schema.py` and
   `crates/sase_core/src/bead/schema.rs`.

3. **`src/sase/core/bead_mutation_facade.py`** — decode the note list in
   `_issue_payload` / `issues_from_list`. Refresh `tests/contract_manifest.txt` with
   `tools/refresh_contract_manifest` if the binding contract moves.

4. **Text consumers switch to `notes_text`**, no behavior change:
   `src/sase/bead_pages/rendering_identity.py`,
   `src/sase/scripts/_bead_task_triage_gates.py` (all three payload sites),
   `src/sase/bead/_task_gate_preview.py`, `src/sase/bead/_flag_gate_preview.py`,
   `src/sase/bead/_snooze_gate_preview.py`,
   `src/sase/integrations/_mobile_helper_beads.py`, `src/sase/bead/work.py`,
   `src/sase/ace/tui/widgets/artifacts/beads_filtering.py`, and
   `src/sase/ace/tui/models/agent_associated_plan.py` (`normalize_bead_notes`).
   Gate-preview and bead-page bytes must not move for any bead whose notes were written
   by `sase bead note`; assert that in tests.

**Done when** `just check` passes, gate-preview and bead-page snapshot tests are
unchanged, and no module outside the bead package touches `Issue.notes` as a string.

## NOTES rendering in `sase bead show`

1. **`src/sase/bead/note_presentation.py`** (new) — the shared vocabulary, matching
   `plus_one_presentation.py` and `reopen_presentation.py`: `NOTE_ACCENT = "#AFAF87"`,
   the derived rich and CLI styles, `NOTE_SECTION_LABEL = "NOTES"`,
   `NOTE_EDITED_MARKER`, `bead_note_label(note, ordinal, *, relative: bool)`, and
   `bead_note_search_text(notes)`.

2. **`src/sase/bead/cli_detail_sections.py`** — add
   `render_bead_note_lines(issue, *, palette, style, wrap)` producing the layout in the
   Design section: `NOTES (<n>)` header, one accented label line per record, body
   through `prose_lines(indent="     ")`, and a blank line between records. Wire it into
   `src/sase/bead/cli_detail_render.py` in place of the current blob block.

3. **JSON payloads** — `src/sase/bead/cli_detail_json.py` and
   `src/sase/bead/cli_query_render.py` emit `"notes"` as the list of records and add
   `"notes_text"` with the flattened form. This is a deliberate breaking change to the
   `--format json` shape; call it out in the phase's bead notes so the `docs` phase
   documents it.

4. **`src/sase/ace/tui/widgets/artifacts/beads_detail.py`** — render the same per-record
   structure as Markdown (`### <label>` + body) instead of dumping the blob under
   `## Notes`.

5. **Tests** — extend `tests/test_bead/test_cli_show.py`, `test_cli_show_style.py`,
   `test_cli_show_style_wrap.py`, and `test_cli_show_json.py`. Cover: zero notes
   (section absent), one note, many notes, an author with no timestamp recoverable, a
   note whose body contains a fenced code block, a note body that looks like a
   `[ts · actor]` marker, and every `DetailStyle` level including no-color.

**Done when** `just check` passes and `sase bead show` on a real multi-note bead renders
every note with its own date, age, and author.

## Append-only write surface

1. **`src/sase/main/parser_bead_lifecycle.py`** — in the `update` parser, replace
   `-n/--notes` with `-n/--note`, help text
   `Append this attributed note to each issue; <@path reads it>`, matching
   `sase bead close --note`. Keep a hidden `--notes` that argparse still accepts so the
   handler can emit the teaching error rather than an opaque `unrecognized arguments`.
   Keep option registration alphabetically sorted per `sase/memory/cli_rules.md`.

2. **`src/sase/bead/cli_crud_update.py`** — `--notes` exits non-zero with
   `Error: --notes was removed; use \`sase bead note <id> "<text>"\` to append, or
   \`sase bead note <id> --edit <n>\` to correct an existing
   note.` `--note`appends one record to every named bead inside the same mutation and commit, reusing the batch-append path`close
   --note`already exercises, and reports each bead in the existing`✓ Updated`/`·
   Unchanged` output.

3. **Tests** — extend `tests/test_bead/test_cli_update_bulk.py` and
   `tests/test_bead/test_cli_note.py`: the tombstone error text and exit code, a
   multi-bead append writing one record per bead with one commit, and an append combined
   with other field updates in a single invocation.

**Done when** `just check` passes and no code path in either repo can write an
untimestamped note.

## Note edit and retraction

1. **sase-core** — add `NoteEdited { note_id, text }` and `NoteRemoved { note_id }` to
   `BeadEventOperationWire` / `BeadEventPayloadWire`, their reducer branches, and
   `edit_issue_note` / `remove_issue_note` mutations that resolve a `note_id`, reject an
   unknown one, and stamp `edited_at`/`edited_by` on an edit. Follow whatever
   `BEAD_EVENT_SCHEMA_VERSION` precedent the existing operations set when they were
   added. Expose `bead_note_edit` and `bead_note_remove` bindings in
   `crates/sase_core_py/src/lib.rs`.

2. **`src/sase/main/parser_bead_lifecycle.py`** — `sase bead note` gains mutually
   exclusive `-e/--edit N` and `-x/--remove N`; `text` relaxes to `nargs="*"` with
   handler-side validation (`--edit` requires text, `--remove` forbids it, neither
   requires it as today). `N` is the 1-based ordinal from `sase bead show`; the handler
   resolves it to the record's `id` before mutating, and the help text says ordinals are
   positional so a caller re-reads `show` after a retraction.

3. **`src/sase/bead/cli_crud_evidence.py`** and `src/sase/bead/_project_mutations.py` /
   `src/sase/core/bead_mutation_facade.py` — the two new mutations, out-of-range and
   unknown-id errors, and commit messages (`note-edit`, `note-remove`) through
   `require_mutation_commit_message`.

4. **Render** — `bead_note_label` appends `· edited <instant>` (and `by <author>` when
   the editor differs from the original author) when `edited_at` is set.

5. **`src/sase/ace/tui/modals/bead_editor_modal.py`** — remove the Notes label,
   `TextArea`, and the `notes` entry from `BeadEditorResult` / `changed_fields`. The
   editor can no longer clobber the log; the existing `action_beads_add_note` flow is
   the TUI's note surface. Update `tests/` coverage and any TUI snapshot that shows the
   editor.

6. **Tests** — edit preserves the original timestamp and author, retraction removes the
   record while `sase bead history` still replays it, ordinals resolve to the right
   record after an earlier retraction, and both refuse an out-of-range ordinal without
   writing an event.

**Done when** `just check` passes, and every operation that once needed `update --notes`
has an explicit, evented replacement.

## Documentation and memory

1. **`docs/beads.md`** — replace the `-n, --notes | Replace notes` row with
   `-n, --note | Append an attributed note to each issue`, rewrite the "Use
   `sase bead update --notes` for an explicit field replacement" paragraph, document the
   note record shape, the `--format json` `notes` / `notes_text` change, and the
   `--edit` / `--remove` repair path. Refresh the `sase bead note` and
   `sase bead history` sections so `--lost-notes` is described as a historical repair
   for stores that predate the append-only log.

2. **`docs/configuration.md`** — update the `sase bead update` option table row.

3. **`sase/memory/sase_beads.md`** — the "Notes And History" section currently says
   `update --notes` replaces the whole field. Rewrite it for the append-only timestamped
   log.

   **This edit needs the user's explicit permission in the implementing agent's own
   conversation.** A plan file is not permission — `sase/memory/gotchas.md` says so
   directly. Ask for it, and when it is granted, make the edit under `sase/memory/` and
   then run `sase memory init` to regenerate `AGENTS.md`, the provider shims, and the
   memory README. If permission is not granted, land everything else and file the memory
   update as a `memory` task bead through `/sase_new_task`.

**Done when** `just check-full` passes through `/sase_monitor` and the docs describe the
shipped behavior.
