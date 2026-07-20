---
tier: tale
title: Keep repeated template clan declarations on their newly allocated generation
goal: 'Repeated #research_swarm and other template-clan declarations resolve the clan
  from the newly allocated member token, while template joiners retain their existing-clan
  lookup semantics.

  '
create_time: 2026-07-20 17:28:20
status: done
prompt: 202607/prompts/fix_template_clan_declarations.md
---

# Plan: Fix repeated template clan declarations

## Context and root cause

The Telegram launch reached SASE normally; `sase-telegram` only surfaced a `DirectiveError` raised by SASE's
multi-prompt clan prepass. The log records a newly planned agent named `research.j.cdx`, but the prepass attached it to
the older clan `research.i`, then correctly rejected the resulting hood mismatch.

The mismatch comes from resolving every templated clan target such as `research.@` against the reserved clan registry
before considering whether the prompt declares a new clan. That ordering is valid for a joiner, which should target the
newest matching clan when one exists, but it violates `%clan`'s create-only contract. A declaration must derive its
concrete clan token from the member names allocated for the current launch. The same registry-first ordering exists in
the runner fallback resolver, so fixing only the multi-prompt/Telegram path would leave direct or re-executed
declarations with the same latent bug.

## Clan resolution semantics

Make declaration-versus-join intent explicit during clan target resolution:

- For a templated `%clan` declaration, match the current member's concrete name against its name template and render the
  clan with that same token. Do not reuse a historical clan merely because it matches the clan template.
- For a templated `%id(..., clan=...)` join, preserve the existing behavior: resolve the newest registered matching clan
  first, and derive an implicit new clan from the member token only when no matching clan exists.
- Keep concrete clan names, create-only collision errors, hood validation, generation pinning, reservation rollback, and
  join behavior unchanged.
- Centralize or clearly share the member-token derivation so the batch prepass and runner fallback cannot drift.
  Continue using the canonical Rust-backed agent-name template match/render helpers rather than introducing a parallel
  template grammar.

## Implementation

Update `src/sase/agent/clan_membership.py` so its internal target resolver uses the declaring member first when
`create_only` is true, while retaining registry-first lookup for joiners. Preserve typed `ClanMembershipError`
diagnostics when a template cannot be derived and preserve the atomic registry reservation step after resolution.

Update `src/sase/agent/multi_prompt_launch_plan.py` to determine whether each raw clan group contains a declaration
before resolving its templated clan name. Declaration groups must derive the token from one of the already planned
member names in that launch; join-only groups must keep newest-existing-clan resolution with the current
implicit-creation fallback. Perform the existing single-declaration and hood checks against the resulting concrete clan
before any member process is spawned.

No `sase-telegram` change is needed: its inbound handler called the public SASE launcher correctly and reported the
exception it received.

## Regression coverage

Extend the focused clan launch tests to recreate the production sequence:

1. Launch and persist one templated clan generation.
2. Launch the same create-only templated swarm again.
3. Assert that every new member and the shared clan membership payload use the next token, and that no spawn is bound to
   the historical clan.

Add runner-fallback coverage for a templated declaration when an older matching clan exists, proving that it derives and
reserves the new concrete clan. Add or retain focused assertions showing that a template joiner still resolves an
existing clan and that a missing template join target can still be created implicitly. Existing concrete declaration
collision and outside-hood tests must continue to pass.

## Validation

Run `just install` before repository checks, as required for an ephemeral SASE workspace. Run the targeted clan launch
and metadata test modules first for fast feedback, then run `just check` for the mandatory full lint, type, and test
validation. If any failure exposes a declaration/join semantic regression, adjust the implementation and repeat both
targeted and full validation until clean.
