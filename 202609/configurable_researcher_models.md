---
tier: tale
title: Configurable research-swarm researcher models
goal:
  Let each research role choose its launch model while preserving the swarm's current
  defaults and behavior.
size: small
proposed_by: bbugyi200.athena.0lk
create_time: 2026-09-15 16:40:18
status: wip
---

# Plan: Configurable Research-Swarm Researcher Models

## Scope and design

Implement this change in the linked `sase-research-artifacts` repository, opening that
repository through `/sase_repo` before reading or modifying it. No SASE core behavior
needs to change because Markdown xprompts already support typed, defaulted inputs and
Jinja interpolation inside model directives.

Add these optional `word` inputs to `#research_swarm`, appending them after the existing
`prompt`, `wait`, `priority`, and `runners` inputs so existing positional calls keep
their current meaning:

- `primary_model`, defaulting to `@sol_or_grok`, controls researcher A / the `.cdx`
  primary independent researcher.
- `second_opinion_model`, defaulting to `@opus_or_grok`, controls researcher B / the
  `.cld` independent second-opinion researcher.
- `lead_model`, defaulting to `@xlarge`, controls the `.final` lead researcher and
  consolidator.

These role-based names are stable and informative even if agent suffixes or provider
preferences change. Render each value through a `%m` directive in only its matching
researcher segment. Leave the fourth `.image` segment on its existing `@image` model
alias because it is the infographic agent, not one of the three researchers requested
here. With all three inputs omitted, the expanded swarm must be model-equivalent to the
current implementation.

## Implementation

1. Update `src/sase_research_artifacts/xprompts/research_swarm.md` to declare and
   describe the three model inputs with the exact defaults above, then replace the three
   hard-coded researcher model directives with `%m` directives interpolating the
   corresponding input. Do not alter swarm membership, dependency edges, report
   handling, queue controls, or the image agent's model selection.
2. Extend `tests/test_xprompt_loading.py` to assert the public typed-input order and
   exact defaults, prove an omitted-model expansion retains `@sol_or_grok`,
   `@opus_or_grok`, and `@xlarge` in the intended segments, and prove simultaneous
   custom values route independently to `.cdx`, `.cld`, and `.final` without changing
   `.image`. Retain the existing wait, priority, runners, dependency, and queue
   assertions so the model parameters are covered in composition with established
   behavior.
3. Synchronize the public contract in `docs/xprompts.md` and `README.md`: list the new
   named inputs, state their role mappings and defaults, and include a concise override
   example. Adjust `docs/configuration.md` and the repository `AGENTS.md` wording from
   unconditional model selection to default model selection so the alias/configuration
   guidance remains accurate. Do not change `default_config.yml`; its aliases remain the
   out-of-box defaults consumed by the xprompt.

## Validation

From the opened `sase-research-artifacts` repository:

1. Run the focused xprompt suite with `pytest tests/test_xprompt_loading.py` while
   iterating.
2. Run `just check` for the repository's full lint, type-check, and ordinary test gate.
3. Run `just test-wheel` to verify the modified packaged xprompt and its metadata work
   from a freshly built wheel and remain compatible with the published dependency
   floors.

The change is complete when default invocations reproduce all three prior researcher
model aliases, callers can override any or all researcher roles independently by name,
the image model and swarm graph are unchanged, documentation matches the effective
signature, and all validation commands pass.
