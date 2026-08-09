---
tier: tale
title: Generic typed input collection in the ACE gate modals
goal:
  Extract the typed-field form out of `InputCollectionModal` into a reusable
  `TypedInputForm` widget, render each selected gate option's declared `inputs:` inside
  `GateBranchControls`, and deliver the collected values per option through the custom
  and plan submission paths — so plan, epic, custom, triage, and snooze gates all
  collect typed, validated input with no per-kind code.
size: large
proposed_by: bbugyi200.athena.sase-h7.6
bead: sase-h7.6
create_time: 2026-08-07 20:10:44
status: done
---

- **PARENT:**
  [202608/gate_input_collection.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_input_collection.md)
- **BEAD:**
  [sase-h7.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-h7.6](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-h7.6.md)
- **COMMITS:**
  - [e1da6d1](https://github.com/sase-org/sase/commit/e1da6d1b76fd1ea28bc620ab20ad63085842e932)
    — feat(notification-gates): collect typed gate inputs in the ACE modals

# Plan: Generic typed input collection in the ACE gate modals

This is phase `inputs-ace` of epic `sase-h7` (bead `sase-h7.6`). The epic plan lives at
`plans:202608/gate_input_collection.md`; read its `inputs-ace` section for the framing.
This plan is self-contained and supersedes that section wherever the two differ — every
deviation is called out under "Deviations from the epic plan" at the end, with its
reason.

## Background — what is already landed and what is missing

The transport, the authoring vocabulary, and the enforcement layer all exist:

- `GateInputField` and `compile_gate_input_schema` in
  `src/sase/notification_gates/model_inputs.py` are the closed `inputs:` vocabulary.
  `GateOption.inputs` (`model_options.py:92`) parses and round-trips it, and the
  compiled JSON Schema is stored as the option's `input_schema`
  (`model_options.py:142-163`), which is what the executor enforces.
- `execute_gate_selection` already accepts `option_inputs: Mapping[str, object] | None`
  (`src/sase/notification_gates/executor.py:69`), resolves one value per selected option
  (`executor_inputs.py:17-61`), injects the reviewer note as `input.feedback` wherever a
  schema declares it, and redacts `secret: true` fields on the way into `response.json`
  (`executor_inputs.py:64-95`).
- `src/sase/notification_gates/input_collection.py` already answers the three questions
  every surface has to answer — `collected_input_fields()` (union of a selection's
  declared fields, deduped by id, conflicting redeclarations rejected),
  `input_arg_for_field()` (projection onto the shared xprompt `InputArg` rules),
  `coerce_field_text()`, and `option_inputs_from_values()` (distribute one collected set
  back to each declaring option). **Reuse these; do not reimplement any of them in
  ACE.**

What is missing is exactly one thing: **no ACE surface can produce a value.**

- `GateBranchControls` composes exactly one text widget, `#gate-feedback-input`
  (`src/sase/ace/tui/modals/gate_branch_controls.py:163-164`), and its `Resolved`
  message carries only `selected_option_ids` and `feedback` (`:36-46`).
- `CustomGateModalResult` carries the same two fields
  (`src/sase/ace/tui/modals/custom_gate_modal.py:66-72`), and
  `_notification_custom_gate.py:56-65` submits with no input at all.
- `GateSubmission` (`_notification_gate_execution.py:18-25`) has `input_data` but no
  `option_inputs`, so even a caller that had per-option values could not send them.
- `tests/gate_conformance/_surfaces.py:35-40` records this as a declared gap:
  `("ace", CAP_OPTION_INPUTS): "sase-h7.6 (inputs-ace)"`. Closing that entry is a
  deliverable of this plan.

The typed-field machinery that ACE needs already exists, but is welded into
`src/sase/ace/tui/modals/input_collection_modal.py` (566 lines) and coupled to
`PromptInputPlan`: `_InputCollectionInput` and `_PathField` with its `ctrl+t` completion
(`:33-91`), the per-field block builder (`:201-229`), per-field validation against
`InputArg.validate_and_convert` with an inline error label (`:385-407`), the
optional-input reveal (`:298-302`, `:470-478`), and the confirm-enable rule
(`:409-432`).

## Goals

1. One reusable typed-field form widget, driven by the shared xprompt `InputArg` rules,
   used by both `InputCollectionModal` and the gate modals.
2. `GateBranchControls` renders each selected option's declared `inputs:` generically —
   no gate-kind branching anywhere in the widget.
3. Collected values reach the executor as `option_inputs`, one value per option, on both
   the custom-gate path and the plan/epic path.
4. `just test-visual` goldens for the xprompt input modal are unchanged, because the
   extraction is a pure refactor of that modal.

## Non-goals

- No general JSON-Schema-to-form renderer. The closed `inputs:` vocabulary plus a raw
  YAML escape hatch is the contract (epic non-goals).
- The `question` gate keeps `user_question_modal.py`. Do not touch it.
- `SchemaObjectForm` (`schema_object_form.py`) stays where it is; only its YAML-fallback
  _pattern_ is borrowed for the escape hatch.
- No new input path into `argv`. Stdin JSON is the channel.
- Do not add declared `inputs:` to any built-in gate. That is `sase-h7.11`
  (`retire-smuggling`). This plan only makes them render and submit when declared.

## Step 1 — Extract the typed-field form into a reusable widget

**Land this step and run its tests before touching any gate code.**
`InputCollectionModal` is on the agent-launch path, which is used far more than gates; a
regression here breaks prompt launching.

New file `src/sase/ace/tui/widgets/typed_input_form.py`:

```python
@dataclass(frozen=True)
class TypedFormField:
    """One field a TypedInputForm renders, with its presentation."""

    arg: InputArg              # type rules, choices, default (UNSET == required)
    label: str                 # header text; defaults to arg.name
    placeholder: str | None = None
    secret: bool = False

    @property
    def required(self) -> bool:
        return self.arg.default is UNSET
```

```python
class TypedInputForm(Vertical):
    """Typed, validated single-page field collection driven by `InputArg` rules."""

    class Changed(Message):
        """A field's text changed; the host should refresh its submit state."""

    class Submitted(Message):
        """<enter> was pressed past this form's last visible field."""
```

Constructor:
`TypedInputForm(fields, *, id_prefix="field", index_offset=0, optional_toggle=True, id=None, classes=None)`.

Move into it, unchanged in behavior: `_InputCollectionInput`, `_PathField` (with its
`ctrl+t` completion), `_load_type_rules`, the field block builder, the placeholder-text
default hint, per-field inline validation, and the optional reveal.

Widget ids stay `f"{id_prefix}-input-{index_offset + i}"`,
`f"{id_prefix}-block-{index_offset + i}"`, `f"{id_prefix}-error-{index_offset + i}"`,
plus `f"{id_prefix}-toggle-optional"` when `optional_toggle` and optional fields exist.
`InputCollectionModal` passes `id_prefix="field"` and `index_offset=len(placeholders)`,
so **every existing id is byte-identical** and the visual goldens and existing tests do
not move.

> The existing button id is `toggle-optional`, not `field-toggle-optional`. Keep the
> exact literal `toggle-optional` when `id_prefix == "field"` — either by special-casing
> that one id or by taking the toggle id as its own constructor argument. Changing it
> would break `tests/ace/tui/visual/test_ace_png_snapshots_inputs.py:108` and the
> `InputCollectionModal #toggle-optional` rule at `src/sase/ace/tui/styles.tcss:2856`.

Public API the hosts need:

- `values() -> dict[str, str]` — raw text per field name, omitting empty optional
  fields.
- `typed_values() -> dict[str, Any]` — converted values via
  `sase.notification_gates.input_collection.coerce_field_text` for gate fields, or
  `InputArg.validate_and_convert` for xprompt fields. Implement one converter that
  handles `repeatable` the way `coerce_field_text` does, and call _that_; do not fork
  the conversion rules.
- `is_valid() -> bool` — every visible required field non-empty and every non-empty
  field converting cleanly. Hidden fields never block.
- `focus_first() -> bool`, `focus_field(index)`, `focus_next_after(index) -> bool`.
- `set_field_visible(field_name, visible)` and `visible_field_names()`.
- `visible_control_ids() -> list[str]` — focusable ids in render order, for the gate
  modals' focus ring.

Two new editor kinds:

- **`InputType.ENUM`** — `_EnumField(Button)` cycling the declared choices. It renders
  `choice.label or choice.value`; pressing it advances to the next choice and posts
  `Changed`. A **required** enum with no default starts on a `— select —` sentinel that
  is not a submittable value, so "required blocks submit" still holds; an optional enum
  starts on its declared default, or on the sentinel when it has none. Cycling wraps and
  never returns to the sentinel once a real choice is made.
- **`secret=True`** — `Input(password=True)` rather than the vim text area. Both
  `SingleLineVimTextArea` (`single_line_vim_text_area.py:82`) and `Input` expose
  `.value`, so read through one small accessor rather than branching at every call site.

`InputCollectionModal` then keeps its raw-placeholder machinery, its `ctrl+l` literal
handling, its subtitle/filled-status labels, and its confirm button, and composes one
`TypedInputForm` for the declared fields. Wire the traversal so `<enter>` still walks
placeholders → declared fields → confirm: the modal's placeholder chain hands off with
`form.focus_first()`, and the form's `Submitted` message triggers `_try_confirm()`.
`_all_valid()` becomes "every placeholder valid **and** `form.is_valid()`", and
`_filled_count()` keeps counting placeholders plus required declared fields — read the
counts off the form rather than re-deriving them.

Styles: the existing `InputCollectionModal .field-header`, `.field-desc`,
`.field-error`, `.input-field-block`, and `InputCollectionModal SingleLineVimTextArea`
rules (`styles.tcss:2803-2855`) are descendant selectors and keep matching through the
new wrapper. Give `TypedInputForm` `height: auto` with no border, padding, or margin so
the extraction cannot shift layout.

## Step 2 — Render declared inputs in `GateBranchControls`

In `src/sase/ace/tui/modals/gate_branch_controls.py`:

**Compose.** For each branch, after that branch's controls and before the shared
feedback field, mount one `TypedInputForm` holding the union of _all_ that branch's
options' declared fields — computed once with
`collected_input_fields([options in branch])` — with
`id_prefix=f"gate-branch-{branch_index}-field"` and `optional_toggle=False`. Composing
the full union once and revealing per selection avoids remounting widgets on every
toggle. Give the form container id `f"gate-inputs-{branch_index}"` and hide it entirely
when the branch declares no fields, so a gate written before this phase renders exactly
as it does today.

Precede a non-empty form with `Static("Inputs", classes="gate-review-section-title")`
inside the same container, so the header hides with it. Keep it visually subordinate to
**Decision** — that section title style already reads as a muted sub-header, so no new
style rule is needed for it.

`collected_input_fields` raises `GateError("conflicting_input_field", ...)` when two
options in the same branch declare one id with a different `type`, `choices`, or
`repeatable`. Catch it at compose time, render the message as a `Static` inside that
branch's container, and permanently block that branch's submit with the existing
`_update_submit_state` path. A gate that cannot be collected must say so, not silently
collect the wrong thing.

**Reveal on toggle.** Only the currently selected options' fields are visible. On
`toggle_option` and on `_set_active_branch`, recompute
`collected_input_fields(selected options)` and call `set_field_visible` for each field
in the branch's union. For a singleton branch every field is always visible.

**Block submit while invalid.** Extend `_update_submit_state` (`:398-410`) and
`_refresh_submit_states` (`:325-332`) so a branch's submit control — singleton button or
group submit — is disabled while that branch's form reports `is_valid() is False`. Today
`_update_submit_state` returns early for singleton branches (`:400`); it must now handle
them too, because a singleton option can declare required inputs. Subscribe to
`TypedInputForm.Changed` to refresh.

**Resolve.** `GateBranchControls.Resolved` gains a third field:

```python
option_inputs: Mapping[str, dict[str, Any]]
```

built in `_resolve_branch` (`:334-350`) as
`option_inputs_from_values(selected_options, form.typed_values())`, or `{}` when the
branch collected nothing (no declared fields and no raw editor). `{}` means "submit the
way ACE does today"; a non-empty mapping means "deliver per option". Keep the field last
with a `{}` default so the two existing positional constructions in
`tests/ace/tui/test_custom_gate_modal.py` keep working.

If `typed_values()` raises `XPromptValidationError` at resolve time (it should not —
submit is disabled while invalid), notify and refuse rather than submitting a partial
value.

**Button ids.** `on_button_pressed` (`:174-190`) stops every event whose button id
starts with `gate-`, and dispatches on `gate-option-`. Namespace every new control under
`gate-branch-<n>-field-…` and `gate-inputs-…` and dispatch those **before** the existing
branches, or the enum-cycling button will be swallowed by the branch dispatcher.

## Step 3 — Raw `input_schema` escape hatch

An option that declares no `inputs:` but carries a raw `input_schema` with at least one
property renders a single YAML `TextArea` for that option (id
`f"gate-branch-{n}-raw-{option_id}"`), seeded with the schema's declared defaults —
`SchemaObjectForm`'s fallback _pattern_ (`schema_object_form.py:505-524`), not its
config-layering model. Validate on change with
`sase.notification_gates.model_validation.first_schema_error(parsed, option.input_schema)`
after `yaml.safe_load`, showing the message inline and blocking that branch's submit
while it fails. Its value goes into `option_inputs[option.id]` directly rather than
through `option_inputs_from_values`.

Render nothing at all when the schema has no `properties` — that covers
`NO_INPUT_SCHEMA` and the `{"type": "object", "additionalProperties": false}` shape that
`bead_snooze` and `task_triage` options use today
(`src/sase/bead/snooze_gate.py:143-146`), so those gates stay visually unchanged.

**Properties the host already collects are skipped.** `GateBranchControls` takes a new
constructor argument:

```python
host_collected_properties: frozenset[str] = frozenset({"feedback"})
```

Any property in that set is dropped from the raw editor's seeded value and from its
"does this schema have anything to collect" test; when nothing is left, no editor
renders. `feedback` is in the default set because the feedback control already collects
it and the executor injects it.

`PlanApprovalModal` passes
`frozenset({"feedback", "coder_prompt", "coder_model", "epic_launch_mode"})`, because
each of those is already collected by that modal's own controls — `c` opens
`ApproveOptionsModal` for the coder fields, and `epic_launch_mode` is chosen by the host
in `execute_plan_approval_response`. Without this, every plan and epic gate would grow a
raw YAML box duplicating UI that already exists (`src/sase/plan_gate.py:564-591`
declares exactly those properties). This is a presentation fact the plan modal already
owns, not gate-kind logic inside the shared widget — `GateBranchControls` itself stays
generic.

`CustomGateModal` passes the default.

## Step 4 — Deliver the collected values

**`GateSubmission`** (`_notification_gate_execution.py:18-25`) gains
`option_inputs: Mapping[str, object] | None = None`, forwarded to
`execute_gate_selection(..., option_inputs=submission.option_inputs)` at `:140-150`.

**Custom path.** `CustomGateModalResult` (`custom_gate_modal.py:66-72`) gains
`option_inputs: Mapping[str, dict[str, Any]] = field(default_factory=dict)`, filled from
the `Resolved` event at `custom_gate_modal.py:201-210`.
`_notification_custom_gate.py:56-65` stops relying on "no input is sent" and passes
`option_inputs=result.option_inputs or None`. Keep the comment's point: when nothing was
collected we still send nothing and the executor injects feedback.

**Plan and epic path.** `PlanApprovalResult` (`plan_approval_modal.py:182-194`) gains
the same field with a `{}` default (so `_prompt_bar_submit.py:140` and
`llm_provider/_plan_utils.py` keep constructing it unchanged).
`PlanApprovalModal._result_for_selection` (`:621-658`) takes `option_inputs` and carries
it onto the result; `on_gate_branch_controls_resolved` (`:434-443`) passes the event's
mapping.

Then thread it to the executor:

- `submit_neutral_plan_response` (`_notification_plan_gate.py:67-110`) passes
  `option_inputs=result.option_inputs or None` to `execute_plan_approval_response`.
- `execute_plan_approval_response` / the `_execute_*` body in
  `src/sase/plan_approval_actions.py:237-265` gains
  `option_inputs: Mapping[str, Mapping[str, Any]] | None = None`. When it is `None` or
  every entry is empty, call `execute_gate_selection` **exactly as today** with the
  shared `input_data`. Otherwise build

  ```python
  per_option = {
      option_id: {**input_data, **dict(option_inputs.get(option_id, {}))}
      for option_id in selected_option_ids
  }
  ```

  and call with `option_inputs=per_option` and no `input_data`. Merging the kind's
  shared keys into each option's value means every plan command receives everything it
  receives today plus its own collected fields, so nothing that reads `response.json` or
  a command's stdin changes. `input_data` and `option_inputs` are mutually exclusive at
  the executor (`executor_inputs.py:43-48`) — never pass both.

  No built-in plan option declares `inputs:` today, so this branch is inert on landing;
  it exists so `retire-smuggling` can declare inputs on a plan-family gate without
  reopening this path. Say that in the docstring.

## Step 5 — Presentation, traversal, and help

- **Secrets.** Masked at render (`Input(password=True)`). Neither gate modal has an
  input-copy action today; do not add one. `PlanApprovalModal.action_copy_plan` /
  `action_copy_plan_path` copy plan content and the plan path only — confirm by test
  that neither reaches input values.
- **Focus ring.** `GateBranchControls.visible_control_ids()` (`:288-302`) includes each
  branch's visible input control ids, positioned right after that branch's controls.
  `GateActionsMixin.focus_gate_control` (`gate_action_runner.py:202-214`) currently does
  `query_one(f"#{target}", Button)`; widen it to `Widget` so the ring can land on a text
  area or an `Input`. Keyboard-only review must reach every field.
- **Footers.** Add an inputs hint to `custom_gate_modal._footer_text` (`:295-308`) and
  `plan_approval_modal._footer_text` (`:343-370`), rendered only when the gate declares
  at least one input field, so gates without inputs keep their current footer text.
  Mention `^t` for path completion only when a `path`-typed field is present.
- **Styles.** Add `TypedInputForm` rules under the gate section of
  `src/sase/ace/tui/styles.tcss` (near the `GateBranchControls` block at `:1640-1710`):
  `height: auto`, no border, and a muted enum button that reads as a control rather than
  a submit. Do not restyle `.gate-review-section-title`.
- **Help modal.** `src/sase/ace/CLAUDE.md` requires the `?` popup to stay in sync when a
  `sase ace` option changes. Grep `src/sase/ace/tui/modals/help_modal/` for gate-modal
  bindings: today it documents none, and this phase adds no new _global_ key (the input
  controls are modal-scoped and reached by the existing navigate/toggle keys). If that
  is still true when you implement, make no help-modal change and say so in the bead
  close note. If you do add a global binding, honor the 57-character box width and
  32-character description limit.

## Step 6 — Close the conformance gap

In `tests/gate_conformance/_surfaces.py`:

- `_submit_via_ace` passes `option_inputs=submission.option_inputs` into
  `GateSubmission`.
- The `ace` surface gains `CAP_OPTION_INPUTS` (`:206-209`).
- Delete the `("ace", CAP_OPTION_INPUTS)` entry from `PENDING_CAPABILITY_PHASES`
  (`:35-40`).

Every per-option case in `tests/gate_conformance/_cases.py` then runs through ACE
instead of skipping. If any of them fails, that failure is the real deliverable of this
phase — fix the code, not the case.

Note that `_submit_via_ace` drives the tracked-submission worker body, not the modal.
The modal-level guarantee (that the form actually produces the mapping) is covered by
the widget tests below.

## Tests

Run the existing suites **before** each step as the regression guard.

New `tests/ace/tui/test_typed_input_form.py`:

- Every `InputType` validates and converts: `word` rejects whitespace, `int`/`float`
  reject non-numeric text, `bool` accepts the documented spellings, `path` keeps its
  `ctrl+t` binding, `enum` accepts only declared values.
- Enum cycling advances through the declared choices, renders `label` when declared and
  `value` otherwise, and wraps.
- A required enum with no default starts unselected and blocks `is_valid()`.
- A `repeatable` field splits on newlines and converts each line, matching
  `coerce_field_text`.
- A `secret` field renders masked and its value never appears in the widget's rendered
  text.
- Optional reveal shows and hides its blocks and does not affect `is_valid()`.

New `tests/ace/tui/test_gate_branch_inputs.py`:

- A branch whose options declare no inputs renders no Inputs section and no container.
- Toggling an AND member reveals and hides exactly that member's fields.
- A required field blocks both the singleton submit and the group submit until filled,
  and re-enables when filled.
- Two AND members with mutually exclusive `additionalProperties: false` schemas each
  receive **only** their own declared fields in `Resolved.option_inputs`, and a field
  declared by both is collected once and delivered to both.
- Two options declaring one id with different types render the conflict message and
  leave the branch unsubmittable.
- A raw non-empty `input_schema` renders the YAML editor, rejects a value the schema
  refuses, and delivers a valid one under that option's id.
- A raw schema whose only property is in `host_collected_properties` renders nothing.
- A raw schema with no `properties` renders nothing.

Update:

- `tests/ace/tui/test_custom_gate_modal.py` — assert `CustomGateModalResult` carries the
  collected mapping; existing positional assertions must keep passing unchanged.
- `tests/ace/tui/test_notification_custom_gate.py` — the submitted `GateSubmission`
  carries `option_inputs`, and carries `None` when nothing was collected.
- `tests/ace/tui/test_notification_plan_gate.py` (and, if the plan path is covered
  elsewhere, that file) — a plan submission with no collected inputs calls
  `execute_gate_selection` with today's arguments exactly; one with collected inputs
  calls it with `option_inputs` and no `input_data`.
- `tests/ace/tui/test_prompt_input_collection_launch.py` — unchanged assertions must
  still pass; that is the extraction's guard.

Visual (`just test-visual`, goldens under `tests/ace/tui/visual/snapshots/png/`):

- Add one custom-gate snapshot with a rendered Inputs section (a required `line`, an
  `enum`, and a `secret`) to
  `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`, following the
  `_option`/`_actions` fixture style already there.
- `test_ace_png_snapshots_inputs.py` and `test_ace_png_snapshots_prompt_inputs.py`
  goldens **must not move**. If one does, the extraction changed layout — fix the widget
  rather than accepting the diff. Only accept new goldens with
  `--sase-update-visual-snapshots`, and only for the new snapshot.

## Verification

```bash
just install
just check          # every lint gate + the diff-scoped test lane
just test-visual    # PNG snapshots, including the new Inputs golden
```

Run `just check-full` before handing off: this change touches the shared gate modal path
and the plan submission path, which is squarely in the broadening set. Also run the
conformance matrix directly once — `pytest tests/gate_conformance -q` — and confirm the
previously skipped ACE per-option cases now run and pass.

## Risks and edge cases

- **The extraction is the risky half.** `InputCollectionModal` is on the agent-launch
  path. Land Step 1 with the existing xprompt tests and prompt-input visual goldens
  green before writing a line of gate code.
- **Id churn breaks visual goldens.** The `index_offset` / `id_prefix` scheme exists
  purely to keep `#field-input-0`, `#field-block-0`, `#field-error-1`, and
  `#toggle-optional` byte-identical. Verify with a grep before and after.
- **Singleton branches now need submit-state refresh.** `_update_submit_state` returns
  early for `len(branch) <= 1` today. Forgetting to handle singletons means a required
  input silently does not block submission — the exact silent trap this epic closes.
- **`input_data` plus `option_inputs` is a hard error.** Both the custom and plan paths
  must send exactly one. The plan path's merge branch is the only place that could send
  both; test it.
- **`option_inputs_from_values` returns an entry for every selected option**, including
  `{}` for options declaring nothing. That is equivalent to today's behavior at the
  executor (`executor_inputs.py:49-61`), so passing it is safe — but pass `None` when
  nothing was collected anyway, so a gate with no inputs takes the identical code path
  it takes today.
- **Do not let the raw editor regress plan review.** The `host_collected_properties`
  guard is what prevents it. A test asserting the plan modal renders no raw editor for a
  real plan gate envelope is worth more than the guard itself.
- **`secret` in a durable file.** Redaction already lives in the executor
  (`executor_inputs.py:64-95`); this phase must not add a second place a secret value
  can land. Do not log collected values, and do not put them in a toast.

## Deviations from the epic plan's `inputs-ace` section

1. **The reusable widget is driven by `Sequence[TypedFormField]`, not
   `Sequence[InputArg]`.** `InputArg` has no home for a gate field's `label`,
   `placeholder`, or `secret`. `TypedFormField` wraps an `InputArg` — the type rules,
   choices, and required-ness still come from the shared vocabulary through
   `input_arg_for_field`, which is the point of the constraint.
2. **The raw YAML escape hatch renders only for properties no other control on the
   surface collects,** via `host_collected_properties`. The epic said "not the empty
   object renders a single YAML text area", which would put a YAML box on every plan and
   epic gate for `coder_prompt`, `coder_model`, and `epic_launch_mode` — all already
   collected elsewhere, and none of them deliverable through the plan path. That is a
   visible regression on the most-used gate in the system.
3. **The plan path threads `option_inputs` through `execute_plan_approval_response`
   rather than only through `GateSubmission`.** The epic's list of touch points stops at
   `GateSubmission`, but the plan and epic modals do not submit through it — they go
   through `submit_neutral_plan_response` → `execute_plan_approval_response`. Without
   this, "plan and epic gates collect input with no per-kind code" would be false for
   the half that matters.

## Follow-ups to record, not to do here

Record any of these on the bead with `sase bead note sase-h7.6 'PROPOSED FOLLOW-UP: …'`
rather than creating beads:

- If the raw YAML escape hatch turns out to be unreachable in practice (h7.5's
  creation-time answerability check rejects a custom gate with an uncollectable required
  raw property), note that it may be worth deleting rather than maintaining.
- If `host_collected_properties` accumulates a third caller, note that it wants to
  become a declared per-option `collected_by_host` flag rather than a modal-side set.
