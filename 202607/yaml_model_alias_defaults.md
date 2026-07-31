---
tier: tale
title: Make shipped model alias defaults editable from one YAML file
goal:
  SASE's shipped implicit model alias defaults (targets, role fallbacks, effort overlays, and descriptions) come from a
  single bundled YAML file, so changing them requires no Python edit and drift from that file fails a test.
proposed_by: bbugyi200.athena.qq
create_time: 2026-07-31 16:08:02
status: wip
---

- **PROMPT:** [202607/prompts/yaml_model_alias_defaults.md](prompts/yaml_model_alias_defaults.md)

# Plan: YAML-Driven Default Model Alias Configuration

## Problem

SASE's _shipped_ defaults for implicit model aliases are hardcoded Python constants in
`src/sase/llm_provider/model_alias_policy.py`:

- `SMARTEST_MODEL_ALIAS_DEFAULT`, `CHEAP_MODEL_ALIAS_DEFAULT`, `CHEAPER_MODEL_ALIAS_DEFAULT`,
  `CHEAPEST_MODEL_ALIAS_DEFAULT` (concrete/selector targets)
- `ROLE_ALIAS_FALLBACKS` (the `@<alias>` each role falls back to)
- `MEDIUM_PHASE_WORKER_MODEL_ALIAS_EFFORT` / `MEDIUM_PHASE_WORKER_MODEL_ALIAS_DEFAULT` (effort overlay)
- `ROLE_ALIAS_DESCRIPTIONS` (Models-panel / completion descriptions)

A user can already override any of these in one YAML (`llm_provider.model_aliases.builtin` in
`~/.config/sase/sase.yml`), but the maintainer cannot change what SASE _ships_ without editing Python and then
hand-syncing four other places that restate the same values:

1. `src/sase/llm_provider/model_alias_policy.py` — the real source of truth
2. `src/sase/default_config.yml` lines ~739-769 — a commented-out block that restates every default
3. `docs/llms.md` and `docs/configuration.md` — tables restating the same target strings
4. `tests/llm_provider/test_config_role_aliases.py` and `tests/llm_provider/test_alias_view.py` — literal assertions

Goal: one bundled YAML file is the single edit point for shipped alias defaults, and drift from it fails a test.

## Approach

Add `src/sase/llm_provider/model_alias_defaults.yml` as the shipped-defaults data file and turn `model_alias_policy.py`
into a thin loader over it.

Deliberately **not** chosen: activating the commented `llm_provider.model_aliases.builtin` block in
`default_config.yml`. That block merges into the same layer users write to, so every implicit alias would start
reporting `configured=True` / `configured_source="builtin"` and lose its implicit-vs-configured distinction in
`alias_view.py` and the ACE Models panel. It also cannot carry per-alias `description` values (the schema types
`builtin` entries as plain strings), and `default` has no concrete implicit value at all — it resolves through the
configured/autodetected provider's tier default, which a config entry would silently replace.

Alias _name_ constants (`SMARTEST_MODEL_ALIAS_NAME`, `PROVIDER_CODER_ALIAS_SUFFIX`, ...) stay in Python. They are
identifiers referenced across ACE, axe, and bead lifecycle code, not configuration.

## Steps

### 1. Add the defaults YAML

Create `src/sase/llm_provider/model_alias_defaults.yml`. It ships automatically — `pyproject.toml` packages `src/sase`
wholesale (`[tool.hatch.build.targets.wheel] packages = ["src/sase"]`), so no packaging change is needed.

Shape (one entry per implicit alias, keys matching the `*_MODEL_ALIAS_NAME` constants):

```yaml
# Shipped defaults for SASE's implicit model aliases. This file is the single
# edit point for changing what @smartest, @cheap, @medium_phase_worker, ...
# resolve to out of the box. User config (llm_provider.model_aliases.builtin)
# still overrides anything here.
schema_version: 1
aliases:
  default:
    description: >-
      Model used when a prompt has no %model directive; every other alias ultimately falls back to it.
  coder:
    fallback: "@default"
    description: >-
      Coder follow-up agents launched from plans (fallback for every <provider>_coder alias).
  medium_phase_worker:
    fallback: "@default@high"
    description: "Medium bead phase agents that implement directly."
  smartest:
    target: "claude/claude-fable-5 || codex/gpt-5.6-sol"
    description: >-
      Highest-capability model used automatically by extra-large phase agents and large epic landers.
  cheap:
    target: "claude/opus@medium | codex/gpt-5.5"
    description: "Load-balanced pool used automatically by small phase agents."
  # ... one entry per alias currently in ROLE_ALIAS_FALLBACKS / IMPLICIT_ALIAS_TARGETS
```

Rules, enforced by the loader:

- Each entry carries at most one of `fallback` (an `@<alias>` reference, optionally with a trailing `@<effort>`) or
  `target` (a concrete model, `provider/model`, `A | B` pool, or `A || B` ordered fallback).
- `default` carries neither — it keeps resolving through the provider tier default in `resolve_default_alias_target`.
- `description` is required for every entry (`ROLE_ALIAS_DESCRIPTIONS` currently covers all of them).
- The `medium_phase_worker` effort overlay lives inside its `fallback` string (`"@default@high"`) rather than in a
  separate key, so the existing `split_model_effort` grammar stays the only parser.

Port every current value verbatim so this step is behavior-neutral.

### 2. Load the YAML in `model_alias_policy.py`

Replace the four `*_MODEL_ALIAS_DEFAULT` constants, `MEDIUM_PHASE_WORKER_MODEL_ALIAS_EFFORT`, `ROLE_ALIAS_FALLBACKS`,
`IMPLICIT_ALIAS_TARGETS`, and `ROLE_ALIAS_DESCRIPTIONS` with a cached loader plus accessors:

```python
@functools.cache
def _load_model_alias_defaults() -> _ModelAliasDefaults: ...

def role_alias_fallbacks() -> Mapping[str, str]: ...
def implicit_alias_targets() -> Mapping[str, str]: ...
def role_alias_descriptions() -> Mapping[str, str]: ...
```

- Read with `importlib.resources.files("sase.llm_provider").joinpath("model_alias_defaults.yml")`, following the
  existing bundled-resource pattern in `src/sase/fakey/scenario.py:85` and `src/sase/config/loading.py:75`.
- Return `MappingProxyType` views so callers cannot mutate the cache; membership tests (`bare in ...`) and `.get()` used
  by current callers keep working unchanged.
- This file ships with the package, so a missing/malformed file is an installation defect, not user error: raise
  `RuntimeError` naming the resource path and the specific problem (unknown alias key, both `fallback` and `target` set,
  missing `description`, non-string value). Do not silently degrade to an empty mapping — that would reroute every role
  alias.
- Validate that YAML keys are exactly the set of `*_MODEL_ALIAS_NAME` constants declared in the module, so adding a role
  constant without a YAML entry (or vice versa) fails fast.
- Keep all `*_MODEL_ALIAS_NAME` constants and `PROVIDER_CODER_ALIAS_SUFFIX` as Python literals.
- Keep the module docstring's explanation of the fallback chain, updated to point at the YAML.

### 3. Update consumers

- `src/sase/llm_provider/model_alias_resolution.py` — the direct dict references at lines ~168-169, ~202-203, ~331, and
  ~432 become accessor calls. Bind the mapping once per `resolve()` call rather than calling the accessor inside the
  resolution loop.
- `src/sase/llm_provider/model_alias_config.py` — imports at lines 14-17 and uses at lines ~211-212, ~240, ~252-253,
  ~279, ~296 become accessor calls.
- `src/sase/llm_provider/config.py` — drop the `*_MODEL_ALIAS_DEFAULT` and `MEDIUM_PHASE_WORKER_MODEL_ALIAS_*`
  re-exports (lines ~145-165); re-export the three new accessors alongside the retained `*_NAME` constants. These value
  constants have no consumers outside `model_alias_policy.py`, `config.py`, and tests, so removing them is clean — check
  `sase/memory/symvision.md` guidance before adding any pragma.
- `src/sase/llm_provider/alias_view.py` needs no change: it already goes through `implicit_model_alias_value` /
  `implicit_model_alias_fallback*` / `model_alias_description`.

### 4. Update tests

- `tests/llm_provider/test_config_role_aliases.py` and `tests/llm_provider/test_alias_view.py` currently assert the
  literal default strings. Rewrite the assertions that duplicate values to compare against the YAML-backed accessors,
  keeping at least one test that pins the _shape_ of each default (e.g. `@smartest` is a `||` ordered fallback, `@cheap`
  is a `|` pool, `@medium_phase_worker` carries `high` effort) so a bad edit still fails.
- Add `tests/llm_provider/test_model_alias_defaults.py` covering: every name constant has a YAML entry and vice versa;
  `fallback` and `target` are mutually exclusive; `default` has neither; every entry has a description; each `target`
  parses under `parse_model_alias_selector`; each `fallback` resolves to a known alias name.
- Other files matching the default strings (`tests/test_config_schema_models.py`,
  `tests/llm_provider/test_load_balanced_aliases.py`, `tests/doctor/test_checks_config_model_aliases.py`,
  `tests/test_llm_provider_invoke.py`, `tests/test_reasoning_effort_metadata_display.py`, the ACE Models-panel PNG
  fixtures) use those strings as arbitrary user-config fixtures, not as assertions about shipped defaults. Leave them
  alone.

### 5. Remove the restated defaults and add drift guards

- `src/sase/default_config.yml` — the commented `model_aliases` block (lines ~739-769) should keep the grammar
  explanation, the `custom`/`buckets` examples, and the list of overridable builtin alias names, but stop restating the
  shipped target values. Point readers at `sase/llm_provider/model_alias_defaults.yml` for current defaults.
- `src/sase/config/sase.schema.json` needs no change — its `model_aliases.builtin` description enumerates alias names
  and grammar, not default values.
- `docs/llms.md` (alias table ~line 689-717 and examples ~566-572, ~731-737) and `docs/configuration.md` (~971-974,
  ~1018-1024) restate the target strings. Update them to the current values and add
  `tests/llm_provider/test_model_alias_defaults_docs_sync.py`: for every YAML `target`, assert the string appears in
  both docs files, applying the markdown table's `|` → `\|` escaping when matching. Four strings, and a stale-doc
  failure names the exact file to fix.

### 6. Verify

```bash
just install
just check
```

`just install` first — workspace virtualenvs are ephemeral and may be stale.

## Acceptance Criteria

- Editing only `src/sase/llm_provider/model_alias_defaults.yml` changes what `@smartest`, `@cheap`, `@cheaper`,
  `@cheapest`, `@medium_phase_worker`, and every role fallback resolve to, plus their descriptions, with no Python edit.
- With the YAML values unchanged from today's constants, behavior is identical: same resolutions, same
  implicit-vs-configured classification in `alias_view.py`, same ACE Models-panel rows and descriptions.
- A malformed or incomplete defaults YAML raises a clear `RuntimeError` naming the file and the problem rather than
  silently emptying the role fallback map.
- Adding a role alias name constant without a matching YAML entry fails a test.
- Changing a YAML target without updating `docs/llms.md` / `docs/configuration.md` fails a test.
- `just check` passes.

## Out of Scope

- Provider tier defaults (`_MODEL_TIERS` in `src/sase/llm_provider/claude.py:24-25`, and the equivalent in `codex.py`)
  that back `@default` when no `default` alias is configured. Those belong to provider plugins and are already
  overridable through `llm_provider.models`. Worth a follow-up bead if the maintainer wants them in the same file.
- Any change to user-facing override precedence: `llm_provider.model_aliases.builtin` and `custom` keep winning over the
  shipped defaults, and temporary overrides keep winning over both.
- The Rust core (`../sase-core`): it references alias _names_ in plan validation and LSP hover text but holds none of
  these default values, so no cross-repo change is required.
