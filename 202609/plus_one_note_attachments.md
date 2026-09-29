---
tier: tale
title: Finish task +1 note attachments and land the CLI epic
goal:
  Task +1 evidence preserves attachment snapshots across the core and Python read
  surfaces, and epic sase-1ck.4.1 plus parent phase sase-1ck.4 close normally.
size: medium
proposed_by: bbugyi200.athena.sase-1ck.4.1.land
bead: sase-1ck.4.1
create_time: 2026-09-29 15:15:07
status: wip
---

- **PARENT:**
  [202609/note_cli.md](https://github.com/sase-org/sase--plans/blob/main/202609/note_cli.md)
- **BEAD:**
  [sase-1ck.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ck/sase-1ck.4.1.md)

# Finish attachment support for task +1 evidence and land the CLI epic

## Context and scope

Epic `sase-1ck.4.1` and its three phases are otherwise implemented. Phase
`sase-1ck.4.1.2` note 1 reported a real hole in its promised `sase bead +1 -n`
attachment behavior. At the core pin `43f744be`, `TaskPlusOneEvidenceWire` has no
attachment manifest. `add_task_plus_one` in
`crates/sase_core/src/bead/mutation/plus_one_snooze.rs` rejects a manifest on an
ordinary +1, while a snooze wake supplies the manifest to a generated wake note that has
no matching `@attachment:` tokens. Python already scans and ingests `+1 -n` references
in `src/sase/bead/cli_crud_evidence.py`, then passes `note_attachments` through its
facade. The refusal tests in `tests/test_bead/test_cli_attach_verbs.py` pin the
incomplete behavior.

The Rust core is in the linked `sase-core` repository; open it with
`sase repo open sase-core`. The parent epic plan is
`plan:202609/bead_note_attachments.md`; this child epic plan is
`plan:202609/note_cli.md`. Follow-up triage is recorded on bead `sase-1ck.4.1`:
unrelated launch mock and fast-path failures are ready tasks `sase-1cm` and `sase-1cn`;
terminology audit defects in the core attachment fixture were recorded on active parent
epic `sase-1ck`. The `sase-1cj.7/.8` Symvision exemptions have already been re-keyed to
still-open `sase-1cj`.

## Implement

1. In `sase-core`, make a +1 evidence entry own the manifest for its own note. Add an
   optional, empty-omitted `attachments` field to `TaskPlusOneEvidenceWire`, validate
   the one-to-one token/name relationship using the existing note-attachment policy, and
   persist it in the `TaskPlusOneRecorded` event and projection. Remove the ordinary-+1
   refusal. For snooze wake, attach the manifest to the evidence note; keep the
   generated wake note attachment-free unless it contains matching tokens. Preserve old
   event and projection bytes when evidence has no attachments. Cover ordinary, snoozed,
   withheld-reopen, duplicate-reporter, and invalid-manifest cases in core tests,
   including no partial event on validation failure.
2. In `sase`, carry evidence attachments through `TaskPlusOneEvidence`, Rust-wire
   conversion, JSONL/SQLite codecs, public JSON, and any task-gate or work-plan
   serialization that includes evidence. Reuse the note attachment presentation and
   availability helpers so show/read render evidence text as safe `[name]` chips and an
   attachments block, and `sase bead attachment list/path` finds evidence attachments.
   Extend bead roster and compact counts if needed. Do not store paths or bytes in
   events or projections; keep flag-off legacy evidence unchanged. Update the refusal
   tests to prove successful byte-exact `+1 -n` attachment persistence, availability
   after source deletion, and sane behavior for snooze wake and no-manifest evidence.
3. Move `sase-core-revision.txt` past the core commit containing the new wire/API,
   rebuild/install the binding, and verify both repositories with their wrapped
   `sase tool run check` gates after formatting. Run focused core and Python +1, note,
   attachment, presentation, and legacy compatibility tests. Treat the previously
   recorded parent-epic terminology audit issue and unrelated CI tasks separately from
   this feature; report any fresh failure with evidence. Do not run `just check-full`.

## Land this epic and its parent phase

4. Re-read `sase-1ck.4.1` and its child notes, verify the successful `+1` behavior and
   post-start drift, then run `sase bead epic-symbols sase-1ck.4.1`. Resolve every
   listed entry or re-key it only to a still-open later bead that genuinely needs the
   exemption. Close this epic normally with
   `sase bead close sase-1ck.4.1 --note "<verified implementation, tests, integration, and follow-up outcomes>"`;
   never use `--force` merely to bypass descendants or symbols. Run `just symvision`
   after close and set `status: done` in the frontmatter of `plan:202609/note_cli.md`
   through the opened plans repository. If close is refused, finish the identified work
   and retry.
5. Parent `sase-1ck.4` is a phase bead. Verify that this child plan now fulfills its
   note/close/update/+1/TUI/attach/list/path/text-rendering scope, then close only
   `sase-1ck.4` normally with `sase bead close sase-1ck.4 --note "<what was verified>"`.
   Leave containing epic `sase-1ck` open for its land agent; its other phases remain in
   progress. Never force a successful nested landing.
