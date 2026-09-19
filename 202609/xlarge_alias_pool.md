---
tier: tale
title: Put Grok in the @xlarge pool
goal: "The shipped @xlarge alias round-robins claude/opus@xhigh,
  codex/gpt-5.6-sol@xhigh, and grok/grok-4.6@xhigh as equal pool members.

  "
size: medium
proposed_by: bbugyi200.apollo.0o
create_time: 2026-09-19 09:04:48
status: wip
---

# Plan: Put Grok in the @xlarge pool

Retarget the shipped `@xlarge` builtin so Claude and Codex use the current
maximum-effort models at `xhigh`, and so Grok is a load-balanced pool member instead of
a last-resort tail.

## Outcome

`@xlarge` must resolve from this exact selector:

```text
claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh
```

That is the same three-provider `|` pool shape as shipped `@large`, with every member at
`xhigh`. It replaces today's last-resort expression:

```text
(claude/claude-fable-5@high | codex/gpt-6-astra@high) || grok/grok-4.6@xhigh
```

After this change, when Claude, Codex, and Grok CLIs are all installed, `@xlarge`
round-robins all three and Grok consumes the pool cursor. Grok is no longer withheld
until Claude and Codex are unavailable.

## Why this is a tale

One follow-up agent can land the YAML default, regenerate the alias table, rewrite the
few real-defaults tests that still pin last-resort `@xlarge` semantics, and update the
prose that currently calls Grok a last-resort `@xlarge` candidate. There is no phase
split, no Rust-core work, and no feature flag.

## Background

`src/sase/llm_provider/model_alias_defaults.yml` is the single source of truth for
shipped size-alias targets and descriptions. User config may override
`llm_provider.model_aliases.builtin.xlarge`; this tale only changes the shipped default.

Per-member `@effort` suffixes are already valid selector grammar. No parser, resolver,
or provider-adapter change is required:

- Claude's `opus` already accepts `--effort xhigh` (and `max`).
- Codex's `gpt-5.6-sol` already accepts `--effort xhigh` and does **not** accept `max`.
- Grok's `grok-4.6` already accepts `--effort xhigh` and does **not** accept `max`.

So `xhigh` is the highest effort every provider in the new pool honors natively. Do not
point Claude at `@max` and do not remap `max` to `xhigh` inside a provider adapter.

`claude-fable-5` and `gpt-6-astra` stay in the catalogs, short-alias tables, picker, and
completion tests. They are no longer the shipped `@xlarge` members.

Last-resort `(A | B) || C` remains valid user-configured grammar. Only the **shipped**
`@xlarge` value leaves that form. Custom-alias and frozen-fixture last-resort coverage
must keep working.

## Frozen fixture vs shipped values

`tests/_model_alias_defaults_fixture.py` is an intentional frozen graph-shape contract
whose values are deliberately distinct from the shipped file. Graph shape here means
YAML `target` vs `fallback` keys, not selector mode. `@xlarge` already has a `target`
and keeps one, so **do not edit the frozen fixture**.

Frozen `@xlarge` today is itself a last-resort selector
(`(claude/opus@max | codex/gpt-5.6-sol@max) || grok/grok-4.6@xhigh`). Leave that frozen
shape in place so last-resort unit tests that do **not** request
`real_model_alias_defaults` still have a last-resort `@xlarge`. In particular, leave
these frozen-based tests alone unless a rename is required for honesty:

- `test_size_aliases_own_selector_targets` in
  `tests/llm_provider/test_config_role_aliases.py` (asserts frozen `@xlarge`
  `fallback_members`)
- `test_xlarge_pool_skips_unavailable_member_before_last_resort` in
  `tests/llm_provider/test_load_balanced_alias_defaults.py`

Zero-friction value edits (`sase-f1` /
`plan:202608/zero_friction_model_alias_defaults.md`) still apply to YAML **values** that
keep selector mode. This tale **does** change selector mode (last-resort → three-member
`|` pool), so tests that use `real_model_alias_defaults` and pin shipped last-resort
members or last-resort reservation behavior will fail and must be rewritten. Do not try
to restore the zero-friction contract by deleting those tests; update them.

## Implementation

### 1. Change the shipped `@xlarge` target and description

In `src/sase/llm_provider/model_alias_defaults.yml`, set `aliases.xlarge` to match the
other three-member pools in that file:

- `target`: `claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh`
  (single-line string, same style as `aliases.large.target`)
- `description`: `Extra-large launch alias for maximum-effort work.`

Drop the "Grok is last resort when Claude and Codex are unavailable" clause. Do not
change `@xsmall`, `@small`, `@medium`, or `@large`.

In `src/sase/default_config.yml`, the commented `model_aliases.builtin` block is labeled
as grammar examples, not a second copy of the shipped defaults. Still update the
commented `xlarge:` example to the new three-member `|` pool so it does not keep
advertising last-resort as the `@xlarge` default. Keep the existing comment that
`(A | B) || C` is valid last-resort grammar.

Do not hand-edit `CHANGELOG.md`. Do not edit SASE memory notes. Do not edit `sase-core`.

### 2. Rewrite shipped-`@xlarge` real-defaults tests

These tests request `real_model_alias_defaults` and currently encode last-resort
Fable/Astra/Grok-tail behavior. Update them to the new pool.

**`tests/llm_provider/test_ordered_fallback_aliases.py`** —
`test_shipped_xlarge_pool_uses_last_resort_grok`:

- Rename to `test_shipped_xlarge_round_robins_claude_codex_grok` (mirror
  `test_shipped_large_round_robins_claude_codex_grok`).
- Assert
  `selector.members == ("claude/opus@xhigh", "codex/gpt-5.6-sol@xhigh", "grok/grok-4.6@xhigh")`
  and `selector.fallback_members == ()`.
- With every target available, three `consume=True` resolutions must be `claude/opus`,
  `codex/gpt-5.6-sol`, `grok/grok-4.6` (Grok now rotates in).
- With only Codex available: `codex/gpt-5.6-sol` at `xhigh`.
- With only Grok available: `grok/grok-4.6` at `xhigh`.

Leave other tests in that file that use Fable/Astra as **custom** selector examples
(they are grammar coverage, not shipped-default pins).

**`tests/llm_provider/test_load_balanced_alias_defaults.py`**:

- Add `@xlarge` to `test_packaged_defaults_select_correct_effort_per_provider` with
  `claude/` → `("claude/opus", "xhigh")`, `codex/` → `("codex/gpt-5.6-sol", "xhigh")`,
  `grok/` → `("grok/grok-4.6", "xhigh")`.
- Optionally fold the all-available rotation assertion into a sibling of
  `test_shipped_large_round_robins_claude_codex_grok` if the renamed ordered-fallback
  test does not already cover it. Do not duplicate the same three-consume assertion in
  two files.

**`tests/llm_provider/test_provider_priority_routing.py`** —
`test_shipped_large_honors_grok_priority_while_xlarge_keeps_last_resort`:

- Today, grok priority plus a soft-disabled Claude still resolves shipped `@xlarge` to
  Codex `gpt-6-astra@high` because Grok is only a tail.
- After this change both `@large` and `@xlarge` must honor grok priority the same way,
  because Grok is a pool member in both.
- Rename to a name that states that (for example
  `test_shipped_large_and_xlarge_honor_grok_priority`).
- Keep the same routing context (priority `grok`, Claude soft-disabled).
- `@large` stays `("grok", "grok-4.6", "high")`.
- `@xlarge` becomes `("grok", "grok-4.6", "xhigh")`.

**`tests/test_pooled_alias_single_consumption_soft_disable.py`** —
`test_healthy_tail_does_not_invalidate_all_soft_primary_reservation`:

- This test uses shipped `@xlarge` only because it is currently the shipped last-resort
  alias. After the change, a healthy Grok **pool member** would divert a soft Claude
  reservation, which is a different rule from a healthy **tail**.
- Retarget the test onto a **custom** last-resort alias such as
  `(claude/opus | codex/gpt-5.5) || grok/grok-4.6`, following
  `test_soft_primary_reservation_survives_healthy_tail` in
  `tests/llm_provider/test_provider_priority_routing.py`.
- Do not drop last-resort reservation coverage, and do not keep it coupled to shipped
  `@xlarge`.

Sweep `rg -n 'real_model_alias_defaults' tests` together with `claude-fable-5@high`,
`gpt-6-astra@high`, `fallback_members`, and `@xlarge` so no other shipped last-resort
pin remains. Parser tests that hardcode
`(claude/opus@xhigh | codex/gpt-5.6-sol@xhigh) || grok/grok-4.6@xhigh` as a **grammar
example** (for example `tests/llm_provider/test_load_balanced_aliases.py`) stay; they
are not shipped-default pins.

### 3. Regenerate the alias table and fix last-resort prose

Run `just fmt-docs` so `tools/render_model_alias_docs` rewrites the generated
`model-alias-defaults` block in `docs/llms.md` from the YAML, then `just fmt-md`. Do not
hand-edit that generated block. After regeneration it must show the new `|` pool and the
shortened description.

Then update **behavior** prose that currently says Grok is the last `@xlarge` fallback
candidate. Name the five size aliases as load-balanced pools and point at the generated
table for the live members. Do not leave a sentence that calls shipped `@xlarge` a
pool-with-tail, ordered fallback, or last-resort.

Required sites (verified by search; sweep again after editing):

- `README.md` — Grok is reached through the shipped `@xsmall`, `@small`, `@medium`,
  `@large`, and `@xlarge` pools (drop "last candidate in the `@xlarge` fallback").
- `docs/getting_started.md` — same. Update
  `tests/test_docs_getting_started_providers.py` so it no longer asserts
  `"as the last \`@xlarge\` fallback candidate"` and instead asserts the new pools
  wording (keep the existing Muse/Grok autodetection assertions).
- `docs/llms.md` provider-selection step 5 (~line 190) and Grok Build Integration →
  Selection (~line 889) — Grok is a member of all five shipped size pools, not the last
  `@xlarge` ordered-fallback candidate.
- `docs/llms.md` Role Aliases override paragraph (~line 1862) — an override on `@xlarge`
  suspends its **pool rotation**, like `@xsmall`/`@small`/`@medium`/`@large`. Do not say
  it suspends last-resort selection.
- `docs/llms.md` Grok Reasoning Effort (~line 946) — keep the true claim that a
  Grok-selected `@xlarge` launch passes `--effort xhigh`. The nearby sentence that
  Claude and Codex `@xlarge` candidates also run at `@xhigh` becomes accurate; keep it.
  Delete any leftover implication that Codex `@xlarge` still carries `@max`.
- `docs/agent_providers.md` Grok selection (~line 158) — today it incorrectly calls both
  `@large` and `@xlarge` ordered fallbacks. Say the five shipped size aliases are
  load-balanced pools that can select Grok when a `grok` executable is available.
- `docs/agent_providers.md` Effort ceiling (~line 201) — drop "Codex's `@xlarge`
  candidate remains `@max`" and the skip-path story for the shipped default. The shipped
  Codex member is `codex/gpt-5.6-sol@xhigh`. Unsupported `max` on a **user-configured**
  Codex target may still be mentioned as a general caveat, not as the shipped `@xlarge`
  default.
- `docs/configuration.md` Grok env paragraph (~line 4944) — same pools wording as
  `docs/llms.md` selection.
- `docs/ace.md` Launch Control override bullets (~lines 4023 and 4079) — list shipped
  `@xlarge` with the other size **pools**, not as "the `@xlarge` pool-with-tail". Keep
  general last-resort grammar documentation for user-composed selectors.
- `docs/ace.md` Examples (~line 4147) — the "Leave `@xlarge` implicit" example currently
  quotes the Fable/Astra last-resort expression. Quote the new three-member `|` pool (or
  name `@xlarge` and point at the generated table). Do not say "last-resort is currently
  selected".

Leave last-resort **grammar** documentation in `docs/llms.md` and `docs/ace.md` (the
`(A | B) || C` form, cursor rules, healthy-tail reservation). That grammar is still
real; it is just no longer the shipped `@xlarge` value.

`docs/xprompt.md`'s Grok comment already says "never auto-detected; also in alias
pools". Leave it unless a later sweep still claims `@xlarge` is a fallback.

Do not edit generated `site/` HTML.

### 4. Straggler sweep

After the edits, confirm these patterns no longer describe **shipped** `@xlarge`:

```bash
rg -n 'claude-fable-5@high|gpt-6-astra@high|pool-with-tail|last `@xlarge`|@xlarge` fallback|@xlarge` ordered fallback' \
  README.md docs src/sase/llm_provider/model_alias_defaults.yml \
  src/sase/default_config.yml tests
```

Allowed leftovers:

- Catalog/picker/completion tests that mention `claude-fable-5` or `gpt-6-astra` as
  published models.
- Custom-selector last-resort examples in tests and docs.
- Frozen-fixture `@xlarge` last-resort values.
- Grammar comments that define `(A | B) || C` without claiming shipped `@xlarge` uses
  it.

## Verification

1. Run `just install` first; this ephemeral workspace may have stale or missing
   development dependencies.
2. Run `just fmt` (it includes `fmt-docs` then `fmt-md`). A second `just fmt` must
   produce no additional diff. Confirm the generated `docs/llms.md` table matches the
   YAML and was not hand-edited.
3. Run the rewritten real-defaults tests, the getting-started docs test, and the
   packaged-effort parametrize, then run `just check` for whole-repository lint gates
   and the diff-scoped test lane. If it broadens or reports an unusual selection, run
   `just check-full` only through `/sase_monitor`.
4. Do not regenerate TUI PNG goldens unless a visual test actually fails. Models-panel
   snapshots patch `AliasView` fixtures and do not read the shipped YAML.
5. Inspect the final diff to confirm:
   - the exact `claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh`
     expression
   - no other shipped alias targets changed
   - no frozen-fixture churn
   - no catalog/picker removals of Fable or Astra
   - no Grok/Codex adapter remapping of `max` to `xhigh`
   - no `CHANGELOG.md` or memory-file edits

## Risks and out of scope

- **Grok now takes one third of `@xlarge` launches** when all three CLIs are installed,
  because it consumes the pool cursor. That is the requested product change, not a
  regression.
- **Soft-disable and provider-priority behavior for `@xlarge` now match `@large`.** A
  grok priority can select Grok on `@xlarge`; a healthy Grok pool member can replace a
  soft-disabled Claude reservation. Last-resort "healthy tail does not invalidate a soft
  primary" no longer applies to shipped `@xlarge`. Keep that rule covered on a custom
  last-resort alias.
- **Availability is still CLI presence**, not authentication or invocation success. Do
  not add runtime-failure retries.
- **Overrides remain authoritative.** Persistent, temporary, launch-scoped, and
  approval-selected model overrides continue to bypass the shipped `@xlarge` selector.
- Out of scope: changing `@xsmall` / `@small` / `@medium` / `@large`; removing Fable or
  Astra from the catalog; remapping `max` inside provider adapters; adding a feature
  flag; editing SASE memory files; changing `sase-core`; changing last-resort selector
  grammar or the Models-panel last-resort builder.
