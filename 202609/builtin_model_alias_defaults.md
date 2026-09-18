---
tier: tale
title: Retune shipped builtin size-alias defaults
goal:
  Ship the research-recommended size-alias pools on SASE-valid model IDs, with Luna and
  Terra published in the Codex catalog and every user-facing quote of the old shipped
  selectors updated.
size: medium
proposed_by: bbugyi200.athena.0nd
create_time: 2026-09-18 19:17:16
status: wip
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                    | Why                                                       |
| ------------ | --------------------------------------------------------------------------- | --------------------------------------------------------- |
| derives-from | [research:202609/model_alias_budget_policy/model_alias_budget_policy.md][1] | Adapted recommended alias set onto SASE catalog model IDs |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/model_alias_budget_policy/model_alias_budget_policy.md

<!-- sase:links:end -->

# Plan: Retune shipped builtin size-alias defaults

Update SASE's five shipped size aliases (`@xsmall`, `@small`, `@medium`, `@large`,
`@xlarge`) to the budget-policy research recommendation, rewritten onto model IDs SASE
actually publishes. Keep the five alias names, selector grammar, and scalar launch
defaults (`default_model: "@large"`, `epic_lander_model: "@large"`,
`big_epic_lander_model: "@xlarge"`, empty `default_effort`).

This is one cohesive defaults-and-catalog change. One follow-up coding agent can
implement it from this file; it is not an epic.

## Source and adaptation

The recommended set is the closing YAML in
`research:202609/model_alias_budget_policy/model_alias_budget_policy.md`. That research
is policy, not a drop-in config: several Claude IDs it uses are not SASE catalog names,
and two Codex IDs it uses are not published yet.

Shipped selector members must name a registered provider model
(`tests/llm_provider/test_model_alias_defaults.py::test_every_shipped_selector_member_names_a_registered_provider_model`).
The model picker and `%model` catalog also refuse unpublished IDs
(`tests/test_model_picker_options.py` and
`tests/test_xprompt_model_completion_catalog.py` assert `claude-opus-5` and
`claude-sonnet-5` stay out of the catalog).

Use this mapping. Do not add `claude-sonnet-5`, `claude-opus-5`, or `claude-fable-5-1`
to the Claude catalog.

| Research member                | SASE member                  | Reason                                                                  |
| ------------------------------ | ---------------------------- | ----------------------------------------------------------------------- |
| `claude/claude-haiku-4-5`      | `claude/claude-haiku-4-5`    | Already published.                                                      |
| `claude/claude-sonnet-5@…`     | `claude/sonnet@…`            | Catalog uses the floating family alias `sonnet`, not `claude-sonnet-5`. |
| `claude/claude-opus-5@…`       | `claude/opus@…`              | Catalog uses the floating family alias `opus`, not `claude-opus-5`.     |
| `claude/claude-fable-5-1@high` | `claude/claude-fable-5@high` | Catalog publishes `claude-fable-5`; there is no `claude-fable-5-1`.     |
| `codex/gpt-5.6-luna@low`       | `codex/gpt-5.6-luna@low`     | Add `gpt-5.6-luna` to the Codex catalog in the same change.             |
| `codex/gpt-5.6-terra@…`        | `codex/gpt-5.6-terra@…`      | Add `gpt-5.6-terra` to the Codex catalog in the same change.            |
| `codex/gpt-5.6-sol@high`       | `codex/gpt-5.6-sol@high`     | Already published.                                                      |
| `codex/gpt-6-astra@high`       | `codex/gpt-6-astra@high`     | Already published.                                                      |
| `grok/grok-4.6@…`              | `grok/grok-4.6@…`            | Already published.                                                      |

Leave Antigravity out of `@xsmall`. That is an intentional membership change, not an
accidental omission. Do not add Gemini back as a last-resort tail: the first four
aliases already include Claude, Codex, and Grok, so extra tails from a hard-disabled
provider add no capacity.

Keep equal unweighted `|` pools. Do not add integer weights. Do not describe round-robin
as quota balancing.

## Exact shipped targets

Replace the `aliases` mapping in `src/sase/llm_provider/model_alias_defaults.yml` with:

```yaml
aliases:
  xsmall:
    target: "claude/claude-haiku-4-5 | codex/gpt-5.6-luna@low | grok/grok-4.6@low"
    description: >-
      Extra-small launch alias for lookup, formatting, and tiny edits with obvious
      checks.
  small:
    target: "claude/sonnet@low | codex/gpt-5.6-terra@low | grok/grok-4.6@low"
    description: "Small launch alias for straightforward task and phase work."
  medium:
    target: "claude/sonnet@medium | codex/gpt-5.6-terra@medium | grok/grok-4.6@medium"
    description: "Medium launch alias for ordinary implementation work."
  large:
    target: "claude/opus@high | codex/gpt-5.6-sol@high | grok/grok-4.6@high"
    description: "Large launch alias for planning-heavy work and default launches."
  xlarge:
    target:
      "(claude/claude-fable-5@high | codex/gpt-6-astra@high) || grok/grok-4.6@xhigh"
    description: >-
      Extra-large launch alias for maximum-effort work; Grok is last resort when Claude
      and Codex are unavailable.
```

Haiku has no effort suffix on purpose. Claude Code rejects `none`/`minimal`, and the
research treats Haiku as a workload class rather than an effort overlay.

`@large` is now a three-provider pool, not `(Claude | Codex) || Grok`. Grok is a full
member at `high`. Only `@xlarge` keeps the parenthesized last-resort tail, and that tail
stays `grok/grok-4.6@xhigh`.

Do not change `llm_provider.default_model`, `epic_lander_model`,
`big_epic_lander_model`, or `default_effort` in `src/sase/default_config.yml`. Those
already match the research (`@large`, `@large`, `@xlarge`, empty effort).

## Catalog publication for Luna and Terra

`src/sase/llm_provider/codex.py`:

- Add `gpt-5.6-luna` and `gpt-5.6-terra` to `llm_known_model_names()`.
- Add short aliases `gpt56luna` and `gpt56terra` in `llm_model_short_aliases()`,
  matching the existing `gpt-5.6-sol` → `gpt56sol` pattern.

Keep `gpt-5.5` in the catalog. This change stops using it as a shipped alias member; it
does not retire the model name.

Update the hand-maintained Codex rows in `docs/llms.md` under Automatic Provider
Resolution and Model Short Aliases so they list the new IDs and shorthands. `just fmt`
regenerates only the `<!-- BEGIN GENERATED: model-alias-defaults -->` block; it will not
update those catalog tables.

## Docs and grammar examples that quote shipped selectors

Update every user-facing quote of the current shipped selectors, not just the generated
table:

- `docs/llms.md` generated alias table: run `just fmt` (or
  `tools/render_model_alias_docs`) after the YAML change. The renderer must match
  `implicit_alias_targets()` / `role_alias_descriptions()`.
- `docs/ace.md` Launch Control examples that currently quote
  `(claude/opus@xhigh | codex/gpt-5.6-sol@xhigh) || grok/grok-4.6@xhigh` for `@large`
  and `(claude/claude-fable-5@xhigh | codex/gpt-6-astra@xhigh) || grok/grok-4.6@xhigh`
  for `@xlarge`. Rewrite the `@large` example as a three-member pool at `high`. Rewrite
  the `@xlarge` example with `@high` on Fable/Astra and `@xhigh` only on the Grok tail.
- `docs/getting_started.md` Grok routing sentence. After this change Grok is a full
  member of `@xsmall`, `@small`, `@medium`, and `@large`, and remains only the last
  `@xlarge` fallback candidate. Update `tests/test_docs_getting_started_providers.py` to
  the same wording.
- Commented `model_aliases.builtin` grammar examples in `src/sase/default_config.yml`.
  Those comments are not shipped defaults, but they currently teach `gpt-5.5` and
  `xhigh` as the ordinary pattern. Restate them with the new models/efforts, and keep
  one weighted-pool example so the grammar comment still demonstrates `A | 3 B`.

Do not churn override-example YAML in `docs/llms.md` that is already labeled as
custom/override illustration (`claude/haiku | codex/gpt-4.1-mini`, and similar) unless
it claims to be the shipped default.

Do not edit `CHANGELOG.md`. Release-please generates it from the conventional commit
message.

## Tests that pin shipped values

`tests/_model_alias_defaults_fixture.py` is a frozen graph-shape fixture, not a mirror
of shipped values. Leave it alone unless an alias switches between `target` and
`fallback` (this change does not).

Update tests that use the `real_model_alias_defaults` fixture or otherwise parse the
packaged YAML:

- `tests/llm_provider/test_load_balanced_alias_defaults.py`
  - `test_packaged_defaults_select_correct_effort_per_provider`: drop the `agy/` xsmall
    expectation; Haiku has `effort is None`; Luna/Terra/Grok use `low` on
    `@xsmall`/`@small`. Extend coverage to `@medium` (`medium` effort) and `@large`
    (`high` effort, Grok as a pool member rather than a tail).
  - Replace `test_shipped_large_uses_last_resort_grok` with a test that `@large`
    round-robins Claude/Codex/Grok with no `fallback_members`.
  - Broaden `test_small_size_alias_has_no_antigravity_member` (or add a sibling) so
    `@xsmall` also has no `agy/` member.
- `tests/llm_provider/test_ordered_fallback_aliases.py::test_shipped_xlarge_pool_uses_last_resort_grok`:
  primary members become `claude/claude-fable-5@high` and `codex/gpt-6-astra@high`; the
  Grok tail stays `@xhigh`.
- `tests/test_model_picker_options.py::test_build_model_options_has_known_models`:
  assert `gpt-5.6-luna` and `gpt-5.6-terra` are pickable. Keep the existing assertions
  that `claude-opus-5` and `claude-sonnet-5` are absent.
- `tests/test_xprompt_model_completion_catalog.py`: keep excluding `claude-opus-5` /
  `claude-sonnet-5`. If a test lists the live Codex catalog exhaustively, include Luna
  and Terra.

Leave tests that construct their own alias strings (`codex/gpt-5.5`,
`claude/sonnet@high`, and similar) alone unless they import packaged defaults.

Visual PNG goldens that stub alias views or filter the picker to `opus` / `@medium`
should not move. If a live-catalog picker snapshot shows the unfiltered Codex list,
refresh only those goldens with `just fix-tui-screenshots` and the matching selector.

## Implementation order

1. Publish `gpt-5.6-luna` and `gpt-5.6-terra` (names plus short aliases) on the Codex
   provider.
2. Write the five targets and descriptions in `model_alias_defaults.yml`.
3. Point docs, getting-started prose, ACE examples, and `default_config.yml` grammar
   comments at the new selectors.
4. Update the shipped-value tests listed above.
5. Run `just fmt` so the generated `docs/llms.md` alias table matches the YAML, then
   `just check`.

## Out of scope

- Quota-aware routing, projected-headroom weights, or `used/elapsed` soft-disable
  automation. Those are future router work, not this defaults retune.
- Changing `default_model` from `@large` to `@medium`.
- Adding `claude-sonnet-5`, `claude-opus-5`, or `claude-fable-5-1` to the Claude
  catalog.
- Removing `gpt-5.5` from the Codex catalog.
- Personal `llm_provider.model_aliases.builtin` overrides (for example in chezmoi).
  Those will keep masking shipped defaults until the operator clears them; do not edit
  linked dotfiles in this tale.
- A measurement harness or one-to-two-week trial. Shipping the adapted defaults is the
  whole tale.
- Feature flags. Users who want the old pools can set
  `llm_provider.model_aliases.builtin`.

## Verification

Run `just fmt` then `just check`. The scoped suite should include the
`real_model_alias_defaults` tests, the Codex catalog/picker tests, the docs renderer
parity test, and `tests/test_docs_getting_started_providers.py`. If `just check`
escalates or a live-catalog PNG golden fails, follow `sase/memory/lint_and_test.md`
rather than skipping those gates.
