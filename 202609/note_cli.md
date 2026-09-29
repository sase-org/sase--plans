---
tier: epic
title: Bead note attachment CLI
goal: 'With the bead_note_attachments beta flag on, inline @path references in bead
  note text and sase bead attach snapshot files into the local content-addressed store
  and persist @attachment tokens plus a manifest. With the flag off, note text keeps
  today''s behavior. show, read, JSON, history, list, and path render those snapshots
  as text. Bytes never enter the bead store, and this work does not upload, fetch,
  or draw images.

  '
phases:
- id: note_authoring
  title: Flag, authoring service, and note verb
  depends_on: []
  size: medium
  description: 'note_authoring: add the beta flag, the note attachment model, and
    the shared authoring service, and wire them into sase bead note and the TUI add-note
    modal.'
- id: attach_verbs
  title: Close, update, +1, and attach
  depends_on:
  - note_authoring
  size: medium
  description: 'attach_verbs: run the authoring service from close -n, update -n,
    and +1 -n, and add sase bead attach including stdin.'
- id: read_surface
  title: Text rendering, list/path, and beta docs
  depends_on:
  - note_authoring
  - attach_verbs
  size: medium
  description: 'read_surface: render text chips, the attachments block, history, list,
    and path, and document the beta commands.'
proposed_by: bbugyi200.athena.sase-1ck.4
parent_bead: sase-1ck.4
create_time: 2026-09-29 12:24:43
status: done
bead_id: sase-1ck.4.1
---

- **PROMPT:** [prompts/202609/note_cli.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/note_cli.md)
- **BEAD:** [sase-1ck.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.4.1.md)

# Plan: Bead note attachment CLI

This is the implementation plan for parent phase **sase-1ck.4** (`note_cli` in
`plan:202609/bead_note_attachments.md`). Phases 1–3 of that epic are closed: the
grammar, names, classification, wire, reducer, mutation kwargs, and the local CAS are
already on sase-core pin `43f744be` and in `src/sase/bead/attachments/`. Do not edit
sase-core and do not bump the pin.

Each phase below is one direct implementation. Close only that phase's own bead. Do not
close `sase-1ck` or `sase-1ck.4`. This epic's land agent owns the cascade that closes
`sase-1ck.4`. Record discovered work as `PROPOSED FOLLOW-UP:` on the phase bead. Do not
create beads.

## Shared contract

Call the installed bindings. Do not reimplement the grammar.

| Binding                                               | Python signature                                                           |
| ----------------------------------------------------- | -------------------------------------------------------------------------- |
| `scan_note_attachment_refs`                           | `(text, roster_names) -> scan dict`                                        |
| `compose_note_attachment_text`                        | `(text, scan, assigned_names) -> stored text`                              |
| `note_attachment_source_text`                         | `(stored, manifest_names) -> source text`                                  |
| `sanitize_attachment_name` / `unique_attachment_name` | candidate, digest, existing `{name, sha256}` list                          |
| `classify_attachment`                                 | `(name, head: bytes) -> {mime_type, class}`                                |
| `attachment_sensitive_path_reason`                    | `(path, home, extra_patterns=None) -> str or None`                         |
| `bead_attachment_roster`                              | `(beads_dir, issue_id) -> current roster`                                  |
| `bead_append_note` / `bead_note_edit`                 | optional `attachments` list; edit `None` keeps the manifest, `[]` detaches |
| `bead_close` / `bead_plus_one`                        | optional `note_attachments`                                                |

`BeadNoteAttachmentWire` is `{name, sha256, size_bytes, mime_type, image?, origin?}`.
`image` is `{width, height}` only when `probe_image` returns dims. No paths, store
locations, or availability belong in the wire, in `issues.jsonl`, or in the SQLite note
JSON. `notes_to_dicts` omits `attachments` when the tuple is empty so attachment-free
notes stay byte-identical.

The mental model, copied into note help and `docs/beads.md`:

> **`@` in bead notes.** Inside note text, `@<path>` attaches a snapshot of that file.
> The bead keeps the exact bytes on every machine, even after the file is gone. Accepted
> forms: `@./shot.png`, `@~/logs/crash.log`, `@/tmp/trace.json`, `@docs/plan.md`,
> `@"name with spaces.png"`, or a bare `@name.ext` for a known file type. Write `@@`
> where you need a literal `@` that would otherwise start a reference. `me@host`,
> `@large`, and `@research:…` citations never need escaping. A note argument that is
> _only_ `@<file>` still reads the note's text from that file, and any `@<path>` inside
> that text then attaches. To attach files without prose, use `sase bead attach`.

Resolution uses the invocation cwd, including text loaded from a whole-argument `@file`,
with `~` expanded. `Path.resolve()` follows symlinks, then ingest the resolved regular
file. `ingest_path` opens with `O_NOFOLLOW`, so pass the resolved path rather than
changing the CAS. Sensitive paths use the core policy plus
`bead.attachments.sensitive_patterns` unless `-S/--allow-sensitive`. Directories get the
existing `tar czf` hint. FIFOs, devices, and sockets are refused. Collect every problem
and write nothing to the bead store.

There is no shared store in this epic. Every successful echo says the bytes are local.
Availability is only `cached` (object present in the local CAS, view materialized) or
`unavailable` (no local object, prose still renders). Do not add `-L`, upload, fetch,
outbox, rclone, `--images`, kitty, cell thumbnails, pager links, `attachment open`,
purge, doctor, prune, or bead-page rendering. Those belong to later phases of
`sase-1ck`.

Flag checks call `current_flags().enabled(FeatureFlag.bead_note_attachments)` at the use
site. No import-time resolution and no cached snapshot. Tests use
`override_flags(bead_note_attachments=True|False)`.

`-S/--allow-sensitive` is free on `note`, `close`, `update`, and `+1`. Do not add it to
`close -r`, `snooze -r`, descriptions, or `create -w`.

Before the first code change, import `scan_note_attachment_refs` from the installed
`sase_core_rs`. If it is missing, run `just rust-install` and hand that command to
`/sase_monitor` when it is long. Each phase runs `just fix`, then `sase tool run check`
(not raw `just check`). Split modules before they hit the `toobig` limit.

## Flag, authoring service, and note verb

1. Create the flag with `sase flag new bead_note_attachments -k beta` and these
   sentences:
   - `--when-enabled`: `@<path>` references inside bead note text attach
     content-addressed file snapshots, and `sase bead attach` is available.
   - `--when-disabled`: Bead note text keeps today's behavior (only a whole-argument
     `@<path>` reads text from a file) and `sase bead attach` refuses with an enable
     hint.
   - `--remove-when`: The bead note attachments epic lands with sharing, image viewing,
     TUI support, and purge verified on athena.

   Paste the printed registry entry into `src/sase/feature_flags/registry.py` (member
   and definition, alphabetical with the existing keys). Run
   `tools/sync_feature_flags_schema --write`. The flag must have a non-test
   `FeatureFlag.bead_note_attachments` reference in this same change.

2. Add frozen `BeadNoteAttachment` on `BeadNote` in `src/sase/bead/model.py` with
   `attachments: tuple[BeadNoteAttachment, ...] = ()`. Decode and encode it in
   `note_codec.py`. Empty manifests omit the key. Image and origin are omitted when
   absent. `bead_wire`, `jsonl`, and `_db_codec` already travel through this codec; do
   not add a SQL column.

3. Thread optional attachments through `bead_mutation_facade` and
   `BeadProjectMutationEvidenceMixin` / the close mixin:
   - `append_note(..., attachments=None)` and `append_note_many`
   - `edit_note(..., attachments=None)` — `None` keeps, a list replaces
   - `close(..., note_attachments=None)` and `plus_one(..., note_attachments=None)`

   Pass a list only when this phase has a manifest to store. An append with no
   attachments still omits the argument.

4. Add `read_note_text_value()` in `src/sase/cli_file_values.py`. Flag off delegates to
   `read_at_path_value`. Flag on:
   - a value that does not start with `@` is returned unchanged
   - a whole value that starts with `@@` is returned untouched so the scanner collapses
     the escape once
   - a bare `@` stays literal
   - any other whole-argument `@<path>` reads UTF-8 note text, at most 256 KiB (`262144`
     bytes). A binary or oversized file fails with `CliFileValueError` naming
     `sase bead attach <id> <path>` and the inline form `"… @<path>"`. On a TTY, a dim
     stderr line confirms
     `note text read from <display> (<size>) · to attach it instead: sase bead attach <id> <display>`.

5. Add `src/sase/bead/attachments/authoring.py`. One pipeline, used by the CLI and the
   TUI, and only imported when the flag is on:
   1. Scan with the bead's current roster names.
   2. Resolve and stat every path. Apply the sensitive-path policy.
   3. Raise one error for every scanner diagnostic and every resolution problem. Render
      caret diagnostics with `rich.cells.cell_len` on the source line (spans are byte
      offsets). Pluralize `N attachment problem(s) in note text — nothing was written.`
      A missing file hints `Write @@… for literal text, or fix the path.` A bare-word
      mention whose `./word` exists is a dim hint, not a failure.
   4. Ingest each unique resolved path once (`ingest_path`). A failure here also writes
      no bead event.
   5. `classify_attachment` on the ingest head. `probe_image` on the object (lazy
      Pillow, never raises, never SVG/EPS). `origin` is `get_machine_name()` when it is
      non-empty.
   6. `unique_attachment_name` against the roster plus names already assigned in this
      text. `compose_note_attachment_text` with `assigned_names` parallel to
      `path_refs`.

   Return stored text, wire dicts, echo rows, and names detached relative to the
   previous manifest of an edit. Same name and same digest reuses the name. A different
   digest echoes `stored as login-2.png`.

   Add `get_attachment_sensitive_patterns()` in `src/sase/bead/config.py`, failing open
   to `[]`. Add `bead.attachments.sensitive_patterns: []` to
   `src/sase/default_config.yml` and `src/sase/config/sase.schema.json` (`bead` has
   `additionalProperties: false`). Document that key in `docs/configuration.md`. Do not
   add the other `bead.attachments` keys.

6. Wire `handle_bead_note` (`src/sase/bead/cli_crud_evidence.py`):
   - Flag off: today's `read_at_path_value` and append/edit with no attachments
     argument.
   - Flag on: `read_note_text_value`, then the service. Append passes the manifest only
     when it is non-empty. `--edit` passes the new manifest, including `[]` when the
     edit detaches every attachment, and the echo names the detached files. `--remove`
     is unchanged.
   - Add `-S/--allow-sensitive`.
   - Echo rows go to stderr and are plain when stderr is not a TTY or `SASE_AGENT` is
     set. Then the existing `Noted:` / `Note #N edited:` line.
   - Replace the note help's `single-token @<path>` sentence with the mental model and
     update `tests/main/test_parser_command_help.py`.

7. Fast path (`src/sase/main/bead_fast_path.py`): `close` and `update -n` already stay
   in Python. When the flag is on, `note` argv containing `@` anywhere (including the
   middle of an argument) returns `None` before `bead_cli_execute`. Flag off keeps
   `_argv_requests_at_path`. Do not import the attachments package from the fast path.

8. TUI `action_beads_add_note`: when the flag is on, run the service before
   `_submit_bead_mutation`. On failure, notify and re-open `BeadNoteModal` with the
   typed text (add an optional initial value; the modal currently starts empty). Do not
   append raw `@path` text. Flag off appends the typed text unchanged. No path
   completion, paste handling, or thumbnails.

9. Tests, both flag states, on a temporary bead store:
   - inline, quoted, reused, and `@@` references round-trip to `@attachment:<name>` plus
     a manifest; `me@host`, `@large`, and `@research:…` stay literal
   - a missing path, a directory, a sensitive path, and a scanner diagnostic leave the
     bead store unchanged
   - the source file can be deleted immediately after a successful note
   - edit reuses a roster name, replaces a changed digest, and detaches when the new
     text drops the token
   - whole-argument `@file` still loads note text; inside that text, `@path` attaches
     only when the flag is on
   - flag off stores inline `@./x.png` as literal text and writes no manifest
   - fast path: with the flag on, `note` argv with an embedded `@` never calls
     `bead_cli_execute` and never stores the raw path
   - attachment-free `notes_to_dicts` output has no `attachments` key

## Close, update, +1, and attach

The authoring service, flag, and `-S` parsing pattern already exist. This phase only
connects the remaining writers.

1. `close -n` and `+1 -n` use `read_note_text_value` and, when the flag is on, the
   service. Pass `note_attachments` only when the manifest is non-empty. Flag off is
   today's whole-argument read and a note with no manifest. `close -r` and `+1 --ref`
   stay literal. `+1` fast path follows the same flag-on `@`-anywhere rule as `note`.
   Add `-S` to both parsers. The `+1` withheld-reopen follow-up note is generated text
   and is not scanned.

2. `update -n` ingests each unique path once, then composes per bead against that bead's
   roster, so the same filename can uniquify differently on each bead. Field updates in
   the same command still apply. `--description` is not scanned. Add `-S`. `update -n`
   is already forced onto the Python path.

3. `sase bead attach <id> <file|->...` with `-a/--author`, `-n/--note`, `-N/--name`, and
   `-S/--allow-sensitive`. No `-L` yet.
   - Flag off exits with an error that names `sase flag enable bead_note_attachments`.
   - `-N` is valid only with one file and is required when a file argument is `-`.
   - Build source text the service already scans: optional `-n` prose, a blank line,
     then one `@<path>` or `@"<path>"` per file. Stdin (`-`) is `ingest_stream` plus the
     `-N` name, then the same uniquify step, because stdin has no path for the scanner.
   - The stored note is the prose (when present), a blank line, and the
     `@attachment:<name>` tokens.
   - Register `attach` alphabetically in `register_bead_parser` (after `apply-status`,
     before `blocked`), in `sase.bead.cli` exports, and in the `entry.py` handler map
     and usage string.

4. Tests, both flag states:
   - close note and `+1` note attach and, with the flag off, do not
   - multi-id update ingests a shared file once and composes names per bead
   - a failed file in a multi-file attach writes no note
   - stdin attach requires `-N`, round-trips the bytes, and works when the source stream
     is not a filesystem path
   - `attach` with the flag off writes nothing and prints the enable hint

## Text rendering, list/path, and beta docs

Rendering is not flag-gated. A note written with the flag on renders on a machine where
the flag is off. Beads whose notes have empty attachment tuples do no attachment work
and do not import the store.

1. Extend `src/sase/bead/note_presentation.py` (or a sibling imported only when a note
   has attachments) so CLI, JSON, and history share one chip formatter. This phase uses
   the plain text form, not the later image form:
   - prose replaces each `@attachment:<name>` token with `[name]`
   - filenames shown to the terminal are stripped of control and bidi characters
   - build Rich `Text`; never `from_ansi`
   - the note label gains `📎 N` when that note has attachments
   - under the prose, an `ATTACHMENTS` block lists
     `name · mime · dims · size · sha256:<12 hex>` and, when cached, the absolute view
     path from `LocalAttachmentStore.materialize_view`
   - unavailable objects render `✕ unavailable offline` and no path, and `show` / `read`
     still succeed
   - `show` and `read` both use this plain form. Do not add `--images`.

   `render_bead_note_lines` in `cli_detail_sections.py` is the prose hook. Compact
   list/search rows (`cli_query_render.py`) gain a `📎N` suffix only when the total
   attachment count is non-zero. Count it from the manifests already on `BeadNote`.

2. Public JSON (`issue_to_wire_dict` only, not `notes_to_dicts`) adds `availability`
   and, when cached, `local_path` onto each attachment object. Never add those keys to
   the store codec. JSON contains no bytes and no escape codes.

3. `sase bead history` today diffs flattened note text and does not carry manifests
   (`sase-core` `history.rs`). Do not change sase-core. In `cli_history.py`, when a
   notes change contains `@attachment:<name>` tokens, print descriptor lines by joining
   those names to the bead's current note manifests. A token whose name is gone prints
   `name (not on the current bead)`. If a history payload already includes attachment
   dicts, render those instead of the join. Record a `PROPOSED FOLLOW-UP:` if per-event
   historical mime and size are still missing.

4. `sase bead attachment list <id>` and `sase bead attachment path <id> <name>`. Bare
   `sase bead attachment` delegates to `list` because the group has a `list` child
   (`default_list_subcommands`); do not reimplement that notice. `list` supports
   `-j/--json`. `path` prints the absolute view path or a clear unavailable error.
   Neither command fetches and neither checks the flag. Register `attachment`
   alphabetically next to `attach`.

5. Docs:
   - `docs/beads.md`: an **Attachments (beta)** section with the mental model, the
     grammar table from the parent plan, the commands this epic actually ships,
     local-only storage, and the text viewing rules
   - `docs/cli.md`: rows for `attach`, `attachment list`, `attachment path`, and `-S` on
     the note-bearing verbs
   - help for `attach` and `attachment` matches those commands
   - `docs/configuration.md` already mentions `sensitive_patterns` from the first phase;
     do not document keys this epic does not add

6. Tests:
   - `read`, `show`, and JSON for an attached note contain bracket chips, the
     descriptor, and the view path, and contain no ANSI escapes and no file bytes
   - a missing local object renders unavailable and the command exits 0
   - a bead with no attachments has the same `show` / `read` / compact output as before
     this epic
   - rendering is identical with the flag off
   - `attachment list` and `path` work with the flag off
   - history full output names the attachment on the note event
   - help snapshots cover the new commands
