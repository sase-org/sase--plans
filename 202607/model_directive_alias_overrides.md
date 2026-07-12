---
create_time: 2026-07-11 19:50:29
status: done
prompt: 202607/prompts/model_directive_alias_overrides.md
tier: tale
---
# Plan: Keyword-Argument Alias Overrides on the `%model` Directive

## Problem

Model aliases (`@coder`, `@phase_worker`, `@epic_creator`, custom aliases, …) are resolved from machine-global
configuration (`llm_provider.model_aliases` in sase.yml, plus the machine-global temporary-override file
`~/.sase/llm_override.json`). There is no way to pin an alias for a _single agent family_ at launch time. If a user
wants this particular plan's coder follow-up to run on Sonnet while everything else keeps the configured coder model,
they must either edit config, set a machine-global temporary override (which affects all concurrent launches), or wait
for the plan-approval modal and pick a model manually.

## Feature

Extend the `%model` / `%m` directive's paren form with keyword arguments that override specific model aliases **for the
agent family launched by this prompt**:

```
%m(opus, coder=sonnet)
```

launches the root agent on Opus, and any agent in the launched family whose model resolves through the `@coder` alias
(e.g. the plan-accept coder follow-up) resolves to Sonnet instead of the configured value. Overrides are scoped to the
family: concurrent launches and unrelated agents are unaffected, and nothing persists after the family is done.

### Grammar and accepted forms

- `%m(opus, coder=sonnet)` — positional model + one or more alias overrides.
- `%m(coder=sonnet)` — kwargs-only; the root agent's own model is unchanged (falls through to the existing no-`%model`
  default lane).
- `%m(opus@high, coder=sonnet)` — the positional keeps its existing `@effort` suffix support.
- Values follow the exact grammar of the positional model value: bare known model (`sonnet`), explicit `provider/model`,
  nested provider-local path, or an `@alias` reference (`coder=@phase_worker`). Quoting works as it does today for
  values containing spaces/parens (`coder="provider/model with spaces"`), reusing the existing `parse_args` tokenizer.
- Keys are spelled **bare** (no `@`), matching the user's example and the existing kwarg style of `%wait(time=5m)` and
  `#pr(status=ready)`. A key must be a known alias name (`model_alias_names()`): builtin roles (`default`, `coder`,
  `<provider>_coder`, `epic_creator`, `epic_lander`, `phase_worker`) and custom aliases.
- Only the paren form supports kwargs (the colon form is single-value by grammar, same as `%wait`'s `time=`).

### Validation errors (raised as `DirectiveError` at prompt-submission time, so the TUI surfaces

them before launch, like the existing `@`-spelling rule)

- Unknown key: `Unknown model alias 'foo' on %model — valid aliases: @default, @coder, ...`.
- Key spelled with `@` (`%m(opus, @coder=sonnet)`): error telling the user to drop the `@` on keys.
- Value that is a bare alias name (`coder=phase_worker`): reuse the existing "must be prefixed with @" rule from
  `_validate_model_alias_prefix`.
- Value referencing an unknown `@alias`: reuse the existing unknown-alias error.
- Value carrying an `@effort` suffix (`coder=sonnet@high`): rejected in v1 with a clear message (per-role effort
  overrides are a possible follow-up; the follow-up-effort machinery intentionally re-derives effort from the
  follow-up's own prompt today).
- Empty value (`coder=`), duplicate keys (`coder=a, coder=b`): explicit errors.
- `=` detected in the **colon** form's value (`%m:coder=sonnet`): helpful error pointing at the paren form.
- Self-referencing value (`coder=@coder`): rejected at parse time as a no-op.

Existing behaviors preserved: single `%model` per prompt, multi-positional error (`multi_model_unsupported_message` —
its positional count must keep excluding named args, which `parse_args` already separates), backtick-literal form stays
literal, `%model` fan-out still goes through `%{%m:... | %m:...}` (kwargs inside an alt branch apply to that branch's
family).

## Design overview

Three layers: **parse** → **propagate** → **resolve**.

### 1. Parse (src/sase/xprompt/)

- `_directive_types.py`: add `model_alias_overrides` to `PromptDirectives` (immutable mapping of alias → raw value;
  empty by default) and a collection field on `_CollectedDirectives`.
- `_directive_collect.py`: in the `%model` paren branch, consume `named_args` from `parse_args` (today they are silently
  dropped for `%model`; only `%wait`/`%name` consume kwargs). Mirror the `%wait` pattern: validate/collect kwargs, keep
  the single-positional rule, and allow an absent positional when kwargs are present.
- `_directive_values.py`: new validation helper alongside `_validate_model_alias_prefix` implementing the error list
  above (keys against `model_alias_names()`, values via the existing `@`-spelling and effort-suffix rules). Kwarg values
  also get `#xprompt` reference expansion like other directive args.
- `_directive_extract.py`: assemble the validated mapping into `PromptDirectives`.

### 2. Propagate (launch + family plumbing)

The override map must reach every process that resolves an alias for a family member. Stored raw (as typed, e.g.
`{"coder": "sonnet"}`), resolved lazily at each launch.

- **Root launch**: where directives are consumed at agent start (`axe/run_agent_directives.py`;
  `llm_provider/_invoke.py` for the in-process invoke path):
  - persist the map to the agent's `agent_meta.json` as a new `model_alias_overrides` field (via `update_meta_field`),
    and
  - export `SASE_MODEL_ALIAS_OVERRIDES` (JSON object) into the agent process environment, alongside the existing
    `SASE_AGENT_*` identity vars. Because in-process follow-ups (coder, epic creator, custom roles launched by
    `handle_accepted_plan`) run in the same runner process, they inherit the env var with zero extra work.
- **Meta inheritance**: add `model_alias_overrides` to the `create_followup_artifacts` inherit allow-list
  (`axe/run_agent_helpers_artifacts.py`) so every family member's meta carries it — robustness for retry/resume paths
  that re-derive env from meta, and observability.
- **External family-attach launches** (`%n(parent, suffix)`): extend `FamilyAttachLaunchPlan`
  (`agent/_family_attach_types.py`) with the override map; `resolve_family_attach_plan`
  (`agent/_family_attach_resolution.py`) reads it from the parent's `agent_meta.json` via a direct JSON read of
  `parent_artifacts_dir` (deliberately NOT via the Rust scan wire — see "Rust core boundary" below);
  `prepare_family_attach_launch` re-exports `SASE_MODEL_ALIAS_OVERRIDES` into the child env exactly as it already does
  for `SASE_AGENT_FAMILY_ATTACH` / `SASE_PLAN`.

### 3. Resolve (src/sase/llm_provider/)

- New small module (e.g. `llm_provider/launch_alias_overrides.py`): typed accessor for the active launch-scoped
  overrides — reads `SASE_MODEL_ALIAS_OVERRIDES` from the environment (cached per process, mirroring
  `_active_alias_overrides()`), plus a scoped way for launcher-side code to supply the map explicitly (parameter, not
  `os.environ` mutation — the ACE TUI is long-lived and must not leak overrides across launches).
- `config.py resolve_model_alias`: at each alias hop, consult launch-scoped overrides **before** the machine-global
  temporary-override short-circuit and configured aliases. When an override fires, continue the resolution loop with its
  value; the existing `seen` set + depth limit guarantee termination for `@alias`-valued overrides.
- **Provider-coder shadowing**: a `coder=` override must also intercept `<provider>_coder` hops (unless the map contains
  the more specific `<provider>_coder` key). Without this, `%m(opus, coder=sonnet)` silently does nothing whenever the
  user has `claude_coder` configured, because the plan-accept follow-up emits `%model:@claude_coder` and the configured
  hop would win before ever reaching `@coder`. This mirrors the existing `_ROLE_ALIAS_FALLBACKS` chain
  (`<provider>_coder` → `coder` → `default`).
- **`default=` key**: supported uniformly through the resolver (any `@default` hop picks it up), and the no-`%model`
  default lane (`resolve_effective_default_provider_model`) checks the launch-scoped `default` override first when one
  is active in the process.
- `registry.resolve_model_provider` gains optional explicit-override threading for the few launcher-side call sites that
  resolve before the env var exists (launch preview, follow-up meta resolution).

### Key decisions (called out for review)

1. **Precedence: launch-scoped beats machine-global.** Order per hop: family override → temporary override
   (`~/.sase/llm_override.json`) → configured alias → role fallback. The family override is the most specific, explicit
   user intent for exactly this family.
2. **Provider-coder shadowing** as described above — the feature is near-useless without it for anyone with
   `<provider>_coder` configured.
3. **Explicit approval-modal / `sase plan approve --model` choices still win** — they emit a concrete model (bypassing
   alias hops) or an `@alias` the user picked deliberately (which then resolves _through_ the family override —
   consistent and desirable).
4. **Scope of "family"**: the root agent itself plus all plan-chain follow-ups (coder, questions/feedback, epic creator,
   custom roles) and `%n(parent, suffix)` attach launches. **Out of scope**: phase workers / epic landers launched later
   by `sase bead work` — those are independent launches in new processes whose models already have a dedicated per-bead
   surface (`sase bead create --model`); wiring family overrides across that boundary can be a follow-up if wanted.

### Rust core boundary

No `sase-core` changes in v1. The entire model-alias subsystem (`resolve_model_alias`, temporary overrides) lives
Python-side today, so extending it here doesn't newly cross the boundary. The new `agent_meta.json` field is additive;
the Rust artifact scanner ignores fields it doesn't project, and the one cross-process consumer (family attach) reads
the parent meta JSON directly. If/when the alias resolver migrates into `sase_core_rs`, or another frontend needs the
field via the scan wire (`AgentMetaWire`), that promotion is a separate change in `../sase-core`.

## UX surfaces

- **Directive completion** (`ace/tui/widgets/directive_completion.py`):
  - hint `_DIRECTIVE_ARGUMENT_HINTS["model"]` → `":model or (model, alias=model)"`;
  - admit `model` in the paren-completion branch (currently hard-restricted to `wait`);
  - kwarg-aware token scanning mirroring `_extract_wait_paren_arg_token`;
  - key completion from `model_alias_names()`, value completion reusing the existing model catalog
    (`xprompt/model_completion.py`), keeping the `@effort`-after-`@` carve-out working for the positional.
- **Launch preview** (`agent/launch_preview.py`): render overrides next to the model line, e.g.
  `Model: CLAUDE(opus)  ·  alias overrides: coder → sonnet`.
- **Doctor** (`doctor/checks_config_xprompts.py`): validate `%model` kwargs (keys and values) embedded in xprompt files
  the same way bare `%model` tokens are validated today.
- **Directive docs**: update the `%model` docstring surfaces (`xprompt/directives.py`, `extract_prompt_directives`
  docstring, completion description).

## Testing

- **Parsing** (`tests/test_directives_extract.py` + siblings): every accepted form and every error case listed above;
  kwargs with quoted values; kwargs-only form; kwargs inside `%{...}` alt branches (`test_directives_split_models*.py`);
  positional-count check unaffected by kwargs; `@effort` on positional still works alongside kwargs
  (`test_directives_effort.py`).
- **Resolver** (new/extended llm_provider config tests): override hit on direct hop; chained `@alias` override values;
  provider-coder shadowing (both with and without a configured `claude_coder`, and with a specific `claude_coder=` key
  present); precedence over temporary overrides; `default=` in both the resolver and the no-`%model` lane; cycle/depth
  safety.
- **Propagation**: env var + meta persistence at root launch; `create_followup_artifacts` inheritance; plan-accept coder
  follow-up resolves through the override (`run_agent_exec_plan_accept` tests) including `_resolve_model_meta` output;
  family-attach plan carries and re-exports the map; approval-picker explicit model still wins.
- **Completion + preview**: `tests/ace/tui/widgets/test_directive_completion_*`,
  `tests/test_xprompt_model_completion.py`, launch-preview tests.

Run the standard gates (`just check`) plus targeted pytest subsets for the touched areas.

## Explicitly excluded

- `memory/xprompts.md` documents the `%model` directive in its table; memory files must not be edited without explicit
  user permission granted in-conversation, so this plan does **not** include that edit. The user should update it (or
  explicitly authorize the update) once the feature lands.
- Per-role effort overrides (`coder=sonnet@high`), bead-work phase-worker/epic-lander propagation, TUI indicator/panel
  for active family overrides, and Rust-core wire promotion — all noted above as possible follow-ups.
