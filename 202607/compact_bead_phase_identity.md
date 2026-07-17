---
tier: tale
title: Compact phase bead identity in the ACE context lane
goal: Show a phase bead's ID beside the phase label in the BEAD lane header and remove
  the redundant ID field row without weakening responsive rendering or coverage.
create_time: 2026-07-17 17:43:30
status: done
prompt: 202607/prompts/compact_bead_phase_identity.md
---

# Plan: Compact phase bead identity in the ACE context lane

## Context and desired presentation

The Agents detail panel currently renders a selected epic phase as a `BEAD` SASE CONTEXT lane whose header says
`BEAD · phase`, followed by a dedicated `ID:` row. Compact the identity into the header so the logical presentation is
`BEAD · phase <bead-id>`, with the existing `Description`, `Epic Plan`, and `Epic Title` rows following immediately. The
bead ID must appear exactly once, retain the lane's primary bead styling, and remain available in both the logical
header text and Rich's responsive render output.

This is presentation-only work. Keep phase-bead discovery, typed summaries, cached enrichment, plan-path hints, and
phase-versus-plan suppression behavior unchanged. The render path must remain memory-only and must not add I/O or any
new refresh work.

## Renderer update

Update the responsive phase BEAD section in `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py` so its lane
header can compose mixed-style detail text: the word `phase` uses the existing summary style, a single separating space
follows it, and `summary.id` uses the existing primary bead style. Continue routing the assembled details through the
shared context-lane header helper so the BEAD glyph, label, separator, and newline conventions remain consistent with
other lanes.

Remove `ID` from the section's field rows. Preserve the current label width and two-column folding behavior for the
three remaining fields, including lossless wrapping, hanging indentation, file hint registration, and missing or
unreadable plan states. Ensure `logical_text` and `__rich_console__` derive the same header and field ordering so
inspection, responsive substitution, and on-screen output cannot diverge.

## Regression coverage and visual acceptance

Revise the focused BEAD renderer tests to assert the exact `BEAD · phase <bead-id>` ordering and styles, the absence of
an `ID:` field, exactly one occurrence of the bead ID, and unchanged alignment/order for `Description`, `Epic Plan`, and
`Epic Title`. Retain the existing wide and narrow lossless-wrapping cases and cover that the moved identity remains
present without truncation in rendered output.

Update all integration assertions that currently expect `BEAD · phase` plus a separate `ID:` row, including context
composition, deferred header enrichment, phase-plan suppression, and phase-aware plan-section coverage. Preserve their
existing guarantees that the BEAD lane is enriched only on the deferred path, suppresses the full epic PLAN lane for
phase workers, and handles modern and legacy phase metadata.

Adjust the phase-BEAD ACE visual test to require the bead ID beside `phase`, explicitly reject `ID:`, and retain its
checks for the descriptive fields and private epic-roadmap suppression. Regenerate only the intentionally affected
`agents_phase_bead_context_120x40.png` golden, inspect the result for the compact one-line header and removed row, then
run the visual suite normally to verify exact acceptance.

## Verification

Run `just install` before repository checks as required for an ephemeral workspace. Exercise the focused BEAD/context
widget tests and the phase-BEAD visual snapshot while iterating. Finish with `just test-visual` against the accepted
golden and `just check`, which is mandatory for source/test changes and covers formatting, linting, typing, validation,
and the full test suite.

Acceptance is complete when a selected phase displays `BEAD · phase <bead-id>` with no `ID:` row, the remaining fields
retain their styling and responsive behavior, no phase metadata or roadmap-suppression semantics change, the inspected
PNG reflects the requested compact layout, and all checks pass.
