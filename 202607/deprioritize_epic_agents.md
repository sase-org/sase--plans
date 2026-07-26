---
tier: tale
title: Deprioritize epic-associated built-in xprompts
goal: Agents launched through bd/land_epic and bd/next yield runner-slot admission
  to agents using the default priority.
create_time: 2026-07-26 07:55:35
status: wip
---

- **PROMPT:** [202607/prompts/deprioritize_epic_agents.md](prompts/deprioritize_epic_agents.md)

# Plan: Deprioritize Epic-Associated Built-in XPrompts

## Goal

Make agents launched through `#bd/land_epic` and `#bd/next` yield runner-slot admission to newly launched agents that
use the implicit priority of `10`, while preserving the existing bead-selection, implementation, integration, and
landing instructions.

## Context

Both built-ins are config-backed xprompts defined in `src/sase/default_config.yml`. Xprompt bodies are expanded before
launch directives are extracted, so a `%wait(priority=15)` directive authored at the start of each body becomes launch
metadata and is stripped before the prompt reaches the model. Runner-slot admission gives lower numeric values
precedence; therefore explicit priority `15` is below the default priority `10` and receives the existing bounded
deference behavior. A priority-only wait does not add an agent dependency, time floor, bead dependency, or explicit
runner threshold.

## Implementation

1. In `src/sase/default_config.yml`, prepend `%wait(priority=15)` to the `content` block for both `bd/land_epic` and
   `bd/next`.
   - Keep the directive inside each built-in xprompt so every launch path that expands the prompt receives the same
     priority.
   - Do not add `runners=`, dependency names, or other wait conditions.
   - Leave the remaining xprompt wording and inputs unchanged.

2. Extend the built-in bead-xprompt coverage in `tests/test_bead_xprompt_tags.py`.
   - Load both `bd/land_epic` and `bd/next` from the effective built-in prompt catalog.
   - For each prompt body, run the normal directive extractor and assert that `wait_priority` is exactly `15`.
   - Assert that no agent dependencies, bead dependencies, time wait, absolute wait, or explicit runner threshold were
     introduced, and that directive stripping leaves the existing task instructions intact.
   - This regression test should fail if either built-in loses the priority directive or if it is accidentally changed
     into a stronger wait condition.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused regression tests for `tests/test_bead_xprompt_tags.py`.
3. Run `just check` to cover formatting, linting, SASE validation, committed-plan validation, and the full test suite.

## Non-goals

- Do not change the global default wait priority, runner-slot ordering, or bounded-deference implementation.
- Do not change bead claiming, epic orchestration, or the prompts' user-facing task instructions.
- Do not duplicate the already documented `%wait(priority=N)` and default-priority semantics.
