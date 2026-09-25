---
tier: epic
title: Repair the retry agent-session visual case
goal: The retry countdown visual test and golden use the renamed agent session and
  pass verification.
parent_bead: sase-17m.5.1.6
phases:
- id: retry-visual
  title: Repair the retry countdown visual test and golden
  size: small
  description: 'retry-visual: update the stale retry query, inspect the targeted golden
    update, and verify the test and just check.'
  depends_on: []
proposed_by: bbugyi200.athena.sase-17m.5.1.6.land
create_time: 2026-09-25 10:02:00
status: wip
bead_id: sase-17m.5.1.6.5
---

- **PROMPT:** [prompts/202609/retry_session_visual_gap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/retry_session_visual_gap.md)
- **PARENT:** [202609/agent_session_ace_cutover_finish.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_ace_cutover_finish.md)
- **BEAD:** [sase-17m.5.1.6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.5.1.6.5.md)

# Repair the retry agent-session visual case

## Context

The `sase-17m.5.1.6.3` identifier rename changed
`tests/ace/tui/_retry_agent_session_loader_fixture.py` from `retry-family` to
`retry-session`, but `test_real_loader_plan_agent_session_retry_countdown_png_snapshot`
in `tests/ace/tui/visual/test_ace_png_snapshots_agents_retry_e2e.py` still filters for
`retry-family`. Its committed golden
`agents_retry_e2e_plan_session_countdown_120x40.png` still shows the old name.
`sase-17m.5.1.6` note #2 records this mismatch. It is epic-caused work.

The separate `sase-173` task concerns unseeded random hex IDs in other retry E2E
goldens. Do not accept any unrelated random-ID pixel drift. The retry case and its
golden should agree on `retry-session` without changing the behavior under test.

## Phase: retry-visual

1. Change the stale query in the retry countdown visual test to `retry-session`. Check
   the rest of that test for any other stale family-concept spelling and keep
   intentional opaque data in unrelated visual tests intact.
2. Install the workspace dependencies if needed, then run the targeted visual capture
   with `just fix-tui-screenshots --` using the retry countdown test node. Inspect the
   visual report and the golden diff. Accept only pixels explained by the renamed
   fixture and query. Verify the same node in check mode.
3. Run `just fix` and `sase tool run check`; do not run `just check-full`.

## Completion

The retry countdown visual test selects the renamed agent session, its golden shows that
session name, no unrelated retry goldens change, and `just check` passes.
