---
tier: tale
title: Preserve chop clan tribes across bounded agent views
goal: 'Active toobig-@ clan generations remain members of the @chop tribe even when
  their completed or dismissed declaring agent is outside a bounded agent view.

  '
create_time: 2026-07-19 21:32:43
status: done
prompt: 202607/prompts/preserve_chop_clan_tribe.md
---

# Plan: Preserve chop clan tribes across bounded agent views

## Outcome and diagnosis

Keep the existing clan-scoped chop launch contract and make its clan-level attributes survive partial agent views. The
`toobig_split` integration is already emitting `clan: "toobig-@"`; SASE already allocates one concrete generation,
scaffolds the first accepted proposal with `%clan(<concrete-clan>, tribe=chop)`, and scaffolds later proposals as clan
joiners. Live artifacts confirm that the declaring member persists `clan_tribe: "chop"` and every member persists the
same `agent_clan` and `agent_clan_generation`.

The failure occurs after visibility filtering. The Agents tab's Tier-1 artifact-index query returns active members plus
a capped recent-completion window and excludes dismissed rows. A serial toobig clan can therefore retain many waiting
joiners while its completed/dismissed declarer is absent. `project_clan_tree` only sees the returned member rows, finds
no explicit `clan_tribe`, and incorrectly projects the synthetic `toobig-N` container with no tribe. The same partial-
view hazard applies to the other declaration-backed clan attribute, `clan_summary`, and to presentation-neutral agent
lists that currently expose only per-agent tribe metadata.

Fix the information boundary rather than changing chop prompts, duplicating standalone per-agent tribe assignments, or
inferring `chop` from a clan name. The visible record set must remain bounded, while a separate, compact context payload
retains the authoritative resolved attributes for every clan generation represented by that record set.

## Core snapshot and index contract

Extend the `sase-core` agent-artifact snapshot/query wire with generation-scoped clan context that is distinct from
visible `records`. Each context item should identify the clan name and generation and carry the resolved tribe and
summary, including stable source identity/timestamp where useful for diagnostics and parity. Reuse the existing Rust
`resolve_clan_tribe` and `resolve_clan_summary` precedence rules: only explicit declarations participate, the latest
declaration wins, later non-declaring members do not clear a value, and generations never bleed into each other.

For artifact-index queries, first select visible active/recent/full-history records exactly as today. Derive the clan
generation keys represented by those records, then resolve context for only those keys from all indexed members of the
same generations. Declaration sources must remain eligible even when they are terminal, dismissed/hidden, or outside the
recent-completion limit; they are semantic context, not rows to render. Keep those source rows out of `records` so
recent-history limits, dismissal behavior, member counts, ordering, and startup payload size do not change. Add or
adjust denormalized index columns/indexes if needed to make the lookup key- and declaration-scoped instead of scanning
the archive, and bump/rebuild the index schema through the existing lifecycle mechanism.

Populate the same context shape for source scans before their result is handed to Python. Full-history scans can resolve
from their complete record set. Bounded missing/stale-index fallback must remain startup-safe: do not add O(archive)
JSON reads to first paint. It may return context available from its bounded records while the existing background index
repair/Tier-2 reconciliation restores complete context; make that degraded state explicit in tests and preserve the
current coalesced follow-up refresh rather than introducing a new refresh path.

Carry the additive wire through Rust serialization, the PyO3 binding, Python dataclasses/converters, and index schema
capability checks. Keep the core contract frontend-neutral so CLI, TUI, mobile/editor integrations, and future clients
can consume the same clan result without reimplementing hidden-row searches.

## SASE projections and refresh behavior

Teach the agent-loading normalization layer to apply snapshot clan context to loaded members and synthetic containers by
exact `(agent_clan, agent_clan_generation)` key. Treat this as an in-memory effective projection, not a rewrite of
member `agent_meta.json`, `raw_xprompt.md`, or `agent_tribes.json`. Preserve the distinction between a clan-level tribe
and a standalone agent tribe, and retain the legacy per-member tribe aggregation only when neither visible declarations
nor snapshot context supplies a new-style clan declaration.

Thread the context through Tier-1, Tier-2, exact-delta, and optimistic refilter/reprojection paths so a later refresh,
kill, or dismissal cannot drop a previously resolved clan tribe. Context-only declaration sources must never become
rows, affect clan runtime/member aggregation, revive dismissed agents, or participate as actionable selection targets.
Use the existing cached/background agent reload pipeline; perform no filesystem or index reads in render, keypress,
selection, or Textual pump callbacks.

Update the presentation-neutral agent-list projection used by `sase agent list --json` and integrations to report the
same effective clan tribe for a clan member when context is available, while keeping standalone assignments unchanged.
Prefer one shared projection helper over separate CLI/TUI precedence logic. Do not guess a tribe when all declaration
evidence has truly been deleted or is unavailable.

The `toobig_split` result schema, bugyi-chops implementation, and Athena `agent_clan: {name_prefix: toobig-}` inhibit
configuration need no changes: they already produce the correct clan generation and guard active members. Keep chop
preview tests proving exactly one declarer uses `%clan(..., tribe=chop)` and later proposals use `clan=` joins.

## Regression coverage

Add a real Rust artifact-index regression with a `toobig-0` declarer carrying `clan_tribe: "chop"`, active/waiting
joiners from the same generation, enough newer completed records to exceed the recent cap, and dismissal of the
declarer. Assert that visible records and their limits are unchanged, the declarer is not returned as a row, and the
snapshot still resolves the generation to `chop`. Cover latest-declaration overrides, generation isolation, absent
declarations, clan summaries, project filters, and index rebuild/serialization parity.

Add Python wire/facade parity tests and a Tier-1 Agents-tab regression that builds the clan from joiners only plus the
snapshot context. Assert the synthetic container appears in the `@chop` panel, every visible member stays under that
container, member/status counts exclude the declaration context, and a reload/refilter/dismiss cycle retains the same
grouping. Also cover the bounded source-fallback degradation followed by Tier-2/index reconciliation, and verify the
context path performs no render-time disk access.

Extend presentation-neutral list/JSON coverage so active `toobig-0.*` members expose effective tribe `chop` even when
the declaring row is omitted, without changing standalone tribe output. Retain existing chop launch, deduped-head,
partial-launch, wait-chain, and active-clan inhibit tests to prove the fix does not alter proposal ordering, clan
allocation, or policy decisions.

## Validation and handoff

In `sase-core`, run `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, focused clan
resolution/index tests, and `cargo test --workspace`. In SASE, run `just install` before tests, then focused wire,
artifact-index lifecycle, agent-loader/tree/panel, agent-list, and chop clan suites while iterating, followed by the
mandatory `just check`. Re-run the live-shape model probe (or an isolated equivalent) with an omitted declarer and
active `toobig-0` joiners to confirm the container resolves to `@chop` without adding the declarer to the visible
roster.

Finish with a cross-repository diff/status review. The change is complete when clan attributes remain authoritative
under bounded and dismissed-row views, no new work reaches the TUI event loop, the visible-history contracts remain
unchanged, and both repositories' prescribed checks pass.
