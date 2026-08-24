---
tier: tale
title: Preserve chop project routing through typed admission
goal:
  Agents proposed by the toobig_split chop launch in the proposal's sase project instead
  of falling back to the home project.
size: medium
proposed_by: bbugyi200.athena.0cs
---

# Plan: Preserve chop project routing through typed admission

## Diagnosis

The configured `bugyi_chop_toobig_split` producer is not the source of the bad project
choice. Its durable result gives every proposal the target workspace `gh:sase-org/sase`,
and SASE's initial chop scaffolding converts that value into an explicit workspace tag.
The current live `toobig-*` agents nevertheless appear under project `home`, with prompt
snippets containing `#git:home`.

The workspace is lost after typed admission. `plan_chop_proposals()` builds prompts with
the correct tag, `_resolve_typed_batch_project()` resolves those prompts to the correct
selected project and source checkout, and `_unit_dispatch_metadata()` persists each
proposal's `workspace`. After a `%if` condition becomes eligible, however,
`make_axe_chop_agent_dispatcher()` calls `agent_unit_dispatch_prompt()` and sends only
that reconstructed identity/model/body prompt to `launch_agents_from_cwd()`. The
reconstruction deliberately removes admission-only directives but has no workspace
argument, so the launcher's normal bare-prompt rule inserts `#git:home`. This regression
was introduced when chop proposals were moved onto durable typed admission; the older
direct launch path sent the fully scaffolded prompt and retained its project tag.

The correct repair belongs in SASE's Axe-specific typed dispatcher. The external
`bugyi-chops` package already fulfills the producer contract, and the Rust planner
already retains the selected project needed by condition and proc admission. Widening
the Rust wire format is unnecessary for this chop-specific loss because the exact
validated workspace ref is already present in per-unit Axe metadata.

## Implementation

1. In `src/sase/axe/chop_typed_admission.py`, centralize reconstruction of the final Axe
   AgentUnit launch prompt so it combines `agent_unit_dispatch_prompt()` with the unit's
   durable `workspace` metadata before calling `launch_agents_from_cwd()`. Normalize the
   stored value to a single leading `#` and preserve the existing launch parser's alias
   canonicalization, so `gh:sase-org/sase` reaches the launched agent as the configured
   `#gh:sase` project tag. Use this final project-qualified prompt when building chop
   linkage and prompt-hash environment metadata as well.

2. Keep dispatch fail-closed: if a resumed or malformed Axe admission bundle has no
   usable workspace value, return a typed launch error without invoking the launcher. It
   must never silently turn missing project metadata into `#git:home`. Preserve the
   existing condition stripping, identity, clan declaration/promotion, model, finalizer,
   wait-runner, priority, and chop ownership behavior.

3. Extend the focused typed chop tests in `tests/test_axe_chop_proposal_launch.py` to
   assert that an eligible conditioned proposal reaches the mocked launcher with its
   explicit workspace tag and that missing workspace metadata does not launch. Extend
   `tests/test_axe_chop_proposal_launch_clan_dispatch.py` with a `toobig_split`-shaped
   `gh:sase` assertion covering the first surviving member promotion path, proving the
   project tag survives even when admission replaces the statically planned clan
   declarer.

## Verification

1. Run `just install` so this ephemeral checkout uses the current repository and Rust
   binding dependencies.
2. Run the focused typed proposal and clan-dispatch tests, including the new ordinary,
   fail-closed, and promoted-member routing cases.
3. Run `just check` for the repository-wide lint gates and diff-scoped tests.
4. Inspect the resulting dispatch-prompt assertions to confirm the launcher receives
   `#gh:sase` (or the proposal's corresponding explicit project tag), never an
   unqualified prompt that can normalize to `#git:home`.
