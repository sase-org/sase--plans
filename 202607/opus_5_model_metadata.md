---
tier: tale
title: Register the Claude 5 model family and correct stale Opus version text
goal: Explicit Claude 5 model ids (claude-opus-5, claude-sonnet-5, claude-haiku-4-5)
  resolve to the claude provider, user-facing Opus version text is accurate, and the
  Antigravity catalog keeps working unchanged.
create_time: 2026-07-25 06:52:25
status: wip
---

- **PROMPT:** [202607/prompts/opus_5_model_metadata.md](prompts/opus_5_model_metadata.md)

# Register the Claude 5 model family and correct stale Opus version text

## Goal

Make Opus 5 a first-class, explicitly-resolvable model in sase, and correct the user-facing text that still names a
superseded Opus version — without breaking the Antigravity provider's model catalog.

## Background

Investigation of the current tree turned up three facts that shape the scope.

**1. `%m:opus` already runs Opus 5. No change is required for that.**

`ClaudeCodeProvider` maps tiers to bare CLI aliases and passes them straight through:

- `src/sase/llm_provider/claude.py:23-26` — `_TIER_TO_MODEL = {"large": "opus", "small": "sonnet"}`
- `src/sase/llm_provider/claude.py:252` — `--model <alias>` is handed to the `claude` CLI verbatim

sase never pins an Opus point version, so the `claude` CLI resolves `opus` to whatever the current Opus is, which is now
Opus 5. Verified in the workspace venv:

```
'opus' -> ('claude', 'opus')
```

**2. The only literal "Opus 4.6" text in the repo belongs to Antigravity, not to Anthropic's `opus` alias.**

Three sites, all mirroring the same catalog:

- `src/sase/llm_provider/agy.py:283-284` — `llm_known_model_names()` entries
- `src/sase/llm_provider/agy.py:297-298` — `llm_model_short_aliases()` (`sonnet46t`, `opus46t`)
- `docs/llms.md:778,802` — hand-maintained tables mirroring the two hooks
- `tests/llm_provider/test_agy_provider_core.py:46,108,112` — assertions on those exact strings

These strings are passed verbatim to `agy --model` (`agy.py` already carries a comment saying so). Running `agy models`
on this host returns:

```
claude-sonnet-4-6
claude-opus-4-6-thinking
```

Antigravity offers no Opus 5. Renaming these to "Opus 5" would make every `%m:opus46t` /
`%m("agy/Claude Opus 4.6 (Thinking)")` launch fail. **These stay as they are.**

**3. There is a real gap: explicit Claude 5 model ids do not resolve to the claude provider.**

`llm_known_model_names()` is `["opus", "sonnet", "haiku", "claude-fable-5"]` (`src/sase/llm_provider/claude.py:78`).
Anything not in that list falls back to the default provider with a warning. Verified:

```
'claude-opus-5'            -> (None, 'claude-opus-5')
'claude-sonnet-5'          -> (None, 'claude-sonnet-5')
'claude-haiku-4-5-20251001'-> (None, 'claude-haiku-4-5-20251001')
'claude-fable-5'           -> ('claude', 'claude-fable-5')
```

So `%m:claude-opus-5` silently routes to the default provider instead of Claude. Registering the Claude 5 family closes
that gap and is the concrete, non-breaking way to land "Opus 5 support".

The precedent for exactly this change is commit `7c7a5c6a4` ("feat: add Claude Fable 5 model metadata"), which touched
`claude.py`, `_agent_list_helpers.py`, and three test files.

## Scope

### 1. Register Claude 5 family model ids (`src/sase/llm_provider/claude.py`)

Extend `llm_known_model_names()` to:

```python
return [
    "opus",
    "sonnet",
    "haiku",
    "claude-opus-5",
    "claude-sonnet-5",
    "claude-haiku-4-5",
    "claude-fable-5",
]
```

Keep `opus`, `sonnet`, and `haiku` first and unchanged — they are the version-agnostic aliases every existing prompt,
config, and xprompt uses, and they must keep resolving exactly as they do today.

Extend `llm_model_short_aliases()` to:

```python
return {
    "claude-opus-5": "opus5",
    "claude-sonnet-5": "sonnet5",
    "claude-haiku-4-5": "haiku45",
    "claude-fable-5": "fable",
}
```

Short aliases are display-only (agent-name suffixes and model-picker filter terms); they are not resolvable as `%model`
values. `docs/llms.md` already states this — do not change that contract.

Leave `_TIER_TO_MODEL` alone. Tier mapping must stay on the floating `opus` / `sonnet` aliases so sase keeps tracking
the CLI's current model without a code change on the next release.

### 2. Correct the stale docstring (`src/sase/ace/tui/thinking/parser.py:135`)

`_extract_thinking_blocks` says "Claude Opus 4.7 encrypts extended-thinking content server-side". This is now the
behavior of the whole Opus 4.7-and-later family, including Opus 5. Reword to name the family rather than a single
superseded version, e.g. "Claude Opus 4.7 and later encrypt extended-thinking content server-side". No behavior change —
the parser already keys off the presence of a `signature` field, not off a model name.

### 3. Refresh the docs (`docs/llms.md`)

- **Automatic Provider Resolution table (line ~778)** — update the `claude` row to list the new known model names
  alongside the existing aliases.
- **Model Short Aliases table (line ~802)** — update the `claude` row with the three new shorthands.
- **Model Mapping section (line ~178)** — add a sentence noting that `opus` / `sonnet` are floating CLI aliases that
  resolve to the provider's current model (Opus 5 today), which is why sase does not pin a point version.

Do **not** touch the `agy` or `opencode` rows.

### 4. Tests

Mirror the Fable 5 precedent:

- `tests/test_llm_provider_core.py` — assert `resolve_model_provider("claude-opus-5") == ("claude", "claude-opus-5")`
  (and the same for `claude-sonnet-5`, `claude-haiku-4-5`); assert the three new entries in the short-alias map. Keep
  the existing `claude-fable-5` assertions.
- `tests/ace/tui/widgets/test_agent_list_helpers.py` — assert `short_model_name("claude-opus-5") == "opus"`.
  `short_model_name` matches by substring (`src/sase/ace/tui/widgets/_agent_list_helpers.py:13`), so `opus` is already
  the expected label and no source change is needed there — the test pins that.
- `tests/test_model_picker_modal.py` — extend the picker coverage the way `7c7a5c6a4` did, so the new ids show up as
  pickable options.

Add a regression test asserting `resolve_model_provider("opus") == ("claude", "opus")` if one does not already exist, so
a future catalog edit cannot silently break the floating alias.

## Out of scope

- **Renaming Antigravity's catalog entries.** See Background item 2. `Claude Opus 4.6 (Thinking)` and
  `Claude Sonnet 4.6 (Thinking)` must stay byte-identical to what `agy --model` accepts.
- **The separate `agy` catalog drift.** `agy models` on this host now emits slug-style ids (`claude-opus-4-6-thinking`,
  `gemini-3.6-flash-high`) rather than the display names `agy.py` hardcodes, and it lists Gemini 3.6 models sase does
  not know about. That is a real staleness bug, but it is an Antigravity concern unrelated to Opus 5 — file it
  separately rather than folding it in here.
- **`src/sase/llm_provider/opencode.py`** (`anthropic/claude-opus-4-5`) — OpenCode's own catalog, same reasoning.
- **Changing `_TIER_TO_MODEL` or any `%model` alias defaults** in `default_config.yml` / `model_alias_policy.py`. They
  are alias-based and already correct.

## Verification

1. `just install` (workspace venvs are ephemeral; dependencies may be stale).
2. `just check`.
3. Confirm resolution manually:

   ```bash
   .venv/bin/python -c "
   from sase.llm_provider.registry import resolve_model_provider
   for m in ['opus', 'sonnet', 'haiku', 'claude-opus-5', 'claude-sonnet-5', 'claude-haiku-4-5']:
       print(m, resolve_model_provider(m))
   "
   ```

   `opus` must still be `('claude', 'opus')`; `claude-opus-5` must now be `('claude', 'claude-opus-5')`.

4. Confirm the Antigravity path is untouched: `grep -n "Opus 4.6" src/sase/llm_provider/agy.py` still returns the two
   original lines, and `tests/llm_provider/test_agy_provider_core.py` passes unmodified.
