---
create_time: 2026-07-11 10:06:35
status: done
prompt: 202607/prompts/coders_model_alias_bucket.md
tier: tale
---
# Consolidate coder model aliases into the `coders` bucket

## Context

The ACE Models panel currently renders the generic `coder` role alias and every registered `<provider>_coder` alias as
independent top-level rows. Recent bucket support added a display-only `BucketView`, top-level bucket folding, drill-in
navigation, aggregate model/override summaries, and custom bucket configuration. The coder aliases are a natural family:
an unconfigured `<provider>_coder` resolves through `@coder`, while explicit per-provider configuration or temporary
overrides can make individual members diverge.

This change should use that existing bucket UI to reduce top-level noise without changing model-alias resolution,
configuration paths, override precedence, prompt completion, or coder handoff behavior.

## Product behavior

- Replace the top-level `coder` and `<provider>_coder` rows with one always-present, collapsed bucket named `coders`.
- Keep `default` as the first Models-panel row and put `coders` immediately after it, in the position currently occupied
  by the generic `coder` role. Keep `epic_creator`, `epic_lander`, and `phase_worker` after the bucket, followed by
  custom buckets and ungrouped custom aliases in their existing deterministic order.
- When opened, order members as `coder` first, then all registered `<provider>_coder` aliases alphabetically. Preserve
  each member's existing kind badge, configured/implicit provenance, effective provider/model, description, persistent
  edit/reset path, and temporary override actions.
- Give the built-in bucket a concise fixed fallback description explaining that it contains the generic coder default
  and planner-provider-specific coder follow-up aliases. Continue to show the existing aggregate effective-model mix and
  active-override count so divergences within the family remain visible while collapsed.
- Treat `coders` as a built-in display bucket, but coalesce any custom aliases configured with `bucket: coders` into the
  same bucket rather than producing duplicate bucket rows/IDs or hiding one group. A configured
  `model_aliases.buckets.coders.description` may override the fallback description, and diagnostics should recognize the
  built-in coder members so that metadata for `coders` is not reported as dangling.
- Keep bucket membership presentation-only. `@coder` and `@<provider>_coder` remain separate, flat aliases everywhere
  outside the Models panel, including resolution chains, `%model` completion, launch routing, state-file keys, and
  persistent `llm_provider.model_aliases.builtin.<alias>` edits.

## Implementation plan

1. Extend the display aggregation policy in `src/sase/llm_provider/alias_view.py`.
   - Define the built-in `coders` bucket identity and fallback description alongside the existing canonical alias-order
     policy.
   - Generalize `BucketView` documentation and the row-folding helper so it can contain builtin and custom aliases, not
     only custom rows.
   - Extract `coder` plus every `provider_coder` view from the normal top-level sequence, merge any custom `coders`
     members, and build exactly one `BucketView` in the canonical second-row position.
   - Preserve deterministic member order using alias semantics (`coder`, provider-specific aliases, then any coalesced
     custom members) and retain existing alphabetical handling for all other custom buckets/aliases.
   - Resolve the bucket description from configured metadata when present, with the built-in explanation as the
     fallback. Do not add filesystem work or other blocking behavior to panel navigation or highlight handlers; folding
     remains a pure in-memory transformation performed during the panel's existing refresh path.

2. Align configuration metadata and diagnostics with the built-in bucket in
   `src/sase/doctor/checks_config_model_aliases.py`, `src/sase/config/sase.schema.json`, and
   `src/sase/default_config.yml`.
   - Make the dangling-bucket check count `coders` as populated by built-in aliases even when no custom alias references
     it, while retaining warnings for genuinely unused custom bucket metadata.
   - Clarify schema/default-config prose that bucket metadata is display-only, that `coders` is supplied by ACE, and
     that custom `bucket: coders` membership coalesces with it.
   - Avoid any config migration or required new setting: existing configurations remain valid and the bucket appears
     automatically.

3. Reuse the existing Models-panel navigation/rendering paths and adjust only where the new built-in row exposes an
   assumption in `src/sase/ace/tui/modals/models_panel_display.py` or
   `src/sase/ace/tui/modals/models_panel_rendering.py`.
   - Verify open/back navigation, contextual footer/title/description, highlight restoration after edit/override/clear,
     and bucket action guards all work unchanged for `coders`.
   - Ensure a refresh while inside `coders` retains the selected alias and that the always-present generic `coder`
     member prevents the panel from spuriously leaving the bucket.
   - Keep aggregate rendering literal and width-safe with the existing `BucketView` renderer; only add a focused
     rendering adjustment if the built-in description or mixed member kinds require it.

4. Add regression coverage around aggregation, configuration, and interaction.
   - In `tests/llm_provider/test_alias_view.py`, assert the exact top-level order, absence of standalone coder-family
     rows, exact bucket membership/order across a pinned provider registry, configured and implicit members, model
     summaries, active override counts, and coalescing of a custom `coders` bucket without duplicate rows.
   - In `tests/doctor/test_checks_config_model_aliases.py` and `tests/test_config_schema.py`, cover accepted `coders`
     metadata without custom members, retained warnings for other empty buckets, and documented custom coalescing
     config.
   - In `tests/test_models_panel.py`, exercise entering/leaving the built-in bucket, selecting and acting on the generic
     and provider-specific aliases, cursor restoration, description/footer state, and refresh behavior from inside the
     bucket.
   - Update the Models-panel PNG fixtures/tests in `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py` and its
     snapshot directory so the default and override states show the collapsed `coders` row, and pin an opened `coders`
     view that makes member ordering, state tags, and aggregate override reporting visually reviewable. Retain the
     existing custom-bucket snapshots to guard the generic bucket path.

5. Update user-facing Models-panel documentation in `docs/ace.md` and the relevant alias/configuration references in
   `docs/llms.md` and `docs/configuration.md`.
   - Describe the top-level ordering and `l`/Right/Enter to open a bucket plus `h`/Left to return.
   - Explain the automatic `coders` grouping, aggregate summary, optional description metadata, and the fact that
     aliases remain independently editable/overridable inside the bucket.
   - Keep resolution and role-alias documentation unchanged except for cross-referencing the display grouping.

6. Verify the completed implementation.
   - Run focused alias-view, doctor/schema, Models-panel interaction, keymap, and visual snapshot tests while iterating.
   - Inspect generated actual/expected/diff PNG artifacts and update goldens only for the intentional coder-bucket
     layout.
   - Run `just install` first as required for an ephemeral workspace, then finish with the repository-mandated
     `just check`.

## Acceptance criteria

- The Models panel has exactly one top-level `coders` bucket and no standalone `coder` or `<provider>_coder` rows.
- Opening `coders` exposes every coder-family alias in deterministic order, with all existing alias actions and state
  indicators working independently.
- The collapsed row accurately reports total members, distinct effective models, and active member overrides.
- Existing alias resolution, plan-to-coder routing, completions, config edit paths, and temporary override persistence
  are unchanged.
- A custom `bucket: coders` configuration yields one coalesced bucket; `coders` metadata is not diagnosed as dangling,
  while unrelated empty bucket metadata still is.
- Automated unit, interaction, schema/doctor, visual snapshot, and full repository checks pass.
