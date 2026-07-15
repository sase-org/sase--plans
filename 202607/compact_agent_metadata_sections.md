---
tier: tale
goal: Make every section in the ACE agent metadata panel start its content directly
  beneath its heading, while giving the SASE PLAN Goal value a clearly distinct neutral
  color.
create_time: 2026-07-15 18:29:38
status: done
prompt: 202607/prompts/compact_agent_metadata_sections.md
---

# Plan: Compact agent metadata sections

## Context and outcome

The selected-agent detail panel currently uses two spacing conventions. `SASE PLAN`, commits, deltas, artifacts, and
error bodies begin their content on the line immediately after the heading, while optional metadata sections and
prompt/reply sections add a redundant blank row. The duplication exists in several render paths, including the normal
Rich render, file-hint mode, pinned retry attempts, workflow-step source/output views, and top-level workflow details.
Some headings and their bodies are also separate Rich renderables, so logical string checks alone can miss a blank row
introduced by a renderable's default line ending.

Adopt one compact heading contract throughout the agent detail panel: retain the existing blank space and rule above
major sections, emit exactly one line ending after each section heading, and place the first content row or placeholder
on the next rendered line. Preserve intentional spacing inside content, such as blank lines in prompts, replies,
tracebacks, and between distinct `SASE CONTEXT` lanes.

Change the `SASE PLAN` goal value from the current warm yellow to `italic #D7D7FF`. This cool neutral keeps the goal
readable and differentiated from the gold, underlined section title while preserving the cyan `Goal:` field label and
the established styling of the remaining plan fields.

## Implementation

1. Define and use a shared prompt-panel section-heading convention in the existing rendering helpers. It should append
   the styled heading with one explicit newline and support standalone Rich `Text` chunks without allowing `Text.end` or
   `Group` boundaries to create an additional row. Keep `append_major_section_divider` responsible only for the spacing
   and rule before a major section.

2. Apply the compact convention to all agent-detail section builders that currently insert a post-heading spacer:
   - Header metadata sections such as `OUTPUT VARIABLES`, `WORKFLOW VARIABLES`, `SASE CONTEXT`, and `SLOW TOOL CALLS`.
   - Normal and file-hint views for `AGENT XPROMPT`, `AGENT PROMPT`, `AGENT REPLY`, and `AGENT CHAT`.
   - Bash/Python/parallel workflow-step headings and `STEP OUTPUT`.
   - Pinned-attempt headings (`ATTEMPT ERROR`, `AGENT PROMPT`, and the attempt reply) and top-level workflow headings
     (`WORKFLOW DETAILS`, workflow variables/inputs, `WORKFLOW STEPS`, and `AGENT PROMPT`).
   - Leave already-compact headings and field-like subsections unchanged, and preserve ordering, dividers, indentation,
     file hints, and responsive plan rendering.

3. Update the SASE plan palette constant so only the goal value uses `italic #D7D7FF`; do not change the section
   heading, field-label, tier, path, phase, missing-data, or unavailable-data styles.

## Verification

Add regression coverage at both logical-text and rendered-console levels. Assert that each affected heading is
immediately followed by its first content line (and never `heading\n\ncontent`) for representative populated and
empty-placeholder states. Cover normal and file-hint rendering, running and terminal agents, follow-up replies, pinned
attempts, bash/Python/parallel steps, and top-level workflows. For headings separated from their bodies by Rich `Group`
boundaries, render through a real `Console` at representative narrow and wide widths so implicit line endings are
exercised.

Extend the SASE PLAN tests to assert the exact new goal-value span while retaining the existing compact-heading,
responsive wrapping, and no-truncation checks. Reuse or extend the shared metadata assertion helpers where that makes
the compact contract clearer than scattered substring checks.

Run focused widget tests first, then the dedicated ACE PNG visual snapshot suite. Regenerate only the intentionally
affected agent-detail goldens and inspect the actual/expected/diff artifacts to confirm that vertical whitespace was
removed beneath headings and the Goal value changed to the cool neutral without unintended layout, divider, wrapping,
hint, or color changes. Finally run `just install` as required for the ephemeral workspace and `just check` for the
complete repository validation.

## Acceptance criteria

- Every section heading in every agent-detail render mode places its first content row directly below it, matching
  `SASE PLAN`.
- Major-section rules and spacing above headings remain intact; body-internal whitespace remains intact.
- The SASE PLAN Goal value renders in `italic #D7D7FF`, visually distinct from the gold section title.
- Logical, actual Rich-rendered, focused widget, visual snapshot, and full repository checks pass without adding
  synchronous work to the TUI render path.
