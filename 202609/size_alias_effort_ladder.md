---
tier: tale
title: Descend the effort ladder across the five shipped size aliases
goal:
  Every shipped size-alias member runs its model at xhigh the first time that model
  appears (scanning xlarge down to xsmall) and one rung lower on each reappearance, with
  Antigravity's Gemini 3.8 Flash taking Grok's xsmall slot, and a decisions strand
  recording the rule.
size: medium
proposed_by: bbugyi200.apollo.0o.f0
create_time: 2026-09-20 06:31:53
status: wip
---

# Plan: Descend the effort ladder across the five shipped size aliases

## Goal

Retune `src/sase/llm_provider/model_alias_defaults.yml` so the five built-in size
aliases follow one rule: walking the aliases from `@xlarge` down to `@xsmall`, a model's
**first** appearance runs at `xhigh`, and every later appearance drops exactly one rung
on the canonical effort ladder. Replace Grok in `@xsmall` with the best Gemini model
SASE supports. Record the rule — and the companion rule that every alias should use a
pool and/or fallback for LLM-provider redundancy — as a new `decisions` memory-web
strand.

## Read this first: two facts the implementer must not rediscover the hard way

### 1. The approved `@xlarge` change is NOT in the repo

The user approved `plan:202609/xlarge_alias_pool.md` (turn `xlarge_alias_pool`), which
was to set `@xlarge` to
`claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh`. That work **never
landed**. `master` (`5082ca8cf3`) and every branch still carry the pre-plan value:

```yaml
xlarge:
  target: "(claude/claude-fable-5@high | codex/gpt-6-astra@high) || grok/grok-4.6@xhigh"
```

`git log --all -- src/sase/llm_provider/model_alias_defaults.yml` shows no commit
implementing it; the coding agent's turn ended without a captured final response.

**This plan therefore re-applies that approved `@xlarge` change as step 1.** That is a
deliberate assumption, called out here so the plan reviewer can reject it: the user's
prompt says "the **other** builtin model aliases", which only reads correctly if
`@xlarge` is already `opus | gpt-5.6-sol | grok-4.6`, all at `xhigh`. Under that
reading, every other alias value falls out of the ladder rule with no contradiction (see
the table below). If the reviewer instead wants `@xlarge` to keep
`claude-fable-5`/`gpt-6-astra`, then those two members move to `@xhigh`, `@large` gains
`opus@xhigh | gpt-5.6-sol@xhigh` (their first appearance), and `@medium`/`@small` are
unchanged from this plan — say so in gate feedback and the implementer adjusts only
`@xlarge` and `@large`.

### 2. Antigravity supports **no** reasoning effort — `agy/...@xhigh` would raise

`AgyProvider.invocation_option_args` (`src/sase/llm_provider/agy.py`) calls
`effort_cli_args(..., supported={})`. Per `src/sase/llm_provider/_effort_args.py`, an
**explicit** effort against an empty support map raises `LLMInvocationError`; only a
config-_default_ effort is warn-and-skipped. Antigravity instead bakes the effort into
the model slug: the catalog publishes `gemini-3.8-flash-high`, `-medium`, and `-low`,
and `gemini-3.8-flash-high` is the top variant of the newest Gemini SASE knows (the only
other high-tier Gemini is the older `gemini-3.1-pro-high`).

So the xsmall Gemini member is **`agy/gemini-3.8-flash-high`, with no `@effort` suffix**
— the `-high` in the slug _is_ the maximum-effort selection. This matches the precedent
set by `630f4ea717` ("no separate effort suffix"). Do not write
`agy/gemini-3.8-flash-high@xhigh`.

## Target values

Canonical ladder (`src/sase/xprompt/effort.py`):
`none < minimal < low < medium < high < xhigh < max`. "One rung lower" means one step
down this list.

| Alias     | Claude                        | Codex                      | Third member                | Notes                                            |
| --------- | ----------------------------- | -------------------------- | --------------------------- | ------------------------------------------------ |
| `@xlarge` | `opus@xhigh` (1st)            | `gpt-5.6-sol@xhigh` (1st)  | `grok-4.6@xhigh` (1st)      | re-applies the approved plan; `\|` pool, no tail |
| `@large`  | `opus@high` (2nd)             | `gpt-5.6-sol@high` (2nd)   | `grok-4.6@high` (2nd)       | **already correct — do not touch**               |
| `@medium` | `sonnet@xhigh` (1st)          | `gpt-5.6-terra@xhigh`(1st) | `grok-4.6@medium` (3rd)     | Claude/Codex rise from `@medium`                 |
| `@small`  | `sonnet@high` (2nd)           | `gpt-5.6-terra@high` (2nd) | `grok-4.6@low` (4th)        | Claude/Codex rise from `@low`                    |
| `@xsmall` | `claude-haiku-4-5@xhigh`(1st) | `gpt-5.6-luna@xhigh`(1st)  | `agy/gemini-3.8-flash-high` | Gemini replaces Grok; Grok would need a 5th rung |

Grok's four rungs (`xhigh → high → medium → low`) already match `master` for `@large`,
`@medium`, and `@small`; only its `@xlarge` promotion out of the last-resort tail
changes. That is the internal consistency check that the ladder rule and the approved
`@xlarge` target agree.

Accepted consequence of the rule, worth stating so it is not later mistaken for a bug:
`@small` runs Claude at `high`, the same rung `@large` uses, because Sonnet's second
appearance and Opus's second appearance land on the same rung. Size ordering is carried
by the _model_, not by the effort alone.

The final YAML (descriptions are free to be re-worded; `@xlarge`'s must lose its "Grok
is last resort" clause):

```yaml
xsmall:
  target:
    "claude/claude-haiku-4-5@xhigh | codex/gpt-5.6-luna@xhigh |
    agy/gemini-3.8-flash-high"
small:
  target: "claude/sonnet@high | codex/gpt-5.6-terra@high | grok/grok-4.6@low"
medium:
  target: "claude/sonnet@xhigh | codex/gpt-5.6-terra@xhigh | grok/grok-4.6@medium"
large:
  target: "claude/opus@high | codex/gpt-5.6-sol@high | grok/grok-4.6@high"
xlarge:
  target: "claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh"
```

`claude-fable-5` and `gpt-6-astra` drop out of every shipped alias. Leave both in their
provider catalogs, short-alias maps, and docs — they stay reachable by explicit
`%model:`. Do **not** remove them.

## Steps

### 1. Retune `src/sase/llm_provider/model_alias_defaults.yml`

Apply the five targets above. `@large` keeps its exact current target string. Update
`@xlarge`'s description so it no longer claims a Grok last resort, and `@xsmall`'s if it
names its members. Keep the file's leading comment block accurate — it documents
`(A | B) || C` as a supported shape, which is still true even though no shipped alias
uses it any more.

### 2. Update the shipped-value tests

- `tests/llm_provider/test_load_balanced_alias_defaults.py`
  - `test_packaged_defaults_select_correct_effort_per_provider`: update the `@xsmall`,
    `@small`, and `@medium` expectation dicts to the new efforts; `@large` is unchanged.
    The `@xsmall` entry must replace its `"grok/"` key with an `"agy/"` key expecting
    `("agy/gemini-3.8-flash-high", None)`. Add an `@xlarge` case.
  - `test_shipped_large_round_robins_claude_codex_grok`: unchanged; assert it still
    passes rather than editing it.
  - `test_shipped_size_aliases_have_no_antigravity_member`: **this test now contradicts
    the requested behavior** — it asserts no shipped alias has an `agy/` member. Replace
    it with a test that pins the intended shape: `@xsmall` has exactly one `agy/` member
    and it is `agy/gemini-3.8-flash-high`, and the other four aliases have none. Do not
    simply delete it; the "only `@xsmall` may carry Antigravity" invariant it was
    guarding (from `630f4ea717`) is still the intent.
  - `test_xlarge_pool_skips_unavailable_member_before_last_resort`: it drives the frozen
    fixture, not shipped values, so it keeps working. Confirm, do not edit.
  - Add `test_shipped_xlarge_round_robins_claude_codex_grok`, mirroring the `@large`
    test: selector members are exactly
    `("claude/opus@xhigh", "codex/gpt-5.6-sol@xhigh", "grok/grok-4.6@xhigh")` and
    `fallback_members == ()`.
  - Add a rule-level regression test that reads `implicit_alias_targets()` and asserts
    the ladder invariant directly: walking `xlarge, large, medium, small, xsmall`, each
    `provider/model`'s first occurrence carries `xhigh` and each later occurrence is
    exactly one index lower in `EFFORT_LEVELS_ORDERED`. Exempt members whose provider
    publishes no effort levels (Antigravity) — assert those carry no `@effort` suffix at
    all instead. This test is what keeps the rule from silently rotting.

- `tests/test_pooled_alias_single_consumption_soft_disable.py`
  - `test_healthy_tail_does_not_invalidate_all_soft_primary_reservation` is the one real
    risk: its premise is that `@xlarge` has a _last-resort tail_, which this change
    removes. Re-point it at a selector that still has a tail (build a custom
    `(A | B) || C` alias in the test, or use the frozen fixture's `@xlarge`, which keeps
    its `(claude/opus@max | codex/gpt-5.6-sol@max) || grok/grok-4.6@xhigh` shape) rather
    than the shipped `@xlarge`. The behavior under test — a healthy tail does not
    invalidate a soft-disabled primary's reservation — must keep a tail to test.
  - `test_shipped_large_soft_disable_after_bootstrap_selects_codex_not_tail` uses
    `@large`, which is unchanged; confirm it still passes.

- `tests/_model_alias_defaults_fixture.py`: **do not touch.** Its header states shipped
  _value_ changes need no fixture change, and the alias graph _shape_ is unchanged for
  four of five aliases. `@xlarge` moves from a last-resort target to a plain pool
  target, but the fixture's `_shape_signature` only distinguishes target / fallback /
  empty, so `FROZEN_ALIAS_SHAPE` still matches and
  `test_shipped_defaults_match_the_frozen_graph_shape` stays green. Verify that claim by
  running the test; if it does fail, that is a genuine shape change and the fixture
  header authorizes the edit.

- `tests/test_docs_getting_started_providers.py`:
  `test_getting_started_muse_grok_wording_separates_provider_selection` asserts the
  exact getting-started sentence. Grok leaves `@xsmall` and stops being the `@xlarge`
  last resort, so both asserted phrases change. Update the assertions and the doc prose
  (step 3) together.

### 3. Update prose docs

- `docs/getting_started.md` (~line 73): Grok is reachable through the shipped `@small`,
  `@medium`, `@large`, and `@xlarge` pools. Drop `@xsmall` and drop the "last `@xlarge`
  fallback candidate" clause — `@xlarge` has no fallback tail any more.
- `README.md` (~line 77): same two corrections.
- `docs/agent_providers.md`, Grok Build section (~line 156): currently says the
  `@xsmall`/`@small`/`@medium` pools can select Grok "as can the last candidate in the
  `@large` and `@xlarge` ordered fallbacks". Both halves are now wrong (`@large` has
  been a plain pool since `17084a6e27`). Rewrite as: whenever a `grok` executable is
  available, the shipped `@small`, `@medium`, `@large`, and `@xlarge` round-robin pools
  can select it.
- `docs/llms.md`: the alias table lives between
  `<!-- BEGIN GENERATED: model-alias-defaults -->` markers. **Do not hand-edit it** —
  run `just fmt-docs` (which runs `tools/render_model_alias_docs`) and commit the
  result.

Leave the Antigravity CLI section of `docs/agent_providers.md` alone unless the
implementer finds a stale alias-membership claim in it; as of `5082ca8cf3` it makes no
routing claim.

### 4. Add the decisions-web strand

Create `sase/memory/decisions/size-alias-effort-ladder.md` following the shape of
`sase/memory/decisions/check-full-is-explicit.md` — frontmatter with `keyword`,
`aliases`, `summary`, and `metadata: {status: accepted, decided: 2026-09-20}`, then a
body of `**Claim.**` / `**Why.**` / `**Cost.**` / `**Reopens when.**` paragraphs.

The record must state, as the accepted decision:

1. **The ladder.** Scanning the built-in size aliases from `@xlarge` down to `@xsmall`,
   a model's first appearance runs at `xhigh` and each later appearance drops exactly
   one rung of `EFFORT_LEVELS_ORDERED`. A model that would need a rung below the bottom
   of its provider's supported range does not belong in that alias — swap in a different
   model rather than repeating or floor-clamping a rung. That is precisely why Gemini,
   not a fifth Grok rung, holds the `@xsmall` slot.
2. **The exception.** A provider that publishes no reasoning-effort mechanism and bakes
   effort into the model slug — Antigravity today — expresses its rung by choosing the
   model variant (`gemini-3.8-flash-high`) and carries no `@effort` suffix. Attaching
   one is not a cosmetic error: it raises `LLMInvocationError` at invoke time, because
   an alias-supplied effort is explicit.
3. **Redundancy.** Every model alias should reach more than one LLM provider, through a
   `|` round-robin pool, a `||` ordered fallback, or a `(A | B) || C` last resort,
   whenever more than one provider can serve that size. A single-provider alias makes
   one vendor's outage or rate limit a total outage for that size.

Write the `**Why.**` around the two forces that actually produced the rule — a size
alias should buy capability through the _model_, not through starving a good model of
reasoning budget; and reusing one model across adjacent sizes without dropping its rung
collapses the distinction between those sizes. Name the credible rejected alternative:
per-alias hand-tuned efforts, rejected because five aliases times three members is
fifteen independently drifting values with no invariant a test can check. State the
`**Cost.**` honestly: rungs are coupled, so replacing one model in one alias can force
edits in the aliases below it, and `@small` ends up at the same Claude rung as `@large`.
Give a real `**Reopens when.**` — for example, a provider ships a non-linear effort
ladder, or measured quality at `xhigh` on a small model proves worse than the same model
at `high`.

Author `[[...]]` links for any other memory the strand names in prose. Do not hand-edit
`sase/memory/decisions.md`; its `<!-- sase:strands -->` roster is generated.

### 5. Republish memory

Run `sase memory init` to regenerate `AGENTS.md`, the provider shims, the memory README,
and the `decisions` roster. Commit the regenerated files with the strand. Never
hand-edit `AGENTS.md` or `CLAUDE.md`.

## Verification

1. Targeted first, before any full lane:

   ```sh
   .venv/bin/python -m pytest \
     tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_load_balanced_alias_defaults.py \
     tests/test_pooled_alias_single_consumption_soft_disable.py \
     tests/test_docs_getting_started_providers.py \
     tests/llm_provider/test_alias_view.py \
     tests/llm_provider/test_provider_priority_routing.py \
     tests/test_llm_provider_usage_limit_disable_enforcement.py
   ```

2. `just fmt-docs`, then re-run the doc tests.
3. `just check`.

`just check` will escalate to the full suite (`core-identity-changed`,
`src-data-asset`), because `model_alias_defaults.yml` is a packaged data asset. That
escalation is expected here and is not a reason to switch lanes. Per
`decisions:check-full-is-explicit`, do **not** run `just check-full`; a prior attempt on
this same file timed out after 90 minutes inside `just test-cost`.

## Constraints

- Exactly three members per alias, each a `|` pool with no `||` tail anywhere in the
  shipped file.
- Do not remove `claude-fable-5` or `gpt-6-astra` from any provider catalog, short-alias
  map, or doc.
- Do not edit `tests/_model_alias_defaults_fixture.py` unless
  `test_shipped_defaults_match_the_frozen_graph_shape` actually fails.
- Do not hand-edit the generated `docs/llms.md` alias table, `AGENTS.md`, or
  `CLAUDE.md`.
- Do not touch `../sase-core`; this is packaged Python data plus tests and docs, with no
  Rust wire change.
- Do not regenerate TUI PNG goldens unless a visual test actually fails.
- No `CHANGELOG.md` edit.
