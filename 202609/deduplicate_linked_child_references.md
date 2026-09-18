---
tier: tale
title: Deduplicate linked child memory references
goal:
  Prefer Linked References over Children when a referenced note is also a direct child,
  while preserving every non-overlapping child and link.
size: small
proposed_by: bbugyi200.athena.0mn.f0
create_time: 2026-09-18 06:21:09
status: wip
---

# Plan: Deduplicate Linked Child Memory References

## Problem

Flat reference notes expose two independent relationships during `sase memory show` and
`sase memory read`: discovered child notes are appended under `Children`, while authored
`[[target]]` links are appended under `Linked References`. When an authored reference
resolves to one of the note's direct children, the Markdown and rich renderers show the
same note twice and the JSON shape repeats it in both `children` and
`linked_references`.

The desired precedence is local to that overlap: keep the authored link and its
`Linked References` entry, but omit the same target from the visible child collection.
Unlinked children must remain under `Children`, and links to notes that are not children
must remain ordinary linked references. If every direct child is linked, no `Children`
section or rich child block should render and the JSON `children` list should be empty.

## Implementation

1. Update the shared flat-note presentation logic in `src/sase/memory/render.py` to
   identify resolved reference-link targets that are also discovered direct children.
   Use the resolved `MemoryNoteLinkTarget` identity/relative path rather than comparing
   authored link text, so equivalent supported spellings resolve consistently. Fold
   those overlapping paths into the existing child-visibility filtering used for inline
   expansion and separately rendered batch targets.

2. Keep the precedence at the shared visible-children boundary so all consumers agree:
   Markdown omits overlapping rows and naturally drops an empty `## Children` section;
   rich output omits the same rows, child block, and header count; and both single-note
   and selector-batch JSON omit the overlap from `children` while retaining it in
   `linked_references`. Preserve authored link order, unresolved-link reporting,
   non-child links, sorted unlinked children, inline-link behavior, depth handling, and
   show/read output parity.

3. Add focused regression coverage in `tests/memory/test_memory_selector_render.py` for
   both boundary cases:
   - complete overlap, where every child is referenced and Markdown/rich have no
     `Children` block while JSON has no visible children and all authored references
     remain under `Linked References`;
   - partial overlap, where only the referenced child moves to `Linked References` and
     the unlinked child remains as the sole `Children` entry across Markdown, rich, and
     JSON.

   Retain or extend CLI-level coverage in `tests/main/test_memory_read_selectors.py` if
   needed to prove `show` and audited `read` expose the same deduplicated result through
   the public command path.

4. Exercise the real `tui.md` case with `sase memory show tui.md` (and JSON output) to
   confirm both linked child notes appear only in `Linked References` and `Children` is
   absent. Run the focused memory renderer/CLI tests, format with `just fix` (or at
   minimum `just fmt`), and finish with the required repository gate, `just check`.

## Acceptance Criteria

- A direct child reached by an authored reference link is rendered only in
  `Linked References`, never duplicated in `Children`.
- A direct child without a reference link is still rendered in `Children`.
- When all direct children are linked references, Markdown and rich output omit the
  child section/block entirely and JSON reports an empty `children` list.
- Linked targets that are not children, unresolved links, inline links, depth-limited
  reads, and batch suppression retain their existing behavior.
- `sase memory show` and `sase memory read` remain output-identical for equivalent
  selectors and formats, and the focused tests plus `just check` pass.
