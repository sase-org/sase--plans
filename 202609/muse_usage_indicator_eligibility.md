---
tier: tale
title: Count built-in size-alias pools as provider references
goal:
  A provider that appears only in a shipped built-in size-alias pool counts as
  referenced, so Muse becomes background-eligible for usage collection and its weekly
  window renders in the TUI header usage indicator.
size: small
proposed_by: bbugyi200.athena.0og
create_time: 2026-09-20 16:48:15
status: wip
---

# Count built-in size-alias pools as provider references so Muse becomes usage-eligible

## Problem

After epic sase-14c closed, the TUI header shows no usage window for the Muse provider,
even though every piece the epic built works correctly.

Muse usage data is collected and stored. `sase usage list` shows both windows:

```
provider provider=muse status=ok
window provider=muse key=session label="Muse 5-hour session" remaining="100% left" used_percent=0 reset="reset passed" age=29m freshness=unknown state=allowed source=probe scope=account
window provider=muse key=weekly label="Muse weekly all models" remaining="100% left" used_percent=0 reset="in 3h 18m" age=29m freshness=unknown state=allowed source=probe scope=account
```

The header is empty anyway because the indicator projection filters on the
_background-eligible_ provider set, and `eligible_usage_providers()` omits `muse`:

```
eligible: ('claude', 'codex', 'grok')
```

## Root cause

`eligible_usage_providers()` in `src/sase/llm_provider/usage/refresh.py:280` requires a
provider to be _referenced_ (or explicitly enabled) before it is background-eligible:

```python
explicit_enable = settings.providers.get(name) is True
if name not in referenced and not explicit_enable:
    continue
```

`_referenced_provider_ids()` (`src/sase/llm_provider/usage/refresh.py:471`) builds that
set by scanning only these targets:

```python
targets: list[str] = [
    get_default_model(),
    get_epic_lander_model(),
    get_big_epic_lander_model(),
    *get_builtin_model_aliases().values(),
    *get_custom_model_aliases().values(),
]
```

Every one of those misses Muse:

- `get_default_model()` is `@large`, `get_epic_lander_model()` is `@large`, and
  `get_big_epic_lander_model()` is `@xlarge`. These are _size alias references_, and
  `provider_for_resolved_target("@large")` returns `None` — the scan never expands an
  `@alias` into the pool it names.
- `get_builtin_model_aliases()` returns only the **user's overrides** of
  `llm_provider.model_aliases.builtin`. It is `{}` here, because the user has overridden
  no size alias. It is _not_ the shipped size-alias defaults.
- `get_custom_model_aliases()` yields exactly `claude`, `codex`, `grok`, and `opencode`
  — which is precisely the observed referenced set:
  `referenced: {'codex', 'claude', 'opencode', 'grok'}`.

The shipped size-alias pools live in `src/sase/llm_provider/model_alias_defaults.yml`
and are exposed by `implicit_alias_targets()`
(`src/sase/llm_provider/model_alias_policy.py:292`). **Nothing scans them.** Commit
`68d727bbf` added Muse to exactly those pools and nowhere else:

```
xsmall => claude/claude-haiku-4-5@xhigh | codex/gpt-5.6-luna@xhigh | agy/gemini-3.8-flash-high | muse/muse-spark-1.3-contributor@medium
small  => claude/sonnet@high | codex/gpt-5.6-terra@high | grok/grok-4.6@low | muse/muse-spark-1.3-contributor@high
medium => claude/sonnet@xhigh | codex/gpt-5.6-terra@xhigh | grok/grok-4.6@medium | muse/muse-spark-1.3-contributor@xhigh
large  => claude/opus@high | codex/gpt-5.6-sol@high | grok/grok-4.6@high
xlarge => claude/opus@xhigh | codex/gpt-5.6-sol@xhigh | grok/grok-4.6@xhigh
```

So Muse is a provider SASE really does launch agents on (`@xsmall`/`@small`/`@medium`),
yet it never counts as referenced, is never background-eligible, and is therefore
dropped from the header projection.

This is a pre-existing gap in `_referenced_provider_ids()`, not a defect in any sase-14c
phase. Epic sase-14c was the first change to put a provider _only_ in the shipped
size-alias pools, which is what exposed it.

The same gap explains the stale `age=29m freshness=unknown` on the stored Muse windows:
because Muse is not background-eligible, the 300s refresh cadence never probes it. The
stored values are leftovers from an explicit probe. One fix addresses both the missing
header indicator and the absent background collection.

### Proof the rest of the chain is correct

Running the real Rust projection against the real stored snapshot, changing only the
eligible set, is the whole difference:

```
=== eligible = (claude, codex, grok)          # what ships today
     claude weekly                 85.0
     claude weekly:claude-fable-5 100.0
     codex  codex:primary            0.0
     grok   included_weekly          0.0
=== eligible = (claude, codex, grok, muse)    # with the fix
     claude weekly                 85.0
     claude weekly:claude-fable-5 100.0
     codex  codex:primary            0.0
     grok   included_weekly          0.0
     muse   weekly                 100.0     <-- appears
```

Note that `muse session` is correctly absent in both runs: phase sase-14c.3's
`indicator.providers.muse.windows.session: never` default works, and
`weekly_all: always` selects the weekly window. The config policy, the Rust normalizer,
the probe, and the schema are all already right. Eligibility is the single blocker.

## Approach

Teach `_referenced_provider_ids()` to scan the **effective** built-in size-alias
targets: the shipped defaults from `implicit_alias_targets()`, with the user's
`get_builtin_model_aliases()` overrides applied on top (an override replaces the shipped
target for that alias name, matching resolution precedence in
`src/sase/llm_provider/model_alias_resolution_resolve.py:120`).

This keeps the existing meaning of "referenced" — a provider SASE could actually launch
an agent on — and simply stops the scan from having a blind spot. It deliberately does
not special-case Muse.

Keep the fix in Python. `_referenced_provider_ids()` is local plumbing that reads this
repo's Python model-alias config and provider registry; the shared backend behavior
(usage normalization and the indicator projection) already lives in `sase-core` and
needs no change. This respects the Rust core backend boundary: the selection policy is
already core and already correct, and only the Python-side input to it is wrong.

## Steps

1. **Expand built-in size-alias pools in the reference scan.**

   In `src/sase/llm_provider/usage/refresh.py`, change `_referenced_provider_ids()` to
   import `implicit_alias_targets` from `sase.llm_provider.model_alias_policy` and build
   the effective built-in size-alias map before scanning:
   - Start from `dict(implicit_alias_targets())`.
   - Apply `get_builtin_model_aliases()` on top so a user override of a size alias wins
     over the shipped default.
   - Feed that map's values into `targets` alongside the existing entries, replacing the
     bare `*get_builtin_model_aliases().values()` entry (its values are now subsumed).

   Keep the existing per-target tokenization. While editing it, make the tokenizer
   tolerate the documented parenthesized last-resort grammar `(A | B) || C` by stripping
   `(` and `)` from each token before the `@` split, so a pooled member is not lost as
   `(claude/opus`. Continue skipping tokens that resolve to no provider, which keeps a
   bare `@alias` reference harmless (its pool is now scanned directly anyway).

   Add a short comment recording _why_ the shipped defaults are scanned: a provider that
   appears only in a size-alias pool is still a provider SASE launches agents on.

2. **Cover the regression with unit tests.**

   Add tests to `tests/llm_provider/test_usage_eligibility.py`, which already owns
   `_referenced_provider_ids` and `eligible_usage_providers` coverage:
   - A provider that appears **only** in a shipped built-in size-alias pool is
     referenced, and therefore background-eligible when its CLI is ready and collection
     is enabled. This is the direct Muse regression; assert it without depending on the
     live `model_alias_defaults.yml` contents by patching `implicit_alias_targets`.
   - A user override of a built-in size alias replaces the shipped target for that
     alias: a provider only in the shipped default for an overridden alias is no longer
     referenced, and a provider only in the override is.
   - A parenthesized last-resort target `(A | B) || C` contributes all three providers.

   Follow the existing monkeypatch style in that file (patch
   `sase.llm_provider.usage.refresh._referenced_provider_ids` only where eligibility
   itself is under test; here patch the alias getters instead so the scan is exercised).

3. **Verify the live eligible set changes.**

   Confirm `eligible_usage_providers()` now returns Muse on this host:

   ```bash
   sase update -y
   python -c "from sase.llm_provider.usage.refresh import eligible_usage_providers; print(eligible_usage_providers())"
   ```

   Expect `muse` to be present alongside `claude`, `codex`, and `grok`.

4. **Run verification.**

   Run `just fix` inline first, then `sase tool run check`. Hand it to `/sase_monitor`
   with the `TESTING` / `TESTED` status pair if it runs long. Do **not** run
   `just check-full`; nothing here names it.

   This change does not alter TUI rendering code, so PNG visual goldens are not expected
   to move. If `just check` reports otherwise, treat a golden change as a signal to
   re-examine step 1 rather than accepting it.

5. **Visually verify the Muse indicator in the real TUI.**

   Use `/sase_monitor` to run:

   ```bash
   sase update -y && sase screenshot -o /tmp/muse_usage_indicator.png
   ```

   `sase update -y` is required so the TUI the screenshot launches runs the fixed code
   rather than the previously installed build.

   In the monitor follow-up, read the printed `png=` key and **open and look at the
   PNG**. Confirm in the header's top-right usage indicator that:
   - a Muse window badge is present,
   - it is the **weekly** window (the 5-hour session window must stay hidden, per the
     `session: never` default), and
   - the existing Claude, Codex, and Grok badges are still rendered and not displaced or
     truncated by the new badge.

   Inspect the image itself; do not infer success from the file existing or from the
   command's exit status. If the header is width-constrained and the Muse badge is
   pushed into the overflow count, say so explicitly rather than reporting a clean pass.

   Note that Muse's stored windows currently read `freshness=unknown` at ~29m old. Once
   Muse is background-eligible the 300s cadence will re-probe it; if the badge renders
   as stale immediately after the fix, allow one refresh cadence (or trigger a refresh)
   before judging the result.

## Risks

- **Widening "referenced" widens background probing.** Any provider in a shipped
  size-alias pool with `probe: True` and a ready CLI becomes background-eligible. Today
  that adds `muse` and potentially `agy` (which appears in the `@xsmall` pool) on hosts
  where those CLIs are installed. This is the intended, correct behavior — those are
  providers SASE launches agents on — but it means one more probe per refresh cadence on
  such hosts. The Muse probe is free (echo-mint, no model call, no tokens), which is
  what makes this safe to switch on by default. Confirm in step 3 exactly which
  providers the eligible set gains and report it.
- **Users who do not want a newly eligible provider probed** can still turn it off with
  `llm_provider.usage_metrics.providers.<name>: false`, which is checked after the
  reference test and is unaffected by this change.
- The tokenizer tweak for parenthesized targets is a small behavior change to an
  existing loop. The added test pins it.

## Out of scope

- Changing which providers are in the shipped size-alias pools, including adding Muse to
  `@large` / `@xlarge`. The pools are correct; the scan was wrong.
- Any change to the Rust normalizer, the indicator projection, the echo-mint probe, or
  the `indicator.providers.muse.windows.session: never` default. All four are verified
  working above.
- The known Muse authoritative-empty blip recorded on bead sase-14c (a missed mint can
  blank the weekly indicator for up to one refresh cadence). That was a decision
  accepted inside the epic, not open work.
- The indicator.rs provider allowlist refactor already tracked by bead sase-14i.
