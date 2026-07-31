---
tier: tale
title: Refresh Antigravity's Gemini model catalog and coding presets
goal: SASE exposes the current Antigravity model catalog, and Bryan's Gemini presets select the top two coding models.
proposed_by: bbugyi200.athena.qk
create_time: 2026-07-31 13:19:18
status: done
---

- **PROMPT:** [202607/prompts/refresh_antigravity_gemini_models.md](prompts/refresh_antigravity_gemini_models.md)
- **AGENTS:**
  - [bbugyi200.athena.qk](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.qk.md)
- **COMMITS:**
  - [88f57b1](https://github.com/bbugyi200/dotfiles/commit/88f57b1504550fd0158dc5cb8498c2af4c511f62) — fix(sase): update
    Antigravity model presets

# Refresh Antigravity's Gemini model catalog and coding presets

## Objective

Bring SASE's Antigravity (`agy`) provider metadata and model pickers into sync with the current Antigravity CLI catalog,
then update Bryan's chezmoi-managed model xprompts so `#m_agy` selects the second-best currently available Gemini model
for coding and `#m_agy_pro` selects the best.

## Current-state findings and decisions

- On 2026-07-31, the installed Antigravity CLI reports version 1.1.8. Its `agy models` command returns these stable
  model slugs, in order:
  1. `gemini-3.6-flash-high`
  2. `gemini-3.6-flash-medium`
  3. `gemini-3.6-flash-low`
  4. `gemini-3.5-flash-high`
  5. `gemini-3.5-flash-medium`
  6. `gemini-3.5-flash-low`
  7. `gemini-3.1-pro-high`
  8. `gemini-3.1-pro-low`
  9. `claude-sonnet-4-6`
  10. `claude-opus-4-6-thinking`
  11. `gpt-oss-120b-medium`

- The `agy changelog` says version 1.1.5 introduced stable, user-facing model slugs that appear in the model picker and
  are accepted by `--model`. SASE still declares Antigravity CLI 1.0.10's older display labels such as
  `Gemini 3.5 Flash (High)`, so its picker catalog, default tiers, aliases, examples, and several configured launches
  are stale.
- Google's current Gemini 3.6 Flash evaluation ranks the available Gemini families consistently for coding:
  `3.6 Flash > 3.5 Flash > 3.1 Pro` on SWE-Bench Pro, DeepSWE v1.1, Terminal-Bench 2.1, and MLE-Bench. Therefore use the
  high-reasoning variants and map the requested personal presets as follows:
  - `#m_agy` (second-best for coding) -> `agy/gemini-3.5-flash-high`
  - `#m_agy_pro` (best for coding) -> `agy/gemini-3.6-flash-high`

  The `m_agy_pro` name remains a stable personal preset name even though its best-current target is now a Flash-family
  model. Evidence: <https://deepmind.google/models/gemini/flash/> and
  <https://ai.google.dev/gemini-api/docs/models/gemini-3.6-flash>.

- Keep the provider catalog static and ordered from the CLI snapshot, matching the existing provider-metadata
  architecture. Do not invoke `agy models` while rendering a picker or refreshing TUI state.

## Implementation

### 1. Migrate the bundled Antigravity provider to current model slugs

Update `src/sase/llm_provider/agy.py` so all data sent to or advertised for Antigravity uses the exact current
`agy models` slugs:

- Change the `large` and `small` tier defaults to `gemini-3.6-flash-high` and `gemini-3.6-flash-low`, respectively, so
  SASE defaults resolve to valid models in the newest family.
- Replace `llm_known_model_names()` with the complete 11-entry catalog above, preserving the CLI's ordering so picker
  rows are deterministic and easy to compare with `agy models`.
- Replace the old display-name alias keys with slug keys. Retain recognizable compact suffixes for existing families
  (`flash35h`, `flash35m`, `flash35l`, `pro31h`, `pro31l`, `opus46t`, and `gptoss120m`) and add
  `flash36h`/`flash36m`/`flash36l` plus an accurate alias for `claude-sonnet-4-6` (for example `sonnet46`).
- Rewrite comments and docstrings that claim Antigravity uses space-and-parenthesis display names or cite CLI 1.0.10. Do
  not mix in the separate Antigravity effort/output-format integration work introduced after 1.0.10; that is outside
  this model-catalog task.

The picker itself should require no special-case changes: `build_model_rows()` already consumes the provider registry's
known models and short aliases. Confirm this remains true and avoid duplicating the catalog in TUI code.

### 2. Update regression coverage and active documentation/examples

Update Antigravity-specific assertions and fixtures to validate the new canonical slugs and tier mappings, with focused
coverage for:

- the complete provider catalog and short-alias map;
- `large`/`small` resolution and the exact value passed to `agy --model`;
- provider inference and explicit `agy/<model>` routing;
- picker inclusion, Gemini filtering, visible ordering/jump hints, and model-row aliases;
- doctor metadata for the Antigravity tiers; and
- xprompt/model-completion and integration paths that currently advertise an old Antigravity model.

At minimum, review and update the directly affected tests under `tests/llm_provider/test_agy_provider_core.py`,
`tests/test_agy_integration_polish.py`, `tests/test_llm_provider_core.py`,
`tests/llm_provider/test_registry_resolution.py`, `tests/test_model_picker_options.py`,
`tests/test_model_picker_modal.py`, `tests/test_model_picker_jump.py`, `tests/doctor/`,
`tests/test_xprompt_model_completion.py`, and `tests/test_gemini_active_surface_guard.py`. Migrate Antigravity
integration fixtures to real slugs; preserve generic quoted-model parser tests only when they are explicitly testing the
grammar rather than claiming that an obsolete label is available.

Refresh current user-facing material that enumerates, recommends, or launches Antigravity models, including the model
mapping/alias tables and examples in `docs/llms.md` and `docs/xprompt.md`, the bundled `sase/xprompts/reads.md` preset,
the active blog/demo examples, and source docstring examples. Do not rewrite historical entries in `CHANGELOG.md`.

### 3. Update the chezmoi-managed Antigravity launch presets

Open the linked `chezmoi` repository through `sase repo open chezmoi` and edit `home/dot_config/sase/sase.yml`:

- Set `m_agy` to `%model:agy/gemini-3.5-flash-high`.
- Set `m_agy_pro` to `%model:agy/gemini-3.6-flash-high`.
- Replace the duplicated obsolete literals in `m_agy_pro_flash` with the same best/second-best pair, and replace the
  obsolete Antigravity target in `m_swarm` with the intended second-best preset target. Keep the existing xprompt names
  and fan-out behavior stable.

Also update `home/bin/executable_tmux_ai_window`, whose printed Antigravity launch command still uses the obsolete
`Gemini 3.5 Flash (High)` label, to a valid current slug consistent with its intended high-quality default.

The directory-specific chezmoi instructions require LuaSnip counterparts only for xprompts marked as snippets. These
model xprompts are not snippet definitions, so `_snip_utils.lua` should not change. Do not edit generated SASE memory or
provider instruction files.

## Validation

1. Before running repository checks, run `just install` in the SASE workspace as required for an ephemeral workspace.
2. Run focused tests for the provider catalog/resolution, picker rows and filtering/jump behavior, doctor metadata,
   model completion, and Antigravity integration files changed above.
3. Run `agy models` and compare its output exactly with `AgyProvider.llm_known_model_names()`, including order; also
   confirm the resolved `large` and `small` models occur in that output.
4. Run `sase doctor -C llm.registry -C llm.default -v` against the updated installed SASE package and confirm the
   Antigravity catalog/tier metadata uses current slugs.
5. Validate the chezmoi YAML and expanded xprompts, then run
   `sase doctor -C config.model_xprompts -C config.xprompt_definitions --strict` once the chezmoi-managed config is the
   active config. Confirm `#m_agy` expands/routes to `agy/gemini-3.5-flash-high`, `#m_agy_pro` expands/routes to
   `agy/gemini-3.6-flash-high`, and the related fan-outs route to the intended two models without fallback warnings.
6. Inspect the chezmoi diff to ensure only the intended source-managed files changed. If the changes are committed by
   the normal SASE finalizer, follow the repository instruction to run `chezmoi update -a --force`, then repeat the
   config doctor checks against the applied home configuration.
7. Run the mandatory full SASE gate: `just check`.

## Acceptance criteria

- Every model returned by the current `agy models` command appears once under Antigravity in SASE's model pickers, in
  deterministic CLI order, with useful compact aliases; obsolete 1.0.10 display labels are no longer advertised as
  available models.
- SASE's Antigravity `large` and `small` defaults are valid current Gemini 3.6 Flash slugs and are passed unchanged to
  the CLI.
- Bryan's `#m_agy` and `#m_agy_pro` presets select Gemini 3.5 Flash High and Gemini 3.6 Flash High, respectively,
  matching the current official coding-quality ranking, and related checked-in launch helpers contain no obsolete model
  labels.
- Focused tests, doctor checks, xprompt/config validation, catalog parity checks, and `just check` all pass.
