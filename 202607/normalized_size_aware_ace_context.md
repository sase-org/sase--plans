---
tier: tale
title: Normalized size-aware ACE agent context
goal: 'ACE presents the authoritative small, medium, or large scope of every epic
  phase to authors, landers, and the selected phase worker, while preserving legacy
  compatibility, phase-local privacy, responsive rendering, and the memory-only Agents
  navigation path.

  '
create_time: 2026-07-20 14:11:41
status: done
prompt: 202607/prompts/normalized_size_aware_ace_context.md
---

# Plan: Normalized size-aware ACE agent context

## Context and outcome

Epic phase size already participates in validation, bead creation, and worker routing, but the Agents-tab projections
discard it before rendering. This change will carry the normalized value returned by the Rust-backed plan validator
through the existing immutable, file-signature-cached enrichment model. Epic authors and landers will see a labeled size
chip on every PLAN roadmap phase, while a phase worker will see only its own phase size in the BEAD lane. No roadmap,
dependency, goal, or peer-phase metadata will cross the phase-worker information boundary.

The display contract will use launch-consumption validation for authored epic context. Modern plans retain their exact
`small`, `medium`, or `large` values; valid historical plans with omitted phase sizes normalize to `small`; explicit
invalid sizes and unrelated schema damage continue to make phase metadata unavailable. All file reads, parsing, and
validation remain inside the deferred enrichment worker and its mtime/size cache, leaving immediate selection and
rendering memory-only.

## Normalized enrichment model

- Extend the frozen associated-plan phase projection with a typed phase size and populate it directly from
  `ValidatedPlanPhase.size` in the cached plan loader.
- Invoke the validator in launch-consumption mode from that loader so compatibility normalization is centralized in the
  authoritative backend contract instead of duplicated in ACE. Preserve all-or-nothing phase availability for invalid
  epic documents and retain the existing cache signature, bounded capacity, negative TTL, and exception degradation
  behavior.
- Extend `PhaseBeadSummary` with an optional selected-phase size. Populate it only when the exact phase ordinal maps to
  a successfully validated parent-plan entry; missing/unreadable/damaged plans, invalid sizes, and out-of-range phase
  IDs retain the existing identity/path fallbacks and report size as unavailable.
- Keep author, lander, modern phase, and legacy phase association behavior intact, including direct-reference priority,
  bounded compatibility bead lookups, family propagation, resolved-path deduplication, and distinct phase-authored PLAN
  artifacts.

## Shared accessible size presentation

Introduce a small pure Rich helper for phase-size normalization and chips using the established epic clan-summary
palette: blue `small`, gold `medium`, and rose `large`, with the literal word as the accessible primary signal. Give the
helper an honest unavailable representation for phase-local BEAD rows; do not reinterpret invalid authored data as a
known size. Keep the helper independent of filesystem, validation, bead queries, or subprocess work so later ACE
surfaces can reuse the same vocabulary safely.

In the responsive PLAN roadmap, add a fixed-width size chip to each phase title row. The ordinal and diamond remain the
structural prefix, the title is the only flexible column, and the chip remains visible at narrow widths and in metadata
zoom. Preserve complete Unicode-aware folding with no ellipses and keep phase order, metadata, descriptions, file-hint
numbering, and static-versus-live semantics unchanged. Include the size label in the unwrapped logical text so search,
copy, style inspection, and header splicing observe the same content as width-aware Rich rendering.

In the phase-local BEAD lane, add `Size` alongside Description, Epic Plan, and Epic Title, using the same chip styles
for known values and a quiet `unavailable` value otherwise. Recompute the common label width from the complete field set
so wide and narrow hanging indentation remains aligned. Do not add a PLAN lane or expose any other epic phase to a
worker.

Where it can be done without changing the clan summary's standalone behavior, serialized markup, or pixels, point that
summary at the same shared palette/chip helper; otherwise leave the launch script stable and lock palette equivalence
with tests rather than expanding this phase's risk.

## Coverage and documentation

Expand model/cache tests to cover all three modern sizes, missing legacy sizes normalized to `small`, explicit invalid
sizes remaining unavailable, cache reuse and signature invalidation, author and lander role resolution, exact
phase-ordinal selection, out-of-range and unreadable plans, and the absence of bead-store reads on modern phase paths.
Assertions for frozen summaries will include size so no projection can silently drop it again.

Expand PLAN and BEAD rendering tests for exact labels and foreground/background styles, logical-text visibility, field
ordering/alignment, phase-worker isolation, and narrow/wide rendering with long ASCII tokens and wide Unicode. Verify
that only titles fold around fixed PLAN chips, values remain complete without ellipses, metadata zoom/search can see
sizes, path hints retain their numbering, invalid epics keep one quiet unavailable state, and tale plans remain
unchanged.

Update the focused Agents epic-roadmap visual fixture to show `small`, `medium`, and `large` chips and the phase-context
fixture to show only the selected phase's size. Regenerate only the intentional PNG goldens and inspect actual,
expected, and diff artifacts before accepting them. Amend `docs/ace.md` in both the overview and detailed Agents
metadata contract to explain phase sizes, launch-mode legacy fallback, unavailable behavior, fixed responsive chips, and
phase-worker privacy.

## Validation and acceptance

Run `just install` before repository validation. Exercise the focused associated-plan model/cache tests, PLAN and BEAD
rendering/responsiveness tests, header enrichment and search/zoom tests affected by logical text, the two targeted PNG
snapshot cases, and the Agents j/k slow benchmark or trace to confirm selection still consumes cached memory-only state
and stays within the established p95 target. Then run `just test-visual` and the mandatory `just check`.

Review the final diff for accidental Artifacts Plans or backend scope, inspect every changed PNG, and confirm the bead
can be closed without closing its parent epic. Acceptance requires exact modern sizes, only missing legacy size falling
back to `small`, malformed explicit data remaining unavailable, identical logical and rendered semantics, phase-local
worker context, no render-time I/O or validation, and no responsiveness regression.
