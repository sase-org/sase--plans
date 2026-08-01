---
tier: tale
title: Use default runner priority for task-bead workers
goal: Bundled task-bead workers use the same implicit runner priority as other bundled bead-worker agents.
proposed_by: bbugyi200.athena.r0.f0
create_time: 2026-08-01 08:26:54
status: done
---

- **PROMPT:** [202608/prompts/task_bead_default_priority.md](prompts/task_bead_default_priority.md)

# Plan: Use Default Runner Priority for Task-Bead Workers

## Current behavior and scope

The bundled `bd/work_task` xprompt is the only bundled bead-worker prompt that still contains `%wait(priority=15)`.
`sase bead work <task-id>` renders a single worker segment containing the task xprompt reference; the launch renderer
does not add a priority itself, so expanding the bundled xprompt assigns task-bead workers the explicit lower queue
priority `15` instead of the normal implicit priority `10`.

Remove that bundled override so task-bead workers join epic phase and lander agents in using the default runner
priority. Do not change runner-slot admission or `%wait(priority=...)` support generally, task reservation/checkpoint
behavior, agent naming or model selection, VCS routing, task instructions, or xprompt discovery. Project, user, config,
and plugin overrides remain free to author their own wait or priority directives.

## Implementation

1. In `src/sase/default_config.yml`, remove `%wait(priority=15)` from the bundled `bd/work_task` content while
   preserving its `work_task_bead` tag, input, and worker instructions.
2. In `tests/test_bead_xprompt_tags.py`, replace the task-specific priority-15 expectation with regression coverage that
   passes `bd/work_task` through the same directive extraction used for the built-in epic phase and lander prompts.
   Assert that all three bundled bead-worker bodies author no priority, agent, bead, duration, time, or runner wait and
   that directive removal leaves each prompt's identifying worker instruction intact. Keep tag resolution and custom
   xprompt override coverage unchanged.
3. Update `docs/xprompt.md` and `docs/troubleshooting/runner-slots.md` to state that bundled task, phase, and lander
   workers use the default priority (`10`) when otherwise eligible. Retain the warning that higher-precedence xprompt
   overrides supply their own bodies and can choose another priority.
4. Verify that the product-specific priority-15 references are gone while generic priority examples and parser tests
   remain. Run `just install`, the focused `tests/test_bead_xprompt_tags.py` test file, and the repository-required
   `just check`.

## Acceptance criteria

- Expanding the bundled `bd/work_task` xprompt produces no authored `%wait` metadata and therefore leaves task workers
  at the runner's implicit priority `10`.
- The bundled task, epic phase, and epic lander prompts all have focused regression coverage for the absence of authored
  waits without weakening genuine waits rendered elsewhere for epic dependencies.
- Documentation consistently distinguishes default bundled behavior from project, user, config, or plugin overrides.
