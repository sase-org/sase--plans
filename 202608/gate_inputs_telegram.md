---
tier: tale
title: Telegram declared-input step flow for gate options
goal:
  A Telegram reviewer is prompted for each input a selected gate option declares,
  answers it with a typed reply or an inline keyboard, and submits one value per option
  through `option_inputs` — with the local feedback heuristic deleted so the executor
  owns that rule for every surface.
size: large
proposed_by: bbugyi200.athena.sase-h7.8
bead: sase-h7.8
create_time: 2026-08-07 20:09:55
status: wip
---

- **PARENT:**
  [202608/gate_input_collection.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_input_collection.md)
- **BEAD:**
  [sase-h7.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.8.md)

# Plan: Telegram declared-input step flow for gate options

This finishes phase `inputs-remote` (`sase-h7.8`) of epic `sase-h7`. Sections 1–5 of the
phase plan `202608/gate_inputs_remote.md` have already landed; only its sections 6 and 7
— the `sase-telegram` surface — remain.

## What already landed

Verified in these checkouts, not assumed:

- **`sase-core`** —
  `65e0ec1 feat(mobile)!: carry declared gate inputs on the mobile wire` added
  `MobileGateInputFieldWire` and `MobileInputChoiceWire`, put `option_inputs` on
  `GateActionRequestWire`, widened the envelope reader to accept `schema_version` 2 and
  3, added `choices` to the xprompt input wire, threaded the gateway route, bumped
  `MOBILE_NOTIFICATION_WIRE_SCHEMA_VERSION` 4 → 5, and regenerated the frozen contract
  snapshot. `a35fe91` (from `sase-h7.3`) added `InputType::Enum` to the Rust frontmatter
  engine.
- **`sase`** —
  `7bbd82a4 feat(mobile): accept per-option gate inputs on the mobile bridge` widened
  the bridge's `schema_version` check to the accepted range, parsed `option_inputs`
  through to `execute_gate_selection`, and added
  `src/sase/notification_gates/input_collection.py` with `collected_input_fields`,
  `input_arg_for_field`, `coerce_field_text`, and `option_inputs_from_values`, all
  re-exported from `sase.notification_gates`.
- **`sase-telegram`** — nothing. `feedback_is_command_input` (`gate_flow.py:326-337`) is
  still the only feedback rule on this surface, `GateProgress` has no input state, and
  every submission sends `input_data={}`.

Because the shared helper already exists, this plan writes no conversion rules of its
own: it is a step-flow state machine plus its Telegram presentation.

## Background — how a Telegram gate is answered today

`render_gate_keyboard` (`src/sase_telegram/formatting.py:1026`) emits four compact
callback tokens under the 64-byte ceiling: `c<i>` picks a branch or expands an AND
group, `x<i>` toggles one member of the expanded group, `s<i>` submits the group, and
`f<i>` submits it through the two-step feedback flow.

`_handle_gate_callback` (`src/sase_telegram/scripts/sase_tg_inbound.py:1346`) decodes
them, keeps selection state in the bundle-local `telegram_gate_progress.json` through
`gate_flow.load_progress`/`save_progress`, and ends in one of two places: a required
feedback mode routes to `_begin_gate_feedback`, which records an
`awaiting_feedback.json` entry so the user's next text message resolves the gate;
anything else builds a `ResponseAction` with `input_data={}` and calls
`resolve_gate_response` (`src/sase_telegram/inbound.py:342`), which calls
`execute_gate_selection`.

The text leg runs through `_handle_text_message` (`sase_tg_inbound.py:3706`) →
`process_text_message` (`inbound.py:296`), which today copies the note into
`input_data["feedback"]` when the stored `feedback_is_command_input` flag is set. That
flag tests `input_schema.required`, so an option whose schema declares `feedback` only
under `properties` is silently starved — the divergence the epic's `feedback-input`
phase already deleted host-side.

`question_flow.py` plus `_handle_question_text_message` is the precedent this plan
models: a pure decision module holding progress and transitions, and a script-side
handler that performs the Telegram calls.

## 1. `gate_flow.py` — input state on `GateProgress`

Add four fields, all defaulted so every existing construction site keeps working:

```python
input_option_ids: tuple[str, ...] = ()
input_field_index: int | None = None
input_values: dict[str, Any] | None = None
input_feedback_requested: bool = False
```

`input_option_ids` is the selection the reviewer already committed to — the input block
is opened only once a branch resolves, so it does not duplicate `selected_option_ids`,
which stays the toggle state of an expanded AND group.

`input_feedback_requested` is a deliberate addition to the three fields the phase plan
named. Without it, pressing **💬 … with feedback** on a gate that also declares inputs
loses the reviewer's request for the feedback step, because collection runs _before_
feedback and `feedback_mode` alone cannot distinguish "optional feedback, asked for it"
from "optional feedback, skipped it".

`save_progress` persists all four. `load_progress` recovers them defensively, in the
same spirit as the existing fields — any of these resets the whole input block to its
empty state rather than raising:

- `input_option_ids` missing, not a list of strings, empty, or naming an option the
  verified envelope does not declare;
- `input_field_index` absent or not a non-negative integer;
- `input_values` present but not an object with string keys.

Resetting the block leaves the gate itself untouched and answerable from its keyboard,
which is the correct outcome for a stale or hand-edited progress file.

Delete `feedback_is_command_input` in the same module (section 6 below).

## 2. New `src/sase_telegram/gate_inputs.py` — pure decisions

No Telegram API calls, no I/O. It wraps the shared helpers from
`sase.notification_gates` and never re-implements a conversion rule.

```python
@dataclass(frozen=True)
class GateInputStep:
    field: GateInputField
    index: int      # 0-based position in the collected field list
    total: int

    @property
    def position(self) -> int:   # 1-based, for "Input 2/3"
        ...

def pending_fields(view: GateView, option_ids: Sequence[str]) -> tuple[GateInputField, ...]
def unsupported_fields(fields: Sequence[GateInputField]) -> tuple[GateInputField, ...]
def begin_input(progress: GateProgress, option_ids: Sequence[str], *,
                feedback_requested: bool) -> GateProgress
def clear_input(progress: GateProgress) -> GateProgress
def current_step(view: GateView, progress: GateProgress) -> GateInputStep | None
def advance(progress: GateProgress) -> GateProgress
def apply_text_answer(values: Mapping[str, Any], field: GateInputField, text: str) -> dict[str, Any]
def apply_choice(values: Mapping[str, Any], field: GateInputField, value: str) -> tuple[dict[str, Any], bool]
def skip_step(values: Mapping[str, Any], field: GateInputField) -> dict[str, Any]
def submitted_option_inputs(view: GateView, progress: GateProgress) -> dict[str, dict[str, Any]]
def encode_input_token(index: int, verb: str) -> str
def decode_input_token(token: str) -> tuple[int, str] | None
```

Rules:

- `pending_fields` resolves the ids to verified `GateOption`s and delegates to
  `collected_input_fields`, so a field two AND members both declare is collected once
  and a genuinely conflicting redeclaration raises the shared
  `GateError("conflicting_input_field", ...)`. An id the envelope does not declare is
  skipped rather than raising — the caller has already validated the selection.
- `unsupported_fields` returns the `secret: true` fields. Telegram refuses them; see
  section 4.
- `apply_text_answer` calls `coerce_field_text`, which raises `XPromptValidationError`
  with the shared message on a bad answer and splits a `repeatable` field on newlines.
- `apply_choice` handles the enum keyboards. For a scalar enum it sets the value and
  returns `(values, True)`. For a `repeatable` enum it toggles membership of a list kept
  in declared choice order and returns `(values, selected_now)` so the caller can answer
  the callback with the right verb.
- `skip_step` returns `values` with the field's id absent, so the compiled schema's
  `required` list stays the only enforcement layer. Skipping a required field is refused
  by the caller, not silently allowed here.
- `submitted_option_inputs` delegates to `option_inputs_from_values` over the resolved
  selection, so an option that declares nothing maps to `{}` and two AND members with
  mutually exclusive `additionalProperties: false` schemas are both answerable from one
  collected set.
- Tokens are `i<field_index><verb>` where `verb` is `v<choice_index>` (pick an enum
  value), `k` (skip), `d` (done with a repeatable enum), or `c` (cancel). Encoded as
  `gate:<8-char prefix>:<token>`, the longest realistic token is well inside the 64-byte
  ceiling `callback_data.encode` already enforces. `decode_input_token` returns `None`
  for anything malformed rather than raising.

The field index in every token is what makes a stale prompt harmless: a tap whose index
does not match `current_step` is answered "This input step is no longer active" and
changes nothing.

## 3. `formatting.py` — the prompt and its keyboard

Two additions beside `render_gate_keyboard`:

- `format_gate_input_prompt(step: GateInputStep) -> str` — a MarkdownV2 message built
  through the existing `escape_markdown_v2`: a `📝 Input <n>/<total>` header with the
  field label, its declared type, `help` when set, `placeholder` rendered as an example,
  `Optional — use Skip to leave it unset.` for a non-required field, and
  `Send one value per line.` for a `repeatable` non-enum field.
- `render_gate_input_keyboard(prefix, step, values) -> InlineKeyboardMarkup` — one
  button per declared choice for an `enum` field (prefixed ☑️/⬜ and followed by a
  **Done** button when the field is `repeatable`, reusing the AND-group toggle idiom),
  then a **Skip** button for an optional field and a **Cancel** button on every step. A
  non-enum field renders only Skip and/or Cancel; its value arrives as a text reply.

Choice buttons show `label` when declared and `value` otherwise.

## 4. `sase_tg_inbound.py` — the step flow

**Where collection starts.** Both terminal paths in `_handle_gate_callback` funnel into
one new helper instead of deciding for themselves:

```python
def _start_or_submit_gate_selection(callback_query, action, prefix, view, progress,
                                    selected_option_ids, *, feedback_requested) -> None
```

- The `f<i>` branch calls it with `feedback_requested=True` after its existing
  `feedback_mode == "disabled"` guard.
- The fall-through submit path calls it with
  `feedback_requested=(feedback_mode(view, selected) == "required")`, replacing the
  inline `_begin_gate_feedback` call and the `ResponseAction(input_data={})`
  construction.

The helper resolves `pending_fields`; a `GateError` from a conflicting redeclaration is
answered with its message and changes nothing. If any field is `secret`, it refuses the
whole submission — `Telegram cannot collect secret input: <ids>` — and leaves the gate
pending. Otherwise, declared fields open the input block and send the first prompt; no
declared fields fall through to today's behavior (feedback step, or immediate
submission).

**Prompting.** `_send_gate_input_prompt` sends the step as a **new** message carrying
the step keyboard, then re-points the awaiting entry at it:
`clear_awaiting_feedback_by_prefix(prefix)` followed by
`save_awaiting_feedback(str(message.message_id), prefix, {"action_type": "gate_input", "bundle_path": ...})`.
Exactly one awaiting entry per gate stays live, so an un-quoted text reply still routes
through `load_awaiting_feedback(None)`'s single-entry fallback exactly as the feedback
flow relies on today.

`"gate_input"` is a new `action_type`. `process_text_message` already returns `None` for
any type it does not recognize, so a stray reply can never be mistaken for a gate
submission.

**Callback leg.** `_handle_gate_callback` gains an early branch — before the
`c`/`x`/`f`/`s` dispatch, which would otherwise fall through to "Invalid gate callback":

```python
if cb.choice.startswith("i"):
    _handle_gate_input_callback(callback_query, action, cb.notif_id_prefix, view, progress)
    return
```

`_handle_gate_input_callback` decodes the token and then:

- `c` — clears the input block, clears the awaiting entry, strips the prompt's keyboard,
  and answers `Input cancelled — the gate is still answerable`. The original gate
  message keeps its keyboard throughout collection, so cancelling needs no re-render and
  a re-tap simply restarts collection from the first field.
- index mismatch with `current_step` — `This input step is no longer active`, no state
  change.
- `k` — refused for a required field; otherwise records the skip and advances.
- `d` — valid only for a `repeatable` enum; refused when the field is required and
  nothing is selected; otherwise advances.
- `v<n>` — out-of-range choice index is `Invalid gate callback`. A `repeatable` enum
  re-renders the prompt keyboard with the new toggles and stays on the field; a scalar
  enum strips the keyboard and advances.

**Text leg.** New
`_handle_gate_input_text_message(message, text, *, reply_key) -> bool`, dispatched from
`_handle_text_message` immediately after `_handle_question_text_message` and before
`process_text_message`, mirroring that precedent. It returns `False` unless the matched
awaiting entry is a `gate_input` one, and then:

- a missing pending action, an unreadable gate, or an already-written `response.json`
  clears the awaiting entry and replies with the existing stale/expired text;
- a completed or absent step replies `This input step is no longer active`;
- an `enum` step replies `Choose one of the buttons above` and stays put;
- `XPromptValidationError` replies with the shared message and **re-sends the same
  prompt**, so a bad answer neither advances nor is dropped;
- otherwise it records the value and advances.

**Advancing and completing.** `_advance_gate_input` bumps the index, saves, and either
sends the next prompt or completes. Completion computes `submitted_option_inputs`,
clears the input block and the awaiting entry, then either routes to
`_begin_gate_feedback` (carrying the collected `option_inputs`) when feedback was
requested or required, or submits.

`_execute_gate_callback_response` gains an optional `message` parameter so the same
submission path serves both legs: with a callback it answers the callback as today; with
a text message it also replies in-chat, reusing `_send_confirmation` on success and
`_gate_error_answer_text` on failure. Existing call sites pass nothing and are
unchanged.

## 5. Submission — one shape on this surface

`ResponseAction` (`inbound.py:118`) gains `option_inputs: dict[str, Any] | None = None`.

`resolve_gate_response` passes `option_inputs=response.option_inputs` and
`input_data=None` whenever it is set; the two are mutually exclusive and the executor
raises `conflicting_input` for the combination. When it is `None` — only reachable from
an awaiting entry written before this change — the existing shared-value call is
unchanged.

Every Telegram submission path sets it, including gates that declare nothing, where it
resolves to `{}` per selected option. That is byte-identical to the `input_data={}` this
surface sends today, and it means there is one submission shape here rather than two.

`_begin_gate_feedback` writes `option_inputs` into the awaiting entry, and
`process_text_message` reads it back onto the `ResponseAction`.

## 6. Delete the feedback heuristic

- `gate_flow.feedback_is_command_input` and its import in `sase_tg_inbound.py:74`.
- The `"feedback_is_command_input"` key written by `_begin_gate_feedback`
  (`sase_tg_inbound.py:1332`) and read by `process_text_message` (`inbound.py:326-327`),
  along with the `input_data["feedback"] = text` injection it guarded.

`apply_feedback_input` inside `execute_gate_selection` now owns the rule and tests
`properties` rather than `required`, so an optional-feedback command this surface
silently starved starts receiving the note. The note also keeps its existing home in
`response.feedback`, unchanged.

## Tests

**New `tests/test_gate_inputs.py`** — the pure decision layer: collected-field order and
dedupe across two options; the conflicting-redeclaration `GateError`;
`unsupported_fields` picking out a secret; token encode/decode round trip and `None` for
malformed tokens; `apply_text_answer` converting each scalar type and raising on a bad
`int`; newline splitting for a `repeatable` field including a blank line; scalar-enum
set and repeatable-enum toggle-off; `skip_step` leaving the id absent;
`submitted_option_inputs` giving each option only its declared ids and `{}` to an option
that declares none.

**`tests/test_gate_flow.py`** — drop the `feedback_is_command_input` test; add a round
trip of the four new progress fields through `save_progress`/`load_progress`, and a
malformed input block (bad index, non-list option ids, an option id the envelope does
not declare) resetting to the empty block while leaving branch selection intact.

**`tests/test_custom_gates.py`** — the end-to-end step sequence against a real bundle
built with `create_gate`, reusing the module's `gate_home` fixture and callback helpers:

- a selection declaring a required `line`, an optional `enum`, and a `bool` prompts in
  declared order, the enum keyboard carries one button per choice plus Skip and Cancel,
  and the collected values reach the command's stdin as `option_inputs` — asserted from
  the persisted `response.json`;
- two selected AND members with mutually exclusive `additionalProperties: false` schemas
  each receive only their own declared ids;
- an invalid answer re-prompts the same field and leaves the gate pending with no
  `response.json`;
- Skip omits an optional id and is refused on a required field;
- a `repeatable` enum accumulates toggles and submits on **Done**;
- **Cancel** clears the input block and leaves the gate answerable from its original
  keyboard;
- a `secret` field is refused with a pointed message and nothing is submitted;
- a gate declaring no inputs answers exactly as it does today (the existing tests cover
  this and must not change), and an optional-feedback option receives the note through
  the executor now that the local heuristic is gone.

**`tests/test_inbound.py`** — update the three `feedback_is_command_input` fixtures, and
assert `process_text_message` carries `option_inputs` from an awaiting entry onto the
`ResponseAction` while an entry written without the key still resolves through the
shared-value path.

`gate-cli` (`sase-h7.9`) owns `tests/gate_conformance/`, which does not exist yet. These
stay ordinary tests, and a `PROPOSED FOLLOW-UP:` note records folding them into that
matrix's Telegram leg.

## Verification

1. **`sase` workspace** — `just install` first (the workspace may have drifted, and that
   recipe rebuilds the `sase_core_rs` binding from the linked `sase-core` checkout,
   whose built module the `sase repo open` refresh cleared). Then `just check`. No file
   in this repo changes, so this is a regression check on the already-landed sections 4
   and 5, not a gate on new work.
2. **`sase-telegram`** — `just install`, then install this workspace's `sase` over the
   PyPI copy the way CI does:
   `uv pip install --python .venv/bin/python --no-deps -e <workspace sase path>`, with
   `sase_core_rs` made importable from the `sase-core` checkout the same way the sase
   workspace venv does it. Without that step the venv holds released `sase`, whose
   `execute_gate_selection` has no `option_inputs` parameter and whose `GateOption` has
   no `inputs`, and every new test fails for a reason unrelated to the change. Then
   `just check` (ruff + mypy + pytest).
3. Confirm `sase bead show sase-h7.8` is still `in_progress` after the `sase-telegram`
   commit — `sase commit` in a linked repo has auto-closed this workspace's assigned
   phase bead before, twice on this bead. Reopen with `sase bead open sase-h7.8` if it
   happens.

## Risks

- **The awaiting-entry key is shared state.** Both the feedback flow and the input flow
  write `awaiting_feedback.json` under the same prefix. Every prompt must clear the
  gate's previous entry before saving its own, or the un-quoted-reply fallback
  (`load_awaiting_feedback(None)`, which returns an entry only when exactly one exists)
  silently stops routing. The tests assert the map holds exactly one entry per gate
  across a multi-step flow.
- **Ordering inputs before feedback changes what `f<i>` means.** It now opens collection
  and only then asks for the note. `input_feedback_requested` is what keeps the note
  step from being dropped; a test covers the optional-feedback-plus-inputs combination
  end to end.
- **Deleting the feedback heuristic changes behavior for existing gates.** Options whose
  schema declares `feedback` only under `properties` start receiving the note. That is
  the epic's intended one rule, already audited and landed host-side by
  `feedback-input`; the Telegram change only stops the surface from second-guessing it.
- **`secret` is refused, not masked.** A chat transport persists the message in
  Telegram's history well beyond the gate, and masking is a display property we do not
  control there. Refusing at submission keeps the gate answerable from another surface.
- **The telegram venv is the likeliest source of confusing failures.** Its default
  installation pins released `sase`; a test failure mentioning an unexpected
  `execute_gate_selection` keyword or a missing `GateOption.inputs` means the override
  install did not take, not that the code is wrong.
