---
tier: tale
title: Create named future Pomodoros from capture
goal:
  A Pomodoro-linked capture can create and target a correctly positioned named future
  Pomodoro when no open name matches.
size: medium
proposed_by: bbugyi200.athena.0fw
create_time: 2026-09-09 19:59:51
status: wip
---

# Create named future Pomodoros from Pomodoro-linked capture

## Goal

Extend an explicitly named Pomodoro-linked `bob capture` so that it keeps linking to an
existing matching open Pomodoro, but creates and targets a new named future Pomodoro
when no open entry matches. The new ledger entry must be placed immediately after the
current Pomodoro's complete block, otherwise after the last completed Pomodoro's
complete block, otherwise before the first Pomodoro in today's `## Pomodoros` section.

## Scope and contract decisions

- The repository's Pomodoro-linked spelling is `@<route>:<block-id>#<pomodoro>`. The
  literal `+` spelling in the request is already the incompatible
  `@<route>+<block-id>#<task-section>` sub-bullet family, so this plan changes the colon
  family and deliberately leaves the plus family unchanged.
- Keep the existing whole-slug-then-prefix lookup among named **open** Pomodoros. The
  first existing open match remains the destination and no new entry is created.
- When no open name matches, treat the valid authored component as the name of a new
  future Pomodoro. This includes cases where the same slug exists only on a completed
  Pomodoro or where the old lookup would have offered a close spelling suggestion.
- Canonicalize the new visible name with the same rule as `bob capture-pomodoro-name`:
  collapse whitespace, ASCII-uppercase, and preserve allowed punctuation. Since the
  inline component is whitespace-free, `#after-tui-fix` creates `AFTER-TUI-FIX`; do not
  guess whether authored hyphens were intended as spaces.
- Render the new top-level entry as `- [ ] () — NAME`, followed by the captured task
  link as its child. Preserve the daily note's LF/CRLF convention and reuse its
  Pomodoro-child indentation where available.
- Define "current" as the unique open timed Pomodoro already recognized by the ledger
  scanner. Multiple open timed Pomodoros remain an invariant error rather than silently
  choosing an anchor. With no current, use the last completed entry; with neither,
  insert before the first ledger entry (or directly into an empty `## Pomodoros`
  section).
- Continue to require an existing daily note and `## Pomodoros` section.
  Cancelled/nested/fenced lookalikes are not anchors. Preserve batch planning, dry-run
  behavior, and all-or-nothing replacement of the routed and daily notes.
- Do not change the shared marker parser, completion candidate schema, or capture JSON
  schema. A novel syntactically valid name is already parsed; the change is
  execution-time ledger planning. Existing output fields continue to report the routed
  task and link insertion.

## Implementation plan

1. Centralize named-Pomodoro creation primitives in the existing capture modules.
   - Expose one internal canonical-name helper used by both `capture-pomodoro-name` and
     inline capture so their ALL-CAPS/title grammar cannot drift.
   - Add a small formatter/helper for the canonical `- [ ] () — NAME` ledger line rather
     than rebuilding that syntax inside the capture transaction.
   - Keep the existing scanner and open-name selection behavior intact for discovery,
     completion, and explicit assignment callers.

2. Extend Pomodoro-link planning in `src/native/capture.rs` with an existing-or-create
   path.
   - Attempt the current named-open selector resolution first.
   - On any no-open-match result, validate/canonicalize the authored name, compute the
     creation anchor from the parsed top-level ledger entries, and insert one new
     placeholder Pomodoro plus its child block link.
   - Insert after an anchor's entire descendant block so its existing children remain
     owned by the anchor. In the no-anchor case, insert before the first real Pomodoro
     so existing future entries retain their order.
   - Factor/reuse the child-indentation and line-ending helpers used by existing
     Pomodoro child insertion. Re-scan or otherwise verify that the planned entry is
     open, named, selectable, and owns the new link before staging the daily-note
     snapshot.
   - Preserve the unique-current guard, duplicate block-link/ID preflight, same-file
     guard, in-memory batch snapshots, and atomic commit/rollback path. A later batch
     item naming the just-created Pomodoro must resolve it instead of creating a
     duplicate.
   - Remove or narrow the now-obsolete execution errors for unknown, completed-only,
     unnamed-only, and close-suggestion cases; retain malformed marker, missing
     note/section, duplicate ID/link, and ambiguous-current failures.

3. Expand focused unit and CLI regression coverage.
   - Verify an existing named open entry is still selected without creation.
   - Verify new canonical placeholders and child links are placed after a current entry
     including all its children, after the last completed entry when no current exists,
     and first when neither exists (both ahead of existing future entries and in an
     empty section).
   - Cover a completed-only matching name and a genuinely novel/nearby name to prove
     both create instead of returning the former lookup errors.
   - Cover LF and CRLF, surrounding child-indent reuse, fenced/nested/cancelled
     lookalikes, missing note/section, and multiple timed entries.
   - Cover `--dry-run` and JSON success without writes, failure atomicity across both
     notes, and a multi-item batch that creates a name once then reuses it from the
     staged daily-note snapshot.

4. Update the public contract in `README.md`, `docs/capture.md`, and
   `bob capture --help`.
   - Describe `#<pomodoro>` as selecting an existing open name or creating a canonical
     named future Pomodoro.
   - Document the current/last-completed/first insertion order, placeholder format,
     canonicalization, preserved ambiguity errors, and transactional dry-run/batch
     behavior.
   - Remove statements that unknown or completed-only names always fail while retaining
     the distinction between the colon Pomodoro family and the plus task-section family.

## Acceptance criteria

- A valid `@route:id#name` capture links to the first existing matching open Pomodoro
  exactly as before.
- With no matching open Pomodoro, the same capture creates exactly one top-level
  `- [ ] () — NAME` entry at the specified anchor and places `[[route#^id]]` beneath it.
- A completed-only match may seed a new future Pomodoro of the same canonical name;
  completed history is never modified.
- Multiple open timed Pomodoros, missing ledger structure, duplicate IDs or links, and
  all other preflight failures remain write-free for both files.
- Dry-run and multi-item batches use the same in-memory result as a real capture, with
  no duplicate future Pomodoro when later items reuse the name.
- Existing `@route+id#section`, parser/preview fields, completion JSON, and capture JSON
  contracts remain compatible.

## Validation

Run focused capture unit/integration tests while developing, then run the full
repository checks:

```bash
cargo test native::capture
cargo test --test cli capture_pomodoro
cargo test --test cli capture_named_pomodoro
just all
```
