---
tier: epic
title: Google Gemini 4 Argon and Hydrogen support
goal: 'Gemini 4 Hydrogen shows up as a catalogued Antigravity model on every manifest-driven
  sase model completion surface. Argon''s entitlement advisory and the agy tier defaults
  are correct again. #research_swarm''s Gemini researcher defaults to agy/gemini-4-hydrogen-xhigh.

  '
phases:
- id: agy-hydrogen-catalog
  title: Catalog Gemini 4 Hydrogen on Antigravity (sase)
  depends_on: []
  size: medium
  description: 'agy-hydrogen-catalog: in the sase repo, add the four Hydrogen slugs
    and their hydrogen4* aliases to the agy section of models.yml. Retext the stale
    Argon advisory to Pro/Ultra and move the broken agy tiers to Gemini 3.8 Flash.
    Teach short_model_name about hydrogen. Update the pinned tests, regenerate and
    hand-edit the agy docs, refresh any shifted ACE PNG goldens, and record the five
    research follow-ups.'
- id: research-swarm-hydrogen
  title: Default the swarm's Gemini researcher to Hydrogen (sase-research-artifacts)
  depends_on:
  - agy-hydrogen-catalog
  size: small
  description: 'research-swarm-hydrogen: in the linked sase-research-artifacts plugin,
    change the #research_swarm gemini_model default and the @image pool''s agy member
    to agy/gemini-4-hydrogen-xhigh. Update the input description, README, docs, and
    pinned tests, then verify with the plugin''s sase tool run check.'
proposed_by: bbugyi200.athena.1cb
proposed_with: claude/opus
create_time: 2026-11-13 18:46:09
status: done
bead_id: sase-1ss
---

- **PROMPT:** [prompts/202611/argon_hydrogen_support.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/argon_hydrogen_support.md)
- **BEAD:** [sase-1ss](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ss/README.md)

# Google Gemini 4 Argon and Hydrogen support

## Goal

Make Google's Gemini 4 Hydrogen models first-class Antigravity (`agy`) models in sase,
so every manifest-driven model completion surface offers them. Bring the existing Gemini
4 Argon entries up to date, and switch the `#research_swarm` macro's Gemini researcher
from `agy/gemini-3.8-flash-high` to `agy/gemini-4-hydrogen-xhigh`.

## Context

Design of record:
`research:202611/argon_hydrogen_completion_and_swarm/argon_hydrogen_completion_and_swarm.md`
(read it with `sase artifact read <ref> "<reason>"`). The user accepted every
requirement it recommends. Facts the phases rely on:

- **Argon is already catalogued; Hydrogen is not.** `src/sase/llm_provider/models.yml`
  (agy section) lists `gemini-4-argon-{high,medium,low}` with `argon4h/m/l` short
  aliases. Live `agy models` (agy 1.5.8, checked 2026-11-13) also serves
  `gemini-4-hydrogen-{xhigh,high,medium,low}`. These are Antigravity CLI slugs. Effort
  is part of the slug, and Hydrogen's `-xhigh` is the first agy slug at the xhigh rung.
- **The manifest is the single edit point for nearly every completion surface.**
  `models.yml` → `model_manifest.py` → `AgyProvider` hooks (`llm_known_model_names`,
  `llm_model_short_aliases`, `llm_model_advisories`) → the registry, which feeds all of
  these:
  - ACE prompt `%model:` / `%m:` completion
  - macro `type: model` arguments, including `gemini_model=`
  - the concrete `==model` shortcut and the Rust provider-scoped filters
  - ACE model picker rows, jump hints, and advisory badges
  - the Rust macro LSP's JSON catalog, used by editors such as sase-nvim
  - bare-name resolution and the model-validity snapshot
  - the generated docs tables

  sase-core has no hard-coded Gemini catalog IDs. The agy usage probe matches on the
  `gemini-` prefix. So there is **no sase-core change and no `sase-core-revision.txt`
  ratchet**. sase-nvim, sase-telegram, and sase-github have no hard-coded model lists.

- **Surfaces the manifest does not reach:**
  - `short_model_name()` in `src/sase/ace/tui/widgets/_agent_list_helpers.py` uses a
    hand-kept keyword tuple. Without `"hydrogen"` in it, Hydrogen rows render as
    `gemini`.
  - The hand-written Argon prose in `docs/reference/providers/agy.md`.
  - Shell `--model` completion (`src/sase/completion/candidates/catalog_build.py`). By
    test-pinned design it lists only the five size aliases and no concrete model for any
    provider, so Hydrogen is treated exactly like every Claude and Codex model there.
    Per the research, changing that design is a deferred feature follow-up, not part of
    this epic.
- **Entitlements, from Google's current Antigravity docs table:**
  - Hydrogen is on every plan (Free, AI Plus, Pro, Ultra, Enterprise) with no footnote,
    so it gets **no advisory**.
  - Argon needs paid Google AI Pro (or a trial on eligible premium devices), Ultra, or
    Enterprise. Its current "needs Google AI Ultra … (not … AI Pro)" advisory is stale.
  - Gemini 3.8 Flash is on every plan. 3.7/3.6/3.5 Flash and 3.1 Pro are Enterprise-only
    now.
- **The agy tier defaults are broken today.** `tiers: large: gemini-3.7-flash-high`,
  `small: gemini-3.7-flash-low`. A live probe shows `agy` rejects
  `gemini-3.7-flash-high` on this account, so every launch that resolves the bare `agy`
  tier fails. The fix is to move the tiers to `gemini-3.8-flash-{high,low}`.
- **`#research_swarm` lives in the linked `sase-research-artifacts` plugin**
  (`src/sase_research_artifacts/xprompts/research_swarm.md`), not in this repo. The
  Gemini researcher is opt-in (`gemini: false`) and reads its model from the
  `gemini_model` input (default `agy/gemini-3.8-flash-high`). The plugin's
  `default_config.yml` `@image` pool also has `agy/gemini-3.8-flash-high` as its agy
  member. The user's chezmoi snippets call `#research_swarm` without overriding
  `gemini_model`, so the plugin default is the only value that needs to change.

## Phase graph

```text
agy-hydrogen-catalog (sase, medium)
        │
        ▼
research-swarm-hydrogen (sase-research-artifacts, small)
```

The phases are sequential. The plugin builds without the sase change, because sase
accepts any `agy/<slug>` value and `agy` validates the slug at launch, so the plugin
needs **no sase version floor**. But the plugin default should only ship once sase's
catalog lists Hydrogen, so that completion rows, picker rows, and docs show the new
default as a known model.

## Phases

### Phase `agy-hydrogen-catalog`: catalog Gemini 4 Hydrogen on Antigravity (sase)

- **id:** `agy-hydrogen-catalog`
- **size:** `medium`
- **depends on:** none
- **repo:** this sase repo only

Steps:

1. **`src/sase/llm_provider/models.yml`, agy section:**
   - Insert the four Hydrogen slugs directly after `gemini-4-argon-low` and before
     `gemini-3.8-flash-high`, highest effort first: `gemini-4-hydrogen-xhigh`,
     `gemini-4-hydrogen-high`, `gemini-4-hydrogen-medium`, `gemini-4-hydrogen-low`.
     Argon stays first. `tests/test_model_picker_jump.py` pins
     `__default__, gemini-4-argon-high, gemini-4-argon-medium` for the `gemini` filter,
     and that pin must stay green unchanged.
   - Add short aliases `hydrogen4x`, `hydrogen4h`, `hydrogen4m`, `hydrogen4l`, placed
     after the `argon4*` aliases. This follows the `<family><gen><level-initial>`
     pattern, with `x` for xhigh. The manifest loader requires aliases to be unique, and
     none of these collide.
   - Add **no** advisory for any Hydrogen slug.
   - Rewrite all three Argon advisories. Keep `severity: info`. Set
     `label: needs Google AI Pro or Ultra`. The `detail` should say that Antigravity
     offers Gemini 4 Argon on paid Google AI Pro plans (or on trial with eligible
     premium devices), Google AI Ultra, and Gemini Enterprise, but not Free or AI Plus;
     that `agy` exits with an error when the signed-in account cannot use it; and to run
     `agy models` to check access.
   - Change `tiers` to `large: gemini-3.8-flash-high` and `small: gemini-3.8-flash-low`.
   - Rewrite the agy comment block above `models:`. The current text says "keep CLI
     ordering" (Argon already breaks that, since the CLI prints medium first) and "Argon
     is Ultra-gated". Make it say:
     - families appear in CLI order, with effort levels descending within a family;
     - Hydrogen is available on every plan, and Argon needs Pro or Ultra;
     - tier defaults sit on Gemini 3.8 Flash because it is the long-established
       all-plans model (3.7 Flash and older are Enterprise-only);
     - do not catalog the bare `gemini-4-hydrogen` slug, because its effort level floats
       with the CLI default.

     Make the same correction to the "keep Antigravity's CLI order" phrase in the
     file-header Layout comment.

   - Do **not** touch the stale 3.7/3.6/3.5 Flash, 3.1 Pro, Claude 4.6, or
     `gpt-oss-120b-medium` rows. Do not change any `aliases:` size-alias pool; the
     `@xsmall` agy member stays `agy/gemini-3.8-flash-high`. Do not add a `supersedes`
     link. Each of these is a separate follow-up (see the last step).

2. **`src/sase/ace/tui/widgets/_agent_list_helpers.py`:** add `"hydrogen"` to the
   `short_model_name()` keyword tuple next to `"argon"`.
3. **Tests.** Update the pins, and add coverage for the new rows:
   - `tests/llm_provider/test_agy_provider_core.py`:
     - add the Hydrogen slugs, in manifest order, to `_AGY_MODELS`;
     - add the `hydrogen4*` entries to the alias-map assertion;
     - move the tier assertions to 3.8 Flash. This covers `resolve_model_name()` /
       `"large"` / `"small"`, the `--model` value for the large-tier and small-tier
       invokes, and the `not in cmd` assertion in the model-override test.

     Leave the `resolve_model_provider("…3.7-flash…")` assertions alone, because those
     rows remain catalogued.

   - `tests/test_agy_integration_polish.py`: change `_AGY_LARGE`/`_AGY_SMALL` to the 3.8
     Flash slugs and the alias assertions to `flash38h`/`flash38l`.
   - `tests/llm_provider/test_model_advisories.py`:
     - rename and retext the Argon test (e.g. `test_agy_flags_argon_as_pro_gated`) so it
       asserts the new label and that the detail mentions Google AI Pro;
     - add a `_HYDROGEN_SLUGS` tuple and a test that asserts Hydrogen carries no
       advisory.

     `test_agy_tier_mapping_never_routes_to_an_advisory_model` must stay green, because
     3.8 Flash carries no advisory.

   - `tests/llm_provider/test_agy_usage_probe.py`: add the Hydrogen slugs to
     `_GEMINI_IDS`. Also add one Hydrogen id to `_GEMINI_IDS` in
     `tests/llm_provider/test_agy_usage_indicator_default.py`, so the `gemini-`-prefix
     usage-window match is exercised for the new family.
   - `tests/test_model_picker_options.py`: assert `gemini-4-hydrogen-xhigh` and
     `gemini-4-argon-high` are picker ids.
   - `tests/ace/tui/widgets/test_agent_list_helpers.py`: add
     `short_model_name("gemini-4-hydrogen-xhigh") == "hydrogen"`.
   - Leave `tests/doctor/test_checks_providers_default.py` unchanged; it builds its own
     payload fixture.

4. **Docs.** Read the `docs_writing` reference memory first
   (`sase memory read docs_writing.md -r "<why>"`).
   - Run `just fix` to regenerate `docs/reference/providers/models.md` and the generated
     `model-tiers-agy` table in `docs/reference/providers/agy.md`.
   - Then hand-edit the prose after that table in `agy.md`:
     - Argon is catalogued for Google AI Pro (paid or eligible trial), Ultra, and
       Enterprise accounts.
     - Hydrogen is catalogued as `gemini-4-hydrogen-xhigh/-high/-medium/-low`, available
       on every plan, with effort chosen by slug and `-xhigh` as the only agy xhigh
       rung.
     - Explain why the tiers sit on Gemini 3.8 Flash.
     - Replace the `%m:agy/gemini-3.6-flash-high` example in that section, which is now
       Enterprise-only, with a Hydrogen slug.
5. **TUI PNG goldens.** New picker and completion rows, and the retexted Argon advisory
   badge, can shift ACE PNG goldens. These include the models-panel, model-picker, and
   model-completion snapshots under `tests/ace/tui/visual/`.
   - Read the `tui` reference memory and its `tui_screenshot` child first.
   - Run a targeted `just fix-tui-screenshots -- <model-related selectors>` through
     `/sase_monitor` using the `TESTING`/`TESTED` pair.
   - Inspect every created and updated golden in the run report before keeping it.
   - If the run reports `partial`, resolve the skipped nodes. Do not treat it as clean.
6. **Verify.** Run `just fix`, then `sase tool run check`. Never run raw `just check`,
   and do not run `check-full`. If `agy` is installed, optionally sanity-check
   `agy models | grep hydrogen`. Do not launch paid live agy agent runs as a gate.
7. **Follow-ups.** Record these as `PROPOSED FOLLOW-UP:` notes on the phase bead rather
   than filing beads:
   1. feature: `agy --effort` passthrough. Upstream agy 1.5.8 accepts
      `--effort low|medium|high|xhigh|max` with base slugs. This needs per-model level
      maps (Argon has no xhigh), base slugs in the catalog, a CLI version gate, and a
      decision record that partly supersedes `size-alias-effort-ladder`.
   2. bug: refresh agy catalog drift.
      - Add the Claude Opus/Sonnet 5.5 slugs that agy now serves.
      - Prune or mark the Enterprise-only 3.7/3.6/3.5 Flash and 3.1 Pro rows.
      - Remove Claude 4.6 and GPT-OSS after their 2026-11-18 upstream removal.
   3. feature: list concrete catalog models in shell `--model` completion.
      - Project the packaged `models.yml` through `importlib.resources`; the fast path
        must not import `sase.llm_provider`.
      - Keep the five size aliases, and offer bare and `agy/`-qualified forms.
      - Update `model_source_path()` for cache invalidation.
      - Reclassify `--next-model` as `ValueKind.MODEL`.
   4. bug: migrate stale `agy/gemini-3.7-flash-high` pins in the following, plus a
      chezmoi task for `m_agy*`, `m_swarm`, and `tmux_ai_window`:
      - `sase/macros/reads.md` (pinned by `tests/test_macro_swarm_local_helpers.py`);
      - `docs/reference/macros/directive-syntax.md` and other doc examples;
      - the tmux agy `model` example in `src/sase/default_config.yml` and
        `src/sase/config/sase.schema.json`.
   5. policy: once a Free account confirms access, promote Hydrogen into the agy tiers
      and/or the size-alias ladder. Also decide whether to catalog Carbon and
      Orchestrator (`gemini-pro-orchestrator-g1-pro`).

Done when:

- `sase tool run check` passes.
- Hydrogen appears in `%m:` completion, the model picker, and the generated docs.
- `short_model_name` renders `hydrogen`.
- The Argon advisory reads "needs Google AI Pro or Ultra".
- `AgyProvider().resolve_model_name()` returns `gemini-3.8-flash-high`.
- Any golden changes have been inspected and committed.

### Phase `research-swarm-hydrogen`: default the swarm's Gemini researcher to Hydrogen (sase-research-artifacts)

- **id:** `research-swarm-hydrogen`
- **size:** `small`
- **depends on:** `agy-hydrogen-catalog`
- **repo:** the linked `sase-research-artifacts` plugin. Open it with your `/sase_repo`
  skill (`sase repo open sase-research-artifacts -r "<why>"`), work only in the printed
  path, and read its `AGENTS.md` first.

Steps:

1. **`src/sase_research_artifacts/xprompts/research_swarm.md`:**
   - Change the `gemini_model` default to `"agy/gemini-4-hydrogen-xhigh"`. This matches
     the other researchers, which all default to their provider's top rung.
   - Change the description to: "Model for `<clan>.gem`. SASE's `agy` adapter rejects
     explicit `@effort` suffixes; choose effort through the model slug
     (`-xhigh`/`-high`/`-medium`/`-low`)."
   - Keep the `gemini` / `gemini_model` names, the `.gem` identity, and the `__gem.md`
     suffix. Hydrogen is still a Gemini model, so renaming them would be churn.
2. **`src/sase_research_artifacts/default_config.yml`:** in the `@image` pool, change
   the agy member `agy/gemini-3.8-flash-high` to `agy/gemini-4-hydrogen-xhigh`. Leave
   the other members unchanged.
3. **Docs and tests.** Replace the old default and update the surrounding wording, which
   should now name the `-xhigh` slug and attribute the `@effort` rejection to SASE's agy
   adapter. The files are:
   - `README.md`
   - `docs/configuration.md`
   - `docs/macros.md` (the input table row, the effort paragraph, and the researcher
     list)
   - `tests/test_macro_loading.py` (the input default and the rendered `%m:` segment
     assertions)
   - `tests/test_default_config.py` (the pool string and member assertions)

   Keep the explicit-override assertions and the "no `@` suffix on agy members"
   assertions. Do **not** edit `CHANGELOG.md`; release-please manages it.

4. **Verify.** Run `sase tool run check` from inside the plugin checkout. Raw
   `just check` is refused there too. `sase macro show research_swarm` reflects the
   globally installed plugin. Agents must not reinstall it (global install is
   human-only), so an old default there is expected and not a failure.
5. The plugin repo becomes a commit obligation in this turn's final declaration, so
   commit it through `/sase_final`.

Done when:

- The plugin's `sase tool run check` passes.
- `research_swarm.md`, `default_config.yml`, the docs, and the tests all name
  `agy/gemini-4-hydrogen-xhigh`.
- No `agy/gemini-3.8-flash-high` remains in the plugin outside `CHANGELOG.md`.

## Out of scope

- Shell `--model` completion of concrete models (deferred design; follow-up 3).
- `agy --effort` passthrough and the bare `gemini-4-hydrogen` slug (follow-up 1).
- Orchestrator (`gemini-pro-orchestrator-g1-pro`) and Carbon cataloguing (follow-up 5).
- Pruning stale agy catalog rows, and adding agy's Claude 5.5 slugs (follow-up 2).
- Promoting Hydrogen into tiers or size aliases (follow-up 5; governed by
  `decisions:size-alias-effort-ladder`).
- Stale 3.7-Flash pins in macros, doc examples, config examples, and chezmoi (follow-up
  4).
- Any sase-core change or `sase-core-revision.txt` ratchet.

## Deployment notes for the user

- Install sase before the plugin, so the new swarm default shows as a catalogued model
  everywhere it is displayed. Global installs are human-only.
- After installing, restart long-running ACE, editor, and macro-LSP processes. The LSP
  serves a catalog snapshot taken at launch and will not see the Hydrogen rows until it
  restarts.
