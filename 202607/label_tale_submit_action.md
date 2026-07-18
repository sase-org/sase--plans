---
tier: tale
title: Label tale plan submission action as Tale
goal: 'Tale plan review gates label the grouped approval submit control as "Tale"
  across canonical gate consumers and ACE fallback rendering, while preserving option
  identities, selection semantics, and runner protocol behavior.

  '
create_time: 2026-07-18 08:23:17
status: done
prompt: 202607/prompts/label_tale_submit_action.md
---

# Plan: Label tale plan submission action as Tale

## Context and outcome

The tale PlanApproval gate presents two selected member options—"Launch coder agent" and "Commit plan file to the plans
sidecar"—followed by a green grouped submit control still labeled "Approve". That final control represents the
product-level tale action when both default selections are active, so rename its visible label to "Tale". The canonical
neutral gate request must carry the new label for every consumer, including mobile integrations and notification-backed
ACE reviews, and ACE's display-only fallback must render the same wording for direct or legacy modal callers.

This remains a presentation-only change. Preserve the tale query and branch shape, the stable `approve` and `commit`
option IDs, their individual labels, icons and default selections, command resources, input/result schemas, response
messages, and the translated `action="approve"`, `commit_plan`, and `run_coder` fields. Preserve the epic singleton's
"Epic" label and behavior. Do not rename generic custom-gate groups, launch approvals, CLI or auto-approval vocabulary,
keybindings, persistence/status values, or unrelated uses of "Approve".

## Canonical tale group presentation

- Define the tale submit-group label once in the canonical plan-gate presentation layer and use it when publishing the
  `approve AND commit` group, so adapters receive `label: "Tale"` without changing the underlying group members.
- Make the trusted plan-gate validator compare against the same canonical group metadata, including an accurate
  diagnostic for invalid or stale group labels. This keeps request production and validation synchronized instead of
  introducing two independent literals.
- Reuse the canonical tale group definition in `PlanApprovalModal`'s display-only fallback. Notification-backed ACE
  should continue to render the verified envelope directly, while direct/legacy modal construction reaches the same
  visible result without consumer-specific rewriting.
- Keep the generic gate model's inferred group-label behavior unchanged; the wording is specific to authored tale plan
  submission, not a global reinterpretation of an option whose protocol ID happens to be `approve`.

## Contract and interaction coverage

- Update plan-gate request and end-to-end assertions so the tale group contains `approve` and `commit`, carries the
  "Tale" label and existing icon, and still translates selected members into unchanged runner fields. Retain explicit
  checks for the individual "Launch coder agent" option and the separate epic "Epic" singleton.
- Add or adjust validator coverage to prove the canonical "Tale" group is accepted while a forged/stale group label is
  rejected, protecting the trusted adapter contract rather than testing only the producer's output.
- Update notification-loaded and fallback ACE modal coverage to assert that the rendered group submit button says "Tale"
  while its two toggles retain their current labels and selection/result behavior. This should verify both data paths
  that can construct the plan review modal.
- Refresh only the focused tale plan-gate PNG golden and inspect its diff for the intended button-text replacement.
  Confirm the shorter label remains readable and does not alter the other controls; the epic snapshot should remain
  unchanged.

## Validation and handoff

- Run the focused plan-gate contract, end-to-end, notification-modal, direct-modal, and tale PNG snapshot tests while
  iterating, including strict comparison after accepting the intentional golden change.
- Install the workspace's current development dependencies as required, then run `just check` before handoff. If a
  repository-wide failure is unrelated or requires modifying protected generated memory/provider instruction files,
  leave those files untouched, isolate the relevant test lanes, and report the evidence rather than expanding scope.

## Risks and boundaries

The primary risk is changing a generic `approve` presentation globally, which would relabel unrelated gates or disturb
protocol compatibility. Constrain the new text to the tale plan group's presentation metadata and prove that option IDs
and translated results do not change. A second risk is drift between canonical request creation, trusted validation, and
ACE fallback rendering; sharing the group presentation definition and exercising both modal paths addresses it.
