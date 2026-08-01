---
tier: tale
title: Make @cheapest a load-balanced pool
goal:
  SASE round-robins explicit @cheapest launches between the existing Haiku and GPT-5.3 Codex Spark members, with
  independent cursor state and consistent tests, Models-panel presentation, examples, and documentation.
proposed_by: bbugyi200.athena.rc.f1
create_time: 2026-08-01 10:39:24
status: done
---

- **PROMPT:** [202608/prompts/make_cheapest_load_balanced.md](prompts/make_cheapest_load_balanced.md)

# Plan: Make `@cheapest` a load-balanced pool

## Goal

Change the shipped `@cheapest` definition from the ordered provider fallback `claude/haiku || codex/gpt-5.3-codex-spark`
to the round-robin pool `claude/haiku | codex/gpt-5.3-codex-spark`, and keep runtime expectations, Models-panel
presentation, configuration examples, and delegated-work documentation consistent with that policy.

## Context and decisions

`src/sase/llm_provider/model_alias_defaults.yml` is the source of truth for implicit alias targets. The existing
selector parser and resolver already give `|` the requested semantics: they filter out unavailable providers, peek at
the next member without advancing it for display/read-only resolution, advance a persistent machine-global cursor only
when a launch consumes the alias, and key that cursor by the owning alias plus its member fingerprint. No parser,
schema, Rust-core, or cursor-state format change is needed.

This remains an explicit-use alias with no automatic consumer. The member order and models remain unchanged, so a fresh
`@cheapest` rotation starts with Claude Haiku and then selects Codex GPT-5.3 Codex Spark when both providers are
available. `@cheapest` must own a cursor independent of `@cheap` and `@cheaper`; a temporary or launch-scoped override
must continue to suspend selection without consuming that cursor. `@smartest` remains an ordered `||` provider fallback.
User-configured aliases remain free to use either supported selector mode, including overriding `cheapest` with `||`;
only claims about the shipped implicit default should be rewritten.

## Implementation

1. Update the `cheapest.target` entry in `src/sase/llm_provider/model_alias_defaults.yml` to the exact selector
   `claude/haiku | codex/gpt-5.3-codex-spark`, and revise its bundled description to call it a lowest-cost load-balanced
   pool for explicit use. Synchronize the policy comment in `src/sase/llm_provider/model_alias_policy.py` so it
   identifies only `@smartest` as the shipped ordered fallback and groups `@cheapest` with the independent built-in
   pools.
2. Update default-dependent regression coverage while preserving generic fallback fixtures that intentionally test `||`:
   - In `tests/llm_provider/test_model_alias_defaults.py`, pin `@smartest` as a fallback and all three cheap-family
     aliases as round-robin pools; also pin `@cheapest`'s ordered members so the one-character policy change cannot
     silently reorder or replace them.
   - In `tests/llm_provider/test_config_role_aliases.py`, `tests/llm_provider/test_alias_view.py`, and
     `tests/llm_provider/test_config_aliases.py`, change the `@cheapest` selector-mode and description assertions from
     fallback to pool semantics while retaining its first-member result for non-consuming resolution.
   - In `tests/llm_provider/test_load_balanced_aliases.py`, explicitly prove that consuming `@cheapest` alternates Haiku
     and GPT-5.3 Codex Spark, peeking does not advance it, provider availability filtering still skips an unavailable
     member, and its cursor is independent of the `@cheap` and `@cheaper` cursors. Retain or strengthen the
     fingerprint-reset regression so a differently configured prior `@cheapest` pool cannot move the new shipped pool
     off its first member.
   - Include `@cheapest` in the selector-backed/default-override coverage in
     `tests/llm_provider/test_alias_override_resolution.py`, and keep the existing generic tests that show temporary and
     launch-scoped overrides suspend pool consumption.
   - Update the representative builtin value in `tests/test_config_schema_models.py` to the new single-pipe default; do
     not weaken schema/doctor coverage for valid ordered fallbacks or invalid mixed/empty selectors.
3. Update `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py` so the default `@cheapest` row is a
   round-robin selector with the new pool description and the same two members. Refresh only the Models-panel PNG
   goldens whose rendered alias list or detail text changes, and inspect representative default and drilled-in/detail
   renders before accepting them; leave arbitrary configured-value fixtures unchanged when they are not assertions about
   the shipped default.
4. Synchronize shipped-policy text and copyable examples:
   - In `src/sase/default_config.yml`, describe `cheap`, `cheaper`, and `cheapest` as independent load-balanced pools
     and make the commented `cheapest` grammar example use `|`; keep `smartest` as the `||` example.
   - In `docs/llms.md`, update every default configuration sample, the implicit-role-alias overview/table, explicit-use
     explanation, and override behavior to describe `@cheapest` as an independent rotation.
   - In `docs/configuration.md`, `docs/ace.md`, `docs/beads.md`, and `docs/sdd.md`, replace shipped-default claims and
     examples that call `@cheapest` an ordered provider fallback or say it has no pool cursor. Preserve the statement
     that it has no automatic role consumer.

## Verification

1. Run focused tests for bundled defaults, role/alias resolution, selector rotation and overrides, alias views,
   configuration schema, and Models-panel behavior. Run the Models-panel visual snapshot modules, inspect the generated
   diffs, accept only intentional `@cheapest` pool-mode changes, and rerun them with exact pixel comparison.
2. Search `src`, `tests`, and `docs` for the old exact selector and for `cheapest` near `fallback`, `ordered`, or
   no-cursor language. Classify every remaining match: shipped-default claims must be gone, while generic parser,
   validation, doctor, and user-override examples may deliberately retain `||`.
3. Run `just install` as required for the ephemeral workspace, then run the repository-mandated `just check`. Fix any
   formatting, lint, unit, schema, documentation, or visual-snapshot failures caused by this change; record unrelated
   pre-existing failures through the repository's task-bead workflow.

## Acceptance criteria

- The bundled `@cheapest` target is exactly `claude/haiku | codex/gpt-5.3-codex-spark`, with both members unchanged and
  `@smartest` still using ordered fallback semantics.
- With both providers available, consuming explicit `@cheapest` launches alternates between Haiku and GPT-5.3 Codex
  Spark; non-consuming resolution reports the next member without advancing it, and availability filtering skips
  unavailable providers.
- `@cheapest` has rotation state independent from `@cheap` and `@cheaper`, and alias overrides suspend rather than
  consume the underlying rotation. It remains available only for explicit use and is not added to automatic bead or epic
  routing.
- The Models panel identifies `@cheapest` as a pool and shows its current selection, and all shipped-default source,
  tests, examples, and documentation agree on its single-pipe definition.
- Focused functional and visual tests pass, stale-default searches are clean after intentional generic cases are
  reviewed, and `just check` passes.
