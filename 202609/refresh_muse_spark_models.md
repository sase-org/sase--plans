---
tier: tale
title: Refresh the Muse Code provider for Muse Spark 1.3
goal:
  SASE routes Muse launches to the current safe model and exposes Meta's full current
  Muse Spark catalog with accurate advisories and effort behavior.
size: medium
proposed_by: bbugyi200.apollo.0x
create_time: 2026-09-19 13:45:36
status: wip
---

# Plan: Refresh the Muse Code provider for Muse Spark 1.3

## Finding

SASE is behind Meta's current Muse Code model catalog. The provider currently maps both
model tiers to `muse-spark-1.2` and publishes only `muse-spark-1.2`,
`muse-spark-1.2-contributor`, and `muse-spark-1.1` to provider resolution, model
pickers, and `%model` completion. Meta released Muse Spark 1.3 on 2026-09-02 and says it
is available in Muse Code; Meta's current catalog lists both `muse-spark-1.3` and
`muse-spark-1.3-contributor` in addition to the older models.

Primary references:

- Meta's release announcement says 1.3 improves agentic and coding work, uses fewer tool
  calls and tokens than 1.2, and is available in Muse Code:
  <https://research.meta.ai/blog/introducing-muse-spark-1-3>
- Meta's current overview lists standard models 1.3, 1.2, and 1.1 and Contributor models
  1.3 and 1.2: <https://dev.meta.ai/docs/overview>
- Meta's pricing documentation confirms that both Contributor models permit training on
  prompts and completions, while standard models do not:
  <https://dev.meta.ai/docs/pricing-rate-limits>

## Outcome

Make Muse Spark 1.3 the safe, current default for SASE's Muse provider, expose both 1.3
variants everywhere provider metadata feeds the UI and editor integrations, retain the
still-supported older model IDs for explicit selection, and preserve SASE's rule that no
automatic route opts a user into Meta's Contributor data terms.

## Implementation

1. Update `src/sase/llm_provider/muse.py` so both `large` and `small` tiers resolve to
   `muse-spark-1.3`. Keep the two tiers on the standard model rather than a Contributor
   model; Meta charges the same standard price for 1.3, 1.2, and 1.1, and the small tier
   must not silently authorize training on user code.
2. Publish the full current Muse Spark catalog in newest-first order: `muse-spark-1.3`,
   `muse-spark-1.3-contributor`, `muse-spark-1.2`, `muse-spark-1.2-contributor`, and
   `muse-spark-1.1`. Add unambiguous display/filter shorthands `spark13` and `spark13c`
   while retaining the existing 1.2 and 1.1 shorthands. This provider metadata is the
   source of truth for implicit provider resolution, the TUI picker, `%model`/`%m`
   completion, `==model` completion, and the xprompt LSP catalog; do not add a second
   hard-coded catalog in those consumers.
3. Generalize the Muse Contributor advisory to cover both 1.3 and 1.2 Contributor IDs.
   Keep the warning severity and `trains on your data` label, but make each detail
   identify its matching standard model and accurately describe Meta's current data use
   and discounted/rate-limited tier. Ensure neither advisory model can appear in a tier
   map or other built-in automatic route.
4. Align Muse's top reasoning-effort spelling with the current CLI while updating the
   model generation. Muse Code now accepts `--reasoning-effort max` and Meta documents
   max reasoning for standard `muse-spark-1.3`; stop rewriting SASE's canonical `max` to
   the older `ultra` spelling. Preserve the existing explicit-versus-default error
   contract, and document that Meta limits max reasoning to standard 1.3 rather than
   claiming it for Contributor or older models. Account for the shared
   `invocation_option_args()` path used by headless and tmux-agent launches so their
   argv expectations remain consistent.
5. Refresh current user-facing examples and reference material that present 1.2 as the
   recommended Muse choice, including `README.md`, `docs/getting_started.md`,
   `docs/agent_providers.md`, `docs/llms.md`, `docs/xprompt.md`,
   `docs/configuration.md`, the default-config grammar example, and maintained blog
   examples. The Muse integration reference should show the complete current catalog,
   1.3 tier mapping, `spark13`/`spark13c` shorthands, both Contributor advisories,
   current prices, and the model-specific max-effort constraint. Retain 1.2 references
   that intentionally test explicit backward-compatible selection or preserve a
   release-keyed historical Muse event-stream capture.

## Tests and verification

1. Extend `tests/llm_provider/test_muse_provider_core.py` to pin the new tier default,
   full known-model ordering, short aliases, implicit 1.3 provider resolution, explicit
   selection of older models, and `max` CLI argument behavior.
2. Parameterize the Contributor coverage in
   `tests/llm_provider/test_model_advisories.py` and the doctor/model-picker/completion
   tests so both Contributor generations carry warnings, both standard generations do
   not, and no advisory model is automatically selected. Include a direct assertion that
   the generated `%model`/LSP catalog contains both 1.3 model IDs with the correct
   metadata.
3. Update documentation assertions and tmux-agent argv expectations affected by the
   recommended model and current `max` spelling. Do not rewrite the sanitized `R708.1`
   JSONL fixtures: their 1.2 identities are evidence from that historical release, not
   stale catalog configuration.
4. Run the focused provider, advisory, completion/LSP, doctor, tmux-agent, and
   documentation tests, then run the repository's required `just check` verification.

## Non-goals

- Do not auto-detect Muse from a generic `muse` executable; explicit provider selection
  remains unchanged.
- Do not route any built-in alias or tier to a Contributor model.
- Do not remove 1.2 or 1.1 while Meta still lists them as available.
- Do not add live network model discovery to completion construction in this change; the
  provider hook remains the deterministic catalog boundary used by every SASE frontend.
