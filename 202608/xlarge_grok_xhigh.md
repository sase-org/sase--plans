---
tier: tale
title: Use Grok xhigh on the shipped @xlarge fallback
goal:
  When the shipped @xlarge ordered fallback selects Grok, the launch requests
  grok/grok-4.6@xhigh so Grok receives --effort xhigh instead of skipping an unsupported
  @max suffix and running at the CLI default.
size: small
proposed_by: bbugyi200.athena.08a
create_time: 2026-08-19 17:44:39
status: wip
---

# Plan

Change only the Grok candidate on the shipped `@xlarge` ordered fallback from
`grok/grok-4.6@max` to `grok/grok-4.6@xhigh`. Keep Claude and Codex on `@max`, keep the
existing `||` availability order, and make the docs and the one real-defaults regression
test describe the per-candidate efforts that actually ship.

## Background and decisions

`src/sase/llm_provider/model_alias_defaults.yml` is the single source of truth for
shipped size-alias targets. Today `@xlarge` is:

```text
claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@max
```

Grok Build's `grok-4.6` model accepts only `--effort low|medium|high|xhigh`. That matrix
is already encoded in `GrokProvider._EFFORT_CLI_ARGS` and covered by
`tests/llm_provider/test_grok_provider_core.py`. An explicit `%effort:max` still raises
`LLMInvocationError`. Alias-borne effort is config-derived (`explicit=False`), so an
unsupported suffix is logged and skipped and the CLI runs at its own default instead of
erroring.

That skip is the defect: when `@xlarge` falls through to Grok, the launch is supposed to
be maximum-effort work, but Grok currently gets no `--effort` flag at all. The highest
level Grok can honor is `xhigh`, which `@medium` already uses for its Grok pool member.
Point the `@xlarge` Grok candidate at `@xhigh` so a Grok-selected xlarge launch actually
receives `--effort xhigh`.

This reverses one earlier product choice. The `@smartest` ordered-fallback work that
later became `@xlarge` deliberately kept `@max` on every candidate and documented the
Grok/Codex skip as intended. Do not restore that Grok skip. Do not silently rewrite
`max` to `xhigh` inside `grok.py`; keep the explicit-vs-default contract and fix the
shipped alias value instead.

Codex also rejects `max` (`minimal|low|medium|high|xhigh`). Leave
`codex/gpt-5.6-sol@max` unchanged unless the user expands this tale. Docs that currently
say `@xlarge` carries `@max` on every candidate, including Grok, must stop saying that;
they may still describe Codex's remaining skip.

No feature flag. This is a correction of an already-shipped default so Grok uses the
highest effort it already supports. Users are not meant to choose the old skip forever,
and a flag would keep the broken Grok candidate reachable.

Do not touch the Rust core. The shipped selector lives in this repo's YAML; Grok's
effort argv translation already lives in `src/sase/llm_provider/grok.py`.

## Implementation

### 1. Change the shipped Grok candidate

In `src/sase/llm_provider/model_alias_defaults.yml`, set `aliases.xlarge.target` to
exactly:

```text
claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@xhigh
```

Keep the description ("Extra-large launch alias for maximum-effort work.") and every
other shipped alias unchanged. Per-member `@effort` suffixes are already valid selector
grammar; no parser, resolver, or provider-adapter change is required for this value to
take effect.

Do not edit `tests/_model_alias_defaults_fixture.py`. That fixture is an intentional
frozen graph-shape contract whose values are deliberately distinct from the shipped
file. A shipped-value change does not change graph shape (`xlarge` remains a `target`
ordered fallback).

Leave `src/sase/default_config.yml`'s commented `xlarge` example as a grammar example,
not a second copy of the shipped default. Do not edit generated `site/` HTML.

### 2. Pin the per-candidate efforts in the real-defaults test

Update `test_shipped_smartest_ordered_fallback_selects_by_availability` in
`tests/llm_provider/test_ordered_fallback_aliases.py`. It already uses
`real_model_alias_defaults` and availability masks, but it currently asserts
`effort == "max"` for every selected candidate, including Grok.

- Rename the test to `test_shipped_xlarge_ordered_fallback_selects_by_availability` so
  the name matches `@xlarge` rather than the retired `@smartest` alias.
- Parametrize `expected_effort` alongside `available` and `expected_target`.
- Keep the three availability masks and the Claude → Codex → Grok priority order.
- Expected resolutions:
  1. Claude, Codex, and Grok available → `claude/opus` at `max`.
  2. Claude unavailable while Codex and Grok are available → `codex/gpt-5.6-sol` at
     `max`.
  3. Only Grok available → `grok/grok-4.6` at `xhigh`.

Do not add a second copy of this coverage under
`test_packaged_defaults_select_correct_effort_per_provider` (that test is for
round-robin pools). Do not add Grok-adapter tests; explicit `xhigh` argv and unsupported
`max` skip/error are already covered in `tests/llm_provider/test_grok_provider_core.py`.

### 3. Regenerate the alias table and fix Grok effort prose

Run `just fmt-docs` so `tools/render_model_alias_docs` rewrites the generated
`model-alias-defaults` block in `docs/llms.md` from the YAML, then `just fmt-md`. Do not
hand-edit that generated block. After regeneration it must show

```text
claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@xhigh
```

Update only prose made inaccurate by the new Grok suffix:

- `docs/llms.md` Grok Build Integration → Reasoning Effort (the paragraph that says
  `@xlarge` carries `@max` on every candidate and that a Grok-selected fallback skips
  `max`).
- `docs/agent_providers.md` Grok Build → Effort ceiling (the matching sentence).

Those sections should now say:

- The shipped Grok `@xlarge` candidate is `grok/grok-4.6@xhigh`, so a Grok-selected
  xlarge launch passes `--effort xhigh`.
- Explicit `%effort:none` / `%effort:minimal` / `%effort:max` still raise a clean SASE
  error.
- Config-derived unsupported levels are still logged and skipped. Codex's `@xlarge`
  candidate remains `@max` and still takes that skip path; do not imply Codex was
  retargeted to `xhigh`.

Leave the provider support matrix, the global Explicit vs. Default Semantics section,
Grok selection/autodetect prose, `docs/configuration.md`'s Grok paragraph, and
`docs/xprompt.md` examples unchanged unless a sentence still claims Grok's shipped
`@xlarge` suffix is `@max`.

## Verification

1. Run `just install` first; this ephemeral workspace may have stale or missing
   development dependencies.
2. Run the renamed real-defaults test and the Grok effort table tests, then run
   `just fmt-docs` and confirm a second `just fmt-docs` produces no additional diff.
3. Run `just check` for whole-repository lint gates and the diff-scoped test lane. If it
   broadens or reports an unusual selection, run `just check-full` only through
   `/sase_monitor`, with a `--next` action as required by project instructions.
4. Inspect the final diff to confirm:
   - the exact `claude@max || codex@max || grok@xhigh` expression
   - no other shipped alias targets changed
   - no frozen-fixture or visual-snapshot churn
   - no hand-edited divergence in the generated docs block
   - no Grok adapter remapping of `max` to `xhigh`

## Risks and out of scope

- **Codex still carries `@max`.** When `@xlarge` selects Codex, alias-borne `max` is
  still skipped and Codex runs at its own default. Fixing that the same way
  (`codex/gpt-5.6-sol@xhigh`) is a sibling product choice and is out of scope here.
- **Do not remap `max` inside the Grok adapter.** `%effort:max` must keep raising. Only
  the shipped alias value changes.
- **Availability is not health.** Ordered fallback still selects by CLI presence, not
  authentication or invocation success. Documented existing `||` behavior; do not
  redesign it.
- **Overrides remain authoritative.** Persistent, temporary, launch-scoped, and
  approval-selected model overrides continue to bypass the shipped `@xlarge` selector.
- Out of scope: changing `@xsmall` / `@small` / `@medium` / `@large`; changing Claude's
  `@max`; adding runtime-failure retries; changing provider autodetection; editing any
  SASE memory file; adding a feature flag.
