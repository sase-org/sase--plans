---
tier: tale
title: Make Enter submit every notification gate's primary action
goal: 'Every SASE notification gate declares one primary resolution branch, and the
  ACE notification review panels submit that primary action with Enter while Space
  remains the explicit option-toggle key.

  '
create_time: 2026-07-18 08:41:44
status: wip
prompt: 202607/prompts/gate_primary_enter.md
---

# Plan: Make Enter submit every notification gate's primary action

## Context

ACE's shared plan/custom gate controls currently bind Enter to activation of the focused widget. A tale plan opens with
the `Launch coder agent` checkbox focused, so Enter toggles that checkbox instead of pressing the gate's `Tale` submit
button. The renderer cannot correct this generically because the durable gate request describes branches and groups but
does not identify which displayed branch is the primary action.

The fix should establish one source of truth in the gate protocol and consume it across the TUI. The tale plan primary
branch is `approve AND commit` and renders as `Tale`; the epic plan primary branch is the singleton `approve` option and
renders as `Epic`. Existing question, launch, workflow HITL, and custom gate producers also need explicit primary
declarations so omission or ambiguity is a creation-time error rather than a UI convention based on ordering.

## Gate contract and producer migration

- Add a required canonical primary-branch declaration to the current gate request model and persisted envelope.
  Represent it as the complete option-ID branch that identifies one displayed singleton or AND-group control, not as a
  mutable checkbox selection. Validate that it is non-empty, contains valid and unique IDs in canonical query order, and
  exactly matches one normalized branch. Reject missing, partial, cross-branch, unknown, or ambiguous primary
  declarations before materializing a gate.
- Treat this required field as a protocol evolution: newly created requests must use the new schema and provide it
  explicitly, while already-persisted v2 bundles remain hash-verifiable and answerable through a read-only compatibility
  projection that derives their historical first branch. Do not let the legacy fallback make omission valid for newly
  created gates.
- Include the primary branch in request hashing, verification, debug/display projections, and the typed `GateSpec`/ACE
  branch data so every consumer sees the same reviewed value. Where automatic resolution is supported, use the declared
  primary branch plus its default-selected members so automation and Enter agree on the default outcome.
- Update every built-in producer and its privileged adapter checks: tale plans declare `approve + commit`, epic plans
  declare `approve`, user questions declare `submit`, launch approvals declare `approve`, and workflow HITL gates
  declare `accept`. Update reusable fixtures and direct custom-gate examples to choose an explicit singleton or group
  branch.
- Update the source template for `/sase_gate` under `src/sase/xprompts/skills/` to document and demonstrate the required
  primary branch. Regenerate managed skills with `sase skill init --force` and deploy them with `chezmoi apply` per the
  generated-skill workflow; do not edit generated provider copies directly.

## ACE primary-action behavior

- Extend the shared `GateBranchData`/`GateBranchControls` projection with the verified primary branch and a dedicated
  `submit_primary_branch` operation. Submitting a primary AND group must press the semantic group button and honor the
  group's current checkbox state; it must not silently restore defaults. Thus an untouched tale submits both `approve`
  and `commit`, while a deliberate Space toggle is preserved when Enter submits the `Tale` branch.
- Replace the shared `activate_control` keymap action with an explicit `submit_primary` action whose default is Enter.
  Keep `toggle_option` on Space and retain `submit_branch` (Ctrl+S by default) for deliberately submitting the currently
  active non-primary branch. Mouse activation and branch navigation should continue to work, and a primary action
  requiring feedback should use the existing validation/focus behavior rather than bypass it.
- Update default configuration, typed keymap metadata/loading, config schema, footer hints, and keymap documentation
  together. Handle the retired `activate_control` override deliberately (with a compatibility alias or a clear migration
  warning) so existing user config cannot accidentally restore the buggy Enter behavior.
- Apply the same Enter-to-primary rule to the specialized notification gate panels: approve for launch gates, accept for
  workflow HITL gates, and submit answers for user-question gates. Space remains the choice toggle in structured
  question controls; focused free-text editors may consume Enter to finish their input before the modal can submit.
  Preserve legacy non-gate notification fallbacks, but give legacy plan/HITL review panels the same visible primary
  behavior where their historical primary action is unambiguous.
- Render or otherwise identify the declared primary singleton/group consistently so the button users see as primary is
  the action Enter invokes. Refresh the affected ACE visual snapshots if the primary styling or footer wording changes.

## Validation and regression coverage

- Add model/service tests for the new required field, canonicalization, schema compatibility, request hashing, and each
  invalid primary-branch shape. Assert that all registered producers and adapter-owned gate shapes emit and enforce
  their intended primary branch.
- Add interaction tests that press Enter, rather than invoking action methods directly: an untouched tale returns
  `approve + commit` and never toggles `Launch coder agent`; an epic returns the `Epic`/`approve` choice; a custom gate
  submits its declared primary even when focus is elsewhere; and a prior Space toggle is honored by the primary group
  submission. Keep coverage for Ctrl+S active-branch submission and required-feedback blocking.
- Cover Enter on launch, HITL, and user-question notification panels, including Space-based question selection and the
  free-text input handoff. Update keymap default, override, duplicate-key, config-schema, documentation, and PNG
  snapshot expectations for the renamed primary-submit action.
- Run `just install` before repository checks, then run focused gate/model/TUI tests during development,
  `just test-visual` for the gate snapshots, and the required final `just check`. Confirm the worktree contains no
  regenerated home/provider skill files or unrelated changes before handoff.

## Risks and boundaries

- The primary declaration identifies a branch control, while the selected members of an AND branch remain mutable UI
  state. Keeping those concepts separate avoids re-enabling options that a reviewer intentionally toggled.
- Protocol evolution must not invalidate unanswered durable v2 notifications; compatibility belongs only in bundle
  loading, not in new-request validation.
- Enter bindings need priority over focused gate buttons and selection lists but must not steal Enter from an active
  feedback/free-text editor. Exercise both focus states explicitly to prevent the fix from trading the plan checkbox bug
  for broken text entry.
