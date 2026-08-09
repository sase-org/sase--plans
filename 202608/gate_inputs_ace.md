---
tier: tale
title: Generic typed input collection in the ACE gate modals
goal:
  A reviewer can supply typed, validated input to any gate option from the ACE plan,
  epic, custom, triage, and snooze modals, and those values reach the gate command's
  stdin — with no per-kind code on the ACE side and no behavior change to xprompt input
  collection.
size: large
proposed_by: bbugyi200.athena.sase-h7.6
bead: sase-h7.6
create_time: 2026-08-07 18:41:11
status: wip
---

- **PARENT:**
  [202608/gate_input_collection.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_input_collection.md)
- **BEAD:**
  [sase-h7.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.6.md)

# Plan: Generic typed input collection in the ACE gate modals

Phase `inputs-ace` of epic `sase-h7` (bead `sase-h7.6`). Epic plan:
`plans:202608/gate_input_collection.md`.

## Background

`inputs-core` (bead `sase-h7.3`, commit `8e52e4638`) landed the authoring and transport
layers:

- `GateInputField` and `compile_gate_input_schema` in
  `src/sase/notification_gates/model_inputs.py` — the closed `inputs:` vocabulary,
  compiled into the option's `input_schema` at creation.
- `GateOption.inputs` (`src/sase/notification_gates/model_options.py:79`), re-parsed by
  `GateBranchData.from_envelope` so every ACE reader already sees the declared fields.
- `execute_gate_selection(..., option_inputs=...)`
  (`src/sase/notification_gates/executor.py:59`) plus `resolve_option_inputs` /
  `redact_option_inputs` in `executor_inputs.py` — one submitted JSON value per selected
  option, mutually exclusive with the legacy shared `input_data`.
- `InputType.ENUM` and `InputArg.choices` in `src/sase/xprompt/models.py`, with
  `InputArg.validate_and_convert` accepting only declared choice values.

What is still missing is the **collection layer in ACE**. `GateBranchControls`
(`src/sase/ace/tui/modals/gate_branch_controls.py`) composes exactly one text control,
`#gate-feedback-input` (`:162-163`), and its `Resolved` message carries only
`selected_option_ids` and `feedback` (`:36-46`). `_notification_custom_gate.py:55-63`
sends no input at all. So an option that declares `inputs` today renders nothing, and
its values can never be supplied from the TUI.

The typed-field machinery that should render them already exists, but is welded into
`InputCollectionModal` (`src/sase/ace/tui/modals/input_collection_modal.py`, 566 lines),
which is coupled to `PromptInputPlan` and sits on the far more heavily used agent-launch
path.

Two surfaces consume `GateBranchControls`: `CustomGateModal` (custom, `task_triage`, and
`bead_snooze` gates — every adapter with `generic_form=True`, see
`src/sase/notification_gates/adapters.py:352-379`) and `PlanApprovalModal` (plan and
epic gates). Both must gain generic input collection with no per-kind branching.

## Goals

1. Extract the typed-field form out of `InputCollectionModal` into a reusable widget,
   with the existing xprompt input-collection tests unchanged as the guard.
2. Render each selected option's declared `inputs` generically inside
   `GateBranchControls`, so plan, epic, custom, triage, and snooze gates all collect
   typed input with no per-kind code.
3. Carry the collected values through `CustomGateModalResult` / `PlanApprovalResult` and
   `GateSubmission` into `execute_gate_selection(option_inputs=...)`, so what the
   reviewer typed actually reaches the command's stdin.
4. Block submission while a required field is empty or invalid, with the same rule the
   feedback field already uses.
5. Add an `enum` editor and a raw-`input_schema` YAML escape hatch, so no declared gate
   is unanswerable from ACE.

## Non-goals

- **No behavior change for xprompt input collection.** The extraction is a pure
  refactor; `tests/ace/tui/modals/test_input_collection_modal.py`,
  `tests/ace/tui/test_prompt_input_collection_launch.py`, and
  `tests/ace/tui/visual/test_ace_png_snapshots_prompt_inputs.py` / `..._inputs.py` must
  pass with no assertion edits (import-path edits only).
- **No changes under `src/sase/notification_gates/`.** The executor contract landed in
  `inputs-core` and is consumed as-is. The one exception is documentation-only wording
  if a docstring is found to be wrong; if a real gap appears, record a
  `PROPOSED FOLLOW-UP:` note rather than widening the phase.
- **The `question` gate keeps `user_question_modal.py`.** It does not use
  `GateBranchControls` and is untouched.
- **No general JSON-Schema-to-form renderer.** The closed vocabulary plus one raw-YAML
  box is the contract.
- **No gate actions / Actions section.** That is `gate-actions-ace` (`sase-h7.7`).
- **No docs rewrite.** `docs/notifications.md` and the `/sase_gate` skill are the `docs`
  phase (`sase-h7.12`).
- **No snooze/triage schema changes.** Declaring their durations as `inputs` is
  `retire-smuggling` (`sase-h7.11`), which this phase unblocks.

## Design

### 1. `TypedInputForm` — the extracted widget

New `src/sase/ace/tui/widgets/typed_input_form.py`.

The widget is driven by a small presentation record rather than a bare `InputArg`,
because both callers need display metadata (`label`, `help`, `placeholder`) and the gate
caller needs `secret` and `repeatable` — none of which belong in the xprompt vocabulary.
`InputArg` remains the single **validation** contract: every field converts through
`InputArg.validate_and_convert`.

```python
@dataclass(frozen=True)
class TypedInputField:
    """One typed form field: its validation contract plus its presentation."""

    arg: InputArg                 # name doubles as the value key
    label: str
    required: bool
    help: str | None = None
    placeholder: str | None = None
    secret: bool = False
    repeatable: bool = False      # multi-line editor, one value per line

    @classmethod
    def from_input_arg(cls, arg: InputArg, *, required: bool) -> TypedInputField: ...
```

`from_input_arg` is the xprompt projection: `label=arg.name`, `help=arg.description`,
`placeholder` from `arg.default`, `secret=False`, `repeatable=False` — so the launch
path renders exactly what it renders today.

```python
class TypedInputForm(Vertical):
    """Reusable typed-field form driven by a sequence of TypedInputField."""

    class Changed(Message):
        """A field value changed; the host should re-check its submit state."""

    def __init__(
        self,
        fields: Sequence[TypedInputField],
        *,
        id_prefix: str = "field",
        index_offset: int = 0,
        collapse_optional: bool = True,
        id: str | None = None,
        classes: str | None = None,
    ) -> None: ...
```

Moved verbatim (renamed only where noted) from `input_collection_modal.py`:

- `_InputCollectionInput` → `_TypedFormInput` (single-line vim editor,
  `soft_wrap=True`).
- `_PathField` with its `ctrl+t` completion cycling, unchanged.
- `_build_field_block` / `_build_input` / `_field_header` / `_field_guidance`,
  `_load_type_rules`, the optional-reveal toggle and its label, per-field validation
  against `InputArg.validate_and_convert` with the per-type rule as the error text, and
  `_scroll_editor_cursor_visible`'s cursor-reveal behavior (the form posts `Changed`;
  the host owns the scroll container it belongs to, so the reveal stays where the scroll
  container is — see step 2.6).

Widget ids are `{id_prefix}-block-{i}`, `{id_prefix}-input-{i}`, `{id_prefix}-error-{i}`
where `i = index_offset + position`, and `{id_prefix}-toggle-optional` for the reveal
button, with `id_prefix="field"` producing exactly today's ids — including the bare
`toggle-optional` id, which stays a special case for the default prefix so the existing
xprompt tests and PNG snapshots do not move.

Public API used by both hosts:

| Method                                    | Purpose                                              |
| ----------------------------------------- | ---------------------------------------------------- |
| `field_indices() -> range`                | The flat indices this form owns                      |
| `field_at(index) -> TypedInputField`      | Lookup by flat index                                 |
| `text_value(index) -> str`                | Raw editor text                                      |
| `is_field_valid(index) -> bool`           | Empty-optional is valid; empty-required is not       |
| `is_field_visible(index) -> bool`         | Hidden by the optional reveal or by the host         |
| `focus_field(index) -> None`              | Focus and prime the vim mode display                 |
| `all_valid() -> bool`                     | Every visible field valid                            |
| `required_filled() -> tuple[int, int]`    | `(filled, total)` over visible required fields       |
| `text_values() -> dict[str, str]`         | Non-empty values keyed by `arg.name`                 |
| `converted_values() -> dict[str, Any]`    | `validate_and_convert` applied; invalid fields raise |
| `set_field_visible(name, visible)`        | Host-driven reveal (gate option toggles)             |
| `first_invalid_required() -> str \| None` | Label of the first blocking field, for the notify    |

New editors:

- **`enum`** — `_EnumField(Static)`, `can_focus = True`, rendered as
  `◀ <label or value> ▶` with a `n of m` suffix. `space`/`enter`/`right`/`l` advance,
  `left`/`h` go back; posts `TypedInputForm.Changed`. `text_value` returns the current
  `InputChoice.value`, so validation and conversion stay on the `InputArg.ENUM` path.
- **`secret: true`** — Textual `Input(password=True)`. A `TextArea` cannot mask, and
  this is the one place a different base widget is warranted. Value reading goes through
  a single `_editor_text(widget)` dispatch so the rest of the form is agnostic.
- **`repeatable: true`** — a multi-line `VimTextArea`; one item per non-blank line, each
  converted individually by `validate_and_convert`. `converted_values()` returns a list.
  Never set on the xprompt path, so launch behavior is unchanged.

`converted_values()` raises `XPromptValidationError` only when a field is invalid, which
callers avoid by checking `all_valid()` first; this keeps one conversion path rather
than duplicating type rules.

### 2. `GateBranchInputs` — per-branch gate input collection

New `src/sase/ace/tui/modals/gate_branch_inputs.py`. Keeping this out of
`gate_branch_controls.py` (already 421 lines) matches the repo's module-splitting habit.

```python
class GateBranchInputs(Vertical):
    """Declared and raw input collection for one gate branch."""

    class Changed(Message): ...

    def __init__(self, options: Sequence[GateOption], *, branch_index: int) -> None: ...
```

**2.1 Field projection.** `gate_form_fields(options) -> tuple[TypedInputField, ...]`
maps each option's `GateInputField` to a `TypedInputField`:

```python
InputArg(
    name=field.id,
    type=field.type,
    default=UNSET if field.required else field.default,
    description=field.help,
    choices=field.choices,
)
```

with `label=field.label`, `required=field.required`, `placeholder=field.placeholder`,
`secret=field.secret`, `repeatable=field.repeatable`.

Fields are deduped by id across the branch's options, keeping the first declaration in
branch order, so a field shared by two AND members is collected once and delivered to
both (the epic plan's requirement). Two options declaring the same id with different
types is an authoring error this phase does not police — the executor validates per
option and records a diagnosable `schema_validation_failed` under `errors/`. Record a
`PROPOSED FOLLOW-UP:` note proposing that `custom-validation` reject it at creation.

**2.2 The `feedback` field is never a form field.** A declared field whose id is
`feedback` is skipped by the projection: the shared feedback control is the one place a
note is typed, and `apply_feedback_input` (`feedback_input.py:43`) overwrites that key
on the way to the command anyway, so rendering both would silently discard what the
reviewer typed in the form. Instead `GateBranchInputs.feedback_floor(selected_ids)`
returns `"required"` when a selected option declares a required `feedback` field,
`"optional"` when it declares an optional one, and `"disabled"` otherwise;
`GateBranchControls` folds that into its existing `_feedback_mode` maximum. This is what
lets `retire-smuggling` declare `feedback` as an ordinary input field without a special
case here.

**2.3 Raw `input_schema` escape hatch.** An option that declares no `inputs` but carries
a raw `input_schema` that is _substantive_ gets one YAML editor. "Substantive" means the
schema declares at least one property other than `feedback`. Excluding a feedback-only
schema keeps `plan_gate.py`'s feedback option (`:563-569`) and any custom gate that only
wants a note rendering nothing extra.

The editor is a multi-line `VimTextArea` seeded with `yaml_dumps` of the schema's
declared per-property `default` values (empty string when there are none), labelled with
the option's label. Its text is parsed with `yaml_loads`
(`src/sase/ace/tui/modals/config_edit_helpers.py:123,135`) and validated on change with
`validate_json_instance(value, option.input_schema, option.id)` from
`src/sase/notification_gates/command_runner.py:215` — the exact enforcement the executor
will apply, so the modal never accepts something submission would reject. A parse or
validation failure shows the message in an error label and blocks the branch's submit.

`PlanApprovalModal` passes `raw_input_forms=False` to `GateBranchControls`. This is the
one intentional per-modal switch, and the reason is concrete rather than kind-shaped:
the tale plan gate's `approve`/`commit` options declare `coder_prompt` and `coder_model`
(`src/sase/plan_gate.py:571-577`), which the modal already collects through its
dedicated Coder options modal on `c`, and the epic gate's `epic_launch_mode` is chosen
by the host, not the reviewer. Rendering a YAML box for them would duplicate an existing
control and offer a second, worse way to set the same values. Declared `inputs` still
render in the plan modal, generically — the switch turns off only the raw fallback. Note
the switch in `GateBranchControls`' docstring with that reason.

**2.4 Visibility.** `set_selected(selected_ids)` shows exactly the declared fields and
raw editors belonging to currently selected options, and hides the rest, so toggling an
AND member reveals and hides its inputs.

**2.5 Values.** `option_inputs(selected_ids) -> dict[str, dict[str, Any]]` builds one
JSON object per selected option: every declared field the option declares, taken from
the shared form value, plus the parsed YAML object for a raw option. Fields left empty
are omitted, so a declared default in the compiled schema is not overwritten by `""`.
Options that declare nothing get no entry. When the whole mapping is empty the host
submits `option_inputs=None`, which keeps the executor on its existing shared-value path
and leaves today's gates byte-identical.

**2.6 Blocking.** `blocking_field(selected_ids) -> str | None` returns the label of the
first visible required field that is empty or invalid, or the first raw editor whose
YAML does not validate.

### 3. `GateBranchControls` changes

`src/sase/ace/tui/modals/gate_branch_controls.py`:

1. `__init__` gains `raw_input_forms: bool = True`.
2. `compose` yields, after all branch controls and before the feedback label:
   `Static("Inputs", id="gate-inputs-label", classes="gate-inputs-header hidden")` and
   one `GateBranchInputs` per branch that has any input at all, id
   `gate-inputs-{branch_index}`, hidden unless that branch is active. One shared slot
   that follows the active branch is exactly how the feedback control already behaves
   (`_update_feedback_controls`, `:367-390`), and it avoids putting a form inside the
   `gate-singleton-row` `Horizontal`.
3. `_update_input_controls()` — called from `on_mount`, `_set_active_branch`,
   `toggle_option`, `apply_selection`, and on `GateBranchInputs.Changed` — displays the
   active branch's form, calls `set_selected`, toggles `#gate-inputs-label`, and
   refreshes the submit state.
4. `_feedback_mode(branch_index)` also takes `GateBranchInputs.feedback_floor` into
   account (step 2.2).
5. `_update_submit_state(branch_index)` additionally disables the group submit when
   `blocking_field` is non-`None`. Singleton branches have no separate submit control,
   so `_resolve_branch` refuses with `"<label> is required before submitting"` and
   focuses the offending field — mirroring today's feedback-required refusal
   (`:340-343`).
6. `_resolve_branch` posts `Resolved(selected, feedback, option_inputs)` where
   `option_inputs` is `{}` when nothing was collected.
7. `_visible_control_ids` includes the active branch's visible field ids after that
   branch's control ids, so `j`/`k` reach them. `_focus_relative` and its `query_one`
   are widened from `Button` to `Widget`. The feedback `Input` is deliberately **not**
   added: it is not in the cycle today and adding it is a separate behavior change.
8. `on_button_pressed` stops only ids it actually handles (`gate-singleton-`,
   `gate-group-expand-`, `gate-group-submit-`, `gate-option-`) instead of every id
   starting with `gate-`, so a button inside a mounted form is never swallowed.
9. Enter in the last visible input field focuses the feedback field when it is shown,
   and otherwise resolves the active branch — mirroring `on_input_submitted`
   (`:197-199`).

`Resolved.__init__` gains a third parameter
`option_inputs: Mapping[str, dict[str, Any]] = {}` (an immutable default via
`MappingProxyType`), so existing constructions keep working.

### 4. Threading the values to the executor

- `CustomGateModalResult` gains `option_inputs: Mapping[str, dict[str, Any]]` (default
  empty), populated in `CustomGateModal.on_gate_branch_controls_resolved`.
- `GateSubmission` (`_notification_gate_execution.py:19`) gains
  `option_inputs: Mapping[str, object] | None = None`, forwarded to
  `execute_gate_selection(..., option_inputs=submission.option_inputs)`.
- `_notification_custom_gate.py:55-63` passes
  `option_inputs=result.option_inputs or None` and its comment is updated: the executor
  still injects the note, and now the reviewer's typed fields ride along.
- `PlanApprovalResult` gains the same field, set in
  `PlanApprovalModal.on_gate_branch_controls_resolved` and carried through
  `_result_for_selection`.
- `submit_neutral_plan_response` (`_notification_plan_gate.py:85`) forwards
  `option_inputs=result.option_inputs` to `execute_plan_approval_response`, which gains
  the keyword and passes it to `_execute_neutral_plan_approval_response`
  (`src/sase/plan_approval_actions.py:69,190`). The legacy branch ignores it.
- In `_execute_neutral_plan_approval_response`, extract the existing shared-input
  assembly into a pure helper:

  ```python
  def plan_option_inputs(
      selected_option_ids: tuple[str, ...],
      shared_input: Mapping[str, Any],
      collected: Mapping[str, Mapping[str, Any]],
  ) -> dict[str, dict[str, Any]]:
      """Per-option plan input: the reviewer's declared values, host fields last."""
      return {
          option_id: {**collected.get(option_id, {}), **shared_input}
          for option_id in selected_option_ids
      }
  ```

  When `collected` is empty the call keeps `input_data=` exactly as today (so
  `response["input"]` and `tests/test_plan_gates_action_api.py` are unaffected);
  otherwise it submits `option_inputs=` and `input_data=None`. The host-computed fields
  win over a reviewer value of the same name, because `coder_prompt`, `coder_model`, and
  `epic_launch_mode` are protocol fields the modal's own controls own.

  Rendering a field and then discarding it would be the worst outcome, so this plumbing
  lands now even though no built-in plan option declares `inputs` yet.

### 5. Presentation

- Section header `Inputs`, styled subordinate to the `Decision` section title so the
  decision stays the focal point: `$text-muted`, bold, no accent colour.
- `styles.tcss`: generalize the `InputCollectionModal .field-header` /`.field-desc`
  /`.field-error` /`.input-field-block` /`.input-section-header`
  /`SingleLineVimTextArea` rules to `TypedInputForm ...` selectors (the modal's own
  container/title/status rules stay `InputCollectionModal`-scoped), and add
  `GateBranchControls #gate-inputs-label`, `GateBranchControls GateBranchInputs`, and a
  `.gate-raw-input-*` block. Verify the xprompt PNG snapshots still match byte-for-byte;
  they are the guard that the selector move did not change the launch modal's look.
- Secret fields render masked and no copy action reads them; the modals' `y`/`Y` copy
  actions read the plan file only, and none is added for form values. State this in the
  widget docstring.
- Footers: `custom_gate_modal._footer_text` (`:269-280`) and
  `plan_approval_modal._footer_text` (`:327-355`) append a `<tab> fields` hint, rendered
  only when the gate declares any input. No new modal key is introduced, so the ACE help
  modal (`help_modal/`, which documents no gate-modal keys today) needs no change; the
  57-character box rules in `src/sase/ace/CLAUDE.md` therefore do not come into play.

## Files

| File                                                              | Change                                                                                           |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `src/sase/ace/tui/widgets/typed_input_form.py`                    | new (imported by module path, like `single_line_vim_text_area`; no `widgets/__init__.py` export) |
| `src/sase/ace/tui/modals/input_collection_modal.py`               | refactor onto the widget                                                                         |
| `src/sase/ace/tui/modals/gate_branch_inputs.py`                   | new                                                                                              |
| `src/sase/ace/tui/modals/gate_branch_controls.py`                 | render + resolve inputs                                                                          |
| `src/sase/ace/tui/modals/custom_gate_modal.py`                    | result field + footer                                                                            |
| `src/sase/ace/tui/modals/plan_approval_modal.py`                  | result field + footer + `raw_input_forms=False`                                                  |
| `src/sase/ace/tui/modals/__init__.py`                             | exports                                                                                          |
| `src/sase/ace/tui/actions/agents/_notification_gate_execution.py` | `GateSubmission.option_inputs`                                                                   |
| `src/sase/ace/tui/actions/agents/_notification_custom_gate.py`    | pass `option_inputs`                                                                             |
| `src/sase/ace/tui/actions/agents/_notification_plan_gate.py`      | pass `option_inputs`                                                                             |
| `src/sase/plan_approval_actions.py`                               | `option_inputs` + `plan_option_inputs`                                                           |
| `src/sase/ace/tui/styles.tcss`                                    | selectors + new blocks                                                                           |

## Implementation order

1. Extract `TypedInputForm` and rewire `InputCollectionModal`. Run
   `just test-scoped`-equivalent selection for the xprompt input tests plus
   `just test-visual -k inputs` before touching anything gate-shaped. Update only the
   `_PathField` import path in `tests/ace/tui/modals/test_input_collection_modal.py`.
2. Add the `enum`, secret, and repeatable editors to the widget, with widget-level
   tests.
3. Add `GateBranchInputs` and its pure projection helpers, with unit tests that need no
   Textual app where possible.
4. Wire `GateBranchControls`: rendering, reveal, blocking, `Resolved.option_inputs`.
5. Thread the values through both modals into the executor.
6. Styles, footers, PNG snapshot.

## Testing

New `tests/ace/tui/test_typed_input_form.py`:

- reveal-on-optional-toggle and required-blocks-`all_valid`
- `enum` cycling honours declared labels and wraps at both ends
- a secret field renders masked (`Input.password is True`) and still yields its value
- a repeatable field yields one converted item per non-blank line
- `converted_values()` returns `int`/`float`/`bool` for those types, not strings

New `tests/ace/tui/test_gate_branch_inputs.py` (Textual harness, modelled on
`tests/ace/tui/test_custom_gate_modal.py`):

- an option declaring `inputs` renders the `Inputs` section; an option declaring none
  renders nothing and `Resolved.option_inputs == {}`
- toggling an AND member reveals and hides its declared fields
- a field declared by two selected AND members is rendered once and delivered to both
- two selected AND members with divergent fields receive divergent values
- the group submit is disabled while a required field is empty, and a singleton branch
  refuses with a notify naming the field
- a declared `feedback` field is not rendered and raises the branch's feedback mode
- a raw non-empty `input_schema` renders the YAML box seeded with its defaults, blocks
  on invalid YAML, and delivers the parsed object; a feedback-only raw schema renders
  nothing
- `PlanApprovalModal` renders declared `inputs` but no raw YAML box

Extended `tests/ace/tui/test_notification_custom_gate.py`:

- an end-to-end submission of a bundle whose option declares `inputs` reaches the
  command's stdin and lands in `response.json`'s `option_inputs` — the acceptance test
  for the whole phase
- a gate with no declared inputs still submits `option_inputs=None`, leaving
  `response["input"]` as it is today

Extended `tests/test_plan_approval_actions.py`:

- `plan_option_inputs` merges collected values under the host protocol fields
- an empty `collected` leaves `execute_plan_approval_response` on the shared-input path

PNG snapshot in `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`:
`custom_gate_inputs_120x40`, an AND branch whose members declare a `line`, an `enum`,
and an optional `int`, with the golden generated via `--sase-update-visual-snapshots`
under `tests/ace/tui/visual/snapshots/png/`.

Run `just install`, then `just check`. This change touches the shared gate modal path
and the agent-launch input modal, so finish with `just check-full` plus
`just test-visual`.

## Risks

- **Breaking prompt launching.** `InputCollectionModal` is on the agent-launch path,
  which is used far more than gates. Mitigated by doing the extraction first, alone,
  with the existing tests and PNG goldens unchanged as the acceptance gate.
- **CSS selector move.** Rewriting `InputCollectionModal .field-*` as
  `TypedInputForm .field-*` is the likeliest source of a silent visual regression; the
  xprompt PNG snapshots are the check, and they must not be re-baselined in this phase.
- **Modal key capture.** `j`/`k`/`space`/`enter` are modal-scoped gate bindings.
  `SingleLineVimTextArea` consumes them (vim navigation, and `enter` posts `Submitted`),
  and `_EnumField` binds them itself, so focus does not jump while typing. Assert this
  in a widget test rather than trusting it.
- **`option_inputs` and `input_data` are mutually exclusive.** Submitting an empty
  `option_inputs` mapping would silently zero `response["input"]` for gates that rely on
  the shared value. Both the custom-gate and plan paths therefore submit `None` when
  nothing was collected, and a test pins that.
- **Textual `Input(password=True)` inside a `VerticalScroll`** behaves differently from
  the vim text areas around it (no vim modes). Acceptable: it is the only widget that
  can mask, secrets are rare, and the alternative is an unmasked durable-audit hazard.
