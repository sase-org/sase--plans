---
tier: tale
title: Finish and land epic sase-1g4 (named macro input types)
goal:
  "The named macro input types epic sase-1g4 has no remaining epic-caused defects and is
  closed: no Python stale-binding fallbacks, call-scoped TUI span grouping, correct
  `sase macro types` output, catalog-backed config-macro modal validation,
  design-conformant value-menu titles, one hint-to-wire serializer, complete CLI docs,
  and the epic bead closed with its plan file marked done."
size: medium
proposed_by: bbugyi200.athena.sase-1g4.land
bead: sase-1g4
create_time: 2026-10-05 20:26:13
status: wip
---

- **PARENT:**
  [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:**
  [sase-1g4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/README.md)

# Finish and land epic sase-1g4 (named macro input types)

## Context

Epic `sase-1g4` ("Named macro input types: finish enum, add model/effort, share plugin
enums") has all seven phases closed. Its land agent checked the delivered work against
the epic plan (`plan:202610/macro_named_input_types.md`), the source, and the commits
(sase `2a55a03deb`…`abcfedc31d`; sase-core up to `af5df614`, which is the pinned
`sase-core-revision.txt` and core `origin/master`). It also checked every commit that
landed during the epic. No non-epic commit needs integration changes.

The land agent recorded the full triage on the epic as bead note #3; read it with
`sase bead read sase-1g4 --no-links -r "<why>"`. Always pass `--no-links`, because plain
`sase bead read` currently fails on a link-validation bug (`sase-1gs`). The follow-up
proposals are already resolved: tasks `sase-1gz`, `sase-1h0`, and `sase-1h1` were filed,
and duplicates `sase-1br`, `sase-1fy`, `sase-1gp`, `sase-15h`, and `sase-1gs` were
corroborated. Do not file or +1 them again.

Seven defects caused by the epic remain. They are the scope of this tale. Fix them,
verify, then close the epic (final section). Do not create commits; the host commits
after your turn. Before touching the TUI, read the `tui.md` reference memory with
`/sase_memory_read`. Before finishing, read `lint_and_test.md` the same way.

The governing rule is `decisions:rust-core-required`: shared backend behavior lives in
sase-core, with no Python fallback for a missing or stale `sase_core_rs` binding. The
pinned core exposes every binding used below with the call shapes used here:

- `macro_argument_choice_candidates(dict)`
- `macro_input_type_label(dict)`
- `macro_argument_spans(text, entries=None)`
- `check_input_value(dict)`
- `classify_model_value(dict)`
- `macro_input_type_catalog(request=None)`
- `frontmatter_input_type_schema(registry=None)`
- `validate_frontmatter(text, registry=None)`
- `load_macro_input_type_registry(dict)`

No test depends on the fallbacks.

## Work

### 1. Remove the stale-binding fallbacks the epic added

Delete each guard below and call the binding or adapter directly, so errors propagate.
Search by symbol, because line numbers drift.

- `src/sase/ace/tui/widgets/_file_completion_macro_args.py`, in the value-candidate
  builder:
  - Delete the `try/except` around `choice_candidates_for_hint`. This includes the
    "Stale wheel" branch with its hard-coded `("true", "false")` bool list, which the
    epic plan said must go.
  - Delete the `try/except` around `type_label_for_hint`, whose fallback is
    `active.named_type or active.type`.
  - Import the adapter functions at module level unless that creates an import cycle.
- `src/sase/ace/tui/widgets/_macro_arg_assist_detection.py`,
  `_rust_span_bounds_for_cursor`:
  - Delete the guards around the `require_rust_binding` import, the
    `macro_argument_spans` call, the byte/offset conversions, and the span-shape
    parsing.
  - Return `None` only when the cursor is not on a value position.
- `src/sase/ace/tui/widgets/_macro_arg_assist_inputs.py`, `_type_label_for_hint`: return
  `type_label_for_hint(input_hint)` directly, or inline it.
- `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_simple.py`: the input-row
  `type_text = type_label_for_hint(...)` `try/except`.
- `src/sase/ace/tui/widgets/typed_input_form.py`:
  - the `try/except` in `_type_label_for_arg`;
  - the `except AttributeError:` "Stale wheel" clause in `_validate_field`;
  - the `except Exception: return {}` in `_load_type_rules`. This one predates the epic,
    but it is the same violation in the same file.
- `src/sase/macro/cli_types.py`:
  - `_catalog_entries`: delete the `except TypeError: entries = binding()` retry. It
    silently drops plugin types.
  - `_normalize_distribution`: call `sase.version._utils.normalize_distribution_name`
    directly, without the hand-written fallback.
- `src/sase/macro/frontmatter_schema.py`:
  - `_registry_argument`: delete its two broad guards.
  - `input_type_schema` and `validate_frontmatter`: delete their `except TypeError`
    retries.
- `src/sase/macro/_loader_parsing_inputs.py`, `_plugin_registry_snapshot`: delete the
  same two guards.
  - `get_plugin_input_type_registry()` already isolates discovery and import failures
    and returns manifest problems as `diagnostics`.
  - Before deleting the guards here and in `frontmatter_schema.py`, read
    `src/sase/macro/plugin_input_types.py` and confirm that it does.
- `src/sase/doctor/checks_config_macros.py`:
  - In the model-default warning helper, the guard around
    `require_rust_binding("classify_model_value")` returns `[]`. That is a false
    all-clear, so delete it. The doctor registry already turns a check exception into an
    ERROR row.
  - Delete the per-value `try/except` guards around `classify(...)`. In the model-macro
    scan, they turn a binding failure into a misleading "does not resolve" problem. In
    the default-warning loop, they skip silently. First confirm that the binding returns
    `{ok: false, ...}` rather than raising for any string value.
  - Delete the two `try/except` blocks around `get_plugin_input_type_registry()` in
    `check_config_macro_input_types`.
  - Delete the hand-written normalize fallback near the `plugins.required` scan.
  - Keep `# noqa: BLE001` guards that protect non-binding work, for example building the
    Python `model_validity_snapshot()` from config.

Keep the legitimate guards:

- per-macro load-error isolation;
- plugin import and discovery;
- file IO and `OSError` on stat;
- `MacroValidationError` and `ValueError` handling of user input;
- `query_one` lookups;
- the LSP-startup guard in `src/sase/macro/model_completion.py`;
- the `strict_macro_input_types` unknown-type→`line` path.

### 2. Replace the +64-byte span window with call grouping

`_rust_span_bounds_for_cursor` (in `_macro_arg_assist_detection.py`) currently keeps
every span whose `call_name` matches and whose start lies in
`[ref_start, max(cursor, ref_end) + 64]` bytes. Two things go wrong:

- Values from a later call to the same macro within 64 bytes leak into `selected`, so
  repeatable choice menus hide values that the current call never used.
- Values more than 64 bytes past the end can be missed.

Group spans by call instead. `macro_argument_spans` emits `arg_delimiter` spans for the
opening `(` or `:`, for each `,`, and for the closing `)`. For example,
`#pr(x, status=wip) then #pr(y, status=ready` yields `( x , status = wip )` and then
`( y , status = ready`.

- Start at the opening delimiter span located at this reference's base end, right after
  `#name`.
- Take subsequent spans until the call's closing `)` delimiter, or until the next
  opening `(` / `:` delimiter, which belongs to a new call.

Add unit tests in `tests/ace/tui/widgets/` with a repeatable enum input:

- With the cursor in the first of two adjacent calls (`#m(tags=a, tags=) #m(tags=b)`),
  `b` stays selectable.
- A call whose earlier values exceed 64 bytes still excludes all of its own selected
  values.

The quote-blind comma splitter that picks the active input predates the epic and is
tracked separately as `sase-1h1`. Leave it alone.

### 3. Fix `sase macro types` rendering (`src/sase/macro/cli_types.py`)

- **Literal markup.** `_render_detail` prints `[bold]<name>[/bold]` literally, because
  the console uses `markup=False`. Render the name with a bold `rich.text.Text`, or with
  `style="bold"`.
- **Duplicate rule line.** Print the `rule:` line only when it differs from the
  description. Builtin and plugin cards currently repeat the same sentence.
- **Color when piped.** `_console()` forces `color_system="256"`, so ANSI codes leak
  into pipes. Let Rich auto-detect the terminal: drop `force_terminal=False` and the
  forced color system, keep `markup=False`, `emoji=False`, and `highlight=False`, and
  keep the width logic.
- **Unused import.** Remove the unused `resolve_color` import.
- **Tests.** Extend `tests/macro/test_plugin_input_types.py` or the nearest CLI test.
  Check that `handle_types` output for `effort` has no `[bold]` and no ANSI escapes when
  captured, and has a single description line.

### 4. Route MacroConfigEntryModal type validation through the catalog

`src/sase/ace/tui/modals/macro_config_modal.py` is the live "new config macro" modal,
opened from `macro_browser_actions.py`. It validates `name type` pairs against
`_VALID_TYPES = {t.value for t in InputType}`. As a result it:

- rejects `model`, `effort`, `builtin@model`, and `<dist>@<id>` plugin types;
- accepts `enum`, even though a shortform pair cannot declare `choices`.

Change it as follows:

- **Validate** each type with `sase.macro` `parse_input_type(arg_type, name=arg_name)`,
  the Rust-backed resolver used by `input_item_modal.py`. Show the resolver's
  `MacroValidationError` text (with did-you-mean) as the modal error.
- **Reject `enum`** with a message saying that enum inputs need `choices` declared in
  the config file. Accept named enums such as `effort` and plugin types, because they
  carry their own choices.
- **Build the hint line** (`Types: …`) from the advertised catalog,
  `input_type_schema()` names. Delete `_VALID_TYPES` and the `InputType` import if
  nothing else uses them.
- **Tests.** Add or extend a modal test with these cases:
  - `effort` and `model` are accepted;
  - `enum` is rejected;
  - `enmu` shows an `enum` suggestion;
  - a plugin type resolves when a registry with that type is installed. Use a
    fixture-built registry or monkeypatch the registry snapshot. Do not depend on the
    installed plugin.

### 5. Value-menu title follows the design: `<input> · <named_type or enum>`

The design (epic plan § "Completion and editor experience"; phase note sase-1g4.3 #2)
titles the prompt-bar choice menu `edition · sase-research-artifacts@audio_edition`.
`_macro_arg_value_panel_title` in
`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel_labels.py` already has that
docstring. However, `MacroArgValueMetadata.type_label` carries the Rust type label, so
the title renders `status · wip|draft|ready`.

- Carry the title type on the metadata: `active.named_type or active.type`. That gives
  `enum` for inline enums, `bool` for bool, and the named type for `effort` or plugin
  enums.
- Have `_macro_arg_value_panel_title` use it.
- If `type_label` on `MacroArgValueMetadata` has no other reader, replace it instead of
  adding a field.
- Update the affected unit tests and the `macro_arg_completion` visual goldens. Run
  `just fix-tui-screenshots` with a selector after `--` targeting
  `tests/ace/tui/visual/test_ace_png_snapshots_macro_arg_completion.py`, then inspect
  the updated PNGs.
- Update `docs/ace.md` if it describes the title.

### 6. One hint-to-wire serializer for the shared Rust label

Three places build the same Rust `MacroInputHint` wire dict (phase note sase-1g4.3 #3):

- `_hint_to_wire` in `src/sase/ace/tui/widgets/_macro_arg_choice_adapter.py`;
- `macro_input_type_label` in `src/sase/macro/_catalog_format.py`;
- the hint builder plus `_macro_input_choice_to_wire` in `src/sase/macro/highlight.py`.

Move the dict shape and the choice normalization into one function under
`src/sase/macro/`, so each caller only maps its own object into it. The mappings differ:

- `InputArg.type` is an `InputType` enum, so pass `.value`.
- `required` is `default is UNSET` for `InputArg`.

Keep the wire output byte-identical. The existing parity and projection tests must pass
unchanged, including:

- `tests/ace/tui/widgets/test_macro_arg_choice_tui_parity.py`
- `tests/macro/test_named_input_type_parity.py`
- the highlight and catalog tests

### 7. Docs for `sase macro types`

- `docs/configuration.md` "CLI Flags" has `### sase macro …` sections for `expand`,
  `explain`, `list`, `show`, `graph`, and `catalog`, but none for `types`. Add
  `### \`sase macro
  types\``after`catalog`, covering `NAME`(bare, alias,`builtin@<name>`, or `<dist>@<id>`) and `-j/--json`,
  in the same style.
- `docs/cli.md`: point the `sase macro types` row at `macros.md#sase-macro-types`. That
  section is `### \`sase macro
  types\``in`docs/macros.md`; it currently links to `#named-plugin-input-types`.

## Verification

- Run the focused suites first:
  - `tests/macro/test_plugin_input_types.py`
  - `tests/macro/test_named_input_type_parity.py`
  - `tests/ace/tui/widgets/test_macro_arg_choice_tui_parity.py`
  - `tests/ace/tui/widgets/test_macro_arg_value_completion.py`
  - `tests/ace/tui/widgets/test_macro_arg_assist_detection.py`
  - `tests/ace/tui/widgets/test_typed_input_form.py`
  - `tests/test_macro_frontmatter_schema.py`
  - the doctor macro checks
  - the new modal and CLI tests
- Then run `just fix` and `sase tool run check`. Read triage per `lint_and_test.md`. Two
  failures are already known and unrelated, so do not chase them:
  - `test_macro_docs_and_memory_avoid_xprompt_terms` (docs/images infographic prompt,
    owned by epic sase-1eq);
  - parallel-lane TUI flakes already tracked as `sase-1fy`, `sase-1br`, and `sase-1gp`.
- Run `sase doctor -C config.macro_input_types` and confirm it is still OK.
- Run `sase macro types effort | cat` and confirm it shows no markup tags and no ANSI
  codes.
- Do not run `just check-full`.

## Closeout (final step, in this same turn)

1. Run `sase bead epic-symbols sase-1g4`. The land agent found none. If any entry
   appears, resolve it per the Symvision epic-whitelist policy (read `symvision.md` with
   `/sase_memory_read`). Re-key it only to a still-open bead that needs it.
2. Close the epic. Never use `--force`. If the close is rejected, fix the cause and
   close again.

   ```bash
   sase bead close sase-1g4 --note "<verification>"
   ```

   The note should summarize:
   - the phases verified;
   - "no integration needed";
   - the seven fixes above, with their tests;
   - the `sase tool run check` ToolRun id and verdict;
   - the follow-up outcomes from bead note #3.

3. Run `just symvision`, or `sase tool run symvision` if the raw recipe is refused, and
   confirm the whitelist is clean.
4. Change `status: wip` to `status: done` in the frontmatter of the epic's plan file.
   That is the PLAN path printed by
   `sase bead read sase-1g4 --no-links -r "Need the plan path"`, which is
   `plan:202610/macro_named_input_types.md` in the plans repo.
5. `sase-1g4` has no `parent_bead`, so the landing ends here.
