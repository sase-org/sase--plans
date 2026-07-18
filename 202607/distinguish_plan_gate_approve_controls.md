---
tier: tale
title: Distinguish tale plan gate approval controls
goal: 'Tale plan review gates label the coder-launch option as "Launch coder agent"
  while retaining "Approve" for the grouped submit action, so each control states
  its distinct purpose consistently across Telegram and ACE without changing gate
  protocol behavior.

  '
create_time: 2026-07-18 07:26:29
status: wip
prompt: 202607/prompts/distinguish_plan_gate_approve_controls.md
---

# Plan: Distinguish tale plan gate approval controls

## Context and outcome

The expanded tale PlanApproval gate currently renders two controls named "Approve": the first is the selectable
`approve` option whose protocol effect is to set `run_coder`, and the second submits the currently selected `approve`
and/or `commit` options. This makes the Telegram presentation shown in the report ambiguous, and ACE consumes the same
neutral gate metadata. Rename the first control to "Launch coder agent" so the option describes its consequence, while
leaving the grouped submit control labeled "Approve".

This is a presentation-only change. Preserve the stable `approve` option ID, the tale option query and branch structure,
icons and defaults, command resources, response schemas, and the resulting `commit_plan`/`run_coder` protocol fields.
The epic gate's singleton approval option must remain "Approve": its consequence is launching epic/bead work, not a
coder agent, and it has no duplicate grouped submit control to disambiguate.

## Gate presentation contract

- Make plan-gate option labeling tier-aware so a tale's `approve` option is emitted as "Launch coder agent", while all
  other option labels and the tale group's "Approve" label remain unchanged.
- Keep the canonical neutral gate request as the source of truth used by mobile consumers such as Telegram and by
  notification-backed ACE plan reviews. Do not introduce mobile-specific label rewriting.
- Align the display-only/direct-call fallback used by `PlanApprovalModal` with the canonical contract: tale fallback
  data uses "Launch coder agent", whereas epic fallback data continues to use "Approve". Prefer sharing the tier-aware
  presentation definition over adding another independent label that can drift.
- Leave the broader approval vocabulary alone: CLI actions, auto-approval modes, custom coder-options choices,
  persistence/status messages, and response text still represent product actions named `approve`, `tale`, and `epic` and
  should not be renamed as part of this gate-control clarification.

## Regression coverage and verification

- Update the plan-gate contract and end-to-end smoke assertions to require "Launch coder agent" on the tale `approve`
  option, continue requiring "Approve" on its grouped submit control, and explicitly protect the epic option's unchanged
  "Approve" label.
- Extend ACE coverage for both notification-loaded gate metadata and the direct fallback modal so the rendered top
  toggle and lower submit control have the intended distinct labels. Keep selection/result assertions proving that the
  renamed option still produces the same option IDs and coder/commit behavior.
- Refresh the focused tale plan-gate PNG golden to capture the intentional text change, and inspect the resulting
  snapshot to ensure the longer label fits the existing layout without obscuring the commit or submit controls.
- Run the focused plan-gate, modal, and visual snapshot tests while iterating. Before handoff, install the workspace
  dependencies as required by the project and run `just check` for the full lint, type-check, and test suite.

## Boundaries and risks

The main regression risk is applying the new label to every internal `approve` option and thereby mislabeling
EpicApproval or unrelated custom/launch gates. Constrain the change to the tale plan-gate presentation contract and
cover both tiers explicitly. The longer text may alter ACE layout, so the visual snapshot is part of the acceptance
criteria rather than an incidental golden update.
