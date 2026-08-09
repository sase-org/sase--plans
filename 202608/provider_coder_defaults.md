---
tier: tale
title: Pin the Claude and Codex coder alias defaults
goal:
  Make unconfigured @claude_coder and @codex_coder resolve to claude/sonnet and codex/gpt-5.5 respectively, while
  preserving explicit override precedence and @coder fallback behavior for other providers.
size: medium
proposed_by: bbugyi200.athena.rp.f2
create_time: 2026-08-02 07:41:06
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.rp.f2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.rp.f2.md)
- **COMMITS:**
  - [e4c13b3](https://github.com/sase-org/sase/commit/e4c13b3e837b8d8464013cf52194a596a1c4ac9b) — feat(llm): add
    provider-specific coder defaults

# Pin `@claude_coder` and `@codex_coder` to provider-local defaults

## Goal

Make the two primary planner-provider coder aliases resolve out of the box as follows:

- `@claude_coder` -> `claude/sonnet`
- `@codex_coder` -> `codex/gpt-5.5`

Keep provider-coder aliases dynamically generated from the registered provider set. Provider-specific aliases without a
shipped target must continue to fall back through `@coder`, and users must retain the existing ways to override coder
routing at configuration, temporary-override, launch-scoped, and approval-selection layers.

## Current behavior and design constraints

- `src/sase/llm_provider/model_alias_defaults.yml` is the bundled source of truth for fixed implicit-alias defaults, but
  it currently contains only the generic `coder` role with fallback `@default`. The Python policy synthesizes every
  registered `<provider>_coder` alias at runtime and gives all of them the hard-coded fallback `@coder`; therefore this
  request cannot be implemented as two existing-string substitutions.
- The bundled schema must distinguish fixed roles from provider-derived coder targets. Adding `claude_coder` and
  `codex_coder` to the fixed role list would incorrectly expose or classify them when their providers are not
  registered. Store provider-keyed targets under the existing `coder` policy entry, validate them with the bundled
  resource, and derive `<provider>_coder` names only at the model-alias configuration boundary.
- Preserve the current override hierarchy. A launch-scoped specific or generic `coder` value wins first; a temporary or
  configured provider-specific value wins over the shipped target; and an explicitly configured or temporarily
  overridden generic `@coder` continues to act as a fleet-wide coder override. Use the provider-specific shipped target
  only when none of those explicit values applies. Providers with no shipped target continue through `@coder` to its
  configured value or `@default`.
- Direct provider targets carry no reasoning-effort suffix. Normal outer effort suffixes and approval-time explicit
  model choices remain authoritative.
- ACE and `%model` completion metadata must describe the actual path: the two new direct defaults should appear as
  implicit concrete targets rather than `implicit -> @coder`; when an explicit generic coder override is shaping them,
  their effective target and provenance must still agree.
- Existing examples that intentionally configure cross-provider handoffs (for example a Claude planner handing coding to
  Codex) remain valid override examples. They should not be mechanically rewritten merely because the shipped defaults
  become provider-local.

## Implementation

1. Extend the bundled coder policy and its validating loader.
   - Add a provider-target mapping beneath `aliases.coder` in `src/sase/llm_provider/model_alias_defaults.yml`, with
     `claude: claude/sonnet` and `codex: codex/gpt-5.5`.
   - In `src/sase/llm_provider/model_alias_policy.py`, validate that this optional mapping appears only on `coder`, has
     non-empty provider names and non-empty string targets, and does not weaken the existing exhaustive fixed-role,
     fallback/target exclusivity, or description checks. Cache and expose it as an immutable provider-coder target
     mapping while keeping the fixed-role accessors semantically unchanged.
   - Update nearby policy and `src/sase/default_config.yml` comments so they explain that selected provider-coder
     aliases have shipped direct defaults while unpinned providers inherit `@coder`. Keep the commented configuration
     block clearly labeled as override/grammar examples rather than a second source of shipped truth.

2. Thread provider-coder defaults through resolution without making them fixed roles.
   - Update `src/sase/llm_provider/model_alias_config.py` to translate configured provider names into registered
     `<provider>_coder` aliases, expose the appropriate implicit value, preserve `provider_coder`
     kind/description/order, and retain `@coder` as the structural fallback for aliases without a target.
   - Update `src/sase/llm_provider/model_alias_resolution.py` so provider-coder resolution observes the precedence
     stated above: launch-scoped specific/generic override, provider-specific temporary/configured value, explicit
     generic temporary/configured coder value, shipped provider target, then the generic `@coder` fallback. Reuse the
     existing cycle/depth, effort, selector, and provider lookup paths rather than adding a parallel resolver.
   - Include the provider-coder implicit values anywhere alias chains or selector members are validated or inspected so
     a reference to either alias is treated consistently by launch, doctor/editor validation, preview, and display
     callers. Do not make unavailable providers appear in alias catalogs; registration still controls which
     `<provider>_coder` names exist.

3. Keep alias views and completion presentation aligned with resolution.
   - Update `src/sase/llm_provider/alias_view.py` and any small presentation helpers needed so the unconfigured Claude
     and Codex rows carry direct `implicit_value` metadata and no fallback-reference arrow, while unpinned provider
     coder rows still show `implicit -> @coder`.
   - Preserve truthful provenance when a configured or temporary generic `@coder` override supersedes a shipped provider
     target, and retain the existing `coders` bucket membership, ordering, edit/reset paths, and custom-builtin shadow
     warnings.
   - Ensure `%model` completion rows use the same effective provider/model and reference metadata as the Models panel.

4. Add and retarget focused regression coverage.
   - In `tests/llm_provider/test_model_alias_defaults.py`, pin the provider-target mapping, its exact concrete values,
     and its validation/shape independently from the fixed-role target and fallback sets. Extend
     `tests/llm_provider/test_model_alias_defaults_docs_sync.py` so both documentation sources must quote the shipped
     provider-coder targets as well as the fixed implicit targets.
   - In `tests/llm_provider/test_config_role_aliases.py`, cover direct unconfigured Claude/Codex resolution regardless
     of the active default provider, no inherited effort, and continued `@coder` fallback for an unpinned registered
     provider. Replace only assertions whose purpose was the old universal provider-coder fallback.
   - Add precedence cases across the existing resolution/override tests for specific configuration and temporary
     overrides, generic configured/temporary `@coder` overrides, launch-scoped generic and specific overrides, and an
     outer effort suffix. Confirm that the more specific existing paths still win and that unrelated role aliases do not
     change.
   - Update alias-view, bucket, rendering, and completion tests to distinguish direct provider defaults from unpinned
     `@coder` fallbacks and to assert accurate model/provenance metadata. Leave manually constructed arbitrary-model
     fixtures unchanged where their purpose is layout, editing, selector grammar, or warning behavior rather than the
     shipped default.

5. Refresh intentional visual fixtures and documentation.
   - Make the default Models-panel fixture show implicit `claude_coder` as Claude Sonnet and implicit `codex_coder` as
     Codex GPT-5.5; keep separate configured/override variants for their existing purposes. Update the model-completion
     fixture so its provider-coder row is a direct target with no obsolete `@coder` reference. Regenerate and inspect
     every affected Models-panel and prompt-completion PNG golden rather than assuming only the drilled-in bucket image
     changes.
   - Update `docs/llms.md`, `docs/configuration.md`, `docs/xprompt.md`, and the relevant Models-panel/approval guidance
     in `docs/ace.md`. The implicit-alias table should name the two concrete defaults separately, retain a generic row
     for other registered providers, and explain override precedence and the fallback used when planner-provider
     metadata is absent.
   - Run a final targeted search for claims that every `<provider>_coder` defaults to `@coder` and for old examples near
     `claude_coder`/`codex_coder`. Classify blog posts and cross-provider YAML snippets as intentional configured
     examples when they remain accurate; update only text that claims to describe shipped behavior.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run focused non-visual coverage for the bundled policy, resolution and precedence, view metadata, completion, docs
   synchronization, and Models-panel rendering, for example:

   ```bash
   just test tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_model_alias_defaults_docs_sync.py \
     tests/llm_provider/test_config_role_aliases.py \
     tests/llm_provider/test_config_alias_resolution.py \
     tests/llm_provider/test_alias_override_resolution.py \
     tests/llm_provider/test_alias_view.py \
     tests/llm_provider/test_alias_view_panel_rows.py \
     tests/llm_provider/test_config_aliases.py \
     tests/test_models_panel_alias_rendering.py \
     tests/test_xprompt_model_completion.py
   ```

3. Regenerate the intentional visual changes with the repository's snapshot update flag for
   `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py` and
   `tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py`. Inspect the actual/expected/diff artifacts under
   `.pytest_cache/sase-visual/`, then rerun both files without update mode to prove exact convergence.
4. Search for stale universal-fallback wording and default-specific literals, and confirm every remaining cross-provider
   value is an intentional user-configuration example rather than a stale shipped-default claim.
5. Run the mandatory full repository gate with `just check`, covering formatting, linting, type checks, SASE validation,
   the full test suite, and committed PNG visual snapshots.

## Acceptance criteria

- With no explicit coder override, `@claude_coder` resolves to `claude/sonnet` and `@codex_coder` resolves to
  `codex/gpt-5.5`, both with no alias-borne effort.
- A registered provider without a shipped coder target still resolves `<provider>_coder` through `@coder` and then
  `@default` when the generic alias is also unconfigured.
- Provider-specific configuration/temporary overrides, explicit generic coder configuration/temporary overrides,
  launch-scoped generic/specific overrides, approval-selected models, and outer effort suffixes retain their documented
  precedence.
- Models-panel and `%model` completion rows show the two direct implicit targets without an obsolete `@coder` fallback
  reference, while unpinned provider rows and generic-override cases retain truthful provenance.
- Documentation consistently distinguishes the two shipped provider-local defaults from cross-provider override
  examples, all intended PNG goldens converge exactly, and `just check` passes.
