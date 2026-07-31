---
tier: tale
title: ACE TUI task bead surfaces and PNG goldens
goal: Task beads render and behave as first-class Plans and Agents pane entities, with deterministic visual coverage.
bead: sase-bg.4
create_time: 2026-07-30 20:34:23
status: done
---

- **PROMPT:** [202607/prompts/tui_task_surfaces.md](prompts/tui_task_surfaces.md)
- **PARENT:** [202607/task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202607/task_beads.md)
- **BEAD:** [sase-bg.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bg/sase-bg.4.md)

# ACE TUI task bead surfaces and PNG goldens

## Goal

Make task beads first-class on the ACE Plans and Agents surfaces without adding blocking work to render or keystroke
paths. The Plans pane must collect and present task beads in their own section, support task-aware filtering, detail,
editing, and status handling, and preserve stable row identities in both single-project and all-project scopes. The
Agents pane must recognize a task-worker bead association and render a type-aware BEAD lane instead of misclassifying
the worker as an epic land agent. The `@bead:` completion catalog and deterministic PNG snapshots must expose the same
task identity and ready-state language.

## Context and constraints

- The shared model and presentation phases already provide `IssueType.TASK`, `Status.READY`, `bead_type_presentation()`,
  `bead_type_chip()`, and the task glyph/accent (`✦`, `#D787FF`) plus the ready glyph (`◇`).
- Keep all bead-store and plan-document reads in the existing off-thread snapshot/detail-enrichment workers. List,
  detail, filter, autocomplete rendering, and navigation must remain memory-only.
- Preserve the Plans pane's immutable snapshot, cached source-key, stable option-id, selection, and filter-index
  contracts.
- Task beads are top-level. Collect task issues independently of epic/phase trees and do not invent plan-document
  associations for them.
- Continue using tracked background tasks for edits/status writes. Do not add a new launch path in this phase; task
  launching belongs to the separate task-launch phase.
- Update deterministic visual fixtures with both an open task and a ready task, regenerate only intentional PNG goldens,
  and inspect the rendered glyphs for tofu before accepting them.

## Implementation

1. Extend the Plans snapshot and collection pipeline.
   - Add a project-attributed `tasks` tuple to `PlansSnapshot`.
   - During the existing off-thread bead-store load, collect top-level `IssueType.TASK` issues, order them
     deterministically using the established timestamp/id/project ordering, and retain all active tasks plus the
     pane-consistent closed task history.
   - Keep task membership in the existing source-key/cache lifecycle and leave linked-plan loading scoped to
     epics/phases.
   - Update snapshot fixtures, cache/data tests, and all direct constructors.

2. Add task rows and the Tasks section to the Plans pane.
   - Extend `PlanRowKind`, row targets, and stable option IDs with `task`.
   - Insert a `Tasks` section between Proposals and Epics, including matched/total counts and an empty state.
   - Render task rows with the shared type accent/glyph, shared status glyph, bead id, title, compact age, and
     all-project project badge. Add the task count to the status line.
   - Give the Tasks header a concise ready-state legend using the shared ready presentation instead of duplicating
     presentation constants.
   - Export compatibility helpers through `plans_pane.py` where the existing tests and integrations consume row
     renderers.

3. Make Plans detail, filtering, and actions task-aware.
   - Render the bead type as the shared type chip in the detail property grid and render the shared size chip for tasks
     as well as phases. Mirror task size in full-screen bead preview markdown.
   - Add task records to the in-memory filter index and add `kind:task` to inline filter hints/completions. Verify
     `status:ready` is supplied exactly once by the shared status display order.
   - Admit task rows anywhere phase rows are valid for preview and edit. Pass the issue type into `BeadEditModal` so its
     title uses the shared type glyph/label.
   - Make status cycling total over `Status.READY` and type-aware: task drafts progress through ready before active
     work, while existing epic/phase behavior remains unchanged. Keep mutations on the existing tracked-task path and
     improve selection warnings to include tasks.
   - Do not let task rows enter epic-only expand, launch, or external-bug behavior.

4. Generalize the Agents-pane bead association and BEAD lane.
   - Extend associated-plan role resolution with a task role. Resolve an ambiguous bead-named worker against the
     existing off-thread/local-only bead lookup, distinguish task from epic/phase, and avoid plan-file work for a task.
   - Evolve the phase-only summary into a type-aware bead summary while keeping compatibility aliases where useful to
     limit churn. Carry the bead type, title, description, size, and phase-only parent-plan/epic metadata.
   - Render the BEAD lane header with the shared type glyph/accent and `task`/`phase` label. For tasks, show
     `Task Title`, `Description`, and optional `Size`; keep `Epic Plan` and `Epic Title` only for phases.
   - Thread the generalized summary through detail-header caching/rendering and clan aggregation without adding
     synchronous lookups or broadening cache keys incorrectly.
   - Add focused role-resolution and responsive-lane tests proving task agents are not classified as land agents and
     phase behavior remains intact.

5. Complete the remaining task identity surface and visual coverage.
   - Classify `IssueType.TASK` as `task` in the cached `@bead:` autocomplete catalog while preserving its mtime-bounded,
     read-only completion path.
   - Extend Plans fixtures with deterministic open and ready task beads, including a sized task where needed to exercise
     the detail chip.
   - Add/adjust unit and interaction assertions for section ordering, row identity, counts, project badges, filters,
     detail chips, edit modal title, status cycling, autocomplete detail, and task/phase agent lanes.
   - Regenerate the affected ACE Plans PNG goldens and inspect them to ensure `✦` and `◇` render with the pinned Fira
     Code setup. Use the documented `❖`/`◈` fallback only if inspection proves tofu, and keep shared presentation
     authorities consistent if a fallback is necessary.

## Validation

1. Run focused non-visual tests for Plans data/rendering/filtering/interactions, bead edit modal and autocomplete, and
   agent association/BEAD-lane rendering.
2. Run the dedicated Plans visual snapshot module with `--sase-update-visual-snapshots`, inspect each changed PNG, then
   rerun the module without the update flag.
3. Run `just test-visual` to ensure the complete deterministic PNG suite passes.
4. Run `just check` for formatting, lint, typing, and the full test suite.
5. Recheck `git diff --check`, review the final diff for scope, and verify the bead remains the only phase being closed.
