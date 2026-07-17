---
tier: tale
title: Phase bead SASE context lane
goal: 'Phase agents present their bead identity, selected-phase description, and epic
  plan provenance in a polished BEAD lane at the top of SASE CONTEXT, without blocking
  navigation or leaking the full epic roadmap.

  '
create_time: 2026-07-17 14:14:04
status: wip
prompt: 202607/prompts/phase_bead_context_lane.md
---

# Plan: Phase bead SASE context lane

## Product contract

Move positively identified phase-agent bead metadata out of the flat metadata rows and into a new `BEAD` lane that is
always the first lane in `SASE CONTEXT`. The lane will use a compact descriptive header such as `▸ BEAD · phase`,
followed by four aligned fields in this order:

1. `ID:` — the phase bead ID.
2. `Description:` — the normalized description for the selected phase, preserving the existing
   explicit-description/generated-description semantics.
3. `Epic Plan:` — the canonical workspace/SDD-relative POSIX path, such as
   `sase/repos/plans/202607/some_epic_plan_file.md`.
4. `Epic Title:` — the normalized `title` value from the epic plan frontmatter.

Do not render the old top-level `Bead:` row for an agent represented by this lane, and never render both forms in the
completed detail view. Preserve the existing top-level bead compatibility behavior for land agents and other confirmed
bead-like agents that are not identified as phase workers; this change should not imply a broader redesign of those
identities.

The lane should remain useful under partial failure. Keep a trustworthy ID even when plan enrichment is unavailable,
show quiet `unavailable` values for missing description/title data, and show a known relative plan path with the
established missing-file treatment when the target cannot be read. A malformed or outdated epic must not expose
unrelated phase data or the full roadmap.

## Typed phase-bead enrichment

Replace the phase path's fused `"id - description"` presentation value with an immutable, render-ready phase-bead
summary in the associated-plan model. Carry the ID and description independently, plus separate actual and display
epic-plan paths, plan existence/readability state as needed for honest fallbacks, and the epic title. Keep the resolved
actual path available for artifact de-duplication and file opening while exposing only the canonical relative display
path in the panel.

Build this summary inside the existing deferred detail-header enrichment worker. Reuse the role-aware plan association,
validated epic phase ordering, canonical plan selection/path resolution, and mtime-and-size-keyed frontmatter cache
instead of introducing another read or cache. Modern phase launch metadata must remain authoritative and must not
consult bead storage; legacy records may use the existing bounded compatibility lookup only when needed to confirm their
role/association. Continue suppressing `AssociatedPlanSummary` for phase agents so the `PLAN` lane cannot reveal the
complete epic roadmap.

Normalize the title and description once during enrichment. For a valid selected phase, retain the current precedence of
an authored phase `description` followed by the generated phase description. For missing, invalid, or out-of-range phase
metadata, fail closed to the bare ID and unavailable optional fields. Derive the display plan path from the canonical
SDD/workspace reference as a relative POSIX path while retaining the resolved absolute target only in memory for
navigation.

Thread the typed phase-bead summary through `AgentPlanEnrichment` and `DetailHeaderSummary`. Keep the generic cached
bead-display value independent so non-phase compatibility behavior and list badges continue to work, and ensure
detail-summary cache keys still cover every launch field that can change the phase/epic association.

## Responsive BEAD lane

Add a focused responsive BEAD-lane renderable alongside the existing responsive PLAN lane. Use an amber/gold bead
palette that is visually distinct from PLAN while harmonizing with the existing bead accent; use the shared subdued
styles for labels/fallbacks and the established artifact-path styling for the plan basename. Align all four colons in
one label column, preserve complete values, and fold long descriptions, titles, paths, long tokens, spaces, and wide
Unicode with hanging indentation. Cap the wide logical layout consistently with the existing 80-cell descriptive-lane
contract rather than truncating content.

Register `Epic Plan` as a file hint using its resolved actual path. Because BEAD is the first context lane, its hint
must also be allocated before hints from PLAN, ARTIFACTS, MEMORY, or later lanes. A path with spaces must remain both
readable and openable.

Extend the declared context order to `BEAD`, `PLAN`, `ARTIFACTS`, `MEMORY`, `SKILLS`, `WORKSPACES`, and add the BEAD
renderer to the same presence-driven composition used by every other lane. The lane must create `SASE CONTEXT` when it
is the only available context and remain separated from following lanes by the existing single compact gap. Preserve the
responsive PLAN range bookkeeping when BEAD precedes it.

Update the agent header composition to construct the BEAD section only from the precomputed summary. The immediate/cheap
selection path must stay memory-only and perform no stat, frontmatter parse, or bead lookup; the existing debounced
detail worker supplies and caches the enriched lane. Ensure selection changes and stale worker completions retain their
current last-selection-wins behavior.

## Verification and visual acceptance

Add model tests for modern and confirmed legacy phase roles, nested epic IDs, authored and generated descriptions,
normalized frontmatter titles, exact relative-versus-actual paths, cache invalidation after plan edits, and missing,
unreadable, malformed, and out-of-range plans. Assert modern phase enrichment never touches bead storage and that phase
plan files remain excluded from the generic ARTIFACTS lane.

Add renderer and header integration tests covering:

- BEAD-first ordering for every lane-presence combination and BEAD-only context;
- exact field order, aligned labels, palette, quiet unavailable/missing states, compact lane spacing, and absence of the
  old phase `Bead:` row;
- complete lossless wrapping at wide and narrow widths, including spaces, long unbroken tokens, and wide Unicode;
- file-hint ordering and actual-path mappings, including paths with spaces;
- the cold/cheap path, deferred enrichment result, stale-selection protection, no duplicate PLAN lane/full epic phase
  list, and preserved non-phase bead behavior.

Add or adapt an Agents-tab PNG snapshot around a realistic phase agent so the final hierarchy, spacing, colors,
wrapping, and relative epic-plan path receive visual review. Accept a new golden only after inspecting the generated
actual/expected/diff artifacts. Run `just install` first as required for an ephemeral workspace, then the focused
model/widget/async tests and dedicated visual snapshot, followed by the repository-mandated `just check`.
