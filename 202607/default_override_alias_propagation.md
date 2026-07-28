---
tier: tale
title: Propagate a default temporary override through model alias resolution
goal: An active temporary override on the `default` alias wins wherever `@default`
  resolves — explicitly and at every nested hop — so aliases such as `@medium_phase_worker`,
  `@smart`, and `@coder` follow it, with launch-scoped overrides and outer effort
  suffixes keeping their existing precedence.
create_time: 2026-07-25 11:17:43
status: wip
---

- **PROMPT:** [202607/prompts/default_override_alias_propagation.md](prompts/default_override_alias_propagation.md)
- **AGENTS:**
  - [bbugyi200.athena.km.f0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.km.f0/README.md)
  - [bbugyi200.athena.km.f0--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.km.f0.md#member-code)
- **COMMITS:**
  - [a0b40ef](https://github.com/sase-org/sase/commit/a0b40ef37c1ec7afd8581fb31dd38ff0ca372937) — fix: propagate default overrides through alias resolution

# Plan: Propagate a `default` temporary override through alias resolution

## Problem

In the reported screenshot a `default` temporary override to `claude/opus@xhigh` is active (gold top-bar pill, `8h58m`
left), yet the Models panel's `phase_worker` bucket still shows the _un-overridden_ default:

```
role   medium_phase_worker   CODEX(gpt-5.6-sol) @ high   implicit → @default @ high
role   large_phase_worker    CODEX(gpt-5.6-sol)          implicit → @smart
```

Both rows resolve through `@default` (medium via the `@default@high` overlay shipped in `e0f310d8`, large via `@smart` →
`@default`), so both should read `CLAUDE(opus)`. `@xlarge_phase_worker` in the same screenshot _does_ follow its
`@smartest` override, which is the same feature working correctly for a non-`default` alias.

The cause is a deliberate carve-out, not an accident:

- `src/sase/llm_provider/model_alias_resolution.py:186` guards the temporary-override branch with
  `if known_alias and bare != DEFAULT_MODEL_ALIAS_NAME:`, so `@default` is the one alias whose override is never
  consulted during resolution.
- `resolve_default_alias_target` (same file, lines 53-69) documents this: "Temporary overrides are intentionally not
  consulted: an explicit `@default` reference means 'the configured default'".
- The carve-out predates per-alias overrides. Commit `9f933053e` generalized a single global override into a per-alias
  map and explicitly preserved "the existing default-lane behavior", where the override only won the no-`%model` launch
  slot.

Every _other_ alias override already "takes effect wherever that alias is resolved"
(`src/sase/llm_provider/temporary_override.py:20-22`). This plan makes `default` obey that same rule.

## Behavioral contract

1. An active temporary override on `default` wins wherever `@default` resolves: at the top level (`%model:@default`,
   `resolve_model_alias("@default")`) and at every nested hop — `@coder`, `@epic_lander`, `@smart`,
   `@medium_phase_worker` (`@default@high`), `@large_phase_worker` (via `@smart`), `<provider>_coder` (via `@coder`),
   and any configured/custom alias whose value references `@default`.
2. Precedence is unchanged and becomes uniform across all aliases: launch-scoped `%model(default=…)` beats the
   machine-wide temporary override, which beats a configured `llm_provider.model_aliases.builtin.default`, which beats
   the provider tier default. The launch-scoped win must hold at nested hops too, not just at the top level.
3. Effort precedence is unchanged: an outer or earlier effort still wins over the override's stored effort. With
   `default` overridden to `claude/opus@xhigh`, `@medium_phase_worker` resolves to `claude/opus` at **high** (its own
   `@default@high` overlay wins), `@smart` resolves at **xhigh** (it inherits the override's effort), and `@default@low`
   resolves at **low**.
4. Unchanged: explicit concrete `%model` targets, an explicit `provider_name=` argument to `invoke_agent()`,
   already-running agents, `SASE_MODEL_TIER_OVERRIDE`, and the no-`%model` launch lane (which already applied the
   override and keeps returning the same answer).
5. Selector-owning aliases (`@smartest`, `@cheap`, `@cheaper`, `@cheapest`) pin concrete provider/model targets and
   never reference `@default`, so a `default` override still does not move them or their dependents
   (`@xsmall_phase_worker`, `@small_phase_worker`, `@xlarge_phase_worker`, `@big_epic_lander`). This is correct, but it
   is now the only asymmetry left, so it must be stated in the docs.

### Scope decision to confirm at review

This makes an explicit top-level `@default` follow the override as well, not only nested references. The narrow
alternative — nested-only — would require the resolver to distinguish "first hop" from "later hop" and would leave
`@smart` and `@default` disagreeing about what "the default" is, while keeping the second-truth machinery in `AliasView`
that this plan deletes. The uniform rule is recommended; if the narrow variant is wanted instead, the implementation
differs materially and the plan should be revised rather than adapted mid-implementation.

### Rust core boundary

No `sase-core` change. Verified in the linked checkout: there is no alias resolution, no `model_aliases` config model,
and no `llm_override.json` reader in `crates/sase_core`. `crates/sase_core/src/effort.rs` mirrors only the effort
vocabulary and `split_model_effort`; `crates/sase_core/src/plan/validate.rs` only _names_ the five phase-size aliases in
its schema description. Alias policy, resolution, override state, and presentation are Python-resident.

## Remove the carve-out in the resolver

In `src/sase/llm_provider/model_alias_resolution.py`:

- Change the guard at line 186 from `if known_alias and bare != DEFAULT_MODEL_ALIAS_NAME:` to `if known_alias:`. The
  existing branch body already returns `f"{override.provider}/{override.model}"` with `effort or override_effort` and
  the current `selector_owner`, which satisfies contract items 1-3 with no new plumbing. `known_alias` (line 169)
  already includes `bare == DEFAULT_MODEL_ALIAS_NAME`.
- Keep the branch exactly where it is: below the launch-override branch (lines 174-183), which is what keeps
  `%model(default=…)` winning at every hop, and above the configured/implicit lookups (lines 200-204), which is what
  makes the override beat a configured `builtin.default`. Keep the "A temporary override suspends selector behavior for
  that alias" comment, and extend it to say the rule now covers `default`.
- Rewrite `resolve_default_alias_target`'s docstring (lines 53-60): it is now reached only when `default` is neither
  overridden nor configured; drop the "Temporary overrides are intentionally not consulted" paragraph and the "an
  explicit `@default` reference means 'the configured default'" claim.
- `resolve_model_alias`'s docstring (lines 306-317) already describes the intended ladder ("Launch overrides win first,
  followed by temporary overrides, configured aliases, and implicit role fallbacks"); leave it, since it becomes true
  for `default` as well.

Nothing else in this module changes. `validate_model_alias_selector_value` is a static-config check that never consulted
overrides, and the cycle/depth guards are untouched because the override branch is terminal.

## Give `@default` one meaning in the registry

After the resolver change, `registry.resolve_default_alias_provider_model` and
`resolve_default_alias_provider_model_with_effort` (`src/sase/llm_provider/registry.py:358` and `:393`) would answer
inconsistently: their _configured_ branch delegates to `resolve_model_provider("@default")` and would pick the override
up, while their unconfigured branch returns the provider tier default and would not. Close that:

- Consult the active `default` override at the top of both functions and return `(override.provider, override.model)` /
  `(override.provider, override.model, override.effort)` when one is active. Import `get_active_alias_override` lazily
  from `.temporary_override` inside the function, matching the existing lazy import at `registry.py:484` that avoids the
  import cycle.
- Update both docstrings: the current "This is the no-`%model` launch default, minus any temporary override" sentence is
  exactly the claim that stops being true. Note that `model_tier` is ignored while an override is active, which is
  already how `resolve_effective_default_provider_model` behaves.
- Leave `resolve_effective_default_provider_model[_with_effort]` (`src/sase/llm_provider/temporary_override.py:543` and
  `:590`) functionally alone — their explicit override short-circuit returns the same value and documents the
  launch-lane precedence. Refresh the module docstring (lines 18-22) where it says the `default` override "only changes
  the no-`%model` launch default … and an explicit `@default` reference ignores it".

## Delete the second-truth machinery in the display layer

`AliasView.reference_provider` / `reference_model` exist only to carry "what a nested `@default` really points at" past
the carve-out (`src/sase/llm_provider/alias_view.py:468-480` computes them for the `default` row while an override is
active). With the fix they are always equal to `provider`/`model`, so they become dead weight that Symvision would flag.
In `src/sase/llm_provider/alias_view.py`:

- Delete the `reference_provider` / `reference_model` fields (lines 177-178), the
  `if name == DEFAULT_MODEL_ALIAS_NAME and override is not None:` block that computes them (lines 468-480), the two
  constructor keyword arguments (lines 493-494), the `selection_provider` / `selection_model` properties (lines
  236-245), and the three corresponding `AliasView` docstring entries (lines 153-158).
- Keep `reference_effort` (lines 226-233) — it describes the `@<alias>@<effort>` overlay and is unrelated.
- The module docstring's "the _currently effective_ provider/model (an active temporary override wins)" (lines 12-15)
  stays accurate and now also covers rows that inherit the `default` override.

In `src/sase/ace/tui/modals/model_picker_rows.py`:

- Replace `view.selection_provider` / `view.selection_model` with `view.provider` / `view.model` at lines 165 and
  199-200.
- Drop the `and view.name != "default"` carve-out at line 174 so the `override now · snapshot` label also renders for
  `@default`. Selecting `@default` in the temporary-override flow snapshots the currently effective target through
  `set_alias_override`, so the label is now accurate for that row too.

No other Models-panel rendering change is needed: `state_tag`, the effort description strip, the bucket summary, and the
override chip all read `provider` / `model` / `effort` / `override`, which now carry propagated values. After the fix
the screenshot's rows read `CLAUDE(opus) @ high · implicit → @default @ high` and
`CLAUDE(opus) @ xhigh · implicit → @smart`.

**Deliberate non-goal:** no new "inherited from an overridden alias" marker on dependent rows. The `default` row already
shows `override · <time> left` and every dependent row shows its `→ @default` / `→ @smart` chain in the state column.
Surfacing override provenance per row would mean threading a new field out of `_ResolvedModelAlias` through
`registry.resolve_model_provider_with_effort` into `AliasView`, which is a larger change than this bug warrants; raise
it as a follow-up if the chain proves hard to read in practice.

## Performance note

The override map is still loaded lazily, at most once per top-level resolution, and only after a token is recognized as
a known alias. The only new work is that a pure `@default` resolution now performs that one locked read of
`~/.sase/llm_override.json`; every non-`default` alias already did. TUI render paths are unaffected because they pass
concrete model names (`src/sase/ace/tui/provider_styles.py:147` resolves an agent's stored model, which never enters the
known-alias branch), and `build_alias_views` — the one caller that resolves ~20 aliases — is Models-panel-only and
already reads that file for every other row. No caching change is required; do not add one speculatively.

## Documentation

- `docs/llms.md`:
  - 976-978: the `default` override no longer "only changes the _default_ provider/model selection for new agent
    launches" — it takes effect wherever `@default` resolves, like every other alias.
  - 986-987: delete the "An explicit `@default` reference — it ignores the machine-wide `default` override" bullet,
    keeping the launch-scoped sentence (that precedence is unchanged) as its own bullet or folded into the launch-scoped
    one.
  - 1006-1010: the numbered resolution order stays correct; rewrite the trailing paragraph so `default` is described by
    the same three-step ladder as every other alias, and drop "an explicit `@default` reference is not short-circuited
    by the machine-wide temporary default".
  - Add one sentence (near 976-987) recording contract item 5: aliases backed by concrete selectors do not follow a
    `default` override, so override `@cheaper` / `@cheap` / `@smartest` or the size alias itself to move those lanes.
- `docs/ace.md`:
  - 1599-1601: the `default` bullet gains "and every alias that resolves through `@default`".
  - 1610-1613: drop "and an explicit `@default` reference always resolves to the configured default (ignoring the
    `default` override)".
  - 1619-1622: replace "Nested `@default` references intentionally ignore a temporary `default` override, so change or
    override the size-specific alias itself to move that lane" with the new rule plus the selector-backed exception.
- `src/sase/llm_provider/temporary_override.py` module docstring (18-22), as described above.
- Verify and leave unchanged: `docs/xprompt.md:1169-1171` (launch-scoped wins at each hop — still true) and
  `docs/configuration.md:827-831` (alias-value grammar — does not restate the carve-out). Grep both before deciding.
- Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims; no memory file records this
  behavior and editing them needs explicit user permission this task does not carry.

## Regression coverage

- `tests/llm_provider/test_alias_override_resolution.py` (its module docstring, lines 3-6, asserts the old rule):
  - `test_default_override_preserves_two_path_semantics` (161-184) is the direct expression of the bug: it asserts
    `resolve_model_alias("@default") == "claude/opus"` and `@medium_phase_worker → ("claude", "opus", "high")` while
    `default` is overridden to `codex/o3`. Rename it (for example `test_default_override_propagates_to_references`) and
    flip it: both `@default` forms resolve to `codex/o3`, `@medium_phase_worker` resolves to `("codex", "o3", "high")` —
    the model follows the override while its own `@high` overlay wins the effort — and
    `resolve_effective_default_provider_model()` still returns `("codex", "o3")`. Add `@smart` and `@coder`.
  - `test_default_override_effort_drives_no_directive_launch_lane` (187-208):
    `resolve_model_provider_with_effort ("@default")` becomes `("codex", "gpt-5.6-sol", "medium")`; add
    `@default@xhigh → xhigh` to pin outer-effort precedence over the stored override effort.
  - `test_nondefault_override_leaves_default_lane_unchanged` (141-158) stays valid (a `coder` override must not move
    `@default`); fix its comment.
  - New: a launch-scoped `default` beats the machine-wide override at a _nested_ hop —
    `resolve_model_provider("@smart", {"default": "claude/sonnet"})`.
  - New: with `default` both configured and overridden, the override wins.
  - New: a `default` override does not move `@smartest` / `@cheap`-backed lanes (contract item 5).
- `tests/llm_provider/test_registry_resolution.py::test_active_override_wins_over_configured_default_alias` (45-67)
  asserts `resolve_default_alias_provider_model() == ("codex", "gpt-5.6-sol")` and
  `resolve_model_alias("default") == "codex/gpt-5.6-sol"` against an active `agy` override — both now return the
  override. Update and rename; add an unconfigured-`default` case so the registry's second branch is covered.
- `tests/llm_provider/test_alias_view.py::test_default_override_is_surfaced_on_default_row` (616-637): drop the
  `selection_provider` / `selection_model` assertions (those properties are gone) and assert instead that a dependent
  row (`smart` or `coder`) reports the override's provider/model while its own `override` is `None`.
- `tests/test_model_picker_aliases.py`: add a case that a `@default` row carrying an override renders the
  `override now · snapshot` label — that label currently has no test coverage anywhere.
- Re-run and adjust if they assumed the carve-out: `tests/llm_provider/test_temporary_override_resolution.py`,
  `tests/llm_provider/test_alias_overrides.py`, `tests/llm_provider/test_config_alias_resolution.py`,
  `tests/test_temporary_llm_override_agent_meta.py`, `tests/test_directives_split_models_naming.py`.
- No PNG golden change is expected: the Models-panel visual fixtures construct `AliasView`s directly and set no
  overrides. Confirm by running the visual suite; if a golden does move, inspect the artifacts under
  `.pytest_cache/sase-visual/` before accepting anything.

## Verification

Run `just install` first — workspace virtualenvs are ephemeral and this repo requires it before other recipes. Then run
the focused alias-resolution, registry-resolution, alias-view, model-picker, and Models-panel tests; then
`just test-visual`; then `just check`, which is mandatory because file changes land in this repo.

Finish by grepping `src/`, `tests/`, and `docs/` for `selection_provider`, `selection_model`, `reference_provider`,
`reference_model`, and for prose that still says an explicit or nested `@default` ignores the machine-wide override, and
re-run any focused test touched by that cleanup.
