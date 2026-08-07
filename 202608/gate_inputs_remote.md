---
tier: tale
title: Mobile wire and Telegram step flow for declared gate inputs
goal:
  The mobile app and Telegram can see what input a gate option declares and submit one
  typed value per selected option, using the same closed vocabulary and the same
  feedback rule as every other surface.
proposed_by: bbugyi200.athena.sase-h7.8
bead: sase-h7.8
create_time: 2026-08-07 18:43:38
status: done
---

- **PROMPT:**
  [prompts/202608/gate_inputs_remote.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_inputs_remote.md)
- **PARENT:**
  [202608/gate_input_collection.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_input_collection.md)
- **BEAD:**
  [sase-h7.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.8.md)

# Plan: Mobile wire and Telegram step flow for declared gate inputs

This is phase `inputs-remote` of epic `sase-h7` (Gate input collection and repeatable
non-terminal gate actions). It carries the `inputs-core` contract (`sase-h7.3`) and the
one feedback rule (`feedback-input`, `sase-h7.2`) out to the two remote surfaces.

## Background

`GateOption.inputs` and `compile_gate_input_schema` already exist
(`src/sase/notification_gates/model_inputs.py`), `execute_gate_selection` already
accepts `option_inputs` and validates each selected option against its own schema
(`src/sase/notification_gates/executor_inputs.py`), and the executor already injects the
reviewer note as `input.feedback` wherever an option's schema declares that property
(`src/sase/notification_gates/feedback_input.py`). Neither remote surface can produce a
value, and neither can see what a gate asks for.

The mobile route is app → `sase_gateway` (Rust) → host bridge → Python
`mobile_notifications.py` → executor. The gateway serializes `GateActionRequestWire`
straight onto the bridge's stdin (`crates/sase_gateway/src/host_bridge.rs:521-565`), so
one added field has to land in the struct, the route body, the declared contract, and
the committed snapshot together — `contract.rs::committed_contract_snapshot_is_current`
compares the committed JSON against `api_v1_contract_snapshot()` byte for byte.

Four facts verified in these checkouts shape the work:

1. **Mobile renders no gate branches at all today.** `gate_branches_from_notification`
   returns an empty vector unless the envelope's `schema_version` is exactly `2`
   (`crates/sase_core/src/notifications/mobile.rs:630-632`), but every gate created
   since `005f431eb` (2026-07-18) writes `schema_version: 3`
   (`src/sase/notification_gates/service.py:288`). The Rust snapshot tests do not catch
   it because their fixtures are hand-written `"schema_version": 2` JSON. This phase
   owns the gate wire, so it fixes it; without the fix "render the declared inputs" has
   nothing to render into.
2. **Mobile's feedback heuristic is already gone.** `feedback-input` (`sase-h7.2`)
   landed it; `_mobile_notification_actions.py` has no
   `"feedback" in selected_option_ids` branch left. Only Telegram's
   `feedback_is_command_input` (`gate_flow.py:326-337`, used at
   `scripts/sase_tg_inbound.py:1332`) still needs deleting.
3. **Telegram tests already run against a `sase` checkout, not the release.**
   `.github/workflows/ci.yml` checks out `sase-org/sase`, builds `sase_core_py` with
   maturin, and installs the checkout with
   `uv pip install --no-deps -e .sase-deps/sase`. No dependency floor bump is needed,
   and the repo's own history (it adopted `sase.notification_gates.registry` while still
   declaring `sase>=0.1.0`) says not to add one.
4. **`GateOption.to_dict()` already writes `inputs` into the envelope**
   (`src/sase/notification_gates/model_options.py:166-178`), with `type` as its
   spelling, `choices` as `{value, label}` objects, and `default` as a raw JSON value.
   The Rust envelope reader can deserialize those fields directly; nothing new has to be
   written host-side.

## Scope

**In scope:** the `sase-core` gate wire (declared input fields on `GateOptionWire`,
`option_inputs` on `GateActionRequestWire`, the v3 envelope fix), `choices` on the
xprompt input wire, the gateway route body and regenerated contract snapshot, the Python
mobile bridge, a shared per-option input-collection helper in this repo, and the
Telegram declared-input step flow.

**Out of scope, owned by sibling phases:** rendering inputs in the ACE modals
(`inputs-ace`, `sase-h7.6`); `sase gate answer` and the `tests/gate_conformance/`
fixture matrix (`gate-cli`, `sase-h7.9`); creation-time answerability and dialect
pinning (`custom-validation`, `sase-h7.5`); gate actions on either remote surface
(`gate-actions`, `sase-h7.4`, and `gate-actions-ace`, `sase-h7.7`); retiring the snooze
and triage free-text smuggling (`retire-smuggling`, `sase-h7.11`);
`docs/mobile_gateway.md` and every other document (`docs`, `sase-h7.12`). Do not
implement any of those here.

## 1. `sase-core` — declared input fields on the gate wire

Open the repo with `/sase_repo` (`sase repo open sase-core -r "..."`) and use the
printed path as the only path for reads and writes.

In `crates/sase_core/src/host_bridge.rs`, beside the existing `MobileXpromptInputWire`:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct MobileInputChoiceWire {
    pub value: String,
    #[serde(default)]
    pub label: Option<String>,
}
```

One choice type serves both the gate input wire and the xprompt input wire (section 2),
because both project the same `InputChoice`.

In `crates/sase_core/src/notifications/mobile.rs`:

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct MobileGateInputFieldWire {
    pub id: String,
    pub label: String,
    #[serde(rename = "type")]
    pub r#type: String,
    #[serde(default)]
    pub required: bool,
    #[serde(default)]
    pub default: Option<JsonValue>,
    #[serde(default)]
    pub choices: Vec<MobileInputChoiceWire>,
    #[serde(default)]
    pub placeholder: Option<String>,
    #[serde(default)]
    pub help: Option<String>,
    #[serde(default)]
    pub secret: bool,
    #[serde(default)]
    pub repeatable: bool,
}
```

It mirrors `GateInputField` field for field, so the envelope's `inputs` array
deserializes with no host-side projection step and no lossy re-encoding of `default`.
Attach it to the option wire:

```rust
pub struct GateOptionWire {
    // ... existing fields ...
    #[serde(default)]
    pub inputs: Vec<MobileGateInputFieldWire>,
}
```

`GateActionRequestWire` gains the submission side:

```rust
#[serde(default, skip_serializing_if = "Option::is_none")]
pub option_inputs: Option<BTreeMap<String, JsonValue>>,
```

`skip_serializing_if` keeps the JSON the gateway writes to the Python bridge
byte-identical to today's whenever the app sends nothing, so no existing client changes
shape.

**`Eq` has to come off four structs.** `JsonValue` is not `Eq`, so
`MobileGateInputFieldWire`, and therefore `GateOptionWire`, `GateBranchWire`,
`MobileActionDetailWire`, and `MobileNotificationDetailResponseWire`, keep `PartialEq`
and drop `Eq`; `GateActionRequestWire` does the same. Nothing uses any of them as a map
key (`gate_branches_from_notification`'s `options_by_id` is keyed by `String`) and the
only other references are re-exports in `lib.rs` and `notifications/mod.rs` plus one
route signature, so the change is mechanical. `MobileInputChoiceWire` itself stays `Eq`.

**Fix the envelope version gate.** Replace the `envelope.schema_version != 2` early
return with an accept-set covering both live versions, named so the next bump is
obvious:

```rust
/// Gate request envelope versions this reader understands. `2` is the legacy
/// envelope; `3` is what every gate created since 2026-07-18 writes.
const READABLE_GATE_ENVELOPE_VERSIONS: [u32; 2] = [2, 3];
```

Bump `MOBILE_NOTIFICATION_WIRE_SCHEMA_VERSION` from `4` to `5`. `validate_schema`
already accepts `1..=CURRENT`, so an app pinned to `4` keeps working.

Re-export the two new types from `crates/sase_core/src/notifications/mod.rs` and
`crates/sase_core/src/lib.rs` alongside `GateOptionWire`.

Rust tests in `mobile.rs`:

- A `schema_version: 3` envelope carrying an option with a declared `inputs` array
  yields branches whose option wire reproduces every field, including an `enum` field's
  choices and a raw JSON `default`. Assert the branch list is non-empty — that assertion
  is the regression guard for fact 1.
- A `schema_version: 2` envelope with no `inputs` still yields branches with an empty
  `inputs` vector, and a `schema_version: 4` envelope yields none.
- `GateActionRequestWire` with `option_inputs: None` serializes without the key; with a
  value it round-trips.
- Add a `schema_version: 3` gate to one of the existing
  `tests/fixtures/mobile/*_contract.json` snapshots (regenerate the fixture from the
  assertion failure, do not hand-edit the version alone) so the committed fixtures stop
  describing an envelope version no producer writes.

## 2. `sase-core` — `choices` on the xprompt input wire

Closes the `PROPOSED FOLLOW-UP` recorded on `sase-h7.3`: `InputType.ENUM` exists in both
the Python and Rust frontmatter engines, but `MobileXpromptInputWire.type` is a
free-form string with no choice list, so an `enum` xprompt input reaches the app as an
unrenderable type. It rides along here because it lands in the same regenerated contract
snapshot.

- `crates/sase_core/src/xprompt_catalog.rs`: `CatalogInput` gains
  `choices: Vec<MobileInputChoiceWire>`; both branches of `parse_inputs` (the shortform
  mapping and the longform sequence) read a `choices` key holding either scalars or
  `{value, label}` mappings, matching `validate_input_choices`'s accepted shapes in
  `crates/sase_core/src/editor/frontmatter.rs:846-910`; `structured_inputs` copies it
  onto the wire. Step inputs keep an empty vector.
- `MobileXpromptInputWire` gains
  `#[serde(default)] pub choices: Vec<MobileInputChoiceWire>`. It stays `Eq`.
- Update the two construction sites in `crates/sase_xprompt_lsp` tests and
  `crates/sase_gateway/src/routes.rs:2968` for the added field.

Rust test: a shortform and a longform `type: enum` xprompt declaration each reach
`structured_inputs` with their declared choices and labels intact; a non-enum input
carries an empty vector.

## 3. `sase-core` — gateway route and the frozen contract

- `crates/sase_gateway/src/routes.rs`: `GateActionBody` gains
  `#[serde(default)] option_inputs: Option<BTreeMap<String, JsonValue>>` and
  `gate_action` threads it into `GateActionRequestWire`. Nothing in `host_bridge.rs`
  changes — it is generic over `Serialize`; the plan's reference to
  `host_bridge.rs:380,457,664` is three signatures that take `&GateActionRequestWire`
  and need no edit. Confirm that rather than assuming it.
- `crates/sase_gateway/src/contract.rs`: extend the declared `GateOptionWire` shape with
  `"inputs": "MobileGateInputFieldWire[]; default [] when absent"`, the
  `GateActionRequestWire` shape with
  `"option_inputs": "{option_id: json}|null; per-option submitted values, mutually exclusive with the shared value"`,
  the `MobileXpromptInputWire` shape with
  `"choices": "MobileInputChoiceWire[]; default [] when absent"`, and add
  `MobileGateInputFieldWire` and `MobileInputChoiceWire` record entries. Describe
  `default` as `"json|null; the declared default value, verbatim"` and `type` as the
  closed spelling set `word|line|text|path|agent|int|bool|float|enum`.
- Regenerate the snapshot rather than hand-editing it:

  ```bash
  cargo run -p sase_gateway -- --contract-out crates/sase_gateway/contracts/api_v1/mobile_api_v1.json
  ```

  then confirm `committed_contract_snapshot_is_current` passes.

## 4. This repo — the mobile bridge accepts `option_inputs`

`src/sase/integrations/mobile_notifications.py`:

- `MOBILE_NOTIFICATION_WIRE_SCHEMA_VERSION` goes from `4` to `5`.
- The request check stops demanding exact equality. It currently rejects anything but
  the current constant, which would turn the bump into a hard break for any app or
  gateway still sending `4`. Accept
  `1 <= schema_version <= MOBILE_NOTIFICATION_WIRE_SCHEMA_VERSION`, mirroring
  `validate_schema` in `crates/sase_core/src/notifications/mobile.rs:914-926` so the two
  sides of the bridge agree, and keep echoing the current constant in the response. Say
  in a comment that the two rules must stay in sync.
- Parse `option_inputs` for the `gate-action` operation: absent or `null` → `None`;
  otherwise it must be a JSON object whose keys are strings, else
  `MobileGateActionError("invalid_request", "option_inputs", ...)`. Values are passed
  through untouched — the executor's per-option schema validation is the enforcement
  layer, and duplicating it here would be a second, driftable one.
- A request whose `schema_version` is below `5` may not carry `option_inputs`; that
  combination is a confused client, so reject it with a pointed message rather than
  guessing.

`src/sase/integrations/_mobile_notification_actions.py`: `execute_mobile_gate_action`
gains keyword-only `option_inputs: Mapping[str, object] | None = None` and forwards it
to `execute_gate_selection`. When it is `None` the call is unchanged, so every bundle
answered today takes the same path.

## 5. This repo — one shared input-collection helper

Telegram, and later ACE and the CLI, all have to answer the same three questions: which
fields does this selection ask for, how does a typed text answer become a JSON value,
and which option gets which value. Put the answer once in this repo rather than three
times in three surfaces — that duplication is exactly the drift the epic exists to
delete.

New `src/sase/notification_gates/input_collection.py`:

```python
def collected_input_fields(options: Sequence[GateOption]) -> tuple[GateInputField, ...]:
    """Declared fields for one selection, in first-declared order, deduped by id."""

def input_arg_for_field(field: GateInputField) -> InputArg:
    """Project a field onto the shared xprompt InputArg conversion rules."""

def coerce_field_text(field: GateInputField, text: str) -> Any:
    """Convert one raw text answer to the field's declared type."""

def option_inputs_from_values(
    options: Sequence[GateOption], values: Mapping[str, Any]
) -> dict[str, dict[str, Any]]:
    """Distribute collected values to each option that declares them."""
```

Rules:

- `collected_input_fields` dedupes by id so a field two AND members both declare is
  collected once, which is the same rule `inputs-ace` states for the ACE form. Two
  selected options declaring the same id with a different type, `choices`, or
  `repeatable` cannot be collected once and cannot be collected twice; raise
  `GateError("conflicting_input_field", "inputs.<id>", ...)` naming both option ids.
  Differing `label`, `help`, `placeholder`, `required`, or `default` are presentation
  and the first declaration wins.
- `input_arg_for_field` sets `name` to the field id (it is what appears in
  `XPromptValidationError` messages), `default` to `UNSET` when the field is required
  and to the declared default otherwise, and copies `type`, `choices`, and `repeatable`.
  It never re-implements a conversion rule.
- `coerce_field_text` calls `InputArg.validate_and_convert` for a scalar field. For a
  `repeatable` field it splits the text on newlines, drops blank lines, converts each
  line, and returns a list — `validate_and_convert` has no repeatable branch because
  xprompt repeatability consumes positional arguments, which a chat transport has no
  equivalent of.
- `option_inputs_from_values` returns, for each option, only the values whose field ids
  that option declares. An option that declares nothing maps to `{}`. This is what makes
  two AND members with mutually exclusive `additionalProperties: false` schemas both
  answerable from one collected set.

Re-export the four functions from `src/sase/notification_gates/__init__.py` so
`sase-telegram` imports them from the package root.

`inputs-ace` (`sase-h7.6`) and `gate-cli` (`sase-h7.9`) are in flight and may land
overlapping helpers. This is a new module, so the worst case is a duplicate for the land
agent to reconcile rather than a conflicting edit; record that in a bead note.

## 6. `sase-telegram` — the declared-input step flow

Open with `/sase_repo` and use the printed path as the only path for reads and writes.
Telegram has no `InputArg` usage today, so this is new surface. Model it on
`question_flow.py`, which already splits a multi-step chat conversation into a pure
decision module plus a script that performs the Telegram calls.

**State.** Extend `GateProgress` in `src/sase_telegram/gate_flow.py` with
`input_option_ids: tuple[str, ...] = ()`, `input_field_index: int | None = None`, and
`input_values: dict[str, Any] | None = None`, persisted by `save_progress` and recovered
defensively by `load_progress` exactly as the existing fields are (a malformed or stale
block resets the input flow rather than raising). It lives in the bundle-local
`telegram_gate_progress.json`, which both the callback path and the text-reply path can
reach from the pending action's `action_data`; the `awaiting_feedback.json` map keeps
its present job of routing a plain text reply to the right gate.

**New `src/sase_telegram/gate_inputs.py`** — pure decisions, no Telegram calls:

- `pending_fields(view, selected_option_ids)` → the shared `collected_input_fields` over
  the selected options.
- `unsupported_fields(fields)` → the fields Telegram refuses (see below).
- `current_step(view, progress)` → the field being collected, its 1-based position, and
  the total, or `None` when collection is complete.
- `apply_text_answer(field, text)` / `apply_choice(field, value)` → the next
  `input_values` mapping, raising `XPromptValidationError` with the shared message on a
  bad answer.
- `skip_step(field)` → advance past an optional field, leaving its id unset so the
  compiled schema's `required` list stays the enforcement layer.
- `submitted_option_inputs(view, progress)` → `option_inputs_from_values` over the
  selection.

**Prompting.** After a branch submit resolves a selection and before the existing
feedback step, if `pending_fields` is non-empty the script sends one message per field:
the label, the declared type, `help` when set, and `placeholder` as the message's
example. `enum` fields carry an inline keyboard of their choices; every other type is
answered with a text reply. An optional field's keyboard carries **Skip**; every step
carries **Cancel**, which clears the input block and leaves the gate pending and
answerable. A `repeatable` non-enum field says "one value per line"; a `repeatable` enum
renders its choices as ☑️/⬜ toggles with a **Done** button, reusing the AND-group
toggle idiom already in `render_gate_keyboard`.

**Callback tokens.** Callback data is `gate:<prefix>:<choice>` under a 64-byte ceiling,
so the choice tokens stay compact and are decoded in `gate_flow.py` beside the existing
`c`/`s`/`f`/`x` tokens: `i<field>v<choice>` selects an enum value, `i<field>k` skips,
`i<field>d` is done with a repeatable enum, and `i<field>c` cancels.

**Text answers.** Add `_handle_gate_input_text_message(message, text, reply_key)` to
`scripts/sase_tg_inbound.py`, dispatched from `_handle_text_message` immediately after
`_handle_question_text_message` and before `process_text_message`, mirroring that
precedent. It converts the answer with `coerce_field_text`, and on
`XPromptValidationError` replies with the message and re-prompts the same field rather
than dropping the answer or advancing.

**Refusals.** Telegram declines two declarations rather than half-supporting them, and
says which and why in the callback answer:

- A `secret: true` field. A chat transport is not an acceptable channel for one, and the
  message would persist in Telegram's history well beyond the gate.
- Nothing else. A field with a raw `input_schema` and no declared `inputs` is not a
  refusal: it submits `{}` for that option exactly as Telegram does today.

**Submission.** `ResponseAction` gains `option_inputs: dict[str, Any] | None = None`.
`resolve_gate_response` passes `option_inputs=` and leaves `input_data` at `None` when
it is set — the two are mutually exclusive and supplying both raises
`conflicting_input`. Telegram uses the per-option path for every gate, including ones
that declare nothing (where it resolves to `{}` per option, which is what it sends
today), so there is one submission shape on this surface rather than two.

## 7. `sase-telegram` — delete the feedback heuristic

Delete `feedback_is_command_input` (`gate_flow.py:326-337`) and its use at
`scripts/sase_tg_inbound.py:1332`, and drop the `feedback_is_command_input` key from the
awaiting-feedback entry and from `process_text_message`'s injection at
`inbound.py:326-327`. The executor's `apply_feedback_input` owns that rule now, and it
tests `properties` rather than `required`, so an optional-feedback command that Telegram
silently starved starts receiving the note. Update `tests/test_inbound.py`'s three
`feedback_is_command_input` fixtures accordingly.

## Landing order and commits

`sase-core` first, then this repo's bridge, then `sase-telegram`. Commit each repo
separately with `/sase_git_commit`, with `feat(mobile)!:` on the `sase-core` commit (the
wire schema version bump matches the precedent of `df32b81`, the last bump) and a plain
`feat` on the other two. After committing a linked repo, re-check
`sase bead show sase-h7.8` before continuing: `sase-h7.3` recorded that `sase commit`
can auto-close the workspace's assigned phase bead on a linked-repo commit alone. Reopen
with `sase bead open sase-h7.8` if that happens.

## Tests

`tests/test_mobile_notifications_bridge.py`, beside the existing gate-action tests:

- A gate whose option declares `inputs` is answered through
  `execute_mobile_gate_action(..., option_inputs={...})` and its command's echoed result
  shows the typed values on stdin.
- Two selected AND members with mutually exclusive `additionalProperties: false` schemas
  each receive their own value.
- `option_inputs` omitted leaves today's shared-value behavior byte-identical, asserted
  against the existing expected `option_results`.
- The bridge accepts `schema_version: 4` and `5`, rejects `6` and `0`, rejects
  `option_inputs` at `4`, and rejects a non-object `option_inputs` with target
  `option_inputs`.
- A gate declaring a `secret` field submitted from mobile records `{"$redacted": true}`
  in `response.json` while the command saw the real value.

New `tests/test_gate_input_collection.py` for section 5: dedupe order across two
options; the conflicting-declaration error; `repeatable` newline splitting including a
blank line; enum rejection listing the allowed values; `option_inputs_from_values`
giving each option only its declared ids; an option declaring nothing mapping to `{}`.

`sase-telegram`, in `tests/test_gate_flow.py` and `tests/test_inbound.py` plus a new
`tests/test_gate_inputs.py`:

- The step sequence for a selection declaring a required `line`, an optional `enum`, and
  a `bool`: prompts in declared order, the enum keyboard carries one button per choice
  plus Skip, and the collected values submit as `option_inputs`.
- An invalid answer re-prompts the same field and leaves the gate pending with no
  `response.json`.
- Skip on an optional field omits the id; skip is not offered on a required one.
- A `repeatable` text field splits on newlines; a `repeatable` enum accumulates toggles
  and submits on Done.
- Cancel clears the input block and leaves the gate answerable.
- A `secret` field is refused with a pointed message and nothing is submitted.
- A gate declaring no inputs answers exactly as it does today, and an optional-feedback
  option receives the note through the executor now that the local heuristic is gone.
- Progress round-trips through `save_progress`/`load_progress`, and a malformed input
  block resets rather than raising.

`gate-cli` (`sase-h7.9`) owns `tests/gate_conformance/`, which does not exist yet. Write
the Telegram cases above as ordinary tests and record a `PROPOSED FOLLOW-UP` to fold
them into the matrix's Telegram leg once that phase lands, rather than inventing a
fixture format that phase will have to replace.

## Verification

1. `sase-core`: `cargo test` for `sase_core`, `sase_gateway`, and `sase_xprompt_lsp`,
   and `cargo fmt`. `committed_contract_snapshot_is_current` must pass against the
   regenerated snapshot, not a hand-edited one.
2. This repo: `just install` first — the workspace may have drifted, and that recipe is
   what rebuilds `sase_core_rs` from the linked `sase-core` checkout, so it is also the
   check that the Rust edit compiles into the binding. Then `just check-full`, not
   `just check`: this phase touches the executor's callers and the mobile wire contract,
   which is in the broadening set.
3. `sase-telegram`: `just install`, then install this workspace's `sase` over the PyPI
   copy the way CI does —
   `uv pip install --python .venv/bin/python --no-deps -e <workspace sase path>` — then
   `just check`. Without that step the venv holds released `sase` 0.16.0, whose
   `execute_gate_selection` has no `option_inputs` parameter and whose `GateOption` has
   no `inputs`, and every new test fails for a reason that has nothing to do with the
   change.

## Risks

- **The v3 envelope fix is the highest-value line in the phase and the easiest to
  lose.** It is a one-line accept-set change buried in a wire-shape commit. The
  assertion that a `schema_version: 3` envelope yields a non-empty branch list is what
  keeps it from regressing; write that test before the fix.
- **The contract snapshot fails loudly, which is the design.** Regenerate it in the same
  commit as the struct and `contract.rs` change. Never hand-edit the committed JSON —
  its keys are serialized in sorted order and a hand edit will diverge in whitespace or
  ordering even when the content is right.
- **Dropping `Eq` is wider than it looks.** It propagates through `GateOptionWire` →
  `GateBranchWire` → `MobileActionDetailWire` → `MobileNotificationDetailResponseWire`.
  `cargo check` finds every site; do not work around it by encoding `default` as a
  string, which would make an array or boolean default ambiguous on the wire.
- **The schema-version acceptance rule is a compatibility decision, not a detail.**
  Bumping the Python bridge's exact-equality check to `5` without widening it would
  break every app still sending `4` the moment the host updates. The widened rule and
  the Rust `validate_schema` range must stay in sync; the comment in section 4 is what
  keeps them that way.
- **Telegram carries reviewer text through a persistent chat log.** That is why `secret`
  is refused rather than masked: masking is a display property and Telegram's history is
  not ours to redact.
- **`inputs-ace` and `gate-cli` are in flight against the same helper surface.** Keeping
  section 5 in a new module bounds the collision to a duplicate rather than a
  conflicting edit, and the land agent reconciles it.
