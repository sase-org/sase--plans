---
tier: tale
title: Fix plan approval extras schema rejection
goal: "Approving a tale plan from the ACE TUI (or any shared-API caller) succeeds again:
  the executed gate choice matches the user's actual selection, and the gate input never
  carries fields the target choice's input schema rejects.

  "
create_time: 2026-09-09 19:53:13
status: wip
---

# Plan: Fix plan approval extras schema rejection

## Problem

Approving a tale plan proposed by a SASE agent now fails with a jsonschema error
rendered in the ACE TUI:

```
Additional properties are not allowed ('commit_plan', 'run_coder' were unexpected)
Failed validating 'additionalProperties' in schema:
    {'additionalProperties': False,
     'properties': {'coder_model': {'type': 'string'},
                    'coder_prompt': {'type': 'string'}},
     'type': 'object'}
On instance:
    {'commit_plan': True, 'run_coder': True}
```

The gate stays pending (no response is written), so the agent remains blocked in PLAN
state until the user retries after this fix.

## Root cause

Commit `9ab0c0c58` ("feat(ace): support composable notification gates", sase-6i.5)
remodeled tale plan approval as an `approve` choice plus `commit_plan` / `run_coder`
extras (both default-selected). Two pre-existing sharp edges now combine into a
guaranteed failure:

1. **Wrong executed choice (TUI).** `submit_neutral_plan_response` in
   `src/sase/ace/tui/actions/agents/_notification_modals.py` picks the gate choice to
   execute via `_plan_approval_choice_for_status(result)` — a mapping designed for
   status labels and persist markers. Commit `9ab0c0c58` narrowed its `result.choice`
   passthrough from "any non-None choice" to only `{"tale", "epic"}`, so a remodeled
   approval result (`choice="approve"`, `commit_plan=True`, `run_coder=True` — the modal
   defaults) now falls through to flag-based inference and is remapped to the legacy
   `tale` choice (commit-only maps to `commit`).

2. **Over-wide gate input (actions layer).** `_execute_neutral_plan_approval_response`
   in `src/sase/plan_approval_actions.py` unconditionally copies `commit_plan`,
   `run_coder`, `coder_prompt`, and `coder_model` into the gate `input_data`. But the
   per-choice input schemas baked into the gate bundle (`_plan_input_schema` in
   `src/sase/plan_gate.py`) only allow `commit_plan`/`run_coder` for `approve`, only
   `coder_prompt`/`coder_model` for `approve`/`run`/`tale`, and nothing for `commit` —
   all with `additionalProperties: False`. Executing `tale` with the boolean flags is
   therefore always a schema violation. The flags are also meaningless for preset
   choices: `plan_response_json` only honors overrides for choices with
   `allow_protocol_overrides` (only `approve`) and only forwards coder options for
   choices with `allow_coder_options` (`approve`/`run`/`tale`), deriving everything else
   from the registered protocol in `src/sase/plan_approval_choices.py`.

Net effect: every default TUI approval of a tale plan (Enter with both extras checked,
the `t` preset, or the custom modal with commit enabled) executes `tale`/`commit` with
input `{'commit_plan': ..., 'run_coder': ...}` and is rejected by the executor's input
validation in `src/sase/notification_gates/executor.py`. The same latent wall exists for
the mobile bridge (`execute_mobile_plan_action`) and for pre-remodel in-flight neutral
bundles.

Existing tests never caught this because they exercise `execute_plan_approval_response`
only with `choice="approve"`, and execute the preset choices (`tale`/`run`/`commit`)
only with empty input (`tests/test_plan_gates.py`).

## Fix

Two complementary changes, both host-side (no schema changes, so in-flight gate bundles
— including the currently pending one — are fixed without re-creation):

### 1. Actions layer: filter gate input by choice capabilities

In `_execute_neutral_plan_approval_response` (`src/sase/plan_approval_actions.py`),
build `input_data` according to the resolved choice's registered capabilities, mirroring
exactly what `plan_response_json` consumes:

- include `commit_plan`/`run_coder` only when the choice record has
  `allow_protocol_overrides` (i.e. only `approve`);
- include `coder_prompt`/`coder_model` only when the record has `allow_coder_options`
  (`approve`, `run`, `tale`);
- leave `feedback` and `epic_launch_mode` handling unchanged.

This is behavior-preserving by construction: the dropped fields are precisely the ones
the gate command would ignore anyway, and it protects every caller of the shared
`execute_plan_approval_response` API (TUI, CLI, mobile/Telegram bridge) as well as
pre-remodel neutral bundles executed with preset choices.

### 2. TUI: execute the user's actual choice, not the status alias

In `submit_neutral_plan_response`
(`src/sase/ace/tui/actions/agents/_notification_modals.py`), derive the executed gate
choice from `result.choice` (the product-level choice the modal actually recorded:
`approve`, `tale`, or `epic`), falling back to the existing
`_plan_approval_choice_for_status(result)` inference only when `result.choice` is None,
and finally to `"feedback" if result.feedback else "reject"` as today.

Rationale: with fix 1 alone, a remodeled approval would still execute the legacy `tale`
choice, so `selected_extra_ids` would never be computed
(`_execute_neutral_plan_approval_response` only computes them for `approve`), the
extras' add-on commands would not run, and the recorded response would misrepresent the
user's selection (`extras_selection_provided: False`). The remodel's design intent is
that extras-capable results execute `approve` plus `selected_extra_ids`.

Keep `_plan_approval_choice_for_status` untouched for its actual purposes: status-label
overrides ("TALE APPROVED"), persist markers, and the plan-archive side effect all
continue to treat approve+commit+run as tale-equivalent.

Verify the epic path is unaffected: the epic-validation/launch branch in
`submit_neutral_plan_response` must still trigger for `result.choice == "epic"` results.

## Testing

- Add a regression test that drives `execute_plan_approval_response` against a freshly
  created remodeled tale gate with `choice="tale"` and explicit
  `commit_plan=True, run_coder=True` (the exact failing call): it must succeed and
  produce the tale protocol response instead of a `plan_validation`/`invalid input`
  GateError.
- Add a test that `execute_plan_approval_response` with `choice="commit"` plus a
  `coder_prompt` succeeds (coder options filtered for choices without
  `allow_coder_options`).
- Add a TUI-level test (pattern of existing tests in `tests/ace/tui/`) asserting that
  `submit_neutral_plan_response` with a remodeled-approval result (`choice="approve"`,
  both flags True) executes the `approve` gate choice with both `selected_extra_ids`,
  and that a `choice="tale"` result executes the `tale` choice; assert status override
  still resolves to "TALE APPROVED" in both cases.
- Run the full `just check` suite.

## Risks / notes

- Do not modify `_plan_input_schema` or anything else that feeds the hashed gate bundle:
  widening schemas would only affect newly created gates and would break hash
  verification expectations for existing tests.
- The currently pending `bd` plan gate needs no migration — once the fix is in,
  re-approving from the TUI should succeed against the same bundle.
- `sase plan approve --kind ...` CLI calls pass `commit_plan=None` / `run_coder=None`
  today and are unaffected; the filtering only drops values that would otherwise be
  rejected.
