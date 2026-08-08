---
tier: tale
title: Use one required snooze argument in task-bead gates
goal:
  TaskTriage and BeadSnooze gates collect one required wake-time line in the form
  "<wake-time> [+N]", with no preset enum or second custom field, while pending gates
  migrate safely and invalid input remains retryable.
proposed_by: bbugyi200.athena.vo.f0
create_time: 2026-08-08 11:36:45
status: wip
---

# Plan: Use one required snooze argument in task-bead gates

## Goal

Replace the two-control snooze input shared by task-bead gates with one required
single-line `duration` argument. A reviewer should type a required relative duration or
absolute wake time and may append one optional `+N` threshold, for example `3d`,
`2026-08-09T09:00:00-04:00`, or `3d +2`. This applies to both the **Snooze** branch of a
ready task's `TaskTriage` gate and the **Snooze again** branch of a woken task's
`BeadSnooze` gate.

The option command must continue to validate the expression before a response is
persisted, so malformed values leave the gate pending. Optional branch feedback remains
the reason for the deferral and is not part of the wake-time expression.

## Current behavior and boundaries

`src/sase/bead/snooze_gate_input.py` currently declares a required `duration` enum
(`4h`, `1d`, `3d`, `7d`, or `custom`) followed by an optional `custom_duration` line.
Both trusted gate builders reuse that declaration, and their commands normalize the two
fields into the existing `result.duration` string before host-side effects call the
shared `parse_snooze_request()` grammar.

The generic gate model, CLI, and ACE controls already support required line fields, and
the focus-first-invalid behavior already moves a numbered branch shortcut to an empty
required line. Do not add task-specific UI code or change generic enum support. Also do
not change `BeadSnoozeModal`, `sase bead snooze`, the accepted relative/absolute time
grammar, result schemas, response translators, or bead-store mutation semantics.

## Implementation

### 1. Declare one required line while preserving durable-bundle compatibility

Update `src/sase/bead/snooze_gate_input.py` so `snooze_duration_inputs()` returns
exactly one field:

- id `duration`, rendered with an accurate wake-time label;
- type `line`, `required: true`, with no enum choices and no default;
- placeholder/help text that teaches the required wake-time portion and optional `+N`
  suffix using `ACCEPTED_SNOOZE_FORMS`.

Remove the public custom-field/default/choice constants and the preset-selection logic
from the new authoring contract. Simplify the normal resolver path to strip and parse
the single `duration` string with `parse_snooze_request()`, preserving its current
actionable errors and normalized string result.

Keep a narrow, documented legacy normalization branch for already-persisted bundles
whose command resources import the current resolver at execution time: when the old enum
value is `custom`, consume its old `custom_duration` companion. New gate envelopes must
not declare or accept that property—the compiled one-line input schema has
`additionalProperties: false`—but an old pending gate must remain answerable during the
migration window. Preset values from old bundles naturally continue through the same
parser.

Because both `src/sase/bead/_task_gate_spec.py` and `src/sase/bead/snooze_gate.py`
already consume the shared declaration and emit the same `result.duration`, keep their
option result schemas, response translation, and host effects unchanged apart from
comments/import cleanup made obsolete by the input shape.

### 2. Refresh pending task-bead gates onto the new contract

Extend the reconciliation fingerprint in
`src/sase/scripts/sase_chop_bead_task_triage.py` with an explicit gate-contract version,
separate from the preview/presentation renderer version. Introducing the version into
the fingerprint must make every tracked pending `TaskTriage` or `BeadSnooze` gate
mismatch once, causing the existing cancel-and-recreate path to issue a generation-new
gate with the one-line schema on the next chop tick. Keep the existing cancellation
reason and generation mechanics; no state-schema migration is needed.

Update the fingerprint documentation and its focused tests in
`tests/test_axe_chop_bead_task_triage_presentation.py` to establish that a contract
version bump changes the fingerprint while an unchanged version remains stable. This
prevents future trusted option-schema changes from leaving indefinitely pending gates on
an obsolete interaction contract.

### 3. Update trusted gate, execution, and ACE coverage

Adjust the canonical construction assertions in `tests/test_bead/test_task_gate.py` and
`tests/test_bead/test_snooze_gate.py` to require one `duration` line, no
choices/default, and a compiled schema that requires only `duration`. Update the
forged-contract cases in `tests/test_bead/test_task_gate_validation.py` and
`tests/test_bead/test_snooze_gate_validation.py` so mutations still prove the trusted
adapters reject a different line declaration or compiled schema.

Rewrite the TaskTriage and BeadSnooze execution cases in
`tests/test_bead/test_task_gate_snooze.py`,
`tests/test_bead/test_snooze_gate_actions.py`, and
`tests/test_bead/test_snooze_close_regression.py` to submit `{"duration": "3d +2"}`
directly. Retain coverage for a wake time without `+N`, an unparsable expression leaving
the response absent/pending, missing or wrong-typed duration input, and preservation of
optional feedback as the reason. Add an explicit legacy-command regression showing an
old `duration: custom` plus `custom_duration` payload still normalizes to
`result.duration`; separately prove a newly built gate rejects `custom_duration` as an
extra property.

Update `tests/ace/tui/test_notification_custom_gate.py` to assert the real TaskTriage
loader exposes one required line field. Add or extend a modal interaction regression
using the real task-triage gate data: pressing the third branch shortcut with an empty
duration focuses that line without resolving or warning, and filling `3d +2` returns it
under `option_inputs.snooze.duration`. Keep the existing generic enum-focus regression,
because enums remain a supported custom-gate input type even though task-bead gates no
longer use one.

The changed real-gate layout intentionally updates
`tests/ace/tui/visual/snapshots/png/custom_gate_task_triage_120x40.png`; inspect the
actual/expected/diff artifacts before accepting only that golden.

### 4. Align user-facing documentation

Update `docs/notifications.md`, `docs/axe.md`, and the task-bead gate wording in
`docs/beads.md` to describe one required `duration` line containing a wake time with an
optional `+N` suffix. Remove enum/preset/custom-field language, keep examples of both
relative and absolute wake times, and retain the distinction between the duration input
and optional feedback/reason. Note that the reconciler replaces an obsolete gate
contract, not only obsolete preview presentation.

## Verification

1. Run `just install` before repository commands in the ephemeral workspace.
2. Run focused non-visual tests covering both gate kinds, trusted validation, command
   and host effects, real-store snooze regression, ACE loading/interaction, and
   reconciliation:
   `just test -- tests/test_bead/test_task_gate.py tests/test_bead/test_task_gate_validation.py tests/test_bead/test_task_gate_snooze.py tests/test_bead/test_snooze_gate.py tests/test_bead/test_snooze_gate_validation.py tests/test_bead/test_snooze_gate_actions.py tests/test_bead/test_snooze_close_regression.py tests/ace/tui/test_notification_custom_gate.py tests/ace/tui/test_custom_gate_modal.py tests/test_axe_chop_bead_task_triage_presentation.py`.
3. Run the targeted TaskTriage visual test without update mode and inspect
   `.pytest_cache/sase-visual/` to confirm the only difference is the intended one-line
   input layout. Accept that named golden with
   `just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py::test_custom_gate_task_triage_png_snapshot`,
   then rerun the same node without update mode and require exact equality.
4. Run the repository-required `just check`. If its selector escalates because the
   shared gate contract or reconciliation script is broadening, follow the reported
   escalation with `just check-full`.

## Acceptance criteria

- Newly created `TaskTriage` and `BeadSnooze` snooze branches declare exactly one
  required line input named `duration`; neither declares an enum, default, choices, nor
  `custom_duration`.
- The line accepts the established wake-time vocabulary with an optional positive `+N`
  suffix, and the command rejects missing, blank, wrong-typed, malformed, past, or
  non-positive values without settling the gate.
- A valid answer still emits `result.duration`, applies the same wake time and `+1`
  target, and uses optional feedback only as the deferral reason.
- Existing two-field bundles remain answerable during rollout, while the reconciler
  replaces tracked pending task-bead gates once so their visible controls adopt the new
  contract.
- Pressing a snooze branch shortcut with an empty line focuses that required line and
  does not show the generic invalid-input toast; submission succeeds after correction.
- Documentation, focused tests, the intentional TaskTriage PNG golden, and required
  repository checks agree with the one-line contract.
