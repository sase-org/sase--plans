---
tier: tale
title: Add Gemini 4 Argon to the Antigravity model catalog
goal:
  Gemini 4 Argon (gemini-4-argon-high/medium/low) appears on every sase surface that
  lists models without becoming a default that breaks agy users on other plans.
size: small
decisions:
  ultra_advisory:
    ask:
      Should the Argon slugs carry an info model advisory saying they need Google AI
      Ultra?
    choices:
      info:
        Picker, completion, and run labels show ⓘ needs Google AI Ultra; doctor warns on
        routes to Argon
      none:
        No marker on any surface; the AI Ultra gate is documented only in docs/llms.md
    default: info
    why: Argon fails outright for most plans, and these surfaces are where users pick it
    answer: info
  size_alias:
    ask: Should Gemini 4 Argon join a shipped size alias?
    choices:
      catalog_only:
        Reachable by name, picker, and completion only; no size alias routes to it
        automatically
      large_pool:
        Adds agy/gemini-4-argon-high to @large; non-Ultra agy users get some failed
        @large launches
    default: catalog_only
    why:
      agy exits 1 on models the account can't use, and aliases auto-route whenever agy
      is on PATH
    answer: catalog_only
proposed_by: bbugyi200.athena.10f
decided_by: reviewer
decided_via: tui
create_time: 2026-10-14 10:56:32
status: wip
---

# Add Gemini 4 Argon to the Antigravity (agy) model catalog

## Goal

Make Google's Gemini 4 Argon a first-class built-in `agy` model, so it shows up on every
sase surface that lists models:

- the TUI model picker and Models panel selectors;
- `%model:` / `%m:` completion in the prompt input widget;
- external-editor completion served by `sase lsp` (sase-nvim and other LSP clients);
- the provider/model directive validity checks;
- the generated model tables in `docs/llms.md`;
- agy usage-window applicability;
- compact model names in agent rows.

Antigravity gates Argon behind Google AI Ultra, so the new model must never become a
default that breaks agy users on other plans.

## Established facts (do not re-research)

- **Slugs.** Antigravity's official headless CLI docs
  (antigravity.google/docs/cli/headless) show `agy models` printing
  `gemini-4-argon-high  Gemini 4 Argon (High)` and
  `gemini-4-argon-medium  Gemini 4 Argon (Medium)` _above_ the `gemini-3.8-flash-*`
  rows.
  - The models page (antigravity.google/docs/models) gives Argon Low/Medium/High effort
    options with High as the default, and community inspection of the 2.23.0 build lists
    `gemini-4-argon-low`.
  - Catalog all three variants: `gemini-4-argon-high`, `gemini-4-argon-medium`,
    `gemini-4-argon-low`.
  - Do not add the bare `gemini-4-argon`. It is the Gemini _API_ id, and no other agy
    family is catalogued without its effort suffix.
- **Gating.** Antigravity rolled Argon out on 2026-10-13 for Google AI Ultra and
  eligible Gemini Enterprise subscriptions. Free, AI Plus, and AI Pro accounts are
  excluded. Headless `agy` exits 1 rather than falling back when `--model` names a model
  the account cannot use.
  - The local `agy` 1.3.5 on the dev host does not list Argon in `agy models`. So the
    implementer cannot smoke-test a real Argon invocation; verification is
    catalog-level.
- **Context sizes are out of scope.** The 256K/512K/900K context sizes are an in-app
  Antigravity setting. There is no slug or CLI flag for them.
- **Every model surface reads `src/sase/llm_provider/models.yml` dynamically.** Its
  header calls it the single edit point for built-in model maintenance. The consumers:
  - **Registry and provider hooks.** `agy.py` calls the manifest accessors, and
    `_registry_catalog.py` builds the registry from them.
  - **Prompt-input completion.**
    - `macro/model_completion.py` and `macro/_model_completion_catalog.py` read the
      registry payload, with filtering done through the `sase_core_rs` wire.
    - The TUI callers are `ace/tui/widgets/_directive_completion_models.py`,
      `directive_completion.py`, and `_file_completion_*`.
  - **Editor completion.** `integrations/macro_lsp.py` writes
    `model_completion_catalog_payload()` to `<sase>/macro_lsp/model_catalog.json` when
    `sase lsp` starts, and the Rust `sase_macro_lsp` re-reads that file per request.
  - **Model picker.** `ace/tui/modals/model_picker_rows.py`.
  - **Directive validity.** `model_validity_snapshot()` →
    `sase_core_rs.classify_model_value`.
  - **Docs.** `tools/render_model_docs` regenerates the `docs/llms.md` blocks on
    `just fix`.
  - **Policy gate.** `llm_provider/model_policy.py` runs through
    `just model-policy-check`, which is part of `just check`.
- **No hard-coded agy slugs elsewhere.** sase-nvim, sase-telegram, sase-github, and
  sase-core contain none outside test fixtures. No Rust or sibling-repo change is needed
  and the `sase-core-revision.txt` pin does not move.
  - sase-core's agy usage normalizer (`crates/sase_core/src/provider_usage/agy.rs`)
    assigns every catalog id starting with `gemini-` to the `gemini` usage family.
  - So Argon is automatically covered by the `gemini-weekly` / `gemini-5h` windows.
- **One presentation helper needs a code change.**
  `src/sase/ace/tui/widgets/_agent_list_helpers.py:short_model_name` maps a model to a
  compact chip by keyword (`flash`, `fable`, `opus`, `sonnet`, `haiku`, `pro`), falling
  back to the first `-` segment.
  - Today `gemini-4-argon-high` would render as the generic `gemini`.
  - The chip appears in the running-agent fallback annotation (`↻N▸<name>`) and in the
    agents-archive model chip.

## Changes

### 1. Manifest: `src/sase/llm_provider/models.yml`

- **Model list.** In `providers.agy.models`, insert the three slugs at the very top, in
  CLI order: `gemini-4-argon-high`, `gemini-4-argon-medium`, `gemini-4-argon-low`. They
  come before `gemini-3.8-flash-high`, matching the "keep CLI ordering" rule and the
  official `agy models` output.
- **Short aliases.** Add the matching `short_aliases` entries, following the `flash38h`
  / `pro31h` family+version+effort pattern:
  - `gemini-4-argon-high: argon4h`
  - `gemini-4-argon-medium: argon4m`
  - `gemini-4-argon-low: argon4l`

  These collide with no existing alias or bare model name; the loader rejects
  collisions, so the run would fail if one existed.

- **Tiers stay put.** Leave `tiers` unchanged (`large: gemini-3.7-flash-high`,
  `small: gemini-3.7-flash-low`). A tier default must work for every agy account, and
  Argon fails for non-Ultra accounts.
- **Comment.** Add a one-to-three-line YAML comment above the Argon entries recording
  this tier rationale, matching the explanatory comments in the `grok` and `muse`
  blocks.

> [!decision] ultra_advisory = info Add an `advisories` block under `providers.agy`, one
> entry per Argon slug, mirroring the Muse Contributor entries' shape:
>
> - `severity: info`
> - `label: needs Google AI Ultra`
> - `detail`: one sentence saying Antigravity offers Gemini 4 Argon only on Google AI
>   Ultra and eligible Gemini Enterprise subscriptions (not Free, AI Plus, or AI Pro),
>   and that `agy` exits with an error when the signed-in account cannot use it. It
>   should point at `agy models` to check access.
>
> Through the existing registry plumbing, this renders with no code change:
>
> - an `ⓘ needs Google AI Ultra` suffix on model-picker rows, with the detail as
>   secondary text;
> - a `— ⓘ needs Google AI Ultra` tail in `%model` completion descriptions, in both the
>   TUI and the `sase lsp` catalog;
> - an inline blue `ⓘ` on the resolved model label for any Argon run;
> - a `sase doctor -C llm.model_advisory` warning when a configured default or alias
>   resolves to Argon.

> [!decision] ultra_advisory = none Add no `advisories` block.

> [!decision] size_alias = large_pool Append ` | agy/gemini-4-argon-high` to
> `aliases.large.target`, with no `@effort` suffix: agy bakes effort into the slug and
> rejects alias-supplied efforts.
>
> Per the size-alias effort-ladder decision, this is Argon's first appearance scanning
> down from `@xlarge`, so it takes its top variant (`-high`).
>
> If `ultra_advisory = info` also ships, the stock install's `@large` will trigger the
> doctor advisory warning by design. Say so in the docs paragraph in step 4.

> [!decision] size_alias = catalog_only Leave `aliases` untouched.

### 2. TUI compact model name: `src/sase/ace/tui/widgets/_agent_list_helpers.py`

Add `"argon"` to the `short_model_name` keyword tuple, so every Argon variant renders as
`argon` instead of `gemini`. Order is irrelevant here, since no other catalog id
contains "argon". This is presentation-only Python and stays in this repo.

### 3. Tests

Update the literal agy enumerations that intentionally pin the catalog:

- `tests/llm_provider/test_agy_provider_core.py`:
  - prepend the three slugs to `_AGY_MODELS`;
  - prepend the three alias pairs to the expected `llm_model_short_aliases()` dict.
  - Tier assertions stay unchanged.
- `tests/llm_provider/test_agy_usage_probe.py`: prepend the three slugs to
  `_GEMINI_IDS`. This is the Gemini-family `model_ids` applicability list, in catalog
  order. `_3P_IDS` is unchanged.
- `tests/test_model_picker_jump.py`
  (`test_model_picker_jump_hints_follow_filtered_visible_order`):
  - The `"gemini"` filter now leads with `gemini-4-argon-high`, `gemini-4-argon-medium`.
    Update the expected `visible_ids[:3]` to match.
  - Refresh the inline comment ("Antigravity's newest Gemini slugs leading…").
- `tests/ace/tui/widgets/test_agent_list_helpers.py`: add
  `test_short_model_name_recognizes_gemini_argon`, asserting `argon` for
  `gemini-4-argon-high` (and optionally the `-low` variant), in the existing
  one-assert-per-test style.

> [!decision] ultra_advisory = info In `tests/llm_provider/test_model_advisories.py`:
>
> - Add a test that `model_advisory_map()` publishes an `info` advisory labelled
>   `needs Google AI Ultra` for each of the three Argon slugs.
> - Add a test that none of `gemini-3.8-flash-*` / `gemini-3.7-flash-*` carries one.
> - Extend the tier guard, keeping the existing Muse test intact: add an agy test that
>   asserts no `provider_tier_model("agy", tier)` is an advisory-flagged model. Keep the
>   docstring rationale short. This encodes the "tiers must work for every account" rule
>   from step 1.

> [!decision] size_alias = large_pool Update any test that pins the shipped `@large`
> target literally. Most alias tests use the frozen fixture in
> `tests/_conftest_runtime.py`, so expect few or none. Then add or extend a policy-level
> test only if `just model-policy-check` alone would not catch a regression.

These tests derive from the real catalog and should pass unchanged. If one fails,
investigate rather than paper over it:

- `tests/test_macro_model_completion_catalog.py`
- `tests/llm_provider/test_model_manifest.py`
- `tests/test_manifest_synthetic_exercise.py`
- `tests/test_llm_provider_core.py`
- `tests/test_model_picker_options.py`
- `tests/test_render_model_docs.py`, once docs are regenerated
- `tests/llm_provider/test_model_policy.py`

### 4. Docs

- **Generated tables.** Run `just fix` to regenerate the `docs/llms.md` generated
  blocks: `known-models` and `model-short-aliases`, plus `model-alias-defaults` under
  `size_alias = large_pool`. Commit the regenerated output; do not hand-edit inside
  `BEGIN/END GENERATED` markers.
- **Antigravity "Model Mapping" prose.** In `docs/llms.md` under "Model Mapping", after
  the paragraph that follows the `model-tiers-agy` block, add a short paragraph
  covering:
  - Gemini 4 Argon is catalogued as `gemini-4-argon-high|medium|low`, and the effort is
    chosen by the slug variant.
  - Antigravity serves it only to Google AI Ultra and eligible Gemini Enterprise
    accounts, and `agy` fails the run when the account lacks access.
  - For that reason the agy tier defaults stay on Gemini Flash. Under
    `size_alias = catalog_only`, add "and no shipped size alias routes to Argon".

> [!decision] ultra_advisory = info In `docs/llms.md` "Model Advisories", extend the
> closing sentence ("The bundled advisories are the manifest's `advisories` entries…")
> to say they also include Antigravity's informational Gemini 4 Argon entitlement
> advisory, and link to the Antigravity section. Under `size_alias = large_pool`, also
> note that stock `@large` therefore reports a doctor advisory warning.

Leave the illustrative example slugs elsewhere alone: `default_config.yml:321`,
`sase.schema.json:3951`, and the example slugs in `docs/macros.md` and
`docs/configuration.md`. They are examples, not lists. `CHANGELOG.md` is
release-generated; do not edit it.

## Explicit non-goals

- No change to agy tier defaults (rationale in step 1).
- No sase-core / Rust, sase-nvim, sase-telegram, or sase-github change. All of them
  consume the catalog through the registry payload, the `sase lsp` catalog file, or the
  `gemini-` prefix rule.
- No context-window (256K/512K/900K) modeling.
- Do not reconcile the wider agy catalog with the local `agy` 1.3.5 output. On the dev
  host, `agy models` lists only `gemini-3.8-flash-*`, `claude-opus-5-5-*`,
  `claude-sonnet-5-5-*`, and `gpt-oss-120b-medium`. It lacks the 3.7/3.6/3.5 Flash and
  3.1 Pro slugs, including the agy tier default `gemini-3.7-flash-high`.
  - That drift may be account-dependent and is a separate investigation.
  - The implementer should file it as a follow-up task bead through `/sase_new_task`
    (default size `large`) rather than fix it here.

## Verification

1. `just fix`, then `sase tool run check`. That covers lint, `model-policy-check`,
   `fmt-docs-check` for the generated tables, and the diff-scoped tests. If `sase tool`
   is unavailable, use `SASE_TOOL_BYPASS='<why>' just check`. Do not run
   `just check-full`.
2. Catalog spot-check from the venv: confirm `gemini-4-argon-high/medium/low` appear in
   order in these, with the `ⓘ needs Google AI Ultra` text in the completion
   descriptions under `ultra_advisory = info`:
   - `sase.macro.model_completion.model_completion_catalog_payload()`, which is the
     exact payload `sase lsp` hands editors;
   - `sase.llm_provider.registry.model_short_alias_map()`;
   - `sase.ace.tui.modals.model_picker_rows.build_model_rows()`.
3. `sase doctor -C llm.model_policy -C llm.registry -C llm.model_advisory`. Expect
   `model_policy` OK. `model_advisory` should be OK on a stock install unless
   `size_alias = large_pool` and `ultra_advisory = info` both ship; in that case a
   single `@large` warning is expected.
4. PNG snapshots: the model-picker and completion visual fixtures use synthetic rows or
   filters that exclude agy, so no golden should change. Do not run
   `just fix-tui-screenshots` unless a visual test that renders the real agy catalog
   actually fails.
5. In the final summary, note that running TUI sessions and already-started `sase lsp`
   servers pick up the new catalog only after restart. The LSP catalog file is written
   at server start.
