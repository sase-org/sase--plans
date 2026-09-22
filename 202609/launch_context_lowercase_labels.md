---
tier: tale
title: Lowercase launch-context labels (model:/project:)
goal: 'The ACE launch-context cluster reads "model: opus@high · project: +sase" instead
  of the uppercase MODEL:/PROJECT: labels.'
size: small
proposed_by: bbugyi200.apollo.1h.f0.f0
status: done
---

# Lowercase the launch-context cluster labels: `MODEL:` → `model:`, `PROJECT:` → `project:`

## Goal

The labeled launch-context cluster at the right edge of every ACE tab's status row
currently reads:

    MODEL: opus@high · PROJECT: +sase

Switch the two dim micro-labels to lowercase (keeping the colons) so it reads:

    model: opus@high · project: +sase

This also matches the already-lowercase override label (`override opus@high 2h`).

## Scope

- **In scope:** `_MODEL_LABEL_CALM` and `_PROJECT_LABEL` values, the module docstring
  that illustrates them, the unit-test expectations, and the one screenshot golden that
  renders the labels.
- **Out of scope:** the `override` label, tooltips, styling/colors, layout logic, and
  unrelated `PROJECT:` / `MODEL:` strings elsewhere in the repo (e.g.
  `src/sase/main/patch_current.py`, `_TIER_TO_MODEL` in providers) — those are different
  features and must not be touched.

Cell widths are unchanged (`model:` and `MODEL:` are both 6 cells; `project:` and
`PROJECT:` are both 8), so the density/fit math in the widget needs no change.

## Changes

1. `src/sase/ace/tui/widgets/launch_context_bar.py`
   - `_MODEL_LABEL_CALM = "model:"`
   - `_PROJECT_LABEL = "project:"`
   - Update the module docstring examples (the `Full:`, `Override:`, `No proj:` lines
     and the `MODEL:` / `PROJECT:` / `· PROJECT: +sase` references) to the lowercase
     forms.
2. `tests/ace/tui/test_launch_context_bar.py`
   - Update the expectations that currently assert `"MODEL: o3@high · PROJECT: +sase"`,
     `"MODEL: o3@high"`, `len("MODEL: ")`, and `"MODEL: "` to the lowercase forms.
   - Grep the test file (and `tests/ace/tui/visual/`) for any remaining `MODEL:` /
     `PROJECT:` literals tied to this widget and update them too.

## Verification

1. Run the targeted unit tests:
   `.venv/bin/pytest tests/ace/tui/test_launch_context_bar.py`.
2. Run `just check` (read the `lint_and_test` memory first). Known pre-existing failures
   on master (e.g. `tests/ace/tui/test_agent_cleanup_panel_clan_members_e2e.py`, and the
   `selected_gate_shell_output` visual flake) are unrelated; confirm via `git stash` A/B
   if in doubt rather than chasing them.
3. As the last step before finishing, run `just fix-tui-screenshots` so the PNG goldens
   match. Only
   `tests/ace/tui/visual/snapshots/png/launch_context_bar_full_density_160x40.png` is
   expected to change (the other launch-context scenes render at compact density with
   labels hidden). View that PNG to confirm it shows dim `model: … · project: +sase`. If
   the known-flaky `selected_gate_shell_output` golden will not converge after a couple
   of solo retries, leave it and mention it in the summary.

## Commit

Conventional commit, e.g.
`feat(ace): lowercase launch-context labels to model:/project:`.
