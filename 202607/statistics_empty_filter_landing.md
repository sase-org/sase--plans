---
tier: tale
title: Repair Statistics empty-filter recovery and land sase-8a
goal: 'A filtered empty Statistics result offers a truthful one-key escape back to
  all projects, the behavior is protected against multi-project regressions, and epic
  sase-8a is fully validated and closed with its stale Symvision allowances removed
  and its original epic plan marked done.

  '
create_time: 2026-07-20 16:00:39
status: wip
prompt: 202607/prompts/statistics_empty_filter_landing.md
---

# Plan: Repair Statistics empty-filter recovery and land sase-8a

## Context

Epic `sase-8a` redesigned the Admin Center Statistics tab through four closed child beads. Its scope controls, view
descriptions, metric legends, Overview tile navigation, contextual help, configurable keymap, responsive snapshots, and
worker-backed refresh behavior are present in the current source. The focused audit passed 110 unit/integration tests
and all nine Statistics PNG snapshots, and the metric definitions were checked against the Python view model and Rust
statistics aggregation.

One phase-2 acceptance gap remains. When an active project filter produces an empty result, the panel promises that
pressing the project-filter key will clear that filter. The current action instead advances through the retained ranked
project list, so with multiple projects a single press can select another project rather than `All projects`. The
existing test only checks the rendered sentence and therefore preserves misleading copy without proving the promised
interaction.

The audit also reviewed every non-epic commit between the first `sase-8a` commit and the current head. The
project-display regression work is already reflected in Statistics through canonical query keys plus projected labels;
the runner-priority changes do not intersect this feature; and the TUI test harness split preserves the facade used by
the final visual coverage. No other integration change is required.

## Implementation phase: make empty-filter recovery truthful

Preserve normal `p`-key cycling for populated Statistics results. When the currently loaded result is empty and a
project filter is active, make the same action clear the filter directly and reload the all-project scope. Keep the
canonical project key for query identity and the projected display label for all visible text, so this repair continues
to honor the project-label work that landed during the epic.

Add a behavioral regression with at least two retained project options. It must demonstrate that an empty filtered
result plus one effective `cycle_project_filter` key press selects `All projects`, submits an unfiltered reload, and
leaves the existing empty-state instruction truthful. Retain the existing coverage for ordinary ranked-project cycling,
range changes, configured keymaps, and display-label projection. Update help or presentation copy only if the
implemented behavior makes any current wording inaccurate.

Re-run the focused Statistics unit/integration tests, keymap/schema tests, and the Statistics PNG snapshot file. Run
`just install` first as required for an ephemeral workspace, then run `just check` for the primary-repository changes.
Inspect any intentional visual change rather than updating unrelated goldens.

## Final landing phase: close and clean up epic sase-8a

Before closing, re-show `sase-8a` and its four children and confirm the repair and all prior phase requirements remain
satisfied. Then perform the landing in this exact order:

1. Close the original epic with `sase bead close sase-8a`.
2. After it is closed, run `just symvision`. If closing expires `sase-8a` whitelist allowances, use the required
   Symvision memory-read workflow, remove the stale whitelist entries and any now-reported unused code, and rerun
   Symvision until clean. Run `just check` again after any such source or configuration edits.
3. Open the plans sidecar through the required repository workflow, resolve the PLAN path from `sase bead show sase-8a`,
   and set `status: done` in the frontmatter of the original epic plan `202607/statistics_tab_intuitive_redesign.md`
   (not this repair handoff).
4. Verify the epic is closed, the original plan says `status: done`, both repositories have only the intended changes,
   and all final validation is green.

Do not close the epic or mark its original plan done before the behavioral repair and validations pass.
