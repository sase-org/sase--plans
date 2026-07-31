---
tier: tale
title: Hide the fakey test provider from model pickers
goal:
  Model pickers and %model completion never offer the internal fakey provider, while fakey routing and resolution keep
  working.
create_time: 2026-07-31 07:09:13
status: done
---

- **PROMPT:** [202607/prompts/hide_fakey_from_model_pickers.md](prompts/hide_fakey_from_model_pickers.md)

# Hide the Fakey Test Provider From Model Pickers

## Objective

Stop offering the bundled `fakey` test provider in SASE's user-facing model pickers — the ACE modal model picker and the
`%model` completion catalog (which also feeds the Rust xprompt LSP snapshot) — while keeping `fakey` fully registered,
routable, and explicitly selectable so the test harness, retry fallbacks, and demos keep working unchanged.

## Current Behavior and Contract

`fakey` is a real, bundled provider registered through the `sase_llm` entry-point group. Its metadata hooks
(`llm_known_model_names` → `["fakey-large", "fakey-small"]`, `llm_model_short_aliases` →
`{"fakey-large": "fakeyl", "fakey-small": "fakeys"}`) flow into the memoized registry payload, which is the single
source for both pickers _and_ for model→provider routing.

Surfaces that show `fakey` today (each verified against the live registry in this workspace):

1. **ACE modal model picker** — `build_model_rows()` in `src/sase/ace/tui/modals/model_picker_rows.py` groups
   `model_to_provider_map()` by provider, producing a `FAKEY  2 models` header plus `fakey-large` and `fakey-small`
   rows. Reached from the approve-options modal, the Models-panel override picker, and the Models-panel alias-edit
   picker.
2. **`%model` completion catalog** — `_build_static_catalog()` in `src/sase/xprompt/model_completion.py` emits
   `fakey-large` and `fakey-small` model entries plus the implicit `@fakey_coder` alias entry (generated once per
   registered provider). This catalog backs the ACE prompt-bar completion menu and the `model_catalog.json` payload
   materialized for the Rust LSP in `src/sase/integrations/xprompt_lsp.py`.
3. **Alias rows** — `build_alias_views()` in `src/sase/llm_provider/alias_view.py` yields an `@fakey_coder`
   `provider_coder` view (currently resolving to `codex/gpt-5.6-sol`), which renders both in the Models-panel `coders`
   bucket and in the ALIASES section of the modal picker when a picker is opened with an `AliasSelectionContext`.

Behavior that MUST survive this change:

- Bare `fakey-large` / `fakey-small` must keep inferring the `fakey` provider. `tests/fakey/harness.py` drives the
  end-to-end suite with `%model:fakey-large`, `tests/fakey/test_provider.py` asserts
  `model_to_provider_map()["fakey-large"] == "fakey"`, retry tests use `fallback_model="fakey-small"`, and
  `docs/fakey.md` documents `sase run -m fakey-large`.
- Explicit `fakey/fakey-large` syntax, tier resolution (`large` → `fakey-large`, `small` → `fakey-small`), the
  `fakeyl`/`fakeys` short aliases used for multi-model agent-name suffixes (`_model_suffixes` in
  `src/sase/xprompt/_directive_alt_naming.py`), autodetect priority, and the retry config must all be untouched.
- `@fakey_coder` must remain a _resolvable_ alias (`is_provider_coder_alias`, `provider_coder_model_alias_names`,
  `model_alias_names`) so a plan authored by a fakey agent still resolves its coder follow-up model; only its display
  rows go away, and a user who explicitly configures `fakey_coder` in `llm_provider.model_aliases` must still see it.
- Provider-status surfaces stay as they are: `sase doctor` provider checks, the agent-CLI/plugins browser inventory,
  provider badges, and provider styles all legitimately describe `fakey` as an installed bundled CLI. This change is
  about _model selection_, not provider inventory.

Because routing and picker display share one metadata source, deleting `llm_known_model_names` /
`llm_model_short_aliases` from `src/sase/llm_provider/fakey.py` is the wrong fix: it would also delete bare-name
inference and the short aliases. The hiding must therefore happen at the picker layer, driven by an explicit provider
metadata flag rather than a hardcoded `"fakey"` string in TUI code.

Boundary note: no Rust change is required. `crates/sase_xprompt_lsp` only _loads_ `model_catalog.json`; the catalog is
built in Python, so filtering in `_build_static_catalog()` is the single shared point that fixes the TUI and the editor
integration together.

## Implementation

1. Add a provider-level "hidden from model pickers" metadata flag.
   - In `src/sase/llm_provider/_hookspec.py`, add a metadata hookspec
     `llm_hidden_from_model_pickers(self) -> bool | None` (`firstresult=True`, alongside the other `--- Metadata ---`
     hooks). Document it as: providers that exist only for testing return `True`; hiding affects user-facing model
     selection surfaces only and never changes routing, resolution, autodetect, or doctor output. Omitting the hook
     means "not hidden", so third-party providers stay compatible.
   - In `src/sase/llm_provider/registry.py`, have `_provider_metadata()` record
     `"hidden_from_model_pickers": _call_optional(plugin, "llm_hidden_from_model_pickers") is True` and add a public
     accessor (e.g. `model_picker_hidden_provider_names() -> frozenset[str]`) that reads the cached payload's
     `providers` map and tolerates a missing key. Leave `model_to_provider`, `model_short_aliases`,
     `autodetect_candidates`, and every other aggregate exactly as they are.
   - In `src/sase/llm_provider/fakey.py`, implement the hook with `@hookimpl` returning `True`. Change nothing else in
     that module.

2. Filter the ACE modal picker.
   - In `build_model_rows()` (`src/sase/ace/tui/modals/model_picker_rows.py`), skip providers in the hidden set while
     grouping `model_to_provider_map()`. The `Follow-up default` row, alias rows, and the `Custom...` escape hatch stay
     unchanged — `Custom...` remains the supported way to type `fakey/fakey-large` by hand.

3. Filter the `%model` completion catalog.
   - In `_build_static_catalog()` (`src/sase/xprompt/model_completion.py`), drop hidden providers from the computed
     `provider_order` before the known-model loop. That one change also stops `_append_implicit_alias_entries()` from
     emitting `@<provider>_coder` for hidden providers, since it iterates the same order.
   - Also guard the second loop that back-fills entries straight from `model_to_provider`, so hidden providers' models
     do not slip back in through the "partial plugin metadata" fallback path.
   - A user-configured `fakey_coder` alias still surfaces through the existing user-alias loop; do not special-case it.

4. Filter implicit provider-coder alias rows.
   - In `build_alias_views()` (`src/sase/llm_provider/alias_view.py`), skip a name when it is an implicit
     `<provider>_coder` alias for a hidden provider — i.e. when `is_provider_coder_alias(name)` holds, the provider is
     hidden, and the name is not in `get_model_aliases()`. This removes `@fakey_coder` from the Models-panel `coders`
     bucket and from the picker's ALIASES section while keeping a configured `fakey_coder` visible and editable.
   - Do **not** change `provider_coder_model_alias_names()`, `special_model_alias_names()`, `model_alias_names()`, or
     `is_provider_coder_alias()` in `src/sase/llm_provider/model_alias_config.py`: those drive `%model` validation and
     alias resolution, which must keep working for fakey-authored plans.
   - Check the Models-panel callers of `build_alias_views()` / `build_models_panel_rows()`
     (`src/sase/ace/tui/modals/models_panel.py`, `alias_view.build_models_panel_rows`) for any assumption that every
     name in `model_alias_names()` has a view; fix any that exists rather than reintroducing the row.

5. Update tests.
   - `tests/test_llm_provider_registry.py`: extend the existing `FakeProvider`-style `_provider_metadata` tests to cover
     the new flag — present-and-`True`, present-and-`False`, and hook-absent (defaults to not hidden).
   - `tests/test_model_picker_options.py`: assert `build_model_options()` / `build_model_rows()` contain no
     `__header_fakey__` header and no `fakey-large` / `fakey-small` option ids, while the existing non-fakey
     expectations (including `Custom...` and `Follow-up default`) still hold.
   - `tests/test_xprompt_model_completion.py`: add a real-registry assertion that no catalog entry has provider `fakey`,
     no value starts with `fakey-`, and `@fakey_coder` is absent; and add a fixture-metadata test that a synthetic
     hidden provider is filtered from both models and its `@<provider>_coder` alias, so the contract is covered by the
     hook rather than by the literal name `fakey`.
   - `tests/llm_provider/test_alias_view.py`: assert `build_alias_views()` omits `fakey_coder` by default but includes
     it when configured through `llm_provider.model_aliases`, and that other `<provider>_coder` views are unaffected.
   - `tests/fakey/test_provider.py`: keep the existing registry assertions and add regression coverage that hiding is
     display-only — `model_to_provider_map()["fakey-large"] == "fakey"`, `model_short_alias_map()` still carries
     `fakeyl`/`fakeys`, and `resolve_model_provider` handles both `fakey-large` and `fakey/fakey-small`.
   - `tests/llm_provider/test_config_role_aliases.py` (or the nearest existing module): confirm `%model:@fakey_coder`
     still resolves and `is_provider_coder_alias("fakey_coder")` is still `True`.

6. Update documentation.
   - `docs/llms.md`: keep `fakey-large` / `fakey-small` in the automatic-provider table and `fakeyl` / `fakeys` in the
     short-alias table (that behavior is unchanged) and add a short note that `fakey` is deliberately hidden from the
     model picker and `%model` completion, and must be typed explicitly (or entered via `Custom...`).
   - `docs/fakey.md`: state that fakey models are not offered in pickers or completion and show the explicit
     `%model:fakey-large` / `sase run -m fakey-large` spellings that still work.
   - `docs/agent_providers.md`: adjust the fakey paragraph (around the `%model:fakey-large` guidance) if it implies
     picker discoverability.

## Validation

1. Run `just install` first — this is an ephemeral SASE workspace, so dependencies may be stale.
2. Run the focused unit tests: `tests/test_llm_provider_registry.py`, `tests/test_model_picker_options.py`,
   `tests/test_model_picker_aliases.py`, `tests/test_xprompt_model_completion.py`, `tests/llm_provider/`, and
   `tests/fakey/`.
3. Run the Models-panel and model-completion visual suites
   (`tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py`,
   `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py`,
   `tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py`). The completion goldens use fixed fake rows and
   the Models-panel goldens use synthetic alias views, so most should be unaffected; any picker golden that renders live
   registry rows may legitimately lose the FAKEY group. Regenerate only those with `--sase-update-visual-snapshots`,
   inspect the diffs under `.pytest_cache/sase-visual/`, then rerun in normal comparison mode.
4. Run the full `just check` gate and resolve any lint, type, symvision, unit-test, or snapshot fallout before handoff.
5. Sanity-check the runtime contract by hand: `sase run -m fakey-large ...`-style resolution paths are already covered
   by `tests/fakey/`, so confirm that suite passes rather than launching a real agent.

## Acceptance Criteria

- The ACE modal model picker shows no `FAKEY` provider group and no `fakey-large` / `fakey-small` rows; every other
  provider group is unchanged.
- The `%model` completion catalog (and therefore the materialized `model_catalog.json` for the Rust LSP) contains no
  fakey model entries and no implicit `@fakey_coder` entry.
- The Models panel and the picker's ALIASES section no longer list `@fakey_coder` unless the user configured it.
- Hiding is opt-in provider metadata, not a hardcoded provider name in the TUI or completion layer: a synthetic provider
  that implements the new hook is filtered by the same code path, and providers that omit the hook are unaffected.
- Routing is untouched: `fakey-large` / `fakey-small` still infer the `fakey` provider, `fakey/fakey-*` still resolves,
  short aliases still exist, `@fakey_coder` still resolves, and the full `tests/fakey/` suite passes.
- Docs describe the new picker behavior, and `just check` passes.
