---
tier: tale
title: Make epic work auto-approval tier-neutral
goal: 'Epic phase and land agents launch with bare %auto so their authored plan tier
  determines the follow-up path, without coupling generated prompts to the tale compatibility
  argument.

  '
create_time: 2026-07-18 10:23:24
status: wip
prompt: 202607/prompts/epic_work_bare_auto.md
---

# Plan: Make epic work auto-approval tier-neutral

## Context and outcome

`sase bead work` renders one launch segment per open epic phase plus a final land-epic segment. Both roles may discover
follow-up work that is best expressed as either a focused tale or another epic, but the renderer currently injects
`%auto:tale` into every segment. That hard-codes the tale plan-gate path and also couples this internal automation to an
optional `%auto` argument whose vocabulary may be generalized.

Change the generated epic-work contract to use bare `%auto` for every phase and land segment. Bare auto-approval remains
autonomous, while the plan gate selects the correct action from the plan's authored `tier`; a tale can still be
approved, committed, and handed to its coder, while an epic can follow the epic approval and launch path. Existing
names, clan and tribe membership, model selection, VCS wrappers, wait dependencies, xprompt references, and retry
behavior must remain unchanged.

## Implementation

- Update `render_multi_prompt` in `src/sase/bead/work.py` so both the per-phase branch and the final land-agent branch
  emit exactly `%auto`, with no argument. Keep directive order stable to avoid changing any other launch semantics or
  dry-run output.
- Do not change parsing or support for `%auto:tale`, `%auto:epic`, or other adapter-owned arguments. Those remain valid
  for callers that intentionally constrain a plan workflow; this change only removes the constraint from generated epic
  phase and land prompts.
- Audit representative epic-work prompt fixtures as part of the same contract. In particular, update the fast
  planned-launch fixture in `tests/test_launch_planned_bead_work.py` to model the bare directive actually produced by
  `sase bead work`, without otherwise expanding the launcher scope.

## Regression coverage

- Update the exact multi-prompt expectations in `tests/test_bead/test_work_epic_plan.py` and
  `tests/test_bead/test_work_rendering.py` for bare `%auto` across independent, dependency-chained, ChangeSpec-wrapped,
  explicitly modeled, and default-model phase/land segments.
- Make the assertions semantic as well as textual: every rendered phase and land segment should contain a standalone
  `%auto` directive, no generated segment should contain `%auto:tale`, and directive extraction should retain
  auto-approval with no raw argument. This guards the intended tier-neutral behavior if `%auto` arguments change later.
- Update the `sase bead work --dry-run` integration expectation in `tests/test_bead/test_cli_work_epic_launch.py` so its
  rendered prompt count and model-adjacent snippets verify bare `%auto` on all phase and land agents.
- Keep generic `%auto:tale` parser, completion, gate, documentation, and historical changelog coverage intact because
  those describe the still-supported explicit mode rather than generated epic-work prompts.

## Documentation

- Revise `docs/beads.md` to describe bare `%auto` on all epic phase and land segments and explain that an agent may
  author a tale or epic as needed, with the authored tier selecting the automatic plan follow-up. Preserve the existing
  explanation that no human checkpoint is inserted between dependency waves.
- Bring the epic-work walkthrough in `docs/blog/posts/beads-and-sdd.md` in line with the same tier-neutral behavior.
  Avoid rewriting general `%auto:tale` documentation or release history that is not describing `sase bead work`.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run focused regression tests for the changed contracts:
   `pytest tests/test_bead/test_work_rendering.py tests/test_bead/test_work_epic_plan.py tests/test_bead/test_cli_work_epic_launch.py tests/test_launch_planned_bead_work.py`.
3. Audit remaining `%auto:tale` occurrences and confirm none belong to generated epic phase/land prompts, their
   representative fixtures, or their behavioral documentation; legitimate generic and historical uses should remain.
4. Run the mandatory repository-wide `just check` and resolve any formatting, lint, validation, or test failures before
   handing off the implementation.

## Risks and boundaries

Bare `%auto` applies the owning gate's default beyond the plan gate, unlike the plan-specific compatibility spelling.
That broader autonomy is intentional for these already-autonomous phase and land agents, but tests should stay focused
on the concrete requirement that submitted plans are not forced to tale. This plan does not alter plan schemas, bead DAG
construction, agent launch topology, model roles, or the content of the `work_phase_bead` and `land_epic` xprompts.
