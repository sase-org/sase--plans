---
tier: tale
title: Change the implicit @smartest default to Claude Opus at max effort
goal:
  Unconfigured @smartest launches and dependent roles resolve to claude/opus@max while overrides and general selector
  support remain intact.
proposed_by: bbugyi200.athena.rp
create_time: 2026-08-02 06:59:12
status: wip
---

# Change the implicit `@smartest` default to `claude/opus@max`

## Goal

Make an unconfigured `@smartest` resolve to the single concrete target `claude/opus` with a `max` reasoning-effort
overlay. Preserve user-configured `llm_provider.model_aliases.builtin.smartest` overrides and the general model alias
selector grammar, but remove the shipped Claude/Codex availability fallback from this alias. Because `@big_epic_lander`
and `@xlarge_phase_worker` fall back to `@smartest`, both roles must inherit the new target and effort unless they are
explicitly overridden.

## Current behavior and constraints

- `src/sase/llm_provider/model_alias_defaults.yml` is the bundled source of truth. It currently defines `smartest` as
  the ordered fallback `claude/claude-fable-5 || codex/gpt-5.6-sol`; the existing loader and resolver already accept a
  single provider/model target with a trailing effort, so no new resolution machinery or Rust-core API is needed.
- The change intentionally alters selector semantics: implicit `@smartest` will no longer choose a provider by CLI
  availability, expose selector members in ACE, or be insulated from effort resolution as a selector. It will expose
  `max` as alias-borne effort and still remain a concrete pinned target independent of `@default`.
- Selector behavior must remain covered for user-configured aliases. Tests that construct an explicit `smartest` ordered
  fallback to exercise schema, doctor, editor, or generic selector behavior should not be mechanically rewritten as if
  ordered fallbacks were no longer supported.
- The checked-in ACE PNG goldens are exact visual regressions. Updating the shared Models-panel fixture can affect many
  snapshots in which the `smartest` row is visible, not only the snapshot currently named for the ordered fallback.

## Implementation

1. Change the `smartest.target` entry in `src/sase/llm_provider/model_alias_defaults.yml` to `claude/opus@max`. Update
   nearby source comments in `src/sase/llm_provider/model_alias_policy.py` and `src/sase/default_config.yml` so they
   describe `@smartest` as a concrete maximum-effort target rather than an ordered provider fallback. Keep the
   documented `A | B` and `A || B` configuration grammar intact, and make the commented `smartest` example consistent
   with the new default where it is presented as an alias example.

2. Retarget the default-specific unit coverage:
   - In `tests/llm_provider/test_model_alias_defaults.py`, pin the shipped `smartest` value by splitting its effort and
     asserting the concrete `claude/opus` target plus `max`; keep the `cheap`, `cheaper`, and `cheapest` selector-shape
     assertions independent.
   - In `tests/llm_provider/test_config_role_aliases.py` and `tests/llm_provider/test_alias_override_resolution.py`,
     assert through `resolve_model_alias_with_effort` that `@smartest`, `@big_epic_lander`, and `@xlarge_phase_worker`
     resolve to `claude/opus@max`, and replace the obsolete provider-availability matrix with coverage of the
     single-target behavior and effort inheritance.
   - In `tests/llm_provider/test_alias_view.py`, update the implicit `smartest` and dependent-role views to show Claude
     Opus at `max`, with no selector mode or selector members. Replace the obsolete Claude-unavailable/Codex-selected
     case with a direct-target view assertion.
   - Leave tests using explicit configured fallback expressions in `tests/doctor/test_checks_config_model_aliases.py`,
     `tests/llm_provider/test_load_balanced_aliases.py`, `tests/test_config_schema_models.py`, editor tests, and
     rendering-helper tests unchanged when their purpose is to validate the still-supported selector grammar rather than
     the shipped default.

3. Update the Models-panel visual fixture in `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py` so
   `smartest`, `big_epic_lander`, and `xlarge_phase_worker` display Claude Opus with `max` effort, and the `smartest`
   row is a normal non-selector row. Rename the fallback-specific visual test/snapshot key in
   `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py` to describe the maximum-effort target, regenerate all
   affected PNG goldens under `tests/ace/tui/visual/snapshots/png/`, and inspect the actual/diff artifacts to confirm
   that changes are limited to the expected model, effort, and description presentation. Remove the obsolete
   fallback-named golden when its replacement is accepted.

4. Synchronize every user-facing statement of the shipped default in `docs/llms.md`, `docs/configuration.md`, and
   `docs/ace.md`, including YAML examples, the implicit-role table, override behavior, Models-panel behavior, and
   examples. State that `@smartest` is `claude/opus@max`, that dependent xlarge workers and threshold-sized epic landers
   inherit `max`, and that changing the alias through configuration or a temporary override still takes precedence.
   Update the provider-aware wording in `docs/sdd.md` and any remaining source/docs references found by a final targeted
   search, while retaining explanations of ordered fallbacks as a supported user-configured selector feature.

## Validation

1. Run `just install` before project checks, as required for an ephemeral SASE workspace.
2. Run focused non-visual tests for the bundled defaults, resolution, alias views, docs synchronization, and relevant
   override behavior, for example:

   ```bash
   just test tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_model_alias_defaults_docs_sync.py \
     tests/llm_provider/test_config_role_aliases.py \
     tests/llm_provider/test_alias_view.py \
     tests/llm_provider/test_alias_override_resolution.py
   ```

3. Regenerate the intentional Models-panel snapshots with
   `just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py --sase-update-visual-snapshots`,
   inspect `.pytest_cache/sase-visual/`, then rerun the same visual file without `--sase-update-visual-snapshots` to
   prove exact convergence.
4. Search for stale default-specific wording and literals (including `claude/claude-fable-5`, `ordered fallback`, and
   `provider-aware` near `smartest`) and classify each remaining match as either intentional generic selector coverage
   or an unrelated explicit model example.
5. Run the mandatory full repository gate with `just check`; it includes formatting, linting, SASE validation, the full
   test suite, and PNG visual snapshot verification.

## Acceptance criteria

- With no builtin override, resolving `@smartest`, `@big_epic_lander`, or `@xlarge_phase_worker` yields target
  `claude/opus` and effort `max`.
- Implicit `@smartest` no longer exposes or performs ordered provider fallback selection, while explicitly configured
  `A || B` aliases continue to work.
- ACE renders the implicit alias and its dependent roles as Claude Opus with `max` effort and no selector members; all
  committed PNG goldens match the intentional fixture state.
- Documentation consistently names `claude/opus@max` as the shipped default without implying a Codex availability
  fallback.
- User builtin overrides, temporary overrides, and explicit launch models keep their existing precedence, and
  `just check` passes.
