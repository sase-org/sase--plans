---
tier: tale
title: Size-aware Artifacts Plans surface
goal: 'The Artifacts Plans pane shows persisted phase sizes clearly in compact rows
  and bead details, summarizes epic child sizes without extra loading, and preserves
  complete authored plan properties, responsiveness, and legacy behavior.

  '
create_time: 2026-07-20 14:48:12
status: wip
prompt: 202607/prompts/artifacts_phase_sizes.md
---

# Plan: Size-aware Artifacts Plans surface

## Context and contracts

The Artifacts Plans worker snapshot already contains each direct phase bead as an `Issue`, including its persisted
`size`, plus the direct-child grouping for each epic. Those bead values are the execution-time truth for this surface
and may intentionally differ from the size authored in linked plan YAML. Rendering must therefore consume only the
immutable snapshot already delivered by the existing off-thread loader. It must not read files, revalidate plans, query
the bead store, or reconcile authored and persisted values on the UI thread.

The preceding epic phase introduced the shared accessible Rich presentation vocabulary: literal blue `small`, gold
`medium`, and rose `large` chips. A persisted legacy phase with no size uses the same explicit `small` fallback as work
routing and existing epic summaries. Proposal, archive, and linked-plan views remain lossless authored-data surfaces;
their existing `Phases` property must retain each phase's authored size exactly once rather than gaining a second
synthesized roadmap.

## Compact phase rows

Integrate the shared phase-size chip helper into expanded phase row rendering. Keep the existing single-line prefix
order—indentation, status, bead ID, and readiness—then place the fixed size chip immediately before the flexible phase
title. Preserve stable option IDs, readiness glyphs, filtering, expansion, focus, jump hints, and the current one-line
`OptionList` behavior. Exercise all three stored sizes and a missing legacy value, including exact palette styles and
narrow rendering where fixed identity/readiness/size context remains ahead of title ellipsis.

## Bead detail properties

Extend the existing property-grid builder without changing the snapshot model or loader. A phase bead gets a `Size`
property rendered with the shared chip and the explicit legacy `small` fallback; non-phase issues do not get that
property. An epic bead gets a compact `Phase sizes` count assembled from its already-loaded direct children in fixed
`small`, `medium`, `large` order, omitting empty buckets and omitting the property when no child context is available.
Keep status, readiness, dependency state, debounced selection, and linked-plan body rendering unchanged. Cover mixed and
repeated sizes, singular and plural grammar, legacy fallback, phase-only `Size`, fixed ordering, and absence when child
context is unavailable.

## Authored plan preservation audit

Strengthen proposal, archive, and worker-loaded linked-plan tests with valid epic frontmatter containing small, medium,
and large phases. Assert their complete flattened `Phases` property continues to associate every authored size with its
phase and appears exactly once in the relevant detail/source view. Do not parse flattened archive strings into a
TUI-only schema and do not add a redundant roadmap or phase-count projection to these generic authored property
surfaces.

## Visuals, documentation, and verification

Update the populated Artifacts Plans visual fixture so expanded rows and the selected phase detail visibly exercise the
new size presentation, regenerate only the intentional PNG golden, and inspect the actual/expected/diff image artifacts
before accepting it. Amend the Artifacts section of `docs/ace.md` to explain that expanded rows and bead details use the
current persisted bead size with a legacy-small fallback, while proposal/archive/linked-plan properties continue to show
authored values.

Before validation, run `just install` as required for an ephemeral workspace. Run focused shared-helper, Plans
rendering/data/linked-document, navigation, and visual tests; run the Artifacts j/k benchmark to confirm the established
p95 budget remains below 16 ms; then run `just test-visual` and the mandatory `just check`. Review the final diff for
accidental loader/backend changes and verify only `sase-8b.2` is closed after all checks pass, leaving parent epic
`sase-8b` open.

## Risks and safeguards

- Chip width competes with long titles, so fixed metadata must precede the only ellipsizing content and be protected by
  narrow-width assertions.
- A missing persisted size is legitimate legacy state, not corrupt authored YAML; normalize only that case to `small` on
  this bead-backed surface.
- Epic summaries must count the immutable direct children once and avoid any file or store access in render/detail
  paths.
- Generic authored-property views already contain phase sizes. Regression coverage protects completeness without
  introducing duplicate UI or a divergent parser.
