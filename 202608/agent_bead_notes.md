---
tier: tale
title: Display bead notes in the agent metadata BEAD lane
goal:
  Show every persisted note for the selected agent's corresponding bead in a readable, responsive BEAD lane while
  leaving note-free lanes visually unchanged.
proposed_by: bbugyi200.athena.rf
create_time: 2026-08-01 10:30:05
status: wip
---

- **PROMPT:** [202608/prompts/agent_bead_notes.md](prompts/agent_bead_notes.md)

# Display Bead Notes in the Agent Metadata BEAD Lane

## Why this is a tale

This is one cohesive presentation feature in an established pipeline: project a bead's existing `Issue.notes` field into
the deferred agent-detail summary, then render it in the existing responsive BEAD lane. One coding agent can implement
and verify the model, enrichment, rendering, and tests together. There are no independently deployable phases, new bead
semantics, Rust-core API changes, or cross-repository dependencies that would justify an epic.

## Current state and constraints

- The selected-agent metadata panel builds `SASE CONTEXT` from a deferred `DetailHeaderSummary`; expensive plan and
  bead-store work already runs in a `thread=True` worker rather than in header rendering or the j/k selection path.
- `BeadSummary` is the immutable, render-ready value consumed by `ResponsiveBeadSection`. Task summaries come from a
  persisted `Issue`; phase summaries primarily come from validated epic-plan frontmatter. Modern phases with explicit
  parent-plan metadata currently avoid bead lookup entirely.
- `ResponsiveBeadSection` already provides an 80-cell cap, a fixed label column, lossless folding, and hanging
  indentation. Task lanes show `Task Title`, `Description`, and optional `Size`; phase lanes show `Phase Title`,
  `Description`, `Size`, `Epic Plan`, and `Epic Title`.
- Bead notes are stored as one ordered string. Canonical `sase bead note` entries are separated by blank lines and
  include their `[timestamp · author]` attribution, but older or manually replaced note fields may contain arbitrary
  multiline prose.
- The no-notes appearance is a compatibility requirement: do not add `Notes: unavailable`, blank rows, extra separators,
  changed alignment, or updated note-free visual goldens.
- This is TUI-specific projection and presentation of an existing core field. Keep bead storage semantics in the
  existing backend and use the established local-only lookup adapter; do not add parallel parsing or mutation behavior
  to the TUI.

## User experience contract

1. When the corresponding task or phase bead has nonblank notes, insert one `Notes:` field immediately after
   `Description:` and before `Size` or any provenance fields.
2. Render the complete notes value in stored order. Preserve intentional line breaks and the blank line between
   attributed entries; trim only meaningless outer whitespace and normalize platform line endings.
3. Treat note text as literal content, not Rich markup. Do not parse or rewrite attribution prefixes: canonical notes
   remain naturally readable, and legacy/free-form notes cannot be dropped by a brittle parser.
4. Reuse the existing muted prose color and aligned field-label palette so notes read as supporting context rather than
   competing with the bead title. Let the responsive table fold long prose and oversized Unicode tokens losslessly, with
   continuation lines hanging under the value column and the existing 80-cell maximum.
5. When notes are empty, blank, unavailable, or the local bead lookup fails, omit the field entirely. The lane's logical
   text, rendered lines, label width, spacing, styles, hint numbering, and surrounding lane order must remain identical
   to today.
6. Notes are informational only. Do not add editing, folding, truncation, pagination, keybindings, or file hints in this
   change.

## Implementation

### 1. Carry optional notes in the render-ready bead projection

- Extend `BeadSummary` in `src/sase/ace/tui/models/_agent_associated_plan_types.py` with an optional `notes` value whose
  default is `None`, preserving compatibility for existing phase-focused constructors and integrations.
- Add a presentation-only normalization helper near the associated-plan projection code. It should normalize CRLF/CR
  line endings to LF, strip outer whitespace, return `None` for an empty result, and otherwise preserve all internal
  text and blank lines. Do not use `normalize_bead_text`, which intentionally collapses content to one line and would
  destroy note boundaries.
- Extend `ResolvedPlanAssociation` with the normalized notes found on the exact persisted issue. Keep this separate from
  the plan path and task summary so phase metadata can remain plan-authoritative while still gaining bead-owned notes.
- Populate task `BeadSummary.notes` in `_task_bead_summary()` from the already-loaded `Issue`; this must not introduce
  another bead-store or plan-file read.

### 2. Enrich phase summaries without weakening their authoritative metadata

- In `src/sase/ace/tui/models/agent_associated_plan.py`, capture normalized issue notes whenever
  `_resolve_bead_plan_association()` confirms an issue, and retain them in the cached association on both path-bearing
  and pathless outcomes.
- Thread optional notes through `resolve_phase_plan_enrichment()`, `phase_bead_summary()`, and
  `unavailable_phase_bead()` in `src/sase/ace/tui/models/_agent_associated_plan_phase.py`.
- For a legacy phase whose bead association is already required to find its parent epic plan, reuse that same
  association and lookup session; do not reopen the store just to fetch notes.
- For a modern phase with an explicit `phase_bead_id` and direct `epic_plan_ref`, perform an exact, local-only issue
  association lookup inside the existing deferred detail worker solely to obtain notes. Never let missing, stale,
  wrong-type, or unreadable bead data replace the explicit phase identity or the title, description, size, and
  provenance derived from validated parent-plan metadata. A failed lookup simply produces `notes=None` and today's lane.
- Reuse the bounded association cache and `BeadIssueLookupSession` so repeated selection/refresh work does not reopen
  the same bead store. Adjust the cache's positive/negative payload check if necessary so a pathless association
  carrying notes is not treated as an absent result. Preserve the existing short revalidation behavior for missing/blank
  data and bounded positive TTL for populated data.
- Update stale documentation and tests that say modern phases never consult bead storage: the new invariant is that
  modern phase _structure_ never depends on bead storage, while optional notes may be enriched off-thread from the exact
  local bead.
- Keep `associated_plan_cache_key()`, detail-summary caching, and semantic digest behavior coherent. A refreshed summary
  whose notes changed must compare differently so search/hint documents and the visible header can repaint, without
  adding stats or reads to render-time code.

### 3. Render a conditional, responsive `Notes` field

- In `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_section.py`, add `Notes` to the known label set without changing
  `BEAD_FIELD_LABEL_WIDTH` (it is shorter than the current maximum label).
- Build task and phase row sequences so the notes row is inserted after `Description` only when `summary.notes` is
  present. Do not synthesize an unavailable value.
- Construct note content with `rich.text.Text` and the existing supporting-prose style, so brackets and other
  markup-like characters remain literal. Use the same grid cell and `overflow="fold"` path as the other complete BEAD
  values; do not pre-wrap, truncate, summarize, or flatten notes.
- Leave `SASE CONTEXT` lane ordering, family/clan digest presentation, file hints, and the legacy compact `Bead:`
  metadata row unchanged. Clan aggregation may carry the enriched immutable summary, but this tale does not expand clan
  BEAD digests into note bodies.

### 4. Lock down behavior with model, renderer, async, and visual coverage

- Extend associated-plan model tests to prove:
  - task notes are copied from the already-resolved issue with entry order and internal blank lines preserved;
  - legacy phase lookup reuses its association and attaches the exact phase's notes;
  - an explicit modern phase can add notes without changing plan-derived title/description/size/path/title metadata;
  - a missing or failed modern-phase issue lookup leaves the existing plan-derived summary intact and note-free;
  - cached association behavior avoids repeated store opens and can surface refreshed notes after its established
    revalidation boundary.
- Extend `tests/ace/tui/widgets/test_agent_display_bead_section.py` to prove:
  - both task and phase lanes place `Notes:` immediately after `Description:`;
  - two attributed entries, multiline/free-form prose, blank separators, Unicode, and markup-like brackets remain
    complete and ordered;
  - wide and narrow rendering stays within the existing width cap, folds losslessly, and hangs continuation lines under
    the value column;
  - blank/whitespace-only notes omit the row and produce the exact pre-feature logical and rendered lane shape,
    including the same field-label width.
- Extend deferred-enrichment tests to show that a cold panel gains the notes row only after the thread worker completes
  and that cheap/hot header construction still performs no bead-store I/O.
- Add a dedicated 120x40 PNG visual case for a task BEAD lane with two realistically attributed notes, including one
  line long enough to wrap. Assert the semantic SVG text and inspect the new golden for hierarchy, alignment, spacing,
  and readability.
- Re-run the existing note-free phase BEAD PNG snapshot without accepting any update. Its unchanged golden is the visual
  regression proof for the user's no-notes requirement; do not regenerate unrelated snapshots.

## Verification

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run the focused associated-plan, BEAD renderer, context, async-enrichment, and affected visual tests while iterating.
3. Create only the new notes-present PNG golden with `--sase-update-visual-snapshots`, inspect the generated image (and
   diff artifacts if applicable), then rerun the visual test without the update flag. Confirm the existing note-free
   BEAD golden has no diff.
4. Run `just test-visual` to catch cross-snapshot layout regressions.
5. Run the mandatory repository-wide `just check` and resolve every caused failure before handoff.

## Acceptance criteria

- Selecting an agent with a task or phase BEAD lane shows every nonblank persisted note for that exact bead under a
  single `Notes:` label.
- Notes retain their original order, attribution text, paragraphs, literal characters, and full content at both wide and
  narrow terminal widths.
- Empty, whitespace-only, missing, or unreadable notes add no visible row and make no visual change to the existing
  lane.
- Phase notes never override or suppress authoritative phase-plan metadata, and lookup failure degrades silently to
  today's UI.
- Bead-store access remains local-only, bounded/cached, and confined to the existing deferred worker; cheap rendering
  and j/k navigation remain memory-only.
- Existing note-free visual snapshots pass unchanged, the new notes-present snapshot is intentionally reviewed, and
  `just check` passes.
