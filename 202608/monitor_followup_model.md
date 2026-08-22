---
tier: tale
title: Select the follow-up model for sase monitor
goal:
  Agents can select the model used by a delayed monitor follow-up while preserving
  inherited routing by default.
size: medium
proposed_by: bbugyi200.athena.0ad
create_time: 2026-08-22 10:43:07
status: wip
---

# Plan: Select the follow-up model for `sase monitor`

## Problem

`sase monitor start --next ...` launches a successor after the monitored command
settles, but the caller cannot select a different model for that successor. The monitor
member currently inherits the starter's `model` and `reasoning_effort`, and the
follow-up prompt turns those inherited values into live routing directives. Putting
`%model:...` inside `--next` cannot solve this because the continuation body is
intentionally delivered as literal prompt text.

## Outcome

Add an optional `-m/--model MODEL` control to `sase monitor start`. When `--next` is
present, this model expression selects the follow-up agent's model using the same syntax
and normalization as `sase pipe --model` (aliases, provider-qualified model names, and
an optional `@effort`). Omitting the option preserves the existing behavior of
inheriting both model and reasoning effort from the starter. Persist the requested
selection so replay detection, monitor inspection, reconciliation, and delayed
settlement all agree on the launch policy.

## Implementation

1. Extend the shared monitor metadata wire in the linked `sase-core` repository. Add an
   optional `monitor_next_model` field to `AgentMetaWire`, populate it from
   `agent_meta.json` in the scanner, and update Rust serialization/defaulting and
   Python-wire parity coverage. Keep missing fields backward-compatible for existing
   monitor artifacts.

2. Carry the model selection through the Python monitor domain. Add an optional
   successor-model value to `StartMonitorRequest`, include it in the request
   fingerprint, persist it as `monitor_next_model` when creating the monitor member, and
   include it in the durable proc follow-up policy where applicable. Mirror the new core
   field in Python's `AgentMetaWire`, expose it on `MonitorRecord`, and add it to stable
   monitor JSON/detail output so users can audit which model a delayed follow-up will
   use.

3. Add sorted `-m/--model MODEL` parser/help support to `sase monitor start`, matching
   the model-expression wording used by `sase pipe --model`. Normalize blank input to
   omission and reject a nonempty model selection when no `--next` action exists,
   because no successor would consume it. Pass the normalized value into the monitor
   request without resolving aliases early, so delayed launch uses the normal launch
   routing and availability checks.

4. Apply the requested model only at follow-up composition. With no explicit selection,
   retain the current live `%model` plus `%effort` inheritance. With an explicit
   selection, format it through the canonical model-directive helper and emit that
   expression as the live `%model` prefix instead of the inherited routing; let an
   optional `@effort` and alias defaults follow the same semantics as
   `sase pipe --model`. Keep `--next`, monitor reason, command output, and all other
   untrusted text inside the disabled/literal prompt region.

5. Update the canonical generated-skill source at
   `src/sase/xprompts/skills/sase_monitor.md`, not an installed generated copy. Show
   `--model` in the canonical invocation and useful-flags documentation, state that it
   requires a follow-up action, and clarify that `%model` text inside `--next` remains
   literal while `--model` controls routing. Update skill-source contract tests; leave
   global skill deployment until the committed change has landed, as required by the
   generated-skill workflow.

## Tests and verification

- Add parser/help and handler validation tests for `-m/--model`, including its short
  alias, omission, blank values, and the `--model`-without-`--next` error.
- Add monitor request/member/fingerprint tests proving the selection is durable and that
  otherwise-identical starts with different successor models are not treated as
  idempotent replays.
- Add follow-up tests that extract the generated live directives and prove: explicit
  model selection wins; aliases/provider-qualified values and optional effort survive;
  omission still inherits the starter's model and effort; and directive-shaped text in
  `--next` remains inert.
- Add Rust scanner/wire and Python round-trip tests for `monitor_next_model`, plus
  monitor show/list/start JSON coverage and generated-skill source assertions.
- Run `just check` in `sase-core`. In `sase`, run `just install` first, then focused
  monitor/parser/wire/skill tests and `just check`; use `just check-full` through
  `/sase_monitor` if the scoped selector broadens or reports an unusual selection.

## Acceptance criteria

- An agent can invoke `sase monitor start --next '...' --model '@small' ...` and the
  eventual successor is routed through `@small`, even when the starter used another
  model.
- `sase monitor start --next '...' ...` without `--model` behaves exactly as it does
  today, inheriting the starter's model and reasoning effort.
- The selected expression survives detached execution and recovery, appears in monitor
  JSON/detail inspection, participates in replay identity, and remains absent for
  historical records.
- The installed-skill source teaches agents to use `--model` rather than embedding a
  live directive in literal `--next` text.
