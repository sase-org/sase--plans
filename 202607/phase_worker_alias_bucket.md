---
tier: tale
title: Size-specific phase-worker model aliases
goal: Route every implicit phase-agent launch through a size-specific alias while
  preserving one shared phase-worker fallback and presenting the related aliases as
  a built-in Models-panel bucket.
create_time: 2026-07-21 08:33:26
status: wip
prompt: 202607/prompts/phase_worker_alias_bucket.md
---

# Plan: Size-specific phase-worker model aliases

## Context and outcome

SASE currently exposes `@phase_worker` for small and medium phase agents and selects `@smartest` directly for large
phase agents. The Models panel displays `phase_worker` as an individual role row. Introduce `@small_phase_worker`,
`@medium_phase_worker`, and `@large_phase_worker` as first-class implicit builtin aliases, use them for their matching
phase sizes, and group them with the existing `@phase_worker` alias in an always-present `phase_worker` bucket.

Keep `@phase_worker` itself unchanged: when it is not configured, it falls back to `@default`. Each new size alias must
fall back to `@phase_worker`, so existing persistent, launch-scoped, and temporary `phase_worker` overrides continue to
govern all phase sizes unless a size-specific alias is overridden. Preserve `@smartest` as a resolvable/configurable
alias for compatibility and explicit use, but stop selecting it automatically for large phases; users who want that
relationship can configure `large_phase_worker: "@smartest"`.

## Alias policy and Models-panel bucket

- Extend the centralized implicit alias policy in `src/sase/llm_provider/config.py` with constants, descriptions,
  builtin classification, and fallbacks for all three size aliases. Keep alias resolution flat and retain the existing
  cycle/depth and override precedence behavior.
- Define a built-in `phase_worker` bucket alongside the existing `coders` bucket in
  `src/sase/llm_provider/alias_view.py`. Its members should be ordered as the generic `phase_worker` alias followed by
  `small_phase_worker`, `medium_phase_worker`, and `large_phase_worker`; the bucket should occupy the former top-level
  `phase_worker` position while `smartest` remains a separate role row.
- Give the bucket a useful default description, allow `model_aliases.buckets.phase_worker.description` to override it,
  and coalesce custom aliases tagged with `bucket: phase_worker` just as custom coder aliases can join `coders`.
  Generalize shared builtin-bucket/doctor handling where that avoids one-off logic, without changing config paths or the
  independent edit/reset/temporary-override behavior of member aliases.
- Update exports, Models-panel navigation/selection fixtures, row ordering assertions, descriptions, and visual
  snapshots so the collapsed and drilled-in bucket states are covered. Ensure bucket metadata for `phase_worker` is
  recognized as valid even when no custom alias references it.

## Size-derived phase launch routing

- Update the authoritative phase model selector in `src/sase/bead/work.py` so an explicit phase model still wins, while
  implicit routing maps normalized `small`, `medium`, and `large` sizes to `@small_phase_worker`,
  `@medium_phase_worker`, and `@large_phase_worker`, respectively. Missing legacy size metadata must continue to
  normalize to `small`.
- Keep planning behavior independent from model routing: small phases launch directly, medium and large phases still
  append `#plan`, and epic land-agent selection remains unchanged.
- Because actual multi-prompt rendering and plan-file previews already share the selector, update their expectations
  together and verify dry-run output, persisted phase sizes, resumptions, and explicit-model precedence all report and
  emit the same aliases.

## Configuration, completion, and user-facing contract

- Add the new builtin names everywhere aliases are enumerated or described: `%model` completion catalogs, picker/alias
  dependency views, default configuration examples, JSON schema descriptions, doctor diagnostics, and model-alias tests.
  Verify that configured per-size targets shadow their fallback, while an override on `phase_worker` is inherited
  through every unconfigured size alias and a size-specific override affects only that alias.
- Document both built-in buckets and the new fallback chain in `docs/configuration.md`, `docs/llms.md`, and
  `docs/ace.md`. Update phase-routing descriptions and examples in `docs/beads.md` and `docs/sdd.md`, plus any affected
  alias lists/examples in `docs/xprompt.md`. Clearly distinguish the generic shared fallback from the aliases actually
  emitted for each phase size and explain the changed role of `@smartest`.
- Update the canonical generated-skill source `src/sase/xprompts/skills/sase_plan.md` so future plans describe the new
  size-specific routing. Adjust its source-contract tests, then run `sase skill init --force` and `chezmoi apply` as
  required by the generated-skills workflow; do not edit installed/generated `SKILL.md` files directly.

## Verification

- Expand focused unit coverage for implicit alias membership, fallback chains, configured and temporary/launch-scoped
  override precedence, completion order/descriptions, builtin bucket aggregation/custom coalescing, doctor bucket
  recognition, and schema acceptance.
- Update bead renderer, CLI dry-run, and plan preview tests to assert the exact alias for each size, the legacy-small
  fallback, unchanged `#plan` behavior, and explicit-model precedence. Retain coverage showing `@smartest` still
  resolves even though no implicit phase launch selects it.
- Refresh only the intentional Models-panel PNG goldens and run the dedicated visual snapshot suite to verify both the
  new collapsed `phase_worker` bucket and its drilled-in member order/state.
- Run `just install` before repository checks, execute the relevant focused test modules while iterating, and finish
  with `just check` so linting, typing, the full test suite, and visual snapshots pass together.
