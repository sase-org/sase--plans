---
tier: tale
title: Default big epic landers to the smartest model
goal: 'Threshold-selected epic land agents inherit SASE''s provider-aware @smartest
  model policy by default, and the matching personal chezmoi override is removed without
  changing explicit land-model or alias-override behavior.

  '
create_time: 2026-07-21 11:35:08
status: done
prompt: '[202607/prompts/big_epic_lander_smartest_default.md](prompts/big_epic_lander_smartest_default.md)'
---

# Plan: Default big epic landers to the smartest model

## Context and outcome

SASE already selects `@big_epic_lander` for an epic whose authored phase count meets `bead.big_epic_phase_threshold`,
but the alias currently falls through `@epic_lander` and then `@default`. Bryan's chezmoi-managed `sase.yml` compensates
for that policy with an explicit `big_epic_lander: "@smartest"` override.

Make `@smartest` the built-in fallback of `@big_epic_lander` itself, then remove the personal override so the shared
SASE policy is the single source of truth. Afterward, threshold-sized epics use the existing provider-aware `@smartest`
ordered fallback by default, while smaller epics continue to use `@epic_lander` → `@default`. A configured `epic_lander`
override will therefore affect only below-threshold epic landers; users can still configure `big_epic_lander` directly
when they want different large-epic behavior.

This is a focused cross-repository change that one implementation agent can complete and verify, so it is a tale rather
than a multi-phase epic.

## Built-in alias policy

Change the centralized implicit-role fallback for `big_epic_lander` from `@epic_lander` to `@smartest`. Reuse the
current `@smartest` definition and its ordered provider-availability selection; do not introduce a new selector, cursor,
or launch path.

Preserve all existing launch semantics:

- The authored phase-count threshold still decides between `@epic_lander` and `@big_epic_lander`, including on resumed
  work.
- A plan's explicit top-level land model still wins at every epic size.
- Persistent `big_epic_lander` configuration, launch-scoped alias overrides, and temporary Models-panel overrides still
  take precedence over the implicit fallback.
- Below-threshold `@epic_lander` behavior and the concrete/provider-availability behavior of `@smartest` remain
  unchanged.

Update unit coverage around the centralized policy and resolution chain. Replace the old expectation that a configured
`epic_lander` target is inherited by a big epic with coverage that the big role resolves through `@smartest`
independently, including provider availability, while direct `big_epic_lander` overrides still win. Adjust alias
dependency/cycle checks for the new graph.

## User-facing alias surfaces

Update configuration examples, the default configuration commentary, and the LLM, bead, SDD, and delegated-work
documentation wherever they describe the old `@big_epic_lander` → `@epic_lander` → `@default` chain. Explain the new
split clearly: normal epic landers follow `@epic_lander`, threshold-selected landers follow `@big_epic_lander` →
`@smartest`, and an explicit land model or direct alias override remains authoritative.

Keep the alias name, completion entry, threshold setting, and role description. Update Models-panel/alias-view
expectations so an unconfigured `big_epic_lander` row reports `implicit → @smartest` and displays the effective smartest
target. Refresh only visual snapshots whose rendered alias state or resolved model actually changes, inspecting the
resulting images before accepting them.

## Personal chezmoi configuration

In the audited linked chezmoi checkout, remove only the explicit
`llm_provider.model_aliases.builtin.big_epic_lander: "@smartest"` entry from the canonical
`home/dot_config/sase/sase.yml`. Do not broaden this cleanup to unrelated legacy role overrides in that file.

Deploy the updated chezmoi source only after the updated SASE builtin policy is installed or otherwise available, so
removing the personal override cannot temporarily route big epic landers back through `@epic_lander`. Follow the linked
repository's required chezmoi update/apply workflow and verify both the managed source and live SASE configuration no
longer contain the override.

## Verification

Run focused alias-policy, resolution, Models-panel, completion/picker, and epic rendering/preview tests. Confirm
below-threshold and threshold-selected launch directives are unchanged (`@epic_lander` versus `@big_epic_lander`) and
that only the latter now resolves through `@smartest` by default. Run the exact Models-panel visual suite after any
intentional golden refresh.

Because the main SASE checkout changes, first install the workspace dependencies as required for an ephemeral checkout,
then run the repository-wide `just check` gate. Audit both repositories for unrelated changes, verify the chezmoi source
and deployed configuration agree, and leave each repository clean after the normal commit/finalizer workflow.
