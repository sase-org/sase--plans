---
tier: tale
title: Harden fork parent resolution
goal: Make fork launches coalesce shared transcript sources and wait for incomplete
  named parents before resolving fork history or claiming a real workspace.
create_time: 2026-07-20 16:48:01
status: done
prompt: 202607/prompts/harden_fork_parent_resolution.md
---

# Plan: Harden fork parent resolution

## Context and approach

Explicit `#fork` parents already become dependency metadata, but launch-time xprompt expansion executes the fork
resolver before the runner reaches that dependency barrier. Preserve the existing implicit-wait and deferred-workspace
semantics while deferring only the side-effectful fork expansion during launch preview, fan-out planning, and directive
extraction. After dependency admission succeeds, expand the retained fork workflow before runner-slot admission and real
workspace preparation so complete parents still inject history normally and incomplete clans or agents park without a
late workflow failure.

Keep the shared xprompt processor behavior explicit and reusable rather than teaching each launch surface a different
fork workaround. Static launch analysis must still expand unrelated xprompts so model, fan-out, clan, and VCS metadata
continue to work, and raw prompt/history artifacts must retain the user-authored fork reference.

## Parent-source resolution

Adjust fork-source validation to distinguish duplicate user requests from convergent resolution. Repeated textual parent
arguments remain invalid and produce a clear atomic diagnostic. Distinct parent names or grouped members that resolve to
the same canonical transcript are coalesced in stable first-parent order, including grouped family/clan sources, so
history is injected once without losing unique later transcripts or source provenance.

## Verification and completion

Add focused regression coverage for same-transcript parents, repeated textual parents, and grouped-source overlap. Cover
launch ordering with an incomplete clan or named agent to prove the fork resolver is not invoked before the dependency
barrier or real workspace claim, then prove a completed clan proceeds through normal fork expansion and history loading.
Re-run the affected resolver, xprompt, directive, launch-planning, and runner tests, then run `just install` followed by
the repository-required `just check`.

After all checks pass, update only bead `sase-8g.8` with concise implementation notes and status `closed`; leave parent
epic `sase-8g` open and create no additional beads.
