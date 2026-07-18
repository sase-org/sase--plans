---
tier: tale
title: Make tribe membership a clan directive property
goal: 'Clan launches express optional tribe membership through %clan(..., tribe=...),
  reject conflicting standalone %tribe declarations, resolve one deterministic tribe
  per clan, and migrate every maintained prompt source to the new contract.

  '
create_time: 2026-07-18 14:48:46
status: done
prompt: 202607/prompts/clan_tribe_directive_contract.md
---

# Plan: Make tribe membership a clan directive property

## Outcome and user-facing contract

Change tribe assignment from an independent property of a clan member to an optional property of the clan declaration:

- Preserve `%clan:<name>` / `%c:<name>` for joining a clan without assigning its tribe.
- Accept `%clan(<name>, tribe=<tribe-name>)` and the equivalent `%c(...)` alias. The first positional argument remains
  the clan name or clan-name template; `tribe=` is the only supported clan keyword.
- Preserve `%tribe:<name>` / `%t:<name>` for standalone agents and sequential families that are not members of a clan.
- Reject any effective prompt containing both canonical directives, regardless of aliases, order, whitespace, or
  colon/parenthesized spelling. The error should identify the conflict and show the replacement form, for example:
  `Cannot combine %tribe with %clan; use %clan(<clan>, tribe=<tribe>) to assign the clan to a tribe.`
- Continue to ignore directive-like text in fenced blocks and disabled xprompt regions.
- Validate `tribe=` with the same tribe-name rules used by `%tribe`, and produce focused errors for a missing clan name,
  an empty tribe value, duplicate/unknown keywords, excess positional arguments, and malformed parenthesized syntax.

Keep the parsed values distinct: `PromptDirectives.tag` remains the standalone agent/family tribe, while a new
`PromptDirectives.clan_tribe` carries only the tribe declared through `%clan`. This avoids silently recreating the
invalid two-axis model internally.

## Directive parsing, inspection, and rewrite surfaces

Update the shared directive collector/value resolver so the normal extractor and the static multi-prompt clan prepass
consume the same validated clan argument structure. The prepass must fail the entire batch before workspace allocation
or spawning when it sees `%clan` plus `%tribe`, while still grouping segments by the clan's positional name even when
different segments provide different `tribe=` values.

Retain the existing `%clan` versus `%n(parent, suffix)` exclusion. In addition, close the semantic family-attachment
bypass at launch time: if `%tribe` is used while a `%n(parent, suffix)` attachment inherits a clan from its parent,
reject it with an error explaining that tribe membership belongs to the inherited clan and must be supplied by a clan
member's `%clan(..., tribe=...)` declaration.

Update prompt-editing helpers so they cannot synthesize `%tribe` beside `%clan`. A rewrite that is explicitly changing
the tribe of a clan launch should add, replace, or remove the `tribe=` keyword while preserving the clan name and all
ignored literal regions; standalone tribe rewrites should retain their current canonical `%tribe:<name>` behavior. Any
Agents-tab directive-persistence path that targets a clan-bound row must either use the clan-aware rewrite or stop with
the same useful guidance instead of writing a prompt that the runtime will later reject.

Refresh directive inspection/completion metadata to advertise both `%clan:<name>` and `%clan(<name>, tribe=<tribe>)`,
including a `tribe=` keyword hint, without changing the `%tribe` completion available for standalone launches.

## Durable clan-tribe semantics and the core boundary

Represent a clan tribe declaration as optional launch metadata named `clan_tribe`; do not write it into the existing
per-agent `tag` field or `agent_tags.json`. The runner should:

- write `clan_tribe` only for `%clan(..., tribe=...)`;
- preserve it across runner re-exec/retry metadata rewrites;
- continue persisting a standalone `%tribe` through the existing `tag` and tag-store path; and
- retain clan membership and generation metadata for direct members and inherited sequential-family members.

Extend the `sase-core` agent-scan wire/scanner and Python mirror with the optional `clan_tribe` field, including live
scan, indexed-record, JSON, and Python-binding parity. Put the clan-level resolution rule in the Rust core rather than
the TUI: for one clan generation, select the declaration belonging to the latest launched member among members that
actually specified `clan_tribe`. A later member with no `tribe=` does not clear the clan assignment; a later member with
a different value replaces it. Use the stable launch timestamp/identity already carried by artifact records, with a
deterministic tie-breaker, so concurrent runner startup order cannot change the result.

Expose that resolver through the normal Python binding/facade and use it when projecting synthetic clan containers. The
container receives the resolved tribe as its single effective `tag` and clan badge, so its entire subtree stays in one
tribe panel. Real member rows no longer need duplicate per-agent tags for new clan launches. When no member in a
historical clan generation has `clan_tribe`, retain the current legacy member-tag aggregation as a display fallback so
existing artifacts remain readable; once a new-style declaration exists, it is authoritative.

## Maintained prompt and xprompt migration

Replace every maintained prompt that currently emits both directives with the single clan form, then search both
repositories again to catch aliases and inline variants:

- In SASE epic work rendering, emit `%clan(<epic-id>, tribe=epic)` for every phase and land segment. Update launch
  summaries/previews and their fixtures so they parse and display the tribe from the clan declaration rather than a
  separate `%tribe:epic` line.
- In the linked chezmoi repo, migrate every segment of `home/sase/xprompts/research_swarm.md` from `%t:research` plus
  `%clan:research.@` to `%clan(research.@, tribe=research)` while preserving template grouping and wait references.
- Update the in-repo `sase_run` generated-skill source with the new syntax, the standalone/clan distinction, and the
  conflict guidance. Regenerate provider-specific chezmoi skill copies through `sase skill init --force` rather than
  hand-editing generated `SKILL.md` files, and inspect the generated diff before applying/deploying it.
- Re-scan the package xprompts, project xprompts, generated prompt renderers, and chezmoi xprompts for prompts that
  still contain both directives. Historical archived plans and literal test documentation may remain only when they are
  deliberately protected or are explicit migration fixtures.

Do not edit protected `sase/memory/*.md`, `AGENTS.md`, or generated provider instruction shims without separate explicit
permission; those files are outside this implementation even if an exhaustive text search reports historical syntax.

## Test coverage

Add or update focused tests for:

- parser acceptance of long/short clan forms with `tribe=`, unchanged colon-only clan membership, and valid standalone
  `%tribe`;
- canonical conflict detection for every alias/order combination, including directives introduced by xprompt expansion,
  plus fenced/disabled-region exclusions and actionable error text;
- clan argument arity, supported-key, duplicate-key, empty-value, invalid-name, and template handling;
- static clan preflight and fanout behavior, proving a malformed/conflicting batch spawns nothing and same-clan segments
  with different tribe declarations retain one clan generation;
- runner metadata and re-exec preservation, standalone tag-store behavior, and rejection of `%tribe` on a family
  attachment that inherits clan membership;
- Rust live/index scan and Python wire parity for `clan_tribe`, and core resolver cases for absent declarations,
  repeated identical declarations, later overrides, later non-declaring members, separate generations, and stable
  tie-breaking;
- Agents-tab clan tree/panel projection, proving the latest declaration yields one container tribe and the whole clan
  subtree follows it, while legacy artifacts retain their old fallback rendering;
- clan-aware prompt rewrites and directive completion/inspection metadata; and
- exact epic-rendered prompts, CLI previews, and successful expansion of the migrated research swarm and regenerated
  `sase_run` examples.

## Validation and handoff

After implementation, run each repository's native validation in its own checkout:

1. In `sase-core`, format and test the Rust workspace, including scan parity and Python-binding tests.
2. In SASE, run `just install` first as required for an ephemeral workspace, then run focused directive, launch,
   metadata, core-facade, epic, and Agents-tab tests while iterating, followed by the mandatory `just check`.
3. In chezmoi, run the relevant xprompt expansion/smoke checks and `just check`; verify generated skill copies match the
   in-repo source and only intended chezmoi files changed.
4. Perform a final cross-repository search and smoke expansion of representative standalone, clan-without-tribe,
   clan-with-tribe, conflicting, epic, and research-swarm prompts.

The change is complete when no supported launch path can express both an independent member tribe and clan membership,
the newest explicit clan declaration deterministically controls the clan's one tribe, all maintained prompts use the new
form, legacy artifacts still load, and all three repository validation suites pass.
