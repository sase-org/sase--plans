---
tier: tale
title: Integrate gateless plan approval with agent-session syntax
goal: Direct approvals launch coders using canonical syntax and report every outcome
  accurately.
size: medium
proposed_by: bbugyi200.athena.sase-18i.land
bead: sase-18i
status: done
---

- **PARENT:**
  [202609/plan_approve_gateless_tales.md](https://github.com/sase-org/sase--plans/blob/main/202609/plan_approve_gateless_tales.md)
- **BEAD:**
  [sase-18i](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18i/README.md)

# Integrate gateless plan approval with agent-session syntax

The `sase-18i` implementation composes a family coder prompt with
`%id(code, family=<planner>)`. Commit `65d3dfb1d` changed authored attach syntax to
`%id(code, session=<planner>)`; the legacy spelling survives only behind
`legacy_agent_family_syntax` (`sase-18l`). The direct approval route must emit the
canonical spelling so family launches continue to work after that flag is disabled.

## Work

1. Change `compose_coder_prompt` in `src/sase/main/plan_direct_approval.py` to emit
   `session=` for an attached coder. Update the assertions in
   `tests/test_plan_direct_approval.py` and add a test that parses the generated
   directive with the legacy flag disabled, proving that the prompt can actually launch.
2. Update the documentation introduced by `sase-18i` in `docs/xprompt.md` to say agent
   session where it describes the direct approval coder placement. Keep the user-facing
   approval command behavior the same.
3. Complete the CLI phase's missing renderer coverage. The epic plan calls for
   `tests/test_plan_approve_render.py` to cover gate success and dry run, direct family
   and standalone outcomes, refusal, and partial launch failure cards under `NO_COLOR`.
   Test the exit codes at the handler seam where rendering itself does not exit.
4. Check the concurrent `already_answered` gate path in
   `src/sase/main/plan_direct_approval_run.py`: the epic plan requires no second coder
   and a warning. Its current implementation writes a receipt and archives before
   raising a refusal. Make the recorded result and CLI recovery instructions accurately
   reflect the committed state, and test the race.

## Verification

Run the focused plan approval tests and `sase tool run check` (`just check`). Do not run
`just check-full`.

The parent epic's land agent owns follow-up triage, the remaining Symvision exemptions,
epic closure, and plan status updates after this tale lands.
