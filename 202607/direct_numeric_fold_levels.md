---
tier: tale
title: Direct numeric fold-level keymaps
goal: 'ACE users can press the configured fold prefix followed by a valid one-based
  level number to switch directly to that fold level in every real fold-mode context,
  with context-accurate discovery, configuration, and tests.

  '
create_time: 2026-07-19 07:31:24
status: wip
prompt: 202607/prompts/direct_numeric_fold_levels.md
---

# Plan: Direct numeric fold-level keymaps

## Context and scope

ACE currently uses a configurable fold prefix (default `z`) with cycling and toggle subkeys. Two UI contexts actually
render state controlled by that mode:

- The PR detail view on the ChangeSpecs/Artifacts PRs pane has a shared legacy three-level ladder for COMMITS, HOOKS,
  MENTORS, TIMESTAMPS, and DELTAS.
- The Agents metadata document uses an existing kind-specific scale: family summaries have two effective levels, clan
  summaries (and the session state selected from a regular agent) have three, and a selected whole tribe panel has four.

Add one-based direct-selection chords without removing or changing the existing cycle/toggle chords. With the default
prefix, the behavior is:

| Context                                            | Valid chords           | Exact target                                                                                 |
| -------------------------------------------------- | ---------------------- | -------------------------------------------------------------------------------------------- |
| ChangeSpec PR detail                               | `z1`, `z2`, `z3`       | Set every fold-mode section to collapsed, expanded, or fully expanded                        |
| Agents family summary                              | `z1`, `z2`             | Set the panel to the first or second level of `FAMILY_FOLD_SCALE`                            |
| Agents clan summary or regular-agent session scope | `z1`, `z2`, `z3`       | Set the panel to the corresponding level of `CLAN_FOLD_SCALE`                                |
| Selected whole tribe panel                         | `z1`, `z2`, `z3`, `z4` | Set the panel to the corresponding level of `TRIBE_FOLD_SCALE`, including exhaustive level 4 |

The number denotes the displayed position within the active scale, not the ordinal of the global enum. This matters for
family level 1, whose existing effective value is `EXPANDED`. A number outside the active scale is not a valid command:
consume it like another invalid fold subkey, leave fold state and section overrides untouched, and restore the normal
footer. A direct panel-level selection clears per-section overrides exactly as panel-level cycling does. On a regular
agent it retains the current behavior of changing the session scope for the next summary selection and explaining that
scope.

Tree/group folding driven by `h`/`l`, member-number jumps outside fold mode, and zoom on uppercase `Z` are separate
behaviors and remain unchanged. The Axe tab and non-PR Artifact panes have no content controlled by fold mode; do not
advertise or execute the new numeric commands there, and tighten the generic fold-mode gating if needed so the existing
no-op path is not mistaken for a supported context.

## Fold model and dispatch

- Add a small shared fold-scale operation that resolves a one-based position to an exact `FoldLevel`, rejecting zero and
  out-of-range positions instead of clamping. Reuse the existing ordered family, clan, and tribe scales so the direct
  path cannot drift from cycling, headers, or renderers.
- Centralize or reuse the Agents selection-to-scale decision (whole panel, family container, clan container, or regular
  agent) anywhere live dispatch, footer rendering, and command availability need the same answer. Keep this
  presentation/navigation logic in the Python TUI boundary; no Rust core API change is warranted.
- Extend fold-mode dispatch with configured direct-level subkeys. On Agents, resolve the current scale, set the exact
  panel level, clear overrides only after a valid selection, refresh through the existing current-tab path, and retain
  the regular-agent scope notification. On ChangeSpecs PR detail, set all five shared section states to the exact
  selected level and update the existing fold indicator. Preserve DELTAS' renderer-specific interpretation of those
  shared states and all existing cycle/toggle behavior.
- Keep the keystroke path synchronous, cache-only, and free of disk or process work. Reuse the current
  selective/background artifact-loading behavior after a level change rather than adding a new refresh or I/O path.

## Keymap configuration and discoverability

- Add stable direct-level action names and default digit subkeys to both `FoldModeKeymaps` and
  `src/sase/default_config.yml`: levels 1-3 in the ChangeSpec fold map and levels 1-4 in the nested Agents map. Preserve
  deep merging and user overrides so a customized fold prefix or numeric subkey is reflected everywhere instead of
  hard-coding literal `z` chords.
- Register each direct command in the command catalog with a clear “set fold level N” label and the correct tab scope.
  Extend context-aware palette availability so Agents only offers positions supported by the current
  family/clan/regular/tribe scale; ChangeSpecs offers its three levels only on the PR pane.
- Extend the fold-mode footer with direct-level hints derived from the active scale. Ensure all Agents and ChangeSpecs
  refresh callers pass the correct tab and Agents context so an interleaved background redraw cannot replace the
  context-specific fold footer with another tab's bindings.
- Update the Agents and ChangeSpecs help sections to show the configured direct-level sequences and explain their
  ranges. Update the configuration example/reference plus the user documentation for ChangeSpec, family, clan, and tribe
  folding. Reconcile any stale level counts in those docs with the existing kind-specific scales; historical blog posts
  are out of scope.

## Verification

- Add pure fold-scale tests for every valid one-based position and for rejected zero/out-of-range positions, including
  the family scale's non-collapsed first enum value.
- Extend fold-dispatch tests to cover exact jumps in all Agents contexts, invalid-context no-ops, override clearing only
  on success, regular-agent notification behavior, ChangeSpec all-section updates, DELTAS compatibility, and isolation
  between Agents and ChangeSpec state. Retain regression coverage for the existing forward/backward cycles and section
  toggles.
- Extend keymap tests for dataclass/default-config parity, partial nested overrides, customized prefixes/subkeys, and
  complete command-catalog coverage. Test contextual command-palette availability for family (1-2), clan/regular (1-3),
  tribe panels (1-4), PR details (1-3), and unsupported tabs/panes.
- Test footer and help output with configured keymaps and each valid Agents range. Add a mounted ACE interaction that
  jumps directly among tribe levels, including `z4`, and confirms numeric fold chords do not become member jumps. Update
  the Help PNG goldens only for intentional content/layout changes and inspect the generated diffs.
- Run `just install` before repository checks as required for an ephemeral workspace. Run focused
  fold/keymap/catalog/help tests during development, `just test-visual` for intentional snapshot changes, and finish
  with the mandatory `just check`.

## Risks and compatibility

- Direct-level action IDs and configured subkeys must remain distinct between the ChangeSpec and nested Agents scopes
  even though their default digits overlap. Catalog, help, footer, and dispatch should all read the registry as the
  single source of configured sequences.
- Entering fold mode must continue to intercept the second digit before Agents member-jump handling. Invalid numeric
  levels must not leak through and jump to a roster member.
- Do not persist new state or change fold-scale definitions in this work. The feature selects existing states directly;
  it should preserve current session-only panel behavior, per-section inheritance, and lazy disk-backed rendering.
