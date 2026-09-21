---
tier: tale
title: Add an opt-in Gemini Flash 3.8 researcher to research_swarm
goal:
  "#research_swarm accepts gemini=true to launch a fifth independent researcher on
  agy/gemini-3.8-flash-high (off by default), fully wired into the lead's waits, report
  suffixes, provider gating, tests, and docs."
size: small
proposed_by: bbugyi200.apollo.1e
create_time: 2026-09-21 08:02:59
status: wip
---

# Add an opt-in Gemini Flash 3.8 researcher to `#research_swarm`

## Goal

Add a fifth independent researcher to the `#research_swarm` xprompt swarm, backed by
Gemini 3.8 Flash through SASE's Antigravity (`agy`) provider. It is **disabled by
default** (like grok and muse) and enabled per invocation with `gemini=true`, e.g.
`#research_swarm(gemini=true): compare approaches`.

## Where the work lives

All changes are in the **`sase-research-artifacts`** linked repo (not the primary `sase`
repo). Open it with the `/sase_repo` skill
(`sase repo open sase-research-artifacts -r "..."`) and work only in the printed path.
Because that repo is modified, it becomes a commit obligation in the final declaration.

Relevant files there:

- `src/sase_research_artifacts/xprompts/research_swarm.md` — the swarm xprompt.
- `src/sase_research_artifacts/default_config.yml` — `researchers` bucket description.
- `tests/test_xprompt_loading.py` — swarm rendering/graph tests.
- `README.md`, `docs/xprompts.md`, `docs/configuration.md`, `AGENTS.md` — docs.
- `CHANGELOG.md` is release-please managed; do **not** edit it by hand.

## Key facts established during planning

- Gemini models are served by the `agy` provider (`src/sase/llm_provider/agy.py` in the
  `sase` repo). `gemini-3.8-flash-high` is a known model slug (short alias `flash38h`),
  and SASE's own `@xsmall` alias already uses `agy/gemini-3.8-flash-high`.
- **`agy` rejects an explicit reasoning effort** (`effort_cli_args` with an empty
  supported map raises `LLMInvocationError` for an explicit `@effort`). So unlike the
  other researcher defaults, the Gemini default must **not** carry an `@xhigh` suffix;
  the effort level is encoded in the model slug (`-high`). Default model:
  `agy/gemini-3.8-flash-high`.
- Provider gating uses the provider _name_, so the gate is `"agy" | provider_enabled`
  (not `"gemini"`).

## Naming decisions

- Toggle input: `gemini` (bool, default `false`). The user-facing concept is "the Gemini
  researcher"; `agy` is an opaque CLI name. The existing toggles happen to equal
  provider names, but here the model family is the clearer knob.
- Model input: `gemini_model` (word, default `"agy/gemini-3.8-flash-high"`).
- Agent / report short suffix: `gem` → agent `research.{@1}.gem`, report `__gem.md`,
  `#research(suffix=gem)`. Three letters, matching `cdx`/`cld`/`grk`/`mus`.
- Ordering: gemini comes directly after muse everywhere (input list, the `researchers`
  Jinja list, authored segments, docs tables and prose).

## Implementation steps

### 1. `research_swarm.md`

1. Frontmatter `input:`
   - Insert after `muse`:
     ```yaml
     - name: gemini
       type: bool
       default: false
       description: Request the gemini (Antigravity) researcher.
     ```
   - Insert after `muse_model`:
     ```yaml
     - name: gemini_model
       type: word
       default: "agy/gemini-3.8-flash-high"
       description:
         Model for `<clan>.gem`. Antigravity (`agy`) rejects explicit `@effort`
         suffixes; choose effort through the model slug (`-high`/`-medium`/`-low`).
     ```
2. Extend the `researchers` list at the top of the body with:
   ```jinja
   + ([{"short": "gem", "provider": "agy", "model": gemini_model}] if gemini and ("agy" | provider_enabled) else [])
   ```
3. Add a new researcher segment between the `mus` segment and the lead segment, a
   verbatim copy of the `mus` segment with these substitutions:
   - `%if(should_run={{ gemini and ("agy" | provider_enabled) }}) %id(gem, clan=research.{@1})`
   - `%m:{{ gemini_model }}`
   - `rejectattr("short", "equalto", "gem")`
   - `You are researcher gem ...`, `__gem.md` (both occurrences),
     `{{ prompt }} #research(suffix=gem)`
   - Keep the `{% if wait %}%wait:...` and the full `%q(w=0.25...)` directive exactly as
     the other researchers have them (tests assert the exact queue template on every
     authored segment).
   - The segment is separated by `---` lines like the others.
4. The lead segment needs no structural edits: its waits, peer list, suffix list, and
   final-layout tree are all derived from `researchers`, so the gem entry flows through
   automatically. Confirm by rendering (see Verification).

### 2. `default_config.yml`

Update the `researchers` bucket description's provider list to
`(codex, claude, grok, muse, gemini)`. Do not add a new model alias; the Gemini
researcher uses a concrete model default like the other researchers.

### 3. `tests/test_xprompt_loading.py`

Update existing tests for the new segment/inputs and add coverage:

- `test_research_swarm_declares_typed_input`: new expected order
  `... ("muse","bool"), ("gemini","bool"), ("codex_model","word"), ..., ("muse_model","word"), ("gemini_model","word"), ("lead_model","word"), ("should_generate_image","bool")`;
  re-index the default assertions, adding `gemini` default `False` and `gemini_model`
  default `"agy/gemini-3.8-flash-high"`.
- `test_research_swarm_has_six_top_level_segments` → rename to
  `..._has_seven_top_level_segments` and assert 7.
- Every test unpacking `_authored_swarm_segments()` positionally
  (`test_research_swarm_dependency_graph_preserved`,
  `..._lead_mentions_artifact_read_derivation`,
  `..._lead_lists_wait_artifacts_not_transcripts`) must account for the `gem` segment.
  In the dependency-graph test, add assertions:
  `'%if(should_run={{ gemini and ("agy" | provider_enabled) }})'`,
  `"%id(gem, clan=research.{@1})"`, `"%m:{{ gemini_model }}"`, no `%clan(`, and include
  `gem` in the per-segment `priority is not none` / single `%q(` /
  `_WEIGHTED_QUEUE_TEMPLATE` checks.
- `test_research_swarm_defaults_to_three_expanded_agents`: also assert no `%id(gem,`
  segment.
- `test_research_swarm_custom_models_route_to_matching_roles_only`: add
  `"gemini": "true"` and `"gemini_model": "@gemini_custom"`; unpack six segments and
  extend the cross-contamination assertions.
- `test_research_swarm_all_researchers_off_yields_lead_only`: pass `"gemini": "false"`
  too and assert no `%wait:research.{@1}.gem`.
- `test_research_swarm_reports_use_provider_suffixes`: enable gemini and assert
  `#research(suffix=gem)`.
- New `test_research_swarm_gemini_opt_in_adds_segment`: `gemini=true` yields 4 segments;
  the gem segment has `%id(gem, clan=research.{@1})`, `%m:agy/gemini-3.8-flash-high`,
  **no** `@` effort suffix on that model (e.g. assert
  `"agy/gemini-3.8-flash-high@" not in gem`), `some topic #research(suffix=gem)`,
  `3-researcher swarm` (default codex + claude plus gem), and the lead has
  `%wait:research.{@1}.gem` and `__gem.md` in its layout; call
  `_assert_each_segment_has_one_queue(segments)` (this also runs
  `plan_typed_launch_units`, proving the directives parse). Also check all five
  researchers together (`grok`, `muse`, `gemini` true) produce 6 segments.
- New `test_research_swarm_disabled_agy_drops_gemini_segment`:
  `disable_provider("agy", 900.0, source="test")`, render with `gemini=true`, assert no
  `%id(gem,` and no `%wait:research.{@1}.gem` on the lead, mirroring
  `test_research_swarm_disabled_provider_drops_segment`.

### 4. Docs

- `docs/xprompts.md` `#research_swarm` section: add `gemini` (bool, `false`, "Request
  the gemini (Antigravity) researcher") and `gemini_model` (word,
  `agy/gemini-3.8-flash-high`, "Model for `<clan>.gem`; no `@effort` suffix") rows to
  the input table (keep the table aligned); update prose: "up to seven authored segments
  (five researchers, the lead, the image agent)", `grok=true` / `muse=true` /
  `gemini=true` opt-ins, "turning all five off", "six model inputs"; note that `agy`
  rejects explicit `@effort` so the effort is chosen via the model slug; add a numbered
  role entry for `<clan>.gem` after `<clan>.mus` and renumber the lead (6) and image
  (7). Fix the capacity note ("four quarter-weight members fit in it") so it stays
  accurate — with all five researchers plus the lead, six quarter-weight members need
  `capacity` ≥ 2 to run concurrently.
- `README.md`: add `__gem` to the `__cdx`/`__cld`/`__grk`/`__mus` draft-suffix lists
  (research ref provider and research-highlights sections); in the `#research_swarm`
  bullet say "up to five per-provider independent researchers", add `gemini=true`
  opt-in, add `gemini_model` / `.gem` / `agy/gemini-3.8-flash-high` to the model input
  sentence; in "Defaults" include gemini in the per-provider list.
- `docs/configuration.md`: add `gemini_model` to the per-invocation override list.
- `AGENTS.md`: no content change needed unless it enumerates researchers (it does not
  today); leave it alone otherwise.

### Out of scope (do not change)

- `provider.py`'s `research-highlights` `agent_name_globs` only excludes
  `research.*.cld` / `research.*.cdx` (not grk/mus either); draft reports are already
  excluded by the `!20*/*/*__*.md` path glob, so `__gem.md` drafts are covered. Do not
  widen the agent globs in this change.
- No changes to the primary `sase` repo or `sase-core`.

## Verification

In the `sase-research-artifacts` checkout:

1. `just check` (ruff + mypy + pytest) must pass.
2. Sanity-render the swarm with gemini on and eyeball the gem segment and the lead's
   waits/layout, e.g. via a short Python snippet using the test helpers'
   `expand_single_xprompt(xp, ["topic"], {"gemini": "true"}, preserve_segment_separators=True)`;
   confirm the gem segment's model line is exactly `%m:agy/gemini-3.8-flash-high` and
   the lead lists `__gem.md` in the final layout.
3. Do not launch a live swarm.

Commit (via the host finalizer / `sase_git_commit` flow as instructed) with a
conventional message such as
`feat(research): add opt-in gemini researcher to research swarm`.
