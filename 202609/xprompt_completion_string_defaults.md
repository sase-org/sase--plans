---
tier: tale
title: Show string input defaults in the TUI xprompt completion menu
goal:
  Optional xprompt inputs with string defaults (e.g. `#split_epic`'s `lang` = "Rust")
  show `=<default>` in the prompt-bar completion and argument-hint surfaces, while the
  mobile gateway keeps redacting string defaults.
size: small
proposed_by: bbugyi200.athena.0u7
create_time: 2026-09-29 18:40:18
status: wip
---

# Show string input defaults in the TUI xprompt completion menu

## Problem

The prompt input's xprompt completion menu renders each optional input as
`name?: type=<default>`, falling back to `name?: type?` when no default is displayable.
For `#split_epic` (defined in the user's `~/.config/sase/sase.yml`) it shows:

```
file_count?: int=10
lang?: word?              <- default "Rust" is missing
max_line_count?: int=1500
```

`sase xprompt show split_epic`, the xprompt select/browser modal previews, and the
prompt preview all show `default: Rust` correctly. Only the prompt-bar assist surfaces
(the completion row, the post-accept "xprompt args" hint panel, and the
keyword-argument-name rows) drop it.

## Root cause

The TUI builds its xprompt assist entries in
`src/sase/ace/tui/widgets/_xprompt_arg_assist_catalog.py::build_xprompt_assist_entries()`
by projecting the **mobile-safe** structured catalog,
`build_structured_xprompts_catalog()`, and copying each
`StructuredCatalogInput.default_display` verbatim.

That projection's `default_display()` in `src/sase/xprompt/_catalog_structured.py`
deliberately returns `None` for every `str` default:

```python
def default_display(default: object) -> str | None:
    if default is UNSET or default is None or isinstance(default, str):
        return None
    ...
```

This redaction is intentional for the mobile gateway. The sase-27 epic plan
(`plan:202605/mobile_xprompt_argument_hints.md`) says: "Until xprompt inputs support a
first-class `sensitive` flag, emit `default_display = null` for string defaults. Numeric
and boolean defaults may be displayed." The TUI later reused that projection for its
local assist entries and inherited the network privacy policy by accident.
`_default_suffix()` in `src/sase/ace/tui/widgets/_xprompt_arg_assist_inputs.py` then
renders `?` for a `None` `default_display`, which produces `lang?: word?`. So int/bool
defaults (`=10`, `=1500`) show and every string-typed default (`word`, `line`, `text`,
`path`, ...) is hidden.

The TUI is also inconsistent with itself. Prompt-frontmatter local xprompts and
workflow-derived entries go through `input_hint_from_input_arg()` /
`_default_display_from_input_arg()`, which keep string defaults. The same
`lang: {type: word, default: Rust}` input therefore shows `=Rust` when it is defined as
a local `#_helper` and `?` when it comes from the global catalog. The existing test
`tests/ace/tui/widgets/test_xprompt_arg_assist.py::test_assist_adapter_preserves_structured_catalog_fields`
pins the buggy inherited behavior (`("string_default", "line", False, None, 1)`).

## Boundary decision

- **Keep the mobile gateway redaction unchanged.** Its wire (`_mobile_helper_catalog.py`
  → Rust `sase_gateway` → Android) crosses a network boundary, and the sase-27 decision
  stands until a `sensitive` input flag exists. The Rust editor catalog
  (`sase_core/src/xprompt_catalog/parsing.rs::default_display`) mirrors that policy, and
  the xprompt LSP never renders `default_display`. **No sase-core change** is needed.
- The TUI is a local, single-user display. Every other local surface
  (`sase xprompt show/list`, preview modals) already prints string defaults, so the
  prompt-bar assist entries should too. The fix is an explicit opt-in on the Python
  structured projection, used only by the TUI adapter. The projection and adapter
  already live in this repo as Python glue, so no Rust wire or API changes.

## Implementation

### 1. Opt-in string defaults on the structured projection (`src/sase/xprompt/_catalog_structured.py`)

- Add a keyword-only `include_string_defaults: bool = False` parameter to
  `build_structured_xprompts_catalog()`, `structured_entry()`, and
  `structured_inputs()`, and thread it through: `build_structured_xprompts_catalog` →
  `structured_entry(entry, include_string_defaults=...)` →
  `structured_inputs(..., include_string_defaults=...)` →
  `default_display(inp.default, include_strings=...)`.
- Change `default_display(default, *, include_strings: bool = False)`:
  - `UNSET` / `None` → `None` (unchanged).
  - `str` → `None` unless `include_strings` is true. When it is true, return the string
    as-is, but return `None` for an empty string `""`. That matches
    `_default_display_from_input_arg`, so an empty default still renders as `?`.
  - bool/int/float handling is unchanged; any other type still returns `None`.
- Update the module docstring and the `build_structured_xprompts_catalog` docstring.
  They should say the projection is mobile-safe by default, and that
  `include_string_defaults=True` is for local, non-network consumers (the TUI) only. The
  mobile helper must never pass it.
- Keep the private aliases at the bottom of the module (`_default_display`,
  `_structured_inputs`, `_structured_entry`) pointing at the updated functions.

### 2. Pass the flag through the public facade (`src/sase/xprompt/catalog.py`)

- Add the same keyword-only `include_string_defaults: bool = False` parameter to the
  facade `build_structured_xprompts_catalog()` and forward it to `_build(...)`.
- Do **not** change `src/sase/integrations/_mobile_helper_catalog.py`. It must keep
  calling without the flag.
  `tests/test_mobile_helpers.py::test_xprompt_catalog_bridge_returns_structured_projection`
  already asserts the exact kwargs dict, so it guards this.

### 3. TUI adapter opts in (`src/sase/ace/tui/widgets/_xprompt_arg_assist_catalog.py`)

- In `build_xprompt_assist_entries()`, call
  `build_structured_xprompts_catalog(project=project, include_string_defaults=True)`.
- Add a short comment explaining why. The TUI is a local display, and the string-default
  redaction exists only for the mobile wire.

### 4. Keep multi-line defaults on one row (`src/sase/ace/tui/widgets/_xprompt_arg_assist_inputs.py`)

String defaults can now reach the prompt-bar rows, and `text`-typed defaults may span
several lines. `append_input_hints()` renders `=<default>` inline, and
`show_xprompt_arg_hint()` sizes the panel from the content's line count. An embedded
newline would therefore break the row layout.

- In `_default_suffix()`, compact the value with `single_line_default()` from
  `sase.xprompt.properties` before formatting `=<value>`. That is the same helper the
  TUI preview properties panel uses: first line, plus ` …` when more lines follow.
- If the compacted value is empty (for example, a default that is only newlines), fall
  back to `?`.
- This compaction is display-only. Leave `XPromptInputHint.default_display` holding the
  full value, because `sase.xprompt.highlight.xprompt_arg_assist_entries_to_wire`
  forwards it unchanged.
- `input_default_suffix()` delegates to `_default_suffix()`, so the
  keyword-argument-name rows in `_prompt_input_bar_completion_rows_simple.py` pick this
  up automatically. No change is needed there.

## Tests

Update or add focused unit tests. Visual PNG goldens build `XPromptInputHint` fixtures
directly with `default_display=None`, so they are unaffected and need no regeneration.

1. `tests/test_xprompt_catalog_structured.py`
   - Keep `test_structured_catalog_input_metadata_filters_step_inputs` unchanged. The
     default projection still reports `("string_default", "line", False, None, 1)`,
     which is the mobile redaction regression guard.
   - Add a test that `build_structured_xprompts_catalog(include_string_defaults=True)`
     yields `default_display == "secret"` for the string default. The same test should
     check that `None`/int/bool rows are unchanged, and that an empty-string default
     still yields `None`.
2. `tests/ace/tui/widgets/test_xprompt_arg_assist.py`
   - Update `test_assist_adapter_preserves_structured_catalog_fields` to expect
     `("string_default", "line", False, "secret", 1)`.
   - Extend `test_input_label_formatting_and_rich_rendering` (or add a sibling test)
     with a `InputArg(name="lang", type=InputType.WORD, default="Rust")` input. Assert
     that the rendered `append_input_hints` text contains `lang?: word=Rust`. This is
     the exact `#split_epic` symptom.
   - Add a parity test. For the same `InputArg` list containing a string default, the
     catalog adapter (`build_xprompt_assist_entries`) and the local/workflow adapter
     (`xprompt_assist_entry_from_workflow`) must produce the same `default_display` for
     the string input.
   - Add a test that a multi-line string default renders on one line:
     `append_input_hints` output contains `=first …` and no embedded newline inside that
     input's row. Also check that `input_default_suffix` returns the same compacted
     suffix.
3. `tests/test_mobile_helpers.py`: no edit expected. Confirm the exact-kwargs assertion
   still passes, which proves the mobile bridge did not opt in.

## Verification

- Read the `lint_and_test` reference memory note before finishing, and follow its
  required recipe (`just check` through `sase tool run`, per the guarded-recipe rules).
- Run the targeted tests first:
  `tests/test_xprompt_catalog_structured.py tests/ace/tui/widgets/test_xprompt_arg_assist.py tests/test_mobile_helpers.py tests/ace/tui/widgets/test_xprompt_arg_hints.py tests/xprompt/test_argument_surface_parity.py`.
- Manual sanity check (optional): a `#split_epic`-like xprompt with a `word` default
  renders as `lang?: word=Rust` in the prompt completion menu.

## Out of scope

- A first-class `sensitive` input flag and lifting the mobile/editor-catalog redaction.
  That is a separate cross-repo product decision that touches sase-core, the gateway
  contract, and Android.
- Displaying list/dict defaults in the structured projection. They stay `None` there,
  exactly as today.
