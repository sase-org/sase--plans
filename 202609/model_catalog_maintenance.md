---
tier: epic
title: Make built-in model and size-alias updates a one-file edit
goal: Maintainers update one bundled manifest for built-in model catalogs, tier defaults,
  and size-alias pools, then regenerate checked documentation without editing value-pinned
  tests.
phases:
- id: data_driven_tests
  title: Replace shipped-value copies in tests and guard the existing generated alias
    table
  depends_on: []
  size: medium
  description: 'data_driven_tests: derive routing and completion expectations from
    shipped data, preserve behavioral policy tests, and add a generated-doc drift
    check to fast verification and CI.'
- id: unified_manifest
  title: Move built-in provider model data and size aliases into one manifest
  depends_on:
  - data_driven_tests
  size: medium
  description: 'unified_manifest: add a strictly validated, lazy models.yml loader
    and migrate seven provider hooks and tier invocation defaults while preserving
    current metadata and completion output.'
- id: model_policy
  title: Validate shipped size-alias policy from the manifest
  depends_on:
  - unified_manifest
  size: medium
  description: 'model_policy: enforce catalog membership, effort support and descent,
    provider redundancy, and deliberate selector shape with actionable diagnostics.'
- id: generated_model_docs
  title: Generate model tables and prove the maintainer workflow
  depends_on:
  - model_policy
  size: medium
  description: 'generated_model_docs: render mirrored model tables from YAML, replace
    freshness prose with table links, and verify a catalog or pool update requires
    only one hand-edited file.'
proposed_by: bbugyi200.apollo.1t
create_time: 2026-09-25 22:26:12
status: wip
bead_id: sase-1aa
---

- **PROMPT:** [prompts/202609/model_catalog_maintenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/model_catalog_maintenance.md)
- **BEAD:** [sase-1aa](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aa/README.md)

# Context and outcome

The research artifact
`research:202609/model_catalog_and_size_alias_maintenance/model_catalog_and_size_alias_maintenance.md`
found that the prompt input widget, model picker, and Rust xprompt LSP already consume
one Python-built model catalog. The maintenance cost comes from seven provider modules,
`model_alias_defaults.yml`, value-pinned tests, and hand-maintained documentation. Keep
the existing registry hooks, completion wire format, and LSP materialization path. This
is a data and maintenance change within the existing Python provider boundary; do not
move catalog data into `sase-core` or add runtime provider-CLI discovery.

The intended maintainer workflow is: edit `src/sase/llm_provider/models.yml`; run
`just fix` to regenerate `docs/llms.md`; run `just check`. A normal built-in model
addition, tier change, or size-pool retune should require no edits to Python source,
tests, examples, or prose. Generated Markdown is an expected review diff. Preserve all
currently shipped model IDs, ordering, short aliases, tier defaults, advisory wording,
alias targets, and descriptions during migration; any deliberate retune is a separate
change.

# Phase 1: data-driven tests and first drift gate

1. Refactor `tests/llm_provider/test_load_balanced_alias_defaults.py`,
   `test_ordered_fallback_aliases.py`, `test_model_alias_defaults.py`, and the model
   catalog, picker, routing, and LSP payload tests to obtain expected members, efforts,
   and provider/model rows from the shipped alias loader or registry. Exercise
   resolution with one available provider at a time and assert selected target and
   effort. Compare completion and picker rows for visible providers, respecting the
   existing hidden-provider policy. Retain frozen selector-graph fixtures and focused
   behavioral tests for rotation, fallback order, provider filtering, and the
   intentional Muse contributor route. Replace current-model-specific assertions with
   provider/alias behavior where the model ID is incidental. Keep tests whose purpose is
   a specific advisory or historical compatibility case.
2. Make `tools/render_model_alias_docs` support `--check` without writing; compare its
   in-memory output with the marked block on disk and return nonzero with a useful
   diff/error on drift. Add `fmt-docs-check` in `Justfile` and run it in `fmt-check`,
   `just check`, `.github/workflows/master-gate.yml`, and `.github/workflows/ci.yml`
   beside their Markdown formatting checks. Test both current-file success and an
   intentionally stale block. This gate must run independently of scoped pytest
   selection.
3. Capture the model-related fields of the baseline registry metadata and
   `model_completion_catalog_payload()` during this phase for a one-time migration
   comparison in Phase 2; exclude changing cache fingerprints and environment state. Do
   not leave a value-pinned golden file as the long-term test oracle. Keep policy
   tripwires in place until Phase 3 replaces them.

Acceptance: changing only an alias target makes the generated-doc check fail until
regeneration, while routing and completion tests remain meaningful without a fresh list
of model IDs in assertions. Run focused tests and the project's prescribed fast
verification recipe.

# Phase 2: one bundled manifest

1. Replace `src/sase/llm_provider/model_alias_defaults.yml` with
   `src/sase/llm_provider/models.yml`, containing `schema_version`, an ordered
   `providers` map, and the five `size_aliases` (or preserve `aliases` as the section
   name if that minimizes loader churn). Each provider records ordered model IDs,
   optional short aliases, `large`/`small` tier defaults, optional `supersedes` links,
   and any advisory needed to preserve current warnings. Keep `fakey` as a separate
   hidden test provider. Represent nested OpenCode IDs as opaque model strings; preserve
   Antigravity's CLI order and Muse's training advisory. Leave effort-to-CLI mechanics
   in provider Python code.
2. Implement one lazy, cached, immutable manifest loader with strict structural
   diagnostics for unknown keys, duplicates (including globally ambiguous bare names and
   short aliases), missing tier IDs, invalid `supersedes` links/cycles, malformed
   advisory fields, and the existing five-alias grammar. Reuse the current selector
   parser and alias-policy accessors so user config overrides and alias resolution
   retain their behavior. Ensure the manifest is included in the installed package and
   is loaded outside per-keystroke completion paths.
3. Make the seven user-facing provider modules (`claude`, `codex`, `grok`, `agy`,
   `muse`, `qwen`, `opencode`) answer their existing model-name, short-alias, advisory,
   and tier hooks from the manifest. Replace their `_TIER_TO_MODEL` invocation defaults
   with the same accessor. Preserve the pluggy hook contract for third-party providers;
   do not add a second registry or change the Rust completion wire. Update references in
   schema, shipped config comments, renderer, and docs to the new source path.
4. Compare before/after registry metadata, model picker rows, and
   `model_completion_catalog_payload()` including row order, aliases, advisories, and
   provider counts. Verify a packaged install can read the YAML and that a third-party
   provider still contributes models through hooks. Remove temporary parity snapshots
   after the comparison; ongoing tests should check invariants and behavior.

Acceptance: the shipped alias resolutions and model/tier choices are byte-for-byte or
structurally identical to the baseline as appropriate, and both TUI and LSP completion
still derive from the unchanged shared payload.

# Phase 3: policy as executable rules

Create a focused validator for bundled values, used by tests and the docs check, that
reports alias, member, and suggested correction. It must check:

- Every explicit `provider/model` selector member belongs to that built-in provider's
  manifest catalog, including fallback and last-resort members.
- Each explicit effort is supported by the corresponding built-in provider's existing
  effort mechanics; Antigravity members have no `@effort` suffix. Do not duplicate
  capability lists in YAML. If a CLI delegates model-specific validation, state that
  limit in the diagnostic/test rather than claiming stronger validation.
- From `@xlarge` down to `@xsmall`, each primary appearance of a model starts at `xhigh`
  and descends one supported rung on later appearances. A predecessor named by
  `supersedes` continues the successor's descent. Non-primary ordered fallbacks and
  last-resort tails do not consume rungs. Fail when descent would go below a provider's
  supported range; never silently clamp or auto-rewrite authored efforts.
- Each alias can reach at least two providers through its pool or fallback chain, and
  its selector shape follows the existing frozen graph-shape contract. Keep meaningful
  intentional product-claim tests, such as Muse contributor inclusion, separate from
  generic validator rules.

Malformed bundled structure should fail at load with a clear installation-defect
message. Policy violations should fail `just check`/CI with fix-it messages, without
adding runtime startup work. Add negative tests for a typo, unsupported effort, wrong
rung, broken `supersedes`, and single-provider alias. Do not edit the historical
size-alias decision record for routine model retunes.

# Phase 4: generated docs and workflow proof

1. Extend/rename the YAML-only renderer to `tools/render_model_docs`. Keep the
   alias-default block and generate the `docs/llms.md` known-model, short-alias, and
   provider tier-default tables from the manifest, with stable block markers and
   deterministic ordering. `--check` must compare every generated block to the file on
   disk and run in the Phase 1 `fmt-docs-check` gate. Preserve the renderer's
   provider-import-free behavior. Ensure `just fix` produces Markdown that passes
   formatting and a second render with no diff.
2. Replace prose that enumerates the current pool membership or newest model across
   `README.md`, `docs/getting_started.md`, `docs/agent_providers.md`, `docs/ace.md`,
   `docs/configuration.md`, and `docs/llms.md` with links to the generated table where
   behavior permits. Keep examples stable when their model IDs are still supported;
   update stale tests to assert the behavior or the link, not today's slug. Document the
   three-command maintainer workflow and the difference between catalog membership, tier
   defaults, and pool selection.
3. In a temporary working copy or reversible test fixture, add a synthetic built-in
   model, change its tier or alias use, regenerate docs, and verify registry routing,
   TUI completion, LSP payload, alias resolution, and picker visibility. Confirm only
   `models.yml` and generated docs need changing for that scenario. Run `just check`
   through `sase tool run` per project rules; do not run `check-full` without explicit
   instruction.

Acceptance: `just fix` regenerates every mirrored table, `just check` and both CI
workflows fail on stale generated content or invalid manifest policy, and the synthetic
update requires one hand-edited file. The ordinary plugin and completion interfaces
remain unchanged.

# Scope choices and risks

- The manifest covers built-in user-facing providers only. Third-party plugins continue
  to publish models through the existing hooks, and `fakey` remains specialized test
  data.
- Do not add `status: legacy`, stale-user-pin doctor advice, LSP refresh-on-upgrade,
  `extra_models`, a new model selector grammar, or automatic effort rewriting in this
  epic. They are separate behavior changes and are not needed for one-file maintenance.
  A model should remain catalogued while saved bare-name prompts still rely on it;
  retirement semantics deserve their own design.
- Antigravity's current tier defaults stay on Gemini 3.7 during the data migration. The
  generated tier table makes that choice visible for a later deliberate retune.
- The loader must avoid slowing completion and must preserve hook extensibility. The
  parity comparison and focused regression tests protect against changes to ordering,
  advisories, or routing while replacing the data source.
