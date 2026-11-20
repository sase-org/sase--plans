---
tier: epic
title: Gemini 4 Neon for the research swarm
goal: 'SASE catalogs Gemini 4 Neon on Antigravity with accurate plan-gating advisories,
  and the #research_swarm Gemini researcher runs on agy/gemini-4-neon-xhigh instead
  of Gemini 4 Hydrogen.

  '
decisions:
  image_pool:
    ask: Should the plugin's @image infographic pool also swap its Antigravity member
      from Hydrogen to Neon?
    default: true
    why: The image agent is a swarm role, and the Hydrogen change moved both together.
    answer: true
  hydrogen_advisory:
    ask: Should Hydrogen also get a plan-gating advisory like Argon and Neon, now
      that Free and Plus lose it?
    default: true
    why: Antigravity's docs now exclude Free, Plus, and Pro trials from Hydrogen.
    answer: true
phases:
- id: catalog
  title: Catalog Gemini 4 Neon in sase's agy manifest
  depends_on: []
  size: medium
  description: 'catalog: add the four gemini-4-neon slugs, short aliases, and info
    advisories to models.yml, teach the TUI the neon short name, correct the stale
    Hydrogen availability prose, and update agy tests, docs, and model-picker PNG
    goldens.'
- id: swarm
  title: Switch the research swarm's Gemini roles to Neon
  depends_on: []
  size: small
  description: 'swarm: in the linked sase-research-artifacts repo, change the gemini_model
    default (and the @image pool''s agy member) to agy/gemini-4-neon-xhigh and update
    its tests and docs.'
proposed_by: bbugyi200.athena.1eo
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 16:42:47
status: wip
bead_id: sase-1uj
---

- **PROMPT:** [prompts/202611/gemini_neon_research_swarm.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/gemini_neon_research_swarm.md)
- **BEAD:** [sase-1uj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1uj/README.md)

# Plan: Gemini 4 Neon for the research swarm

## Context

Google released Gemini 4 Neon on 2026-11-18. These facts were verified while planning on
2026-11-20:

- **Exact slugs.** `agy models` (Antigravity CLI 1.5.13) lists `gemini-4-neon-high`,
  `gemini-4-neon-xhigh`, `gemini-4-neon-medium`, and `gemini-4-neon-low`. The Neon
  family comes first in CLI order, ahead of Argon and Hydrogen. Antigravity labels the
  levels Low, Medium, High (its default), and Extra; in the CLI, Extra is `-xhigh`.
  There is no bare `gemini-4-neon` slug.
- **Neon plan gating.** This comes from the Antigravity "Choose a model" docs page. Neon
  is ❌ on Free and Plus. It is ✅ on Google AI Pro ("Available with paid Google AI Pro
  plans or on trial with eligible premium devices"), Google AI Ultra, and Gemini
  Enterprise. That is the same gating as Gemini 4 Argon.
- **Hydrogen changed.** The same page now marks Gemini 4 Hydrogen ❌ on Free and Plus.
  On Google AI Pro it is ✅ only for non-trial subscriptions ("Available on Google AI
  Pro for non-trial subscriptions only"). It is ✅ on Ultra and Enterprise. Three places
  in sase are therefore stale:
  - the `models.yml` agy comment, "Hydrogen is available on every plan";
  - `docs/reference/providers/agy.md`, "It is available on every plan";
  - the same agy.md paragraph, "`-xhigh` is the only agy xhigh rung". Neon adds a second
    xhigh rung.
- **Where the macro lives.** `#research_swarm` is not a sase package macro. It ships in
  the linked `sase-research-artifacts` plugin, in
  `src/sase_research_artifacts/xprompts/research_swarm.md`. The user's `#rsa` shortcut
  in their chezmoi-managed config passes `gemini=true` without `gemini_model`, so a new
  default reaches it with no config change.
- **No ordering constraint between phases.** On today's sase, the Rust
  `classify_model_value` binding already accepts the provider-qualified
  `agy/gemini-4-neon-xhigh` (`ok: True`, kind `provider_model`). Only the bare slug is
  rejected until it is catalogued. `sase macro expand` with
  `gemini_model=agy/gemini-4-neon-xhigh` renders `%m:agy/gemini-4-neon-xhigh`. The agy
  adapter passes slugs to `--model` verbatim. So neither phase needs the other, and the
  plugin's `sase>=0.18.0` floor does not move.
- **Precedents to mirror.**
  - sase commit `cf7a09f15b`, "catalog Gemini 4 Hydrogen on Antigravity".
  - sase-research-artifacts commit `fec2ab7`, "default Gemini researcher to
    agy/gemini-4-hydrogen-xhigh".
  - Read both diffs before starting. This epic is the same shape with Neon in place of
    Hydrogen.
- **sase-core.** It holds no Gemini catalog data. The catalog stays in sase's
  `src/sase/llm_provider/models.yml`, so no Rust or binding change is needed.

Keep the research swarm on the `-xhigh` rung. Every other swarm researcher defaults to
`@xhigh`, and the Hydrogen default it replaces was `-xhigh`.

## Phase graph

`catalog` (sase repo) and `swarm` (linked `sase-research-artifacts` repo) are
independent. They touch different repositories, neither reads the other's output, and
either can land first. Both declare `depends_on: []` so they can run in parallel.

## Phase `catalog`: Catalog Gemini 4 Neon in sase's agy manifest

Work in the sase repo. Read the `tui` and `lint_and_test` reference memories (and
`tui_screenshot` before visual runs) with `/sase_memory_read` first.

1. **Reconfirm the slugs.** If `agy` is on `PATH`, run `agy models`. Confirm the four
   `gemini-4-neon-*` slugs above still appear. If they differ, stop and record what the
   CLI reports instead of guessing.
2. **`src/sase/llm_provider/models.yml`, agy provider:**
   - Insert the Neon family at the top of `models:` (CLI order, before Argon), with
     effort descending: `gemini-4-neon-xhigh`, `gemini-4-neon-high`,
     `gemini-4-neon-medium`, `gemini-4-neon-low`.
   - Add the short aliases `neon4x`, `neon4h`, `neon4m`, and `neon4l`, placed first in
     `short_aliases:`.
   - Add one `advisories:` entry per Neon slug. Copy the Argon entries' shape and
     wording exactly, changing only the model name:
     - `severity: info`
     - `label: needs Google AI Pro or Ultra`
     - `detail:` "Antigravity offers Gemini 4 Neon on paid Google AI Pro plans (or on
       trial with eligible premium devices), Google AI Ultra, and Gemini Enterprise
       subscriptions (not Free or AI Plus), and `agy` exits with an error when the
       signed-in account cannot use it. Run `agy models` to check access."
   - Rewrite the agy comment block so it stops claiming Hydrogen is available on every
     plan. It should say:
     - every Gemini 4 family is plan-gated, so the tier defaults stay on Gemini 3.8
       Flash;
     - Gemini 4 models are reachable by name;
     - bare family slugs such as `gemini-4-hydrogen` stay uncatalogued because their
       effort floats with the CLI default.
   - Leave `tiers:` and the five shipped `aliases:` unchanged.
     `test_agy_tier_mapping_never_routes_to_an_advisory_model` must keep passing, and no
     shipped size alias may route to a plan-gated model.

   > [!decision] hydrogen_advisory Also add an `advisories:` entry for each of the four
   > `gemini-4-hydrogen-*` slugs:
   >
   > - `severity: info`
   > - `label: needs paid Google AI Pro or Ultra`
   > - `detail:` "Antigravity offers Gemini 4 Hydrogen on non-trial Google AI Pro
   >   subscriptions, Google AI Ultra, and Gemini Enterprise (not Free, AI Plus, or Pro
   >   trials), and `agy` exits with an error when the signed-in account cannot use it.
   >   Run `agy models` to check access."

   > [!decision] hydrogen_advisory = no Leave Hydrogen without an advisory. Still
   > correct the prose and comment claims that it is available on every plan; they are
   > false either way.

3. **`src/sase/ace/tui/widgets/_agent_list_helpers.py`.** Add `"neon"` to the
   `short_model_name` keyword tuple, just before `"argon"`. No other catalogued model ID
   contains `neon`.
4. **Tests.** Mirror the Hydrogen commit's coverage:
   - `tests/llm_provider/test_agy_provider_core.py`: add the four slugs at the top of
     `_AGY_MODELS`, which is compared for exact equality with `llm_known_model_names()`.
     Add the four `neon4*` entries to the expected short-alias map.
   - `tests/llm_provider/test_agy_usage_probe.py`: add the four slugs to `_GEMINI_IDS`.
   - `tests/llm_provider/test_agy_usage_indicator_default.py`: add `gemini-4-neon-xhigh`
     to `_GEMINI_IDS`.
   - `tests/llm_provider/test_model_advisories.py`: add a `_NEON_SLUGS` tuple and a
     parametrized `test_agy_flags_neon_as_pro_gated`. It asserts severity `info`, the
     exact label, and that `Google AI Pro` appears in the detail.
     - Under `hydrogen_advisory = yes`: replace
       `test_agy_hydrogen_models_carry_no_advisory` with a parametrized
       `test_agy_flags_hydrogen_as_paid_pro_gated` that asserts the Hydrogen
       label/detail.
     - Under `no`: leave that test as is.
   - `tests/ace/tui/widgets/test_agent_list_helpers.py`: add
     `test_short_model_name_recognizes_gemini_neon`, asserting
     `short_model_name("gemini-4-neon-xhigh") == "neon"`.
   - `tests/test_model_picker_options.py`: assert `"gemini-4-neon-xhigh" in ids`.
   - `tests/test_model_picker_jump.py`: Neon now leads Antigravity's filtered rows. The
     first three `"gemini"`-filtered jump targets become `__default__`,
     `gemini-4-neon-xhigh`, and `gemini-4-neon-high`. Keep the comment about the newest
     Gemini slugs leading.
5. **Docs.** Follow the `docs_writing` routing.
   - Run `just fix`. It regenerates the agy rows of the model-list and short-alias
     tables in `docs/reference/providers/models.md`. Do not hand-edit generated blocks.
   - Hand-edit the prose in `docs/reference/providers/agy.md`, "Model Mapping":
     - Add a Gemini 4 Neon paragraph before the Argon paragraph. It names the four
       slugs, says effort is chosen by the slug variant, and gives Neon's plan gating
       (Pro paid or eligible trial, Ultra, eligible Enterprise; `agy` fails the run
       without access).
     - Fix the Hydrogen paragraph. Replace "It is available on every plan, and `-xhigh`
       is the only agy xhigh rung" with its real gating (non-trial Pro, Ultra,
       Enterprise; not Free or AI Plus). Add the fact that Neon and Hydrogen are the
       only agy families with an `-xhigh` rung.
     - End the closing paragraph with "no shipped size alias routes to a Gemini 4 model"
       instead of "…routes to Argon".
   - In `docs/reference/providers/models.md` (the advisory prose near "informational
     Gemini 4 Argon entitlement advisory"), name every flagged family:
     - `hydrogen_advisory = yes`: "Gemini 4 Neon, Argon, and Hydrogen entitlement
       advisories".
     - `no`: "Gemini 4 Neon and Argon entitlement advisories".
6. **PNG goldens.** Four new agy rows (and, under `hydrogen_advisory = yes`, new
   Hydrogen advisory labels) change every model-picker capture that shows the agy group.
   - In `tests/ace/tui/visual/test_ace_png_snapshots_model_picker_usage.py`, the
     `"24 models"` wait string is the agy catalog count. Update it to the new count
     (expected `"28 models"`).
   - In `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py`, the
     builtin effort-step test waits for `gemini-4-argon-high`, the first agy row the
     override picker selects. Update it to the row now selected (expected
     `gemini-4-neon-xhigh`). Confirm against the capture.
   - Run a targeted `just fix-tui-screenshots -- <selectors>` over the model-picker and
     models-panel visual files:
     - `test_ace_png_snapshots_model_picker_usage.py`
     - `test_ace_png_snapshots_models_panel*.py`
     - `test_ace_png_snapshots_config_launch.py`
     - `test_ace_png_snapshots_disabled_provider_launch.py`
   - Route that run through `/sase_monitor` with `TESTING`/`TESTED` if it may outrun the
     turn.
   - Inspect the report and every changed golden. Accept only Neon rows, the agy count,
     Hydrogen advisory labels, and scroll shifts they cause. Investigate any other
     delta, and do not accept a `partial` status as current.
7. **Verify.**
   - Run `just fix`, then `sase tool run check`.
   - `grep -rn "available on every plan" src/sase/llm_provider/models.yml docs/reference/providers/agy.md`
     must not describe Hydrogen.
   - With the workspace venv, `model_advisory_for("gemini-4-neon-xhigh")` returns the
     Neon advisory.

Do not edit `CHANGELOG.md` (release-please owns it). Do not change agy tier defaults or
shipped size-alias pools.

## Phase `swarm`: Switch the research swarm's Gemini roles to Neon

Work in the linked plugin repo. Open it with
`sase repo open sase-research-artifacts -r "<reason>"`, use only the path it prints, and
read that repo's `AGENTS.md` first. This repo's changes become a repository obligation
that host completion commits through `/sase_final`.

1. **`src/sase_research_artifacts/xprompts/research_swarm.md`.**
   - Change the `gemini_model` input default from `"agy/gemini-4-hydrogen-xhigh"` to
     `"agy/gemini-4-neon-xhigh"` (no `@effort` suffix).
   - Keep its existing effort-suffix description, and add one sentence: the Neon default
     needs a Google AI Pro (paid or eligible trial), Ultra, or Enterprise plan, so on
     other accounts pass `gemini_model=agy/gemini-3.8-flash-high`.
2. **`src/sase_research_artifacts/default_config.yml`.**

   > [!decision] image_pool Change the `image` alias's agy member from
   > `agy/gemini-4-hydrogen-xhigh` to `agy/gemini-4-neon-xhigh`, keeping the pool order
   > and the other two members.

   > [!decision] image_pool = no Leave the `image` pool on
   > `agy/gemini-4-hydrogen-xhigh`, and leave `tests/test_default_config.py` and the
   > `@image` text in `docs/configuration.md` untouched.

3. **Tests.**
   - `tests/test_macro_loading.py`: the `inputs[13].default` assertion (about line 687)
     and the `%m:agy/...` assertions in `test_research_swarm_dependency_graph_preserved`
     (about line 806) and `test_research_swarm_gemini_opt_in_adds_segment` (about lines
     1044–1045, including the `@`-suffix negative check) move to
     `agy/gemini-4-neon-xhigh`.
   - Under `image_pool = yes`, the two `tests/test_default_config.py` assertions (about
     lines 30 and 33) move as well.
4. **Docs.**
   - Replace every `agy/gemini-4-hydrogen-xhigh` default mention with
     `agy/gemini-4-neon-xhigh`:
     - `README.md`, the `gemini_model` default list;
     - `docs/macros.md`, the input table row, the effort-suffix paragraph, and the
       `<clan>.gem` step;
     - `docs/configuration.md`, the `image` pool sentence (only under
       `image_pool = yes`).
   - Add one short plan-gating note next to the gemini default in `docs/macros.md`,
     beside the existing muse `warn` advisory sentence. Neon needs Google AI Pro, Ultra,
     or Enterprise; `agy` fails the `.gem` run on other accounts; override
     `gemini_model` there. Phrase it so it holds whether or not the installed sase
     already catalogs Neon.
   - Under `image_pool = yes`, add a matching clause to `docs/configuration.md`. A
     round-robin `@image` launch that lands on the agy member fails on accounts without
     Neon, so such users override the `image` alias in their own config.
   - Do not edit `CHANGELOG.md` (release-please owns it).
5. **Verify.**
   - From inside the plugin checkout, run `just fmt` and then `sase tool run check`.
   - `grep -rni hydrogen` over the checkout (excluding `.git`) must match only
     `CHANGELOG.md`. Under `image_pool = no`, it may also match `default_config.yml`,
     `tests/test_default_config.py`, and `docs/configuration.md`.

## Out of scope

- **agy tier defaults and shipped size aliases.** They stay on Gemini 3.8 Flash. Every
  Gemini 4 family is plan-gated, and the tier mapping must never route to an advisory
  model.
- **Wider agy catalog drift.** `agy models` also reports changes this epic ignores:
  - `claude-opus-5-5-*`, `claude-sonnet-5-5-*`, and the orchestrator slugs are new;
  - Gemini 3.7/3.6/3.5 Flash and 3.1 Pro no longer appear for this account;
  - Antigravity removes `claude-sonnet-4-6`, `claude-opus-4-6-thinking`, and
    `gpt-oss-120b-medium` on 2026-11-30.

  That catalog refresh is separate follow-up work.

- **Global installs.** Reinstalling or syncing the user's editable installs is
  human-only. Both repos pick up the change once the user's primary checkouts update.
