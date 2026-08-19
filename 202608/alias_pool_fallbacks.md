---
tier: tale
title: Last-resort fallbacks for model alias pools
goal: "A parenthesized load-balanced pool can carry an ordered last-resort fallback (`(A
  | B) || C`). The shipped @large alias uses that grammar so Grok is selected only when
  neither Claude nor Codex is available, and ACE, doctor, and docs present the new
  expression consistently.

  "
size: medium
proposed_by: bbugyi200.athena.089
create_time: 2026-08-19 17:05:58
status: wip
---

# Last-resort fallbacks for model alias pools

## 1. Goal and user-visible behavior

Today a model-alias value is either a `|` round-robin pool **or** a `||` ordered
fallback chain. Mixing the operators in one value is rejected, so there is no way to say
"load-balance across these providers, and only if every pool member is unavailable use
this other model."

This plan adds exactly that grammar:

```text
(<p1>/<m1>@<e1> | <p2>/<m2>@<e2> | ... | <pN>/<mN>@<eN>) || <pF>/<mF>@<eF>
```

Given:

```yaml
llm_provider:
  model_aliases:
    builtin:
      large: "(claude/opus@xhigh | codex/gpt-5.6-sol@xhigh) || grok/grok-4.6@xhigh"
```

then `@large`:

- round-robins between Claude Opus and Codex GPT-5.6 Sol at `xhigh` when either
  provider's CLI is installed and not **hard**-disabled;
- selects `grok/grok-4.6@xhigh` when **every** pool member is unavailable (CLI not
  installed, unregistered, or **hard**-disabled);
- does **not** divert to Grok merely because both pool members are **soft**-disabled
  (sparing): an all-soft pool still rotates among itself, matching today's `|` mask.

The shipped `@large` default becomes that expression. Users who already override
`model_aliases.builtin.large` are unchanged.

## 2. Current implementation (read this first)

- `src/sase/llm_provider/load_balancing.py` owns the selector grammar
  (`parse_model_alias_selector`), `ModelAliasSelector` (`mode`, `members`, `weights`,
  `normalized`, `fingerprint`), rotation state in `~/.sase/llm_lb.json`, and the two
  selection entry points `select_model_alias_pool_member` /
  `select_model_alias_fallback_member`. Mixing `|` with `||` currently raises
  `ModelAliasSelectorError` ("cannot mix").
- `src/sase/llm_provider/model_alias_resolution.py` resolves alias chains, builds
  `ModelAliasSelectorMember` display records via `model_alias_selector_details`, and
  produces authoring errors via `validate_model_alias_selector_value`. Selector
  selection is duplicated in `_resolve_model_alias_result` (the normal path **and** the
  hard-disabled-override path).
- `src/sase/llm_provider/model_launch_settings.py` re-implements resolve-members →
  availability → select for a raw selector typed into a scalar launch field.
- `src/sase/llm_provider/model_alias_defaults.yml` is the single source of truth for
  shipped size-alias targets. `@large` is currently
  `claude/opus@xhigh | codex/gpt-5.6-sol@xhigh`.
- `src/sase/doctor/checks_config_model_aliases.py` reports selector validation errors
  and availability notes. The availability-note branch is gated on `"|"` in the target,
  so a compound expression already enters that branch.
- ACE surfaces: `src/sase/ace/tui/modals/models_panel_selector.py` (compose/parse),
  `.../models_panel_selector_builder.py` (guided builder),
  `.../models_panel_rendering_descriptions.py` (member list),
  `.../models_panel_rendering_rows.py` + `src/sase/ace/tui/model_alias_styles.py`
  (`pool <available>/<total>` chip).

Two properties MUST be preserved:

1. `ModelAliasSelector.fingerprint` for an unweighted two-member pool is asserted in
   `tests/llm_provider/test_load_balanced_aliases.py` to equal
   `sha256(json.dumps(members, ...))`. Wrapping that pool in parentheses and appending a
   last-resort fallback MUST keep the **same** fingerprint, so existing `@large` cursors
   in `~/.sase/llm_lb.json` survive the shipped-default change.
2. Unparenthesized `A | B || C` continues to raise the existing "cannot mix" error.
   Parentheses are what disambiguate the new form; do not silently reinterpret the
   rejected mix.

## 3. Grammar

A last-resort expression is a parenthesized `|` pool, then `||`, then one or more
unweighted single-target fallback candidates:

```text
'(' <pool> ')' '||' <fallback> ['||' <fallback>]...
```

`<pool>` is the existing `|` grammar (two or more members, optional `N ` weights).
`<fallback>` is the existing `||` candidate grammar (no weights, no `|`, no
parentheses).

Rules:

- Parentheses are **required** to attach a last-resort tail to a pool. `A | B || C`
  remains a mix error.
- The parenthesized group must be a `|` pool. `(A || B)`, `(A)`, and `()` are errors.
- Nested or stray parentheses are errors (`((A | B) || C)`, `A | (B | C)`,
  `(A | B) || (C | D)`).
- A parenthesized pool with **no** `||` tail is allowed and normalizes to the bare pool:
  `(A | B)` → `A | B`. This is grouping only, not a new mode.
- The `||` tail is an ordered fallback chain of one or more single targets. The
  requested `@large` default uses a one-candidate tail; `(A | B) || C || D` is the
  cheap, consistent generalization and is in scope.
- Weights remain pool-only. A weight prefix on a last-resort candidate is the same error
  as a weight on a `||` chain.
- Members (pool and last-resort) keep the existing single-target grammar, including
  `@alias` references and trailing `@<effort>`. A member that itself resolves to a
  selector still fails closed ("nested …; selector members must resolve to a single
  target").
- Selectors remain config-only: `%model` values, launch-scoped alias overrides, and
  temporary overrides stay single targets.
- Canonical spelling: `(A | B) || C`, with the existing pool-side spacing/weight
  rendering inside the parens and `||` between last-resort candidates. No extra space
  inside the parentheses.

`parse_model_alias_selector` still returns `None` for values with no `|` at all.

### 3.1 Data model

Keep `mode` as `"round_robin"` for a pool-with-tail (so cursor selection, the pool chip,
and ACE "pool" labeling keep working) and add:

```python
fallback_members: tuple[str, ...] = ()
```

on `ModelAliasSelector`. `__post_init__` rejects a non-empty tail when
`mode != "round_robin"`. `members` / `weights` continue to describe **only** the pool.

Add `last_resort: bool = False` to `ModelAliasSelectorMember` so display, the pool chip,
and doctor notes can tell pool rows from tail rows without a third selector mode.

Add a helper that concatenates `members + fallback_members` and a single selection entry
point (see §4). Do not leave a fourth copy of "pick an index" in
`model_launch_settings.py`.

`normalized` for a tailed pool is `f"({pool}) || {tail}"`. `fingerprint` stays the
**pool** fingerprint from today (members, plus weights when any weight is `> 1`) and
MUST ignore `fallback_members`.

## 4. Selection semantics

Reuse the existing masks. Do not invent a third availability enum.

Let `pool_states` be `MemberAvailability` for `selector.members` and `tail_states` the
same for `selector.fallback_members`.

1. If `mode == "fallback"` (no tail by construction): today's
   `select_model_alias_fallback_member(fallback_availability_mask(states))`.
2. If `mode == "round_robin"`:
   - `pool_mask = pool_availability_mask(pool_states)`.
   - If `any(pool_mask)` **or** `fallback_members` is empty: call
     `select_model_alias_pool_member(...)` exactly as today, including the
     all-unavailable "keep member 0 / current rotation choice, do not consume"
     short-circuit when there is no tail.
   - If the pool mask is all false **and** `fallback_members` is non-empty: do **not**
     read or advance the round-robin cursor. Select
     `len(members) + select_model_alias_fallback_member(fallback_availability_mask(tail_states))`
     into the concatenated member list. When the tail is also all-unavailable, that
     helper already preserves tail candidate 0 so ordinary Grok/provider diagnostics
     explain the failed launch (this is the behavior change versus today's all-down
     pool, which preserved pool member 0 — and it is the requested behavior).

Put this in one function, for example
`select_model_alias_selector_index(alias, selector, states, *, consume) -> int`, and
call it from:

- `_resolve_model_alias_result` (both the normal selector branch and the
  hard-disabled-override branch),
- `model_alias_selector_details`,
- the raw-selector path in `model_launch_settings.py`.

`consume=True` advances the pool cursor only when a **pool** member is selected.
Diverting to the tail never writes `llm_lb.json`.

### 4.1 What "unavailable" means

The user-facing phrase "not installed or disabled" maps onto today's
`MemberAvailability.UNAVAILABLE`:

- CLI missing / unregistered / **hard**-disabled → unavailable → skipped in a pool,
  divert to the tail when the whole pool is unavailable.
- **Soft**-disabled → `SPARING`. `pool_availability_mask` treats an all-sparing pool as
  still rotatable, so a last-resort tail is **not** used. A soft disable of the tail
  candidate does not skip it (`fallback_availability_mask`), matching `||`.

Availability remains the cached CLI-installation probe plus the captured disable
snapshot. There is still no retry after an authentication, model, or runtime failure.

## 5. Implementation

### 5.1 `src/sase/llm_provider/load_balancing.py`

- Extend the module docstring: parentheses group a `|` pool as the primary of a
  last-resort `||` tail; unparenthesized mix is still rejected.
- Add `fallback_members` to `ModelAliasSelector` as in §3.1. Update `normalized`. Keep
  `fingerprint` pool-only.
- Refactor parsing so the flat `|` / `||` split is a helper. Detect a leading
  parenthesized pool with a matching close paren (reject nested `(`), then:
  - no trailing text → parse the inside as a pool (grouping);
  - trailing `|| …` → parse the inside as a pool and the remainder as a fallback chain
    (a single target with no `|` becomes a one-member tail);
  - any other trailing text, unmatched parens, or parens that are not wrapping the whole
    leading pool → `ModelAliasSelectorError`.
- Keep the existing mix error for unparenthesized leftover `|` after `||` is stripped.
- Add `select_model_alias_selector_index` per §4. Existing
  `select_model_alias_pool_member` / `select_model_alias_fallback_member` stay the
  primitives; do not change their signatures.

Error messages stay lowercase, no trailing period, and actionable. Pin at least:

| Input                    | Message (wording may be refined, meaning must match)                   |
| ------------------------ | ---------------------------------------------------------------------- |
| `A \| B \|\| C`          | existing `cannot mix '\|' load balancing with '\|\|' ordered fallback` |
| `(A \|\| B) \|\| C`      | parentheses may only wrap a `\|` load-balanced pool                    |
| `(A \| B \|\| C)`        | same / cannot mix inside the group                                     |
| `((A \| B)) \|\| C`      | nested parentheses are not allowed                                     |
| `(A \| B) \|\| (C \| D)` | last-resort candidates cannot be parenthesized pools                   |
| `(A \| B) \|\|`          | empty last-resort candidate                                            |
| `(A \| B) \|\| 2 C`      | ordered fallback chains cannot weight candidates                       |

### 5.2 Resolution, details, validation

`src/sase/llm_provider/model_alias_resolution.py`:

- Resolve `selector.members` **and** `selector.fallback_members`. Fail closed if any
  concatenated member is invalid, matching today's nested-selector behavior.
- Populate `ModelAliasSelectorMember.last_resort` (`False` for pool members, `True` for
  the tail). Weights on tail members stay `1`.
- `model_alias_selector_details` concatenates both groups in order, then uses
  `select_model_alias_selector_index(..., consume=False)` for `selected`.
- `validate_model_alias_selector_value` walks both groups. Label pool rows "pool member
  N" and tail rows "last-resort candidate N". Nested-selector, cycle, depth, and
  unknown-alias errors apply equally to the tail.

`src/sase/llm_provider/model_launch_settings.py`: the raw-selector path must use the
same concatenated resolve + `select_model_alias_selector_index` helper. Scalar launch
fields (`default_model`, lander models) already accept any parseable selector via
`_is_valid_model_expression`; a tailed expression must remain valid there.

`src/sase/llm_provider/model_alias_policy.py`: `_validate_target` already parses the
target; after the parser accepts the new grammar, shipped YAML with a tail will load.
Update `test_every_shipped_selector_member_names_a_registered_provider_model` so it
iterates **concatenated** members (pool + tail), otherwise the new Grok candidate is not
catalog-checked.

### 5.3 Shipped `@large` default

In `src/sase/llm_provider/model_alias_defaults.yml` set `aliases.large.target` to
exactly:

```text
(claude/opus@xhigh | codex/gpt-5.6-sol@xhigh) || grok/grok-4.6@xhigh
```

Keep `@large` as a `target` entry (graph shape unchanged). Optionally tighten the
description so ACE/docs mention Grok as last resort when Claude and Codex are
unavailable; do not retarget `@xsmall` / `@small` / `@medium` / `@xlarge`.

`tests/_model_alias_defaults_fixture.py` must **not** change. It owns a frozen
two-member `@large` pool for generic resolution tests; a dedicated real-defaults test
owns the shipped value (same split as the `@smartest` ordered-fallback change).

Update the commented grammar example in `src/sase/default_config.yml` so `large:`
illustrates the parenthesized-pool form rather than claiming operators cannot be
combined at all. Commented values stay examples, not a second copy of the shipped
defaults.

### 5.4 Doctor

In `src/sase/doctor/checks_config_model_aliases.py`, keep entering the selector-note
branch for any target containing `|` (compound expressions included). Split
`selector.members` on `last_resort`:

- Pool-member unavailable / soft-disabled notes stay as they are, and count "available
  pool members" **excluding** the tail.
- When the pool has no available members **and** a tail member is selected, say the
  last-resort fallback currently selects that candidate — not "current pool member is
  retained for diagnostics."
- When pool **and** tail are all unavailable, say the first last-resort candidate is
  retained for provider diagnostics.
- Do not emit "last-resort candidate is unavailable" notes while the pool still has an
  available member; the tail is unused in that case.

### 5.5 ACE surfaces

Presentation-only. No selector semantics in TUI code — parse, compose, and validate stay
in `llm_provider`.

- `models_panel_selector.py` `compose_selector`: pass `fallback_members` through to
  `ModelAliasSelector` so confirm/header emit the canonical `(A | B) || C` spelling.
- `models_panel_selector_builder.py`:
  - Seed `_fallback_members` from `selector.fallback_members`.
  - Binding `f` adds a last-resort candidate via the existing member picker/effort flow
    (a last-resort row must be a single target, same rejection as `a`).
  - Existing `d` / reorder operate on the highlighted row, which may be a pool member or
    a last-resort candidate. Do not let reorder move a row across the pool/tail
    boundary.
  - `w`/`W` ignore last-resort rows (weights are pool-only), matching fallback mode.
  - `t` (toggle pool ↔ fallback) **refuses** while `_fallback_members` is non-empty,
    with a notify. Do not silently flatten `(A | B) || C` into `A || B || C`.
  - Confirm still requires at least two **pool** members; a tail is optional. Confirm
    composes through `ModelAliasSelector.normalized`.
  - Header shows the live normalized expression. Last-resort rows are visually distinct
    (for example a `fallback` prefix). Extend `_KEYS_HINT` with `f=fallback`.
- `models_panel_rendering_descriptions.py`: keep the `pool:` label when
  `selector_mode == "round_robin"`. After the last pool member, insert a `fallback:`
  separator before last-resort rows. `→` still marks the selected member, including when
  the tail is selected.
- `models_panel_rendering_rows.py`: the `pool N/M` chip counts only members with
  `last_resort is False`. Grok must not inflate `@large` to `pool 2/3` while it is
  sitting unused on the tail.
- Custom-edit hint in `models_panel_alias_edit.py`: mention `(A | B) || C` as the
  last-resort form. Unparenthesized mix must still notify "cannot mix"; a well-formed
  tailed expression must be accepted.
- History (`models_panel_history.py`) may include tail members in the recorded member
  list; they are real launch targets.

### 5.6 Docs and schema

- `docs/llms.md` selector section: replace "operators cannot be mixed" with the
  parenthesized last-resort rule, the availability mapping in §4.1, "tail selection does
  not consume the pool cursor," and the nested-selector prohibition. Run `just fmt-docs`
  so the generated `@large` row shows the new target; do not hand-edit the generated
  block.
- `docs/configuration.md`: same grammar correction on the builtin-alias and scalar
  launch-field paragraphs.
- `docs/ace.md`: Launch Control copy (member list, pool chip, builder keys) plus any
  `@large` edit example that currently shows a bare two-member pool as the shipped
  default.
- `src/sase/config/sase.schema.json`: update the `model_aliases.builtin` and
  `custom.*.model` prose descriptions. There is no regex on these strings, so no pattern
  change. Schema tests that use `@large` as a **valid example** may keep a bare pool;
  optionally add one compound example to `tests/test_config_schema_models.py`.

Do **not** hand-edit `CHANGELOG.md` (release-please owns it).

Do **not** edit SASE memory files.

## 6. Tests

Add to the existing suites; add one focused module rather than a parallel stack.

- `tests/llm_provider/test_load_balanced_aliases.py`
  - Keep the unparenthesized mix rejection.
  - `(A | B)` normalizes to `A | B`, mode `round_robin`, empty `fallback_members`.
  - `(A | B) || C` → pool members `(A, B)`, `fallback_members == ("C",)`,
    `normalized == "(A | B) || C"`.
  - Fingerprint of
    `(claude/opus@xhigh | codex/gpt-5.6-sol@xhigh) || grok/grok-4.6@xhigh` equals the
    fingerprint of the bare two-member pool.
  - Weighted pool inside parens: `(A | 3 B) || C` keeps weights `(1, 3)` on the pool and
    a weight-free tail.
  - Parser errors from the table in §5.1.
- New `tests/llm_provider/test_pool_last_resort_aliases.py`
  - Configure a custom alias with the compound expression. With both pool providers
    available, consuming launches round-robin the pool and never select the tail; peek
    does not consume; diverting to the tail does not call `_locked_state`.
  - Masks: only Claude → Claude; only Codex → Codex; neither, Grok available → Grok;
    nobody available → Grok (tail candidate 0) with no cursor write.
  - Soft-disable both pool providers (sparing) while Grok is preferred: still rotate the
    pool, do **not** divert.
  - Hard-disable both pool providers: divert to Grok.
  - Outer effort `@alias@high` applies to whichever member is selected, including the
    tail.
  - Nested selector via `@inner` in the pool or the tail fails closed; validation names
    the nested alias.
  - `model_alias_selector_details` marks `last_resort` and `selected` correctly before
    and after a consume, including the diverted case.
- `tests/llm_provider/test_load_balanced_alias_defaults.py`
  - With `real_model_alias_defaults`, pin shipped `@large`: both pool providers
    available never yields Grok; only Grok available yields `grok/grok-4.6@xhigh`; the
    parsed selector's pool members and tail match the YAML exactly.
- `tests/doctor/test_checks_config_model_aliases.py`
  - Unparenthesized mix is still a problem.
  - Compound with an unavailable pool and available tail notes that last-resort
    currently selects the tail, not that a pool member is retained.
- `tests/test_models_panel_selector_builder.py`
  - Seed from `(A | B) || C` populates pool + tail; `f` appends a last-resort candidate;
    confirm emits the canonical spelling; `t` notifies and stays in pool mode while a
    tail exists; `w`/`W` do not weight tail rows.
- `tests/test_models_panel_alias_rendering.py`
  - Pool chip ignores `last_resort` members. Description inserts a `fallback:` separator
    and can put `→` on a tail member.
- `tests/test_models_panel_edit.py` (or the custom-input mix test): unparenthesized mix
  still notifies; a tailed expression is accepted.

Visual snapshots: the builder footer hint changes, so
`tests/ace/tui/visual/test_ace_png_snapshots_models_panel_edit.py::test_models_panel_selector_builder_png_snapshot`
needs its golden refreshed via `--sase-update-visual-snapshots`. Navigation snapshots
use fabricated views, so they must not move unless a fixture is intentionally given a
tailed selector. Do not accept unrelated goldens.

## 7. Decisions and non-goals

- **Tale, not epic.** Parser, resolution, ACE, doctor, shipped default, and docs must
  land together: shipping the new `@large` string before ACE/doctor understand it would
  mis-render Launch Control and mis-report doctor notes. One follow-up agent can
  implement this from the grammar in §3–§4.
- **No feature flag.** The parenthesized form is additive; existing `|` and `||` values
  and the unparenthesized mix error are unchanged. Last-resort tails are a config
  expression users choose permanently, which flag guidance excludes. The shipped
  `@large` change is a default, not a beta route — users who want the old pool keep
  setting `model_aliases.builtin.large` to the two-member `|` value.
- **No Rust core work.** The selector stack is Python-only today (`sase_core_rs` exposes
  no alias/load-balancing surface). This extends that stack in place. Moving it behind
  the core boundary is a separate epic.
- **No `(A | B) || (C | D)`**, no `C || (A | B)`, no nested parens, no runtime retry. A
  pool on the right-hand side is follow-up work, not a v1 requirement.
- **State schema stays `version: 1`.** No cursor migration: the `@large` pool membership
  (and therefore fingerprint) is unchanged.
- **TUI performance.** Parsing a short alias string with a linear paren scan is fine. Do
  not add filesystem, glob, or extra resolution work on the Models-panel render path;
  keep using the existing per-token alias-resolution memoization.
- **Out of scope:** retuning `@xsmall` / `@small` / `@medium` / `@xlarge`; putting
  selectors in `%model`; usage-limit policy changes; probing whether an installed `grok`
  binary is really Grok Build.

## 8. Verification

```bash
just install
just check
just test-visual
```

Also run the focused LLM-provider / doctor / Models-panel tests listed in §6, then
`just fmt-docs` and confirm a second `just fmt-docs` is a no-op. Inspect the diff for:

- the exact `@large` string in `model_alias_defaults.yml`;
- fingerprint stability between the old pool and the tailed form;
- no frozen-fixture churn;
- no hand-edited generated docs block;
- only the selector-builder PNG golden (and any description-row golden that
  intentionally shows a tail) accepted.

Sanity-check `sase doctor -C config.model_aliases` against the shipped defaults: the new
expression must parse cleanly, and with Claude or Codex installed it must not claim Grok
is the current `@large` selection.
