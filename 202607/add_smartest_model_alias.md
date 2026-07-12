---
create_time: 2026-07-11 09:48:59
status: wip
prompt: 202607/prompts/add_smartest_model_alias.md
tier: tale
---
# Add the `smartest` Model Alias

## Context

The chezmoi-managed SASE configuration currently maps `epic_lander` and `research_lead` directly to concrete models.
Introduce one reusable custom alias, `smartest`, backed by `codex/gpt-5.6-sol`, and route both roles through that alias
so future changes to the highest-capability model can be made in one place.

The resulting resolution should be:

| Alias           | Configured target   | Effective target    |
| --------------- | ------------------- | ------------------- |
| `smartest`      | `codex/gpt-5.6-sol` | `codex/gpt-5.6-sol` |
| `epic_lander`   | `@smartest`         | `codex/gpt-5.6-sol` |
| `research_lead` | `@smartest`         | `codex/gpt-5.6-sol` |

`smartest` will be a general-purpose, unbucketed custom alias. `research_lead` will retain its existing description and
membership in the `research` bucket; only its model target changes. Other aliases that currently point directly to
`codex/gpt-5.6-sol` are intentionally outside this change.

## Implementation

1. Update `home/dot_config/sase/sase.yml` in the chezmoi repository:
   - Add `smartest` under `llm_provider.model_aliases.custom` with model `codex/gpt-5.6-sol` and a concise description
     identifying it as the shared highest-capability model alias.
   - Change the builtin `epic_lander` override from its current concrete model to `"@smartest"`.
   - Change `research_lead.model` from its current concrete model to `"@smartest"`, preserving its description and
     `bucket: research` metadata.
   - Preserve the existing alias layout and leave `research_a`, `research_b`, the research bucket description, and all
     unrelated builtin/custom aliases unchanged.

2. Apply only the managed SASE configuration target through chezmoi so `~/.config/sase/sase.yml` reflects the source
   change, then confirm a targeted chezmoi diff is empty.

## Validation

1. Inspect the source and applied configuration to confirm there is exactly one concrete `smartest -> codex/gpt-5.6-sol`
   definition and that both consumers reference `@smartest`.
2. Run `sase doctor -C config.model_aliases` to verify the new custom alias has all required metadata and that neither
   consumer introduces a dangling reference, invalid placement, collision, or alias cycle.
3. Resolve `smartest`, `epic_lander`, and `research_lead` through SASE's model-alias resolver and verify all three yield
   `codex/gpt-5.6-sol`; also confirm `research_lead` remains in the three-member `research` bucket and `smartest` is not
   added to that bucket.
4. Run the chezmoi repository's relevant YAML/keep-sorted checks plus `git diff --check`, and review the final diff to
   ensure only `home/dot_config/sase/sase.yml` changed.

## Non-goals

- Do not edit the `research_swarm` xprompt; it already launches the lead through `@research_lead` and will inherit the
  new target through alias resolution.
- Do not reroute `epic_creator`, `phase_worker`, `research_a`, or any other direct GPT-5.6 alias through `@smartest`.
- Do not commit or push as part of implementation unless separately requested or triggered by the normal finalizer
  workflow.
