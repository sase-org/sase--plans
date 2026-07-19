---
tier: tale
title: Toggle every Agents metadata fold between scale extremes
goal: 'The Agents-tab zZ fold chord opens every metadata fold to the active scale''s
  highest level unless that level is already active, in which case it closes every
  fold to the scale''s lowest level.

  '
create_time: 2026-07-19 10:29:44
status: done
prompt: 202607/prompts/toggle_all_metadata_folds.md
---

# Plan: Toggle every Agents metadata fold between scale extremes

## Context and outcome

The Agents-tab fold mode currently gives `zZ` the reverse-cycle behavior paired with `zz`. Replace that behavior with a
deterministic whole-panel extreme toggle while leaving `zz` as the forward level cycle. The command must use the fold
scale selected by the current context: a family has two effective levels, a clan or regular-agent session scope has
three, and a focused tribe panel has four. ChangeSpecs' separate `zZ` all-section toggle and the Agents `zA`
single-section toggle remain unchanged.

For the active scale, compare the panel's effective level with the scale's maximum. If the effective level is anywhere
below the maximum, set the panel to that maximum; if it is already at the maximum, set the panel to the scale's minimum.
This makes intermediate states open fully on the first `zZ`, then close fully on the next. Resolve out-of-scale shared
state through the existing effective-level rules before making the decision, so family, clan/session, and tribe behavior
remains well-defined as selection context changes.

## Fold behavior and keymap contract

Add a focused, scale-aware model operation for this maximum-sensitive extreme toggle rather than changing the existing
single-section toggle semantics. Dispatch the Agents whole-panel `zZ` subkey through that operation, assign the result
to the shared panel fold level, and clear all per-section/member overrides so the resulting view is uniformly at the
chosen extreme. Preserve the existing refresh and regular-agent scope notification behavior and keep the keypress path
purely in-memory.

Rename the nested Agents fold action from its reverse-cycle identity to a name that describes the new whole-panel
toggle, retaining `Z` as the default subkey in both the typed defaults and `default_config.yml`. Teach keymap loading to
honor an existing `cycle_level_back` user override as a deprecated alias when the replacement action is not explicitly
customized, so upgrading does not silently discard a personalized `zZ` binding. Ensure a new-name override wins when
both forms are present, and prevent the retired identifier from surfacing as a second command. Update registry tests to
lock down the migration and precedence behavior.

## User-facing descriptions and documentation

Replace reverse-cycle wording everywhere this Agents action is presented: transient fold-mode hints, the `?` help popup,
and the command palette. Keep the help description within the ACE help modal's width convention. Update the ACE and
agent family/clan/tribe documentation so every `zZ` reference describes the same rule: below the active maximum opens
everything; at the maximum closes everything. Continue to describe `zz` as the way to cycle through intermediate levels.

## Verification

Extend fold-scale unit coverage across every position of the two-, three-, and four-level scales, including shared
levels that must first be clamped to the active scale. Update fold dispatch tests to prove that `zZ`:

- opens family, clan/session, and tribe panels from every effective level below their maximum;
- closes each context from its maximum to its own minimum;
- clears section/member overrides on either transition;
- continues to refresh and notify through the established paths without mutating ChangeSpec fold state.

Adjust the mounted Agents interaction expectation so an intermediate clan level opens fully before the next `zZ` closes
it. Update default-keymap, help, footer, command-catalog, and custom-keymap tests for the renamed action and its new
descriptions. Run the focused model, panel-fold, keymap, help/catalog, and documentation-adjacent test modules first.
Then run `just install` followed by the mandatory `just check` for full repository validation.
