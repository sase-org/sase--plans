---
tier: epic
title: Restore project completion placement
goal:
  Selecting a project completion places the chosen tag at the existing workspace target
  or leading prompt position in both the prompt widget and LSP.
phases:
  - id: core
    title: Restore shared project-tag selection and LSP edits
    depends_on: []
    size: medium
    description:
      "core: restore target-position insertion in sase-core and verify the Rust accept
      and LSP contracts."
  - id: sase
    title: Adopt the core fix in the prompt widget
    depends_on:
      - core
    size: small
    description:
      "sase: ratchet the committed core revision and verify prompt-widget acceptance and
      Python parity."
proposed_by: bbugyi200.apollo.1w
create_time: 2026-09-26 07:27:33
status: wip
---

- **PROMPT:**
  [prompts/202609/restore_project_completion_placement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/restore_project_completion_placement.md)

# Restore project completion placement

## Diagnosis

The regression began with the project tag migration. Before that change,
`apply_vcs_project_selection` stripped the typed `+query`, replaced an existing VCS
workflow tag, or inserted a tag at the established leading offset after frontmatter and
directive prefixes. The current `sase-core` `project_tag_selection_edits` instead makes
the trigger replacement the insertion edit and removes every other workspace target. The
prompt widget consumes its full-text result, and the LSP publishes those same edits, so
both leave the selected tag at the typed `+` instead of the old tag or leading position.
Existing tests explicitly expect the in-place result and need to be updated.

## Phase `core`: Restore shared project-tag selection and LSP edits

1. In the linked `sase-core` checkout, change `project_tag_selection_edits` and
   `apply_project_tag_selection` to remove the typed trigger and put the selected row at
   the earliest existing workspace target in the trigger's `---` segment. Remove any
   further targets in that segment so the accepted prompt has one target. With no
   existing target, insert at the segment's leading project-tag position, following the
   pre-regression frontmatter, whitespace, and `%directive` placement behavior. Preserve
   other segments, unknown tags, literal zones, line breaks, and the selected row's
   `+tag` or `#ref` spelling. Handle a trigger that occupies the destination without
   overlapping edits.
2. Return the caret just after the actual insertion. For LSP, keep the primary edit on
   the trigger token and emit nonoverlapping additional edits for target replacement or
   leading insertion. Merge coincident trigger and destination edits. Keep candidate
   filtering and row order intact.
3. Update Rust accept, binding, completion-candidate, and LSP response tests that
   currently assert in-place insertion. Cover an existing `+tag` or `#workflow:ref`
   before or after the trigger, no target, multiple targets, frontmatter/directives,
   later segments, PR rows, whitespace/newlines, Unicode offsets, and literal zones.
   Assert final text and caret, plus LSP edit ranges and their applied result. Update
   Rust comments that specify in-place placement. Run focused Rust tests and the guarded
   core `sase tool run check` gate. Let the host-owned finalizer land the core change
   before phase `sase` starts.

## Phase `sase`: Adopt the core fix in the prompt widget

1. After phase `core` lands, move `sase-core-revision.txt` past its committed revision
   with the supported ratchet workflow. Do not duplicate the selection algorithm in
   Python; the prompt widget already calls the shared `project_tag_apply_selection`
   binding.
2. Update Python binding and prompt-widget regression tests to assert the restored
   position and caret through actual acceptance, plus existing target removal. Update
   comments or documentation that still promise in-place insertion. Run focused tests
   and the guarded sase `sase tool run check` gate against the new pinned revision.

## Acceptance criteria

- Selecting a project with an existing target puts the selected tag at that target's
  position and removes the typed `+query` and other targets in the same prompt segment.
- Selecting a project with no existing target puts the selected tag at the established
  leading position and removes the typed `+query`.
- Prompt-widget and LSP acceptance yield identical text, preserve unrelated content, and
  place the prompt-widget caret after the selected tag.
- Focused regression tests and the required checks pass in both repositories, and sase
  CI pins the committed core fix.
