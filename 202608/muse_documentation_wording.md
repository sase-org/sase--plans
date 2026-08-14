---
tier: tale
title: Clarify Muse and Grok documentation routing language
goal:
  Documentation distinguishes provider autodetection from builtin model-alias routing
  without changing either behavior.
size: xsmall
proposed_by: bbugyi200.athena.01f.f2
create_time: 2026-08-14 13:05:57
status: wip
---

# Plan

Remove the overloaded `explicit-only` label from Muse Code documentation without
changing provider autodetection or any shipped model-alias pool. State the actual
selection behavior directly instead: Muse is never auto-detected, but users can choose
it through the documented provider/model mechanisms. In the same documentation pass,
correct the remaining README, installation, and xprompt claims that still describe Grok
Build as explicit-only even though Grok now participates in the shipped `@smart`,
`@cheap`, and `@cheaper` pools.

## Background

Muse and Grok deliberately publish no autodetection priority because their executable
names are too generic for PATH-presence detection. That autodetection fact is separate
from model-alias routing. Muse remains absent from every shipped builtin alias, while
`grok/grok-4.6` is now a member of `@smart`, `@cheap`, and `@cheaper`. The existing
`explicit-only` wording conflates those mechanisms: it is confusing for Muse and
factually stale for Grok in several documentation surfaces.

This is a documentation-only correction. Do not edit
`src/sase/llm_provider/model_alias_defaults.yml`, provider registration or autodetection
metadata, generated alias-table contents, or tests.

## Implementation

1. Update the primary provider-selection documentation:
   - In `docs/llms.md`, replace the Muse `explicit-only` and "only by explicit
     selection" language in both the general selection flow and Muse integration section
     with plain statements that Muse never participates in autodetection, followed by
     its existing provider config, `%model:muse/...`, and `SASE_MUSE_PATH` access paths.
   - In `docs/agent_providers.md`, make the same terminology correction in the Muse
     provider overview while preserving its install and authentication guidance.
   - In `docs/configuration.md`, distinguish direct provider/model selection from alias
     routing in the provider configuration table, and avoid the overloaded "explicit
     selection" phrasing in the Grok paragraph while retaining the shipped-pool note.

2. Correct the user-facing entry points and examples:
   - In `README.md`, replace the combined Muse/Grok `explicit-only` prerequisite note
     with an autodetection-specific explanation, retain usable direct launch examples,
     explain that Grok can also be reached through the shipped pools, and mark both
     provider rows simply as supported.
   - In `INSTALL.md`, replace the prerequisite table's "explicit provider/model
     selection required" qualifiers with accurate autodetection wording, including
     Grok's alias-pool path where useful.
   - In `docs/xprompt.md`, revise the Muse and Grok `%model` example comments: Muse is
     never auto-detected, while Grok is never auto-detected but is also available from
     shipped pools.

3. Remove the overloaded Muse label from the published onboarding prose in
   `docs/blog/posts/hello-sase-your-first-15-minutes.md` and
   `docs/blog/posts/structured-agentic-software-engineering.md`. Preserve the direct
   Muse launch examples and explain why they are needed in terms of Muse not being
   auto-detected. Update every nearby prose or configuration-comment occurrence,
   including "explicit provider/model selection" and "selecting Muse explicitly", so the
   articles use one consistent distinction.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run `just fmt-md` and inspect the resulting diff to ensure only the eight intended
   Markdown files changed, the examples still render correctly, and no generated model
   alias table or implementation file was modified.
3. Search `README.md`, `INSTALL.md`, and `docs/` for `explicit-only` and for Muse/Grok
   provider text that still says explicit provider/model selection is required. Confirm
   the confusing label is gone, Muse is consistently described as never auto-detected,
   and Grok's shipped-pool availability is not contradicted.
4. Recheck `src/sase/llm_provider/model_alias_defaults.yml` read-only: Muse must remain
   absent, and Grok must remain in exactly `@smart`, `@cheap`, and `@cheaper` at the
   existing effort levels.
5. Run `git diff --check`, then run the required `just check` repository gate. No new
   tests are needed because behavior and configuration are intentionally unchanged.
