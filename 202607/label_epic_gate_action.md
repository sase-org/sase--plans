---
tier: tale
title: Label the epic plan gate action as Epic
goal: 'Epic plan review gates present their singleton approval option as "Epic" across
  canonical gate consumers while preserving the stable approve protocol and all tale
  gate labels and behavior.

  '
create_time: 2026-07-18 07:44:01
status: done
prompt: 202607/prompts/label_epic_gate_action.md
---

# Plan: Label the epic plan gate action as Epic

## Context and outcome

An authored epic currently produces an `EpicApproval` gate whose singleton `approve` option is labeled "Approve". The
option actually accepts the plan into the epic workflow, so its user-facing label should be "Epic". This follows the
existing tier-aware distinction that labels the tale `approve` option "Launch coder agent" and makes the epic action
explicit without changing what either gate does.

Treat this as a presentation-contract change. Keep `approve` as the durable option ID used by gate queries, command
resources, selection responses, auto-resolution, and host-side translation. Preserve the epic option's icon and default
selection, its `epic_launch_mode` input, the translated `action: epic` result, and the existing commit/run-coder launch
semantics.

## Canonical gate presentation

- Extend the shared tier-aware plan-gate label definition so the epic `approve` option is emitted as "Epic", while the
  tale `approve` option remains "Launch coder agent" and the tale grouped submit control remains "Approve".
- Continue using the canonical neutral gate request as the source of truth for notification and mobile consumers. Do not
  add ACE-, Telegram-, or other consumer-specific rewriting.
- Keep ACE's display-only/direct-call fallback aligned through the same shared label helper. Both notification-backed
  epic reviews and direct epic modal construction should therefore render "Epic" without duplicating label constants.
- Leave adjacent product language outside this one option alone, including the "Epic Review" title, approval status
  text, CLI action names, custom approval choices, notification action types, and the internal `approve`, `tale`, and
  `epic` result vocabulary.

## Regression coverage and verification

- Update the tiered plan-gate contract assertions to require "Epic" on the epic option while continuing to protect the
  epic option ID, query, branch structure, icon, defaults, schemas, and the tale option/group labels.
- Update the end-to-end epic gate exercise to select the unchanged `approve` ID and prove that the renamed control still
  translates to the same epic action and launch booleans.
- Cover both ACE presentation paths: assert the direct epic fallback model uses "Epic", and load an actual epic neutral
  gate into the modal to verify the rendered singleton button also displays "Epic".
- Add a focused epic plan-gate PNG snapshot (or update an existing epic-specific golden if one is introduced first),
  inspect it for the intended text only, and retain the tale snapshot as a regression proving its five controls and
  labels are unchanged.
- Run the focused plan-gate, end-to-end, ACE modal, and strict visual snapshot tests while iterating. Before handoff,
  install the workspace dependencies as required and run `just check`, reporting any unrelated pre-existing failures
  separately from this change.

## Boundaries and risks

The primary risk is confusing a presentation label with the protocol identifier and renaming `approve` in queries,
responses, command paths, or downstream choice handling. Constrain the implementation to the tier-aware label helper and
tests that consume its generated metadata. A second risk is changing generic approval language outside the epic gate;
explicit tale and non-gate assertions should keep this change isolated.
