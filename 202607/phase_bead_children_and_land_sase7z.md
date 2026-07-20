---
tier: tale
title: Show phase bead children in bead show, then land epic sase-7z
goal: 'sase bead show on a phase bead lists its child epics in a CHILDREN section
  (closing the gap found by the sase-7z smoke phase), and epic sase-7z is landed:
  closed, symvision-cleaned, and its plan file marked done.

  '
create_time: 2026-07-20 10:27:49
status: wip
prompt: 202607/prompts/phase_bead_children_and_land_sase7z.md
---

# Plan: Show phase bead children in bead show, then land epic sase-7z

## Context

Epic sase-7z (phase sizes and parented child epics) is verified complete except for one gap its smoke phase (sase-7z.8)
reported: a phase bead that owns a child epic (e.g. `foo-5.2` begets `foo-5.2.1`) renders no CHILDREN section in
`sase bead show`, so the hierarchy is only discoverable from the child side via lineage. The cause is in
`src/sase/bead/cli_query.py` (`handle_bead_show`): the CHILDREN section is gated on
`issue.issue_type == IssueType.PLAN`, so phase beads never reach the children-rendering branch.

This is a presentation-only Python fix. The Rust read facade already supports it: `get_epic_children`
(`crates/sase_core/src/bead/read.rs`, reached via `src/sase/bead/project.py:341`) filters issues by `parent_id` only and
works for any parent bead. No sase-core changes are needed, per the Rust core backend boundary (bead show rendering is
CLI presentation).

Epic sase-7z itself is otherwise fully verified (all 8 phases closed with commits in this repo, sase-core, and the plans
sidecar; integration with commits that landed since the epic started found no conflicts), so this plan finishes with the
epic's landing steps.

## Implementation

### 1. Render CHILDREN for phase beads

In `src/sase/bead/cli_query.py` `handle_bead_show`, render the CHILDREN section for any bead whose
`view.get_epic_children(issue.id)` result is non-empty instead of only for `IssueType.PLAN` beads, keeping the existing
PHASES / CHILD EPICS grouping and line formats unchanged. Plan-bead output must stay byte-identical (existing goldens
`tests/test_bead/golden/cli/show.stdout` and `show_phase_parent_epic_plan.stdout` prove childless beads are unaffected).
`sase bead search --format full` reuses `handle_bead_show`, so it picks the change up automatically.

### 2. Tests

Extend `tests/test_bead/test_cli_show.py` (which already covers the sase-7z show features) with CLI-level coverage for a
phase bead that owns a child epic: the phase's output gains a CHILDREN section listing the child epic with status and
tier, and a childless phase's output is unchanged. Follow the existing test style there (e.g.
`test_show_plan_splits_phases_from_child_epics`, `test_show_child_epic_under_phase_has_lineage_and_own_plan`).

### 3. Docs

`docs/beads.md` (the `sase bead show <id>` section, ~line 229) currently says "A plan's children are grouped as phases
... and child epics ..." — reword so it covers any bead with children, including a phase bead's child epics.

### 4. Validate

Run `just install`, then `just check`, and fix anything it reports.

## Land epic sase-7z (final step)

After the fix is green, finish landing the epic:

1. Close the epic: `sase bead close sase-7z` (all 8 phase children are already closed).
2. Run `just symvision`. Epic-symbol whitelist entries for sase-7z expire at close; before fixing any reported failures,
   review the long-term memory `sase/memory/symvision.md` via the `/sase_memory_read` skill, then remove the stale
   sase-7z whitelist entries and any unused code symvision reports.
3. Mark the epic's plan file done: the PLAN path printed by `sase bead show sase-7z`
   (`202607/epic_phase_sizes_and_child_epics.md` in the plans sidecar repo — open it with the `/sase_repo` skill) gets
   `status: done` in its YAML frontmatter.

## Risks

- Rendering children unconditionally could surface unexpected sections for odd parent/child type combinations; keeping
  the exact PHASES / CHILD EPICS grouping already handles both issue types, and goldens lock plan-bead output.
- `just symvision` may report symbols unrelated to sase-7z; only act on the expired sase-7z whitelist entries and the
  unused code it attributes to them, per the symvision memory's procedure.
