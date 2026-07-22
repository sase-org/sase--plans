---
tier: tale
title: Machine identity configuration and initialization
goal: SASE has an explicit, machine-local identity with selector-safe overlays and
  a fully wired interactive initialization workflow.
bead: sase-8k.1
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 10:59:24
status: wip
---

- **PROMPT:** [202607/prompts/machine_config_init.md](prompts/machine_config_init.md)

# Plan: Machine identity configuration and initialization

## Goal

Complete bead `sase-8k.1` by adding the required per-machine identity configuration that later machine-agent-hood work
can rely on. A machine identity must be explicitly selected through local SASE state, only the matching
`sase_<machine>.yml` overlay may participate in configuration, and users must be able to initialize that identity
through both `sase config init` and the shared `sase init`/doctor workflow. Existing installations without an identity
must continue to load and run with `machine_name` unset until initialization is performed.

## Implementation

1. Add the machine identity schema and state-path contract.
   - Add top-level `machine_name` to `src/sase/config/sase.schema.json` with the `^[a-z_]+$` contract and make it the
     schema's sole required top-level field, without putting a synthetic value in `src/sase/default_config.yml`.
   - Add a `~/.sase/machine_name` path helper and document this bounded top-level state file in
     `src/sase/core/paths.py`.
   - Export `get_machine_name() -> str | None` and `require_machine_name() -> str` from the configuration API. The
     required accessor will fail with an actionable instruction to run `sase config init`; the optional accessor will
     preserve today's behavior for callers that do not yet require a hood.

2. Make overlay loading selector-aware without changing ordinary overlays.
   - Read and validate the one-line selector from the machine-local state path.
   - Classify an overlay as machine-specific when its top-level YAML mapping contains `machine_name`; include it only
     when that value matches the local selector. Continue including every non-machine overlay in the same sorted order
     and with the same merge strategy as today.
   - Reuse the selected-overlay set across merged runtime configuration, config-layer/inventory discovery, and xprompt
     source discovery so foreign machine settings never leak through an alternate consumer.
   - Add the selector file's single stat token to config freshness computation while retaining raw overlay stat
     coverage, so selector changes invalidate the merged-config cache without adding render-path YAML work.

3. Implement the interactive, idempotent config initializer.
   - Add a focused config-init handler with read-only `plan_config_init(args)` and side-effecting
     `run_config_init(args)` entry points.
   - Treat a resolving `machine_name` whose selector agrees as already configured. Otherwise scan all machine overlays,
     report the existing valid names as choices, and prompt on a TTY using `args._init_input_func`/`args._init_stdin`
     injection points for tests.
   - Suggest a lowercase hostname sanitized through `[^a-z_] -> _`, accept the suggestion on empty input, validate and
     re-prompt until the name satisfies the schema, and require an explicit default-no confirmation when the chosen
     top-level hood collides with the durable agent-name registry.
   - Selecting an existing overlay will write only the selector. A new selection will minimally splice `machine_name`
     into `~/.config/sase/sase_<name>.yml` with the existing YAML editor, resolve the actual destination through the
     existing chezmoi target policy, then write the selector and clear config caches.
   - When chezmoi is enabled, feed the changed source file into the existing deferred deploy mechanism used by bare
     `sase init`, or run the normal commit/push/apply deploy path for direct `sase config init`. Surface write/deploy
     failures as non-zero exits and refuse interactive setup cleanly on non-TTY input.

4. Wire every CLI and onboarding surface.
   - Add alphabetically sorted `init` parsing and dispatch under `sase config`, with clear help text and no change to
     the config group's existing bare-command behavior.
   - Register Config first in `iter_init_command_specs()`, before Memory, Repos, and Skills, so shared onboarding and
     `sase doctor` see missing identity before later initialization work.
   - Add the `sase init config` compatibility alias, including a short/long check option accepted after the alias,
     dispatch it through the same planner/runner, and update the onboarding help/current-state text to enumerate Config.
   - Verify the generic `config.init` doctor check consumes the new spec, reports missing identity as planned work, and
     becomes current after configuration.

5. Document and test the complete contract.
   - Expand `docs/configuration.md` with `machine_name`, the selector file, selected-vs-foreign machine overlays, the
     initialization commands, chezmoi behavior, and the no-default/legacy-unset behavior.
   - Add schema coverage for the required field and accepted/rejected names while updating existing partial-schema
     fixtures to provide a test identity rather than weakening the public required-field contract.
   - Add config tests for missing selector, selected and foreign machine overlays, ordinary overlays, config
     layers/xprompts parity, accessors, and selector-driven cache invalidation.
   - Add initializer tests for already-configured idempotence, create and minimal-edit writes, existing-overlay
     selection, invalid-name re-prompting, hostname defaults, registry-collision confirmation, non-TTY refusal, write
     failures, cache clearing, and direct/deferred chezmoi deployment.
   - Extend parser, entrypoint, onboarding-order, help, and doctor tests for both command spellings and check mode.

## Validation and completion

1. Run `just install` because this numbered workspace may have stale dependencies.
2. Run focused schema, config, config-init, parser/onboarding, and doctor tests while iterating.
3. Run `just check` as the required full repository validation, fix any failures, and rerun it until clean.
4. Re-read the bead and inspect the final diff to confirm only phase `sase-8k.1` was implemented, then close `sase-8k.1`
   with implementation notes. Do not close parent epic `sase-8k` and do not create any beads.

## Acceptance criteria

- `machine_name` is a required schema field with no bundled default, and callers can optionally retrieve it or require
  it with an actionable error.
- Only the machine overlay matching `~/.sase/machine_name` contributes to runtime config, config inventory, or xprompts;
  no selector means no machine overlay, while ordinary overlays remain unchanged.
- `sase config init` and `sase init config` safely establish or select a valid machine identity, preserve unrelated YAML
  bytes where possible, respect chezmoi, reject non-interactive prompting, and are idempotent.
- Bare `sase init` runs Config first, and doctor accurately reports whether machine identity initialization is needed.
- Documentation and automated tests cover the user-facing workflow and edge cases, and the full `just check` suite
  passes.
- Bead `sase-8k.1` is closed without changing the state of parent epic `sase-8k` and without creating follow-up beads.
