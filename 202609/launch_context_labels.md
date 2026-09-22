---
tier: tale
title: Rename launch-context cluster labels to MODEL and PROJECT
goal:
  'The status-row launch-context cluster reads "MODEL: <chip> · PROJECT: <chip>" instead
  of "default <chip> · current <chip>", with tests and screenshot goldens updated.'
size: small
proposed_by: bbugyi200.apollo.1h.f0
create_time: 2026-09-22 06:36:18
status: wip
---

# Rename launch-context cluster labels: `default` → `MODEL:`, `current` → `PROJECT:`

## Goal

The labeled launch-context cluster at the right edge of every ACE tab's status row
(added by `sase-14y.2`) currently reads:

    default opus@high · current +sase

Change the two dim micro-labels so it reads:

    MODEL: opus@high · PROJECT: +sase

## Scope

- **In scope:** the calm model label (`default` → `MODEL:`) and the project label
  (`current` → `PROJECT:`), plus the docstrings, tests, and screenshot goldens that show
  them.
- **Out of scope:** the override-state model label (`override`). The user only asked
  about `default` and `current`, so `override opus@high 2h · PROJECT: +sase` stays as it
  is. If the reviewer wants `OVERRIDE:` for consistency, that is a one-constant
  follow-up and can come back as plan feedback.
- Tooltips stay the same. They are sentences, not labels, and are still accurate.
- The compact density does not change, because it already hides both labels.

## Width / density impact

`"MODEL: "` is 7 cells (was 8) and `"PROJECT: "` is 9 cells (was 8), so the full cluster
keeps exactly the same width whenever the project group is shown. When no project
resolves, the full cluster is 1 cell narrower. No density thresholds, gap constants, or
artifacts-spacer logic need to change. The `full_cells` / `compact_cells` properties
already derive widths from the label constants and rendered text, so they adapt without
edits.

## Changes

### 1. `src/sase/ace/tui/widgets/launch_context_bar.py`

- Change the constants:
  - `_MODEL_LABEL_CALM = "MODEL:"`
  - `_PROJECT_LABEL = "PROJECT:"`
  - Leave `_MODEL_LABEL_OVERRIDE = "override"` unchanged.
- The label widgets render `f"{label} "`, so the trailing space still separates label
  from chip. No other code changes are needed.
- Update the module docstring's visual-grammar block and prose so they match:
  - `Full:     MODEL: opus@high · PROJECT: +sase`
  - `Override: override opus@high 2h · PROJECT: +sase`
  - `No proj:  MODEL: opus@high`
  - Change the ` ``default`` / ``current`` are dim micro-labels ... ` sentence to name
    `MODEL:` / `PROJECT:`, and change ` ``· current +sase`` group ` to
    ` ``· PROJECT: +sase`` group `.

### 2. `tests/ace/tui/test_launch_context_bar.py`

Update the literal expectations:

- `test_full_cluster_reads_label_chip_separator_label_chip`:
  `"MODEL: o3@high · PROJECT: +sase"`.
- `test_empty_project_collapses_group_without_dangling_separator`: `"MODEL: o3@high"`
  and `bar.full_cells == bar.compact_cells + len("MODEL: ")`.
- `test_calm_model_label_carries_default_tooltip`:
  `model_label.render().plain == "MODEL: "`.
- Grep the file for any other remaining `"default "` / `"current "` label literals and
  update them too. Leave the override test's `"override "` assertion unchanged.

Also grep `tests/` and `docs/` for other literal occurrences of the old cluster text
(`default opus`, `default o3`, `current +`) and update any that refer to this cluster. A
pre-plan grep found none outside the file above.

### 3. Screenshot goldens

The label text appears in the status row of every ACE tab, so many PNG goldens will
change (at least the six `launch_context_bar_*` goldens plus most full-screen ACE scenes
that are wide enough to show the full cluster).

- After the code and unit-test changes are done and passing, refresh the goldens with
  `just fix-tui-screenshots` (full inventory, no selectors) right before finishing, so
  the goldens match the final code.
- Spot-check a few updated goldens (for example
  `tests/ace/tui/visual/snapshots/png/launch_context_bar_full_density_160x40.png` and
  `launch_context_bar_override_120x40.png`) by viewing the images. Confirm that `MODEL:`
  / `PROJECT:` render dim, and that the override scene still shows the gold `override`
  label.
- If a single known-flaky golden (for example `selected_gate_shell_output`) fails to
  converge after a couple of solo retries, leave it and mention it in the final summary
  rather than looping.

## Verification

1. `just fix`
2. `pytest tests/ace/tui/test_launch_context_bar.py tests/ace/tui/test_launch_context_source.py`
3. `just check`. Per the lint/test memory note, read it before finishing. Some failures
   already existed on master before this change (usage-config / completion-snapshot /
   shard-table drift). If any appear, confirm they are unrelated to this change instead
   of chasing them.
4. `just fix-tui-screenshots` as the last step before finishing (see §3).

## Acceptance criteria

- Full-density cluster reads `MODEL: <model chip> · PROJECT: <project chip>`.
- No-project cluster reads `MODEL: <model chip>` with no dangling separator.
- Override cluster still reads `override <chip> · PROJECT: <project chip>`, with the
  gold `-override` styling.
- Compact cluster is unchanged (`<model chip> · <project chip>`).
- Unit tests updated and passing, and screenshot goldens refreshed.
