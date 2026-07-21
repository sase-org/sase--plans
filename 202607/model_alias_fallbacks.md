---
tier: tale
title: Ordered model alias fallbacks
goal: 'Model aliases support deterministic provider-aware fallback chains with ||,
  and the implicit @smartest alias prefers Claude Fable 5 while falling back to Codex
  GPT-5.6 SOL when the Claude CLI is unavailable.

  '
create_time: 2026-07-21 10:38:17
status: wip
prompt: '[202607/prompts/model_alias_fallbacks.md](prompts/model_alias_fallbacks.md)'
---

# Plan: Ordered model alias fallbacks

## Context and outcome

SASE model alias values currently support a single target or a `|`-separated, availability-filtered round-robin pool.
The pool parser deliberately rejects `||` today as an empty pool member. Add `||` as a distinct ordered-fallback
operator: `A || B` resolves to `A` whenever A's registered agent CLI provider is installed and otherwise resolves to B.
Unlike `|`, this choice is stable and never reads or advances the load-balancing cursor. Make the operator available to
both builtin overrides and custom aliases anywhere the existing alias-value grammar is accepted.

Use the new syntax for an implicit builtin value:

```text
@smartest = claude/claude-fable-5 || codex/gpt-5.6-sol
```

`@large_phase_worker` must continue to reference `@smartest`, so large phases inherit this provider-aware choice
automatically. A configured or temporary override of either alias must retain its existing precedence. Remove the now
redundant `smartest: claude-fable-5` builtin override from the tracked chezmoi `sase.yml` source and apply that source
through the normal chezmoi workflow; leave other config entries unchanged.

## Fallback grammar and semantics

Treat `|` and `||` as two explicit selector modes over the same member grammar:

- `A | B` remains an availability-filtered, persistent round-robin pool. A real launch consumes its next position;
  display and preview callers only peek.
- `A || B` is an ordered fallback chain. Every resolution chooses the first member whose resolved provider is registered
  and whose provider CLI is installed. It never creates or changes `llm_lb.json` state, even when called by an
  authoritative launch lane.
- Support two or more fallback candidates, so `A || B || C` naturally tries providers from left to right. Trim
  surrounding whitespace, preserve member order, and reject leading, trailing, or empty members.
- Do not permit `|` and `||` in the same alias value. Also reject a selector member that reaches another load-balanced
  or fallback-valued alias. Ordinary alias chains inside a member remain valid when they resolve to one concrete target.
  These rules keep selection ownership, cursor behavior, cycle checks, and effort inheritance unambiguous.
- Candidate availability is based only on the existing cached provider-CLI probe. Do not treat a later model/runtime
  error as a reason to relaunch on the next fallback. If every candidate is unavailable, select the first candidate so
  the existing provider lookup/launch diagnostics report the failure rather than silently changing the alias or treating
  it as a model on the default provider.

Preserve the current alias-resolution depth/cycle protection, launch-scoped override precedence, machine-wide temporary
override behavior, provider/model path handling, and trailing `@effort` semantics. Generalize the resolver's selector
provenance so an unavailable explicit provider prefix selected from either kind of expression reaches normal provider
diagnostics. A temporary override on a fallback-owning alias should suspend its fallback expression just as an override
currently suspends a load-balanced pool.

## Resolver, validation, and builtin policy

Refactor the selection parsing used by `config.py` and `load_balancing.py` (or a more appropriately named shared helper)
so callers receive an explicit mode and normalized members instead of inferring semantics from raw pipe characters. Keep
cursor persistence isolated to load-balanced mode; ordered fallback needs no new state file or migration. Use the same
parsed representation for runtime resolution, member inspection, editor validation, and doctor diagnostics so those
surfaces cannot disagree about mixed operators, empty members, nested selectors, cycles, unknown alias references, or
resolution-depth failures.

Move `smartest` from the generic implicit `@default` fallback map to the map of implicit concrete/selector values,
backed by a named default constant containing the exact string `claude/claude-fable-5 || codex/gpt-5.6-sol`. It remains
a builtin role alias with the same name, completion behavior, override path, and large-phase consumer; only its
unconfigured value changes. The Models panel and other introspection APIs should expose the raw implicit fallback
expression as they already do for implicit load-balanced aliases.

Update `sase doctor` to validate both selector modes. Malformed, mixed, cyclic, unknown, or nested selector expressions
are actionable warnings. An unavailable candidate alone remains informational: describe which candidate will be skipped
and which ordered fallback currently wins, without mutating rotation state. Update schema descriptions and the default
configuration example to document the two operators and show the new implicit `@smartest` value.

## Models panel and configuration editing

Generalize alias member metadata in the display layer so a view can distinguish a round-robin pool from an ordered
fallback chain while reusing resolved target, effort, provider, validity, and availability details. For `@smartest`, the
row's effective provider/model badge should show Claude Fable 5 when Claude is available and Codex GPT-5.6 SOL
otherwise. Its description strip should label the expression as a fallback (not a pool), list candidates in priority
order with availability markers, and make the selected candidate evident. Existing pool rendering and peek behavior must
remain unchanged.

Teach the Models-panel custom value editor and its preview validation about both `|` and `||`, including concise
examples and actionable rejection messages for mixed/nested selectors. Update affected unit tests and only the visual
PNG goldens whose intentional text/layout changes are confirmed by inspection.

## Documentation and personal configuration

Update the model-alias reference, general configuration guide, and Models-panel documentation to cover ordered fallbacks
alongside round-robin pools. Document that fallback is based on provider CLI installation, not runtime/model success;
that the first candidate wins when installed; that all-unavailable resolution preserves the first candidate for
diagnostics; and that overrides bypass the selector. Update the implicit-alias tables and examples so `@smartest` shows
`claude/claude-fable-5 || codex/gpt-5.6-sol` rather than `@default`, while `@large_phase_worker -> @smartest` remains
unchanged.

Open the linked chezmoi repository through `sase repo open`, remove only `llm_provider.model_aliases.builtin.smartest`
from `home/dot_config/sase/sase.yml`, and run `chezmoi apply` through the supported workflow so the live configuration
stops shadowing the new builtin default. Audit both the linked checkout and live rendered config afterward. This feature
does not require editing or regenerating the canonical `sase_plan` skill because phase-size routing itself does not
change.

## Verification

Add focused tests that cover:

- parsing and normalization of two- and multi-member `||` chains, plus rejection of empty, mixed, and nested selector
  expressions;
- deterministic first-installed selection, fallback when the primary CLI is absent, and first-member preservation when
  every provider is unavailable;
- no cursor-state reads/writes or advancement for fallback resolution, including launch calls with `consume=True`, while
  `|` round-robin behavior remains unchanged;
- concrete targets, alias-referenced candidates, cycles/depth limits, explicit provider prefixes, and candidate-specific
  reasoning effort;
- launch-scoped and temporary override precedence, plus large phase routing through the new implicit `@smartest` value
  without changing resume or one-consume-per-agent behavior;
- doctor/schema/editor diagnostics, alias-view metadata, Models-panel rendering, completion/config-source
  classification, and the exact builtin defaults;
- provider-availability matrices for Claude-only, Codex-only, both installed, and neither installed.

Run `just install` before repository checks, the focused resolver/registry, launch, doctor, schema, Models-panel, and
phase-routing tests during iteration, then the exact visual snapshot suite and the required full `just check`. Inspect
any updated PNG goldens rather than accepting them solely from the test result. Finally verify the main checkout and
linked chezmoi checkout contain no unrelated changes, and confirm the applied live config no longer defines a `smartest`
override.
