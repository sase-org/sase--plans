---
tier: epic
title: Threshold-aware big epic lander routing
goal: 'Epic land agents automatically use the configurable @big_epic_lander role alias
  for plans at or above a phase-count threshold, while smaller epics and explicit
  land models retain their existing behavior.

  '
phases:
- id: core_phase_count
  title: Stable authored phase count contract
  depends_on: []
  description: '''Stable authored phase count contract'' section: expose the total
    authored phase count through the Rust epic work-plan wire without changing scheduling
    semantics.'
- id: alias_config
  title: Big-epic alias and threshold configuration
  depends_on: []
  description: '''Big-epic alias and threshold configuration'' section: add the implicit
    role alias, its fallback chain, the default threshold, schema coverage, and user-facing
    alias surfaces.'
- id: lander_routing
  title: Threshold-aware lander routing and integration coverage
  depends_on:
  - core_phase_count
  - alias_config
  description: '''Threshold-aware lander routing and integration coverage'' section:
    route implicit epic landers from stable phase counts, preserve explicit-model
    precedence, align previews and docs, and verify the end-to-end contract.'
create_time: 2026-07-18 06:08:27
status: wip
bead_id: sase-6q
---

# Plan: Threshold-aware big epic lander routing

## Context and design decisions

Approved epic plan files become an epic bead plus one phase bead per authored phase, and the Rust core builds the
deterministic work plan consumed by the Python multi-prompt renderer. Today an epic without a top-level `model` always
emits `%model:@epic_lander` for its land agent. Introduce `@big_epic_lander` as another built-in role alias and select
it only when the epic's total authored phase count is greater than or equal to the configured threshold.

The configuration contract will be `bead.big_epic_phase_threshold`, an integer with a shipped default of `5` and a
minimum of `1`. Runtime access should defensively fall back to `5` for missing or malformed values, including booleans.
The alias itself will implicitly fall back to `@epic_lander`, not directly to `@default`. Consequently, users who do
nothing see the same concrete model as before, an existing `epic_lander` override continues to affect both sizes, and
users can specialize only large epics by overriding `model_aliases.builtin.big_epic_lander`.

Selection follows this precedence table:

| Epic land model            |                       Total authored phases | Emitted land model |
| -------------------------- | ------------------------------------------: | ------------------ |
| Explicit top-level `model` |                                         Any | The explicit model |
| Omitted                    |       Below `bead.big_epic_phase_threshold` | `@epic_lander`     |
| Omitted                    | At or above `bead.big_epic_phase_threshold` | `@big_epic_lander` |

Count every phase authored by the epic plan, including phases already closed when an interrupted epic is resumed. Do not
base the decision on wave count or only the currently open phase assignments: those values can shrink during the epic
and would make the lander role unstable. This feature does not alter epic plan frontmatter, per-phase models, dependency
scheduling, or land-agent wait dependencies.

## Stable authored phase count contract

Extend the shared Rust epic work-plan wire with a total phase count derived from all phase children of the epic, while
keeping the existing waves limited to non-closed work. Cover the distinction in core tests so a plan containing closed
and open phases still reports its full authored size. The Python `EpicWorkPlan` adapter should carry that field into
launch rendering and use a safe compatibility fallback for an older binding payload that lacks the new field; no manual
crate-version edits are part of this work.

This keeps phase counting in the shared backend domain and leaves the SASE-specific merged configuration and `%model`
directive choice in the thin Python launch adapter.

## Big-epic alias and threshold configuration

Add `big_epic_lander` to the centralized built-in role-alias policy with the implicit fallback `@epic_lander` and a
description that explains its threshold-selected lander role. Ensure it behaves like every other built-in alias across
resolution, alias-chain and cycle handling, launch-scoped and temporary overrides, doctor validation, model completion,
and ACE Models-panel ordering. Update focused tests for alias discovery, fallback inheritance, explicit shadowing,
completion payloads, and display metadata rather than adding special-case resolution paths.

Add `bead.big_epic_phase_threshold: 5` to the bundled defaults and JSON schema, plus a focused accessor shared by
preview and launch code. Document the field and the new built-in alias in the configuration and model-routing
references, including the reason for the two-step `@big_epic_lander` → `@epic_lander` → `@default` fallback.

## Threshold-aware lander routing and integration coverage

Centralize the implicit land-alias choice so the validated plan preview and the actual multi-prompt renderer cannot
drift. The preview can use `len(plan.phases)` directly; the launch renderer must use the stable total phase count
carried by `EpicWorkPlan`. In both paths, an explicit top-level land model remains authoritative and bypasses threshold
routing.

Add boundary tests immediately below, exactly at, and above the default and a custom threshold. Include regression
coverage for closed phases on resume, explicit epic models on large plans, configured `big_epic_lander` targets, and
unchanged small-epic rendering. Update CLI rendering expectations and the SDD and bead workflow documentation so
previews, emitted directives, and published behavior all describe the same selection rule.

Verification should include targeted Rust core tests, focused Python tests for configuration/alias resolution/work
rendering/plan-file previews, and then the repository-required full sequence `just install` followed by `just check`
with the linked Rust core build in use. Inspect any generated snapshot or golden changes and accept only changes caused
by the additional built-in alias or the new lander directive at the configured boundary.

## Compatibility and risks

- Default behavior is model-compatible because `@big_epic_lander` inherits `@epic_lander`; only users who configure the
  new alias get a different large epic lander model.
- Counting all authored phases makes routing deterministic across initial launch and resume, while a defensive Python
  fallback keeps mixed-version core/Python environments functional during rollout.
- Alias catalogs and the Models panel have explicit ordering lists, so they must be updated alongside the resolver to
  avoid a built-in alias that works at launch but is missing or oddly placed in user-facing controls.
- The threshold is global merged SASE configuration under `bead`; project-local config remains able to override it
  through the existing layer merge rules.
