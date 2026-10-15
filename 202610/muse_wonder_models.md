---
tier: tale
title: Catalog Meta's Muse Wonder and Muse Wonder Full
goal:
  Muse Wonder and Muse Wonder Full are built-in muse models that every SASE model
  surface (TUI completion, model picker, labels, LSP, doctor, docs) shows correctly.
size: medium
decisions:
  large_tier:
    ask: Should Muse's large tier default move from muse-spark-1.3 to muse-wonder-1.0?
    choices:
      wonder:
        Unpinned Muse launches run Wonder, Muse Code's default; API tokens cost ~3-5x
        Spark's
      spark:
        Both tiers stay on muse-spark-1.3; Wonder runs only when named or picked by
        alias
    default: wonder
    why:
      Wonder is on every Muse Code plan and the API; SASE large tiers follow each
      flagship
    answer: wonder
  alias_pools:
    ask: Should the shipped @large size alias also load-balance onto Muse Wonder?
    choices:
      none: All size-alias pools stay as they are; @large never picks Wonder on its own
      wonder:
        Add muse/muse-wonder-1.0@xhigh to @large; about 1/4 of @large launches run
        Wonder
      wonder_contributor:
        Add muse/muse-wonder-1.0-contributor@xhigh to @large; cheaper, but Meta trains
        on it
    default: none
    why:
      Wonder is a days-old preview with self-reported benchmarks; try it by name first
    answer: none
proposed_by: bbugyi200.athena.10h
decided_by: reviewer
decided_via: tui
create_time: 2026-10-15 06:15:29
status: wip
---

# Catalog Meta's Muse Wonder and Muse Wonder Full

## Goal

Add Meta's new frontier models, **Muse Wonder** and **Muse Wonder Full** (released
2026-10-12), to SASE's built-in `muse` provider. They must show up correctly everywhere
SASE displays models:

- TUI `%model` completion in the prompt input widget;
- the `**`/`%m:` model-shortcut completion;
- the model picker modal;
- resolved model labels and advisory markers;
- agent-name fan-out suffixes;
- Agents-tab and archive model chips;
- `sase lsp` completion for external editors such as sase-nvim;
- `sase doctor` advisory checks;
- the generated and hand-written model docs.

## Evidence For The Model IDs

Web coverage disagrees on IDs; one outlet claims Wonder has no Full ID and no
Contributor tier. The authoritative source is the Muse Code CLI's own provider catalog.
Muse Code 1.4.8 caches it under `$XDG_DATA_HOME/muse/model-catalog/` (fetched 2026-10-14
on the dev host). It lists, in this order:

| `model_id`                         | Visibility | Notes                                                   |
| ---------------------------------- | ---------- | ------------------------------------------------------- |
| `muse-wonder-1.0`                  | visible    | `is_default: true` (Muse Code's new default)            |
| `muse-wonder-1.0-contributor`      | hidden     | "Your content … may be used for product improvement."   |
| `muse-wonder-full-1.0`             | visible    |                                                         |
| `muse-wonder-full-1.0-contributor` | hidden     | same Contributor notice                                 |
| `muse-spark-1.3` / `-contributor`  | vis / hid  | already in SASE                                         |
| `muse-spark-1.2` / `-contributor`  | vis / hid  | already in SASE; the only rows **without** `max` effort |

All four Wonder rows share these properties:

- context limit about 1M (1,007,997) and output limit 128k;
- reasoning-effort variants `minimal`/`low`/`medium`/`high`/`xhigh`/`max`;
- release date 2026-10-12.

Spark rows are hidden Contributor variants exactly like the Wonder ones, and SASE
already catalogs the Spark Contributors, so this plan catalogs all four Wonder IDs.

Public facts from Meta's announcement coverage (TechCrunch, eesel.ai, agent-one.dev):

- Wonder costs $4.00 in, $0.20 cached in, and $20.00 out per 1M tokens, with 1M context.
  It is on every Muse Code plan and the Meta Model API.
- Wonder Full is Muse Code-only. It needs the Power ($50/mo) or Maximum ($200/mo) plan,
  is not served through the Meta Model API, and has no per-token price.

Do **not** make a paid live `muse exec` call to re-verify; the catalog is sufficient.

## Where Models Reach The UX

The bundled manifest `src/sase/llm_provider/models.yml` is the single source.
`MuseProvider` (`src/sase/llm_provider/muse_provider.py`) answers every catalog hook
from it via `model_manifest.py`.

These surfaces pick up a manifest edit **automatically**. Verify them; do not edit them:

- **Registry.** `_registry_metadata.py`, `_registry_catalog.py`, and `registry.py`
  provide `model_to_provider_map`, `model_short_alias_map`, `model_advisory_map`, and
  `resolve_model_provider`. `model_validity.py` feeds the routing snapshot to the Rust
  `classify_model_value`.
- **TUI prompt completion.**
  - The catalog is built by `src/sase/macro/model_completion.py` and
    `_model_completion_catalog.py`, then filtered in Rust via
    `_model_completion_wire.py`.
  - The widget glue is `ace/tui/widgets/_directive_completion_models.py`,
    `directive_completion.py`, and `_file_completion_*`.
  - Model IDs must match `^[A-Za-z0-9_\-=./@]+$`; all four new IDs do.
- **Model picker.** `ace/tui/modals/model_picker_rows.py` and `model_picker_options.py`
  (provider header `MUSE  N models`, `(alias)`, advisory glyph).
- **Model label.** `model_label.py` renders `MUSE(model) ⚠/ⓘ @ effort`.
- **Agent fan-out names.** `macro/_directive_alt_naming.py` uses short aliases.
- **Usage labels.** `_usage_indicator_format.py` and `integrations/usage_windows.py`.
- **External editors.**
  - `sase lsp` (`src/sase/integrations/macro_lsp.py`) writes
    `model_completion_catalog_payload()` to `model_catalog.json`
    (`SASE_MACRO_MODEL_CATALOG`).
  - The Rust `sase-macro-lsp` binary re-reads that file on every request.
  - sase-core, sase-nvim, and sase-telegram hardcode no model IDs (checked), so **no
    linked-repo or Rust change is needed**. The Rust filters only receive the catalog as
    data, which satisfies the Rust-core boundary.
- **Doctor.** `doctor/checks_providers_registry.py` and `checks_providers_advisory.py`.
- **Generated docs.** These blocks in `docs/llms.md` are rendered by
  `tools/render_model_docs` via `just fix` and checked by `just fmt-docs-check`:
  `model-tiers-muse`, `model-alias-defaults`, `known-models`, `model-short-aliases`, and
  `provider-tier-defaults`.

These places are **hardcoded** and are edited below:

- the hand-written Muse pricing table and effort prose in `docs/llms.md`;
- the effort comment in `_muse_launch.py`;
- the `short_model_name` keyword list (Agents-tab fallback annotation and archive model
  chip);
- "either" prose in `docs/configuration.md`;
- exact-list test pins.

## Implementation

### 1. Manifest (`src/sase/llm_provider/models.yml`, `providers.muse`)

- **`models`.** Prepend in the Muse Code catalog order. The list becomes:
  `muse-wonder-1.0`, `muse-wonder-1.0-contributor`, `muse-wonder-full-1.0`,
  `muse-wonder-full-1.0-contributor`, `muse-spark-1.3`, `muse-spark-1.3-contributor`,
  `muse-spark-1.2`, `muse-spark-1.2-contributor`, `muse-spark-1.1`.
- **`short_aliases`.** These follow the existing `spark13`/`spark13c` family + version
  digits + `c` convention:
  - `muse-wonder-1.0: wonder10`
  - `muse-wonder-1.0-contributor: wonder10c`
  - `muse-wonder-full-1.0: wonderfull10`
  - `muse-wonder-full-1.0-contributor: wonderfull10c`
- **`tiers`.** Per `large_tier`:
  - `wonder`: set `large: muse-wonder-1.0` and keep `small: muse-spark-1.3`.
  - `spark`: leave both tiers on `muse-spark-1.3`.
  - In both cases no tier may point at a Contributor or at Wonder Full.
- **`advisories`.** Add three entries:
  - `muse-wonder-1.0-contributor`:
    - `severity: warn`
    - `label: trains on your data`
    - `detail`: "Meta uses this model's inputs and outputs to train and improve its AI
      models. Same capabilities as muse-wonder-1.0."
    - Do not copy Spark's "95% lower cost" or region claims; Meta publishes neither for
      Wonder.
  - `muse-wonder-full-1.0`:
    - `severity: info`
    - `label: needs Power plan or higher`
    - `detail`: "Muse Code offers Muse Wonder Full only on its Power and Maximum plans,
      and Meta does not serve it through the Meta Model API, so it is unavailable on
      other plans and to API-key logins."
    - This mirrors the Gemini 4 Argon `needs Google AI Ultra` precedent.
  - `muse-wonder-full-1.0-contributor`:
    - `severity: warn`
    - `label: trains on your data`
    - `detail`: "Meta uses this model's inputs and outputs to train and improve its AI
      models. Same capabilities as muse-wonder-full-1.0, which Muse Code offers only on
      its Power and Maximum plans."
    - The Contributor hazard outranks plan gating; there is one advisory per model.
- **Comment.** Rewrite the `muse:` comment block to say:
  - neither tier maps to a Contributor model (keep the existing training-corpus
    rationale);
  - Wonder Full is plan-gated, so it is never a tier default (like Argon);
  - which shipped pools reach a Contributor, updated per `alias_pools`.
- **`aliases`.** Per `alias_pools`; see the callouts below.
- Keep the YAML within the existing style and line widths. The manifest loader enforces:
  - unique IDs across providers;
  - short aliases that are unique and never equal to a bare model ID;
  - tiers that name catalog models.

> [!decision] alias_pools = none No `aliases:` edit. The `muse_provider.py` module
> docstring and the Selection prose in `docs/llms.md` stay accurate as written.

> [!decision] alias_pools = wonder Set `aliases.large.target` to
> `"claude/opus@high | codex/gpt-6.1-sol@xhigh | grok/grok-4.7@xhigh | muse/muse-wonder-1.0@xhigh"`.
> This is Wonder's first ladder appearance, so it runs at `xhigh`; leave `@xlarge`
> unchanged. Then make these edits:
>
> - In `test_shipped_small_aliases_carry_the_muse_contributor_member`
>   (`tests/llm_provider/test_load_balanced_alias_defaults.py`), expect
>   `["muse/muse-wonder-1.0@xhigh"]` for `large` and still `[]` for `xlarge`. Rename or
>   re-docstring it to cover both members.
> - Update the `muse_provider.py` module docstring to name both routes.
> - Update `docs/llms.md` "Muse Code Integration → Selection". It currently says the
>   size-alias path "lands on a Contributor model". Say that @large lands on Wonder and
>   the smaller sizes on the Spark Contributor.
> - Run `just model-policy-check`.

> [!decision] alias_pools = wonder_contributor Same as `wonder`, but the member is
> `muse/muse-wonder-1.0-contributor@xhigh`. The docs Selection prose then still says the
> path lands on a Contributor model; name both Contributor members.
> `sase doctor -C llm.model_advisory` will now warn for @large on a stock install. That
> is expected; confirm the warning appears.

### 2. Effort prose

- **`src/sase/llm_provider/_muse_launch.py`.** Lines 29-32 now say Meta documents `max`
  for Spark 1.3 only. Replace that with: Muse Code's model catalog lists `max` for the
  Wonder models and Spark 1.3 (including their Contributor variants) but not for Spark
  1.2, and other combinations are left to the CLI to validate. `MUSE_EFFORT_CLI_ARGS` is
  unchanged.
- **`docs/llms.md` "### Muse Reasoning Effort".** Make the same correction. It currently
  says `max` is documented for `muse-spark-1.3` only and "not claimed for Contributor or
  older models".

### 3. Hand-written docs

- **`docs/llms.md` "### Model Mapping".** Add four rows at the top of the hand-written
  Muse table, keeping column alignment (`just fix` reformats it):
  - `muse-wonder-1.0`:
    - Context `1M`; price `$4.00 / $0.20 / $20.00`.
    - Note: "Meta's frontier model (preview) for coding and long-running agentic work;
      Muse Code's catalog default. Every Muse Code plan and the Meta Model API."
  - `muse-wonder-1.0-contributor`:
    - Context `1M`; price "no published price".
    - Note: "Same capabilities as Wonder. **Meta uses its inputs and outputs to train
      and improve Meta's AI models.**"
  - `muse-wonder-full-1.0`:
    - Context `1M`; price "subscription only".
    - Note: "Larger Wonder variant. Muse Code Power and Maximum plans only; not served
      through the Meta Model API."
  - `muse-wonder-full-1.0-contributor`:
    - Context `1M`; price "subscription only".
    - Note: "Same capabilities as Wonder Full. **Meta uses its inputs and outputs to
      train…**"
  - Change the existing Spark 1.3 note from "Current coding-optimized model" to
    something like "Lower-latency coding-optimized model". Wonder is now the flagship;
    Spark remains supported.
- **`docs/llms.md` tier prose.** "**Both tiers map to the full-price model on purpose**"
  becomes "**Neither tier maps to a Contributor model on purpose**". Under
  `large_tier = wonder`, also note that `large` follows Muse Code's Wonder default while
  `small` stays on Spark 1.3.
- **`docs/llms.md` advisory prose.** The "bundled advisories" paragraph near the "Model
  advisories" section mentions the Argon entitlement advisory. Add the Wonder Full
  plan-gating advisory next to it.
- **`docs/configuration.md` Muse paragraph** (about line 5676). "never points a provider
  tier map at either" becomes "at any of them".
- Leave the `%model:muse/muse-spark-1.3` examples (`README.md`,
  `docs/getting_started.md`, `docs/macros.md`, `docs/llms.md`, `docs/configuration.md`
  tmux parity) unchanged. They stay valid and cheap, and `docs/macros.md` and the tmux
  parity example are test-pinned.
- `src/sase/default_config.yml` needs no edit. Its Muse lines are a grammar example and
  provider-wide usage settings, not catalog values.

### 4. Agents-tab and archive short model name

`short_model_name` in `src/sase/ace/tui/widgets/_agent_list_helpers.py` currently
collapses every Muse ID to `muse`. That makes a Wonder→Spark fallback render `↻1▸muse`
and gives every archive chip the same label.

- Add `"wonder-full"` then `"wonder"` to the keyword tuple, ordered so `wonder-full`
  wins.
- Do **not** add `"spark"`: `gpt-5.3-codex-spark` would start rendering as `spark`.
  Spark IDs keep rendering as `muse`.
- Add test cases to `tests/ace/tui/widgets/test_agent_list_helpers.py`:
  - `muse-wonder-1.0` → `wonder`;
  - `muse-wonder-full-1.0-contributor` → `wonder-full`;
  - `muse-spark-1.3` → `muse` (regression).

### 5. Tests

- **`tests/llm_provider/_muse_provider_helpers.py`.** Set `_MUSE_MODELS` to the new
  nine-ID ordered list.
- **`tests/llm_provider/test_muse_provider_identity.py`.**
  - Extend the short-alias dict with the four Wonder aliases.
  - Add `muse-wonder-1.0` and `muse-wonder-full-1.0-contributor` to the
    implicit-resolution parametrize list.
  - Update `test_muse_provider_resolve_model_name_never_routes_a_tier_to_contributor`
    per `large_tier`. Under `wonder`: default and `large` give `muse-wonder-1.0`, and
    `small` gives `muse-spark-1.3`. Keep the no-Contributor intent in the docstring.
- **`tests/llm_provider/test_muse_provider_invocation.py`.** Under
  `large_tier = wonder`, the default `--model` pin at line 29 becomes `muse-wonder-1.0`.
  Grep `tests/llm_provider/test_muse_*.py` for other `muse-spark-1.3` pins that come
  from the tier default rather than an explicit model, and update only those.
- **`tests/llm_provider/test_model_advisories.py`.**
  - Extend `_CONTRIBUTORS` with both Wonder Contributors.
  - `test_muse_flags_the_contributor_model` asserts
    `set(advisories) == set(_CONTRIBUTORS)`. Change it to
    `set(_CONTRIBUTORS) | {"muse-wonder-full-1.0"}`.
  - The render-site tests assume each Contributor's standard sibling has **no**
    advisory. That fails for Wonder Full, whose sibling has the info advisory. Add
    `_UNFLAGGED_SIBLING_CONTRIBUTORS` (every Contributor except
    `muse-wonder-full-1.0-contributor`) and parametrize the sibling assertions over it.
    The affected tests are:
    - `test_registry_publishes_the_contributor_advisory`
    - `test_model_picker_row_carries_the_advisory`
    - `test_model_picker_option_renders_the_advisory`
    - `test_model_completion_detail_carries_the_advisory`
    - `test_model_completion_payload_exposes_the_advisory`
    - `test_model_label_marks_an_active_advisory_model`
  - Keep the warn-label assertions over all of `_CONTRIBUTORS`.
  - `test_tier_mapping_never_routes_to_an_advisory_model` must still pass unchanged.
  - The file is 270 lines; keep the new Wonder tests out of it (next bullet) so it stays
    well under the `toobig` limit.
- **New file `tests/llm_provider/test_muse_wonder_catalog.py`.** One focused module that
  proves each UX surface sees the new models through the **real** manifest (no catalog
  patching):
  - **Resolution:** `resolve_model_provider()` maps `"muse/muse-wonder-full-1.0"`,
    `"muse-wonder-1.0"`, and the short aliases `wonder10` and `wonderfull10` to
    `("muse", <id>)`. Check how short aliases resolve in
    `model_alias_resolution*.py`/`registry.py` first; if short aliases resolve only via
    completion, assert there instead.
  - **Model picker:**
    - `build_model_rows()` contains all four Wonder IDs under the muse provider with
      their `(alias)`.
    - Wonder Full's row has `advisory_severity == "info"` and an `ⓘ` in its label.
    - `muse-wonder-1.0` carries no advisory.
  - **TUI completion:**
    - `build_model_completion_catalog(use_cache=False)` lists the four Wonder entries,
      with `aliases`, under provider `muse`, ahead of the Spark entries (catalog order).
    - The Wonder Full entry's description carries `ⓘ needs Power plan or higher`.
    - Exercise the Rust-backed filter the prompt widget uses
      (`_model_completion_wire.py`, e.g. `filter_model_completion_entries`). For the
      scoped prefix `muse/wonder`, it returns the four Wonder IDs. For the shortcut
      prefix `wonderf`, the explicit-model shortcut filter returns only the two Full
      entries. Find the public wrapper names in that module.
  - **LSP payload:** `model_completion_catalog_payload()["entries"]` contains:
    - `muse-wonder-1.0` with `aliases == ["wonder10"]` and `advisory_label == ""`;
    - `muse-wonder-full-1.0` with `advisory_label == "needs Power plan or higher"`;
    - `muse-wonder-1.0-contributor` with `advisory_label == "trains on your data"`.

    This JSON is what `sase lsp` serves to editors.

  - **Label:**
    `model_value_text("muse-wonder-full-1.0", "muse", "high").plain == "MUSE(muse-wonder-full-1.0) ⓘ @ high"`.
    Check the exact marker spacing against the existing Contributor label test first.

- **`tests/llm_provider/test_load_balanced_alias_defaults.py`.** Change it only under
  `alias_pools != none` (see the callouts).
- These need no change: `test_model_manifest.py`,
  `tests/test_manifest_synthetic_exercise.py`,
  `tests/doctor/test_checks_providers_advisory.py`, `tests/test_model_picker_jump.py`,
  `tests/test_model_picker_usage_hints.py`, and the synthetic-catalog TUI/LSP parity
  tests. Run them anyway as regression coverage.

## Verification

1. Run `just fix` to regenerate the `docs/llms.md` blocks and reformat Python and
   Markdown. Inspect the diff: the Muse rows in `known-models`, `model-short-aliases`,
   `model-tiers-muse`, `provider-tier-defaults`, and (if pools changed)
   `model-alias-defaults`.
2. Run `just model-policy-check`.
3. Run the targeted tests:
   `.venv/bin/pytest -q tests/llm_provider/test_muse_wonder_catalog.py tests/llm_provider/test_model_advisories.py tests/llm_provider/test_muse_provider_identity.py tests/llm_provider/test_muse_provider_invocation.py tests/llm_provider/test_model_manifest.py tests/llm_provider/test_load_balanced_alias_defaults.py tests/test_manifest_synthetic_exercise.py tests/ace/tui/widgets/test_agent_list_helpers.py tests/main/test_lsp_handler_environment.py tests/doctor/test_checks_providers_advisory.py tests/test_model_picker_jump.py tests/test_model_picker_usage_hints.py tests/test_macro_directive_completion_parity.py tests/test_macro_model_alias_shortcut_parity.py`
4. Spot-check the editor surface:
   `.venv/bin/python -c "from sase.macro.model_completion import model_completion_catalog_payload as p; print([(e['value'], e.get('aliases'), e.get('advisory_label')) for e in p()['entries'] if 'wonder' in e['value']])"`
   should print the four Wonder entries.
5. Spot-check doctor: `sase doctor -C llm.model_advisory` should show no new warning
   under `alias_pools = none`.
6. Run `sase tool run check`. If its scoped lane reports a PNG golden diff for a
   model-picker or models-panel snapshot, inspect it. Those goldens filter on `opus` and
   agy, so a Muse change should not move them. Refresh only a golden that changed for
   this reason, with `just fix-tui-screenshots` scoped to that test. Do not run
   `just check-full`.

## Out Of Scope

- **`muse-spark-1.1`** is absent from the cached Muse Code catalog and may be retired.
  Leave it; retiring it needs separate confirmation.
- **Per-model Muse usage windows.** Wonder Full may have its own subscription limits.
  The probe stays account-wide.
- **Muse Code's own `--contributor-mode` account default.** This plan does not change
  how SASE invokes `muse`.
- **Example prompts** that use `muse-spark-1.3` are unchanged.
