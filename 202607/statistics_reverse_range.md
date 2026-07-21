---
tier: tale
title: Reverse Statistics time-range keymap
goal: 'The SASE Admin Center Statistics tab supports a configurable uppercase T shortcut
  that cycles preset time ranges in the exact opposite direction from the existing
  lowercase t shortcut, with consistent help, schema, and tests.

  '
create_time: 2026-07-21 07:44:12
status: done
prompt: 202607/prompts/statistics_reverse_range.md
---

# Plan: Reverse Statistics time-range keymap

## Context and intended behavior

The Statistics pane owns a focused keymap scope so its single-letter controls do not become active on other Admin Center
tabs. Its current `cycle_range` action defaults to lowercase `t` and walks `PRESET_ORDER` forward through Today, Last 24
hours, Last 7 days, Last 30 days, Last 90 days, and All time, wrapping at the end. Add a sibling reverse action whose
default is uppercase `T` and whose behavior is otherwise identical: it selects and resolves a preset, clears any
custom-range value, preserves the current view/group/project filter, and schedules the same debounced statistics reload.

For a normal preset, reverse cycling moves one index backward with wraparound, so `Today` goes to `All time` and
`All time` goes to `Last 90 days`. A custom range has no index in `PRESET_ORDER`; preserve the forward action's boundary
convention symmetrically by making lowercase `t` re-enter at `Today` and uppercase `T` re-enter at `All time`.

This is presentation-only Textual behavior. It does not change statistics range definitions, Rust statistics APIs, or
query semantics.

## Keymap and configuration contract

- Extend `StatisticsPaneKeymaps` and `_STATISTICS_BINDING_META` with a `cycle_range_reverse` action adjacent to
  `cycle_range`, using a user-facing description such as “Previous Time Range.” Give it the bundled default `T` under
  `ace.keymaps.statistics` in `src/sase/default_config.yml`; keep the existing `t` default unchanged.
- Add the new property to the public configuration schema because the scoped Statistics object rejects unknown
  properties. Ensure default loading, partial overrides, canonicalization, validation, and duplicate-key fallback all
  continue to work. In particular, lowercase `t` and uppercase `T` must remain distinct valid defaults, while a real
  collision introduced by an override must revert and warn through the existing scoped-keymap rules.
- Register the action through the existing instance-local binding builder so uppercase `T` dispatches only while the
  Statistics pane is focused. Do not promote either range action to the global app keymap.

## Statistics pane behavior

Refactor the forward range mutation into one direction-aware internal path, then have `action_cycle_range` call it in
the existing direction and `action_cycle_range_reverse` call it in the opposite direction. Centralizing preset
selection, custom-range clearing, `resolve_preset`, and `_selection_changed(reload=True)` prevents the two shortcuts
from drifting and guarantees that all selection/reload side effects remain identical.

Keep the existing forward order intact. Exercise the reverse action from the default `7d` selection, both wraparound
edges, and a custom selection. Verify that rapid forward/reverse changes still coalesce to the latest selection and that
the active project filter survives the reload path.

## Discoverability and presentation

Expose the effective reverse key alongside the forward key in Statistics-specific help and range-control presentation,
including customized keymaps. Update the compact Admin Center guidance shown from the main ACE help surfaces so its
default Statistics shortcut summary includes uppercase `T`.

Adding a metadata entry changes the binding list's order. Replace or isolate fixed numeric indexing in Statistics
rendering with action-based effective-key lookup before updating range, custom-range, grouping, project-filter, refresh,
and help hints. This prevents the new adjacent range action from silently assigning the wrong key to existing labels or
recovery guidance. Preserve the meaning of empty/error recovery copy: widening-range guidance should still point at the
forward control, while the contextual control list documents both directions.

Review the overview, narrow-pane, empty-state, and Statistics-help presentations after the metadata change. Update only
the affected committed PNG goldens when the additional reverse key/control is an intentional visible difference.

## Regression coverage and verification

- Expand keymap default, registry-loading, validation, and public-schema tests to cover the new field, default `T`, a
  custom override, and a duplicate with another Statistics control.
- Extend Statistics binding tests to prove the configured reverse key dispatches while the pane is focused, is inactive
  on sibling Admin Center tabs, and appears correctly in effective help/range hints without shifting existing controls.
- Add interaction coverage for backward movement, wraparound, custom-to-`All time` re-entry, state clearing, reload
  behavior, and continued forward behavior.
- Extend contextual-help completeness/presentation assertions and inspect the Statistics visual tests. If rendering
  changes, regenerate the relevant canonical snapshots and rerun the dedicated visual suite.
- Run `just install` before repository checks as required for an ephemeral workspace. Run the focused keymap,
  configuration-schema, Statistics interaction/help/legend, and Statistics visual tests during iteration, then finish
  with `just check` to cover formatting, lint/type checks, the full test suite, and PNG snapshot consistency.

## Acceptance criteria

With default configuration, pressing `T` on the focused Statistics tab moves from Last 7 days to Last 24 hours; repeated
presses traverse every preset backward and wrap. Pressing `t` continues to traverse the same presets forward. Both
directions use the same selection and reload lifecycle, custom ranges re-enter at the appropriate end of the preset
order, overrides are schema-valid and honored, other tabs do not receive the scoped shortcut, and every displayed key or
help description matches the effective configuration.
