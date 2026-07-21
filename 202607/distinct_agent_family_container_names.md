---
tier: tale
title: Give every agent family its distinct container name
goal: 'ACE consistently presents a sequential agent family by its bare container name
  while preserving every concrete member''s persisted suffixed identity.

  '
create_time: 2026-07-21 16:14:18
status: done
---

- **PROMPT:** [202607/prompts/distinct_agent_family_container_names.md](prompts/distinct_agent_family_container_names.md)

# Plan: Give every agent family its distinct container name

## Context and diagnosis

Sequential-family promotion already establishes the correct durable identity model: when the first follow-up attaches,
the original agent is renamed to a member such as `hc.f0.f0--0`, the bare `hc.f0.f0` name is reserved as the family
container, and later members receive their own `--<suffix>` names. The reported ACE state does not violate that registry
contract. Instead, the `Agent` presentation model currently substitutes the bare family name only for plan-family roots.
A generic promoted root therefore presents `hc.f0.f0--0` on the aggregate family row, and an expanded workflow can show
its concrete `main` agent with the same visible name immediately below it.

This is presentation-only behavior and belongs in the Python/Textual frontend. Do not alter family promotion, name
allocation, the registry, artifact metadata, member lookup, or the Rust core resolution API.

## Implementation

Make the displayed identity policy match the durable family invariant for every sequential family root, regardless of
whether its original member uses `--plan`, `--0`, or a legacy canonical plan suffix:

- Have an `Agent` row that anchors a persisted family derive `presented_agent_name` from `family_reference_name()`. Keep
  `agent_name` unchanged so the original concrete member retains its full suffixed identity and artifact ownership.
- Keep plan-family classification separate from the display decision. It still has distinct responsibilities in concrete
  planner projection and must not be broadened merely to implement the naming fix.
- Ensure ordinary agents, workflow-step children, and non-root family members continue presenting their exact agent
  names. A family root with `agent_name="foo--0"` should present `foo`, while its concrete `main`/member row should
  still present `foo--0`.
- Update nearby model comments and user documentation so they describe the general rule: a family row uses the bare
  container name, and concrete members use `--<suffix>` names. Preserve existing family-root prompt, wait, fork,
  clipboard, and completion resolution through the bare family reference.

## Regression coverage

Extend the focused family-root tests to cover the reported generic promotion rather than only plan families:

- Assert that a persisted generic root keeps `agent_name="foo--0"` but computes `presented_agent_name="foo"`.
- Assert that the Agents-list annotation and selected-row detail header render the bare container name for that root.
- Model an expanded workflow/family shape with a concrete member row and verify that the family row renders `foo` while
  the member renders `foo--0`; also verify another follow-up retains its own full suffix.
- Retain coverage for plan roots, legacy plan suffixes, lone non-family agents, and exact family-member identities so
  the change cannot collapse real agent names into container names.
- Add or adapt an ACE PNG snapshot around a promoted generic `--0` family, inspect the generated actual/expected/diff
  artifacts, and accept only the intentional family-row label change.

## Validation

Because this workspace may be stale, run `just install` before repository checks. Then run the focused model/render
tests, the dedicated `just test-visual` snapshot suite, and finally the required `just check`. If a visual golden
changes, regenerate it only with `--sase-update-visual-snapshots`, inspect the diff, rerun the visual suite without the
update flag, and finish only when the full check passes.

## Acceptance criteria

- A sequential family container is visibly named `hc.f0.f0`, never `hc.f0.f0--0`.
- Its original agent and every other concrete member retain distinct persisted and visible `--<suffix>` identities where
  member rows are rendered.
- Family references and actions continue targeting the bare container, while exact member references continue targeting
  one member.
- Focused tests, visual snapshots, and `just check` pass.
