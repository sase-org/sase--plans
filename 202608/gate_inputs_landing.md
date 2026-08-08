---
tier: epic
title: Close the gate-input epic's own gaps and land it
goal: 'The gate input-collection epic lands green: a custom gate whose option declares
  a raw required property is creatable and answerable again instead of rejected at
  creation, sase-telegram''s custom-gate suite passes against this repo''s validation,
  the conformance matrix asserts the mobile leg the epic actually shipped, submitted
  secrets stay out of journal.jsonl, and the epic''s two input-enforcement layers
  agree.

  '
phases:
- id: answerability
  title: Model what a surface can really submit at creation
  depends_on: []
  size: medium
  description: 'answerability: teach the creation-time answerability probe about the
    raw-schema escape hatch every surface gained later in the epic, so an option declaring
    a required property under `properties` is creatable again, while a required name
    with no control behind it still fails closed.'
- id: telegram-presentation
  title: Repair sase-telegram against the custom-gate presentation contract
  depends_on: []
  size: small
  description: 'telegram-presentation: give sase-telegram''s custom-gate test specs
    the `presentation.title` this epic began requiring, and audit that repo for any
    other spec the same validation now rejects.'
- id: input-integrity
  title: Close the three input-enforcement gaps the epic left open
  depends_on: []
  size: medium
  description: 'input-integrity: redact secret-declared values out of the durable
    journal''s command results, anchor the compiled string patterns so schema validation
    and `InputArg.validate_and_convert` agree, and widen the pre-execution rejection
    recorder so an adapter''s own exception type still reaches `errors/`.'
- id: mobile-conformance
  title: Assert the mobile leg the epic shipped
  depends_on:
  - answerability
  size: medium
  description: 'mobile-conformance: declare the per-option input capability the mobile
    bridge actually has, submit it through the conformance driver, and replace the
    stale closed-phase excuses on whatever the bridge still cannot do with an honest
    reason or the missing support.'
- id: land
  title: Land the epic
  depends_on:
  - answerability
  - telegram-presentation
  - input-integrity
  - mobile-conformance
  size: small
  description: 'land: run the full gate across both repos, close epic bead sase-h7
    with a verification note, clear the epic''s expired symvision whitelist entries,
    and mark the epic plan file done.'
proposed_by: bbugyi200.athena.sase-h7.land
parent_bead: sase-h7
create_time: 2026-08-07 23:11:52
status: wip
bead_id: sase-h7.13
---

- **PROMPT:** [prompts/202608/gate_inputs_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_inputs_landing.md)
- **PARENT:** [202608/gate_input_collection.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_input_collection.md)
- **BEAD:** [sase-h7.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.13.md)

# Plan: Close the gate-input epic's own gaps and land it

Epic `sase-h7` shipped all twelve phases, but three of them contradict each other at
`master` (`6b8c690fc`) and one shipped a gap its own plan forbade. `just check-full` is
red on the combined tree — `6 failed, 27519 passed, 17 skipped` — and every failure is
the epic arguing with itself. This plan closes those gaps and lands the epic.

## What is actually broken

`just check-full` at `6b8c690fc`:

```
FAILED tests/test_gate_cli_show.py::test_show_json_reports_declared_inputs_branches_and_actions
FAILED tests/test_gate_cli_show.py::test_show_prints_a_readable_summary_of_the_decision_surface
FAILED tests/test_gate_cli_show.py::test_show_reports_the_terminal_status_of_an_answered_gate
FAILED tests/test_gate_cli_show.py::test_show_reports_a_cancelled_gate
FAILED tests/gate_conformance/test_gate_conformance.py::test_gate_conformance[cli-legacy_shared_input]
FAILED tests/gate_conformance/test_gate_conformance.py::test_gate_conformance[ace-legacy_shared_input]
6 failed, 27519 passed, 17 skipped, 69 warnings in 295.49s
```

Every lint gate passes, including `sase validate` — the plan-link failure recorded on
the epic bead early in the run is resolved now that the epic plan file is tracked in the
plans sidecar. Five separate agents recorded these six failures as "pre-existing on
master, not mine". They are pre-existing to each of those phases and caused by the epic
as a whole, which makes them this landing's work.

## Phase `answerability` — Model what a surface can really submit at creation

`custom-validation` (`ff0b765a4`) added
`src/sase/notification_gates/kind_validation/custom.py`. Its probe
(`_client_producible_input`) builds "the richest input value a surface can submit" from
the option's declared `inputs` plus `feedback`, then validates that probe against the
option's `input_schema` and rejects the gate at creation when it fails.

That premise was true when it landed and is false now. Two facts make it false:

1. **The probe can only ever fire on raw-schema-only options.** When an option declares
   `inputs`, `GateOption.from_mapping`
   (`src/sase/notification_gates/model_options.py:142-160`) _replaces_ `input_schema`
   with the schema compiled from those inputs — a declared `input_schema` is only
   allowed through when it already equals the compiled one. So for any option with
   `inputs`, the probe is built from exactly the fields the schema was compiled from and
   passes by construction. The check is reachable only for options that declare a raw
   `input_schema` and no `inputs`.

2. **Later phases in this same epic taught three surfaces to answer exactly those
   options.** `inputs-ace` (`e1da6d1b7`) gave `GateBranchControls` a raw-schema YAML
   editor: `_compose_raw_editor` renders a `TextArea` for any option with no `inputs`
   whose `input_schema` declares at least one non-host-collected property, seeds it from
   the properties' defaults, and validates what the reviewer types against that very
   schema (`src/sase/ace/tui/modals/gate_branch_controls.py:272-335`). `gate-cli`
   (`cce9e9e22`) added `--input` and `--option-input OPT=@file|-|literal`.
   `inputs-remote` (`7bbd82a47`) added `option_inputs` to the mobile bridge
   (`src/sase/integrations/mobile_notifications.py:97-118`).

So the check now rejects at creation precisely the class of gate the epic spent three
phases making answerable. Both failing fixtures are that class, and both are deliberate:
`tests/test_gate_cli_show.py`'s `audit` option exists to exercise `show`'s
`raw schema: reason* (* required)` rendering, and `tests/gate_conformance/_cases.py`'s
`_legacy_shared_input` exists because the epic plan requires that "unanswered bundles
created before it must remain hash-verifiable and answerable; the conformance matrix
includes a pre-change fixture bundle to prove it".

**Fix.** Credit the raw-schema escape hatch in the probe, matching what the surfaces do:

- For an option with declared `inputs`, keep today's behavior exactly.
- For an option with no `inputs`, a reviewer edits raw YAML/JSON validated against the
  schema, so any value the schema accepts is producible **provided a control is
  rendered**. Build the probe by contributing a representative value for every name in
  `input_schema.properties` (plus `feedback` under the existing rule), deriving each
  value from that property's own declared `type`/`enum`/`default` where present.
- Keep failing closed on the case that is still a real trap: a name in `required` with
  no matching entry under `properties`. Nothing renders a control for it — ACE's
  `_raw_editor_properties` returns `()` and skips the editor entirely — so the gate is
  genuinely stuck. The existing `unanswerable_option` code, target, and remedy sentence
  stay for that case.

Both failing fixtures must pass **unchanged**; changing them to satisfy the current
check would delete the coverage they exist to provide.

**Alternative considered and rejected.** The stricter reading — keep the validator and
change the fixtures — means raw required schemas become uncreatable, so ACE's raw
editor, `show`'s `*` required marker, and the CLI's `--input` flag would all serve only
pre-epic bundles, and `_legacy_shared_input` would have to hand-write a bundle behind
`create_gate`'s back. The epic never states raw-schema deprecation as a goal, and
acceptance criterion 2 is worded as "cannot accept what **any** client can produce",
which a declared-`properties` required key plainly can. If the strict reading is
preferred instead, say so on the phase bead and the fixtures change instead of the probe
— but do not split the difference.

**Tests.** `tests/test_gate_custom_validation.py` gains: a raw schema with `required`
covered by `properties` is created and answered end to end through the executor; a raw
`required` name absent from `properties` is still rejected with the property named; an
option with `inputs` is unaffected. The six failing tests pass unchanged.

## Phase `telegram-presentation` — Repair sase-telegram against the custom-gate contract

`custom-validation` also made `presentation.title`, `presentation.icon`, and a non-empty
`presentation.notes` mandatory for `kind == "custom"`
(`src/sase/notification_gates/validation.py:184-204`). `sase-telegram`'s
`tests/test_custom_gates.py::_custom_spec()` sets `icon`, `sender`, `notes`, and `files`
but no `title`, so nine tests there fail with `custom gates require presentation.title`.
`inputs-remote` found this, verified it on a stashed tree, and recorded it on the epic
bead without fixing it; it is still live at `sase-telegram` HEAD `afa933b`.

Open the repo with `/sase_repo` (`sase repo open sase-telegram`) and use only the path
it prints.

- Add a `title` to `_custom_spec()` and to any other custom spec in that repo missing
  one; the note claims nine failures, so confirm the count before and after.
- Audit the same repo for specs that omit `icon` or `notes` — the same validation
  requires those too and a fixture may be one edit away from the next failure.
- `sase`'s own `build_task_triage_gate_spec` was named in the same note as a second root
  cause. It already sets `title` via `bounded_gate_title`
  (`src/sase/bead/_task_gate_spec.py:95`) and `task_triage` is its own adapter kind, not
  `custom`, so verify rather than change it; if it needs nothing, say so on the bead.

**Verification.** In `sase-telegram`: `just install`, then install this workspace's
`sase` over the PyPI copy the way CI does
(`uv pip install --python .venv/bin/python --no-deps -e <workspace sase path>`), then
`ruff`, `mypy`, and the full `pytest` run. Report the pass count.

## Phase `input-integrity` — Close the three input-enforcement gaps

Three independent correctness gaps, all recorded as `PROPOSED FOLLOW-UP` by the phase
that found them and all inside this epic's own new code.

**1. Submitted secrets reach `journal.jsonl` unredacted.** `inputs-core` added a
`secret: true` input flag and redacts it out of the write-once response
(`redact_option_inputs`, `executor.py:209`). `executor-integrity`'s journal does not:
`append_journal_event(..., event="option_completed", result=result)`
(`executor.py:186-195`) stores the command's raw result, and `journal.py:111-112` writes
it verbatim. A command that echoes its stdin into its stdout — which the conformance
matrix's own `_ECHO_COMMAND` does — therefore writes the submitted secret into durable
audit data. The epic plan's stated invariant is explicit: "`response.json` and
`journal.jsonl` are audit data … do not ship an unredacted secret field." Apply the same
redaction the response uses to the journal's stored `result` (the `result_digest`
alongside it is already safe and stays), and keep `read_journal_records` /
`incomplete_attempt` / the ACE Journal tab working against the redacted shape.

**2. The two enforcement layers disagree about a trailing newline.** The compiled
fragments in `src/sase/notification_gates/model_inputs.py:55-59` anchor with `$`, which
under Python's `re` matches before a single trailing newline. Confirmed in this
checkout: `Draft202012Validator({'type':'string','minLength':1,'pattern':r'^\S+$'})`
accepts `"ab\n"` for a `word` field, while `InputArg.validate_and_convert`
(`src/sase/xprompt/models.py:138-163`) rejects it. So a value that a typed ACE form
refuses is accepted through `sase gate answer --option-input`, the ACE raw editor, and
the mobile bridge. `inputs-core` implemented the table verbatim from the plan because it
is portable ECMA-262 (where `$` has no such leniency) and named `custom-validation` as
the phase that should pin the dialect and close this; it did not. Close it now — anchor
so the two layers agree, for `word`, `agent`, `line`, and `path`. Cover each type with a
test asserting the schema and `validate_and_convert` reach the same verdict on a
trailing newline.

**3. An adapter's own exception type escapes `errors/`.** `recorded_rejection`
(`src/sase/notification_gates/command_runner.py:253-268`) catches only `GateError`, so
an adapter that rejects with its own type — the plan adapter raises
`PlanApprovalValidationError` — writes no error record and a reviewer pressing `d` sees
nothing. That contradicts acceptance criterion 3, "Every gate rejection — input schema,
bounds, feedback, adapter validation — appears in `errors/`". `gate-actions-ace` hit the
same too-narrow `except` in `accept_edited_origin` and fixed it there only. Broaden the
recorder (or wrap adapter rejections in `GateError` at the `GateAdapter` boundary),
preserving each rejection's own code and target, and test it with a non-`GateError`
adapter rejection.

## Phase `mobile-conformance` — Assert the mobile leg the epic shipped

`gate-cli` built `tests/gate_conformance/` with a capability table and recorded a
`PROPOSED FOLLOW-UP` that `inputs-ace` and `inputs-remote` should flip their entries as
they landed. `inputs-ace` did — its plan spelled out the edit. Nothing told
`inputs-remote` to, so it did not, and `tests/gate_conformance/_surfaces.py` still
reads:

```python
PENDING_CAPABILITY_PHASES = {
    ("mobile", CAP_OPTION_INPUTS): "sase-h7.8 (inputs-remote)",
    ("mobile", CAP_SHARED_INPUT): "sase-h7.8 (inputs-remote)",
    ("mobile", CAP_RETRY): "sase-h7.8 (inputs-remote)",
}
```

All three blame a closed phase, and ten cases skip against it — the epic's own evidence
for acceptance criteria 1 and 4 is not being collected.

- **`option_inputs`: supported, undeclared.** `execute_mobile_gate_action` takes
  `option_inputs` and the host bridge validates it behind `schema_version >= 5`
  (`_mobile_notification_actions.py:47-75`, `mobile_notifications.py:97-118`).
  `_submit_via_mobile` never passes it. Thread it through, add `CAP_OPTION_INPUTS` to
  the mobile `Surface`, and drop that entry. Seven cases start asserting; expect real
  divergences and fix the bridge, not the case, if the mobile leg disagrees with
  CLI/ACE.
- **`shared_input` (1 case) and `retry` (2 cases): genuinely absent.** The bridge takes
  neither a shared `input` nor a `retry`. Decide deliberately: add them, or keep the
  limitation and replace the stale `PENDING_CAPABILITY_PHASES` value with the real
  reason, so a skip stops pointing at a closed bead. Note that without `retry` a mobile
  reviewer who hits a partially executed AND branch gets the `partial_attempt` error and
  cannot proceed — safe, never a silent re-run, but a dead end worth stating.

Depends on `answerability` because the `legacy_shared_input` case cannot be created
until that phase lands.

## Phase `land` — Land the epic

1. `just install`, then `just check-full` on the combined tree — every lint gate plus
   the full suite, which is the gate this epic's plan names for the combined tree. It
   must be exit 0, with no remaining failures and no skip pointing at a closed bead.
2. `just test-visual`. Two phases independently reported
   `test_frontmatter_panel_raw_diagnostics_png_snapshot` failing on clean master; it is
   outside this epic, so confirm whether it still fails and file it with
   `/sase_new_task` rather than fixing it here.
3. `sase bead close sase-h7 --note "<what was verified>"`.
4. `just symvision` — the epic's symbol-whitelist entries expire at close. Remove the
   stale entries and whatever unused code it reports.
5. Set `status: done` in the frontmatter of `202608/gate_input_collection.md`.

## Out of scope

Filed separately with `/sase_new_task` by the landing agent, because they are not caused
by this epic:

- `sase commit`'s before-commit hook closed phase bead `sase-h7.3` on a linked-repo
  (sase-core) commit while the primary repo's implementation was still in progress.
- A single-phase workspace materializes a phase's own plan file but not its `PARENT`
  chain, so `sase plan links validate` fails there and `sase plan links repair` neither
  detects nor fixes it.
- `test_frontmatter_panel_raw_diagnostics_png_snapshot`, if step 2 above confirms it.

Already resolved during the epic and needing nothing: `read_journal_records` is public;
the mobile wire carries `enum` choices via `MobileInputChoiceWire`;
`tests/doctor/test_checks_providers.py` passes again; and `feedback_input.py` is
deliberately retained with a revised deletion trigger in its docstring.
