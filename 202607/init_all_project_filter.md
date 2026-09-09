---
tier: tale
goal:
  Ensure `sase init --all` visits only enabled, valid, non-system SASE projects and
  ignores stale or telemetry-only directories under the project inventory root.
create_time: 2026-09-09 19:53:15
status: wip
---

# Plan: Filter `init --all` to true SASE projects

## Context

The Rust project inventory deliberately discovers every directory beneath the SASE
projects root so diagnostic and cleanup surfaces can report malformed or stale entries.
A directory without an active ProjectSpec receives the default lifecycle state
`enabled`, but the inventory also classifies it as `is_project: false`. The
`sase init --all` resolver currently requests enabled records without applying the
inventory's true-project filter, so telemetry-only directories such as those containing
only skill-use logs are treated as initialization targets and then reported as
unavailable.

The shared backend already exposes the correct `projects_only` query and classifies
these entries correctly. Other project-facing consumers use that classification, so no
Rust core or lifecycle semantics change is needed.

## Implementation

- Update the `init --all` project-scope resolver to request only true projects from the
  core facade and defensively honor each returned record's `is_project` classification
  while preserving the existing enabled-state, non-home, non-system, display-name
  sorting, and per-project availability behavior.
- Strengthen the focused project-scope test fixture so it can represent a non-project
  inventory record, add a stale/telemetry-only enabled directory case, and verify both
  that it is excluded and that the resolver asks the facade for `projects_only=True`.
- Keep unavailable handling for genuine projects unchanged: an actual project whose
  workspace later disappears should still be surfaced as unavailable rather than
  silently omitted.

## Validation

- Run the focused `init --all` project-scope and batch-onboarding tests.
- Run `just install` followed by the repository-required `just check` suite.
- Re-run the real inventory comparison (unfiltered enabled records versus enabled
  projects-only records) and exercise `sase init --all --check` from the installed
  workspace version to confirm only the three valid enabled projects are considered and
  the four stale directories no longer affect the summary or exit status.

## Risks and non-goals

The change intentionally does not delete stale directories or suppress inventory
diagnostics in commands designed to expose malformed entries. It only aligns the batch
initializer with its documented scope. Filtering must occur before target conversion so
non-project warnings do not leak into onboarding output, while true projects with
legitimate availability problems remain visible.
