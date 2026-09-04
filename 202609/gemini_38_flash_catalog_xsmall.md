---
tier: tale
title: Add Gemini 3.8 Flash to the Antigravity catalog and point @xsmall at it
goal:
  SASE offers all three Gemini 3.8 Flash Antigravity variants in its model picker and
  `%model` completion surfaces, and the shipped `@xsmall` pool selects
  `agy/gemini-3.8-flash-high` instead of the 3.7 slug.
size: medium
proposed_by: bbugyi200.apollo.g
create_time: 2026-09-04 11:03:44
status: wip
---

# Add Gemini 3.8 Flash to the Antigravity catalog and point @xsmall at it

## Objective

Google shipped Gemini 3.8 Flash and Antigravity now serves all three reasoning variants.
Refresh SASE's static Antigravity (`agy`) model metadata so the new family appears
everywhere SASE offers model completions — the ACE model picker, `%model` completion
(including the editor LSP catalog), and provider-scoped `agy/` completion — and swap the
Antigravity member of the shipped `@xsmall` load-balanced pool from Gemini 3.7 Flash
High to Gemini 3.8 Flash High.

Everything else about Antigravity stays put. In particular this change does **not** move
the provider's `large`/`small` tier defaults off 3.7 (see Out of scope).

## Current state and verified identifiers

- The Antigravity CLI installed on this host is version `1.1.26`. Run on 2026-09-04,
  `agy --output-format json models` returns this ordered 14-model catalog under
  `command.data.models`:
  1. `gemini-3.8-flash-high` — Gemini 3.8 Flash (High)
  2. `gemini-3.8-flash-medium` — Gemini 3.8 Flash (Medium)
  3. `gemini-3.8-flash-low` — Gemini 3.8 Flash (Low)
  4. `gemini-3.7-flash-high`
  5. `gemini-3.7-flash-medium`
  6. `gemini-3.7-flash-low`
  7. `gemini-3.6-flash-high`
  8. `gemini-3.6-flash-medium`
  9. `gemini-3.6-flash-low`
  10. `gemini-3.1-pro-high`
  11. `gemini-3.1-pro-low`
  12. `claude-sonnet-4-6`
  13. `claude-opus-4-6-thinking`
  14. `gpt-oss-120b-medium`

  These stable CLI slugs — not Gemini API model IDs and not the human-readable labels —
  are the values SASE passes to `agy --model`. Note that the plain `agy models` form
  prints only labels; use the JSON form above when verifying IDs.

- `src/sase/llm_provider/agy.py` is the canonical Antigravity registry.
  `llm_known_model_names()` feeds `model_to_provider_map()`, and that one map supplies
  both `build_model_rows()` for the ACE model picker and
  `build_model_completion_catalog()` for `%model` completion. The Rust xprompt LSP
  receives the resulting catalog as launch-time JSON. There is no second model list in
  presentation code, in the Rust core, or in the completion candidate catalog
  (`src/sase/completion/candidates/catalog_build.py` only enumerates the five size
  aliases), so a registry edit is sufficient to reach every completion surface.

- `src/sase/llm_provider/model_alias_defaults.yml` is the single edit point for shipped
  size-alias targets. Today `xsmall.target` is
  `claude/sonnet@medium | codex/gpt-5.5@medium | grok/grok-4.6@medium | agy/gemini-3.7-flash-high`.
  Antigravity is the last member of that round-robin pool and participates only when the
  `agy` CLI (or `SASE_AGY_PATH`) is available.

- `tests/llm_provider/test_model_alias_defaults.py::test_every_shipped_selector_member_names_a_registered_provider_model`
  asserts every shipped selector member names a model registered to that provider. The
  registry edit therefore has to land together with (or before) the YAML edit; changing
  the YAML alone fails this guard.

- `tests/_model_alias_defaults_fixture.py` deliberately freezes its own alias values so
  behavioral tests are insulated from shipped-value churn. Only the alias _graph shape_
  is a contract there, and this change does not alter it.

## Design decisions

1. Keep Antigravity discovery static at runtime. Copy the verified `agy models` snapshot
   into provider metadata in CLI order; never invoke a network-backed CLI command while
   opening a picker, completing a directive, or starting ACE.
2. Prepend the three 3.8 slugs so SASE's list keeps matching `agy models` ordering,
   newest family first. This is what makes 3.8 High and 3.8 Medium the first two rows
   under a `gemini` filter in the model picker.
3. Use `flash38h`, `flash38m`, and `flash38l` as compact short aliases, following the
   established `flash37*`/`flash36*` convention.
4. Retain every existing catalog entry. Adding a new family is not a deprecation, and
   existing explicit prompts such as `%m:agy/gemini-3.7-flash-high` must keep resolving.
5. Point `@xsmall`'s Antigravity member at `gemini-3.8-flash-high` and add no trailing
   `@<effort>` suffix. Antigravity encodes reasoning level in the stable model slug and
   exposes no independent effort mechanism, so a suffix would only raise an
   unsupported-effort warning. This preserves the deliberate per-provider asymmetry the
   pool already has: medium effort for Claude/Codex/Grok, a bare high slug for
   Antigravity.
6. Keep the Antigravity member last in the pool and leave the other four size aliases
   untouched.
7. Leave `_TIER_TO_MODEL` on 3.7 High/Low. Nothing in the alias change depends on it —
   `@xsmall` names a fully qualified `agy/<slug>` target — so tier defaults, the docs
   tier table, and every doc/demo/xprompt example that illustrates "the current
   Antigravity default" stay internally consistent at 3.7 with no edits.

## Implementation

### 1. Refresh the canonical Antigravity registry

In `src/sase/llm_provider/agy.py`:

- prepend `gemini-3.8-flash-high`, `gemini-3.8-flash-medium`, and `gemini-3.8-flash-low`
  to `llm_known_model_names()`, retaining all 14 existing entries in their current
  order; and
- prepend matching `gemini-3.8-flash-high: flash38h`,
  `gemini-3.8-flash-medium: flash38m`, and `gemini-3.8-flash-low: flash38l` entries to
  `llm_model_short_aliases()`.

Leave `_TIER_TO_MODEL` unchanged. Do not add a model list to the TUI, completion code,
or Rust core.

### 2. Repoint the shipped `@xsmall` pool

In `src/sase/llm_provider/model_alias_defaults.yml`, change the final member of
`xsmall.target` from `agy/gemini-3.7-flash-high` to `agy/gemini-3.8-flash-high`.
Preserve the three other members, their `@medium` effort suffixes, the pool ordering,
the single-pipe round-robin semantics, the alias description, and every other alias.

### 3. Update the narrowly affected tests

- `tests/llm_provider/test_agy_provider_core.py`: prepend the three 3.8 slugs to the
  `_AGY_MODELS` exact-catalog list and the three short aliases to the exact alias-map
  assertion in `test_agy_provider_metadata_hooks`. Leave the `resolve_model_name`
  large/small assertions on 3.7 High/Low, and leave the `--model` argv assertions
  unchanged.
- `tests/test_llm_provider_core.py`: add 3.8 coverage alongside the existing 3.7 lines —
  qualified and bare `resolve_model_provider("agy/gemini-3.8-flash-high")` /
  `("gemini-3.8-flash-high")`, and a `flash38h`/`flash38l` assertion in
  `test_model_short_alias_map_contains_agy_entries`.
- `tests/test_xprompt_model_completion_catalog.py`: add a
  `test_model_completion_catalog_includes_agy_gemini_38_flash_variants` case modeled on
  the existing 3.7 one, asserting all three 3.8 rows carry provider `agy` and the
  expected short aliases, and that they appear under the `agy/` provider-scoped filter.
  Keep the 3.7 test as-is.
- `tests/test_model_picker_options.py`: add `assert "gemini-3.8-flash-high" in ids` to
  `test_build_model_options_has_known_models`, keeping the 3.7 and 3.6 assertions.
- `tests/test_model_picker_jump.py`: this test **will** fail until updated. Under the
  `gemini` filter it asserts
  `visible_ids[:3] == ["__default__", "gemini-3.7-flash-high", "gemini-3.7-flash-medium"]`;
  change the two model IDs to the 3.8 High and 3.8 Medium slugs and refresh the adjacent
  comment.
- `tests/llm_provider/test_load_balanced_alias_defaults.py`: in the packaged-defaults
  parametrization, change the `@xsmall` expectation for the `agy/` prefix from
  `("agy/gemini-3.7-flash-high", None)` to `("agy/gemini-3.8-flash-high", None)`. The
  `None` effort is the point of that row — keep it.

Do not touch `tests/_model_alias_defaults_fixture.py` (frozen values, unchanged graph
shape), `tests/doctor/test_checks_providers.py` or
`tests/tmux_agent/test_shell_script_parity.py` (self-contained synthetic payloads that
pin 3.7 deliberately), or the ACE PNG visual fixtures (they use a synthetic `gemini`
provider, not the real `agy` registry). Avoid broad mechanical 3.7→3.8 replacement: 3.7
and 3.6 remain valid catalog members and useful non-default routing fixtures.

### 4. Refresh documentation

In `docs/llms.md`:

- run `just fmt-docs` to regenerate the `<!-- BEGIN GENERATED: model-alias-defaults -->`
  block from the YAML, and confirm the only changed cell is the `@xsmall` row's
  Antigravity member;
- hand-edit the Antigravity row of the "Automatic Provider Resolution" table to lead
  with the three 3.8 slugs;
- hand-edit the `agy` row of the model-shorthand table to lead with
  `gemini-3.8-flash-high` → `flash38h`, `gemini-3.8-flash-medium` → `flash38m`, and
  `gemini-3.8-flash-low` → `flash38l`; and
- hand-edit the Antigravity "Model Mapping" prose so the sentence about automatic
  `@xsmall` selection names `gemini-3.8-flash-high`.

Leave the Antigravity tier table (`large`/`small` → 3.7 High/Low) unchanged, and leave
the `%m:agy/gemini-3.6-flash-high` illustration alone — it demonstrates reaching a
non-default slug. Both hand-edited tables are prettier-formatted, so run `just fmt-md`
(or `just fmt`) afterwards to realign column widths.

No other documentation, demo, or bundled-xprompt file needs to change:
`docs/xprompt.md`, `docs/configuration.md`, `docs/getting_started.md`, the SASE overview
blog post, `sase/xprompts/reads.md`, `src/sase/default_config.yml`,
`src/sase/config/sase.schema.json`, and `demos/scripts/seed_sase_ace_demo` all reference
3.7 as the current Antigravity default or as an example, and the tier defaults are not
moving.

## Verification

1. Run `just install` first — this workspace is an ephemeral clone whose virtualenv may
   be stale.
2. Confirm the slugs against the live CLI before trusting the plan's snapshot:

   ```bash
   agy --output-format json models
   ```

   Compare `command.data.models[*].id` against `AgyProvider.llm_known_model_names()` and
   confirm the three 3.8 IDs are present and lead the list. If the live catalog no
   longer matches the snapshot above, stop and report rather than guessing at slugs.

3. Run the focused suites:

   ```bash
   just test \
     tests/llm_provider/test_agy_provider_core.py \
     tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_load_balanced_alias_defaults.py \
     tests/test_llm_provider_core.py \
     tests/test_agy_integration_polish.py \
     tests/test_xprompt_model_completion_catalog.py \
     tests/test_model_picker_options.py \
     tests/test_model_picker_jump.py \
     tests/test_model_picker_modal.py \
     tests/test_gemini_active_surface_guard.py
   ```

4. Rerun `just fmt-docs` and confirm it is idempotent (no further diff).
5. Run `sase doctor -C llm.registry -C llm.default -v` and confirm the Antigravity
   registry reports 17 models and that the large/small resolutions still read 3.7
   High/Low.
6. Run the mandatory repository gate, `just check`. This change does not touch the
   broadening set, so `just check-full` is only required if the scoped lane escalates or
   reports an unusual selection — and if it is needed, run it through `/sase_monitor`,
   never inline.
7. Review the final diff and confirm it is limited to the Antigravity registry, the
   shipped `@xsmall` target, the seven test files listed above, and `docs/llms.md`.

## Acceptance criteria

- `AgyProvider.llm_known_model_names()` returns 17 slugs whose first three are
  `gemini-3.8-flash-high`, `gemini-3.8-flash-medium`, and `gemini-3.8-flash-low`, in
  `agy models` order, with every previously supported entry retained.
- `llm_model_short_aliases()` maps those three slugs to `flash38h`, `flash38m`, and
  `flash38l`.
- All three variants appear in the ACE model picker and in `%model` completion,
  including provider-scoped `agy/` completion, with no picker-, completion-, or
  Rust-local catalog edit.
- The shipped `@xsmall` selector is exactly
  `claude/sonnet@medium | codex/gpt-5.5@medium | grok/grok-4.6@medium | agy/gemini-3.8-flash-high`,
  and resolving `@xsmall` with only Antigravity available yields
  `agy/gemini-3.8-flash-high` with no separate reasoning effort.
- Antigravity's `large`/`small` tier defaults still resolve to and pass
  `gemini-3.7-flash-high` and `gemini-3.7-flash-low`.
- `docs/llms.md`'s generated alias table, provider-resolution table, shorthand table,
  and Antigravity automatic-selection prose all reflect the change, and `just fmt-docs`
  is idempotent.
- Focused tests and `just check` pass.

## Out of scope

- **Moving Antigravity's `large`/`small` tier defaults to 3.8.** Not requested. Because
  `@xsmall` names a fully qualified target, nothing in this change depends on the tier
  map, and leaving it alone keeps the docs tier table and every "current default"
  example at 3.7 without further edits. Worth a follow-up decision, but not here.
- **Pruning the stale `gemini-3.5-flash-{high,medium,low}` entries.** The live catalog
  verified above no longer serves Gemini 3.5 Flash, so SASE's registry currently offers
  three models Antigravity will reject. Removing them is a user-reaching removal that
  needs its own deprecation treatment, so it is tracked on task bead `sase-wl` rather
  than folded into an additive change. Leave all three 3.5 entries in place here.
- Bryan's chezmoi-managed Antigravity presets (`#m_agy`, `#m_agy_pro`,
  `#m_agy_pro_flash`, `#m_swarm`, `#m_flash`) and the `tmux_ai_window` launcher, all of
  which currently target 3.7 High and stay there.
- Adding or changing advisories on any Antigravity model.
- Querying `agy models` dynamically in a UI or at SASE startup.
- Adding Gemini 3.8 to non-Antigravity providers such as OpenCode, whose identifiers and
  rollout contracts are separate.
- Changing `%model` grammar, completion rendering, model-picker layout, Rust LSP code,
  or PNG visual snapshots unless verification reveals a real regression in those generic
  layers.
- Rewriting historical `CHANGELOG.md` entries or editing SASE memory files and generated
  provider instruction shims.
