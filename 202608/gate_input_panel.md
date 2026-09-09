---
status: done
tier: epic
title: Collect gate inputs in a dedicated panel instead of the gate modal's left pane
goal: "Selecting a gate option opens a wide, dedicated input panel that shows exactly
  that selection's input components under their owning option, navigable with <tab> and
  <shift+tab>, with full vim insert/normal editing in every freeform box; the gate
  modal's left pane shows only decisions.

  "
phases:
  - id: editors
    title: Vim editing for every typed freeform field
    depends_on: []
    size: medium
    description: "editors: give the shared TypedInputForm a masked single-line vim
      editor for secret fields and a multi-line vim editor for text-typed and repeatable
      fields, and route each editor's vim mode display to a host screen hook.

      "
  - id: panel
    title: The GateInputPanel modal and its pure request model
    depends_on:
      - editors
    size: medium
    description: "panel: add the pure per-option input request/collection model and the
      GateInputPanel modal screen that renders it, with a tab/shift+tab focus ring, live
      validity, drafts, and a submit that returns per-option values.

      "
  - id: wire
    title: Route every gate submission through the panel
    depends_on:
      - panel
    size: medium
    description: "wire: make GateBranchControls open the panel when a selection needs
      typed input, delete the inline Inputs section and the inline feedback box, and
      simplify the focus ring and submit-state bookkeeping in both gate modals.

      "
  - id: keys
    title: Configurable panel keymaps and modal footers
    depends_on:
      - wire
    size: small
    description: "keys: add open_inputs, next_input, and previous_input to the gate
      keymap scope, thread the effective keys into the panel and both gate-modal
      footers, and keep declared gate-action keys from stealing them.

      "
  - id: chrome
    title: Panel styling, option input badges, and visual snapshots
    depends_on:
      - keys
    size: medium
    description: "chrome: style the panel so field text does not wrap, badge every
      option that declares inputs, drop the dead inline-input styles, and rebaseline
      plus extend the gate PNG goldens.

      "
  - id: docs
    title: Document the panel and its keys
    depends_on:
      - keys
    size: small
    description:
      "docs: update the notifications gate-review and gate-inputs prose, the gate keymap
      tables, and the gate remapping example to describe panel-based collection."
proposed_by: bbugyi200.athena.06q
bead_id: sase-q3
create_time: 2026-09-09 19:50:31
---

- **PROMPT:**
  [prompts/202608/gate_input_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_input_panel.md)
- **BEAD:**
  [sase-q3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-q3/README.md)

# Plan: Collect gate inputs in a dedicated panel

## Problem

Every gate-review modal (`CustomGateModal`, `PlanApprovalModal`) renders its decision
controls in a left column that `styles.tcss` pins to `width: 42`, and renders each
branch's declared inputs _inside that column_, below the branch's buttons. Three things
go wrong:

- **No room.** Field headers, per-type guidance, and values are truncated at ~38 usable
  cells while the document pane beside them is often mostly empty. The
  `custom_gate_inputs_120x45` golden shows `(line · requir…` and
  `One of the values declared under `ch…`.
- **No attribution.** `compose_singleton_row` renders a run of adjacent single-option
  branches as one row of buttons and _then_ yields each branch's
  `GateBranchInputSection` below it, so a stack of unlabeled fields sits under a row of
  several options with nothing tying a field to the option whose command consumes it.
  `GateBranchInputSection` compounds this by merging every option of a branch into one
  `TypedInputForm` via `collected_input_fields`.
- **Freeform boxes are second-class.** The reviewer's note (`#gate-feedback-input`) is a
  plain Textual `Input`, the raw-schema editors are plain `TextArea`s, and
  `TypedInputForm` renders `secret` fields as `Input(password=True)` — none of them get
  the vim insert/normal layer that every other SASE text box has. A `repeatable` field
  is worse than second-class: `TypedInputForm._convert` splits its raw text on newlines,
  but the editor is a `SingleLineVimTextArea` that flattens newlines, so a second value
  can never be entered.

## Desired behavior

Answering a gate is two beats: **choose**, then **fill in**. The left pane owns the
first beat only.

- The gate modal's Decision column shows branches, options, and group toggles — and
  nothing else. No `Inputs` section, no feedback box.
- Selecting an option that needs typed input opens `GateInputPanel`, a modal panel
  showing every input component that selection needs, grouped under a section header
  naming the owning option (icon + label). Confirming the panel submits the branch;
  cancelling returns to the gate with the selection and everything typed intact.
- The panel is wide (90% of the terminal, capped at 120 cells) so a realistic value
  never needs to wrap, and it scrolls when a selection declares many fields.
- `<tab>` / `<shift+tab>` walk the panel's input components as one wrapping ring.
- Every freeform box in the panel — declared fields, raw-schema YAML editors, and the
  reviewer's note — is a vim editor with the shared insert/normal modes, readline keys,
  and `<ctrl+t>` path completion where the type calls for it.

### When the panel opens

One rule, applied to the selection being submitted:

| Selection needs                                           | Result                    |
| --------------------------------------------------------- | ------------------------- |
| any declared `inputs` field (required _or_ optional)      | panel opens, then submits |
| a raw `input_schema` with any non-host-collected property | panel opens, then submits |
| `feedback: required`                                      | panel opens, then submits |
| only `feedback: optional`, or nothing at all              | submits immediately       |

The last row keeps the common case at one keystroke: an option whose note is genuinely
optional still answers on `<enter>` or its digit. To attach that optional note, press
`open_inputs` (`i`) on the option first — that opens the same panel for the same
selection. **Confirming the panel always submits**; the panel is a step in the submit
path, never a separate "save my values" mode. Cancel never submits.

For an AND group, the panel opens when the group's submit control is activated, not when
a member is toggled. Toggling members is exploratory — popping a modal on each toggle
would fight the reviewer, and a cancelled toggle would leave a half-selected state. The
panel then shows one section per selected option that declares input, so attribution is
explicit without hijacking the toggles. This is the one place where "when an option is
selected" is read as "when the selection is committed"; every OR option (the dominant
shape, including every plan gate and every `sase gate create` gate seen so far) opens
its panel the moment it is chosen.

### Panel anatomy

```
┌─ Approve production deployment ───────────── custom gate · deploy-production-42 ──┐
│ 🚀 Deploy signed build                                                            │
│                                                                                   │
│ ── 🚀 Deploy signed build ────────────────────────────── 3 inputs · 1 required ── │
│ Target environment                                              line · required   │
│ A single line of text with no line breaks                                         │
│ ┌───────────────────────────────────────────────────────────────────────────────┐ │
│ │ production                                                                    │ │
│ └───────────────────────────────────────────────────────────────────────────────┘ │
│ Mode                                                              enum · optional │
│ [ Full restart (clears cache) ]                                                   │
│ ...                                                                               │
│ ── Note ──────────────────────────────────────────────────────────── optional ─── │
│ ┌───────────────────────────────────────────────────────────────────────────────┐ │
│ │                                                                               │ │
│ └───────────────────────────────────────────────────────────────────────────────┘ │
│                                                              1 of 1 required filled│
│ [ Deploy signed build ]  [ Cancel ]                                               │
├───────────────────────────────────────────────────────────────────────────────────┤
│ <tab>/<shift+tab> field   ^s submit   <esc> back   ^t complete path      INSERT    │
└───────────────────────────────────────────────────────────────────────────────────┘
```

- Border title is the gate's headline; border subtitle is `<kind> gate · <request id>`.
- One section rule per option, carrying the option's icon and label on the left and its
  field count on the right. A single-option selection still gets its rule — that line is
  the attribution the current layout lacks.
- The submit button is labelled with the branch label and uses the `success` variant, so
  it is unmistakable that confirming answers the gate.
- The footer's right end is a live vim mode chip fed by the focused editor.

### Keys inside the panel

| Key                     | Action                                                           |
| ----------------------- | ---------------------------------------------------------------- |
| `<tab>` / `<shift+tab>` | Next / previous input component (wraps; includes the buttons)    |
| `<enter>`               | Single-line field: next field; past the last: submit if valid    |
| `<ctrl+s>`              | Submit from anywhere                                             |
| `<esc>`                 | INSERT → NORMAL; from NORMAL, close the panel without submitting |
| `<ctrl+t>`              | Cycle filesystem completions in a `path` field                   |

Multi-line (`text`, `repeatable`) fields keep `<enter>` for newlines; `<tab>` and
`<ctrl+s>` are how you leave them.

## Design decisions

- **The panel is presentation only.** Which fields a selection collects, how text
  becomes a typed JSON value, and which option gets which value stay in
  `sase/notification_gates/input_collection.py` (`collected_input_fields`,
  `input_arg_for_field`, `option_inputs_from_values`), which Telegram, the CLI, and the
  mobile bridge already share. The panel calls them; it does not restate them. Nothing
  in this epic changes the wire contract, so no `sase-core` change is required and
  `GateBranchControls.Resolved(selected_option_ids, feedback, option_inputs)` keeps its
  exact shape — the executor and both host modals stay untouched by the switch-over.
- **Shared field ids stay collected once.** When two selected AND options declare the
  same field id compatibly, the field renders once, in the first declaring option's
  section, annotated `· also sent to <other option label>`; `option_inputs_from_values`
  still distributes the value to both. An incompatible duplicate keeps today's
  `conflicting_input_field` behavior: the panel shows the message and refuses to submit.
- **An option that declares its own `feedback` field wins.** If any selected option
  declares an input with id `feedback`, that field renders in its option's section and
  the panel's Note section is suppressed; its value is also returned as the result's
  `feedback` so `response.json` records the note exactly as it does today.
- **`GateBranchControls` owns the panel round trip**, not the two host modals. Both
  modals already delegate everything about branches to it, and duplicating the push /
  callback / draft flow in `CustomGateModal` and `PlanApprovalModal` would be the exact
  drift this widget exists to prevent.
- **No feature flag.** The flag rule covers user-reaching behavior that lands _before it
  is ready_; this epic is phased so that never happens. `editors` improves the shared
  form on its own terms, `panel` adds a screen nothing can reach yet, and `wire` is the
  atomic switch-over: it deletes the inline path and lights up the panel in one change.
  If a phase agent finds it cannot land `wire` atomically, that is the signal to stop
  and add a `wip` flag with `sase flag new` rather than to ship a half state.

## Relevant code map

Gate modals and branch controls:

- `src/sase/ace/tui/modals/custom_gate_modal.py` — two-pane shell, `_compose_actions`,
  `_footer_text`, bindings, `focus_gate_control` usage.
- `src/sase/ace/tui/modals/plan_approval_modal.py` — same shell for plans, plus
  `plan_approval_gate_data.py` (`HOST_COLLECTED_PROPERTIES`, static bindings),
  `plan_approval_footer.py`, `plan_approval_decisions.py`.
- `src/sase/ace/tui/modals/gate_branch_controls.py` — selection state, feedback
  bookkeeping, `_resolve_branch`, `visible_control_ids`, `_update_submit_state`.
- `src/sase/ace/tui/modals/gate_branch_input_section.py` — today's inline Inputs section
  (**deleted in `wire`**); owns `gate_declares_inputs` and the raw-schema YAML editors.
- `src/sase/ace/tui/modals/gate_branch_layout.py` — option/group/singleton buttons and
  `toggle_label`.
- `src/sase/ace/tui/modals/gate_action_runner.py` — `focus_gate_control`,
  `gate_modal_taken_keys`, `_gate_control_ids`.

Shared input widgets:

- `src/sase/ace/tui/widgets/typed_input_form.py` — `TypedFormField`, `TypedInputForm`,
  `_InputCollectionInput`, `_PathField`, `_EnumField`.
- `src/sase/ace/tui/widgets/single_line_vim_text_area.py`,
  `src/sase/ace/tui/widgets/vim_text_area.py` — the shared vim layer;
  `_update_vim_mode_display` is the documented host hook.
- `src/sase/ace/tui/modals/axe_entry_editor_rendering.py` — the reference for routing a
  focused editor's vim mode to a host screen (`_route_vim_mode`).
- `src/sase/ace/tui/modals/input_collection_modal.py` — the other `TypedInputForm` host;
  every `editors` change must keep it working.

Gate domain (read, do not restate):

- `src/sase/notification_gates/input_collection.py`,
  `src/sase/notification_gates/model_inputs.py`,
  `src/sase/notification_gates/model_options.py`.

Keymaps, styles, tests, docs:

- `src/sase/ace/tui/keymaps/app_keymaps.py` (`GateModalKeymaps`), `keymaps/metadata.py`
  (`_GATE_BINDING_META`), `keymaps/bindings.py`, `src/sase/default_config.yml`
  (`ace.keymaps.gate`).
- `src/sase/ace/tui/styles.tcss` — `.gate-review-*`, `GateBranchControls …` blocks
  (roughly lines 1640–1900), `.input-field-block` / `.field-*` (roughly 3080–3130).
- `src/sase/ace/tui/modals/__init__.py` + `__init__.pyi` — lazy modal exports.
- `tests/ace/tui/test_gate_branch_inputs.py`, `test_custom_gate_modal.py`,
  `test_typed_input_form.py`, `test_notification_plan_gate.py`,
  `test_gate_primary_specialized_modals.py`,
  `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`.
- `docs/notifications.md` (gate review + `Gate inputs`), `docs/configuration.md`
  (`ace.keymaps` gate table), `docs/ace.md` (`Remapping Gate Modal Keys`).

## Out of scope

- The gate wire format, `sase gate answer`, the executor, Telegram, and the mobile
  bridge. This epic changes one surface's collection UI.
- The read-only gate detail pane in the notification modal
  (`notification_modal_gate.py`) keeps listing declared fields per option; only the
  `chrome` phase touches it, and only to match the new `✎` badge wording.
- Reordering, renaming, or adding declared input types.

---

## Phase `editors`: Vim editing for every typed freeform field

### Goal

`TypedInputForm` renders every freeform field as a vim editor, so both its hosts (the
prompt `InputCollectionModal` today, `GateInputPanel` from the next phase) inherit
insert/normal modes without per-host branching. Nothing about gate layout changes here.

### New widget: `SecretVimTextArea`

Add `src/sase/ace/tui/widgets/secret_vim_text_area.py` — a `SingleLineVimTextArea`
subclass that masks its rendered text:

- Override `render_line(y)`: call `super().render_line(y)` and rebuild the returned
  `Strip` with each `Segment`'s text replaced by `"•" * cell_len(segment.text)`,
  preserving the segment's style and the strip's `cell_length`. Building the mask from
  `rich.cells.cell_len` (not `len`) is what keeps a wide character from shifting the
  line, so the cursor never drifts off its cell.
- Skip masking entirely when `self.text == ""`, so the placeholder still reads as
  placeholder text rather than a row of bullets.
- `.text` / `.value` keep returning the real value; only rendering is masked.

Document in the class docstring that the vim registers and the app clipboard hold the
real characters — masking is shoulder-surfing protection, matching what
`Input(password=True)` gave before.

### New editor: multi-line typed fields

In `typed_input_form.py`, add a private `_MultilineInput(VimTextArea)` with
`soft_wrap=True` and `show_line_numbers=False`. `VimTextArea` deliberately leaves Enter
to `TextArea`, so `<enter>` inserts a newline and `<tab>` (Textual's default
`tab_behavior="focus"`) leaves the field.

Route it from `_build_editor`, in this precedence order:

1. `arg.type is InputType.ENUM` → `_EnumField` (unchanged)
2. `field.secret` → `SecretVimTextArea` (a secret stays single-line and masked even when
   its type is `text`; say so in a comment)
3. `arg.type is InputType.PATH` → `_PathField` (unchanged)
4. `arg.type is InputType.TEXT or arg.repeatable` → `_MultilineInput`
5. otherwise → `_InputCollectionInput` (unchanged)

Step 4 is also the fix for the latent `repeatable` bug: `_convert` already splits raw
text on newlines, but until now no editor could hold a second line.

### Vim mode display hook

Give the form's editors one shared `_update_vim_mode_display(indicator="")` override
(mixin or module-level helper, following `_route_vim_mode` in
`axe_entry_editor_rendering.py`): if the editor's `screen` exposes
`_set_editor_mode_label`, hand it the mode label and indicator; otherwise call
`super()`, which keeps today's border-title behavior for `InputCollectionModal`. Wrap
the host call in a narrow `except Exception` — a host without the hook must never break
editing. Phase `panel` implements the consumer.

### Cleanups

`TypedInputForm` no longer constructs any Textual `Input`, so delete its
`on_input_changed` / `on_input_submitted` handlers and the now-unused `Input` import.
Confirm with `just check` that symvision does not flag the leftovers.

### Tests

Extend `tests/ace/tui/test_typed_input_form.py`:

- Rewrite `test_secret_field_renders_masked_input_and_never_leaks_raw_text` for
  `SecretVimTextArea`, keeping its shape (`render_line` segments contain no raw text)
  and adding that `.text` still returns the raw value and that the empty-field
  placeholder is not masked.
- A `text`-typed field accepts a newline and `typed_values()` preserves it.
- A `repeatable` `line` field typed as two lines converts to a two-item list.
- `<escape>` in a multi-line field enters NORMAL mode and a NORMAL-mode motion works.
- `<tab>` from a multi-line field moves focus instead of inserting whitespace.

Run `tests/ace/tui/test_prompt_input_collection_launch.py` and
`just test-visual -k prompt_inputs` to confirm the shared host is unaffected; rebaseline
`prompt_inputs_*` goldens only if a `text`/`repeatable` field actually appears in them,
and say so in the commit message if you do.

---

## Phase `panel`: The GateInputPanel modal and its pure request model

### Goal

A self-contained, unit-testable panel that turns "these options were selected" into
"here are their per-option input values", with no gate-modal wiring yet.

### `src/sase/ace/tui/modals/gate_input_panel_model.py` (pure, no Textual)

```python
@dataclass(frozen=True)
class GateInputSectionSpec:
    option_id: str
    label: str
    icon: str | None
    fields: tuple[GateInputField, ...]              # first-declared here, in order
    shared_with: Mapping[str, tuple[str, ...]]      # field id -> other option labels
    raw_properties: tuple[str, ...]                 # non-host-collected schema props
    raw_seed_text: str                              # YAML seed from schema defaults

@dataclass(frozen=True)
class GateInputRequest:
    branch_index: int
    branch_label: str
    selected_option_ids: tuple[str, ...]
    sections: tuple[GateInputSectionSpec, ...]
    feedback_mode: GateFeedbackMode
    feedback_field_owner: str | None
    conflict: str | None
    draft: GateInputDraft
```

- `build_gate_input_request(options, selected_option_ids, *, branch_index, branch_label, feedback_mode, host_collected_properties, draft)`
  builds it. Field ownership comes from walking selected options in branch order and
  reusing `collected_input_fields` for the dedupe/conflict rule: a `GateError` becomes
  `conflict`, not an exception.
- `requires_panel` — any section has fields or `raw_properties`, or
  `feedback_mode == "required"`.
- `is_empty` — no sections and `feedback_mode == "disabled"`.
- `collect_option_inputs(request, values, raw_values)` returns the `option_id -> dict`
  mapping by calling `option_inputs_from_values` for declared fields and merging each
  raw option's parsed YAML. Keep the reviewer-facing `GateBranchInputError` message
  wording.
- Move `gate_declares_inputs` here unchanged (signature and both call sites in
  `custom_gate_modal.py` and `plan_approval_footer.py` keep working) and re-export it
  from `gate_branch_controls` so `wire` is a pure deletion there.

`GateInputDraft` is a small frozen dataclass of `values: Mapping[str, str]` (raw text by
field id), `raw_text: Mapping[str, str]` (by option id), and `feedback: str`.

### `src/sase/ace/tui/modals/gate_input_panel_sections.py`

`GateInputSection(Vertical)` renders one `GateInputSectionSpec`:

- a section rule: `{icon} {label}` left, `{n} inputs · {m} required` right;
- a `TypedInputForm(fields, id_prefix=f"gate-input-{option_id}", optional_toggle=False)`
  — optional fields are always visible here, because width is no longer scarce;
- a `VimTextArea(language="yaml")` raw editor plus its error label when `raw_properties`
  is non-empty, seeded from `raw_seed_text` and validated on change with
  `first_schema_error` (lift both from `gate_branch_input_section.py`);
- `· also sent to <label>` guidance appended to a shared field's description.

Public API: `control_ids()`, `is_valid()`, `focus_first_invalid()`, `values()`,
`raw_text()`, `required_progress()`.

### `src/sase/ace/tui/modals/gate_input_panel.py`

`GateInputPanel(ModalScreen[GateInputPanelResult | None])`:

- `GateInputPanelResult(option_inputs, feedback, draft)`; cancel dismisses `None`.
- Compose: `Container#gate-input-container` > header `Static`,
  `VerticalScroll#gate-input-body` (sections, then the Note section when
  `feedback_mode != "disabled"` and `feedback_field_owner is None`),
  `Static#gate-input-progress`, `Horizontal` with the submit and cancel buttons,
  `Static#gate-input-footer`. The Note editor is a `_MultilineInput`-equivalent
  `VimTextArea`; its section rule reads `Note` with `required`/`optional` on the right.
- A `conflict` request renders the message and a cancel-only footer, and refuses submit.
- Static bindings this phase: `tab` → `next_input`, `shift+tab` → `previous_input`, and
  `ctrl+s` → `submit`, all `priority=True` so a focused editor cannot swallow them.
  `escape` → `cancel` must **not** be `priority` — a priority screen binding would
  preempt the focused editor and break the documented two-stage escape, where the first
  `escape` leaves INSERT and only a NORMAL-mode `escape` bubbles up to cancel. This
  mirrors how the gate modals already split their static and keymap bindings.
- Focus ring: `_control_ids()` = each section's `control_ids()` in order, the note
  editor, the submit button, the cancel button; `next`/`previous` wrap, mirroring
  `focus_gate_control`'s index-and-modulo shape.
- `on_typed_input_form_changed` / `on_text_area_changed` refresh the progress label and
  the submit button's `disabled` state. `on_typed_input_form_submitted` advances to the
  next section, or submits when there is none.
- `action_submit`: if invalid, `focus_first_invalid()` on the first invalid section (or
  the note when a required note is empty) and `notify(..., severity="warning")`;
  otherwise build the result with `collect_option_inputs` and dismiss.
- `_set_editor_mode_label(mode, indicator)` updates the footer's mode chip — the hook
  `editors` added.
- `on_mount` focuses the first invalid control, falling back to the first control.

Register `GateInputPanel`, `GateInputPanelResult`, and `GateInputRequest` in
`modals/__init__.py` `_LAZY_EXPORTS` and `modals/__init__.pyi`.

### Tests

New `tests/ace/tui/test_gate_input_panel.py`, driving the panel directly from a small
host app (the `_ControlsApp` pattern in `test_gate_branch_inputs.py`):

- one section per selected option, headed by that option's icon and label;
- a shared compatible field renders once and lands in both options' `option_inputs`;
- an incompatible shared field renders the conflict message and blocks submit;
- `<tab>` / `<shift+tab>` walk every visible editor, enum button, and both buttons, and
  wrap at each end;
- a required field blocks submit until filled, and an attempted submit focuses it;
- a raw-schema option round-trips YAML into its own `option_inputs` entry;
- the Note section appears for `required`/`optional` feedback, is suppressed when an
  option declares its own `feedback` field, and that declared field's value is returned
  as the result's `feedback`;
- cancel dismisses `None` and returns a draft-preserving result to the caller's state
  (assert through the panel's `draft` on the next construction);
- pure-model unit tests for `requires_panel` / `is_empty` across the decision table in
  "When the panel opens".

---

## Phase `wire`: Route every gate submission through the panel

### Goal

The gate modals stop rendering inputs. `GateBranchControls` opens the panel when the
selection needs typed input and posts the same `Resolved` message it posts today.

### `gate_branch_controls.py`

- `compose`: drop `_compose_branch_inputs`, the `#gate-feedback-label` `Static`, and the
  `#gate-feedback-input` `Input`. `_sections` disappears.
- Replace `_feedback_by_branch` with `_draft_by_branch: dict[int, GateInputDraft]`.
- `_resolve_branch(branch_index)` becomes:
  1. existing guards: `_submission_block`, empty selection;
  2. `request = build_gate_input_request(...)` with the branch's selected options, the
     branch label, `self._feedback_mode(branch_index)`,
     `self._host_collected_properties`, and the saved draft;
  3. `request.conflict` → `notify(conflict, severity="warning")` and return;
  4. `not request.requires_panel` → post
     `Resolved(selected, draft.feedback or None, {})` immediately (unchanged fast path);
  5. otherwise `_open_input_panel(request)`.
- `_open_input_panel` guards against a second push with a `_panel_open` flag, calls
  `self.app.push_screen(GateInputPanel(request), callback)`, and in the callback: `None`
  → store the returned draft, clear the flag, refocus the originating control; otherwise
  → store the draft and post
  `Resolved(selected, result.feedback, result.option_inputs)`. The callback must no-op
  when `self.is_mounted` is false, so a gate torn down while the panel is open cannot
  resolve into a dead widget.
- `open_inputs_for_focused_control()` (public, used by the `keys` phase) resolves the
  focused control's branch, builds the request, and opens the panel even when
  `requires_panel` is false — unless `request.is_empty`, which warns
  `"This option takes no input"`.
- `visible_control_ids()` drops its section ids; `_update_submit_state` keeps only
  `_submission_block` and the AND-group "at least one selected" rule (an unfilled input
  no longer disables a submit control — the panel enforces validity now, and disabling
  the only control that opens the panel would be a dead end);
  `_sync_branch_input_visibility`, `_branch_inputs_valid`, `_update_feedback_controls`,
  `on_gate_branch_input_section_validated`, `on_typed_input_form_changed`,
  `on_input_changed`, and `on_input_submitted` are deleted.
- `submit_primary_branch()` loses its focused-feedback-input special case.
- `_feedback_mode` and `_feedback_value` stay: `_feedback_value` now reads the branch's
  draft.

Delete `src/sase/ace/tui/modals/gate_branch_input_section.py`; `gate_declares_inputs`
now comes from `gate_input_panel_model`, re-exported by `gate_branch_controls` so
`custom_gate_modal.py` and `plan_approval_footer.py` need no import churn.

### Host modals

`CustomGateModal` and `PlanApprovalModal` need no structural change beyond footer text
(deferred to `keys`). Verify `HOST_COLLECTED_PROPERTIES` still reaches the request via
`GateBranchControls.__init__`, so plan gates keep hiding `coder_prompt`, `coder_model`,
and `epic_launch_mode` from the raw-schema editor.

### Tests to rewrite

- `tests/ace/tui/test_gate_branch_inputs.py`: the premise of every test moves from
  "fields render inline" to "activating the selection opens the panel with these
  sections". Keep one test asserting a no-input branch still submits with a single press
  and never pushes a screen.
- `tests/ace/tui/test_custom_gate_modal.py`:
  `test_declared_input_value_reaches_resolved_option_inputs`,
  `test_numbered_shortcut_focuses_required_enum_then_submits`,
  `test_task_triage_shortcut_focuses_duration_line_then_submits`, and
  `test_required_feedback_blocks_until_entered` now drive the panel — a digit or
  `<enter>` opens it, values are typed there, and `<ctrl+s>` inside the panel produces
  the `Resolved` payload.
- `tests/ace/tui/test_notification_plan_gate.py` and
  `test_gate_primary_specialized_modals.py`: the feedback branch and the snooze duration
  field route through the panel.
- Add a regression test that cancelling the panel restores what was typed when the same
  option is activated again.

---

## Phase `keys`: Configurable panel keymaps and modal footers

### Goal

The panel's navigation keys are remappable like every other gate key, and both footers
advertise the new flow.

### Keymap plumbing

1. `src/sase/ace/tui/keymaps/app_keymaps.py` — add to `GateModalKeymaps`:
   `open_inputs: str = "i"`, `next_input: str = "tab"`,
   `previous_input: str = "shift+tab"`.
2. `src/sase/ace/tui/keymaps/metadata.py` — append to `_GATE_BINDING_META`:
   `("open_inputs", "Open inputs")`. Leave `next_input` / `previous_input` out of that
   tuple: it feeds `build_gate_modal_bindings`, which binds on the _gate_ modal, and
   these two belong to the panel. Expose them through a small
   `build_gate_input_panel_bindings(keymaps)` in `keymaps/bindings.py` instead.
3. `src/sase/default_config.yml` — add all three under `ace.keymaps.gate`, in the same
   order as the dataclass.
4. `GateBranchControls.__init__` takes `gate_keymaps: GateModalKeymaps | None = None`;
   both host modals pass theirs; the controls hand them to `GateInputPanel`, which uses
   `build_gate_input_panel_bindings` plus `key_display_name` for its footer.
5. `CustomGateModal` / `PlanApprovalModal` gain an `action_open_inputs` bound from
   `open_inputs`, delegating to `GateBranchControls.open_inputs_for_focused_control()`.
   `gate_modal_taken_keys` reads the keymap dataclass's single-character values, so `i`
   is automatically withheld from declared gate-action keys — assert that in a test.

### Footers

- Both gate footers: replace the conditional `^t complete path` hint (it now belongs to
  the panel) with `{open_inputs} note/inputs` when the gate declares any input or any
  non-disabled feedback, using the existing `gate_declares_inputs` call site.
- Panel footer: `{next_input}/{previous_input} field · {submit} submit · <esc> back`,
  plus `^t complete path` only when a `path` field is present, plus the mode chip.

### Tests

- A remapped `gate.open_inputs` opens the panel and is reflected in both footers.
- Remapped `next_input` / `previous_input` drive the panel ring.
- `activate_control` legacy migration still passes (`keymaps/scopes.py` untouched).

---

## Phase `chrome`: Panel styling, option input badges, and visual snapshots

### Goal

The panel looks deliberate, and an option that will ask for input says so before it is
pressed.

### Styles (`src/sase/ace/tui/styles.tcss`)

Add a `GateInputPanel` block:

```tcss
GateInputPanel { align: center middle; }
GateInputPanel > #gate-input-container {
    width: 90%; max-width: 120; min-width: 56;
    height: auto; max-height: 90%;
    background: $surface; border: double $accent;
    border-title-align: left; padding: 0 1;
}
GateInputPanel #gate-input-body { height: auto; max-height: 32; scrollbar-gutter: stable; }
GateInputPanel .gate-input-section-title { height: auto; text-style: bold; margin-top: 1; }
GateInputPanel .input-field-block { margin-bottom: 1; }
GateInputPanel SingleLineVimTextArea, GateInputPanel SecretVimTextArea { height: 3; }
GateInputPanel VimTextArea { height: 8; }
GateInputPanel #gate-input-buttons { height: auto; align-horizontal: right; }
GateInputPanel #gate-input-footer { height: auto; color: $text-muted; border-top: solid $secondary; }
```

Delete the now-dead rules: `GateBranchControls .gate-branch-inputs`,
`GateBranchControls .gate-input-conflict`, `GateBranchControls TypedInputForm …`,
`GateBranchControls .gate-branch-inputs TextArea`,
`GateBranchControls #gate-feedback-label`, `GateBranchControls #gate-feedback-input`.
`GateBranchControls`'s `max-height: 18` can relax now that it holds controls only.

### Option badges

In `gate_branch_layout.py`, give both `compose_singleton_row` and `toggle_label` a
trailing dim badge for an option that declares input: `[dim]✎ {n} inputs[/dim]`
(singular `input` for one), computed from the option's declared `inputs` or its
non-host-collected raw schema properties. Keep `escape()` on all option-authored text.
Widen `GateBranchControls .gate-singleton-row Button` enough that the badge is not
clipped in the two-pane layout.

For consistency, `notification_modal_gate._declared_input_line` keeps listing declared
fields but the pending-state summary line gains the same `✎ n inputs` wording.

### Visual snapshots

The gate footers changed, so **every** gate golden shifts. Run `just test-visual`,
accept with `--sase-update-visual-snapshots`, and eyeball each diff in
`.pytest_cache/sase-visual/` before committing: `custom_gate_actions`,
`custom_gate_choices_only`, `custom_gate_draft_banner`, `custom_gate_extras`,
`custom_gate_frontmatter`, `custom_gate_inputs`, `custom_gate_no_preview`,
`custom_gate_required_feedback`, `custom_gate_task_triage`, `plan_gate_epic_action`,
`plan_gate_frontmatter`, `plan_gate_tale_five_controls`, `plan_gate_tale_stacked`.

Add three panel goldens in `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py`:

- `gate_input_panel_single_120x45` — one option, a required `line`, an `enum`, a
  `secret`, and an optional note;
- `gate_input_panel_group_120x45` — an AND selection with two option sections, one of
  them a raw-schema YAML editor, and a shared field annotated `also sent to`;
- `gate_input_panel_note_120x40` — a `feedback: required` option, note-only panel.

`custom_gate_inputs_120x45` keeps its name but now shows the _left pane with badges and
no Inputs section_ — that contrast is the point of keeping it.

---

## Phase `docs`: Document the panel and its keys

Update, without inventing behavior the code does not have:

- `docs/notifications.md`
  - the gate-review paragraph (around "Enter submits the declared primary branch"):
    describe that a selection needing typed input opens the input panel first, that
    confirming the panel submits, and that `i` opens it for an optional note;
  - the `Gate inputs` section: replace "the reviewer types the value into a raw-schema
    editor — ACE's YAML editor" with the panel's per-option section, and state that
    shared field ids are collected once and distributed;
  - the modal keybinding table: add `i` and the panel's `<tab>` / `<shift+tab>`.
- `docs/configuration.md` — extend the `ace.keymaps` gate table with `open_inputs`,
  `next_input`, `previous_input` and their defaults.
- `docs/ace.md` — extend the `Remapping Gate Modal Keys` YAML example and prose.

Leave `src/sase/xprompts/skills/sase_gate.md` alone unless a sentence there became
false; it documents authoring, not reviewing. If it must change, read
`sase/memory/generated_skills.md` with `/sase_memory_read` first and follow the
regeneration workflow it describes.

---

## Verification

- Every phase: `just install`, then `just check` before handing off. Hand a long run to
  `/sase_monitor` with a `--next` action rather than blocking a turn on it.
- `editors` and `chrome` additionally run `just test-visual` (`chrome` rebaselines).
- The epic's landing change runs `just check-full` through `/sase_monitor`.
- Manual smoke on the landed tree: open a `sase gate create` gate with a required `enum`
  plus a `path` field, confirm the panel opens on the option press, that `<tab>` wraps,
  that `<ctrl+t>` completes the path, that `<esc><esc>` returns to the gate with the
  typed value intact, and that submitting writes the expected `option_inputs` into the
  gate's `response.json`.
