---
tier: tale
goal: 'Render the Goal field directly beneath the SASE PLAN heading while preserving
  the plan section''s responsive layout, styling, and surrounding section spacing.

  '
create_time: 2026-07-15 18:03:45
status: wip
prompt: 202607/prompts/compact_sase_plan_heading.md
---

# Plan: Remove the blank line below the SASE PLAN heading

## Context and diagnosis

The Agents tab's selected-agent detail panel currently leaves an empty terminal row between the underlined `SASE PLAN`
heading and the `Goal:` field. The spacing is produced locally by
`src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py`: `ResponsivePlanSection._heading()` terminates the
heading line and then appends another newline before the responsive field table begins.

That heading helper has two consumers which must remain consistent:

- `logical_text` builds the retained `Text` representation used by header inspection, search, span assertions, and
  suffix appends.
- `__rich_console__` builds the width-aware terminal representation in which the goal, tier, path, and epic phases can
  fold at narrow widths.

The unconditional post-heading blank line affects both tale/plan metadata and epic phase roadmaps. The divider and its
existing breathing room _above_ `SASE PLAN` are intentional and should remain unchanged.

## Implementation

Make the heading boundary compact in the shared responsive plan-section renderer so `Goal:` starts on the row
immediately after `SASE PLAN`. Keep the single newline that terminates the heading, but remove the additional empty row
before the field table.

Apply the behavior through the shared heading construction rather than special-casing a tier or render width. Preserve:

- the major-section divider and spacing between timestamps and `SASE PLAN`;
- heading, label, goal, tier, path, and phase styling;
- the `Goal:`, `Tier:`, `Path:` ordering and optional epic `Phases:` block;
- hanging indentation, terminal-cell-aware folding, and the 80-cell width cap;
- file-hint registration, missing-plan fallbacks, section ordering, and the cheap in-memory navigation path.

This is presentation-only Python TUI work. It does not change associated-plan discovery, plan schemas, persisted data,
the Rust core API, or event-loop behavior.

## Regression coverage

Strengthen `tests/ace/tui/widgets/test_agent_display_plan_section.py` so the spacing contract is explicit instead of
being left only to visual inspection:

- assert the logical header contains `SASE PLAN\nGoal:` and not a blank line between those values;
- assert the actual Rich-rendered lines place `Goal:` immediately after `SASE PLAN` at representative narrow and wide
  widths;
- exercise both a tale/plan summary and an epic summary so future tier-specific changes cannot reintroduce the gap;
- retain the existing ordering, style, wrapping, folding, phase-roadmap, fallback, and hot-path assertions.

The existing Agents-tab visual tests already render both plan forms. Regenerate only the intentional goldens affected by
the one-row compaction:

- `tests/ace/tui/visual/snapshots/png/agents_plan_goal_metadata_120x40.png`
- `tests/ace/tui/visual/snapshots/png/agents_epic_phase_roadmap_120x40.png`

Inspect the generated images or diff artifacts before accepting them: content below `SASE PLAN` should move up by
exactly one row, with no unrelated layout, color, font, wrapping, or panel changes.

## Verification

Because this is an ephemeral workspace and the implementation changes repository files, bootstrap dependencies first:

```bash
just install
```

Run the focused renderer tests, intentionally refresh the two visual snapshots, and then rerun those visual cases
without update mode so the committed goldens are independently checked:

```bash
.venv/bin/python -m pytest tests/ace/tui/widgets/test_agent_display_plan_section.py
just test-visual --sase-update-visual-snapshots \
  tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py::test_agents_sase_plan_metadata_png_snapshot \
  tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py::test_agents_epic_phase_roadmap_png_snapshot
just test-visual \
  tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py::test_agents_sase_plan_metadata_png_snapshot \
  tests/ace/tui/visual/test_ace_png_snapshots_agents_interactions.py::test_agents_epic_phase_roadmap_png_snapshot
```

Finally run the repository-required full gate:

```bash
just check
```

## Risks and constraints

The main risk is fixing only the retained logical `Text` or only the width-aware renderable, leaving inspection and
on-screen output inconsistent. Testing both representations at multiple widths contains that risk. A second risk is
removing the divider's deliberate spacing above the section; assertions should target only the boundary after
`SASE PLAN`, and visual review should confirm the divider remains unchanged.

No broad refactor of prompt-panel section spacing is needed. Other major sections intentionally use their own spacing
contracts and are out of scope.
