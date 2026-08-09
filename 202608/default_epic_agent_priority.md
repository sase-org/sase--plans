---
tier: tale
title: Use default runner priority for epic phase and lander agents
goal:
  Epic phase and lander launches no longer inject an explicit priority-15 wait while retaining their dependency and bead
  waits.
size: medium
proposed_by: bbugyi200.athena.r0
create_time: 2026-08-01 08:05:23
status: done
---

- **PROMPT:** [prompts/202608/default_epic_agent_priority.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/default_epic_agent_priority.md)

# Plan: Use Default Runner Priority for Epic Agents

## Current behavior and scope

`sase bead work` renders each epic phase and the final lander as separate prompt segments. Phase segments already rely
only on their authored agent/bead dependency waits: the bundled `bd/work_phase_bead` xprompt does not set a runner-slot
priority. The bundled `bd/land_epic` xprompt still contributes `%wait(priority=15)` when its segment is expanded, so the
lander is assigned an explicit, worse-than-default queue priority in addition to the genuine waits that the epic
renderer adds for every phase agent and phase bead.

Remove that lander-only priority override so both kinds of epic worker use the normal implicit priority (`10`) when they
become eligible. Do not remove or weaken the phase DAG waits, the lander's waits on all launched phases and phase beads,
runner-slot admission itself, or support for user/plugin xprompt overrides. Keep the separate `bd/work_task` priority
behavior unchanged: it launches standalone task beads and is outside this epic phase/lander policy change.

## Implementation

1. Update the bundled xprompts in `src/sase/default_config.yml` by removing `%wait(priority=15)` from `bd/land_epic`.
   Leave the phase and lander instructions intact, and leave `bd/work_task` unchanged.
2. Refine `tests/test_bead_xprompt_tags.py` so it no longer expects the epic lander to carry a priority-only wait. Add
   explicit regression coverage that both `bd/work_phase_bead` and `bd/land_epic` have no authored wait priority, while
   retaining focused coverage for the standalone task xprompt's existing priority behavior. Assert through directive
   extraction so a future hidden directive in either epic template is caught without conflating it with the renderer's
   legitimate inter-agent and bead dependencies.
3. Update the built-in xprompt catalog in `docs/xprompt.md` and the runner-slot troubleshooting guidance in
   `docs/troubleshooting/runner-slots.md` to stop claiming that bundled epic landers are deprioritized. Document that
   bundled phase and lander prompts use the default queue priority while still waiting for their declared dependencies,
   and preserve the documentation of the independent task-worker priority.

## Validation

1. Run the focused bead-xprompt tests, including the new no-explicit-priority regression assertions.
2. Expand the bundled phase and lander xprompts (or exercise the equivalent expansion path in tests) to confirm neither
   contributes `%wait(priority=15)` and that the prompt instructions remain intact.
3. Run `just install`, then the repository-required `just check`, and resolve any failures caused by the change.
