---
tier: epic
title: Clan-scoped toobig chop launches
goal: 'Every toobig_split run launches its accepted split-file agents as one toobig-@
  clan, and Axe prevents another run while an active clan whose name starts with toobig-
  exists.

  '
phases:
- id: core_contracts
  title: Shared chop contracts and guard engine
  depends_on: []
  description: '''Shared chop contracts and guard engine'' section: extend the Rust
    chop wire, validation, and decision engine with clan-aware proposals and active
    agent-clan prefix guards.

    '
- id: sase_runtime
  title: SASE runtime, SDK, and configuration integration
  depends_on:
  - core_contracts
  description: '''SASE runtime, SDK, and configuration integration'' section: expose
    the core contracts through Python, batch clan-scoped proposals safely, snapshot
    active clan metadata, document the config, and preserve legacy launches.

    '
- id: toobig_chop
  title: toobig_split clan emission
  depends_on:
  - sase_runtime
  description: '''toobig_split clan emission'' section: update bugyi-chops so one
    scan emits stable member IDs in a shared toobig-@ clan while retaining its wait
    chain and dedupe behavior.

    '
- id: athena_rollout
  title: Athena rollout and end-to-end verification
  depends_on:
  - sase_runtime
  - toobig_chop
  description: '''Athena rollout and end-to-end verification'' section: deploy compatible
    SASE and bugyi-chops releases, switch the configured guard to the toobig- clan
    prefix, and verify fire, skip, and rerun behavior.

    '
create_time: 2026-07-19 18:34:59
status: wip
---

# Plan: Clan-scoped toobig chop launches

## Outcome and current behavior

The `toobig_split` script in `bbugyi200/bugyi-chops` currently emits one proposal per oversized file, gives each agent
an independent `split_file.*-@` name, and chains proposals with `wait_on`. Athena config in the `chezmoi` repository
prevents overlap with `inhibit_if.agent_hood: {hood: split_file}`. SASE validates the result in `sase-core`, then its
Python runner scaffolds every proposal as `%id(<name>, tribe=chop)` and launches proposals one at a time. The chop guard
engine only understands `changespec` and `agent_hood`; its active-agent snapshot does not carry canonical clan metadata.

The target behavior is:

- One actionable `toobig_split` result owns one newly allocated `toobig-@` clan, and every accepted proposal is a member
  of that same concrete generation.
- The existing scan order, sequential wait chain, workspace allocation, content-sensitive dedupe, per-agent chop
  lifecycle tracking, and default `chop` tribe attribution remain intact.
- `inhibit_if.agent_clan: {name_prefix: toobig-}` skips dispatch when any active agent belongs to a matching clan.
  Waiting, starting, running, question-blocked, and other non-terminal members count as active; completed historical
  clans do not block the next scheduled run.
- Manual runs continue to honor the guard, while the existing `--force` escape hatch bypasses it along with the other
  declarative policies.
- Chops that do not opt into a clan retain their current result schema behavior and `%id(..., tribe=chop)` scaffold.

Use a first-class optional `clan` value on a launch proposal instead of asking third-party scripts to inject raw clan
directives into `prompt`. Treat `agent_name` as the member identifier when `clan` is set, and have the SASE runner form
and validate the full hood-qualified name. This keeps launch metadata structural, prevents conflicts with the runner's
own `%id` scaffold, and lets the runner coordinate declaration, joins, waits, previews, and lifecycle records.

## Shared chop contracts and guard engine

Implement the frontend-neutral portions in `sase-core/crates/sase_core/src/axe_chop` and carry them through the PyO3
binding without reimplementing decisions in Python.

- Extend the launch-proposal wire with an optional clan template. Validate it as a nonblank, single-token agent-name
  template/reference and validate the composed clan/member identity. Reject ambiguous combinations such as multiple
  template markers in the composed name, blank member IDs, malformed dotted hoods, or a proposal shape that cannot be
  represented by the clan directives. Preserve all existing non-clan proposal normalization.
- Add the `agent_clan` inhibit provider with required, nonblank `name_prefix`. Support both tagged-list and keyed-map
  Axe config forms, reject unknown fields fail-closed, and include the provider in actionable diagnostics.
- Extend agent decision snapshots with optional canonical clan identity. Evaluate the new guard only against `active`
  rows and match the clan itself with case-sensitive `starts_with`; do not infer a clan from the agent name or dotted
  hood. Return a stable skip provider/reason naming the matching active clan and member.
- Add Rust coverage for proposal round trips and invalid clan/member combinations; keyed and tagged guard validation;
  matching and non-matching prefixes; active versus inactive rows; missing clan metadata; guard short-circuiting; and
  schema/binding serialization parity. Let release-plz own Rust crate versions and version pins.
- Validate with `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
  `cargo test --workspace` in `sase-core`.

## SASE runtime, SDK, and configuration integration

Adopt the new core contract throughout SASE's public chop SDK, config schema, host snapshot adapter, proposal launcher,
inventory/docs, and focused tests.

- Add `clan` to `launch_proposal`, `ChopResultBuilder.propose`, and `add_proposal`, and carry it through prepared
  proposals and JSON-safe previews. Document that a clan-scoped proposal's `agent_name` is its member ID, while the
  runner owns the concrete clan allocation and full agent name.
- Route the accepted clan-scoped proposal set through the existing multi-prompt clan planner so all members are
  preplanned against one concrete template token and generation before spawning. The first accepted member of each clan
  template declares the clan and assigns the default `chop` tribe at clan level; later members join it. Reuse the
  established multi-launch partial-failure result so already-started members remain recorded and monitored if a later
  spawn fails.
- Preserve per-proposal environment, model/effort, workspace reference, `wait_on`, dedupe key, launch callback, and
  `agent_chops.json` association. Waits and previews must use the full resolved member identity. If dedupe removes the
  original head of a clan or wait chain, the first surviving proposal becomes the declarer and all effective waits
  remain valid. A dry run must show the same declaration/join and effective-wait plan that a live run would execute,
  without reserving names or spawning agents.
- Keep the old single-proposal/sequential path and exact `tribe=chop` scaffold for results without `clan`. Reject or
  define any explicit `tribe` plus `clan` interaction consistently; clan membership must never produce mutually
  exclusive `%id(..., tribe=...)` and `%id(..., clan=...)` directives.
- Carry `agent_clan` from canonical active agent metadata into the host snapshot sent to Rust. Do not derive it from a
  dotted name, because ordinary hoods and real clans are different concepts. Ensure waiting agents are present so a
  serial clan remains guarded while later members wait.
- Extend both duplicated Axe JSON-schema shapes, Rust-backed config parsing tests, policy tests, public documentation,
  chop SDK examples, dry-run output, and inventory/doctor summaries for `agent_clan: {name_prefix: ...}` and proposal
  clans. Cover legacy non-clan behavior, one and many members, full-name/wait resolution, multiple clan templates,
  deduped heads, partial launches, forced runs, and active-clan skip reasons.
- Run `just install` before repository checks, then `just check` as required for SASE changes. Also exercise focused
  chop SDK/result-policy tests against the just-built Rust binding so stale installed bindings cannot mask failures.

## toobig_split clan emission

Update `bbugyi200/bugyi-chops` only after the released/available SASE SDK accepts clan-scoped proposals.

- Set `clan="toobig-@"` on every proposal from one scan. Change `_agent_name` to return a deterministic, bounded
  `split_file.<path-slug>.<digest>` member ID without its own `@`; the clan template supplies the single allocation
  marker and uniqueness across runs. Keep path normalization and collision-resistant digesting.
- Preserve proposal IDs, `%auto #split_file:<path>` prompts, content-derived dedupe keys, and the exact `wait_on` chain.
  The resulting concrete names should be `toobig-<token>.split_file.<member>`, all with the same clan metadata and
  generation for a scan.
- Update package tests and README examples to assert the clan field, marker-free member IDs, stable order, and unchanged
  scan/error/no-op behavior. Replace the documented `agent_hood` overlap policy with the new active `agent_clan` prefix
  policy.
- Set the package's SASE lower bound to the first release that provides this result contract, follow the repository's
  normal package-version/tag release process, and run `just check` (lint, tests, build, and artifact validation).

## Athena rollout and end-to-end verification

Roll out only after compatible `sase-core`, SASE, and `bugyi-chops` artifacts are installable together.

- In `chezmoi/home/dot_config/sase/sase_athena.yml`, replace the `split_file` hood guard on `toobig_split` with
  `agent_clan: {name_prefix: toobig-}`. Keep the existing cadence, target expansion, scanner limits, and script
  identity.
- Update/install SASE and `bugyi-chops` through their supported package/plugin flows, restart Axe through the supported
  mechanism, and confirm `sase axe chop list --available --verbose` reports the configured guard and compatible script.
  Apply committed chezmoi changes with the repository-required `chezmoi update -a --force` workflow.
- Run the configured chop in dry-run/verbose mode and verify one planned `toobig-@` clan, hood-qualified member names,
  the sequential waits, and no launches. In an isolated test state, exercise a live synthetic scan: the first run
  launches one clan; a second scheduled/manual non-forced run while any member is active records a visible skip naming
  that clan; `--force` still dispatches; and a normal run is eligible again once all members are terminal.
- Run the chezmoi repository's relevant config/bash checks and `just check`. Do not use the production oversized-file
  scan as the first live test, because it would create real coding agents before the guard and grouping assertions are
  proven in isolation.

## Cross-phase acceptance and risks

The feature is complete when the core and Python schemas agree, a third-party chop can express clan membership without
raw directives, dry-run and live launch plans agree, every launched member is durably attributed to one clan generation
and the originating chop, and the configured prefix guard blocks only while a matching clan is active.

Pay particular attention to three race/compatibility risks:

1. Guard evaluation and launch allocation are separate operations. Clan preplanning must remain collision-safe and fail
   closed if another launch wins the name reservation after preflight; never silently split a batch across clan
   generations.
2. Dedupe and partial failure can change the effective head of a batch. Declaration ownership, waits, once-per key
   release, and launched-run finalization must be based on accepted/actually started proposals rather than original list
   position.
3. Result-schema support, the Python SDK, the Rust binding, the external package, and Athena config deploy at different
   times. Land and release them in dependency order, keep non-clan proposals backward-compatible, and do not enable the
   new config provider until the running SASE build validates and evaluates it.
