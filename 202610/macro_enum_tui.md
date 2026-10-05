---
tier: tale
title: Enum choice menus and authoring throughout the TUI
goal:
  Complete sase-1g4.3 with Rust-backed enum and bool completion, typed choice pickers,
  lossless input authoring, and verified visual coverage.
size: medium
proposed_by: bbugyi200.athena.sase-1g4.3
bead: sase-1g4.3
create_time: 2026-10-05 09:09:36
status: wip
---

- **PARENT:**
  [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:** sase-1g4.3

# Plan: Enum choice menus and authoring throughout the TUI

## Ownership and design

Implement the `tui-enum` phase assigned as **sase-1g4.3**. It is already `in_progress`;
do not change its status manually. The dependency, sase-1g4.2, is closed. This is a
single bounded coding task, so use a tale with size `medium`.

Read the assigned bead with:

```bash
sase bead read sase-1g4.3 -r "Need the phase scope and design file"
```

The audited design sources are:

- `plan:202610/macro_named_input_types.md`, especially the shared contract and “Enum
  choice menus in the prompt bar, typed form, and authoring modals.”
- `research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md`.

Use `sase artifact read` for these sources. During planning, the bead's linked read
failed with
`validation: bead id segment must contain only letters, digits, '-' and '_'`. The
fallback `sase bead read ... --no-links` succeeded, and the plan was read separately
through the audited artifact command. The research initially reported missing; opening
its sidecar with `sase repo open sase--research -r "..."` refreshed it, after which the
same audited artifact read succeeded. Do not read sidecar artifact files directly. If
the linked-read defect persists, record its evidence as a proposed follow-up on this
phase rather than expanding this implementation.

Shared behavior belongs in sase-core. Use `/sase_repo` to open `sase-core` before
reading or changing it, and read that checkout's AGENTS.md. Python should provide thin
typed adapters and Textual presentation. The existing Rust functions already own type
labels, enum/bool choice ordering, fuzzy filtering, quoting, default markers, and
exclusion of selected repeatable values. Do not duplicate those rules.

Read `tui.md`, its screenshot/performance child notes, and `lint_and_test.md` using
`/sase_memory_read`. Read `src/sase/ace/AGENTS.md` for help/keymap conventions. No
memory edits are needed. Model routing, the model picker, plugin registry discovery, and
the other epic phases remain owned by their assigned workers. Consume any catalog
entries those phases have already provided, including named enums.

## Existing contracts and gaps

- `StructuredCatalogInput` in `macro/_catalog_models.py` already carries
  `choices: tuple[InputChoice, ...]`, `named_type`, and `value_role`.
- `InputChoice` already carries `value`, `label`, and `description`; `InputArg` carries
  resolved choices, the named type, and the value role.
- The TUI `MacroInputHint` and both catalog/InputArg projections still discard those
  fields. Detection classifies only base `bool` as `macro_arg_value`.
- `_file_completion_macro_args.py` still has a bool-only candidate builder.
- The required core bindings already exist:
  `macro_argument_choice_candidates({hint, partial, replacement, selected})` and
  `macro_input_type_label({hint})`. Candidate rows contain `value`, `insertion`,
  optional `label` and `description`, declared `index`, and `is_default`.
- `macro_argument_spans(text, entries=None)` returns Rust parser spans using UTF-8 byte
  offsets. The Rust completion classifier also exists in
  `editor/completion/trigger_context.rs`; a dedicated macro-context Python binding was
  not found during planning.
- The Python cursor detector currently splits on raw commas and checks for raw closing
  parentheses. This cannot correctly handle quoted punctuation, repeatable lists, or all
  middle-of-value replacements in the shared corpus.
- `tests/fixtures/macro_arg_choice_completion.json` already mirrors the core corpus.
  `tests/macro/test_macro_choice_projection_parity.py` verifies Rust candidates and
  runtime binding, but constructs the request from golden spans instead of testing the
  actual TUI detector.
- `typed_input_form.py` already cycles canonical values while showing labels, but uses
  that button for every enum size.
- `InputItemModal` has a free-text type field and no Choices editor. It can construct an
  enum with no choices and raise before displaying validation feedback. Its composition
  also references `InputType` without importing it on the explored tree.
- `MacroItemModal` represents inputs as a compact comma-separated string that cannot
  represent enum choices and formats named types as their base type. Frontmatter cell
  editing likewise starts from the base type and drops existing choices.

## Implementation

### 1. Carry the resolved hint and use shared labels

Extend `ace/tui/widgets/_macro_arg_assist_models.py` with additive defaults for
`choices`, `named_type`, and `value_role`. Reuse `InputChoice` for choices. Populate
them in `_macro_arg_assist_catalog.py` and `input_hint_from_input_arg` in
`_macro_arg_assist_inputs.py`, including workflow and live local-macro projections.

Add one small typed adapter that serializes a TUI hint to the Rust hint wire and
rehydrates choice candidates. Reuse it for labels and candidates, avoiding duplicate
wire dictionaries. Preserve defaults in the local TUI projection so Rust can mark the
actual default; boolean displays must be canonical `true`/`false`.

Use `macro_input_type_label` in input signatures, name-menu rows, active-input hint
panels, and typed-form headers. Up to four choices get the Rust value union; larger sets
get the named type or `enum` plus a count. Keep optional/default/repeatable markers
intact. Do not compute a separate Python union/count rule.

### 2. Resolve cursor contexts through the Rust parser

Update `_macro_arg_assist_detection.py` so resolved agent roles (and compatible base
`agent`) route to `macro_arg_agent`, choices or base `bool` route to `macro_arg_value`,
and paths retain their existing completion engine. Treat the metadata as additive so
phase 6 can add model-role dispatch later.

Replace raw-delimiter parsing in the choice path with core-provided structural context.
First evaluate whether the existing `macro_argument_spans` binding exposes all required
active-input, empty-value, whole-value replacement, and repeatable information. If it
does not, expose a narrow binding over the existing Rust completion classifier/parser
that returns this information, with a typed Python adapter. Do not reimplement
binding-to-input or quote/list parsing in Python.

The resolved context must provide the active input, partial value before the cursor,
full replacement value, whole value/element span, and already selected values for
repeatable inputs. Convert UTF-8 byte spans to Python string offsets only at the adapter
boundary. Preserve incomplete invocations, literal-zone suppression, and existing
optional/required argument assistance. `#pr:ready` continues to bind the first `name`
input rather than the status enum.

If a new core binding is required, add Rust and PyO3 tests, register it, and add it to
sase's binding validation requirements. The host finalizer owns commits. Follow
`docs/rust_backend.md` so `sase-core-revision.txt` covers every new binding before the
Python callers land; do not manually commit the linked repo to obtain a SHA. Document
any host-finalizer pin obligation explicitly in the final declaration.

### 3. Serve and render enum/bool value menus

Replace `_build_bool_completion_candidates` in `_file_completion_macro_args.py` with the
Rust choice builder. Pass the real partial, full replacement, and selected values. Map
canonical values to candidate names/display and use the Rust insertion verbatim. Carry a
typed metadata record with label, description, and default state.

Audit every kind-dispatch path, including `_file_completion_open.py`,
`_file_completion_tab.py`, `_file_completion_refresh.py`, `_file_completion_accept.py`,
`prompt_completion.py`, live soft completion, and Ctrl+N/Ctrl+P navigation. The existing
`macro_arg_value` kind must work consistently for enums and bools.

- `#m:` and `#m(k=` open a value menu immediately when their active input has choices.
- Accepting a `name=` row chains into its value menu, even when the name menu had only
  one candidate. Auto-open shows a choice without accepting it.
- Empty-prefix menus keep declared order and initial selection rather than moving to the
  default. Prefix/fuzzy ranking and repeatable exclusions come from Rust.
- Acceptance replaces the entire active value/element, preserving adjacent values,
  quotes, and surrounding argument syntax.
- Manual completion, re-filtering, cycling, and soft completion use the same rows.
  Preserve existing submit-versus-accept behavior and path/agent completions.

Update `_prompt_input_bar_completion_panel*` and the appropriate row renderer. Use the
canonical value in the main column, a dim label in the right column, the description as
the detail line, and a quiet `default` badge. Render free text safely as Rich text, not
markup. The title is `<input> · <named_type or enum>` (scalar bool uses `bool`); long
sets show an unobtrusive filter hint. Respect narrow widths and both themes. Keep
rendering and keystroke paths read-only with no filesystem or plugin-discovery work on
the event loop.

### 4. Choose values in typed forms

Keep the existing cycle control for up to five enum choices, displaying labels and
storing canonical values. For more than five, add a searchable choice picker using
existing modal/navigation conventions and the Rust fuzzy/choice binding. Show value,
label, and description; retain default/required sentinel behavior.

Integrate the picker with `TypedInputForm`: acceptance changes the field and emits
`Changed`, cancellation leaves it intact, reopening shows the current selection, and
required validation and optional visibility continue to work. Typed-form values are raw
canonical values, so do not store the prompt-syntax quoted insertion in a form. Check
repeatable enum behavior through the existing field conversion path; preserve each
element when editing rather than collapsing an existing list to one scalar. Model-picker
integration belongs to sase-1g4.6.

### 5. Author inputs without losing their metadata

Use the Rust type catalog (including descriptions and all currently available
named/plugin types) for type pickers in `input_item_modal.py`, `macro_item_modal.py`,
and `_frontmatter_panel_cell_editing.py`. The current `frontmatter_schema` adapter only
exposes name/aliases/rule/advertised; extend the thin catalog projection or reuse the
richer `macro_input_type_catalog` projection as appropriate. Display canonical named
types rather than enum/word bases when editing an existing input.

Choosing inline `enum` reveals a Choices editor. A multiline YAML list is a suitable
minimal editor because it handles scalar choices and `{value, label, description}`
entries without inventing a compact grammar. Parse YAML as glue and validate the
declaration with the existing Rust validator/loader adapter. Show errors for an empty
list, duplicates, non-string/empty/whitespace/`null` values, forbidden choices on
another type, and a non-member default. Never construct an invalid InputArg as part of
live feedback outside an exception-safe validation path.

Replace the local macro's lossy compact-input-only editing with an editable input list
that reuses `InputItemModal` for adding/editing an input. It must support enums and
named types, preserve choice labels/descriptions, repeatability, defaults and roles, and
retain current input order. Existing simple compact input support may remain as an entry
convenience only if it cannot overwrite richer declarations.

Add the choices editing path to frontmatter cell editing or delegate structured enum
changes to the shared modal. Preserve existing input metadata when changing unrelated
cells. Inline enums serialize their authored choices; named enums write the canonical
named type and do not write resolved choices back. Reuse current serialization/loader
behavior rather than writing another serializer. Verify save/reopen/raw-YAML/local-macro
handoff round trips.

### 6. Document and verify the visible behavior

Update the macro-argument completion and live soft-completion sections of `docs/ace.md`,
and the help popup in `modals/help_modal/binding_common.py`. Document enum/bool menus,
labels/descriptions, canonical acceptance, automatic name-to-value chaining, and the
small-cycle/large-picker distinction. Keep help descriptions within the existing box
limits. Update default keymap configuration only if new configurable bindings are
introduced.

## Acceptance and verification

Add meaningful coverage to the existing suites rather than implementation-mirroring
tests:

1. Drive **every case** in `tests/fixtures/macro_arg_choice_completion.json` through the
   real Python TUI detection and candidate adapter. Assert context kind, active input,
   replacement span after coordinate conversion, candidate values and Rust insertions,
   default markers, and runtime acceptance after applying the edit. Retain the
   byte-for-byte corpus parity check with the opened core checkout. Include
   positional/named/colon forms, repeatable list elements, quoted commas and plus signs,
   a cursor inside an existing value, non-BMP text, bools, and the `#pr:ready`
   first-positional-input case.
2. Extend `test_macro_arg_value_completion.py`, detection/hints tests, and relevant
   live-completion tests to prove `#pr(x, status=` opens `wip/draft/ready` with
   descriptions, `status=` acceptance chains into that same menu, bools still offer
   `true/false`, filtering preserves Rust order, and edits do not damage suffixes.
3. Extend `test_typed_input_form.py` with the five/six-choice boundary, labels versus
   values, default/sentinel behavior, filtering, cancellation, acceptance, and
   changed/submittable state. Exercise a repeatable declaration where supported.
4. Extend `test_frontmatter_panel_subeditors.py` and related authoring tests for new
   enum creation without crashes, invalid choices/default feedback, named-type
   preservation, and edit/save/reopen with full choice metadata. Verify that editing
   another cell does not erase choices, roles, or repeatability.
5. Add deterministic PNG snapshots for the enum value menu in dark and light themes at
   120x40 and for the typed-form searchable picker. Extend the existing
   `test_ace_png_snapshots_macro_arg_completion.py` harness or a focused companion.
   Update affected authoring snapshots if their layout changes. Run targeted
   `just fix-tui-screenshots -- <selectors>` and inspect its retained manifest,
   warnings, every created/removed golden, and each update group with actual image
   viewing. A `partial` update is not approval for skipped goldens. Capture and view a
   real `sase screenshot` for the prompt choice flow if fixture logs alone do not
   establish the visible result.

Run focused tests while iterating, then `just fix` or at least `just fmt`, followed by
**`sase tool run check`** from the sase checkout. Do not run `just check-full`. If core
files changed, also run the core's wrapped `sase tool run check` and its binding tests.
Use `/sase_monitor` for long checks or snapshot runs, with a concrete follow-up that
inspects results and continues this phase; always wait for the monitor-start command to
exit before ending its handoff turn. Never skip image inspection through a prepared
completion declaration.

Classify check failures using retained evidence. Fix new failures caused by this phase.
If a failure reproduces identically on the clean base, append a `PROPOSED FOLLOW-UP:`
note to sase-1g4.3 with the reproduction and any existing tracking bead; it does not
prevent closing this phase. Do not create follow-up beads yourself, weaken assertions,
or change unrelated code to quiet a base failure.

## Completion

Before closing, run:

```bash
sase bead epic-symbols sase-1g4.3
```

Planning found no entries, but inspect again after implementation. Resolve every
remaining symbol or re-key its Justfile exemption to a still-open parent/later-phase
bead with the correct ownership. Do not leave exemptions keyed to the closed phase.

Append discovered out-of-scope work only with:

```bash
sase bead note sase-1g4.3 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'
```

Close **only sase-1g4.3**, with a concrete note naming functional tests, fixture parity,
visual inspection, check outcomes, and any independently proven base failure:

```bash
sase bead close sase-1g4.3 --note "<what was verified>"
```

Do not close sase-1g4, an ancestor, or any automatically associated child plan bead. The
parent land agent owns ancestor completion. Follow `/sase_final` as the last action
before a normal final response; the host owns commits and any linked-repo completion
obligations.
