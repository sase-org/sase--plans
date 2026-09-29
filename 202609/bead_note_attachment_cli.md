---
tier: tale
title: Author and read bead note attachments from the CLI
goal:
  Beta-enabled bead note commands snapshot inline files, and every client can read local
  attachment metadata and paths without exposing file bytes.
size: medium
proposed_by: bbugyi200.athena.sase-1ck.4
bead: sase-1ck.4
create_time: 2026-09-29 11:46:19
status: wip
---

- **PARENT:**
  [202609/bead_note_attachments.md](https://github.com/sase-org/sase--plans/blob/main/202609/bead_note_attachments.md)
- **BEAD:**
  [sase-1ck.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.4.md)

# Bead note attachment authoring and CLI

Implement only phase `sase-1ck.4` (`note_cli`) of the bead-note-attachments epic. Its
core grammar, wire/reducer/mutation APIs, and local content-addressed store are
prerequisites from the closed phases `sase-1ck.1` through `.3`. The public behavior in
this phase is a beta-gated way to snapshot files mentioned in bead notes, plus ungated
rendering and local access to existing attachments. Shared-store upload/fetch, image
previews, and the full TUI attachment UI belong to later phases.

## Implementation

1. Refresh the pinned Rust binding with `just rust-install` and confirm the existing
   core scanner, inverse source-text composer, name/media helpers, attachment policy,
   descriptor wires, and note mutation kwargs from the epic design. Use the binding for
   shared semantics; keep file and terminal I/O in Python. Run
   `sase flag new bead_note_attachments -k beta` with the exact enabled, disabled, and
   removal sentences in phase 4 of `plan:202609/bead_note_attachments.md`. Add its
   printed registry entry and schema mirror. The flag defaults off. Only authoring is
   gated: rendering, `attachment list`, and `attachment path` work regardless of the
   flag. This is the specified epic flag scaffold; do not create unrelated task beads or
   close its removal bead.
2. Extend the Python bead model and codecs with an immutable attachment descriptor tuple
   on `BeadNote`. Preserve absent-field and empty-manifest compatibility, wire key
   ordering, and byte-identical legacy events and projections. Thread manifests through
   `notes_to_dicts`, JSON output, the Rust-backed project/facade mutation calls, and
   note edit semantics (`None` versus an explicit empty replacement).
3. Build one `bead/attachments/authoring.py` pipeline for CLI and TUI. Scan with core;
   resolve against invocation cwd with `~` expansion; stat every candidate, follow
   permitted symlinks, reject non-regular/unreadable and sensitive files using core
   policy, and collect every problem before ingest or bead mutation. Render all spans
   with display-cell-aware carets and actionable escape/fix hints. Ingest each unique
   source once into the existing local CAS, classify/probe it, add machine origin, and
   compose names and stored text separately against each target bead's roster. Validate
   all target compositions before any event write. Return the stored text, manifest, and
   echo rows. Handle reuse, collisions, edit detach, and explicit attachment names
   consistently with core.
4. Add a flag-aware `read_note_text_value()` alongside `read_at_path_value()`. With the
   flag off, preserve existing note argument behavior. With it on, a whole argument
   `@file` still loads UTF-8 note prose, capped at 256 KiB, and then scans its contents;
   a leading `@@` reaches the scanner unchanged and is collapsed once. Give
   binary/oversize input an exact `sase bead attach` hint and the inline alternative,
   and give TTY users the dim source-file confirmation. Apply this only to note-bearing
   values, leaving reasons, descriptions, and other free-text fields alone.
5. Route `note` append/edit, `close -n`, `update -n` (including multi-ID), and `+1 -n`
   through that source layer and authoring service. Add `-S/--allow-sensitive` to each
   authoring parser and arrange that the note manifest reaches the corresponding core
   mutation. Ensure attachment-aware argv never takes the Rust fast path when the flag
   is enabled. Preserve existing close, status, +1, and non-note option semantics. Make
   the TUI add-note action use the same pipeline off the event loop when enabled and
   report errors without losing typed text; leave the richer modal UX to phase `tui`.
6. Add alphabetically listed `sase bead attach <id> <file|->...` and
   `sase bead attachment list|path` commands. `attach` accepts optional prose, stdin
   only with `-N/--name`, and the specified author/sensitive/local-only options as
   applicable; it writes one note with tokens. The beta flag gates `attach` with an
   enable hint. `list` reports descriptors and local availability; `path` materializes
   an absolute extension-preserving local view. At this phase, availability is `cached`
   or `unavailable`; do not claim remote fetch. Use the central bare-group-to-`list`
   delegation and check each parser's short-option conflicts.
7. Render attachment chips and per-note metadata/path blocks in `show` and `read`, plain
   and without escape codes or file contents for `read` and piped output; expose
   descriptors, availability, and cached local paths in JSON, plus compact counts and
   history descriptors. Keep legacy note rendering byte-compatible and skip attachment
   imports and store work on beads without attachments. Update help with the epic's `@`
   mental model and document the beta flow in `docs/beads.md` and `docs/cli.md`.

## Verification and completion

- Test both flag states; inline/quoted/escaped/reused references; an unresolved or
  sensitive candidate and aggregate caret diagnostics; whole-argument text-file
  behavior; attach from stdin; delete-source-after-attach; edit/reuse/detach; multi-ID
  update; `+1` and close notes; and no bead mutation on a failed command. Include a
  fast-path regression test for `@` anywhere in a note-bearing argv, and assert that
  read/piped show/JSON have no ANSI or attachment bytes.
- Run `just fix`, then `sase tool run check` from the sase checkout. Investigate
  failures; if an identical failure reproduces on the clean base, record
  `PROPOSED FOLLOW-UP:` on `sase-1ck.4` with existing task-bead provenance, then finish
  the phase. Follow the visual-snapshot rule if the TUI change affects rendered goldens.
- Run `sase bead epic-symbols sase-1ck.4` immediately before close. Resolve every
  remaining symbol or re-key its Justfile line to an open parent/later phase. Close only
  `sase-1ck.4` with `sase bead close sase-1ck.4 --note "<what was verified>"`; never
  close the parent epic or any ancestor. Record discovered out-of-scope work as
  `PROPOSED FOLLOW-UP:` notes on this phase bead, not new task beads.
