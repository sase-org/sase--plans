---
tier: tale
title: Add Gemini 3.7 Flash and make it Antigravity's default
goal:
  SASE offers every Gemini 3.7 Flash Antigravity variant in its model completion
  surfaces, resolves Antigravity tier defaults to the new family, and Bryan's personal
  Antigravity launch presets use Gemini 3.7 Flash High consistently.
size: medium
proposed_by: bbugyi200.athena.01w
create_time: 2026-08-14 18:57:23
status: wip
---

- **PROMPT:**
  [prompts/202608/gemini_37_flash_antigravity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gemini_37_flash_antigravity.md)

# Add Gemini 3.7 Flash and make it Antigravity's default

## Objective

Refresh SASE's static Antigravity (`agy`) model metadata for Google's August 13, 2026
Gemini 3.7 Flash rollout, make the new family the provider's tier defaults, and update
Bryan's chezmoi-managed Antigravity presets and interactive launcher to select Gemini
3.7 Flash High by default.

The completed change must expose all three Antigravity reasoning variants in both the
ACE model picker and `%model` completion (including the editor LSP catalog), while
keeping older models available and avoiding a second model list in presentation code.

## Current state and verified identifiers

- The installed Antigravity CLI is version `1.1.13`. On August 14, 2026,
  `agy --output-format json models` reports this ordered 14-model catalog:
  1. `gemini-3.7-flash-high`
  2. `gemini-3.7-flash-medium`
  3. `gemini-3.7-flash-low`
  4. `gemini-3.6-flash-high`
  5. `gemini-3.6-flash-medium`
  6. `gemini-3.6-flash-low`
  7. `gemini-3.5-flash-high`
  8. `gemini-3.5-flash-medium`
  9. `gemini-3.5-flash-low`
  10. `gemini-3.1-pro-high`
  11. `gemini-3.1-pro-low`
  12. `claude-sonnet-4-6`
  13. `claude-opus-4-6-thinking`
  14. `gpt-oss-120b-medium`

  The first three IDs and labels are respectively Gemini 3.7 Flash High, Medium, and
  Low. These stable CLI slugs, not a Gemini API model ID or the human-readable labels,
  are the values SASE must pass to `agy --model`.

- `src/sase/llm_provider/agy.py` is the canonical Antigravity registry. Its
  `llm_known_model_names()` output feeds `model_to_provider_map()`, and that one map
  supplies both `build_model_rows()` for the ACE model picker and
  `build_model_completion_catalog()` for ACE/editor `%model` completion. The Rust
  xprompt LSP receives the resulting catalog as launch-time JSON, so no Rust or
  picker-specific catalog edit is required.
- `_TIER_TO_MODEL` currently maps `large` to `gemini-3.6-flash-high` and `small` to
  `gemini-3.6-flash-low`. These values are also reported by doctor metadata and used
  when SASE invokes Antigravity without an explicit model.
- Bryan's chezmoi `home/dot_config/sase/sase.yml` currently treats `#m_agy` as a
  second-best preset (`gemini-3.5-flash-high`) and `#m_agy_pro` as the best preset
  (`gemini-3.6-flash-high`). The new request intentionally changes `#m_agy` to mean the
  current Antigravity default. `#m_flash` recursively aliases `#m_agy` and will inherit
  that change.
- The chezmoi `home/bin/executable_tmux_ai_window` separately pins
  `gemini-3.6-flash-high` in the command used for a directly launched Antigravity
  window, so it is another active default surface that must move with `#m_agy`.

## Design decisions

1. Keep Antigravity discovery static at runtime. Copy the verified `agy models` snapshot
   into provider metadata in CLI order; do not invoke a network-backed CLI command while
   opening a picker, completing a directive, or starting ACE.
2. Preserve 3.6, 3.5, 3.1, Claude, and GPT-OSS entries. Adding a new default is not a
   deprecation, and existing explicit prompts must keep resolving.
3. Map `large` to `gemini-3.7-flash-high` and `small` to `gemini-3.7-flash-low`,
   preserving SASE's existing high-for-large and low-for-small convention. The medium
   variant belongs in the selectable catalog but is not a tier default because SASE
   exposes only `large` and `small` provider tiers here.
4. Use `flash37h`, `flash37m`, and `flash37l` as compact aliases. They follow the
   established `flash36*`/`flash35*` convention and keep multi-model agent suffixes
   concise and unambiguous.
5. Point both `#m_agy` and the stable `#m_agy_pro` name at 3.7 High. Their convergence
   is intentional: `#m_agy` now means the requested provider default, while `#m_agy_pro`
   continues to mean the best current Antigravity coding preset. Do not rename or remove
   either personal xprompt as part of a model refresh.
6. Preserve the purpose of `#m_agy_pro_flash` as a best-versus-next-family fan-out by
   changing it to 3.7 High versus 3.6 High. Change `#m_swarm` and the tmux launcher to
   3.7 High so every surface that previously followed `#m_agy` still follows its new
   target.
7. Update references only when they enumerate the live catalog, assert the provider
   default, or present a current recommended launch. Keep older-but-valid 3.6/3.5
   literals in parser, fan-out, badge, and compatibility tests whose purpose is to prove
   arbitrary known model routing rather than advertise the current default.

## Implementation

### 1. Refresh the canonical Antigravity registry

In `src/sase/llm_provider/agy.py`:

- change `_TIER_TO_MODEL` to 3.7 Flash High/Low for `large`/`small`;
- prepend the three exact 3.7 stable slugs to `llm_known_model_names()`, retaining all
  11 existing entries and exact `agy models` ordering; and
- prepend matching `flash37h`/`flash37m`/`flash37l` entries to
  `llm_model_short_aliases()`.

Do not add a second list to the TUI, completion code, or Rust core. Confirm through
tests that the registry propagation already makes the new models available as bare model
rows and provider-qualified insertions such as `agy/gemini-3.7-flash-high`.

### 2. Pin registry, routing, picker, and completion behavior with tests

Update the narrowly affected SASE tests, including:

- `tests/llm_provider/test_agy_provider_core.py`: exact 14-entry catalog and alias map,
  large/small resolution, default `agy --model` arguments, and bare/qualified 3.7
  provider inference;
- `tests/test_agy_integration_polish.py` and `tests/doctor/test_checks_providers.py`:
  aggregated aliases, picker presence, and reported large/small defaults;
- `tests/test_llm_provider_core.py`: aggregated provider resolution and short aliases
  for the new family;
- `tests/test_model_picker_options.py` and `tests/test_model_picker_jump.py`: 3.7
  inclusion and the new first rows under the existing Gemini filter; and
- `tests/test_xprompt_model_completion.py`: an explicit real-registry assertion that all
  three 3.7 slugs appear as Antigravity model rows with the expected aliases, so
  completion coverage does not rely only on a model-picker assertion.

Update `tests/test_xprompt_swarm_local_helpers.py` if the bundled `reads` preset changes
below. Avoid broad mechanical replacement in unrelated tests: 3.6 and 3.5 remain valid
catalog members and useful non-default routing fixtures.

### 3. Refresh live SASE documentation, examples, and bundled presets

Review the active non-historical 3.6 references and update those that describe the
catalog/default or launch the recommended Antigravity model:

- the Antigravity tier, known-model, and short-alias tables and current launch examples
  in `docs/llms.md`;
- the stable-slug example in `docs/xprompt.md` and the current model example in the SASE
  overview blog post;
- `sase/xprompts/reads.md`, which hardcodes the current high-quality Antigravity model;
- the Antigravity entry/alias in `demos/scripts/seed_sase_ace_demo`; and
- source docstring examples that specifically illustrate the Antigravity default.

Keep historical `CHANGELOG.md` entries and examples whose teaching purpose is a
multi-generation fan-out unchanged. Do not edit SASE memory files or generated provider
instruction shims.

### 4. Update chezmoi-managed Antigravity defaults

The implementing agent must first open the linked repo with `/sase_repo` and use the
printed checkout path for all reads and writes:

```bash
sase repo open chezmoi -r "Update Bryan's Antigravity defaults for Gemini 3.7 Flash"
```

In `home/dot_config/sase/sase.yml`:

- set `m_agy` to `%model:agy/gemini-3.7-flash-high`;
- set `m_agy_pro` to the same 3.7 High target;
- set `m_agy_pro_flash` to fan out across 3.7 High and 3.6 High;
- set the Antigravity branch of `m_swarm` to 3.7 High; and
- leave `m_flash: "#m_agy"` unchanged so it inherits the new target.

These xprompts are plain definitions, not entries marked `snippet: true`, so the nested
chezmoi instruction requiring LuaSnip counterparts does not require a `_snip_utils.lua`
change.

In `home/bin/executable_tmux_ai_window`, update the `agy` branch of `provider_command()`
to pass `--model gemini-3.7-flash-high`. Add a focused bashunit test in
`tests/bash/tmux_ai_window_test.sh` that launches the stubbed `agy` provider and asserts
the exact new model argument, preventing this independent default from drifting again.

## Validation

### SASE repository

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run focused pytest coverage for the Antigravity provider, registry/integration,
   doctor metadata, model picker, `%model` completion, and bundled `reads` xprompt tests
   changed above.
3. Run `agy --output-format json models`; parse `command.data.models` and compare its
   IDs in order with `AgyProvider.llm_known_model_names()`. Confirm both tier defaults
   occur in the returned list and that no existing entry was lost.
4. Inspect `model_completion_catalog_payload()` (or exercise it through its tests) and
   confirm `gemini-3.7-flash-{high,medium,low}` have provider `agy`, aliases
   `flash37{h,m,l}`, and provider-qualified completion under the `agy/` scope. Confirm
   the model picker exposes the same three IDs without any UI-local registry edits.
5. Run `sase doctor -C llm.registry -C llm.default -v` against the updated installed
   package and confirm the reported Antigravity large/small resolutions are 3.7
   High/Low.
6. Run the mandatory repository gate, `just check`. This change does not touch the
   broadening set and does not require `just check-full` unless the scoped gate itself
   escalates or reports unusual selection.

### Chezmoi repository and deployed configuration

1. Parse `home/dot_config/sase/sase.yml` and run the focused bash launcher test, then
   run the chezmoi repository's full `just check` gate.
2. Inspect the source diff and `chezmoi diff` to verify the xprompt and launcher changes
   are the only intended deployed differences.
3. After the normal SASE finalizer commits the linked-repo changes, follow its root
   instruction and run `chezmoi update -a --force` so the source is applied to the live
   home configuration.
4. Against that applied configuration, run
   `sase doctor -C config.model_xprompts -C config.xprompt_definitions --strict`, and
   use `sase xprompt expand`/`show` to confirm:
   - `#m_agy`, `#m_agy_pro`, and `#m_flash` resolve to 3.7 High;
   - `#m_agy_pro_flash` produces distinct 3.7 High and 3.6 High branches; and
   - the Antigravity branch of `#m_swarm` resolves to 3.7 High without an unknown-model
     or provider fallback warning.

## Acceptance criteria

- SASE's Antigravity registry exactly matches the current 14 IDs from `agy models` in
  order, including all three Gemini 3.7 Flash variants and all previously supported
  entries.
- Gemini 3.7 Flash High, Medium, and Low appear in the ACE model picker and `%model`
  completion, including provider-scoped `agy/` completion, with compact
  `flash37h`/`flash37m`/`flash37l` aliases.
- Unqualified Antigravity large/small launches resolve to and pass 3.7 High/Low exactly;
  doctor reports the same defaults.
- Bryan's `#m_agy`, `#m_agy_pro`, `#m_flash`, `#m_agy_pro_flash`, `#m_swarm`, and tmux
  Antigravity launcher follow the mappings specified above, with a regression test for
  the independent tmux command.
- Focused tests, live-catalog parity, SASE doctor checks, the SASE `just check`, and the
  chezmoi `just check` all pass; the applied home xprompts expand without routing
  diagnostics.

## Out of scope

- Removing or adding advisories to older Antigravity models.
- Dynamically querying `agy models` in a UI or at SASE startup.
- Adding Gemini 3.7 Flash to non-Antigravity providers such as OpenCode or to the Gemini
  API, whose identifiers and rollout contracts are separate.
- Renaming or deleting Bryan's stable personal xprompt names merely because two now
  select the same best model.
- Changing `%model` grammar, completion rendering, picker layout, Rust LSP code, or PNG
  visual snapshots unless verification reveals a real regression in those generic
  layers.
- Rewriting historical changelog entries or editing generated SASE memory/instruction
  files.
