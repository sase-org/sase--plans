---
tier: epic
title: Gate input collection and repeatable non-terminal gate actions
goal: "A reviewer can supply typed, validated input to any gate command from every
  surface, and can run repeatable non-terminal gate actions — starting with `edit_file`
  — that help them decide without answering the gate. Custom gates that declare required
  input become answerable instead of silently stuck, the three incompatible feedback
  rules collapse into one, and the free-text smuggling that snooze and triage rely on is
  retired.

  "
phases:
  - id: executor-integrity
    title: Diagnosable input failures and non-destructive retry
    depends_on: []
    size: medium
    description: "executor-integrity: move gate input-schema validation inside the
      error-recording path, bound input size and depth, and add a per-attempt execution
      journal so a partially executed AND branch can be resumed or restarted
      deliberately instead of silently re-running its completed commands.

      "
  - id: feedback-input
    title: One feedback-to-input rule for every surface
    depends_on: []
    size: medium
    description: "feedback-input: replace ACE's never-copy, mobile's option-id
      heuristic, and Telegram's required-list heuristic with one shared helper that
      injects the reviewer's note as `input.feedback` whenever the selected option's
      schema declares a `feedback` property, after auditing every built-in schema.

      "
  - id: inputs-core
    title: Declarative per-option inputs and per-option submission
    depends_on: []
    size: large
    description: "inputs-core: add the closed per-option `inputs:` authoring vocabulary
      built on `InputArg`/`InputType` (plus a new `enum` type), compile it into the
      option's `input_schema` at creation, and teach the executor to accept and persist
      one input value per selected option instead of one shared blob.

      "
  - id: gate-actions
    title: Repeatable non-terminal gate actions
    depends_on:
      - executor-integrity
    size: medium
    description: "gate-actions: generalize `operations` into a rendered vocabulary of
      repeatable actions that never answer the gate — `edit_file` gains presentation and
      an origin-file edit target, and a new `run_command` kind runs a hashed bundle
      command whose output is shown to the reviewer.

      "
  - id: custom-validation
    title: Fail closed at creation for unanswerable gates
    depends_on:
      - inputs-core
    size: medium
    description: 'custom-validation: add the missing `kind_validation/custom.py`, reject
      at creation any option whose effective input schema cannot accept what a client
      can produce, pin the JSON Schema dialect, and make an omitted `input_schema` mean
      "no input" instead of the permissive empty schema.

      '
  - id: inputs-ace
    title: Generic typed input collection in the ACE gate modals
    depends_on:
      - inputs-core
    size: large
    description: "inputs-ace: extract the typed-field form out of `InputCollectionModal`
      into a reusable widget and render each selected option's declared inputs inside
      `GateBranchControls`, so plan, epic, custom, triage, and snooze gates all collect
      input with no per-kind code.

      "
  - id: gate-actions-ace
    title: Gate actions in the ACE modals and the plan edit round trip
    depends_on:
      - gate-actions
    size: medium
    description: "gate-actions-ace: render an Actions section in the shared gate modals,
      run `edit_file` in place without tearing down the modal, point the plan and epic
      edit action at the durable `~/.sase/plans/` file, and block submission while an
      unaccepted draft exists.

      "
  - id: inputs-remote
    title: Mobile wire and Telegram step flow for declared inputs
    depends_on:
      - feedback-input
      - inputs-core
    size: large
    description: "inputs-remote: extend the frozen `mobile_api_v1` gate contract in
      `sase-core` with declared input fields and per-option submitted values, update the
      Rust request struct, routes, and Python bridge, and add an input step flow to the
      Telegram gate conversation.

      "
  - id: gate-cli
    title: sase gate answer, act, and show
    depends_on:
      - gate-actions
      - inputs-core
    size: medium
    description: "gate-cli: add headless `answer`, `act`, and `show` subcommands to
      `sase gate` and build the cross-surface conformance fixture matrix that runs the
      same input cases through ACE, the mobile bridge, and the CLI.

      "
  - id: surface-input
    title: Show the input a gate asks for and the input it received
    depends_on:
      - gate-actions
      - inputs-core
    size: small
    description: "surface-input: render declared and submitted input in the gate detail
      pane and Gate Debug, and add `input`, `option_inputs`, `option_results`, and
      executed actions to `sase gate wait --json`.

      "
  - id: retire-smuggling
    title: Retire free-text smuggling from snooze, triage, and launch
    depends_on:
      - feedback-input
      - inputs-ace
      - inputs-remote
    size: medium
    description: "retire-smuggling: express snooze durations as declared `enum`/`line`
      inputs, delete the two `validate_selection` re-parsing special cases, and drop the
      launch gate's fake `feedback` option id now that feedback is an ordinary input
      field.

      "
  - id: docs
    title: Document the input and action contracts
    depends_on:
      - custom-validation
      - gate-actions-ace
      - gate-cli
      - retire-smuggling
      - surface-input
    size: small
    description:
      "docs: add gate-input and gate-action sections to `docs/notifications.md`, rewrite
      the `/sase_gate` skill's input guidance with a worked non-empty example, document
      the new `enum` xprompt input type, and regenerate the deployed skills."
proposed_by: bbugyi200.athena.v2
---

# Plan: Gate input collection and repeatable non-terminal gate actions

## Background

The research report `202608/gate_input_collection/gate_input_collection.md` in the
research sidecar establishes the core finding: **the transport for gate input already
exists and is sound; the collection layer does not exist.**

Every `GateOption` already carries an `input_schema`
(`src/sase/notification_gates/model_options.py:63`). The shared executor already
validates one JSON value against every selected option's schema and writes it to each
command's stdin as canonical JSON (`src/sase/notification_gates/executor.py:84-90`,
`:415`), persisting it verbatim into the write-once response (`:197-207`). Commands are
hashed at creation and executed with `shell=False` against `/proc/self/fd/N`, so user
input can never reach `argv` — that is deliberate and stays.

What is missing:

1. **No generic surface can produce a value.** ACE hardcodes `input_data={}` for every
   generic gate (`src/sase/ace/tui/actions/agents/_notification_custom_gate.py:55-63`).
   `GateBranchControls` composes exactly one text widget, `#gate-feedback-input`
   (`src/sase/ace/tui/modals/gate_branch_controls.py:162-163`), and its `Resolved`
   message carries only `selected_option_ids` and `feedback` (`:36-46`).
2. **A custom gate that declares required input is permanently unanswerable.** There is
   no `kind_validation/custom.py`, so creation accepts it; every ACE submission then
   fails `schema_validation_failed`; and because the validation loop sits _before_ the
   per-option `try` that calls `_record_execution_error` (`executor.py:84-136`), nothing
   is written to `errors/`, so pressing `d` shows nothing.
3. **Three surfaces disagree about feedback.** ACE never copies the note into input;
   mobile copies it iff the literal string `feedback` is among the selected option ids
   (`src/sase/integrations/_mobile_notification_actions.py:67-69`); Telegram copies it
   iff a selected option's `input_schema.required` lists `feedback`
   (`sase-telegram/src/sase_telegram/gate_flow.py:326-337`). The same gate answers
   differently depending on where you tap it.
4. **Every kind that needs a real argument bought it with a bespoke modal.** Plan
   (`src/sase/plan_approval_actions.py`, `approve_options_modal.py`), question
   (`user_question_modal.py`, 654 lines), HITL (`_notification_hitl_modal.py:177-190`),
   and launch — which invents a third option id whose only job is to carry a string
   (`src/sase/agent/launch_request_gate.py:141-148`). Snooze and triage instead smuggle
   structured data through the free-text field and re-parse it host-side
   (`src/sase/notification_gates/adapters.py:218-238`,
   `src/sase/bead/snooze_gate.py:373-388`,
   `src/sase/bead/_task_gate_response.py:123-140`).
5. **`operations` is a dead generic mechanism.** `_GateOperation` accepts only
   `{id, kind, target}` with `kind == "edit_file"`
   (`src/sase/notification_gates/model_request.py:25-50`).
   `src/sase/plan_gate.py:155-161` is the sole producer and
   `kind_validation/plan.py:108-120` pins its exact shape. No generic surface renders
   it, so a custom gate may declare one and nothing happens.

Two corrections to the report's framing, verified in this checkout:

- `edit_file` is **not** tale-plan-only at the bundle layer. `_build_plan_gate_spec`
  declares the `edit_plan` operation for both tiers, `_validate_plan_operations`
  requires it for both, and `EpicApproval` routes to the same `PlanApprovalModal`
  (`src/sase/ace/tui/actions/agents/_notification_modal_flow.py:96`,`:192`). The real
  gaps are that the mechanism is hardcoded per kind rather than declared, that the edit
  tears the modal down and rebuilds it (losing branch selection, feedback text, and
  scroll position), and that it edits the **bundle copy** rather than the durable file.
- The editor today opens `notification.files[0]`, which
  `src/sase/notification_gates/service.py:326-329` resolves to `<bundle>/plan.md`, not
  the `~/.sase/plans/` archive that `sase plan propose` wrote
  (`src/sase/main/plan_propose_handler.py:167-170`). The archive is only rewritten from
  the bundle _after_ approval, by `_sync_reviewed_plan_to_durable_best_effort`
  (`src/sase/plan_approval_actions.py:370-392`).

## Goals

1. One closed, declarative `inputs:` vocabulary, declared per option and submitted per
   option, rendered generically by the shared branch controls on every surface.
2. `input_schema` stays the enforcement layer: `inputs` compiles into it at creation, so
   the executor contract is untouched and old readers keep enforcing.
3. One feedback-to-input rule, in shared code, deleting the other two.
4. A gate that cannot be answered is a creation error with a pointed message, never an
   accepted request that dies silently at submit time.
5. A new class of gate command that is **repeatable** and **never answers the gate**,
   generically rendered, with `edit_file` as its first member — so a reviewer can
   iterate toward a decision instead of being forced to answer or cancel.
6. Plan and epic gates edit the durable `~/.sase/plans/` file, accept the edit only when
   `sase plan validate` passes, and never silently discard a rejected draft.
7. Every input failure is diagnosable in one keystroke, and a partially executed AND
   branch is never silently re-run.

## Non-goals

- **User input never flows into `argv`.** Templating or appending reviewer-supplied
  arguments would break the hashed-command trust model that makes gates safe for
  dangerous operations. Stdin JSON is the channel. Do not add an "extra args" box.
- No general Draft 2020-12 schema-to-form renderer. `question_response_schema()` really
  does emit `allOf`/`if`/`then`/`prefixItems`; a renderer for that is a rabbit hole. The
  closed vocabulary plus a raw-YAML escape hatch is the contract.
- The `question` gate keeps its bespoke 654-line form modal for now. It is the richest
  input UI in the system and migrating it is a separate project; this epic must not
  regress it.
- No fourth form system. `SchemaObjectForm` (`schema_object_form.py`, coupled to
  `sase.config` layering types) stays where it is; only its YAML-fallback _pattern_ is
  borrowed.

## Rust core backend boundary

Submission normalization that ACE, mobile, Telegram, and the CLI must all agree on is
core backend behavior by the boundary rule's own litmus test. This epic keeps the
authoritative implementation in Python for the first pass — it is where every existing
surface already calls in, and splitting it mid-epic would double the wire design — but
`inputs-remote` must land the **wire contract** in `sase-core`
(`crates/sase_gateway/contracts/api_v1/mobile_api_v1.json` is freeze-tested by
`contract.rs::committed_contract_snapshot_is_current`), and the phase must record a
follow-up note proposing the normalization move once the contract is stable.

## Phase `executor-integrity` — Diagnosable input failures and non-destructive retry

All paths under `src/sase/notification_gates/`.

**1. Record input-validation failures.** In `executor.py`, the loop at `:84-90` raises
before the per-option `try` that calls `_record_execution_error`, so a rejected input
leaves no trace in `errors/`. Wrap it:

```python
for option in selected:
    try:
        _validate_json_instance(
            _option_input(option), option.input_schema, f"option {option.id} input"
        )
    except GateError as exc:
        _record_execution_error(
            bundle_path, option_id=option.id, code=exc.code,
            message=str(exc), source=source,
        )
        raise
```

Do the same for `_normalize_feedback` and `adapter.validate_selection` failures, which
have the same blind spot. Every gate rejection must be visible under `d`.

**2. Bound the input.** Add `_check_input_bounds(value, target)` called immediately
before validation: canonical JSON at most 64 KiB, nesting depth at most 16, at most 128
properties in any single object, at most 512 items in any array. Raise
`GateError("input_too_large", ...)`, recorded like every other failure. Input lands in
durable `response.json` and on a command's stdin; unbounded is not acceptable now that
users author it.

**3. Execution journal.** Add `journal.jsonl` to the bundle, appended under the existing
`.response.lock`. One record per attempt boundary:

```json
{"schema_version": 1, "attempt_id": "<uuid4>", "request_hash": "<sha256>",
 "event": "attempt_started" | "option_completed" | "option_failed" | "attempt_completed",
 "option_id": "restart", "input_digest": "<sha256 of canonical option input>",
 "result_digest": "<sha256 of canonical result>", "code": null, "at_unix": 0.0}
```

Never record raw input values — the digest is what a retry needs, and secret-typed
fields (added in `inputs-core`) must not leak here.

**4. Deliberate retry.** `execute_gate_selection` gains
`retry: Literal["resume", "restart"] | None = None`. On entry, read the journal:

- No prior incomplete attempt → proceed as today; `retry` must be `None`.
- An incomplete attempt exists for the **same** `request_hash` **and** the same
  per-option input digests, and `retry is None` → raise
  `GateError("partial_attempt", "<attempt_id>", ...)` naming the completed and failed
  option ids. Never silently re-run.
- `retry="resume"` → skip options recorded `option_completed`, replay their persisted
  results into `option_results`, and start at the failed option.
- `retry="restart"` → run the whole branch again, recording a fresh `attempt_id`.
- Inputs or request hash differ from the incomplete attempt → treat as a new attempt and
  supersede the old one, recording an `attempt_superseded` event.

`_GateTaskOutcome` in `src/sase/ace/tui/actions/agents/_notification_gate_execution.py`
gains a `partial_attempt` variant so ACE can present the choice rather than guessing;
`gate-cli` and `gate-actions-ace` consume it.

**5. Document the idempotency expectation.** Add it to `GateAdapter`'s and
`execute_gate_selection`'s docstrings: commands in an AND branch must tolerate being run
again after a later member fails, because `restart` is a supported reviewer choice.

Tests (`tests/test_gate_executor_integrity.py`, new):

- A required-property schema with `{}` submitted writes exactly one `errors/*.json` with
  `code == "schema_validation_failed"` and leaves the gate pending.
- Oversized, too-deep, and too-wide inputs each raise `input_too_large` and record it.
- A two-option AND branch whose second command exits non-zero leaves a journal with one
  `option_completed` and one `option_failed`; a bare retry raises `partial_attempt`;
  `retry="resume"` runs only the second command; `retry="restart"` runs both.
- Changing the input between attempts supersedes the incomplete attempt rather than
  resuming it.

## Phase `feedback-input` — One feedback-to-input rule for every surface

**The rule.** The reviewer's note is injected as `input.feedback` for a selected option
**iff that option's effective `input_schema` declares a `feedback` property.** This is
Telegram's `feedback_is_command_input()` generalized from `required` to `properties`,
which is what makes it work for optional-feedback commands.

**Where it lives.** New `src/sase/notification_gates/feedback_input.py`:

```python
def option_accepts_feedback_input(option: GateOption) -> bool:
    """Whether this option's command should receive the reviewer note as input."""

def apply_feedback_input(
    options: Sequence[GateOption], option_inputs: Mapping[str, Any], feedback: str | None
) -> dict[str, Any]:
    """Return per-option inputs with the note injected wherever the schema declares it."""
```

Applied inside `execute_gate_selection`, **not** per surface, so every caller —
including `sase gate answer` — gets it for free. Surfaces stop assembling it.

**Deletions after this lands:**

- `src/sase/integrations/_mobile_notification_actions.py:67-69` — the
  `"feedback" in selected_option_ids` heuristic.
- `sase-telegram/src/sase_telegram/gate_flow.py:326-337` `feedback_is_command_input()`
  and its use in `inbound.py:324-336` (this deletion is scheduled in `inputs-remote`,
  which is the phase that touches that repo; leave a `PROPOSED FOLLOW-UP:` note here if
  the ordering makes it awkward).
- `_notification_custom_gate.py:61`'s `input_data={}` becomes "send no input"; the
  executor injects feedback.

**Behavior-change audit — do this before writing code.** Generalizing `required` to
`properties` changes behavior for existing gates on all three surfaces. Enumerate every
built-in schema that has a `feedback` property and confirm each still validates when the
note is injected and when it is absent:

- `src/sase/agent/launch_request_gate.py:109-114` — `feedback` is `required` with
  `minLength: 1` and `additionalProperties: false`.
- `src/sase/plan_gate.py:551-578` — the AND-intersection comment lives here.
- `src/sase/bead/snooze_gate.py` and `src/sase/bead/task_gate.py` — commands currently
  assert **empty** input; injecting `feedback` would break them. Either add the property
  to their schemas in this phase or leave them out of the rule; `retire-smuggling`
  replaces both anyway.

**Compatibility shim, explicitly labelled.** Mark `feedback_input.py` in its module
docstring as a compatibility shim with a stated deletion trigger: once `inputs-core` and
`retire-smuggling` land, `feedback` is an ordinary declared input field and this module
goes. Add the note to the module, not only to this plan.

Tests (`tests/test_gate_feedback_input.py`, new): a schema with `feedback` under
`properties` but not `required` receives the note; a schema without the property does
not and the note stays only in `response.feedback`; a `required` `feedback` with no note
fails with `feedback_required` before any command runs.

## Phase `inputs-core` — Declarative per-option inputs and per-option submission

This is the headline contract and warrants its own planning pass. Everything below is
the required shape, not a suggestion.

**1. Extend the shared vocabulary rather than forking one.** In
`src/sase/xprompt/models.py`:

- Add `InputType.ENUM = "enum"`.
- Add `InputArg.choices: tuple[InputChoice, ...] = ()` where
  `InputChoice = (value: str, label: str | None)`. Required non-empty for `ENUM`,
  rejected for every other type.
- Teach `InputArg.validate_and_convert` to accept only declared choice values for
  `ENUM`, raising `XPromptValidationError` with the allowed values listed.
- Wire `choices` through the xprompt frontmatter parser so `input:` args can declare
  enums too. This is the point of reusing the vocabulary: one type system, two
  consumers.

**2. The gate authoring layer.** New `src/sase/notification_gates/model_inputs.py`:

```python
@dataclass(frozen=True)
class GateInputField:
    id: str                  # [a-z][a-z0-9_]* ; JSON property name
    label: str
    type: InputType          # word|line|text|path|agent|int|bool|float|enum
    required: bool = False
    default: Any = None
    choices: tuple[InputChoice, ...] = ()
    placeholder: str | None = None
    help: str | None = None
    secret: bool = False
    repeatable: bool = False
```

`GateOption` gains `inputs: tuple[GateInputField, ...] = ()`, and `inputs` joins the
closed field set in `GateOption.from_mapping` (`model_options.py:79-92`) and `to_dict`.
`GateBranchData.from_envelope` re-parses it through the same code, so every reader sees
the same fields.

**3. Compilation, not a second enforcement layer.** At creation, `inputs` compiles to a
JSON Schema object and that compiled schema is **stored as the option's `input_schema`
in the envelope**. The executor is untouched, and a reader that knows nothing about
`inputs` still enforces correctly. Mapping:

| `InputType`        | Compiled schema fragment                                     |
| ------------------ | ------------------------------------------------------------ |
| `word`, `agent`    | `{"type": "string", "minLength": 1, "pattern": "^\\S+$"}`    |
| `line`             | `{"type": "string", "pattern": "^[^\\n]*$"}`                 |
| `text`             | `{"type": "string"}`                                         |
| `path`             | `{"type": "string", "minLength": 1, "pattern": "^[^\\n]*$"}` |
| `int`              | `{"type": "integer"}`                                        |
| `float`            | `{"type": "number"}`                                         |
| `bool`             | `{"type": "boolean"}`                                        |
| `enum`             | `{"enum": [<declared values>]}`                              |
| `repeatable: true` | wraps the above in `{"type": "array", "items": {...}}`       |

The compiled object is
`{"$schema": "https://json-schema.org/draft/2020-12/schema", "type": "object", "properties": {...}, "required": [<required ids>], "additionalProperties": false}`,
plus an optional `"feedback": {"type": "string"}` property whenever the option's
feedback mode is not `disabled`, so `feedback-input`'s rule works mechanically without
any author action.

**`inputs` and `input_schema` are mutually exclusive per option.** Declaring both is
`GateError("conflicting_input_declaration", ...)`. A raw `input_schema` remains
available for authors who need a shape the vocabulary cannot express; it is not composed
with `inputs`. This is what keeps the contract legible.

**4. Per-option submission.** `execute_gate_selection` gains
`option_inputs: Mapping[str, object] | None = None`:

- `option_inputs` given → each selected option is validated against **its own** schema
  with **its own** value, and each command receives its own value on stdin. This removes
  the "every AND member must admit every other member's fields" workaround that
  `plan_gate.py:551-578` documents in a comment.
- Only the legacy `input_data` given → today's shared-value behavior, unchanged, for
  every v3 bundle already on disk.
- Both given → `GateError("conflicting_input", ...)`.

`response.json` always records `"option_inputs": {option_id: value}` — under the legacy
shared path every entry is the same value — and keeps `"input"` with its current meaning
so nothing that reads it today breaks. This is additive; `GATE_RESPONSE_SCHEMA_VERSION`
stays `2` and readers must tolerate the key's absence on older responses.

**5. Secrets.** A field with `secret: true` reaches the command's stdin but is written
to `response.json` and `journal.jsonl` as `{"$redacted": true}`. Ship the redaction with
the field type; do not offer `secret` without it. Masked display alone is insufficient
because the response file is durable audit data.

**6. Compatibility.** `schema_version` stays `3`. `inputs` is an optional additive
option field, so bundles created before this phase parse unchanged, and bundles created
after it are readable by the compiled `input_schema` alone.

Tests (`tests/test_gate_inputs_core.py`, new): compilation of every type including
`repeatable` and `enum`; both-declared rejection; per-option submission delivering
different values to two AND members with mutually exclusive
`additionalProperties: false` schemas; legacy shared-value path unchanged; secret
redaction in the response and journal but not on stdin; `InputType.ENUM` round-tripping
through an xprompt `input:` declaration.

## Phase `gate-actions` — Repeatable non-terminal gate actions

The mechanism the user asked for: a gate command that the reviewer may run **as many
times as they need** and that **never dismisses the gate**. It is what makes "edit the
plan until it validates, then approve" work, and it generalizes to anything that helps a
reviewer decide — a diff view, a dry run, a health check, a rendered summary.

**Name and placement.** Reuse the existing `operations` array rather than inventing a
parallel concept: `_GateOperation` is already documented as "a non-terminal operation
supported by a local gate surface". Keeping the field name keeps `schema_version: 3`.
Surfaces present them as **Actions**, distinct from the **Decision** section.

**Two kinds.** Extend `_GateOperation.from_mapping`
(`src/sase/notification_gates/model_request.py:33-47`) — its closed field set becomes
`{id, kind, label, icon, key, description, target, edit_target, command, input_schema, result_schema, targets, display}`
with kind-specific requirements:

```jsonc
// edit_file — existing kind, now presentable and origin-aware
{"id": "edit_plan", "kind": "edit_file", "target": "plan.md",
 "edit_target": "origin",              // NEW: "resource" (default) | "origin"
 "label": "Edit plan", "icon": "✏️", "key": "e",
 "description": "Accepted only when `sase plan validate` passes."}

// run_command — new kind
{"id": "show_diff", "kind": "run_command",
 "command": {"argv": ["commands/show_diff"]},
 "label": "Show diff", "icon": "\U0001f50d", "key": "D",
 "input_schema": {}, "result_schema": {},
 "targets": [],                        // editable resources this action may rewrite
 "display": "markdown"}                // "markdown" | "text" | "json" | "none"
```

`label` defaults from `id` when omitted; `icon` and `description` are optional; `key` is
optional and validated (see below). `run_command`'s `argv[0]` must name an executable
`command` resource — extend the ownership check in
`src/sase/notification_gates/validation.py:74-90`, which currently only checks option
commands and only checks `operation.target`.

**Execution.** New
`execute_gate_operation(bundle_path, operation_id, *, input_data=None, source="host", on_output_line=None, ...) -> GateOperationResult`
in `executor.py`, reusing `_run_owned_command` verbatim, so the trust model is
identical: hash re-verified against the envelope, `shell=False`, `/proc/self/fd/N`,
canonical JSON on stdin. It differs from option execution in exactly four ways, and
these are the whole point:

1. It never writes `response.json` and never calls `_settle_gate_notification`. The gate
   stays pending and answerable.
2. It may be called any number of times.
3. It refuses to run when `response.json` or `cancellation.json` exists.
4. After the command exits it re-hashes **every** resource. Paths listed in `targets`
   are revalidated through `adapter.validate_edited_resource` and re-hashed exactly as
   `refresh_gate_after_edit` does; a change to any resource **not** listed is
   `GateError("hash_mismatch", ...)` and the action is reported as failed. This closes
   the hole where a display command could silently rewrite the reviewed command.

Failures are recorded through `_record_execution_error` with the operation id, so `d`
shows them. Successful and failed runs both append to the `executor-integrity` journal
as `operation_ran` events; the journal is the audit trail for "what did the reviewer run
before deciding".

**Result contract.** `run_command` stdout is one JSON value validated against the
operation's `result_schema`. The renderer reads a small closed display record from it,
and ignores everything else (which stays in the journal):

````json
{ "summary": "3 files changed", "body": "```diff\n...\n```", "refresh": false }
````

`summary` becomes a one-line toast; `body` is rendered with the declared `display`
format in a scrollable output pane; `refresh: true` tells the surface to reload and
re-verify the bundle before re-rendering.

**`edit_file` generalization.** `refresh_gate_after_edit`
(`src/sase/notification_gates/service.py:422-483`) already does the right thing for the
bundle resource. Add:

- `GateResource.envelope_dict()` records `"origin": "<absolute source path>"` when the
  resource was created from `source:` rather than inline `content:`. This is the generic
  mechanism by which `edit_target: "origin"` knows what to open; it does not exist today
  because `envelope_dict` drops `source`.
- `resolve_edit_path(bundle_path, envelope, operation_id) -> EditTarget` returning both
  the path to open and the bundle resource it maps to, falling back to the bundle
  resource when `edit_target == "origin"` but no origin is recorded (legacy bundles) or
  the origin file is missing.
- `accept_edited_origin(bundle_path, operation_id) -> dict[str, Any]`: copy origin →
  bundle resource, run the existing `refresh_gate_after_edit` validation, and **on
  failure restore the bundle resource to its pre-copy bytes** so the gate keeps its last
  accepted reviewed revision. The user's draft stays in the origin file, so re-opening
  the editor resumes their work rather than discarding it.
- `origin_draft_state(bundle_path, operation_id) -> Literal["clean", "draft", "missing"]`:
  compare the origin file's hash to the reviewed resource hash, so a surface can tell
  the reviewer that an edit — theirs, from this gate or from any other editor — was
  never accepted.

**Key validation.** A declared `key` must be a single printable character not in the
reserved set: the static modal bindings (`escape`, `q`, `d`, `g`, `G`, `ctrl+d`,
`ctrl+u`) and the digit branch selectors `1`-`9`. Creation rejects a collision with a
pointed message; two operations may not request the same key. Collisions with
_user-configured_ `GateModalKeymaps` values cannot be known at creation time and are
resolved at render time by `gate-actions-ace`, which reassigns from a deterministic
fallback pool and displays the key it actually bound. Keep the reserved list in one
module shared by validation and the modal so the two cannot drift.

**Plan and epic gates.** `src/sase/plan_gate.py:155-161` grows the new fields —
`edit_target: "origin"`, `label: "Edit plan"` (`"Edit epic plan"` for `epic_plan`),
`icon`, `key: "e"` — and `kind_validation/plan.py:108-120` is updated to require exactly
that shape **for both tiers**, so an epic gate's edit action is a first-class declared
action rather than a hardcoded modal binding. The tier-aware validation
(`adapters.py:240-249` → `require_plan_approval_validation(path, "epic" | "tale")`) is
already correct and is what enforces "only accept the edits if `sase plan validate`
passes".

Tests (`tests/test_gate_operations.py`, new): a `run_command` action runs repeatedly and
leaves the gate pending and answerable; it cannot run after a response exists; a command
that rewrites an undeclared resource fails with `hash_mismatch` and is recorded in
`errors/`; a command that rewrites a declared `targets` resource re-hashes it and bumps
`review_revision`; `edit_target: "origin"` resolves to the recorded origin and falls
back when it is absent; a failed `accept_edited_origin` restores the bundle bytes and
leaves `review_revision` unchanged while `origin_draft_state` reports `draft`; both plan
tiers validate their new operation shape.

## Phase `custom-validation` — Fail closed at creation for unanswerable gates

New `src/sase/notification_gates/kind_validation/custom.py`, registered in
`kind_validation/__init__.py` and dispatched from `validation.py:219-227`.

**1. Answerability.** For every option, build the value a client can actually produce —
`{}` plus every declared `inputs` default plus `feedback` when the mode allows it — and
validate it against the effective `input_schema`. On failure raise
`GateError("unanswerable_option", "options[i].input_schema", ...)` naming the offending
required property and telling the author to declare it under `inputs`. This is the check
whose absence created the silent trap.

**2. Omission means "no input".** When an option declares neither `inputs` nor
`input_schema`, creation stores
`{"$schema": ..., "type": "object", "additionalProperties": false}` — the honest
default, matching every built-in that passes `{}` today. An author who genuinely wants
the permissive schema writes `"input_schema": {}` explicitly and gets it. Apply this
only to **new** creations; bundles already on disk are unaffected because the stored
schema is what the executor reads.

**3. Dialect.** Reject a declared `$schema` that is not Draft 2020-12; stamp the dialect
on every stored schema. Extend `check_json_schema` in `model_validation.py` accordingly.

**4. Declared defaults.** Validate each `GateInputField.default` against that field's
own compiled fragment, so a default that can never be submitted is a creation error.

**5. Bounds at creation.** Reject a compiled or declared schema with more than 128
properties or nesting deeper than 16, matching the runtime bounds from
`executor-integrity` so an author learns at creation rather than at submission.

**6. `format` is annotation-only.** The executor builds `Draft202012Validator` with no
`FormatChecker`; that stays. Say so in the error message when a schema declares `format`
on a required property, and say so in the docs phase.

Tests (`tests/test_gate_custom_validation.py`, new): a required property with no
matching `inputs` field is rejected at creation with a message naming the property; the
same gate declaring that field under `inputs` is accepted and answerable end to end;
omitted schema rejects a non-empty input; explicit `{}` still accepts anything; a
non-2020-12 `$schema` is rejected; a default that violates its own field is rejected.

## Phase `inputs-ace` — Generic typed input collection in the ACE gate modals

**1. Extract the form.** `input_collection_modal.py` (566 lines) is coupled to
`PromptInputPlan`. Extract its typed-field machinery — `_InputCollectionInput`,
`_PathField` with its `ctrl+t` completion, per-field validation against
`InputArg.validate_and_convert`, the required/optional reveal, and the confirm-enable
logic — into a reusable `src/sase/ace/tui/widgets/typed_input_form.py` widget driven by
a `Sequence[InputArg]`. `InputCollectionModal` then composes that widget and keeps its
placeholder handling, so xprompt input collection is unchanged. Add an `enum` editor (a
cycling select honoring the declared choice labels) for the new `InputType.ENUM`.

Guard the refactor with the existing xprompt input-collection tests before touching the
gate side; a regression here breaks prompt launching, which is far more used than gates.

**2. Render inputs in `GateBranchControls`.** In
`src/sase/ace/tui/modals/gate_branch_controls.py`, mount one `TypedInputForm` per
branch, below that branch's controls and above the shared feedback field, showing the
union of the currently selected options' declared fields. It reveals and hides as
toggles change, and the branch's submit control disables while any required field is
empty or invalid — the same rule the feedback field already uses at `:392-400`.

`GateBranchControls.Resolved` gains `option_inputs: Mapping[str, dict[str, Any]]`, built
mechanically from field ids. `CustomGateModalResult` and `PlanApprovalResult` carry it
through, and `_notification_custom_gate.py:55-63` stops hardcoding `input_data={}` and
passes `option_inputs` instead. `GateSubmission` gains the same field.

Fields shared by two selected AND members are collected once and delivered to both.

**3. Raw escape hatch.** An option with a raw `input_schema` (no `inputs`) that is not
the empty object renders a single YAML text area seeded with the schema's defaults,
validated against the schema on change — the `SchemaObjectForm` fallback pattern
(`schema_object_form.py:505-524`), not its config-layering model. An option with no
declared input renders nothing at all.

**4. Presentation.** Secret fields render masked and are excluded from any copy action.
The section header reads **Inputs**; keep it visually subordinate to **Decision** so the
decision remains the focal point. Update `_footer_text` in both
`custom_gate_modal.py:269-280` and `plan_approval_modal.py:327-355`, and update the ACE
help modal per `src/sase/ace/CLAUDE.md`.

Tests: widget tests under `tests/ace/tui/` for reveal-on-toggle, required-blocks-submit,
per-option delivery of divergent values, enum cycling, masked secrets, and the raw-YAML
fallback; plus a PNG snapshot for the inputs section (`just test-visual`, goldens under
`tests/ace/tui/visual/snapshots/png/`).

## Phase `gate-actions-ace` — Gate actions in the ACE modals and the plan edit round trip

**1. `GateActionControls`.** New widget in `src/sase/ace/tui/modals/`, composed by both
`CustomGateModal._compose_actions` and `PlanApprovalModal.compose`, rendering an
**Actions** section above **Decision**. Each action is a focusable button showing
`<key> <icon> <label>`, with `description` as its tooltip line. The section is omitted
entirely when the gate declares no operations, so today's gates look unchanged.

Bind each declared `key` as a modal-scoped binding. Resolve collisions with the user's
configured `GateModalKeymaps` at render time using the deterministic fallback pool, and
render the key actually bound. Include actions in `focus_next_control` /
`focus_previous_control` traversal so keyboard-only review works.

**2. `edit_file` in place.** Replace the dismiss-and-repush round trip in
`_notification_modals.py:179-217`. The action now:

1. Resolves the edit path via `resolve_edit_path` — for plan and epic gates that is the
   durable `~/.sase/plans/` file, because their operation declares
   `edit_target: "origin"`.
2. Runs `$EDITOR` under `app.suspend()` **without dismissing the modal**.
3. Calls `accept_edited_origin`. On success, reloads the reviewed content and re-renders
   the document pane in place; branch selection, toggled AND members, typed feedback,
   typed inputs, and scroll position all survive, which is what makes the action feel
   repeatable rather than destructive.
4. On failure, shows the validation diagnostics in an error toast with a 15-second
   timeout and leaves the gate at its previous accepted revision.

`PlanApprovalResult(action="edit")` and `PlanApprovalModal.action_edit` are deleted; `e`
stays the plan and epic edit key because the operation declares `key: "e"`.

**3. Draft banner.** On modal open — in the existing worker-thread load, never on the
event loop — compute `origin_draft_state`. When it is `draft`, show a banner above the
Decision section:

```
Draft not accepted — ~/.sase/plans/202608/foo.md differs from the reviewed plan.
Press e to fix it, or D to discard your draft.
```

**While a draft is unaccepted, every submit control is disabled.** This is the honest
behavior: approving would run the last accepted content and then overwrite the draft via
`_sync_reviewed_plan_to_durable_best_effort`
(`src/sase/plan_approval_actions.py:370-392`), silently destroying the reviewer's work.
Blocking is uniform across kinds and needs no per-kind reasoning. **Discard draft**
(`D`, confirmed) restores the origin file from the reviewed bundle copy and clears the
banner. This also catches an edit made outside the gate entirely, which today is
overwritten with no warning at all.

**4. `run_command` actions.** Execute through the existing tracked-task queue
(`submit_gate_execution_task`'s sibling), streaming stdout and stderr into the task
reporter exactly as option execution does. On completion: `summary` becomes a toast, and
a non-empty `body` opens a new `GateActionOutputModal` — scrollable, `q` to close, `y`
to copy, rendered per the declared `display` using `markdown_document_syntax` for
`markdown`. `refresh: true` reloads and re-verifies the bundle before re-rendering.

**5. Partial-attempt choice.** When submission raises `partial_attempt` from
`executor-integrity`, present a small modal naming the completed and failed options with
**Resume after the failed step** and **Run the whole branch again**, and resubmit with
the chosen `retry` value. Never guess.

Tests: widget tests for action rendering, key binding and fallback reassignment, and
traversal; an in-place edit test asserting branch selection and feedback survive; a
failed validation test asserting the banner appears, submits are disabled, and the
bundle bytes are unchanged; a discard test; a `run_command` test asserting the output
modal and that the gate is still pending afterward; PNG snapshots for the Actions
section and the draft banner.

## Phase `inputs-remote` — Mobile wire and Telegram step flow for declared inputs

**Mobile (`sase-core`, cross-repo).** Open the repo with `/sase_repo` and use the
printed path as the only path for reads and writes. The route is mobile app →
`sase_gateway` (Rust) → host bridge → Python `mobile_notifications.py` → executor, and
the wire snapshot is freeze-tested, so all of the following must land together:

1. `crates/sase_core/src/notifications/mobile.rs:328` — `GateActionRequestWire` gains
   `option_inputs: Option<BTreeMap<String, JsonValue>>` (`#[serde(default)]`).
2. A new `MobileGateInputFieldWire` mirroring `GateInputField`, modeled on the existing
   `MobileXpromptInputWire` in the same contract — the precedent the research identified
   — so the app can render gate inputs with the widget set it already has. Attach it to
   the existing `GateOptionWire`.
3. `crates/sase_gateway/src/contract.rs:840` — update the declared shapes, then
   regenerate `crates/sase_gateway/contracts/api_v1/mobile_api_v1.json` and confirm
   `contract.rs::committed_contract_snapshot_is_current` passes.
4. `crates/sase_gateway/src/routes.rs:1446-1478` and `host_bridge.rs:380,457,664` —
   thread the new field through.
5. Back in this repo: `src/sase/integrations/mobile_notifications.py:60-95` accepts and
   validates `option_inputs`, bumps `MOBILE_NOTIFICATION_WIRE_SCHEMA_VERSION` from `4`
   to `5`, and `_mobile_notification_actions.py` passes it to `execute_gate_selection`
   and drops the `"feedback" in selected_option_ids` heuristic.

An older app that omits `option_inputs` keeps working: the field defaults to absent and
the executor falls back to the shared-value path.

**Telegram (`sase-telegram`, cross-repo).** Open with `/sase_repo`. Telegram has no
`InputArg` usage at all today, so this is new surface:

1. Add a declared-input step flow to the gate conversation in `gate_flow.py` and
   `inbound.py`: after the reviewer picks a branch, prompt for each required field in
   declared order, with `enum` fields rendered as inline keyboards and everything else
   as a text reply validated through the shared `InputArg.validate_and_convert` rules.
   Optional fields are offered with a **Skip** control.
2. Submit `option_inputs`; delete `feedback_is_command_input()` (`gate_flow.py:326-337`)
   and its use at `inbound.py:324-336` — the executor now owns that rule.
3. Refuse to render `secret` fields over Telegram and say so; a chat transport is not an
   acceptable channel for one.
4. Add the Telegram leg of the `gate-cli` conformance matrix in that repo, using the
   same fixture set.

Commit each repo separately with `/sase_git_commit`, and state the required landing
order in the phase bead: `sase-core` first, then this repo's bridge, then
`sase-telegram`.

## Phase `gate-cli` — sase gate answer, act, and show

`sase gate` currently exposes only `create` and `wait` (`src/sase/main/parser_gate.py`),
so there is no headless way to supply input and no way to exercise a non-trivial
`input_schema` in a test. Add three subcommands, following `sase/memory/cli_rules.md` —
read it with `/sase_memory_read` before adding options.

```bash
sase gate answer --id <request-id> --kind custom --option restart --option verify \
  --set target_env=staging --option-input verify=@verify-input.json \
  --feedback "..." [--resume | --restart] [--json]

sase gate act --id <request-id> --kind plan --operation edit_plan [--json]

sase gate show --id <request-id> --kind custom [--json]
```

- `answer` — repeatable `--option`; `--set key=value` typed by the declared `inputs`
  field (string when the option uses a raw `input_schema`); `--input @file.json` or
  `--input -` for a whole shared value; `--option-input <opt>=@file.json` for a
  per-option value. `--resume` / `--restart` map to the `executor-integrity` retry
  parameter, and their absence on a partial attempt is a clear error naming both flags.
  Exit codes mirror `wait`: `0` answered, `1` usage or validation error, `3` cancelled.
- `act` — runs one declared non-terminal action headlessly and prints its `summary` and
  `body` (or the raw JSON under `--json`). For an `edit_file` action it opens `$EDITOR`
  and reports whether the edit was accepted. This is also the supported alternative to
  the forbidden "run the bundle command by hand".
- `show` — prints the declared branches, each option's declared input fields with types
  and defaults, and the declared actions. This is what an author uses to check that the
  gate they wrote asks for what they intended.

**Conformance fixture matrix.** New `tests/gate_conformance/` holding one fixture set
exercised through ACE (headless), the mobile bridge, and the CLI in this repo, and
through the Telegram adapter in `sase-telegram` (landed by `inputs-remote`). Cases: no
input; required, optional, and defaulted scalars; every `InputType` including `enum` and
`repeatable`; invalid input; two selected options with different inputs; feedback plus
input; a secret field; cancellation racing a submission; retry after a later-command
failure with both `resume` and `restart`. This matrix is what prevents the current
surface-specific drift from returning — say so in the module docstring.

Tests: per-subcommand CLI tests plus the matrix itself.

## Phase `surface-input` — Show the input a gate asks for and the input it received

Neither `src/sase/notification_gates/summary.py` nor `debug*.py` reads `input` today,
and `sase gate wait --json` emits only `status`, `selected_option_ids`, `feedback`, and
`response_path` (`src/sase/notifications/cli_wait.py:56-93`). A producing agent should
not have to open `response.json` to learn what the user chose.

1. `GateSummaryOption` gains `input_fields: tuple[GateInputField, ...]` and
   `submitted_input: dict[str, Any] | None`; `GateSummary` gains `option_inputs`. The
   notification detail pane shows, for a pending gate, which selected commands require
   which fields **before** execution, and for a terminal gate, the values that were
   submitted — with `secret` fields rendered as `••• (redacted)`.
2. Gate Debug (`debug.py`, `debug_rendering.py`) gains declared inputs per option in the
   overview and the submitted `option_inputs` in the response tab, plus the
   `journal.jsonl` attempts and executed actions as a new tab. Keep the existing rule
   that malformed or oversized artifacts render as diagnostics rather than blocking the
   modal.
3. `cli_wait._terminal_payload` gains `input`, `option_inputs`, `option_results`, and
   `operations` (executed non-terminal actions, from the journal). Existing keys keep
   their meaning and position; this is additive.

Tests: summary projection tests for pending and terminal gates including redaction; a
`wait --json` payload test asserting old keys are unchanged and new keys present; a Gate
Debug rendering test.

## Phase `retire-smuggling` — Retire free-text smuggling from snooze, triage, and launch

This is the acceptance test for the whole epic: the new mechanism is only strong enough
if it can retire the workaround it replaces.

1. **Snooze.** `src/sase/bead/snooze_gate.py` declares the duration as a real input
   field on the re-snooze option — an `enum` of the common durations plus a `line` field
   for a custom expression, or a single `line` field if the enum's ergonomics disappoint
   in review. The duration then reaches the command on stdin instead of being re-parsed
   host-side. Delete `validate_bead_snooze_feedback` (`snooze_gate.py:373-388`) and its
   dispatch in `adapters.py:231-234`, and the corresponding re-parse in
   `apply_side_effects`.
2. **Triage.** The same change for `src/sase/bead/task_gate.py` and
   `_task_gate_response.py:123-140`, deleting `validate_task_triage_feedback` and its
   dispatch at `adapters.py:235-238`.
3. **Launch.** `src/sase/agent/launch_request_gate.py:109-114` invents a third option id
   (`feedback`) whose only job is to carry a string, and
   `src/sase/agent/launch_approval_actions.py:141-148` silently rewrites a `reject`
   selection to `feedback` when text is present. With feedback an ordinary declared
   input on `reject`, delete the fake option, the query branch that carries it, and the
   rewrite. Confirm no persisted launch response reader depends on the `feedback` option
   id first; if one does, keep the id and note the removal as a follow-up rather than
   breaking it.
4. **Rewrite `GateAdapter.validate_selection`'s docstring** (`adapters.py:218-238`) — it
   currently documents the compromise this phase removes. If no kind still needs it
   after the three deletions, delete the method and its call site in `executor.py:92-95`
   outright.
5. **Delete the `feedback-input` shim** if `retire-smuggling` leaves no built-in relying
   on it, per that phase's stated deletion trigger. If any remain, record which ones and
   why in a `PROPOSED FOLLOW-UP:` note.

Tests: update the snooze and triage gate tests to submit the duration as declared input;
assert a malformed duration is rejected by the compiled schema at submission and
recorded in `errors/`, leaving the gate pending — the property the deleted host-side
check existed to provide; assert the launch reject path carries its note without the
fake option.

## Phase `docs` — Document the input and action contracts

1. **`docs/notifications.md`** — add a **Gate inputs** subsection under "Command-backed
   interaction gates" stating plainly that arguments are **stdin JSON fields, never
   argv**; how to declare `inputs`; the type table; how `inputs` compiles into
   `input_schema`; that omission now means "no input" while explicit `{}` stays
   permissive; the one feedback-to-input rule; per-option submission; secret redaction;
   and that `format` is annotation-only. Add a **Gate actions** subsection covering
   repeatable non-terminal actions, both kinds, the display record, and the reserved-key
   rules. Update the existing sentence at `docs/notifications.md:879-881` that calls
   plan editing "the one non-terminal operation" — it is no longer true. Document the
   durable `~/.sase/plans/` edit target and the unaccepted-draft block.
2. **`src/sase/xprompts/skills/sase_gate.md`** — replace the unexplained
   `"input_schema": {"type": "object"}` at `:117-119`, which is the copied-without-
   understanding signal that made the trap invisible. Add an inputs paragraph and extend
   the worked example with a real `inputs` declaration (including an `enum`) and one
   `run_command` action, so the next author sees the intended shape. Then run
   `sase skill init --force` to regenerate and deploy.
3. **`docs/prompt.md`** — document the new `enum` `InputType` and `choices` for xprompt
   `input:` args.
4. **`docs/mobile_gateway.md`** — document the wire version bump and the new fields.
5. **ACE help modal** — per `src/sase/ace/CLAUDE.md`, update the `?` popup for the new
   gate-modal keys, honoring the 57-character box width and 32-character description
   limit.

Then run `just check-full` on the combined tree.

## Testing strategy

- Every phase adds unit tests next to its own module; the epic's integration guarantee
  is the `tests/gate_conformance/` matrix from `gate-cli`, which must pass identically
  through ACE, the mobile bridge, and the CLI.
- ACE widget behavior is tested with the existing Textual harness under
  `tests/ace/tui/`; new visual surfaces (Inputs section, Actions section, draft banner)
  get PNG snapshots via `just test-visual`, with goldens under
  `tests/ace/tui/visual/snapshots/png/`.
- Run `just check` after file changes in this repo and `just check-full` before landing
  the combined tree — this epic touches the executor and the shared modal path, which is
  squarely in the broadening set.
- `sase-core` changes are gated on `contract.rs::committed_contract_snapshot_is_current`
  and the crate's own test suite; `sase-telegram` changes are gated on its suite.

## Risks and edge cases

- **The feedback rule change is the riskiest small change.** Generalizing `required` to
  `properties` alters behavior for existing gates on three surfaces. The audit in
  `feedback-input` is not optional, and the snooze and triage commands that currently
  assert empty input will break if the audit is skipped.
- **Extracting the typed form can break prompt launching.** `InputCollectionModal` is on
  the agent-launch path, which is used far more than gates. Land the extraction behind
  its existing tests before touching gate code.
- **The frozen mobile contract fails loudly, which is good.** Regenerate the snapshot in
  the same commit as the struct change or the freeze test fails; do not hand-edit the
  committed JSON.
- **The unaccepted-draft block is deliberately blunt.** Blocking every submit —
  including reject — while a draft exists is stricter than strictly necessary, because
  rejection does not consume the plan. It is chosen because the alternative requires
  per-kind reasoning about which branches consume the edited resource, and being wrong
  there destroys a reviewer's work silently. Revisit only with a declared per-option
  `consumes_edit` flag, not with a heuristic.
- **`option_inputs` and shared `input_data` must never both apply.** The executor
  rejects the combination rather than merging; merging would recreate exactly the
  ambiguity this epic removes.
- **Secrets in durable files.** `response.json` and `journal.jsonl` are audit data. If
  redaction cannot be landed cleanly in `inputs-core`, drop the `secret` field type from
  that phase and record a follow-up — do not ship an unredacted secret field.
- **Legacy bundles.** Every schema and envelope change in this epic is additive within
  `schema_version: 3`. Unanswered bundles created before it must remain hash-verifiable
  and answerable; the conformance matrix includes a pre-change fixture bundle to prove
  it.

## Acceptance criteria

1. A custom gate declaring `inputs` with a required field is created, rendered with
   typed widgets in ACE, answered from ACE, mobile, Telegram, and `sase gate answer`,
   and its command receives the typed values on stdin — with no per-kind code on any
   surface.
2. A custom gate whose option cannot accept what any client can produce fails at
   `sase gate create` with a message naming the offending property, instead of being
   accepted and dying silently at submit time.
3. Every gate rejection — input schema, bounds, feedback, adapter validation — appears
   in `errors/` and is visible by pressing `d`.
4. The same gate answers identically on ACE, mobile, Telegram, and the CLI with respect
   to feedback-to-input, proven by the conformance matrix rather than by inspection.
5. A plan or epic gate's `e` action opens the file in `~/.sase/plans/`, accepts the edit
   only when `sase plan validate` passes, can be repeated any number of times without
   answering or reopening the gate, preserves branch selection and scroll position, and
   blocks submission with a visible banner while an unaccepted draft exists.
6. A `run_command` action runs repeatedly on a pending gate, displays its output, leaves
   the gate answerable, and cannot rewrite any resource it did not declare.
7. A partially executed AND branch never silently re-runs its completed commands: the
   reviewer is asked to resume or restart.
8. `validate_bead_snooze_feedback` and `validate_task_triage_feedback` are deleted, and
   the snooze duration reaches the command as declared input.
9. `sase gate wait --json` reports `input`, `option_inputs`, and `option_results`, and
   the gate detail pane shows what a gate asks for before it is answered.
10. `docs/notifications.md` and the `/sase_gate` skill state the input contract plainly,
    with a worked non-empty example, and `sase skill init` has been run.
