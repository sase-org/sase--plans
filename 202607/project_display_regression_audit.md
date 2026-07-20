---
tier: tale
title: Lock project display names across human-facing surfaces
goal: Add reusable mismatch-key coverage and a documented presentation-boundary audit,
  then prove every repaired surface preserves canonical identity while rendering configured
  labels.
create_time: 2026-07-20 14:11:18
status: wip
prompt: 202607/prompts/project_display_regression_audit.md
---

# Plan: Lock project display names across human-facing surfaces

## Context and success contract

Phases 1–3 established an immutable `ProjectDisplaySnapshot`, projected every Statistics project-bearing view model, and
repaired the confirmed CLI and TUI leaks. This final phase must make those repairs durable. The regression case is
deliberately asymmetric: a canonical key such as `gh_acme__widgets`, configured label `widgets`, and canonical
ChangeSpec identity such as `gh_acme__widgets_feature` must render as `widgets` / `widgets_feature` on human-facing
surfaces while retaining the canonical strings in filters, joins, persistence, paths, task metadata, replay state, and
machine-readable output.

The work stays in the Python presentation and test layers unless the audit exposes a concrete missed presentation
boundary. It must not change Rust aggregation, wire schemas, project lifecycle storage, ChangeSpec storage, or other
machine contracts. Missing project metadata continues to fall back to the canonical key, duplicate labels remain
distinct through canonical identity, and TUI render/navigation paths must not load, stat, glob, parse, or invoke a
subprocess to resolve display metadata.

## Reusable mismatch-key test model

- Introduce one shared test fixture/factory that owns the canonical project key, configured label, canonical and
  projected ChangeSpec names, immutable display snapshot, and any lifecycle record/project layout builders needed by
  downstream tests. Keep filesystem construction opt-in so pure renderer tests remain fast and do not accidentally
  perform metadata I/O.
- Adopt the shared case in the cross-surface regressions instead of repeatedly spelling unrelated local constants. The
  fixture must make `key != label` unavoidable and provide explicit access to both forms, so an assertion cannot pass
  merely because identity and presentation happen to match.
- Preserve or extend focused fallback and duplicate-label cases where those semantics matter. The common fixture is the
  baseline mismatch case, not a replacement for tests of missing metadata, label collisions, or rename refreshes.

## Presentation-boundary audit and regression guard

- Re-run the source audit from canonical project and ChangeSpec fields (`project`, `project_key`, `project_name`,
  `project_basename`, ChangeSpec names/parents, and running-claim names) into human-facing sinks such as Rich/Text
  construction, table rows, option labels, headings, notifications, task labels, Markdown, and ordinary human CLI
  output. Review every remaining match as human-facing, identity-only, or intentional dual display.
- Add a narrowly scoped architecture regression test for the audited active renderers. Prefer syntax-aware inspection
  over a blanket substring ban: flag newly introduced direct rendering of identity-bearing fields, recognize projection
  through the shared snapshot/humanization helpers, and keep explicit exemptions for exact paths, command tokens,
  structured/machine output, and intentionally canonical diagnostic fields. Every exemption must name the site and carry
  a concise rationale so additions require deliberate review.
- Include a purity guard for the repaired TUI boundaries: render/compose/navigation code consumes an already-loaded
  snapshot or explicit label, while lifecycle snapshot loading remains in the established worker/off-thread loaders.
  Scope this guard to the audited project-display flows so unrelated canonical identity code is not prohibited.
- Repair any unprojected human sink found by the audit and add a focused behavioral assertion for it. Keep canonical
  values untouched at all identity boundaries and avoid speculative refactors outside this epic's surfaces.

## Cross-surface behavior matrix

- Exercise Statistics end to end from canonical query payloads through view models and renderers: Overview Top projects,
  filter choices and active heading, project and ChangeSpec tables/drilldown, workspace activity, and project and
  ChangeSpec runtime grouping. Assert the display label is visible, the canonical key is absent from human text, and
  filter submission, joins, stable colors, and refresh state still use the canonical key.
- Exercise the repaired command/report surfaces with paired assertions: the single-project workspace heading and Doctor
  report use `widgets`, while their JSON/data/path identities remain canonical; ChangeSpec Markdown headings, anchors,
  project metadata, parents, and running claims use the projected names without mutating the underlying `ChangeSpec` or
  claim records.
- Exercise the interactive TUI flows with the same mismatch case: project and ChangeSpec choices display projected
  labels while selection/replay persists canonical values; prompt-stash rows and previews render the label while stash
  records remain canonical; launch-approval completion updates only the task display label while request metadata,
  ChangeSpec identity, and project paths stay exact.
- Keep the suite decomposed by surface where existing helpers and interaction harnesses are strongest, but tie it
  together through the shared fixture and consistent label-versus-identity assertions. This supplies cross-surface
  coverage without creating one brittle monolithic UI test.

## Validation and visual/performance verification

- Run the focused projection, Statistics view/pane, workspace, Doctor, Markdown search, project selection/replay,
  prompt-stash, and launch-approval tests while iterating. Run the architecture audit independently so its diagnostics
  identify the exact unclassified sink.
- Exercise the existing Statistics refresh/soak coverage with the mismatch fixture and verify no lifecycle loader is
  reached from renderer or navigation paths. Preserve canonical last-request-wins filter state and the no-stall
  expectations from the TUI performance contract.
- Run the dedicated PNG visual suite after the behavioral matrix passes. The Statistics and project-selection fixtures
  already use mismatched keys; inspect actual/expected/diff artifacts for any failure and update a golden only when the
  change is intentional and reviewed.
- Run `just install` before repository-wide validation, then run the focused suites again as needed and finish with
  `just check`. Re-run the presentation audit after any fixes, and close only `sase-89.4` once all targeted, visual,
  performance-sensitive, and full checks pass; leave parent epic `sase-89` open.

## Risks and safeguards

- A source guard can become noisy or cosmetic. Restrict it to known presentation sinks, use syntax-aware matching, and
  require documented exemptions rather than banning canonical strings globally.
- A shared fixture can hide identity assertions. Expose canonical and projected forms separately and assert both the
  visible result and the unchanged outbound/stored value at every interactive boundary.
- Visual tests can mask unintended churn if goldens are accepted mechanically. Treat existing goldens as authoritative
  unless inspected diffs prove an intentional label-only change.
- Test helpers must not weaken the production performance contract. Keep metadata creation in fixtures/loaders and
  explicitly fail if audited TUI renderers attempt lifecycle I/O.
