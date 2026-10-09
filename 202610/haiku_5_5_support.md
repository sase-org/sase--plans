---
tier: tale
title: Add Claude Haiku 5.5 and move @xsmall onto it
goal:
  claude-haiku-5-5 works on every model surface and heads the shipped @xsmall pool in
  place of Haiku 4.5.
size: small
decisions:
  retire_haiku_45:
    ask: Should claude-haiku-4-5 also be removed from the built-in Claude catalog?
    choices:
      keep:
        Keep it pickable by name with its haiku45 short alias; only @xsmall moves to
        Haiku 5.5
      retire:
        Drop it from the catalog and usage labels; pinned configs lose picker/completion
        rows
    default: keep
    why:
      Haiku 4.5 is still served; keeping it costs one picker row and protects configs
      that pin it
    answer: keep
proposed_by: bbugyi200.athena.0yr
decided_by: auto
create_time: 2026-10-08 21:11:00
status: wip
---

# Add Claude Haiku 5.5 and move `@xsmall` onto it

## Goal

Make Claude Haiku 5.5 (`claude-haiku-5-5`) a first-class built-in Claude model in sase
across every surface: `%model` resolution, the coder model picker, `%m`/LSP model
completion, agent-name provider/model suffixes, the Agents-tab short model label, Claude
usage-window attribution, and the generated docs. Replace Haiku 4.5 with Haiku 5.5 in
the shipped `@xsmall` size alias.

## Background (verified while planning)

- `src/sase/llm_provider/models.yml` is the single edit point for built-in model data.
  `claude.py` reads its catalog (`llm_known_model_names`), short aliases
  (`llm_model_short_aliases`), and tiers from this manifest. The picker, Models panel,
  `%model`/`%m` completion (Python and the Rust `sase_core` completion engine, which
  gets the catalog over the wire), LSP/nvim completion, the doctor checks, and the
  agent-name suffixes all derive from it. **No `sase-core` change is needed**: the Rust
  crate has no shipped Claude catalog, and its only Claude model literals are test
  fixtures.
- The current Claude catalog is `opus`, `sonnet`, `haiku`, `claude-haiku-4-5`,
  `claude-fable-5`, with short aliases `claude-haiku-4-5 → haiku45` and
  `claude-fable-5 → fable`. Tiers stay `large: opus`, `small: sonnet`. These are
  floating CLI aliases and are not part of this change.
- The `@xsmall` target is currently
  `claude/claude-haiku-4-5@xhigh | codex/gpt-6-luna@medium | agy/gemini-3.8-flash-high | muse/muse-spark-1.3-contributor@medium`.
- Size-alias effort-ladder rule (`decisions:size-alias-effort-ladder`, enforced by
  `src/sase/llm_provider/model_policy.py` / `just model-policy-check`): a model's first
  appearance, scanning `@xlarge` down, runs at `xhigh`. Haiku 5.5 first appears in
  `@xsmall`, so it is `@xhigh`. Haiku 5.5 supports effort `low`–`max` (API default
  `medium`). Haiku 4.5 never had real effort support, so `@xhigh` now does something. No
  `supersedes` link is needed: that field only continues an effort descent, and Haiku
  4.5 will appear in no alias.
- The tables in `docs/llms.md` between `<!-- BEGIN GENERATED: ... -->` markers
  (`model-alias-defaults`, `known-models`, `model-short-aliases`) are rendered by
  `tools/render_model_docs` (run by `just fix`). `just check` fails via `fmt-docs-check`
  if they are stale. Never hand-edit those blocks.
- `src/sase/llm_provider/usage/_claude_support_windows.py` maps Claude `/usage` row
  labels to per-model weekly window keys via `_MODEL_LABELS`. `_model_id_from_label`
  returns the **first** entry with any alias that occurs as a whole word in the
  normalized label (`"Haiku 5.5"` normalizes to `"haiku 5 5"`). Today the 4.5 entry owns
  bare `"haiku"`, so a `Current week (Haiku 5.5)` row would be misattributed to
  `weekly:claude-haiku-4-5`.
- `short_model_name()` in `src/sase/ace/tui/widgets/_agent_list_helpers.py` already maps
  any id containing `haiku` to `haiku`. It needs a regression test, not a code change.
- The user's chezmoi-managed SASE config does not override `@xsmall` or pin Haiku, so
  the shipped default change takes effect for them directly.
- Size-alias tests (`tests/llm_provider/test_load_balanced_alias_defaults.py`,
  `test_model_alias_defaults.py`, provider-routing tests) resolve against frozen fixture
  aliases (`tests/_model_alias_defaults_fixture.py`), not the shipped values, and
  contain no Haiku literals. Visual PNG fixtures build their own alias views and do not
  render the shipped `@xsmall` target. Expect neither to need edits.

## Changes

### 1. Manifest: `src/sase/llm_provider/models.yml`

- `providers.claude.models`: insert `claude-haiku-5-5` immediately after `haiku` and
  before `claude-haiku-4-5`. The list is picker/completion order, newest Haiku first.
- `providers.claude.short_aliases`: add `claude-haiku-5-5: haiku55` above the existing
  `claude-haiku-4-5: haiku45` line.
- `aliases.xsmall.target`: replace `claude/claude-haiku-4-5@xhigh` with
  `claude/claude-haiku-5-5@xhigh`. Leave the other three members, their order and
  efforts, and the description unchanged.
- Do not touch `tiers`, other aliases, or other providers. Do not add `supersedes`.

> [!decision] retire_haiku_45 = retire Also remove `claude-haiku-4-5` from
> `providers.claude.models` and its `haiku45` short alias. In step 2, drop the Haiku 4.5
> `_MODEL_LABELS` entry instead of narrowing it. Then `Current week (Haiku 4.5)` falls
> through to the generic `window:<slug>` key. Make the step 2 test assert that, rather
> than `weekly:claude-haiku-4-5`. In step 4, remove the `claude-haiku-4-5` assertions
> and replace them with
> `resolve_model_provider("claude-haiku-4-5") == (None, "claude-haiku-4-5")`, mirroring
> the existing retired `claude-opus-5` assertion. Assert that `"claude-haiku-4-5"` is
> absent from the short-alias map and the picker ids. Leave
> `tests/agent_scan_golden/fixture_builder.py` and
> `tests/test_running_agents_snapshot_all.py` alone: their dated
> `claude-haiku-4-5-20251001` strings are recorded transcript data, not catalog entries.

### 2. Usage-window attribution: `src/sase/llm_provider/usage/_claude_support_windows.py`

Rewrite the Haiku rows of `_MODEL_LABELS` so explicit versions win and bare `haiku`
means the current Haiku:

```python
("claude-fable-5", ("claude fable 5", "fable 5", "fable")),
("claude-haiku-4-5", ("claude haiku 4 5", "haiku 4 5")),
("claude-haiku-5-5", ("claude haiku 5 5", "haiku 5 5", "haiku")),
("sonnet", ("sonnet",)),
("opus", ("opus",)),
```

The 4.5 entry must stay **before** the 5.5 entry. The bare `"haiku"` alias would
otherwise also match `"haiku 4 5"` labels. A short comment saying that order matters is
appropriate.

Add a new focused test module, `tests/llm_provider/test_claude_usage_model_windows.py`.
Do not grow `tests/llm_provider/test_claude_usage.py`, which is already over the
500-line `toobig` limit. Use `parse_usage_windows` from
`sase.llm_provider.usage._claude_support`, and check these cases:

- `Current week (Haiku 5.5): 40% used · resets Sep 12 at 8pm (America/New_York)` → key
  `weekly:claude-haiku-5-5`, applicability
  `{"kind": "models", "model_ids": ["claude-haiku-5-5"]}`.
- `Current week (Haiku 4.5): …` → `weekly:claude-haiku-4-5`. Under `retire`, this is the
  generic `window:` key instead.
- `Current week (Haiku): …` → `weekly:claude-haiku-5-5`.
- A combined probe text with all-models, Fable, Haiku 5.5, and Haiku 4.5 rows yields
  distinct keys and no parse diagnostic. Under `retire`, drop the 4.5 row from this
  case.

Mirror the row format and `observed_at` style used in `test_claude_usage.py`.

### 3. Config grammar example: `src/sase/default_config.yml`

In the commented `model_aliases.builtin` grammar example (around line 1720), change
`xsmall: "claude/claude-haiku-4-5 | codex/gpt-6-luna@medium"` to use
`claude/claude-haiku-5-5`. If `src/sase/config/sase.schema.json` or any config docs
mirror that example text, update them identically. A `git grep -n "claude-haiku-4-5"`
after the change should show only intended leftovers: the `keep` catalog entry, its
short alias, the usage label, and its tests, plus the dated transcript fixtures noted
above.

### 4. Catalog, picker, and display tests

- `tests/test_llm_provider_core.py`:
  - In `test_resolve_model_provider_implicit_mapping`, add
    `resolve_model_provider("claude-haiku-5-5") == ("claude", "claude-haiku-5-5")`.
  - In `test_model_short_alias_map_contains_claude_entries`, add
    `aliases.get("claude-haiku-5-5") == "haiku55"`.
- `tests/test_model_picker_options.py`:
  - In `test_build_model_options_has_known_models`, assert `"claude-haiku-5-5" in ids`.
  - Generalize `test_model_picker_claude_h4_5_row_includes_alias` into a Claude
    point-model row test whose `expected` covers `claude-haiku-5-5 → haiku55` and, under
    `keep`, `claude-haiku-4-5 → haiku45`. Rename it accordingly, for example
    `test_model_picker_claude_point_model_rows_include_alias`. Under `keep`, also assert
    that the 5.5 row's index precedes the 4.5 row's index.
- `tests/ace/tui/widgets/test_agent_list_helpers.py`: add a test that
  `short_model_name("claude-haiku-5-5") == "haiku"`.
- Do **not** add a test that pins the shipped `@xsmall` value, and do not edit
  `tests/_model_alias_defaults_fixture.py`. That fixture deliberately freezes its own
  alias values so that shipped-value changes in `models.yml` need no test edits. The
  shipped value is already guarded by `just model-policy-check` and the generated
  `model-alias-defaults` docs block that `fmt-docs-check` verifies.

### 5. Regenerate generated docs

Run `just fix`. It runs `tools/render_model_docs`, ruff format, prettier, and
keep-sorted. Confirm the diff to `docs/llms.md` contains exactly these generated-block
updates:

- `model-alias-defaults`: the `@xsmall` row now starts with
  `claude/claude-haiku-5-5@xhigh`.
- `known-models`: the claude row lists `claude-haiku-5-5`.
- `model-short-aliases`: the claude row includes `claude-haiku-5-5` → `haiku55`.

No hand-written prose in `docs/llms.md`, `docs/ace.md`, or `README.md` names Haiku 4.5
or the shipped `@xsmall` target. The `claude/haiku@minimal` lines in `docs/ace.md` and
`docs/llms.md` are user-config grammar examples using the floating alias. Leave them
unchanged.

## Verification

1. `just fix`, then confirm the generated-doc diff described in step 5.
2. Targeted tests:
   `.venv/bin/pytest tests/test_llm_provider_core.py tests/test_model_picker_options.py tests/ace/tui/widgets/test_agent_list_helpers.py tests/llm_provider/test_claude_usage_model_windows.py tests/llm_provider/test_claude_usage.py tests/llm_provider/test_model_alias_defaults.py tests/llm_provider/test_load_balanced_alias_defaults.py tests/llm_provider/test_model_policy.py tests/llm_provider/test_model_manifest.py tests/test_macro_model_completion_catalog.py -q`
3. `just model-policy-check` must pass. The new `@xsmall` head is Haiku 5.5's first
   appearance at `xhigh`.
4. `sase tool run check`, the agent-default gate per the `lint_and_test` memory note. Do
   **not** run `just check-full`. If `check` reports a stale visual golden or a
   picker/completion snapshot that legitimately renders the real catalog, update only
   that golden with `just fix-tui-screenshots` scoped to the failing test, and note it.

## Out of scope

- Opus 5.5 and Sonnet 5.5 are already reached through the floating `opus`/`sonnet`
  aliases, so tier and alias defaults stay as they are.
- Claude Fable 5.1 (`claude-fable-5-1`) is not in the catalog. Adding it is a separate
  change.
- No `sase-core`, `sase-nvim`, `sase-telegram`, or chezmoi edits.
