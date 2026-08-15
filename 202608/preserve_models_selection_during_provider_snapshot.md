---
tier: tale
title: Preserve Models selection across provider snapshot completion
goal:
  Provider snapshot completion rebuilds the redesigned Models panel without moving the
  user's current selection or redirecting the action they are taking.
size: small
proposed_by: bbugyi200.athena.sase-mc.5.land
bead: sase-mc.5
create_time: 2026-08-15 18:31:53
status: done
---

- **PROMPT:**
  [prompts/202608/preserve_models_selection_during_provider_snapshot.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/preserve_models_selection_during_provider_snapshot.md)
- **PARENT:** [202608/provider_disable_completion.md](provider_disable_completion.md)
- **BEAD:**
  [sase-mc.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mc/sase-mc.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-mc.5.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-mc.5.land.md)
- **COMMITS:**
  - [5511f04](https://github.com/sase-org/sase/commit/5511f04ed37e0545984957e17e52247cc3fa3256)
    — fix(tui): keep Models selection across provider snapshots

# Preserve Models selection across provider snapshot completion

## Problem

The provider-disable completion work correctly moved authoritative provider-state and
alias-view loading to a Textual thread worker. A later Models-panel redesign added
launch-setting rows ahead of the alias sections. The initial provider snapshot can now
finish after the user has highlighted a row: its row rebuild carries no explicit `keep`
target, so `_restore_highlight()` selects the new first actionable launch row. An action
typed against a bucket can consequently run against a different row.

This race explains the full-suite-only failure reported by proposing phase `sase-mc.5.2`
in `tests/test_models_panel_navigation.py::test_alias_actions_on_bucket_are_guarded[e]`:
the node passed immediately in a focused rerun, while the loaded suite provides enough
scheduling delay for the initial provider worker to complete between selection and the
`e` action. This is an integration defect in `sase-mc.5`, not unrelated flake debt.

## Implementation

1. Preserve the currently highlighted Models row when a provider-snapshot worker
   completes without an explicit `keep` target. Keep explicit targets authoritative for
   write-driven refreshes, and retain the existing fallback when the prior row no longer
   exists. Do not move provider-state reads or alias rebuilding back onto the Textual
   event loop.
2. Add a deterministic regression that delays the initial provider snapshot until after
   a bucket row is selected, completes the snapshot before the edit action is handled,
   and proves the bucket remains selected and the normal guarded-action notification
   fires. Cover the row-reordering shape introduced by the launch-setting redesign
   rather than relying on timing or repeated runs.
3. Re-run the focused Models-panel/provider-disable suite, including the exact
   `test_alias_actions_on_bucket_are_guarded` parameter matrix. Run `just check` for the
   repository-required lane. The unrelated alias-indicator PNG mismatch is already
   routed to active epic `sase-mf` and is outside this plan.

## Acceptance criteria

- Initial or expiry-driven provider snapshot completion does not move a still-present
  Models-panel selection.
- Explicit refresh targets still win, and a removed row still falls back safely.
- The regression deterministically exercises delayed provider completion and passes
  without sleeps or contention.
- Focused Models/provider verification and `just check` pass.
