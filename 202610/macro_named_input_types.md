---
tier: epic
title: "Named macro input types: finish enum, add model/effort, share plugin enums"
goal: "A macro input's `type` names what its value is: a scalar keyword, `enum` with
  inline `choices`, a builtin type (`agent`, `model`, `effort`), or a plugin's shared
  enum (`<dist>@<id>`). Every such value completes, validates, and explains itself the
  same way in the TUI prompt bar, the typed launch form, the LSP (Neovim and other
  editors), `sase macro show`/`types`, and the runtime binder, because sase-core owns
  one type vocabulary, one validator, and one candidate builder.

  "
phases:
  - id: vocab
    title: One input-type vocabulary and strict enum declarations
    depends_on: []
    size: large
    description: "vocab: add the sase-core macro_input_types module (type catalog,
      resolver, did-you-mean, choice value rules, PyYAML-parity quoting check, enum
      value check), move every Rust and Python type parser onto it, fix the seven
      existing enum defects, isolate bad macros, add the strict_macro_input_types sunset
      flag, generate the JSON schemas, add the config.macro_input_types doctor check,
      and make #pr's status a real enum.

      "
  - id: wire-lsp
    title:
      Choices, named types, and roles on every wire; enum completion and diagnostics in
      the LSP
    depends_on:
      - vocab
    size: large
    description: 'wire-lsp: carry choices/named_type/value_role on every hint, catalog,
      mobile, and CLI projection; add the shared Rust choice-candidate builder and type
      label; give the LSP enum completion, invocation diagnostics with "Replace with"
      quick fixes, rich hover, and frontmatter type completion; add shared golden
      fixtures.

      '
  - id: tui-enum
    title: Enum choice menus in the prompt bar, typed form, and authoring modals
    depends_on:
      - wire-lsp
    size: large
    description: "tui-enum: route enum and bool arguments through the Rust choice
      builder in the prompt bar with labelled, described rows; use type labels in hints;
      add a searchable picker for large sets in the typed form; let authoring modals
      pick any type and edit choices; drive the shared golden fixtures from Python; add
      visual snapshots.

      "
  - id: model-core
    title: Builtin model and effort types with one routing classifier
    depends_on:
      - wire-lsp
    size: large
    description: "model-core: register builtin effort (closed enum) and model (domain)
      types; add the Rust model classifier over a model validity snapshot shared by the
      runtime binder, sase doctor, and the LSP (via a routing block in
      model_catalog.json); give model arguments the %model completion menu, warnings,
      quick fixes, and hover in the LSP.

      "
  - id: plugin-types
    title: Plugin-shared enums, sase macro types, and plugins.required
    depends_on:
      - model-core
    size: large
    description: "plugin-types: load plugin input_types.yml files in sase-core, resolve
      <dist>@<id> with clear missing-plugin errors, discover files with their
      distributions in Python and export them to the LSP, add the sase macro types
      command, and extend the doctor check with registry and plugins.required findings.

      "
  - id: model-tui
    title: Model arguments use the %model menu and model picker in the TUI
    depends_on:
      - tui-enum
      - model-core
    size: medium
    description: "model-tui: add the macro_arg_model completion kind that reuses the
      exact %model directive menu (aliases on @, provider drill-down, effort rows), open
      the existing ModelPickerModal for model inputs in the typed form, and add a visual
      snapshot.

      "
  - id: adopt
    title: Dogfood, documentation, and memory
    depends_on:
      - plugin-types
      - model-tui
    size: medium
    description:
      "adopt: ship sase-research-artifacts' audio_edition type and move research_swarm's
      model inputs to type model, type sase's own macros, rewrite the docs input-type
      section around the one rule, update the macros.md memory Inputs line, and add an
      end-to-end parity test across runtime, LSP, and TUI."
proposed_by: bbugyi200.athena.0wj
create_time: 2026-10-04 18:19:25
status: wip
---

- **PROMPT:**
  [prompts/202610/macro_named_input_types.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/macro_named_input_types.md)

# Plan: Named Macro Input Types

## Context

Read the research that motivated this design before starting any phase:

```bash
sase artifact read research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md "Design context for named macro input types"
```

The user agreed with every recommendation in that report. This plan follows it, with a
few decisions made more precise after checking the code (they are called out as
**Refinement** below). In short:

- `enum` already exists: `InputType.ENUM`, inline `choices`, Rust frontmatter
  diagnostics, the mobile wire, and the typed-form cycle button. What is missing is the
  part that matters: no frontend completes enum values, editor diagnostics accept any
  value, and there is no way to share a value set.
- The motivating example (`builtin@model_enum_values`) is not an enum. `%model` accepts
  an open grammar. It becomes the builtin **domain** type `model`. Sharing becomes
  **named input types**.
- Known defects that this epic fixes:
  - YAML 1.1 and 1.2 disagree about values: PyYAML reads `yes` as `True`.
  - Invalid enum defaults are not caught by the Python loader.
  - Choice values that are empty, contain whitespace, or are `null` are accepted.
  - Workflow longform drops `choices`.
  - Unknown types silently become `line`.
  - The type vocabulary is defined in five places (Python, two Rust parsers, two JSON
    schemas).
  - Rust and Python disagree about `path` values.
  - A bad input declaration in an `.md` or config macro raises an exception nothing
    catches.
  - The local-macro handoff JSON drops `choices`.
  - The TUI input modal crashes when `enum` is picked.

Code locations below were verified against sase `aeccf843` and pinned sase-core
`2f16dc4a`. Line numbers drift, so search by symbol.

## Design

This section is the shared contract. Every phase implements its part of it and must not
contradict it. If a phase finds the contract wrong, it records a `PROPOSED FOLLOW-UP:`
note on its own bead instead of silently diverging.

### The rule authors learn

> **`type` names what the value is**: a scalar keyword (`word`, `line`, `text`, `path`,
> `int`, `float`, `bool`, `code`), `enum` with inline `choices`, a builtin type
> (`agent`, `model`, `effort`), or a plugin's shared enum (`<dist>@<id>`). Bare names
> belong to sase; qualified names belong to plugins.

```yaml
---
name: deploy
input:
  env: # inline enum
    type: enum
    choices:
      - { value: staging, description: Pre-prod cluster }
      - { value: prod, label: Production, description: Customer traffic }
    default: staging
  model: { type: model, default: "@large" } # builtin domain, same values as %model
  effort: effort # builtin closed enum, same values as %effort
  edition: sase-research-artifacts@audio_edition # plugin-shared enum
---
```

The shortform `name: <type string>` already works for every named type, so this adds no
new grammar.

### Type resolution (sase-core owns it)

1. **Scalar keywords and `enum`** work as they do today. Aliases are `integer` → `int`
   and `boolean` → `bool`. `string` becomes a **deprecated alias of `line`**: it is
   accepted, the LSP warns with a "Use `line`" quick fix, and the doctor lists it.
   - `enum` requires inline `choices`.
   - `choices` on any other type is an error. If the type is a named enum, the message
     says it already defines its values. Narrowing is out of scope.
2. **Bare names** resolve to builtin types and match case-insensitively, as both parsers
   do today. The canonical spelling is lowercase. Every bare name belongs to sase by
   construction, because plugin types are always qualified. That makes a reserved-names
   list unnecessary.
3. **`builtin@<name>`** is an accepted alias of any bare builtin name. This is the
   user's original instinct, so `builtin@model` works. Hover, labels, and formatters
   always show the bare form.
4. **`<dist>@<id>`** resolves to a plugin-declared type. It uses the existing
   `<plugin>@<id>` grammar (`src/sase/plugins/qualified_id.py`). The distribution is
   canonicalized with PEP 503 rules, ported to Rust: lowercase, and every run of `-`,
   `_`, or `.` becomes `-`. Plugin types are never reachable by a bare name, so
   installing an unrelated plugin cannot change what an existing macro means.
5. **An unknown name is an error** that suggests the closest known names. It replaces
   the silent `line` fallback, behind the sunset flag `strict_macro_input_types`. With
   the flag on (the default) the error applies; with it off, the old silent-`line`
   behavior returns.
6. **Errors stay per macro.** An unresolved type skips only the macro that uses it:
   - The skip is recorded as a load issue (`src/sase/macro/load_issues.py`).
   - It never breaks the catalog.
   - It never raises out of a loader.
   - The editor catalog keeps the entry and treats the unresolved input as `line`.
   - The macro's own file carries the error diagnostic.

### Type catalog (one Rust table, many projections)

Each catalog entry has these fields:

- `name`: canonical spelling.
- `aliases`.
- `kind`: one of `scalar`, `inline_enum`, `named_enum`, `domain`.
- `base`: the base kind older consumers see.
- `value_role`: a `DirectiveValueRole`, or null.
- `choices`: resolved choices for a named enum.
- `description`.
- `rule`.
- `source`: `builtin`, or `plugin` with its distribution and file path.
- `deprecated_alias_of`: only for `string`.

This table feeds:

- the TUI type pickers;
- `frontmatter_input_type_schema`, which becomes a projection of it;
- LSP `type:` completion and hover;
- the generated JSON schemas;
- error suggestions;
- `sase macro types`.

| Type                                                    | Kind        | Base on wires                         | Role    | Values                                                                                                 | Lands in       |
| ------------------------------------------------------- | ----------- | ------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------ | -------------- |
| `word` `line` `text` `path` `int` `float` `bool` `code` | scalar      | itself                                | —       | today's rules                                                                                          | `vocab`        |
| `enum`                                                  | inline_enum | `enum`                                | —       | inline `choices`                                                                                       | `vocab`        |
| `agent`                                                 | domain      | `agent` (kept for wire compatibility) | `agent` | word rules                                                                                             | `vocab`        |
| `effort`                                                | named_enum  | `enum`                                | —       | `none minimal low medium high xhigh max`, using the existing `%effort` suggestion text as descriptions | `model-core`   |
| `model`                                                 | domain      | `word`                                | `model` | [the model contract](#the-model-contract)                                                              | `model-core`   |
| `<dist>@<id>`                                           | named_enum  | `enum`                                | —       | the plugin's declared `choices`                                                                        | `plugin-types` |

The guiding rule for future builtins: **if a directive accepts it, a macro input can be
typed as it, and both complete and validate the same way.**

### Choice value rules (every enum, inline or shared)

- **Values must be strings.**
  - A non-string YAML scalar is an error that tells the author to quote it.
  - Python sees PyYAML's typed values, so `yes` arrives as `True`.
  - The Rust frontmatter validator sees YAML 1.2 strings. It therefore inspects the
    source text: an unquoted plain scalar that PyYAML's implicit resolvers would type as
    bool, int, float, null, or timestamp gets the same error, with a "Quote it" quick
    fix.
  - Port PyYAML's resolver regexes into one Rust helper. This closes the YAML 1.1 vs 1.2
    gap at the source.
- **Values must be words.** Each value must be:
  - non-empty;
  - free of Unicode whitespace;
  - not the literal `null`, which is the binder's "use the default" sentinel;
  - unique.
- **Matching is exact and case-sensitive.** A label is never accepted as input.
- **Choice item keys** are exactly `value` (required), `label`, and `description`, which
  is new. Both `label` and `description` are free text and are never inserted. Scalar
  items stay allowed.
- **Warn** (LSP and doctor only, never a load failure) about characters that need
  quoting in shorthand: `,` `+` `(` `)` `[` `]` `"` `'` and backtick.
- **Defaults:**
  - A closed-set default (inline or named enum) must be a string member. Otherwise it is
    a load error in Python and an Error in the LSP.
  - A `model` default that fails the contract is only a **warning** (LSP and doctor).
  - Defaults are not validated at bind time.
- **Repeatable enum inputs** check each element.
- **Where these rules are enforced:**
  - In the macro, workflow, and config loaders (all go through the Rust validator).
  - **Not** in `InputArg.__post_init__`, so gate inputs, which share `InputArg`, keep
    today's declaration rules.
  - The membership check in `InputArg.validate_and_convert` does move to Rust, so gates
    also gain did-you-mean.

### The model contract

**Refinement:** a `model` value is valid exactly when `%model:<value>` would be accepted
by the directive parser **and** route to a provider without the silent default-provider
fallback. The research cited `_model_token_routes` as this predicate. The code shows
that helper is looser than routing: it accepts any `x/y`, but `registry.py`
`resolve_model_provider_with_cursor` only routes `x/y` when `x` is a registered
provider. The type follows routing, not the helper. Remote `%dispatch` does not weaken
this, because the target processes the remaining directives and expansion
(`docs/remote_dispatch.md`), so the check runs on the machine that routes.

Classification over a `ModelValiditySnapshot`:

- **Fields:** `schema_version`, `providers` (including hidden ones such as `fakey`),
  `models` (the `model_to_provider` keys, including hidden models such as
  `fakey-large`), `aliases` (`model_alias_names()`: configured, plugin-provided, and
  implicit size aliases), and the effort levels from `effort.rs`.

**Steps:**

1. Split a trailing `@<level>` only when `<level>` is a known effort level, the same way
   `split_model_effort` does.
2. `@name` is accepted when `name` is an alias, even if the alias's provider plugin is
   not installed. Otherwise it is rejected.
3. A bare token that is an alias name is rejected, with the same advice the directive
   parser gives: aliases need `@`.
4. `provider/rest` is accepted when `provider` is a registered provider. The model part
   stays open, because new models ship before catalogs update. Otherwise it is rejected:
   the provider is not installed.
5. Any other bare token is accepted when it is a known model. Otherwise it is rejected.
6. When a rejected value has a non-level `@suffix` and its body would route, the error
   says the suffix is not an effort level.

**Guarantees:**

- Validation never consumes an alias round-robin cursor, never makes network calls, and
  never checks provider availability or disable state.
- The type promises "accepted without fallback", not "executable right now".
- `%model` itself stays open and unchanged.

### Plugin-shared enums

A plugin ships `input_types.yml` at its **package root**, beside `default_config.yml`
and its `macros/` (or legacy `xprompts/`) directory:

```yaml
# sase_research_artifacts/input_types.yml
schema_version: 1
types:
  audio_edition:
    description: Narration length for guide-backed audio editions.
    choices:
      - { value: brief, description: About 4 minutes }
      - { value: full, description: About 16 minutes }
```

- Ids match `[a-z0-9][a-z0-9_-]*` (the qualified-id grammar).
- `description` and `choices` are required, and choices follow the value rules.
- Unknown keys are errors.
- A bad type is skipped with a diagnostic, and its siblings still load.
- Only static closed sets are supported in v1. A plugin that needs a live domain checks
  whether a builtin covers it first.
- Macros, including the plugin's own, always use the qualified form,
  `sase-research-artifacts@audio_edition`.

How the file is found and loaded:

- **Discovery:** Python finds each macro-plugin module from the `sase_macros` entry
  points, plus the legacy `sase_xprompts` group while `legacy_xprompt_syntax` is on. It
  keeps `ep.dist` (today `discover_macro_plugin_modules` drops it) and emits
  `{distribution, module, path}` entries.
- **Loading:** Rust loads and validates the files. The runtime calls a binding with the
  entries. The `sase lsp` wrapper exports the same entries as
  `SASE_MACRO_PLUGIN_INPUT_TYPES_JSON`. There is no Python materialization step, so
  nothing goes stale.

### Rust / Python boundary and wires

| sase-core owns                                                                                                                                                                                                                            | sase owns                                                                                                                                                                                   |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Type catalog, resolver, did-you-mean, choice rules, PyYAML-parity scalar check, enum membership, model classifier, `input_types.yml` loader and registry, choice candidates, type labels, LSP completion, diagnostics, quick fixes, hover | Plugin discovery (paths plus distributions), building the model validity snapshot from the LLM registry, TUI presentation, typed-form pickers, doctor and CLI rendering, thin adapters only |

Wire changes are additive. **`type` keeps meaning the base kind**, so older consumers
degrade gracefully: a named enum looks like `enum` with choices, and `model` looks like
`word`.

```text
InputChoiceWire       = { value, label?, description? }      // MobileInputChoiceWire gains description
MacroInputHint       += { choices: [InputChoiceWire],         // resolved closed set; empty for open domains
                          named_type: string?,                // "model", "effort", "agent", "sase-research-artifacts@audio_edition"
                          value_role: DirectiveValueRole? }
MobileXpromptInputWire += { named_type?, value_role? }        // choices already exist
InputArg (Py)        += named_type: str | None, value_role: str | None   // type stays the base InputType
InputChoice (Py)     += description: str | None
```

**Refinement:** the research called the field `type_ref`. This plan uses `named_type`,
because "ref" already means artifact references in sase. Serializers write
`type: <named_type>` when it is set and never write resolved choices back. This covers
`prompt_frontmatter._input_to_yaml` and the local-macro handoff JSON.

### Completion and editor experience

**Prompt bar (TUI).** The `edition` argument of the plugin's `#research/audio` macro
completes like this:

```text
#research/audio(edition=▏
 ┌ edition · sase-research-artifacts@audio_edition ──────────┐
 │ ▸ brief     About 4 minutes                       default │
 │   full      About 16 minutes                              │
 └───────────────────────────────────────────────────────────┘
```

- **Opening the menu.** `#m:` and `#m(k=` open the value menu immediately.
- **Filtering and ordering.**
  - With an empty prefix, rows keep declared order.
  - Otherwise, case-insensitive prefix matches come first, then the shared Rust fuzzy
    filter.
  - Accepting inserts the canonical value, quoted when the value needs it.
- **Row layout.** The label goes in a dim right column and the description in the detail
  line. The default gets a quiet `default` badge but is not preselected.
- **Chaining.** Accepting a `name=` candidate chains straight into its value menu.
- **Bool** becomes a choice list (`true`/`false`) built by the same Rust function. The
  hard-coded bool paths on both sides go away.
- **Model arguments** use the **exact** `%model` menu: alias rows on `@`, provider
  drill-down, and effort rows. Inside a `model` argument, `@` opens model aliases, never
  the generic `@` menu.
- **Type labels.** One Rust `macro_input_type_label` renders the type everywhere:
  - `staging|prod` for up to four choices;
  - `<named_type> (N)` or `enum (N)` for more;
  - the named type for domains;
  - the keyword for scalars.

  For example: `env?: staging|prod = staging`, `model: model`.

- **Typed launch form.**
  - Up to five choices: keep the cycle button, showing labels.
  - More than five: a searchable picker with descriptions.
  - `model`: the existing `ModelPickerModal`. Do not build a second model picker.

**LSP.** The same Rust candidates serve every standard client. sase-nvim needs no
changes.

- **Completion items:**
  - `CompletionItemKind::ENUM_MEMBER` for closed sets; the existing kinds for domains.
  - A `textEdit` that replaces the whole current value span, found by the argument
    parser rather than a regex.
  - `filterText` set to the value, and `sortText` set to the declared index.
  - `labelDetails.description` set to the label, and the description as documentation.
  - `isIncomplete` when the list is truncated.
  - No new trigger characters.
- **Invocation diagnostics:**
  - Closed sets are **Errors** (`invalid_xprompt_arg_choice`).
  - `model` is a **Warning** (`invalid_xprompt_arg_model`), because the editor snapshot
    can be stale and the runtime is the authority.
  - Both carry their suggestions in diagnostic `data`, which a code action turns into a
    preferred **"Replace with `brief`"** fix. Today `actions.rs` ignores
    `params.context.diagnostics`; this adds the first diagnostic-driven fix.
- **Hover** on an argument shows:
  - the type label;
  - its source (`builtin` or `plugin sase-research-artifacts`);
  - the default;
  - a value | label | description table, capped at 12 rows plus "… N more".

  The hover gap for agent-typed arguments is fixed too.

- **Frontmatter:**
  - `type:` value completion lists the whole catalog with descriptions.
  - An unknown type gets a "Change type to `enum`" fix.
  - `string` gets a "Use `line`" fix.
  - Unquoted PyYAML-typed choices get a "Quote `yes`" fix.
  - Choice diagnostics point at the offending item's range, not the whole field.

### Messages (one voice, owned by Rust)

Use backticks, add did-you-mean when a close match exists (OSA edit distance ≤ max(1,
len/3), with case-insensitive equality and prefix matches ranked first, at most 3), and
list at most 8 values before switching to a count:

- ``input `mode` has unknown type `enmu`; did you mean `enum`?``
- ``input `edition` uses `sase-research-artifacts@audio_edition`, but plugin `sase-research-artifacts` is not installed; run `sase plugin install sase-research-artifacts` ``
- ``plugin `sase-research-artifacts` declares no input type `audio_editon`; did you mean `audio_edition`?``
- ``choice `yes` must be quoted ("yes"): YAML reads it as a boolean``
- ``choice `in progress` contains whitespace; choice values are single words``
- ``choice `null` is reserved (it means "use the default")`` ·
  ``choice `prod` is declared twice``
- ``default `turbo` is not one of fast | thorough``
- ``Argument `edition` expects one of brief | full, got `breif`; did you mean `brief`?``
- ``Argument `edition` expects a sase-research-artifacts@audio_edition value (12 choices), got `x` ``
- ``Argument `claude_model` expects a model, got `opsu`: not a known model, `@alias`, or `provider/model`; did you mean `opus`?``
  - Variants: ``got `large`: model aliases need `@`; did you mean `@large`?``,
    ``got `cluade/opus`: provider `cluade` is not installed; did you mean `claude/opus`?``,
    ``got `opus@turbo`: `turbo` is not an effort level (none, minimal, low, medium, high, xhigh, max)``

### Compatibility, flags, and rollout

- **Sunset flag `strict_macro_input_types`.**
  - On (the default): unknown type names are per-macro load errors.
  - Off: they silently become `line`, as before.
  - Create it only with `sase flag new` (read `sase_flags.md` first), and test both
    states.
  - No other flag is needed. Each phase lands complete behavior, and docs land with the
    behavior they describe.
- **Value-rule tightening is direct.** No bundled macro uses `enum`, and gates are
  unaffected. `vocab` must first scan installed plugins and home and project macros for
  `choices`. If it finds offenders, it covers them with the same sunset flag.
- **Mobile contract.** Every mobile/editor wire change regenerates the contract snapshot
  (`UPDATE_MOBILE_CONTRACT=1`).
- **Cross-repo turns.** A phase that changes sase-core opens it with
  `sase repo open sase-core` and works in the printed path. When the turn commits both
  repos, the host commits the pinned sase-core sibling first and moves
  `sase-core-revision.txt` (see `docs/rust_backend.md` § The CI source revision pin).
  Never call a binding the pin does not expose.

### Out of scope (only on demand)

These need a real consumer before they are built (`decisions:corpus-before-mechanism`):

- user and project `input_types:` config;
- more domains (`tribe`, `bead`, `machine`, `duration`, `task_type`);
- narrowing (`{type: effort, choices: [low, high]}`);
- named types on gate inputs;
- a suggest-only `completion:` field;
- plugin value callbacks;
- a bare shorthand for a plugin's own types;
- LSP signature help;
- a Rust parser for whole frontmatter text.

## Phases

Every phase must:

- read `lint_and_test.md` with `/sase_memory_read` before finishing;
- run sase-core's tests for the crates it touched;
- update the docs for the behavior it lands;
- read `tui.md` first if it touches the TUI.

### One input-type vocabulary and strict enum declarations

**sase-core** (`crates/sase_core/src/`):

1. **New module `macro_input_types`.** It holds:
   - the [type catalog](#type-catalog-one-rust-table-many-projections), with scalars,
     `enum`, and `agent` in this phase;
   - `InputTypeRegistry::builtin()`, with an empty slot for plugin types;
   - `resolve_input_type(raw, &registry)`, which returns a resolved type (`base`,
     `named_type`, `value_role`, `choices`) or a typed error with suggestions;
   - the shared `suggest_closest` helper;
   - PEP 503 canonicalization and `<dist>@<id>` parsing. In this phase a qualified name
     always yields the "plugin … not installed" or "declares no input type" errors;
   - `validate_enum_choices`, which takes items as JSON values from Python or YAML
     values from Rust and returns choices plus issues with severities;
   - `pyyaml_plain_scalar_is_non_string(text)`, a port of PyYAML's implicit resolver
     regexes;
   - `check_input_value(resolved, value)` for enum membership, with the canonical
     messages.
2. **Delete the duplicate vocabularies** and route them through the module:
   - `macro_catalog/parsing.rs` (`parse_input_type`, which falls back to `line`, and
     `parse_input_choices` and `value_as_string`, which drop floats);
   - `editor/diagnostics.rs` (`parse_input_type_name`, and `parse_local_inputs`, which
     must now read choices);
   - `editor/frontmatter.rs` (`InputType`, `ALL`, `aliases`, `rule`, `parse_input_type`,
     `XPROMPT_INPUT_TYPE_EXPECTED`, `validate_input_choices`, `validate_input_default`).
3. **Frontmatter diagnostics:**
   - unknown type: Error with suggestions;
   - `string`: Warning (deprecated);
   - per-item choice ranges;
   - unquoted PyYAML-typed choice or enum default: Error;
   - shorthand-hostile characters: Warning;
   - non-member closed-set default: Error.
4. **`frontmatter_input_type_schema()`** becomes a projection of the catalog. Its fields
   grow additively with `kind`, `description`, and `source`.
5. **Align the `path` rule** in `value_matches_input_type` with the runtime: a single
   line, spaces allowed.
6. **Python bindings** in `sase_core_py` for the catalog, resolver, choice validator,
   and value check. `MobileInputChoiceWire` gains `description`; regenerate the mobile
   contract.

**sase:**

1. **Types.** `InputChoice.description`, `InputArg.named_type`, and
   `InputArg.value_role` are added.
2. **Resolver adapter.** `parse_input_type` (`src/sase/macro/loader_parsing.py`) becomes
   a thin adapter over the Rust resolver. Update every caller:
   - shortform and longform in `loader_parsing.py`;
   - `workflow_loader_parse.py`;
   - `ace/tui/modals/input_item_modal.py` and `macro_item_modal.py`;
   - `ace/tui/widgets/_frontmatter_panel_cell_editing.py`.

   Unknown types obey `strict_macro_input_types`.

3. **Choices.** Choice parsing goes through the Rust validator in the loaders. A
   closed-set default must be a string member at load time.
4. **Workflow longform** reads `choices` and `description` (`workflow_loader_parse.py`).
5. **Per-macro isolation.** Every load path catches `MacroValidationError` per macro,
   records a load issue, and skips that macro. This covers:
   - `load_macro_from_file`;
   - `parse_macro_entries` (config and local macros);
   - plugin markdown macros;
   - skills.
6. **Handoff round-trip.** The local-macro handoff JSON (`agent/multi_prompt_macros.py`)
   round-trips `choices`, `description`, `repeatable`, and `named_type`.
   `prompt_frontmatter._input_to_yaml` writes `named_type` when it is set.
7. **Enum membership.** `InputArg.validate_and_convert` delegates enum membership to the
   Rust check. Update gate tests for the did-you-mean text. Gate declaration rules stay
   unchanged.
8. **Sunset flag.** Create `strict_macro_input_types` with
   `sase flag new strict_macro_input_types -k sunset ...` and test both states.
9. **Schemas.** Add `tools/sync_macro_input_schemas` (with `--check`). It regenerates
   the input sections of `src/sase/macros/workflow.schema.json` and
   `src/sase/config/sase.schema.json` from the catalog:
   - type `enum` ∪ the `<dist>@<id>` pattern;
   - `choices` items;
   - `repeatable`, `agent`, `enum`, and `code`;
   - `string` marked deprecated.

   Add a test that fails when the schemas are out of sync, mirroring how
   `tools/sync_feature_flags_schema` is checked.

10. **Vocabulary parity test.** Run a shared corpus of values through Rust
    `value_matches_input_type` and Python `validate_and_convert` for every scalar type.
11. **Doctor check `config.macro_input_types`** (register it in
    `src/sase/doctor/checks_config.py`). It lists input-declaration load issues across
    all macro sources, plus `string` and shorthand warnings, each with suggestions.
12. **Dogfood.**
    - `src/sase/macros/pr.yml` `status` becomes `type: enum` with `wip | draft | ready`.
      Write descriptions from `workflows/commit/commit_tracking_patch.py`. This is the
      first longform-workflow enum.
    - `src/sase/macros/eval_ifs_loops.yml` changes `string` to `line`.
13. **Corpus scan.** Scan existing `choices` users before tightening (see
    [Compatibility](#compatibility-flags-and-rollout)).
14. **Docs:**
    - `docs/macros.md` Supported Types and Enum Choices (value rules, `description`,
      quoting, and the unknown-type error);
    - `docs/workflow_spec.md`: longform `choices` now works;
    - doctor ids in `docs/configuration.md`;
    - the stale placeholder text in `input_item_modal.py`.

**Must pass:**

- `choices: [yes, no]` gives the "quote it" error in the Python loader and the LSP.
- A non-member enum default is rejected at load.
- A longform workflow enum loads.
- `type: enmu` errors with a `enum` suggestion, and only that macro is skipped. Test
  with the flag on and off.
- `type: string` loads as `line` with a warning.
- `#pr(x, status=ready)` binds, and `status=Ready` fails with a `ready` suggestion.
- `#pr:ready` still binds `name` (not `status`).
- An enum local macro round-trips through the handoff.
- The schema-sync check passes.

### Choices, named types, and roles on every wire; enum completion and diagnostics in the LSP

**sase-core:**

1. **Hint wire.** `MacroInputHint` gains `choices`, `named_type`, and `value_role`
   (serde defaults; skip when empty). Update every constructor: `assist_candidates.rs`,
   `trigger_context.rs`, `diagnostics.rs`, `argument_spans.rs`, `semantic_tokens.rs`,
   and the tests.
2. **Catalog and mobile.** `CatalogInput` and `MobileXpromptInputWire` carry the
   resolved values. `assist_entries_from_catalog` and `structured_inputs()` stop
   dropping them. Regenerate the mobile contract.
3. **Context kind.** `completion_kind_for_input` keys off the resolved hint:
   - role `agent` → `MacroArgumentAgent`;
   - choices or base `bool` → `MacroArgumentValue`;
   - `path` → `MacroArgumentPath`;
   - anything else → `MacroArgumentTypeHint`.
4. **`macro_argument_choice_candidates(hint, partial, replacement, selected)`.** This is
   the single builder for closed sets:
   - It handles ordering, filtering, and the quoting/insertion rules from the design.
   - It synthesizes `true`/`false` for bool.
   - It marks the default.
   - It excludes values already selected for repeatable inputs.
5. **`macro_input_type_label(hint)`.** `format_inputs` (`macro_catalog/entries.rs`) uses
   it.
6. **Invocation diagnostics.**
   - Enum values go through `check_input_value`: an Error with the code
     `invalid_xprompt_arg_choice`. Suggestions go in `data`, and each repeatable element
     is checked.
   - Semantic-token type mismatch follows automatically.
7. **Hover** (`editor/hover.rs`): label, source, default, and a choices table. Add the
   missing `MacroArgumentAgent` case.
8. **LSP** (`crates/sase_xprompt_lsp`):
   - `MacroArgumentValue` uses the choice builder, with the item fields from the design.
     Delete `bool_completion_list`.
   - The diagnostic-driven "Replace with" code action.
   - Frontmatter `type:` completion from the catalog.
   - "Change type to" / "Use `line`" / "Quote it" quick fixes.
9. **Golden fixtures.** Add
   `crates/sase_core/tests/fixtures/macro_arg_choice_completion.json`. Each case has
   text, cursor, and input declarations, and expects a context kind, candidate values
   with insertions, and a replacement span. Cover:
   - positional, named, and colon forms;
   - repeatable inputs;
   - `,`/`+` quoting;
   - non-BMP positions;
   - `#pr:ready`, which binds `name`, not the `status` enum;
   - bool.

   Rust tests drive it. Mirror it in sase's `tests/fixtures/` with a parity check, the
   same way `macro_args_corpus.json` mirrors `xprompt_args_corpus.json`.

10. **Bindings** for the candidate builder and the label.

**sase:**

1. **Projections.** `StructuredCatalogInput` (`macro/_catalog_models.py`,
   `_catalog_structured.py`), the mobile helper catalog
   (`integrations/_mobile_helper_catalog.py`), and the highlight wire
   (`macro/highlight.py`) carry `choices`, `named_type`, and `value_role`.
2. **Labels.** `macro/_catalog_format.py` uses the Rust label.
3. **`sase macro show`.** `ShowInput` and its renderer show the type label and the
   choices with labels and descriptions.
4. **Docs:** `docs/editor.md` (completion, diagnostics, quick fixes, hover) and the
   argument-completion notes in `docs/macros.md`.

**Must pass:**

- Golden fixtures pass in Rust.
- A JSON-RPC test completes `#deploy(env=` to `staging, prod` with `textEdit`s that
  cover the whole value.
- `edition=breif` gives an Error whose quick fix produces `brief`.
- Hover lists the choices.
- After applying a completion edit, the runtime binder accepts the same value.

### Enum choice menus in the prompt bar, typed form, and authoring modals

1. **Hint fields.** The Python `MacroInputHint`
   (`ace/tui/widgets/_macro_arg_assist_models.py`) gains `choices`, `named_type`, and
   `value_role`. They are filled in `_macro_arg_assist_catalog.py` and
   `input_hint_from_input_arg`.
2. **Kind detection.** `_macro_arg_assist_detection.py` derives the kind from the role
   and choices, as Rust does: agent role → `macro_arg_agent`; choices or bool →
   `macro_arg_value`.
3. **Candidates.** `_file_completion_macro_args.py` builds `macro_arg_value` candidates
   through the Rust choice builder and deletes the hard-coded bool filter.
4. **Kind dispatch sites.** Every site that dispatches on kind treats value menus
   consistently:
   - auto-open (`_file_completion_open.py`);
   - Ctrl+T, menu re-filter, and single-candidate chaining (`_file_completion_tab.py`);
   - refresh (`_file_completion_refresh.py`);
   - soft completion (`prompt_completion.py`);
   - Ctrl+N/Ctrl+P;
   - `_MACRO_ARG_CHAIN_KINDS` (`_file_completion_accept.py`).
5. **Panel rendering.**
   - Columns: value, dim label, and the description as the detail line, plus the quiet
     `default` badge.
   - Title: `<input> · <named_type or enum>`.
   - Subtitle: a filter hint for long sets.
6. **Type labels.** Input hints and the active-argument hint panel use the Rust type
   label (`_macro_arg_assist_inputs.py`, `_prompt_input_bar_completion_panel.py`).
7. **Typed launch form** (`widgets/typed_input_form.py`):
   - Up to five choices: the cycle button, showing labels.
   - More than five: a new searchable choice picker with descriptions, filtered with the
     Rust fuzzy binding.
8. **Authoring.** The type pickers in `input_item_modal.py`, `macro_item_modal.py`, and
   `_frontmatter_panel_cell_editing.py` list the full catalog with descriptions.
   - Choosing `enum` reveals a Choices editor validated by the Rust choice rules.
   - This fixes the uncaught `__post_init__` crash.
9. **Parity test.** Drive the mirrored golden fixtures through Python detection and Rust
   candidates.
10. **Visual snapshots** (`tests/ace/tui/visual/`): the enum choice menu (dark and
    light, 120x40) and the typed-form picker.
11. **Help and docs.** Keep the help popup in sync (`src/sase/ace/CLAUDE.md`). Update
    `docs/ace.md` macro argument completion.

**Must pass:**

- `#pr(x, status=` opens `wip/draft/ready` with descriptions.
- Accepting `status=` from the `#pr(x, ` name menu chains into the same value menu.
- Bool arguments still complete `true`/`false`.
- Golden fixtures pass from Python.
- Snapshots are approved.

### Builtin model and effort types with one routing classifier

**sase-core:**

1. **Catalog entries.** `effort` is a named_enum. Its choices come from `effort.rs`
   `EFFORT_LEVELS_ORDERED`, with the existing `EFFORT_SUGGESTIONS` text as descriptions.
   `model` is a domain with base `word` and role `model`.
2. **Classifier.** Add `ModelValiditySnapshot` and
   `classify_model_value(value, &snapshot)`, implementing
   [the contract](#the-model-contract) and its messages. Add a binding for it.
3. **Snapshot in the catalog file.** `model_catalog.json` gains an optional top-level
   `routing` object holding the snapshot. This is additive under `schema_version: 1`.
   `load_model_catalog` (`sase_xprompt_lsp/src/server/catalogs.rs`) reads it. When it is
   absent, model diagnostics are skipped.
4. **Completion kind.** Role `model` → a new `MacroArgumentModel` context kind, named
   following the existing variant conventions. The LSP serves it with the rich `%model`
   list (`model_completion_list`), including `@` aliases and full-span `textEdit`s.
5. **Diagnostics.**
   - An invalid model argument is a Warning (`invalid_xprompt_arg_model`) with a
     "Replace with" fix.
   - A frontmatter `model` default that fails is a Warning.
   - A non-member `effort` default is an Error.
6. **Hover** for model arguments: the contract in one line, plus the classification
   result (for example, "routes to `claude`").

**sase:**

1. **Snapshot builder.** Add `model_validity_snapshot()` under `src/sase/llm_provider/`.
   It is built from `_provider_names()`, `model_to_provider_map()`, and
   `model_alias_names()`, including hidden providers, and cached by config token.
   `integrations/macro_lsp.py` `_materialize_model_catalog` writes it as `routing`.
2. **Binder.** `InputArg.validate_and_convert` with `named_type == "model"` calls the
   Rust classifier. Defaults are not validated at bind time.
3. **Doctor.**
   - `config.model_macros` (`doctor/checks_config_macros.py`) classifies through the
     classifier. Delete `_model_token_routes`. Keep the existing output format and the
     retired-alias guidance.
   - `config.macro_input_types` gains "model default does not route" warnings.
4. **Parity test.** Run a token corpus through the classifier and
   `resolve_model_provider_with_effort` plus the directive parser:
   - accepted ⇒ routed, with no fallback and no directive error;
   - rejected ⇒ falls back or raises a directive error.
5. **Docs:** `docs/llms.md` (the model input type and its contract) and the
   `docs/macros.md` types table.

**Must pass:**

- Accepted: `@large`, `claude/opus@xhigh`, `codex/new-model`, hidden `fakey-large`, and
  `fakey/fakey-large`.
- `opsu` is rejected with a suggestion.
- `@lareg` is rejected with a `@large` suggestion.
- Bare `large` is rejected with an `@large` suggestion.
- `cluade/opus` is rejected with a provider message.
- `opus@turbo` is rejected with an effort message.
- The cursor state (`~/.sase/llm_lb.json`) is untouched by validation.
- `%model:opsu` still falls back exactly as before.
- `type: effort` completes the seven levels in the LSP.

Use fixture snapshots, not the live registry, wherever possible.

### Plugin-shared enums, sase macro types, and plugins.required

**sase-core:**

1. **Loader.** Load `input_types.yml` (schema v1) into the registry, with per-type
   isolation and file/line diagnostics.
2. **Resolution.** `<dist>@<id>` resolves through the registry, with the missing-plugin
   and unknown-id messages.
3. **LSP.**
   - Load the registry from `SASE_MACRO_PLUGIN_INPUT_TYPES_JSON` with each catalog load
     and refresh.
   - Frontmatter diagnostics, `type:` completion, and hover use it. Hover shows the
     provenance (`plugin sase-research-artifacts`).
4. **Bindings.** A binding loads the registry from the entries, and the resolver accepts
   it. Frontmatter validation bindings take an optional registry.

**sase:**

1. **Discovery.** `discover_macro_plugin_input_type_files()` (in or beside
   `src/sase/main/plugin_discovery.py`) keeps `ep.dist` and finds the file at each
   module's package root.
2. **Runtime registry.** Cache it per process, keyed by path and mtime. The loaders
   resolve with it.
3. **LSP export.** `integrations/macro_lsp.py` exports the env var, following the
   existing env and legacy-mirror conventions.
4. **`sase macro types [NAME] [-j/--json]`.** Read `cli_rules.md` first. The command:
   - prints a colored table grouped as Scalar / Builtin / Plugin, with name, kind,
     values or rule, and source;
   - with `NAME`, prints a detail card with choices, labels, descriptions, and the
     source file;
   - with `-j/--json`, prints JSON from the Rust catalog.

   The unknown-type message gains ``(run `sase macro types` to list them)``.

5. **Doctor `config.macro_input_types`** adds:
   - registry file diagnostics;
   - macros that use a type from a plugin that is not installed;
   - project-scoped macros (project `sase/macros/`, project config) that use a plugin
     type whose distribution is missing from `plugins.required`, as a WARN naming the
     entry to add (`src/sase/plugins/required.py` conventions).
6. **Docs:** `docs/plugins.md` gets a "Shipping input types" section; update
   `docs/macros.md` and `docs/cli.md` if CLI commands are listed there.

**Must pass:**

- A missing plugin skips only the macro that uses it, with a message naming the plugin.
- A bad type in `input_types.yml` is diagnosed while its siblings load.
- Two macros that use the same type validate identically.
- The LSP resolves plugin types with no materialization step.
- The `plugins.required` warning fires for a project macro.
- `sase macro types` and `sase macro types effort` render, and `-j` output parses.

### Model arguments use the %model menu and model picker in the TUI

1. **Kind.** Role `model` → a new `macro_arg_model` kind in
   `_macro_arg_assist_detection.py`.
2. **Candidates.** Build them through
   `synthetic_directive_clause(kind="directive_argument", directive_name="model", value_role="model", ...)`
   and `build_directive_clause_candidates`. Rows, provider drill-down reopen, alias rows
   on `@`, and effort rows are then identical to `%model`.
3. **Plumbing.**
   - Add the kind to every dispatch site touched in `tui-enum`.
   - `CompletionPanelKinds.model` is true for it, so model columns and subtitles apply.
   - `name=` chains into it.
4. **`@` precedence.** `@` inside a model argument shows alias rows and never the
   artifact menu. Add a test.
5. **Typed form.** `model` inputs open `ModelPickerModal` and write back its selection.
   `effort` uses the choice path from `tui-enum`.
6. **Snapshot and docs.** Add a visual snapshot of a macro model-argument menu. Sync the
   help popup and `docs/ace.md`.

**Must pass:**

- The model-argument rows for `#research_swarm(claude_model=` equal the `%model:` rows
  for the same prefix.
- Provider drill-down works inside the argument.
- `@` lists aliases.
- The typed form round-trips a picker choice.

### Dogfood, documentation, and memory

1. **sase-research-artifacts.** Open it with `sase repo open sase-research-artifacts`.
   - Add `src/sase_research_artifacts/input_types.yml` with `audio_edition` (brief/full,
     with descriptions).
   - In `xprompts/research_swarm.md`, `audio_edition` becomes
     `sase-research-artifacts@audio_edition`, and the nine `*_model` inputs become
     `type: model`.
   - In `xprompts/research_audio.md`, `edition` becomes
     `sase-research-artifacts@audio_edition`.
   - The plugin's hatch packaging already ships the package directory, so the YAML ships
     too. Confirm that with a wheel build or the plugin's tests.
   - Older sase versions degrade these types to `line`, so no version floor is needed.
2. **sase's own macros.** Scan `src/sase/macros/` and other bundled macro sources for
   word inputs that are really models, efforts, or closed sets, and type them.
3. **Docs.** Rewrite the `docs/macros.md` input-type section around the one rule:
   - scalar → enum → builtin → plugin;
   - examples, the value rules, the model contract, and a link to `sase macro types`;
   - cross-links to `docs/editor.md`, `docs/ace.md`, `docs/plugins.md`, and
     `docs/llms.md`.

   Remove any remaining stale type lists outside `docs/blog/`.

4. **Memory** (the user approved this by agreeing with the research's parity
   recommendation). Use `/sase_memory_write`, then run `sase memory init`. In
   `sase/memory/macros.md`, replace the Inputs bullet
   (`word/line/text/path/int/bool/float; defaultless means required.`) with one line:
   `type` names the value — scalars (word/line/text/path/int/float/bool/code), `enum` +
   `choices`, builtin `agent`/`model`/`effort`, plugin `<dist>@<id>`; defaultless means
   required.
5. **End-to-end parity test.** Use one fixture macro with an inline enum, `effort`,
   `model`, and a plugin type. The runtime binder, LSP invocation diagnostics, and TUI
   candidates must accept and reject the same values.
6. **Flag bead.** Confirm the `strict_macro_input_types` flag bead and its thresholds
   exist. Do not remove the flag in this epic.

**Must pass:**

- The parity test passes.
- `#research_swarm(claude_model=opsu)` fails at bind time with a suggestion instead of
  launching on the default provider.
- `#research/audio:breif` is rejected with a `brief` suggestion.
- `sase doctor -C config.macro_input_types` is clean on this project.

## Risks

- **A stale LSP model snapshot.** Model diagnostics are warnings only, and the wrapper
  replaces the snapshot atomically when it re-materializes. Do not build a live editor
  channel.
- **Behavior changes.**
  - Unknown types and stricter value rules can skip user macros that were previously
    degraded without notice. Mitigations: the corpus scan, the doctor check, and the
    sunset flag.
  - `#pr` status values become case-sensitive members.
- **Large sets.** Hints, hovers, and errors truncate to a count. The completion menu
  holds the full set.
