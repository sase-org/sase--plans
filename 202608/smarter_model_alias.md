---
tier: tale
title: Add the @smarter model alias and retarget the shipped alias defaults
goal:
  SASE ships a new @smarter builtin model alias, @default delegates to it, and @smart,
  @cheap, @medium_worker, and @large_worker resolve to their new targets across
  resolution, doctor, completion, the ACE Models panel, and docs.
size: medium
proposed_by: bbugyi200.athena.wv.f4
create_time: 2026-08-10 12:00:27
status: wip
---

# Add `@smarter` and retarget the shipped model alias defaults

## Goal

Reshape SASE's shipped builtin model alias graph:

| Alias           | Shipped default after this change              | Change kind                                                    |
| --------------- | ---------------------------------------------- | -------------------------------------------------------------- |
| `smarter`       | `codex/gpt-5.6-sol@xhigh \| claude/opus@xhigh` | **new alias**                                                  |
| `default`       | `@smarter`                                     | **new shape** (was: neither target nor fallback)               |
| `smart`         | `codex/gpt-5.5@xhigh \| claude/sonnet@xhigh`   | **new shape** (was fallback `@default`)                        |
| `cheap`         | `claude/sonnet@high \| codex/gpt-5.5@high`     | value only (was `claude/sonnet@xhigh \| codex/gpt-5.5@medium`) |
| `medium_worker` | `@smart`                                       | **new shape** (was target `codex/gpt-5.5@xhigh`)               |
| `large_worker`  | `@smarter`                                     | **new fallback target** (was `@smart`)                         |

Every other alias (`epic_lander`, `big_epic_lander`, `xsmall_worker`, `small_worker`,
`xlarge_worker`, `smartest`, `cheaper`, `cheapest`) keeps its current entry unchanged.

## Why this is not a one-file change

`src/sase/llm_provider/model_alias_defaults.yml` is the single edit point for changing
an alias _value_, and two of the six rows above (`cheap`, and the concrete pool now
owned by `smart`) are exactly that. The other four are structural and need code:

1. **`smarter` is a new alias name.** `_parse_model_alias_defaults` requires the YAML
   alias key set to equal `_ROLE_ALIAS_NAME_CONSTANTS` exactly
   (`src/sase/llm_provider/model_alias_policy.py:222-232`). A YAML-only addition raises
   `RuntimeError: ... unknown entries for ['smarter']` at import time.

2. **`default` may not currently carry a `fallback`.** The parser rejects it outright
   (`model_alias_policy.py:248-254`).

3. **Even if the parser allowed it, the resolver would ignore it.** In
   `_resolve_model_alias_result`, the `bare == DEFAULT_MODEL_ALIAS_NAME`
   provider-tier-default branch (`model_alias_resolution.py:247-257`) runs _before_ the
   implicit-fallback branch (`:259-266`). `@default` would silently keep resolving to
   the configured/autodetected provider's `large` tier model and the new `@smarter` link
   would be dead. **This is the single highest-risk item in the change: it fails
   silently, not loudly.**

4. **`medium_worker` losing its concrete target crashes `sase doctor`.**
   `src/sase/doctor/checks_config_model_aliases.py:260-262` does an unguarded
   `implicit_alias_targets()[MEDIUM_WORKER_MODEL_ALIAS_NAME]` lookup to build the
   `model_aliases.builtin.phase_worker` migration message. Once `medium_worker` is a
   fallback it is absent from that mapping and the check raises `KeyError`.

## Assumptions

- "`@default` should map to `@smarter`" is read literally: `default` delegates to
  `smarter`, so a temporary/ACE override on `@smarter` also moves `@default`. (The
  user's sentence was truncated mid-clause; this is the reading that makes `@smarter`
  the single canonical high-capability knob.) The mirror-image design — `default` owning
  the pool and `smarter` falling back to `@default` — resolves to the same models today
  but does **not** propagate a `@smarter` override to `@default`, so it is rejected.
- Consequence worth stating plainly: once `default` delegates to a concrete pool,
  `resolve_default_alias_target()` (provider autodetection +
  `resolve_model_name("large")`) no longer shapes `@default` in the shipped
  configuration. It stays reachable for users who override
  `model_aliases.builtin.default` away from a concrete target, and remains the
  terminator when `default` declares no fallback. Do not delete it.
- Targets are valid today: `codex/gpt-5.6-sol` is the codex `large` tier model
  (`src/sase/llm_provider/codex.py:28`), `gpt-5.5` is a registered codex model (`:224`),
  and codex accepts `xhigh` (`:35-38`). `claude/opus` and `claude/sonnet` are registered
  claude models.
- No sibling-repo work. `sase-core` keys its LSP alias rendering off the generic
  `implicit_alias` / `user_alias` _kind_, never off specific alias names
  (`crates/sase_xprompt_lsp/src/server.rs:1954`, `lsp_convert.rs:280`), and `sase-nvim`
  contains no builtin alias names at all. The completion catalog wire schema is
  unchanged — a new alias is just an extra row.

## Implementation

### 1. Alias name constant — `src/sase/llm_provider/model_alias_policy.py`

- Add `SMARTER_MODEL_ALIAS_NAME = "smarter"` with a docstring comment matching the
  neighbours (e.g. `#: The implicit "smarter" high-capability alias.`), placed between
  `SMART_MODEL_ALIAS_NAME` and `SMARTEST_MODEL_ALIAS_NAME`.
- Add it to `_ROLE_ALIAS_NAME_CONSTANTS` in the same relative position.
- Update the module-level explanatory comment block (`:29-43`), which currently says
  "Most roles fall back to another alias (ultimately `@default`)" and "`default` itself
  falls back to the configured or autodetected provider's tier default". Both statements
  become wrong. Describe the new terminal set (`smarter`, `smart`, `smartest`, `cheap`,
  `cheaper`, `cheapest` own concrete targets/pools; `default` delegates to `@smarter`
  but still falls through to the provider tier default when it declares no fallback).

### 2. Parser rule for `default` — same file

Replace the "must not set 'fallback' or 'target'" rule (`:248-254`) with a narrower one:
`default` may declare a `fallback`, must still not declare a `target`, and remains the
only entry allowed to declare neither.

Rationale for keeping `target` forbidden on `default`: it preserves the invariant that
an _undelegated_ `default` has exactly one concrete source of truth (user config, else
the provider tier default). Pinning a concrete model into the shipped `default` row
would create a second one. Keep the existing
`f"entry {name!r} must not set 'fallback' or 'target'"` error text updated to name only
`'target'`.

### 3. Fallback-graph validation — same file, `_validate_fallback_graph`

The walk currently treats `default` as an unconditional terminator (`:187`:
`if referenced == DEFAULT_MODEL_ALIAS_NAME or referenced in target_names`). With
`default` able to delegate, that special case both short-circuits real traversal and
hides cycles that route through `default` (e.g. `smarter → @default` plus
`default → @smarter` would validate clean).

Restructure the loop body so that, per step:

1. `referenced in target_names` → break (terminal concrete target).
2. `referenced in seen` → raise the existing cycle error.
3. `referenced not in fallback_refs` → break if `referenced == DEFAULT_MODEL_ALIAS_NAME`
   (an undelegated `default` legitimately terminates at the provider tier default);
   otherwise raise the existing "terminates at alias ..., which has no target" error.
4. Otherwise record and continue.

Keep both existing error message shapes intact so
`test_parser_rejects_unknown_fallback_reference` and
`test_parser_rejects_two_alias_fallback_cycle` keep matching.

### 4. Resolver ordering — `src/sase/llm_provider/model_alias_resolution.py`

Move the `if bare == DEFAULT_MODEL_ALIAS_NAME:` block (`:247-257`) to sit _after_ the
`fallback_reference` block (`:259-266`), so an implicit fallback on `default` is
honoured and the provider-tier default becomes the last resort. Leave the earlier
branches untouched — launch overrides, temporary overrides, and `bare in aliases` (user
config) must all keep winning ahead of the shipped fallback.

Update `resolve_default_alias_target`'s docstring (`:52-57`): it is now reached only
when `default` is not overridden, not user-configured, **and** declares no shipped
fallback.

### 5. Re-export and display order

- `src/sase/llm_provider/config.py:139-156`: add
  `SMARTER_MODEL_ALIAS_NAME as SMARTER_MODEL_ALIAS_NAME` (keep the block alphabetically
  sorted — it sits between `SMART_` and `SMARTEST_`, and `keep-sorted` /
  `just fix-keep-sorted` may govern this block).
- `src/sase/llm_provider/alias_view.py:61-74` `_ROLE_ALIAS_ORDER`: insert
  `SMARTER_MODEL_ALIAS_NAME` between `SMARTEST_MODEL_ALIAS_NAME` and
  `SMART_MODEL_ALIAS_NAME` (descending capability), and add the import.
- `src/sase/xprompt/model_completion.py:52-66` `_IMPLICIT_ALIASES`: same insertion and
  import.

Do **not** add `smarter` to `_BUILTIN_BUCKET_SPECS` / the `worker` bucket — it is a
capability alias, not a size-specific worker role.

### 6. Doctor KeyError fix — `src/sase/doctor/checks_config_model_aliases.py`

Replace the unguarded `implicit_alias_targets()[MEDIUM_WORKER_MODEL_ALIAS_NAME]`
(`:260-262`) with a shape-agnostic lookup of the shipped default _expression_ — the
concrete target when one exists, else the fallback reference (which is `@smart` after
this change). Import `role_alias_fallbacks` alongside `implicit_alias_targets`. The
rendered message ("... or remove it to accept the shipped `@smart` default") stays
accurate under either shape, so this fix is worth keeping even independent of the
current values.

### 7. Shipped defaults — `src/sase/llm_provider/model_alias_defaults.yml`

Apply the six rows from the Goal table. Add the `smarter` entry between `smart` and
`smartest` (YAML order drives both the Models-panel-adjacent constant order and the
generated docs table row order, since `role_alias_descriptions()` preserves YAML
insertion order). Give `smarter` a `description` in the same voice as its neighbours,
e.g. "Higher-capability pool used automatically by large phase agents and by
`@default`."

Also refresh the file's header comment: the sentence "`default` carries neither: it
resolves through the configured/autodetected provider's tier default" must now describe
the delegated form and note that the provider tier default applies only when `default`
declares no fallback. Refresh `medium_worker`'s and `smart`'s descriptions if their
current wording no longer matches their role (`smart` is now a concrete pool rather than
a `@default` passthrough).

### 8. Config schema and default config

- `src/sase/config/sase.schema.json:1687`: add `smarter` to the enumerated builtin alias
  names in the `builtin` description string, in display order.
- `src/sase/default_config.yml` (~`:823-861`): add a `smarter` line to the commented
  `builtin:` grammar example and correct the surrounding prose, which currently asserts
  "Normal epic landers and @smart fall back to @default" — `@smart` no longer does.

## Tests

### Frozen defaults fixture — `tests/_model_alias_defaults_fixture.py`

This file freezes the shipped _graph shape_ as a contract while keeping its concrete
values deliberately distinct from the shipped ones. All four shape changes plus the new
alias must be mirrored here, or `test_shipped_defaults_match_the_frozen_graph_shape`
fails:

- add a `SMARTER_MODEL_ALIAS_NAME` entry with its own `target` (a distinct fixture
  value, not the shipped pool) and description;
- `DEFAULT_MODEL_ALIAS_NAME` → `fallback: "@smarter"`;
- `SMART_MODEL_ALIAS_NAME` → `target: <distinct fixture pool>` (drop the `@default`
  fallback);
- `MEDIUM_WORKER_MODEL_ALIAS_NAME` → `fallback: "@smart"` (drop its target);
- `LARGE_WORKER_MODEL_ALIAS_NAME` → `fallback: "@smarter"`;
- add the `SMARTER_MODEL_ALIAS_NAME` import.

**Ripple to watch:** `FROZEN_TARGETS` / `FROZEN_TARGET_DETAILS` /
`FROZEN_SELECTOR_MEMBER_DETAILS` are keyed off entries that have a `target`, so
`medium_worker` disappears from all three. Every
`FROZEN_TARGETS[MEDIUM_WORKER_MODEL_ALIAS_NAME]`,
`FROZEN_TARGET_DETAILS[MEDIUM_WORKER_MODEL_ALIAS_NAME]`, and
`frozen_provider_model_effort(MEDIUM_WORKER_MODEL_ALIAS_NAME)` call site must be
retargeted to `SMART_MODEL_ALIAS_NAME` (the alias medium workers now inherit from).
Known call sites — re-grep rather than trusting this list:

- `tests/llm_provider/test_alias_view.py`
- `tests/llm_provider/test_config_role_aliases.py`
- `tests/llm_provider/test_config_alias_resolution.py`
- `tests/llm_provider/test_alias_override_resolution.py`
- `tests/doctor/test_checks_config_model_aliases.py`

Choosing fixture values that keep the _provider_ of `@smart` equal to the old
`medium_worker` provider will minimize churn in provider assertions.

### `tests/llm_provider/test_model_alias_defaults.py`

- Add `SMARTER_MODEL_ALIAS_NAME` to the independent `_DECLARED_ROLE_ALIAS_NAMES`
  ground-truth set (and its import). This set is deliberately hand-enumerated rather
  than derived from the loader — keep it that way.
- Replace `test_default_alias_has_neither_fallback_nor_target` with a test that the
  shipped `default` declares a fallback and no target.
- Add parser tests for the new rules:
  - `default` with a `target` is still rejected;
  - `default` with neither is still accepted (guards the provider-tier terminator path);
  - a cycle routed through `default` (`default → @smarter`, `smarter → @default`) is
    rejected. **This test is the regression guard for step 3** — it fails against the
    current unconditional-terminator implementation.
- `test_fallback_and_target_are_mutually_exclusive_and_cover_every_role` needs no change
  (its `| {DEFAULT_MODEL_ALIAS_NAME}` union holds under both shapes), but re-read it to
  confirm.

### New resolver regression test

Add a test (natural home: `tests/llm_provider/test_config_alias_resolution.py`)
asserting that an unconfigured, un-overridden `@default` resolves through the shipped
fallback rather than the provider tier default. **This is the guard for step 4** —
without it, forgetting the resolver reorder produces a green suite and a silently dead
`@smarter` link. Pair it with a test that a user-configured
`model_aliases.builtin.default` and an active temporary override both still win over the
fallback.

### Other test surfaces

- `tests/test_xprompt_model_completion.py:52-66`: the asserted completion value list is
  exact and ordered — insert `"@smarter"` between `"@smartest"` and `"@smart"`.
- Re-grep `tests/` for `"smartest"` and for hardcoded 13-alias expectations; the builtin
  alias count goes 13 → 14. `tests/_models_panel_helpers.py`,
  `tests/_model_picker_modal_helpers.py`, and
  `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py` build _synthetic_
  `AliasView` rows and are expected to need no change — verify rather than assume.
- `tests/main/test_init_skills_sources.py:406-424` asserts the `sase_plan` skill body
  leaks no internal alias names. `"smarter"` contains `"smart"`, which is already in
  that list, so the existing substring check still covers it; no change needed unless
  the skill text changes.

## Docs

Regenerate the table, then hand-fix the prose around it — nothing lints for staleness in
the hand-written parts.

- `docs/llms.md`: run `just fmt-docs` (invokes `tools/render_model_alias_docs`) to
  refresh the `<!-- BEGIN GENERATED: model-alias-defaults -->` block. Then hand-update
  the surrounding prose (~`:970-1030`), which currently claims "Most fall back through
  other aliases to `@default`" and "`@medium_worker` and `@smartest` also own concrete
  targets", plus the "Phase sizes likewise diverge: ... large uses `@smart`" sentence
  and the YAML example block.
- `docs/configuration.md`: alias enumeration (`:1112-1115`) and the implicit-role
  narrative (`:1334-1352`), which describes `@smart` falling back to `@default` and
  `@medium_worker`/`@smartest` as the concrete-target pair.
- `docs/ace.md`: alias enumerations (`:2413`, `:2434`) and the override/routing
  narrative (`:2581-2611`), which names the concrete-target aliases explicitly.
- `docs/beads.md:1543-1549`: the phase-size routing chain.
- `docs/sdd.md:370-382`: the phase-size alias resolution sentence.

Do **not** hand-edit `CHANGELOG.md` (release-please owns it) and do not touch
`sase/memory/*.md`, `AGENTS.md`, or the provider instruction shims — no memory edit was
authorized for this work.

## Verification

1. `just install` first (workspaces are ephemeral and dependencies may be stale).
2. `just check-full`, not `just check`. This change touches the broadening set: the
   config JSON schema, generated docs, and the core alias policy consumed across ACE,
   xprompt completion, doctor, and bead routing.
3. `just test-visual` for the ACE PNG snapshot suite (the Models panel gains a row, and
   the `%model` completion list gains an entry). If snapshots shift intentionally,
   accept with `--sase-update-visual-snapshots` and inspect the diff artifacts under
   `.pytest_cache/sase-visual/` first.
4. Manual smoke check that the resolver reorder actually took effect — with no user
   alias config, `@default`, `@smarter`, and `@large_worker` should all resolve to a
   member of `codex/gpt-5.6-sol@xhigh | claude/opus@xhigh`, and `@medium_worker` to a
   member of `codex/gpt-5.5@xhigh | claude/sonnet@xhigh`. The ACE Models panel (leader
   `,m`) shows all of this in one view.
5. `sase doctor -C config.model_aliases` must not raise (guards the step 6 `KeyError`).
