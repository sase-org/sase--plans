---
tier: tale
title: Centralize coder follow-up defaults on @coder
goal:
  All unconfigured provider-coder aliases inherit codex/gpt-5.5 through the shared @coder alias while preserving
  provider-specific overrides.
size: medium
proposed_by: bbugyi200.athena.sp
create_time: 2026-08-03 08:16:34
status: wip
---

- **PROMPT:**
  [prompts/202608/coder_alias_default.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/coder_alias_default.md)

# Centralize coder follow-up defaults on `@coder`

## Goal

Make the shipped `@coder` alias resolve directly to `codex/gpt-5.5`, and make every unconfigured registered
`@<provider>_coder` alias—including `@claude_coder` and `@codex_coder`—inherit that value through `@coder`. Preserve
provider-specific aliases as independent configuration and override points while removing the internal machinery that
only existed to give selected providers separate shipped targets.

## Behavioral contract

- With no user or temporary configuration, `@coder`, `@claude_coder`, `@codex_coder`, and every other registered
  provider-coder alias resolve to `codex/gpt-5.5`.
- The structural chain for an unconfigured provider alias is `@<provider>_coder -> @coder -> codex/gpt-5.5`; it no
  longer reaches `@default` unless a user explicitly configures `coder` to reference `@default`.
- Existing precedence remains intact: launch-scoped provider-specific/generic coder choices win first, then a
  provider-specific temporary or configured value, then the generic `@coder` temporary/configured/implicit value. An
  outer effort suffix remains authoritative.
- Provider-specific aliases remain visible and independently editable; this change consolidates only their shipped
  defaults, not their names or override paths.

## Implementation

1. Consolidate the bundled model-alias policy in `src/sase/llm_provider/model_alias_defaults.yml` by replacing `coder`'s
   `@default` fallback with the concrete `codex/gpt-5.5` target, removing the Claude/Codex `provider_targets` entries,
   and updating the surrounding description/comments to describe `@coder` as the common provider-coder default.

2. Remove the now-unused provider-specific shipped-target plumbing from the Python policy and resolver layers:
   - Simplify `src/sase/llm_provider/model_alias_policy.py` so the loaded defaults contain role fallbacks, implicit
     targets, and descriptions only; delete `provider_targets` parsing/validation and the `provider_coder_targets()`
     accessor.
   - Remove the compatibility-facade export from `src/sase/llm_provider/config.py`.
   - In `src/sase/llm_provider/model_alias_config.py`, make every recognized unconfigured provider-coder alias expose
     `@coder` as its implicit fallback and limit implicit concrete values to the normal role-target map.
   - In `src/sase/llm_provider/model_alias_resolution.py`, remove the direct-provider-target branch. Let an unconfigured
     provider-coder alias proceed through its ordinary `@coder` fallback after the existing launch, temporary, and
     configured provider-specific checks. Keep the generic launch override handling and selector validation behavior
     that apply to the shared alias graph.

3. Simplify display/completion provenance to match the uniform fallback graph:
   - Remove `AliasView.implicit_reference` and the special computation in `src/sase/llm_provider/alias_view.py`; the
     standard implicit fallback now correctly reports `coder` for every provider-coder row.
   - Retain `src/sase/xprompt/model_completion.py`'s generic-coder temporary-override propagation so cached completion
     rows for all implicit provider-coder aliases show the live generic target, while a specifically overridden provider
     alias still wins.
   - Update Models-panel, picker dependency, completion, and alias-view tests to assert the common fallback/reference,
     effective target, effort behavior, and unchanged provider-specific override precedence. Remove tests dedicated to
     parsing or exposing the deleted shipped provider-target map.

4. Update the public explanation of the policy in `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`,
   `docs/llms.md`, `docs/configuration.md`, `docs/xprompt.md`, and `docs/ace.md`:
   - List `codex/gpt-5.5` as `@coder`'s shipped value and `@coder` as every `@<provider>_coder` default.
   - Describe the shortened override order without a separate shipped-provider-target tier.
   - Explain that coder follow-ups are independent of `@default` out of the box, while provider-specific configuration
     still routes one planner provider differently.
   - Keep example configuration clearly labeled as overrides rather than shipped defaults.

5. Refresh deterministic visual fixtures so the coder row uses `CODEX(gpt-5.5)`, every unconfigured provider-coder row
   renders `implicit -> @coder`, and a configured generic coder target is reflected by provider-coder completion rows.
   Run the visual suite, inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/`, and update only
   the PNG goldens whose changes follow from the new target/reference provenance.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused unit tests covering bundled-default loading/docs sync, alias resolution and override precedence,
   alias views, completion metadata, picker dependency guards, and Models-panel rendering.
3. Run `just test-visual`; inspect any failures and accept only intentional coder target/reference snapshot changes with
   `--sase-update-visual-snapshots`, then rerun the visual suite cleanly.
4. Run the mandatory full `just check` and resolve any lint, type, unit, documentation-sync, or visual snapshot failures
   before handing off the implementation.

## Non-goals

- Do not rename or remove `@claude_coder`, `@codex_coder`, or other registered provider-coder aliases; plan approval
  continues to select the alias associated with the planner provider.
- Do not change user configuration keys, approval-time model selection, or provider-specific launch metadata.
- Do not change the defaults for `@default`, phase-worker aliases, lander aliases, or the smart/cheap alias families.
