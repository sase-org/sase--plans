---
tier: tale
title: Repair remaining project and ChangeSpec presentation surfaces
goal: Eliminate canonical project-key leaks from the remaining human-facing CLI and
  TUI surfaces while preserving canonical identity in persistence, lookup, paths,
  commands, replay, and structured output.
create_time: 2026-07-20 13:14:55
status: wip
prompt: 202607/prompts/remaining_project_surfaces.md
---

# Plan: Repair remaining project and ChangeSpec presentation surfaces

## Context and contract

Phase 1 established `ProjectDisplaySnapshot` as the immutable, batch-loaded mapping from canonical project keys to
configured labels, plus helpers for project-prefixed ChangeSpec names. This phase applies that contract outside
Statistics. A deliberately mismatched project such as `gh_acme__widgets` with display name `widgets` must render as
`widgets`, while the canonical key continues to own identity, filesystem layout, ProjectSpec lookup, task metadata,
saved selections, and machine-readable fields. Missing lifecycle metadata must fall back to the canonical key, and
duplicate display labels must remain distinct through canonical identity and deterministic label/key ordering.

No Rust aggregation or persistence migration is expected. Human presentation may use projected labels, but paths, exact
repair commands, JSON payloads, prompt-stash records, launch requests, ChangeSpec files, and replay keys must retain
canonical values. TUI renderers must be pure with respect to project metadata: load or refresh one snapshot in an
existing worker/off-thread data path and pass it into row, preview, modal, and task-label builders.

## CLI and report presentation

- Repair the single-project `sase workspace list` heading by taking its visible project label from the already-collected
  inventory/project record rather than `ProjectContext.project_name`. Keep project resolution, workspace paths, warnings
  that identify exact records, and the JSON payload's `project`/`project_key` fields canonical. Add paired human/JSON
  regression coverage for a mismatched key and label.
- Project `sase doctor` through a display snapshot only when rendering the human report. Humanize the summary panel's
  project heading and the `project.current` summary, while retaining canonical `DoctorContext.project`, diagnostic
  `data`, exact ProjectSpec paths, and copyable command arguments. Cover the human Rich rendering and the unchanged JSON
  identity side by side.
- Load one display snapshot for `sase changespec search --format markdown` and use it consistently for project metadata,
  ChangeSpec headings, parents, running-claim labels, and summary quick links. Derive link targets from the same
  displayed heading representation so every generated link still lands on its humanized section. Do not mutate the
  `ChangeSpec` objects or the plain/rich/structured identity data.
- Complete the requested source audit of human CLI, Rich, Markdown, notification, and progress output that interpolates
  raw `ChangeSpec.name`, `parent`, `claim.cl_name`, `project_basename`, or lifecycle `project_name`. Apply the shared
  humanization helpers at confirmed presentation boundaries such as commit/archive/mail/revert/status progress, while
  explicitly preserving raw values used as branch revisions, workspace/session names, file paths, lookup keys, log-only
  diagnostics, and exact commands. Add narrow tests or rationale comments for every repaired or intentionally canonical
  boundary touched by the audit.

## TUI projection and identity separation

- Replace ambiguous launchable-project strings with an explicit project choice/projection carrying `project_key` and
  `project_label`. Build project and active-ChangeSpec picker rows from a preloaded lifecycle snapshot: options display
  configured project labels and humanized ChangeSpec labels, but `SelectionItem.project_name`, `cl_name`, option
  identity, deletion targets, prompt contexts, and lookups remain canonical. Sort visible choices by label with a
  canonical-key tiebreaker and keep duplicate labels independently selectable.
- Make saved selection and replay behavior canonical end to end. Persist canonical project and ChangeSpec identities,
  recompute presentation labels when loading/replaying, and safely normalize any existing display-form project reference
  through the established alias resolver before constructing ProjectSpec paths. Tests must prove selecting `widgets`
  launches and later replays `gh_acme__widgets`, including after a label change, without writing a display label into
  the identity fields.
- Extend the off-thread prompt-stash read boundary to return both the canonical stash entries and a display snapshot.
  Pass that snapshot into the restore and update-pinned modals, row chips, and preview metadata so they render project
  labels without disk access. Keep the stash wire schema and restore/update operations keyed solely by the original
  entry IDs and canonical `entry.project`; verify the stored records remain byte/field compatible.
- During the existing launch-approval worker metadata read, resolve the request's canonical project through the same
  snapshot and return a projected task display label. Update the completed task row with that label on the UI thread
  while preserving canonical project files and request metadata. Humanize a project-prefixed ChangeSpec label only where
  the task queue presents it, never where it is used for matching or execution.
- Audit adjacent history, notification, task-queue, modal, preview, banner, and progress renderers reached by these
  flows. Reuse an already-loaded snapshot or attach an explicit label to the presentation model; do not introduce
  lifecycle reads, `stat`, globbing, JSON parsing, or subprocess work in widget composition, render methods, key
  handlers, debounced detail paints, or Textual pump callbacks.

## Focused verification and handoff

- Add mismatch-key regression tests for workspace human versus JSON output, doctor human versus structured identity,
  Markdown headings/anchors/parents/claims, picker label-versus-selection identity, persisted replay, prompt-stash
  rows/previews/storage, and completed launch-approval task labels. Include missing-record fallback and duplicate-label
  cases where the interaction can otherwise become ambiguous.
- Run the focused CLI, doctor, Markdown search, project-selection/replay, prompt-stash, and launch-approval test modules
  while iterating. Exercise any affected TUI interactions to confirm no project metadata loader is called from a render
  or navigation path.
- Run `just install` before repository-wide validation, then finish with `just check`. Do not accept or update PNG
  goldens unless an intentional visible fixture change actually requires it; leave the epic's cross-surface architecture
  audit, broad end-to-end matrix, and final visual regression sweep to Phase 4.

## Risks and boundaries

- A visible label must never become a query or storage key. Keep key and label separate in every interactive model and
  assert canonical outbound values.
- Markdown links can drift from headings if humanization is applied independently. Build both from the same projected
  label and test the exact pair.
- Cached convenience helpers can hide synchronous first-use I/O. TUI code must receive a worker-loaded snapshot
  explicitly; convenience lookup remains acceptable for non-rendering CLI paths.
- Some canonical strings are intentionally user-visible because they identify a directory, exact command argument,
  branch, or storage record. Preserve those sites and document their purpose instead of applying a blanket replacement.
