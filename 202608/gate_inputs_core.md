---
tier: tale
title: Declarative per-option gate inputs and per-option submission
goal:
  A gate option declares what it needs with a closed `inputs:` vocabulary that compiles
  into its `input_schema` at creation, and the executor accepts and persists one input
  value per selected option instead of one shared blob.
proposed_by: bbugyi200.athena.sase-h7.3
bead: sase-h7.3
create_time: 2026-08-07 17:24:18
status: done
---

- **PROMPT:**
  [prompts/202608/gate_inputs_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_inputs_core.md)
- **PARENT:** [202608/gate_input_collection.md](gate_input_collection.md)
- **BEAD:**
  [sase-h7.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h7/sase-h7.3.md)

# Plan: Declarative per-option gate inputs and per-option submission

This is phase `inputs-core` of epic `sase-h7` (Gate input collection and repeatable
non-terminal gate actions). It lands the authoring contract that five later phases
consume; it deliberately renders nothing and changes no surface.

## Background

Every `GateOption` already carries an `input_schema`
(`src/sase/notification_gates/model_options.py:63`), the executor already validates one
JSON value against every selected option's schema and writes it to each command's stdin
as canonical JSON (`src/sase/notification_gates/executor.py:84-90`, `:415`), and the
value is persisted verbatim into the write-once response (`:197-207`). The transport is
sound. What is missing is a way to _declare_ what an option needs, and a way to submit a
_different_ value to each member of an AND branch.

The cost of the missing layer is visible in the tree today:
`src/sase/plan_gate.py:551-578` makes both AND members of the tale plan gate admit the
coder fields even though only `approve` consumes them, and says so in a comment, because
one shared value must satisfy every selected option's schema.

Two facts constrain the design and are not negotiable:

- **User input never reaches `argv`.** Commands are hashed at creation and executed with
  `shell=False` against `/proc/self/fd/N`. Stdin JSON is the only channel.
- **`input_schema` stays the single enforcement layer.** `inputs` compiles into it at
  creation and the compiled object is what the envelope stores, so the executor contract
  is untouched and a reader that knows nothing about `inputs` still enforces correctly.

## Scope

**In scope:** the `inputs:` authoring vocabulary, its compilation into `input_schema`,
`InputType.ENUM` plus `choices` across both consumers of the shared xprompt vocabulary,
per-option submission and persistence in the executor, and secret redaction in
`response.json`.

**Out of scope, owned by sibling phases:** rendering inputs in ACE (`inputs-ace`,
`sase-h7.6`); the mobile wire and Telegram step flow (`inputs-remote`, `sase-h7.8`);
creation-time answerability, dialect pinning, declared-default validation, and input
bounds (`custom-validation`, `sase-h7.5`); the error-recording wrap, input bounds, and
`journal.jsonl` (`executor-integrity`, `sase-h7.1`); injecting the reviewer note as
`input.feedback` (`feedback-input`, `sase-h7.2`); docs and the `/sase_gate` skill
(`docs`, `sase-h7.12`). Do not implement any of those here.

## 1. Extend the shared input vocabulary — `src/sase/xprompt/models.py`

Add to the existing vocabulary rather than forking one. Both gates and xprompts then
speak one type system.

- New frozen dataclass `InputChoice` with `value: str` and `label: str | None = None`.
- `InputType.ENUM = "enum"`.
- `InputArg.choices: tuple[InputChoice, ...] = ()`.
- `InputArg.__post_init__` raises `XPromptValidationError` when `type is InputType.ENUM`
  and `choices` is empty, when `choices` is non-empty for any other type, and when two
  choices share a `value`. Every existing `InputArg(...)` construction in the tree
  passes no `choices` and a non-enum type, so this check is inert for them.
- `InputArg.validate_and_convert` gains an `ENUM` branch that accepts only a declared
  choice `value` (exact match, no case folding, no label matching) and otherwise raises
  `XPromptValidationError` naming the argument and listing the allowed values.

`input_binding.py` needs no change: `_convert_value` already routes through
`validate_and_convert`.

## 2. Parse and serialize `choices` in xprompt frontmatter

- `src/sase/xprompt/loader_parsing.py`: add `"enum": InputType.ENUM` to
  `parse_input_type`'s map. Add a `parse_input_choices(raw, name)` helper accepting
  either a list of scalars (`choices: [fast, slow]`) or a list of `{value, label}`
  mappings, returning `tuple[InputChoice, ...]` and raising `XPromptValidationError` on
  any other shape. Call it from `_parse_shortform_input_metadata` and from the longform
  branch of `parse_inputs_from_front_matter`, and pass the result to `InputArg`.
- `src/sase/xprompt/prompt_frontmatter.py`: `_input_to_yaml` emits `choices` (scalar
  values when no labels are set, `{value, label}` mappings otherwise) and takes the
  mapping form whenever choices are present, so a declaration round-trips through
  `parse_inputs_from_front_matter`.

Every other `inp.type.value` consumer in the tree (catalog, explain, graph, the
frontmatter panel, the ACE preview renderers) renders the new type name with no change.
The ACE input-collection modal will render an enum arg as a plain text field validated
by `validate_and_convert` until `inputs-ace` adds the cycling select; that degradation
is acceptable and must not be papered over here.

## 3. New module `src/sase/notification_gates/model_inputs.py`

```python
@dataclass(frozen=True)
class GateInputField:
    id: str                  # ^[a-z][a-z0-9_]*$ ; the JSON property name
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

Public functions:

- `parse_gate_input_fields(value, target) -> tuple[GateInputField, ...]` — closed field
  set via `reject_unknown_fields`, id pattern and uniqueness, required `label`, known
  type spelling, enum-requires-choices and choices-forbidden-elsewhere, boolean-typed
  `required`/`secret`/`repeatable`, string-or-null `placeholder`/`help`. Every failure
  is a `GateError("invalid_input_field", "<target>.inputs[i].<field>", ...)`.
- `compile_gate_input_schema(fields, *, feedback_mode) -> dict[str, Any]`.

Compilation table — this is the contract:

| `InputType`        | Compiled fragment                                            |
| ------------------ | ------------------------------------------------------------ |
| `word`, `agent`    | `{"type": "string", "minLength": 1, "pattern": "^\\S+$"}`    |
| `line`             | `{"type": "string", "pattern": "^[^\\n]*$"}`                 |
| `text`             | `{"type": "string"}`                                         |
| `path`             | `{"type": "string", "minLength": 1, "pattern": "^[^\\n]*$"}` |
| `int`              | `{"type": "integer"}`                                        |
| `float`            | `{"type": "number"}`                                         |
| `bool`             | `{"type": "boolean"}`                                        |
| `enum`             | `{"enum": [<declared values, in declared order>]}`           |
| `repeatable: true` | wraps the fragment in `{"type": "array", "items": {...}}`    |

The compiled object is

```json
{"$schema": "https://json-schema.org/draft/2020-12/schema",
 "type": "object", "properties": {...}, "required": [<required ids>],
 "additionalProperties": false}
```

plus a `"feedback": {"type": "string"}` property whenever `feedback_mode` is not
`disabled`, so `feedback-input`'s rule works with no author action. The injected
property is never added to `required` — feedback presence is enforced by
`_normalize_feedback`, not by the schema. **A declared field whose id is `feedback`
takes precedence and suppresses the injection**, so `retire-smuggling` can later declare
feedback as an ordinary input field without a special case.

This module must not import `model_options`; `model_options` imports it.

## 4. `GateOption` gains `inputs` — `src/sase/notification_gates/model_options.py`

- Add `inputs: tuple[GateInputField, ...] = ()` and add `"inputs"` to the closed field
  set in `from_mapping`.
- When `inputs` is declared, compile it and use the compiled object as `input_schema`.
- **`inputs` and a raw `input_schema` are mutually exclusive.** Implement that as an
  exact-equality check rather than a bare presence check: if `input_schema` is present
  _and_ differs from the compiled object, raise
  `GateError("conflicting_input_declaration", "<target>.input_schema", ...)`. This is
  the mechanism that makes envelope re-parse work, and it is a deliberate refinement of
  the epic plan's wording. `to_dict()` writes both `inputs` and the compiled
  `input_schema`, and `_options_from_envelope` (`executor.py:277`) and
  `GateBranchData.from_envelope` (`branches.py:48`) re-parse that envelope through this
  same code path; a bare presence check would reject every envelope the phase itself
  produces. A hand-authored raw schema will not accidentally equal the compiled form, so
  creation still fails closed for a genuine double declaration.
- `to_dict()` emits `"inputs": [field.to_dict() for ...]` always (`[]` when none),
  beside the existing keys. Envelope shape stays `schema_version: 3`; the field is
  additive and optional, so bundles created before this phase parse unchanged.
- Extend the class docstring to say that the envelope's `input_schema` is the
  enforcement layer and `inputs` is the authoring layer projected into it.

Re-export `GateInputField` and `compile_gate_input_schema` from
`src/sase/notification_gates/models.py` and `__init__.py` for the later phases.

## 5. Per-option submission — `executor.py` and new `executor_inputs.py`

New `src/sase/notification_gates/executor_inputs.py` (kept out of `model_inputs.py`
because it needs `GateOption`, which would otherwise be an import cycle):

- `resolve_option_inputs(selected, input_data, option_inputs) -> dict[str, Any]`
  - both `input_data` and `option_inputs` non-`None` →
    `GateError("conflicting_input", "option_inputs", ...)`; merging would recreate
    exactly the ambiguity this phase removes.
  - `option_inputs` given → it must be a mapping; a key that is not a selected option id
    raises `GateError("unknown_option", "option_inputs", ...)`; a selected option with
    no entry gets `{}` and is then judged by its own schema.
  - only `input_data` given (or neither) → today's behavior: every selected option maps
    to the same normalized value.
- `redact_option_inputs(selected, resolved) -> dict[str, Any]` — a response-safe copy in
  which every value of a declared `secret: true` field is replaced by
  `{"$redacted": true}` (the whole array for a repeatable secret field). Options with a
  raw `input_schema` cannot declare secrets and pass through unchanged.

In `execute_gate_selection`:

- Add keyword-only `option_inputs: Mapping[str, object] | None = None` and document the
  per-option contract in the docstring.
- Replace the shared-value loop at `:84-90` with a resolve-then-validate loop that keeps
  the existing
  `_validate_json_instance(value, option.input_schema, f"option {option.id} input")`
  call shape, so `executor-integrity`'s error-recording wrap merges over it cleanly.
- Pass `resolved[option.id]` to `_run_owned_command` instead of the shared value.
- The response gains `"option_inputs": redact_option_inputs(...)` and keeps `"input"`
  with its current meaning — the shared submitted value, `{}` on the per-option path.
  `GATE_RESPONSE_SCHEMA_VERSION` stays `2`; readers must tolerate the key's absence on
  older responses.

Secrets reach the command's stdin unredacted and are redacted only on the way to
`response.json`, which is durable audit data. `executor-integrity`'s `journal.jsonl`
records digests rather than values, so it inherits the property; say so in
`redact_option_inputs`'s docstring so that phase does not have to rediscover it.

## 6. Close the `response["input"]` reader gap

`src/sase/notification_gates/adapters.py:175` reads
`response["input"]["epic_launch_mode"]` and falls back to `"detached"`. Once
`inputs-ace` starts submitting `option_inputs`, that reader would silently launch an
epic detached after the reviewer chose `skip`.

Add `effective_response_input(response, option_id) -> dict[str, Any]` to
`src/sase/notification_gates/model_results.py` — return the option's entry from
`option_inputs` when present, else the shared `input`, else `{}` — and use it at
`adapters.py:175` with `selected_ids[0]`, the id the surrounding code has already
resolved. Three lines here remove a silent failure from a sibling phase.

Leave `_plan_input_schema` (`src/sase/plan_gate.py:551-578`) and its AND-intersection
comment alone: it is still accurate until a surface submits `option_inputs`. Tightening
it belongs to `inputs-ace`.

## 7. Cross-repo: `enum` in the Rust frontmatter engine (`sase-core`)

The xprompt `input:` type catalog, its diagnostics, and the xprompt LSP live in
`sase-core`, not here (`src/sase/xprompt/frontmatter_schema.py` is a thin adapter over
`frontmatter_input_type_schema` and `validate_frontmatter`). Landing `enum` in Python
alone would make the editor and the ACE frontmatter panel flag a valid declaration,
which is precisely the surface-versus-host drift this epic exists to delete.

Open the repo with `/sase_repo` (`sase repo open sase-core -r "..."`) and use the
printed path as the only path for reads and writes.

In `crates/sase_core/src/editor/frontmatter.rs`:

1. `InputType` gains `Enum`; add it to `InputType::ALL`, `aliases()` (none), and
   `rule()` ("One of the values declared under `choices`.").
2. `parse_input_type` accepts `"enum"`; `declared_type_name` returns `"enum"`; extend
   the `XPROMPT_INPUT_TYPE_EXPECTED` constant.
3. Add `"choices"` to the accepted key lists in `validate_nested_input_unknown_keys` and
   `validate_longform_unknown_keys`, so a valid declaration stops emitting
   `unknown_xprompt_frontmatter_input_field`.
4. Validate `choices`: required and non-empty for `enum`, an error on any other type, a
   sequence of scalars or `{value, label}` mappings, unique values.
5. `validate_input_default` validates an `enum` default by membership in the declared
   choices rather than by a character rule.

Also add `"enum" => "enum"` to the two other type-name maps that currently fall through
to `"line"`: `crates/sase_core/src/editor/diagnostics.rs`'s `parse_input_type_name` and
`crates/sase_core/src/xprompt_catalog.rs`'s `parse_input_type`.

Cover each with Rust unit tests next to the existing ones; the existing
`input_type_schema_matches_parser_spellings` test picks up the new variant
automatically.

**Landing order is not a constraint here, deliberately.** Nothing in this repo's change
requires the new core, and no Python test asserts that the binding catalog contains
`enum` — the ACE type picker simply gains an entry once core lands. Either repo can land
first; the only cost of an interleaving is a temporary editor diagnostic (this repo
first) or a picker entry the loader maps to `line` (core first). Commit each repo
separately with `/sase_git_commit`, and after the Rust edit run `just install` in this
repo so the locally built binding is what the suite exercises.

## Tests

New `tests/test_gate_inputs_core.py`:

- Compilation of every `InputType` including `repeatable` and `enum`, table-driven, plus
  `required` collection, `additionalProperties: false`, and the pinned `$schema`.
- The auto-injected `feedback` property appears for `optional` and `required` feedback
  modes, never for `disabled`, and is never listed in `required`; a declared field named
  `feedback` suppresses the injection.
- `inputs` plus a differing raw `input_schema` raises `conflicting_input_declaration`.
- Envelope idempotence: create a gate declaring `inputs`, re-parse the stored envelope
  through `GateOption.from_mapping` and `GateBranchData.from_envelope`, and assert the
  fields and compiled schema survive the round trip unchanged.
- Per-option submission delivers different values to two AND members whose schemas are
  mutually exclusive under `additionalProperties: false` — the case that is impossible
  today — asserting each command's stdin from its echoed result.
- Legacy shared `input_data` path is unchanged and `option_inputs` in the response
  mirrors `input` for every selected option.
- Supplying both raises `conflicting_input`; an `option_inputs` key outside the
  selection raises `unknown_option`; both leave the gate pending with no
  `response.json`.
- A required declared field submitted as `{}` fails `schema_validation_failed` and
  leaves the gate pending.
- A `secret: true` field reaches the command's stdin intact (the command echoes a
  digest) while `response.json` records `{"$redacted": true}`.

xprompt coverage in `tests/test_xprompt_loader_parsing.py` and
`tests/test_prompt_frontmatter.py`:

- Shortform and longform `type: enum` with both scalar and `{value, label}` choices
  parse into `InputArg` with `InputChoice`s.
- `validate_and_convert` accepts a declared value and rejects an undeclared one with a
  message listing the allowed values.
- `_input_to_yaml` round-trips choices and labels.
- Enum without choices, choices on a non-enum type, and duplicate choice values each
  raise `XPromptValidationError`.

Rust unit tests as described in section 7.

## Verification

Run `just install` first — this workspace may have drifted dependencies, and the linked
`sase-core` checkout is built from source by that recipe.

Then `just check-full`, not `just check`: this phase touches `executor.py` and the
shared option model, which is squarely in the broadening set. In `sase-core`, run the
crate test suite (`cargo test`) for the edited crates.

## Risks

- **Envelope re-parse is the sharp edge.** `GateOption.from_mapping` is both the
  creation parser and the envelope reader. The exact-equality conflict rule in section 4
  exists only because of that; get it wrong and every gate declaring `inputs` becomes
  unreadable immediately after creation. The idempotence test is the guard.
- **`executor.py:84-90` is contended.** `executor-integrity` (`sase-h7.1`) rewrites the
  same loop to record validation failures. Keep this phase's diff there structurally
  obvious — resolve above the loop, leave the `_validate_json_instance` call shape
  intact — so the land agent's merge is mechanical.
- **Do not ship an unredacted secret.** If `redact_option_inputs` cannot be landed
  cleanly, drop the `secret` field from the vocabulary and record a follow-up rather
  than shipping the field without redaction.
- **Compiled `pattern` anchors are approximate under Python's `re`.** `$` in Python
  matches before a single trailing newline, so `"ab\n"` satisfies the compiled `word`
  and `line` fragments even though `InputArg.validate_and_convert` rejects it. The table
  above is implemented verbatim because it is portable ECMA-262, where `$` has no such
  leniency. Record this as a `PROPOSED FOLLOW-UP` for `custom-validation` (`sase-h7.5`),
  which is the phase that pins the dialect.
- **Mobile carries the new type name harmlessly.** `MobileXpromptInputWire.type` is a
  free-form `String`, so an `enum` xprompt input reaches the app as an unrecognized type
  and renders as text. Note it as a follow-up for `inputs-remote` rather than fixing it
  here.
