---
tier: tale
title: Retune the cheap model alias pools
goal:
  SASE ships the requested Sonnet- and GPT-5.5-based defaults for @cheap and @cheaper, preserves their effort
  annotations through resolution, and presents the same values consistently in tests and user documentation.
size: medium
proposed_by: bbugyi200.athena.rb
create_time: 2026-08-01 10:02:10
status: wip
---

- **PROMPT:** [prompts/202608/retune_cheap_aliases.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retune_cheap_aliases.md)

# Plan: Retune the cheap model alias pools

## Context and decisions

`src/sase/llm_provider/model_alias_defaults.yml` is the bundled source of truth for implicit alias targets. Today,
`@cheap` is `claude/opus@medium | codex/gpt-5.5` and `@cheaper` is `claude/sonnet | codex/gpt-5.3-codex-spark`. Replace
them with these exact round-robin selectors:

- `@cheap`: `claude/sonnet@xhigh | codex/gpt-5.5`
- `@cheaper`: `claude/sonnet@medium | codex/gpt-5.5@medium`

Keep the existing `|` round-robin semantics, independent cursor state for the two aliases, and the `@small_phase_worker`
-> `@cheap` / `@xsmall_phase_worker` -> `@cheaper` routing. Do not change `@cheapest`, alias descriptions, provider
catalogs, or generic selector examples and fixtures that intentionally exercise other valid configurations. The selector
strings' changed fingerprints should naturally start fresh pool rotations; no persisted cursor migration is needed.

## Implementation

1. Update only the `cheap.target` and `cheaper.target` entries in `src/sase/llm_provider/model_alias_defaults.yml` to
   the exact selectors above. Rely on the existing validated YAML loader and resolution pipeline, because both models
   and both effort levels are already recognized by their provider adapters.
2. Refresh default-dependent regression expectations in the LLM-provider and work-rendering tests. In particular:
   - Exercise `@cheap`, `@cheaper`, `@small_phase_worker`, and `@xsmall_phase_worker` through
     `resolve_model_alias_with_effort()` where necessary so the tests assert both the selected concrete model and the
     requested `xhigh`/`medium` effort, not merely the effort-stripped model name.
   - Keep coverage that the size aliases share their owning pool's rotation, `@cheap` and `@cheaper` retain independent
     rotations even though their concrete members now overlap, provider availability filtering still selects an
     available member, and unrelated/default overrides do not reroute selector-backed lanes.
   - Update only assertions and representative builtin-alias samples that claim to reflect shipped `cheap`/`cheaper`
     behavior; retain generic pool/parser, override, model-picker, and visual fixture values when their purpose is to
     cover arbitrary valid configurations rather than document these defaults.
3. Synchronize every user-facing restatement of the shipped values:
   - In `docs/llms.md`, update the configuration examples, implicit-role-alias table, and delegated-work examples while
     preserving the explanation that both aliases are independent load-balanced pools.
   - In `docs/configuration.md`, update the builtin-alias configuration example.
   - In `docs/ace.md`, update the Models-panel edit examples for `cheap` and `cheaper` so copied values reproduce the
     new defaults and their effort annotations. Leave generic effort-suffix illustrations unchanged when they are not
     presented as an alias default.

## Verification

1. Run the focused LLM-provider/default-sync, alias-resolution, configuration-schema, Models-panel, and work-rendering
   tests touched by the change. Confirm they cover the exact selector members and the effective effort of each member.
2. Search the source and documentation for the retired default strings and inspect any remaining matches to ensure they
   are deliberately generic test data rather than stale claims about `@cheap` or `@cheaper`.
3. Run `just install` as required for an ephemeral SASE workspace, then run the repository-mandated `just check` suite
   and resolve any formatting, lint, validation, unit-test, or visual-snapshot regressions caused by the update.

## Acceptance criteria

- With both providers available, `@cheap` round-robins between Sonnet at `xhigh` effort and GPT-5.5 with no member-level
  effort suffix, while `@small_phase_worker` consumes that same pool.
- With both providers available, `@cheaper` round-robins between Sonnet at `medium` effort and GPT-5.5 at `medium`
  effort, while `@xsmall_phase_worker` consumes that same pool.
- The two pools remain independently cursor-tracked, availability-aware, and overrideable through the existing alias
  mechanisms; `@cheapest` and all other implicit defaults are unchanged.
- The canonical YAML, executable expectations, and all documentation that states these defaults agree exactly, and the
  full repository check passes.
