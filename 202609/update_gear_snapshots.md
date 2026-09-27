---
tier: epic
title: Finish the yellow and red update gear visual coverage
goal: The restart-queued and failed gears have deterministic ACE PNG snapshot tests
  and inspected goldens, completing the remaining visual scope of sase-1bd.
parent_bead: sase-1bd
phases:
- id: gear-goldens
  title: Capture and inspect the yellow and red update gear goldens
  size: small
  depends_on: []
  description: 'gear-goldens: add deterministic visual tests for restart-pending and
    failed update indicators, capture only their visual test module, inspect and retain
    the two new PNG goldens, and run just check.'
proposed_by: bbugyi200.apollo.sase-1bd.land
create_time: 2026-09-27 16:50:50
status: done
bead_id: sase-1bd.5
---

- **PROMPT:** [prompts/202609/update_gear_snapshots.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/update_gear_snapshots.md)
- **PARENT:** [202609/update_gear_states.md](https://github.com/sase-org/sase--plans/blob/main/202609/update_gear_states.md)
- **BEAD:** [sase-1bd.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bd/sase-1bd.5.md)

# Finish the yellow and red update gear visual coverage

## Context

Epic `sase-1bd` added the three-state update gear in commits `1d60ffcf4`, `9814d8980`,
`3786032ef`, and `21d4e12c8`. Its four phases are closed and the production gear,
journal, failure report, Update panel row, and documentation are present. Phase
`sase-1bd.1` note #1 and phase `sase-1bd.3` note #2 each proposed a missing visual
golden because Rust compilation and build-lock contention prevented capture. The
original epic plan allowed those workers to omit the visual test temporarily and record
a follow-up. This child plan completes only that omitted work. The parent epic's land
agent must review the result and close `sase-1bd`; its close and plan-status update are
not child-phase work.

The relevant test module is
`tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`. It already captures
the green updating gear using `AcePage`, `patch_startup_loaders`, `wait_for_state`,
`wait_for_visual_idle`, and `ace_png_visual.assert_page_png`. The new states are set
through `UpdatesAvailableIndicator.set_restart_pending(PendingUpdateRestart(...))` and
`set_last_failure(UpdateFailure(...))`. Goldens belong under
`tests/ace/tui/visual/snapshots/png/`.

## Phase gear-goldens

1. Add a yellow restart-pending test and a red failed test to the existing
   update-indicator visual module. Follow its setup and settle pattern. Use fixed
   timestamps and deterministic labels, and make the expected rendered text assert the
   same three-cell `⚙` inset. Include a red case with zero available-update counts so
   the gear alone keeps the badge visible. Keep all fixture data local and avoid a live
   journal or background update.
2. Run only the targeted capture:
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`.
   Install workspace dependencies first if needed. Read the maintenance report,
   including any `partial` warning and skipped cases. Inspect both new PNGs visually and
   the changed golden list; accept only expected indicator changes. Resolve capture
   blockers instead of dropping these two tests again. Re-run the module in strict
   visual check mode.
3. Run `just fix` and `sase tool run check` (the `just check` gate). Do not run
   `just check-full`. If unrelated master-red checks remain, record the exact evidence
   and existing task bead in the phase note; do not treat that as a reason to omit the
   goldens.

## Completion

The update-indicator visual module contains both tests; its yellow and red PNG goldens
have been inspected and pass the targeted check; no unrelated goldens changed.
`just check` is run and its result is recorded. The child land agent then returns
control to the waiting `sase-1bd` land agent through `parent_bead`.
