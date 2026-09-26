---
tier: epic
title: Remove adjacent model directives on shortcut acceptance
goal:
  Accepting =alias or ==model leaves exactly one standalone model directive in the
  active prompt segment, even when redundant directives are adjacent at a line boundary.
parent_bead: sase-1ao
phases:
  - id: adjacent_cleanup
    title: Make adjacent directive cleanup disjoint and verify both frontends
    description:
      "adjacent_cleanup: repair the shared Rust edit planner, pin its commit in sase,
      and add ACE/LSP applied-document regression coverage."
    depends_on: []
    size: medium
proposed_by: bbugyi200.apollo.sase-1ao.land
create_time: 2026-09-26 13:15:15
status: wip
---

- **PROMPT:**
  [prompts/202609/model_shortcut_adjacent_directives.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/model_shortcut_adjacent_directives.md)
- **PARENT:**
  [202609/model_shortcut_replacement.md](https://github.com/sase-org/sase--plans/blob/main/202609/model_shortcut_replacement.md)

# Remaining work from sase-1ao landing

The parent epic implemented segment-scoped model shortcut replacement in sase-core
commit `e1179e6` and adopted it in the prompt widget in sase commit `e922c424e`. The
landing review reproduced one unmet part of its acceptance contract on the installed
binding:

`=la %m:a %m:b` with `@large` selected becomes `%m:@large %m:b`.

The first directive is replaced and the shortcut is deleted, but the second directive
remains. In `crates/sase_core/src/editor/model_alias_shortcut.rs`,
`destination_replacement` consumes the space after `%m:a`; `orphan_strip_region` tries
to consume that same space before the trailing `%m:b`. Their ranges overlap, so
`plan_segment_model_accept` skips the second removal. This is epic work, not a separate
follow-up task. The existing test for repeated directives has intervening prose and
misses adjacent trailing directives. No post-feature core commit edits the planner, and
no post-widget commit edits its ACE acceptance path.

## Phase `adjacent_cleanup`

Open the linked `sase-core` repository with `sase repo open sase-core`. Update the
shared planner so every eligible standalone `%model`/`%m` directive after the earliest
is removed even when the normal whitespace strip overlaps the destination or another
removal. Keep the resulting edit ranges disjoint in original-document UTF-16
coordinates, preserve the destination's spacing and post-edit caret, and do not touch
alternation bodies, literal zones, or other `---` segments. Remove the duplicate overlap
predicate in the target filter while editing this code.

Add Rust applied-document and caret tests for adjacent directives at end of line, before
a newline, and on separate lines, for both `=alias` and `==model`. Cover a protected
alternation branch next to an eligible directive. Extend the PyO3 round-trip and LSP
tests where needed to prove the second removal survives the wire and the LSP primary and
additional edits stay disjoint.

Advance `sase-core-revision.txt` past the repair commit. Add a prompt-widget and ACE/LSP
parity regression using the installed binding: assert the applied text has one
standalone directive, the same caret in both frontends, and a single undo restores the
original prompt. Preserve compatibility with older single-edit payloads. Run focused
Rust/PyO3/LSP and Python tests, the linked repo's `sase tool run check`, and sase's
`sase tool run check` (`just check`). Do not run `just check-full`.

## Acceptance

- Both shortcuts remove redundant adjacent directives in the active segment while
  preserving protected branches and other segments.
- All edits remain pairwise disjoint and the caret matches the applied document in Rust,
  ACE, and LSP.
- The core revision pin contains the repair; focused tests and both required check gates
  have recorded outcomes.

The parent land agent will handle follow-up triage, epic-symbol cleanup, epic close,
symvision, and the parent plan-file status after this child epic lands.
