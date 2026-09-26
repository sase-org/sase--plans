---
tier: epic
title: Finish model manifest parity cleanup and maintenance proof
goal: Remove the temporary migration snapshot and stale manual model lists, then prove
  that a synthetic built-in model update reaches every existing model surface through
  models.yml alone.
parent_bead: sase-1aa
phases:
- id: retire_baseline
  title: Remove the temporary model-catalog parity snapshot
  depends_on: []
  size: small
  description: 'retire_baseline: remove the phase-1 migration snapshot after confirming
    no runtime or test consumer needs it; retain behavior-based parity checks.'
- id: refresh_prose
  title: Replace stale model and pool enumerations with generated-table links
  depends_on: []
  size: medium
  description: 'refresh_prose: remove the duplicate manual model list and rewrite
    current-value prose that would go stale after a manifest-only catalog or alias
    retune.'
- id: exercise_manifest
  title: Prove a synthetic manifest update across model surfaces
  depends_on:
  - retire_baseline
  - refresh_prose
  size: medium
  description: 'exercise_manifest: demonstrate that one manifest edit reaches routing,
    alias resolution, TUI picker/completion, and the LSP payload, with generated docs
    as the only derived file.'
proposed_by: bbugyi200.apollo.sase-1aa.land
create_time: 2026-09-26 08:43:31
status: wip
bead_id: sase-1aa.5
---

- **PROMPT:** [prompts/202609/finish_model_catalog_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_model_catalog_landing.md)
- **PARENT:** [202609/model_catalog_maintenance.md](https://github.com/sase-org/sase--plans/blob/main/202609/model_catalog_maintenance.md)
- **BEAD:** [sase-1aa.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aa/sase-1aa.5.md)

# Remaining work from sase-1aa landing audit

The original epic moved the seven built-in providers and five size aliases into
`src/sase/llm_provider/models.yml`, added policy validation, and generated eleven
documentation blocks. Its four phases are closed. The landing audit found three
unfinished acceptance details. `tests/llm_provider/_phase2_model_catalog_baseline.json`
is still tracked although the phase-2 plan calls for removing temporary parity
snapshots. `docs/llms.md` retains a hand-maintained full model list under Automatic
Provider Resolution beside the new generated catalog. README and several docs still
state today's Grok pool members and fallbacks in prose. The synthetic update test in
`tests/test_render_model_docs.py` proves rendering and policy, but does not exercise the
registry, TUI model surfaces, alias resolution, or LSP payload required by phase 4.

## Phase 1: retire_baseline

Remove the unreferenced phase-1 JSON snapshot after checking that no code or test reads
it. Keep the invariant and behavior tests that derive expectations from the shipped
manifest and registry. Do not replace the snapshot with another value-pinned model list.

Acceptance: the temporary file is gone, and the existing model-manifest, routing, and
completion tests still pass.

## Phase 2: refresh_prose

Replace the duplicate hand-maintained Automatic Provider Resolution model list in
`docs/llms.md` with a link to the generated Built-in Model Catalog, while preserving the
separate explanation of hidden `fakey` and third-party hooks. Sweep `README.md`,
`docs/getting_started.md`, `docs/agent_providers.md`, `docs/ace.md`,
`docs/configuration.md`, and `docs/llms.md` for prose that enumerates current size-alias
pool membership, tier defaults, newest built-in models, or a fixed count of a provider's
catalog. Link to the generated tables where a routine manifest retune would otherwise
make the statement false. Keep stable command examples and explanations of provider
mechanics and privacy choices. In particular, preserve Muse Contributor warnings without
pinning the shipped pool membership in ungenerated prose.

Acceptance: a routine manifest-only addition or pool retune does not leave a second
manual catalog or stale shipped-value claim in those docs, and the docs formatter/check
passes.

## Phase 3: exercise_manifest

Use a temporary manifest copy or reversible fixture to add a synthetic built-in model
and retune a tier or size-alias selector to it. Regenerate the documentation blocks and
check that a second render is a no-op and manifest policy passes. With the same
synthetic data, verify the existing registry hooks and routing, alias resolution, model
picker visibility, TUI completion, and `model_completion_catalog_payload()` consumed by
the LSP. Keep the current hook and wire contracts; use the existing registry/payload
helpers instead of adding a parallel catalog. The proof may be a focused regression test
or a documented reversible exercise, but it must report results for every named surface
and show that only `models.yml` is hand-edited and generated docs are the only derived
diff.

Run focused tests, then `sase tool run check` (`just check` through the guarded recipe).
Do not run `just check-full`. Address any failures caused by this work; unrelated test
infrastructure failures belong on their existing task beads.

Acceptance: a synthetic manifest update reaches all named surfaces without changes to
provider Python code, value-pinned tests, examples, or prose; generated docs and policy
checks are green, and the temporary parity snapshot is absent.
