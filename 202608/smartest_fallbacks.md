---
tier: tale
title: Add Claude, Codex, and Grok ordered fallbacks to the shipped @smartest alias
goal:
  The shipped @smartest alias prefers claude/opus, then codex/gpt-5.6-sol, then
  grok/grok-4.6 according to provider CLI availability, with every candidate carrying
  the requested max effort and the documented behavior matching runtime semantics.
size: small
proposed_by: bbugyi200.athena.01f.f1
create_time: 2026-08-14 13:04:26
status: wip
---

# Plan

Change the shipped `@smartest` target from the single `claude/opus@max` target to the
exact ordered fallback requested:

```text
claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@max
```

Keep `@smartest` as the capability owner inherited by `@xlarge_worker` and
`@big_epic_lander`, add a regression test for the shipped priority order, regenerate the
derived alias table, and correct documentation that still describes `@smartest` as a
single concrete Claude target.

## Background and decisions

`src/sase/llm_provider/model_alias_defaults.yml` is the single source of truth for
shipped implicit alias values. `model_alias_policy._load_model_alias_defaults()` parses
it, and user configuration under `llm_provider.model_aliases.builtin.smartest` still
overrides it. The current graph already routes extra-large work and threshold-selected
epic landers through `@smartest`; no role-edge or launch-routing change is needed.

The selector grammar and runtime already implement `A || B` as an ordered,
availability-based fallback. The first candidate whose registered provider CLI is
installed wins. Selection is stateless, does not read or advance the round-robin cursor,
and does not retry the next candidate after an authentication, model, invocation, or
runtime failure. If none of the three CLIs is installed, candidate zero remains selected
so normal Claude diagnostics explain the failed launch. This plan uses those existing
semantics; it does not add failure-triggered retries.

The requested `@max` suffix has a provider-specific consequence that the docs must make
clear:

- Claude supports `max`, so a Claude-selected launch receives `--effort max`.
- Codex and Grok currently reject an _explicit_ `%effort:max`, but an effort suffix
  carried by an alias target is deliberately marked `explicit=False` by
  `resolve_effective_effort()`. Their provider adapters therefore log that `max` is
  unsupported, omit the effort argument, and let the selected CLI use its own default
  effort. Do not silently rewrite either fallback candidate to `xhigh`, and do not
  expand Codex or Grok's declared effort support as part of this change.
- A prompt-level `%effort:<level>` remains authoritative over the alias-borne value and
  retains the existing explicit-error behavior when the selected provider cannot honor
  it.

The frozen test defaults in `tests/_model_alias_defaults_fixture.py` deliberately avoid
coupling routine resolution tests to shipped target values. `@smartest` remains a
`target` entry, so the shipped graph shape does not change and that fixture must not be
edited. A focused real-defaults test will pin this requested value and order without
replacing the test-owned defaults used elsewhere.

## Implementation

### 1. Define the shipped ordered fallback

Edit `src/sase/llm_provider/model_alias_defaults.yml`:

- Set `aliases.smartest.target` to
  `"claude/opus@max || codex/gpt-5.6-sol@max || grok/grok-4.6@max"`, preserving the
  requested provider order exactly.
- Reword its description from a singular highest-capability model to a
  highest-capability ordered fallback used by extra-large phase agents and large epic
  landers. The description feeds the generated documentation and ACE metadata.
- Leave every other shipped target and fallback edge unchanged, including the recently
  expanded `@smart`, `@cheap`, and `@cheaper` round-robin pools.

Update the explanatory comment in `src/sase/default_config.yml` so it no longer says
only the existing `smart`/`smarter`/cost aliases own independent pools: distinguish
`@smartest`'s ordered fallback from the aliases that own round-robin pools. Keep the
commented YAML values as override/grammar examples rather than turning them into a
second copy of the shipped defaults.

### 2. Add a shipped-default regression test

In `tests/llm_provider/test_ordered_fallback_aliases.py`, add one parametrized test that
requests `real_model_alias_defaults`, controls `_resolved_target_is_available`, and
resolves `@smartest` with effort retained. Cover these availability masks:

1. Claude, Codex, and Grok available selects `claude/opus@max`.
2. Claude unavailable while Codex and Grok are available selects
   `codex/gpt-5.6-sol@max`.
3. Only Grok available selects `grok/grok-4.6@max`.

This single test pins the exact candidates, priority order, fallback mode as observed
through resolution, and retained alias-borne effort. Existing generic tests already
cover stateless fallback selection, all-unavailable preservation, nested-selector
rejection, and override suspension; do not duplicate them.

Rename `test_every_shipped_pool_member_names_a_registered_provider_model` in
`tests/llm_provider/test_model_alias_defaults.py` to use “selector” rather than “pool”
in its name/docstring. Its existing loop already covers both `|` and `||` selectors, so
the new Codex and Grok fallback candidates will automatically be checked against the
registered provider/model catalog without changing its assertions.

Do not update `tests/_model_alias_defaults_fixture.py` and do not accept ACE PNG
snapshots: the graph shape and generic selector presentation are unchanged, while the
new real-defaults test owns the shipped-value contract.

### 3. Regenerate and reconcile documentation

Run `just fmt-docs` to render the generated model-alias block in `docs/llms.md` from the
YAML, then `just fmt-md` to normalize Markdown formatting. Do not hand-edit the
generated block. It must show `@smartest` as the three-member `||` chain and use the
updated description.

Update only prose made inaccurate by the new shipped default:

- In `docs/llms.md` provider-selection step 5 and “Grok Build Integration → Selection”,
  `docs/agent_providers.md`'s Grok introduction, and the Grok provider paragraph in
  `docs/configuration.md`, add `@smartest` as an automatic Grok route and distinguish
  its ordered fallback from the load-balanced `@smart`/`@cheap`/`@cheaper` pools. Keep
  the fact that Grok never participates in provider autodetection.
- In the Grok effort sections of `docs/llms.md` and `docs/agent_providers.md`, replace
  the obsolete “`@smartest` resolves to Claude” statement with the precise behavior:
  explicit unsupported effort still errors, while the alias-borne `max` is best-effort
  and is logged/skipped when the fallback selects Grok. Where useful, note the same
  Codex behavior rather than implying every `@max` suffix becomes a provider flag.
- In the implicit-alias and temporary-override prose in `docs/llms.md`, and the matching
  Models-panel prose/examples in `docs/ace.md`, replace statements that call `@smartest`
  a concrete target with ordered-fallback terminology. Make clear that an override on
  `@smartest` suspends its selector just as overrides on the pool-owning aliases suspend
  their rotations, while the fallback itself has no cursor.
- Use “selected fallback candidate” where documentation currently implies dependent
  roles inherit one permanent `@smartest` model. Generic examples that intentionally
  demonstrate a user overriding `smartest` with a single maximum-effort target remain
  valid and should stay examples, not be rewritten as shipped defaults.

Leave `docs/beads.md` routing, the provider support matrix, and the selector grammar
unchanged except if formatting requires it: they already accurately say the roles route
through `@smartest`, Codex/Grok do not support explicit `max`, and `||` selects by CLI
availability without runtime retry.

## Verification

1. Run `just install` first because this ephemeral workspace may have stale or missing
   development dependencies.
2. Run the focused real-default regression test and the bundled defaults tests, then run
   `just fmt-docs` and confirm a second `just fmt-docs` produces no additional diff.
3. Run `just check` for whole-repository lint gates and the diff-scoped test lane. If it
   broadens or reports an unusual selection, run `just check-full` only through
   `/sase_monitor`, with a `--next` action as required by project instructions.
4. Inspect the final diff to confirm the exact `claude || codex || grok` order, `@max`
   on all three candidates, no changes to the other shipped aliases, no frozen-fixture
   or visual-snapshot churn, and no hand-edited divergence in the generated docs block.
5. Sanity-check `sase doctor -C config.model_aliases` and
   `sase doctor -C llm.model_advisory`; the shipped expression must parse cleanly and
   all three provider/model pairs must remain registered without advisories.

## Risks and out of scope

- **Availability is not health.** An installed but unauthenticated or failing Claude CLI
  prevents selection of Codex/Grok because ordered fallback does not retry runtime
  failures. This is existing `||` behavior and should be documented, not redesigned
  here.
- **`max` is best-effort on two candidates.** Codex and Grok will run with their own
  default effort after warning because their adapters do not support `max`. This follows
  the exact requested alias expression. Provider-specific remapping or new effort
  support is out of scope.
- **Grok executable-name collision.** If Claude and Codex are unavailable, any `grok`
  executable on `PATH` satisfies the shallow availability check even if it is the stale
  community CLI or Homebrew regex tool. The existing deep doctor identity advisory and
  `SASE_GROK_PATH` guidance remain the mitigation; do not add subprocess identity probes
  to alias resolution.
- **No rotation migration.** `@smartest` changes from one target to a stateless ordered
  fallback and therefore creates no load-balancing cursor state or distribution reset.
- **Overrides remain authoritative.** Persistent, temporary, launch-scoped, and
  approval-selected model overrides continue to bypass the shipped `@smartest` selector.
- Out of scope: changing `@smart`, `@smarter`, `@cheap`, `@cheaper`, or `@cheapest`;
  changing role fallback edges; adding runtime-failure retries; changing provider
  autodetection; or editing any SASE memory file.
