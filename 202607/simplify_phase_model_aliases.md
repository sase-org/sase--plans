---
tier: tale
title: Simplify phase model aliases and split cheap model pools
goal: 'Phase agents use simpler size defaults, cheaper and cheapest expose distinct
  load-balanced pools, explicit per-phase models work at every size, and the planning
  skill explains outcomes without exposing SASE''s alias internals.

  '
create_time: 2026-07-21 10:06:09
status: done
prompt: '[202607/prompts/simplify_phase_model_aliases.md](prompts/simplify_phase_model_aliases.md)'
---

# Plan: Simplify phase model aliases and split cheap model pools

## Context and outcome

The current phase-routing chain sends small phases through `@cheapest`, medium phases through the shared `@phase_worker`
alias, and large phases through `@smartest`. Simplify that policy by removing `@phase_worker` as an implicit builtin
alias, routing `@medium_phase_worker` directly through `@default`, and renaming the old cheap pool role to `@cheaper`.
Reuse `@cheapest` for a new, independent pool without making it the fallback of any other implicit alias.

The resulting builtin behavior is:

| Alias                  | Implicit value or fallback                   | Automatic use                                         |
| ---------------------- | -------------------------------------------- | ----------------------------------------------------- |
| `@small_phase_worker`  | `@cheaper`                                   | Small phase agents without an explicit phase model    |
| `@medium_phase_worker` | `@default`                                   | Medium phase agents without an explicit phase model   |
| `@large_phase_worker`  | `@smartest`                                  | Large phase agents without an explicit phase model    |
| `@cheaper`             | `claude/opus@medium \| codex/gpt-5.5`        | The small-phase fallback                              |
| `@cheapest`            | `claude/sonnet \| codex/gpt-5.3-codex-spark` | No automatic consumer yet; available for explicit use |

`@phase_worker` will no longer be automatically defined, completed, selected, or treated as a builtin override key. The
existing `phase_worker` Models-panel bucket is a display grouping rather than an alias, so retain that bucket name and
place only the three size-specific phase aliases in it. As with any arbitrary name, a user can deliberately define a
custom alias named `phase_worker`, but it has no special routing behavior.

This is a compatibility change. A former `llm_provider.model_aliases.builtin.phase_worker` value must not silently keep
controlling medium phases; diagnostics and documentation should direct users to `builtin.medium_phase_worker` when that
was their intent. Existing machine-global load-balancer entries need no destructive migration: `cheaper` uses a new
alias key, while the changed `cheapest` pool has a new membership fingerprint and therefore resets its cursor safely
under the existing state contract.

## Central alias policy and phase routing

Update the centralized model-alias policy in `sase.llm_provider.config` so the fixed role set, fallback graph,
descriptions, implicit pool targets, and public constants exactly express the table above. Remove the phase-worker alias
constant and fallback, add `cheaper` with the former `cheapest` pool, redefine `cheapest` with the requested
Sonnet/Spark pool, point the small alias at `cheaper`, and point the medium alias at `default`. Preserve large routing
through `smartest`.

Propagate that source of truth through alias resolution, model completion, configuration/schema descriptions, directive
validation, picker cycle guards, doctor guidance, and public exports. A launch-scoped or temporary `phase_worker`
override should cease to have builtin meaning; if no custom alias of that name is configured, normal
unknown/retired-alias diagnostics should apply. Add a focused migration diagnostic for stale
`model_aliases.builtin.phase_worker` entries so users are told to move the target to `medium_phase_worker` (or remove it
to accept `@default`) rather than being given generic custom-alias advice. Also remove `@phase_worker` from older
retired-`@worker` recommendations.

Keep the load-balancing engine generic. Cover the new policy at its integration boundary: small-phase launches and
explicit `@cheaper` launches consume one shared `cheaper` cursor; explicit `@cheapest` launches consume a separate
cursor and use the new pool; unavailable provider CLIs continue to be skipped for both pools; peeks do not consume; and
the old `cheapest` fingerprint cannot carry its cursor into the new membership.

## Models panel and configuration surfaces

Adjust the Models-panel role order and builtin bucket specification so the `phase_worker` bucket remains always present
with exactly `small_phase_worker`, `medium_phase_worker`, and `large_phase_worker`, followed by any custom members
assigned to that display bucket. Update the bucket description so it no longer claims to contain a shared fallback.
Surface both `cheaper` and `cheapest` as independently selectable/editable implicit pool aliases, with distinct
descriptions, resolved members, overrides, and state. Preserve custom bucket metadata under the existing `phase_worker`
bucket key.

Update panel/picker unit tests and PNG goldens for the three-member bucket, changed medium fallback, the additional
top-level alias row, the two pool values, navigation/member counts, override behavior, and builtin-shadow warnings. The
implementation agent must consult the required `tui_perf` memory before changing this presentation path and keep alias
aggregation as a cheap, side-effect-free peek rather than advancing pool rotation.

Refresh `default_config.yml`, the JSON schema, doctor checks, completion catalogs, examples, and all model/phase
documentation. Explain the breaking migration clearly: small phases now inherit `cheaper`; `cheapest` names a new
standalone pool; medium phases inherit `default`; and `phase_worker` is no longer a builtin alias even though the
display bucket retains that name. Remove examples that teach users to configure or reference `@phase_worker`.

## Explicit models for every phase size

Treat phase size and an authored `phases[].model` as orthogonal inputs. The current plan wire type and bead launch
helper already carry a model independent of size and check an explicit value before selecting a size default; preserve
that contract rather than adding a new schema restriction or duplicating model selection. If implementation tracing
reveals any size-specific path that overwrites or rejects an explicit model, route it through the same centralized
precedence helper.

Add an end-to-end size matrix covering `small`, `medium`, and `large` phases: strict plan validation accepts `model` on
each phase, epic materialization stores each value on the corresponding bead, dry-run/launch rendering emits the
explicit `%model` value instead of a size alias, and relaunch uses the stored explicit model. Include accepted concrete
provider/model syntax and an explicit configured/builtin alias value. Keep planning behavior independent: small phases
still launch directly, while medium and large phases still append `#plan` even when their model is explicit.

## Simplify the generated `sase_plan` skill

Edit only the canonical source at `src/sase/xprompts/skills/sase_plan.md`. Remove the alias names, fallback chains, pool
details, and alias-override advice from the phase-size instructions. Replace them with user-level facts:

- small phase agents implement directly and do not create plans;
- medium and large phase agents create plans before implementation;
- by default, phase size also selects the model capability appropriate for the work, unless that phase has an explicit
  `model` override.

Retain the frontmatter syntax showing that `phases[].model` is optional, and state without alias terminology that an
explicit phase model is allowed for every size and takes precedence over the size-derived default. Keep the top-level
tale/epic model guidance if it remains relevant, but do not expose SASE's internal model-routing names or graph.

Update source-discovery tests to assert the three behavioral facts and explicit override rule, and add negative
assertions that the generated planning skill does not mention `phase_worker`, `cheaper`, `cheapest`, `smartest`, or
size-specific model alias names. Follow the generated-skills workflow after the source edit: use this workspace's SASE
executable to run `sase skill init --force`, deploy with `chezmoi apply`, and audit all provider copies. Access the
linked chezmoi repository through `/sase_repo`; never edit generated provider `SKILL.md` files by hand.

## Verification and acceptance criteria

Update focused resolver, load-balancing, completion, doctor, Models-panel, plan-validation, bead-materialization,
prompt-rendering, generated-skill, and documentation tests. Refresh only intentional Models-panel PNG snapshots and
inspect the bucket and pool rows visually before accepting them.

Before final handoff, run `just install` as required for the ephemeral workspace, then `just check`. Also run the
focused exact visual suite and the generated-skill consistency audit after regeneration. The implementation is complete
when:

- no implicit or automatically used `@phase_worker` remains, while the `phase_worker` display bucket still groups the
  three size aliases;
- small, medium, and large defaults resolve through `@cheaper`, `@default`, and `@smartest`, respectively;
- `@cheaper` and `@cheapest` own the exact requested, independent pools and retain availability-aware round-robin
  behavior;
- explicit phase models survive validation and launch for every phase size without changing which sizes plan first; and
- every generated `sase_plan` copy describes only phase behavior and contains none of the internal alias vocabulary.
