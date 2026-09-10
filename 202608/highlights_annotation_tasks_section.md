---
tier: tale
title: Put highlights annotation tasks in an H2 "Tasks" section
goal:
  "`bob highlights` writes annotation-derived `#task` bullets into an H2 `## Tasks`
  section of the reference note — creating that section directly below the generated
  `^ref` task when it is missing — instead of splicing the tasks in as bare siblings
  immediately under the `^ref` line."
size: medium
proposed_by: bbugyi200.athena.0fj
create_time: 2026-09-09 20:00:08
status: wip
---

# Plan: Put highlights annotation tasks in an H2 `Tasks` section

## Why

`bob highlights sync`/`scan` turns `#task` bullets found in Highlights annotations into
real Obsidian tasks in the reference note. Today it splices those task lines in directly
under the generated `^ref` lifecycle task, so a reference note grows an unheaded pile of
tasks wedged between its H1 title and the rest of the note.

Bryan wants those tasks collected under an H2 `## Tasks` section instead. The section is
created on demand at the top of the note — below the `^ref` task and a blank line — and
its first task is separated from the heading by a blank line. This also aligns
`bob highlights` with the `## Tasks` convention `bob capture` already uses everywhere
else in the vault (`src/native/capture.rs:2717` `tasks_section`,
`src/native/capture.rs:2366` `insert_task_line`), so a `bob capture` into a reference
note and a `bob highlights sync` of the same note now agree on where tasks live.

## Current behavior (verified in this repo)

All `bob highlights` code lives in `src/native/highlights_ref/` (`mod.rs` ~9.5k lines,
`create.rs`). The annotation-task path:

- `finalize_annotation_task_plans` (`mod.rs:1196`) accepts candidates against the
  vault-wide processed index, renders each accepted candidate with
  `render_annotation_task_line` (`mod.rs:4647`), and partitions them by target.
- Reference-note lines are spliced by `insert_annotation_task_lines_after_pdf_task`
  (`mod.rs:6224`), called at `mod.rs:1250` on `plan.rendered_body`:

  ```rust
  if !reference_task_lines.is_empty() {
      plan.rendered_body = insert_annotation_task_lines_after_pdf_task(
          &plan.rendered_body,
          &reference_task_lines,
      )?;
      refresh_stable_rendered_note(plan);
  }
  ```

  It locates the `^ref` line with `parse_pdf_task_line` (`mod.rs:4963`), errors with
  `"reference note is missing the generated PDF task line with ^ref; cannot create annotation tasks"`
  when the line is absent, and otherwise calls
  `insert_lines_after(body, task_line.line_index, lines)` (`mod.rs:6281`).

- `insert_lines_after` walks `body.split_inclusive('\n')` and re-emits each inserted
  line with the target segment's own line ending (`split_line_segment`, `mod.rs:6344`),
  appending a `\n` first when the target line is the unterminated last line. **Reuse it;
  do not reimplement line splicing.**
- Routed `@name` tasks take a different path: `append_task_lines` (`mod.rs:6243`)
  appends them to the end of the routed root note. That path is **out of scope** (see
  Decisions).
- `plan.rendered_body` is the note body with frontmatter already stripped (`ParsedNote`,
  `mod.rs:200`; `parse_note`, `mod.rs:7000`), so a body line index is never a
  frontmatter line.
- `default_note_body` (`mod.rs:6056`) renders a new note as:

  ```text
  \n# <title>\n\n- [ ] #task #ref [[<pdf>]] #hide ^ref\n\n## Highlights\n\n<!-- highlights:begin -->\n\n…\n<!-- highlights:end -->\n
  ```

  There is no `## Tasks` section today.

- `insert_missing_annotation_tasks` (`mod.rs:6187`) is a `#[cfg(test)]` wrapper over the
  same splice, exercised by the unit test
  `annotation_task_insertion_is_idempotent_and_preserves_existing_states`
  (`mod.rs:8677`). It is the only test that asserts placement, via
  `updated.contains("- [ ] #task [[lib/example.pdf]] #hide ^ref\n{new_line}\n")`
  (`mod.rs:8727`).
- Shared Markdown helpers already exist in a sibling module and are reachable with
  `use super::markdown;` (exactly how `src/native/capture.rs:19-30` imports it):
  `markdown::atx_heading` (`src/native/markdown.rs:68`, returns `(level, title)` and
  strips closing ATX hashes so `## Tasks ##` matches), `markdown::fence_marker`
  (`markdown.rs:10`), `markdown::closes_fence` (`markdown.rs:24`). `highlights_ref` does
  not import it yet.
- Duplicate suppression is position-independent: `ProcessedTaskIndex::accept`
  (`mod.rs:1291`) matches on the `[h:: …]` processed ID, legacy properties, and legacy
  anchors — never on where the task sits in the note. Moving tasks into a section
  therefore cannot cause re-creation.
- `bob move-done-tasks` (`src/native/collect_done.rs`) has no section awareness; it
  removes task lines only, so an archived-out `## Tasks` section can be left empty. A
  later sync reuses that empty heading.

Ordering consequence of today's code: each sync splices its new batch immediately under
`^ref`, i.e. **above** tasks created by earlier syncs (newest-first), while tasks within
one batch keep annotation order.

## Desired behavior

For reference-note (non-routed) annotation tasks, given a note body whose `^ref` line is
present:

1. If a `Tasks` heading already exists in the body (outside the managed Highlights
   region and outside fenced code), reuse it.
2. Otherwise insert `## Tasks` immediately after the `^ref` line, separated from it by
   one blank line.
3. Insert the new task lines into that section, after the last top-level task block
   already in the section; when the section holds no task lines, insert directly after
   the heading.
4. The first task under the heading is separated from the heading by exactly one blank
   line, and the inserted block is separated from following non-blank content by one
   blank line.
5. Everything else is unchanged: identical rendered task lines, identical processed-ID
   suppression, identical `^ref` error when the lifecycle line is missing, identical
   routed-note behavior, identical reports and counters.

Target shape for a fresh note that imports two tasks:

```md
# Task Notes

- [/] #task #ref [[lib/books/task-notes.pdf]] #hide ^ref

## Tasks

- [ ] #task Reconcile with chapter 3. [[#^h-2b91f0a4c7de|🔖]] [h:: 4c0a13d2…]
      [created::2026-08-28]
- [ ] #task Ask about the standalone note. [[#^h-8f42a61a90cc|🔖]] [h:: 910f6ce7…]
      [created::2026-08-28]

## Highlights

<!-- highlights:begin -->

…

<!-- highlights:end -->
```

## Implementation

All changes are in `src/native/highlights_ref/mod.rs` unless stated otherwise.

### Step 1 — Import the shared Markdown helpers

Extend the existing `use super::{…}` block (`mod.rs:25-28`) with `markdown`. Do not copy
`atx_heading`/fence logic into this module.

### Step 2 — Add constants

Beside the existing body constants (`mod.rs:67-69`):

```rust
const TASKS_SECTION_TITLE: &str = "Tasks";
const TASKS_SECTION_HEADING: &str = "## Tasks";
```

### Step 3 — Locate the tasks section

Add a small line-index scanner. It works on the frontmatter-stripped body, so it only
has to skip fenced code and the managed Highlights region.

```rust
/// Line index of the note's `Tasks` heading, ignoring fenced code and the
/// managed Highlights region.
fn tasks_heading_line_index(body: &str) -> Option<usize>;
```

Rules:

- Iterate `body.lines().enumerate()`.
- Track a `managed` flag: set it when a line contains `MANAGED_BODY_BEGIN`, clear it
  when a line contains `MANAGED_BODY_END`; skip every line while it is set, including
  the two marker lines. Generated notes always put each marker on its own line, and this
  keeps a sidecar-derived page label such as `### Tasks` inside the managed region from
  being mistaken for the section.
- Track fences with `markdown::fence_marker` / `markdown::closes_fence` and skip fenced
  lines.
- Return the first index where `markdown::atx_heading(line)` yields a title equal to
  `TASKS_SECTION_TITLE`. Match **any** heading level, not only H2, so a hand-written
  `# Tasks` is reused instead of gaining a duplicate `## Tasks` sibling. This matches
  `capture.rs`'s `tasks_section` (`capture.rs:2717`), which also matches on title alone.

Add a companion that bounds the section body:

```rust
/// Exclusive end line index of the `Tasks` section that starts at
/// `heading_line_index`.
fn tasks_section_end_line_index(body: &str, heading_line_index: usize) -> usize;
```

The section ends at the first later line that is an ATX heading of any level (outside
fences) or contains `MANAGED_BODY_BEGIN`, else at the body's line count.
(`## Highlights` normally supplies the bound; the marker check is a belt-and-braces stop
for hand-edited notes.)

### Step 4 — Create the section when it is missing

```rust
/// Ensure the body has a `Tasks` section, creating `## Tasks` one blank line
/// below the generated `^ref` task. Returns the body and the heading's line
/// index.
fn ensure_tasks_section(body: &str, pdf_task_line_index: usize) -> (String, usize);
```

- When `tasks_heading_line_index(body)` is `Some(index)`, return
  `(body.to_string(), index)` unchanged.
- Otherwise splice with the existing helper:
  `insert_lines_after(body, pdf_task_line_index, &[String::new(), TASKS_SECTION_HEADING.to_string()])`,
  which yields `^ref` line, blank line, `## Tasks`, using the `^ref` line's own line
  ending (CRLF-safe) and handling an unterminated last line. The new heading's line
  index is `pdf_task_line_index + 2`.

Note what this deliberately does for a pre-change note whose tasks already sit under
`^ref`: the heading lands above them, so those legacy tasks become the section's
existing content and Step 5 appends below them. No separate migration is needed or
attempted.

### Step 5 — Insert the task lines into the section

```rust
fn insert_lines_into_tasks_section(
    body: &str,
    heading_line_index: usize,
    lines: &[String],
) -> String;
```

Compute the insertion anchor over
`heading_line_index + 1 .. tasks_section_end_line_index(...)`:

1. Find the **last top-level task line** in that range and extend past its continuation
   block. Mirror `capture.rs`'s predicates rather than inventing new ones:
   - top-level task: the line starts with `- [`, has `]` at index 3, and the remainder
     after the checkbox starts with whitespace and contains `#task`
     (`is_top_level_task_line`, `capture.rs:2834`);
   - continuation: following lines that are indented, plus blank lines whose next
     non-blank line is indented (`task_block_end`, `capture.rs:2783`). Skip fenced lines
     so a `- [ ] #task` inside a code block is not treated as a task. If found, the
     anchor is that block's last line index; no leading blank line is added.
2. Otherwise, when the line at `heading_line_index + 1` exists and is blank, the anchor
   is that blank line and no leading blank is added.
3. Otherwise the anchor is `heading_line_index` and a leading blank line is prepended to
   the inserted block, so the first task is always one blank line below the heading.

Then build the payload — `[leading blank?] ++ lines ++ [trailing blank?]` — where the
trailing blank is added when the line after the anchor exists and is non-blank, so the
block never runs straight into `## Highlights` or manual prose. Splice it with
`insert_lines_after(body, anchor, &payload)`.

Empty strings in the payload are correct: `insert_lines_after` writes the line then the
target segment's line ending, so a blank line is `""` + `\n` (or `""` + `\r\n`).

### Step 6 — Rewire the splice entry point

Rename `insert_annotation_task_lines_after_pdf_task` (`mod.rs:6224`) to
`insert_annotation_task_lines_into_tasks_section` and reimplement its body as:

```rust
fn insert_annotation_task_lines_into_tasks_section(
    body: &str,
    lines: &[String],
) -> Result<String> {
    if lines.is_empty() {
        return Ok(body.to_string());
    }

    let task_line = match parse_pdf_task_line(body)? {
        PdfTaskLineState::Present(task_line) => task_line,
        PdfTaskLineState::Missing => {
            return Err(CommandError::new(
                "reference note is missing the generated PDF task line with ^ref; cannot create annotation tasks",
            ));
        }
    };
    let (body, heading_line_index) =
        ensure_tasks_section(body, task_line.line_index);
    Ok(insert_lines_into_tasks_section(&body, heading_line_index, lines))
}
```

Keep the missing-`^ref` error exactly as it is (same text, same trigger) — the `^ref`
line is still what anchors a newly created section, and a legacy note without one keeps
failing the way it does today.

Update the call site at `mod.rs:1250` and the `#[cfg(test)]`
`insert_missing_annotation_tasks` (`mod.rs:6187`) to call the renamed function.
`refresh_stable_rendered_note` (`mod.rs:1315`) and everything downstream are unchanged.

### Step 7 — Leave `default_note_body` alone

Do **not** emit `## Tasks` in `default_note_body` (`mod.rs:6056`). Reference notes that
never produce an annotation task should not grow an empty heading; the section is
created on demand by Step 4.

### Step 8 — Tests

Unit tests in the `mod.rs` test module:

- Update `annotation_task_insertion_is_idempotent_and_preserves_existing_states`
  (`mod.rs:8677`). Its fixture body has two pre-existing tasks directly under `^ref` and
  no `Tasks` section, so the new expectation is `^ref`, blank, `## Tasks`, blank, the
  two pre-existing tasks, then the new task, then a blank line before `## Manual Notes`.
  Keep every existing assertion about idempotence, preserved checkbox states,
  `## Manual Notes`, and `## Highlights\n\n<!-- highlights:begin -->`.
- New: a body that already has `## Tasks` with two tasks gets the new task appended
  below them, and no second heading is created.
- New: an existing `# Tasks` (H1) or `## Tasks ##` (closed ATX) section is reused.
- New: a `Tasks` heading inside a fenced code block, and one inside the managed
  Highlights region, are both ignored — a real `## Tasks` is created below `^ref`.
- New: an existing but task-free `## Tasks` section receives the first task one blank
  line below the heading, with a blank line before the following heading.
- New: CRLF body round-trip — every inserted line (blank lines, heading, tasks) uses
  `\r\n` and no `\n`-only line is introduced.
- New: `^ref` as the unterminated last line of the body still produces
  `^ref\n\n## Tasks\n\n- [ ] …\n`.
- New: two consecutive insert calls put the second batch below the first (documents the
  ordering decision).

Integration tests in `tests/cli.rs` (helpers already exist:
`find_created_annotation_task` at `tests/cli.rs:23145`, `created_annotation_task_count`
at `:23155`, `highlight_block_ids` at `:23133`):

- `highlights_ref_sync_creates_tasks_from_pdf_note_task_bullets` (`tests/cli.rs:19855`):
  add assertions that the note contains `## Tasks\n\n- [ ]` and that every created task
  line appears after the `## Tasks` heading and before `## Highlights`. Its existing
  assertions are placement-independent and must keep passing, including the resync and
  completed/cancelled-preservation rounds — which also prove no duplicate section is
  created on later syncs.
- `highlights_ref_task_checked_sync_creates_annotation_tasks_before_closing`
  (`tests/cli.rs:18703`): assert the same-note task lands under `## Tasks`, and that the
  routed `alice.md` task is still appended to the routed note unchanged (no `## Tasks`
  section created there).

The other highlights tests assert `contains` on the `^ref` line, the managed region, or
report counters, so inserting a section between the `^ref` line and `## Highlights` does
not disturb them. Confirm with a full `cargo test` rather than by inspection.

### Step 9 — Docs

`docs/highlights-ref-sync.md`:

- Line ~705, replace "By default, created tasks are top-level siblings immediately under
  the generated PDF `^ref` line:" with the new contract: created tasks go into the
  note's `## Tasks` section; the section is created directly below the `^ref` task,
  separated by a blank line, when the note has none; an existing `Tasks` heading at any
  level is reused; new tasks are appended after the tasks already in the section. Update
  the fenced example immediately below it to show the heading, the blank line, and the
  two task lines.
- The generated-note paragraph at ~line 545 ("New generated notes include a title, a PDF
  wikilink Obsidian task line …") should state that `## Tasks` is not part of a new
  note's skeleton and appears only when the first annotation task is imported.
- Keep the routed-`@name` paragraph (~line 715) as is, and say explicitly that routed
  tasks are still appended to the end of the routed note.

`README.md` needs no change: its `highlights` section (lines 342-366) does not describe
annotation-task placement.

## Verification

```bash
just fmt     # cargo fmt --check
just lint    # cargo clippy --all-targets --all-features
just test    # cargo test
```

Then an end-to-end check against a scratch vault — **never** `~/bob`:

```bash
# Build a throwaway vault with a wip PDF + sidecar containing `- #task …` bullets,
# mirroring tests/cli.rs:19855, then:
bob highlights sync --dry-run -b /tmp/scratch-vault /tmp/scratch-vault/lib/books/x.pdf
bob highlights sync -b /tmp/scratch-vault /tmp/scratch-vault/lib/books/x.pdf
cat /tmp/scratch-vault/ref/books/x.md   # ^ref, blank, ## Tasks, blank, tasks, blank, ## Highlights
bob highlights sync -b /tmp/scratch-vault /tmp/scratch-vault/lib/books/x.pdf
                                        # second run: no new tasks, no duplicate heading
```

Do not run a writing `bob highlights sync`/`scan` against `~/bob`; leave that to the
owner.

## Decisions and rejected alternatives

- **New tasks are appended below the tasks already in the section**, not inserted at its
  top. This changes today's incidental newest-first ordering (each batch landed directly
  under `^ref`, above older tasks) to chronological order, and it matches
  `bob capture`'s behavior for the same `## Tasks` convention in the same vault. Within
  a single sync, annotation order is preserved either way.
- **Reuse a `Tasks` heading at any level, create at H2.** The user asked for an H2
  section, and that is what gets created; matching an existing `# Tasks` or `### Tasks`
  only avoids producing a second, competing section in a hand-edited note, consistent
  with `capture.rs`'s title-only match.
- **Re-implement the section helpers inside `highlights_ref` rather than exporting
  `capture.rs`'s.** `capture.rs`'s `insert_task_line` cannot create a missing section
  (it falls back to the last top-level task block anywhere in the file), operates on
  full note contents rather than the frontmatter-stripped body, and reaching it would
  mean making six private helpers (`line_spans`, `tasks_section`,
  `last_task_block_insert_index_in_range`, `insert_at`, `insertion_text`,
  `empty_section_insertion_text`) `pub(crate)` in a heavily tested module. The genuinely
  shared parsing — ATX headings and fences — is already factored into
  `src/native/markdown.rs` and is used, not copied.
- **Routed `@name` tasks keep appending to the end of the routed note**
  (`append_task_lines`, `mod.rs:6243`). The reported problem is specific to placement
  under the reference note's `^ref` line; routed root notes have no `^ref` anchor, and
  moving them into a `## Tasks` section is a separate, user-visible change to notes this
  plan was not asked to touch. Worth raising with the owner separately.
- **No migration of existing notes.** Tasks already sitting under `^ref` are not moved
  by a sweep. When the next annotation task arrives for such a note, the new heading is
  inserted directly below `^ref` and those legacy tasks naturally end up inside the
  section — which is the desired end state without a risky vault-wide rewrite.
- **`default_note_body` does not pre-create the section**, so notes that never import an
  annotation task stay exactly as they render today.
