---
tier: tale
title: Add grok/grok-4.6 to the shipped @smart, @cheap, and @cheaper pools
goal:
  The shipped @smart, @cheap, and @cheaper load-balanced pools each include
  grok/grok-4.6 at the same effort level their existing members use, and every doc
  statement that says Grok is reachable only by explicit selection is corrected.
size: small
proposed_by: bbugyi200.athena.01f
create_time: 2026-08-14 12:46:10
status: wip
---

# Plan

Add `grok/grok-4.6` as a third member of the three shipped load-balanced model-alias
pools the user named — `@smart`, `@cheap`, and `@cheaper` — reusing the effort level the
other members of each pool already carry. Regenerate the docs table that is derived from
the defaults file, correct the provider docs that currently claim Grok is reachable
_only_ by explicit selection, and add one guard test that every shipped pool member
names a real provider/model pair.

## Background

`src/sase/llm_provider/model_alias_defaults.yml` is the single edit point for what the
implicit aliases resolve to out of the box
(`model_alias_policy._load_model_alias_defaults` parses and validates it; user config
under `llm_provider.model_aliases.builtin` still overrides it). The three target aliases
today:

| Alias     | Shipped target                                 |
| --------- | ---------------------------------------------- |
| `smart`   | `codex/gpt-5.5@xhigh \| claude/sonnet@xhigh`   |
| `cheap`   | `claude/sonnet@high \| codex/gpt-5.5@high`     |
| `cheaper` | `claude/sonnet@medium \| codex/gpt-5.5@medium` |

Facts established while scoping this change:

- **Effort levels line up.** `GrokProvider._EFFORT_CLI_ARGS` declares exactly `low`,
  `medium`, `high`, `xhigh`. The three pools use `xhigh`, `high`, and `medium`
  respectively, so every new member names a level Grok supports. (This is why `@smarter`
  and `@smartest` are out of scope: `@smartest` is `claude/opus@max`, and `max` is one
  of the levels Grok rejects.)
- **`grok-4.6` is a known model.** `GrokProvider.llm_known_model_names()` returns
  `["grok-4.6"]` and both tiers map to it, so `grok/grok-4.6` is a valid explicit
  target.
- **Users without Grok are unaffected.** Pool selection masks members by availability:
  `model_alias_resolution.resolved_target_is_available()` →
  `registry.provider_cli_available("grok")`, which is a `PATH`/`SASE_GROK_PATH` lookup.
  A member whose CLI is absent is skipped by
  `load_balancing.select_model_alias_pool_member()`, and the all-unavailable case keeps
  the unfiltered pool so normal provider diagnostics still explain a failed launch.
- **Tests are insulated from shipped values by design.** `tests/conftest.py`
  autouse-installs the frozen fixture in `tests/_model_alias_defaults_fixture.py` for
  every test that does not request `real_model_alias_defaults`. That fixture's own
  targets are deliberately different from the shipped ones, and its docstring states
  that shipped-_value_ changes need no fixture change — only shipped-_shape_ changes
  (target ↔ fallback, or a fallback retargeted) do. This change keeps all three aliases
  as `target` entries, so
  `tests/llm_provider/test_model_alias_defaults.py::test_shipped_defaults_match_the_frozen_graph_shape`
  keeps passing untouched. **Do not edit the frozen fixture.**
- **The docs table is generated.** `tools/render_model_alias_docs` rewrites the block
  between `<!-- BEGIN GENERATED: model-alias-defaults -->` and
  `<!-- END GENERATED: model-alias-defaults -->` in `docs/llms.md` from the YAML. It is
  wired to `just fmt-docs`; no lint gate regenerates or diff-checks it, so it must be
  run by hand as part of this change.
- **Rotation cursors reset once.** `ModelAliasSelector.fingerprint` hashes the member
  tuple, and `select_model_alias_pool_member` resets to member zero on a fingerprint
  mismatch. Changing membership therefore resets each of the three cursors exactly once,
  which is the intended designed behavior — no migration is needed.

## Implementation

### 1. Add the pool members

Edit `src/sase/llm_provider/model_alias_defaults.yml`, appending `grok/grok-4.6` as the
**last** member of each of the three pools and leaving existing member order untouched
(smallest diff; ordering only affects which member a fresh cursor picks first):

- `smart.target`: `"codex/gpt-5.5@xhigh | claude/sonnet@xhigh | grok/grok-4.6@xhigh"`
- `cheap.target`: `"claude/sonnet@high | codex/gpt-5.5@high | grok/grok-4.6@high"`
- `cheaper.target`:
  `"claude/sonnet@medium | codex/gpt-5.5@medium | grok/grok-4.6@medium"`

Leave `default`, the `*_worker` roles, `smarter`, `smartest`, and `cheapest` alone, and
leave every `description` unchanged — the descriptions do not name providers.

### 2. Regenerate the derived docs table

Run `just fmt-docs` (which runs `tools/render_model_alias_docs`) and then `just fmt-md`
so prettier normalizes the regenerated Markdown table widths. Do not hand-edit the
generated block in `docs/llms.md`.

### 3. Correct the "explicit-only" prose

Adding Grok to shipped pools makes Grok reachable _without_ explicit selection whenever
the `grok` CLI is present. Three passages currently say otherwise and must be corrected.
Keep the autodetect facts intact — Grok still declares no `llm_autodetect_priority` and
still never participates in provider autodetection; what changes is that the shipped
`@smart`/`@cheap`/`@cheaper` pools now name `grok/grok-4.6` directly and select it when
its CLI is installed.

- `docs/llms.md`, provider-resolution step 5 (the sentence beginning "Muse and Grok are
  reachable only by explicit selection"): keep the statement true for Muse, and reword
  the Grok half so it says Grok never participates in autodetection but is reached by
  the shipped `@smart`/`@cheap`/`@cheaper` pools when the CLI is installed.
- `docs/llms.md`, "Grok Build Integration → Selection" (the paragraph beginning "Grok is
  **explicit-only**." and listing `llm_provider.provider: grok`, `%model:grok/grok-4.6`,
  and `SASE_GROK_PATH`): add the shipped pools as a fourth route, and keep the existing
  sentence about `sase doctor`'s wrong-binary advisory.
- `docs/agent_providers.md`, "## Grok Build" intro paragraph (same "explicit-only" claim
  plus the "Select it with ..." sentence): make the same correction.
- `docs/configuration.md`, the "Grok Build uses ..." paragraph in the provider section
  (the clause "so Grok is explicit-only like Muse"): same correction. Do **not** touch
  the neighboring Muse paragraph — Muse's `muse-spark-1.2-contributor` advisory
  statement that SASE "never routes a tier map, `@cheap`, or `@cheaper` to it
  automatically" stays true and is unrelated.

While in `docs/llms.md`, the "Model Mapping" note under Grok ("Inventing a distinct
`small` mapping to a model that may not exist would make ordinary `@cheap`/`@cheaper`
routing fail") and the "Reasoning Effort" note about `@smartest` resolving to
`claude/opus@max` both remain accurate — leave them as they are.

### 4. Add a shipped-defaults validity guard

Add one test to `tests/llm_provider/test_model_alias_defaults.py` that requests the
`real_model_alias_defaults` fixture and asserts, for every shipped target (parsing
selector members with `parse_model_alias_selector` and stripping efforts with
`split_model_effort`), that a `provider/model` target names a registered provider
(`registry.registered_provider_names()`) and a model that provider actually publishes
(`registry.model_to_provider_map()` maps the model back to that provider). This is the
gate that would have caught a mistyped `grok/grok4.6` or a member pointing at an
unregistered provider; note that `model_to_provider_map()` is usable against the real
registry in tests (see `tests/test_agy_integration_polish.py`). Keep it to one test
function, and do not assert any specific shipped value — asserting values is exactly
what the frozen fixture exists to avoid.

## Verification

- `just install` first (workspace virtualenvs are ephemeral), then `just check`.
- Confirm the regenerated table in `docs/llms.md` shows all three pools with the new
  third member, and that re-running `just fmt-docs` produces no further diff.
- Sanity-check resolution end to end, e.g. `sase doctor -C llm.model_advisory` stays
  clean (Grok publishes no model advisory) and, if the `grok` CLI is present on the
  host, that `@cheap` rotates across three members rather than two.
- If `just check`'s scoped test lane reports an unusual selection, escalate to
  `just check-full` through `/sase_monitor` rather than inline.

## Risks and notes

- **Distribution shift.** Each of the three pools moves from 50/50 to 33/33/33 across
  its members for hosts that have all three CLIs, and each pool's rotation cursor resets
  to member zero once on the first resolution after the upgrade. Both are inherent to a
  membership change and need no mitigation.
- **Wrong-binary `grok` on `PATH`.** `provider_cli_available` only checks that _some_
  `grok` is on `PATH`, and three unrelated tools compete for that name (Grok Build,
  `grok-dev`, Homebrew's deprecated regex tool). Before this change that exposure was
  limited to users who opted into Grok explicitly; after it, any host with an impostor
  `grok` on `PATH` can have shipped-pool launches routed to it and fail. The existing
  mitigation is `sase doctor`'s deep `providers.cli_version` identity advisory
  (`_apply_grok_identity_advisory` in `src/sase/doctor/checks_deep_providers.py`), which
  already runs for every registered provider regardless of configuration and tells the
  user to set `SASE_GROK_PATH`. Deliberately **not** in scope here: making pool
  availability itself probe binary identity, which would put a subprocess call on the
  alias-resolution path. If the reviewer wants that hardening, file it as a separate
  task bead through `/sase_new_task`.
- **Out of scope.** `@smarter`, `@smartest`, and `@cheapest` membership; any change to
  Grok's tier map, autodetect priority, or effort declarations; any SASE memory file
  edit (none of `sase/memory/*.md` names these shipped targets).
