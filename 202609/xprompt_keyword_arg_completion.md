---
tier: tale
title: Selectable xprompt keyword-argument completion in the prompt input widget
goal:
  Typing an xprompt's opening argument delimiter offers its keyword inputs as a live,
  metadata-rich menu that <ctrl+n>/<ctrl+p> select and <enter> inserts as `name=`,
  chaining into that input's value menu, without ever stealing <enter> from prompt
  submission.
size: medium
proposed_by: bbugyi200.athena.0lr
create_time: 2026-09-15 22:08:28
status: wip
---

# Plan: Selectable xprompt keyword-argument completion in the prompt input widget

## Problem

Typing `#review(` in the ACE prompt input shows a panel that looks exactly like a menu
and behaves like a poster. Verified against master on 2026-09-15 with a probe driving
`CompletionTestApp`, using an entry whose inputs are `path: path`,
`enabled?: bool=true`, `count?: int=3`, `label: str`:

```
title: xprompt args                      subtitle: [:] colon  [(] named args
#review( arguments
  ▸  path: path - file to review
     enabled?: bool=true - turn on
     count?: int=3
     label: str - a free-form label
```

That is `show_xprompt_arg_hint` in
`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`. It draws a `▸` cursor,
it marks an "active input", and nothing selects it. `<ctrl+n>` and `<ctrl+p>` there fall
past the completion branch in `_prompt_text_area_key_handling.py` into
`_handle_vcs_mru_cycle_key` — which does nothing at all when the MRU ring is empty, and
rewrites the prompt's leading VCS tag when it is not. Either way, the two keys the user
reaches for to pick an input do anything except pick an input.

The real menu exists but is reachable only by pressing `<ctrl+t>` first:

```
title: xprompt arg names                 subtitle: (empty)
▸ count=
  enabled=
  label=
  path=
```

Six concrete defects, all verified:

1. **The menu never opens on its own.** `_try_auto_xprompt_arg_completion` in
   `src/sase/ace/tui/widgets/_file_completion_open.py` returns early unless
   `arg_ctx.completion_kind` is `xprompt_arg_agent`. Keyword names and bool values are
   excluded, so the only automatic assistance at `#review(` is the static hint plus a
   one-shot `[^L] accept enabled=` soft suggestion in the bar subtitle.
2. **`<ctrl+n>` / `<ctrl+p>` are inert at a keyword position.** They already drive an
   _open_ menu correctly — `_move_file_completion` and `_accept_file_completion` need no
   change — but there is no path from "cursor is at a keyword position" to "menu is
   open".
3. **The rows throw away everything the hint panel shows.** The `kinds.arg_completion`
   branch of `_append_candidate_row` in `_prompt_input_bar_completion_panel_content.py`
   renders one flat `bold yellow` / `yellow` string. Type, required marker, default, and
   description are all dropped, and the panel's `border_subtitle` is left empty.
4. **The rows are sorted alphabetically.** `_build_named_arg_completion_candidates` in
   `_file_completion_xprompt_args.py` ends with `candidates.sort(key=...)`, so the menu
   shows `count, enabled, label, path` while the hint panel, the `input_signature` on
   the xprompt row, and `build_xprompt_arg_name_candidates` in the Rust core's
   `crates/sase_core/src/editor/completion.rs` — which the xprompt LSP serves to
   external editors — all present declaration order. Three surfaces, two orders, and the
   order the user sees changes as inputs are renamed.
5. **Accepting `name=` dead-ends.** Enter on `enabled=` produces `#review(enabled=` and
   closes the menu back to the static hint. The `true` / `false` menu for that input
   exists and is one `<ctrl+t>` away, but nothing chains into it.
6. **The title never says whose arguments these are.** `xprompt arg names` is generic
   where every neighbouring menu (`vcs_ref`, `vcs_repo`, the `@` menu) titles itself
   with the thing being completed.

## Design

### One panel, two states — the hint grows the ability to be driven

The static hint and the keyword menu are the same list of the same inputs in the same
order. The design does not add a second surface; it promotes the existing one.

- **Cursor at a keyword-name position inside `(...)`** → a live, selectable menu.
- **Cursor in colon form (`#review:…`) or at a positional slot** → the static hint,
  unchanged. Colon arguments are positional; there is no keyword to insert, so a
  selectable list would be a lie.

The menu deliberately keeps the hint's visual language — the `▸` cursor the panel body
already draws for a selected row, the `#D7AF87` required / `dim #D7AF87` optional input
palette from `_xprompt_arg_assist_inputs.py`, and the `name?: type=default` label shape
from `input_label`. Someone who has been reading the hint panel for months sees the same
panel, now with the cursor under their control. That is the whole aesthetic claim: no
new chrome, one fewer dead end.

### Auto-open where the candidate set is closed; stay manual where it is open-ended

`_try_auto_xprompt_arg_completion` gains `xprompt_arg_name` and `xprompt_arg_value`
alongside today's `xprompt_arg_agent`. It does **not** gain `xprompt_arg_path`.

The rule is the candidate set, not the kind: keyword names are a short closed list off
the entry the user just typed; `xprompt_arg_value` is literally `true` / `false`; fork
targets are already auto-opened. Paths are unbounded, need disk work, and the
`auto_file_paths: false` default in `src/sase/default_config.yml` is a standing decision
that path menus do not appear uninvited. `<ctrl+t>` and the soft suggestion keep serving
paths exactly as they do today.

Adding `xprompt_arg_value` is what closes the `foo=bar` round trip the request is about:
accepting `enabled=` lands the cursor on a value position whose whole candidate set is
two rows, and the user finishes the argument without touching `<ctrl+t>`.

No new configuration key. The existing `prompt_completion.auto_xprompt_menu` flag
already gates every automatic reference menu including today's fork-target arg menu, and
this is the same class of surface. No `src/sase/default_config.yml` change is needed for
keys either: the completion navigation keys are hardcoded literals in
`_prompt_text_area_key_handling.py`, not entries under `keymaps:`.

### `<ctrl+n>` / `<ctrl+p>` open the menu at a keyword position

Even with auto-open, the user can dismiss a menu, or run with
`auto_xprompt_menu: false`. In both cases `<ctrl+n>` at `#review(` today reaches
`_handle_vcs_mru_cycle_key`, which either does nothing or — once the MRU ring has
entries — edits the prompt's leading VCS tag, a distant part of the prompt the user was
not looking at.

So: when no completion menu is open and the cursor resolves to an `xprompt_arg_name`
context, `<ctrl+n>` opens the menu with the **first** row selected and `<ctrl+p>` opens
it with the **last** row selected. This branch is placed before the MRU branch in
`_on_key`. MRU cycling is untouched everywhere else, including inside `(...)` at a
positional or value slot, where no keyword context resolves.

This makes the request literally true: `<ctrl+n>` / `<ctrl+p>` select the next and
previous input, from a cold start, with no prefix key.

### Enter is never stolen

An auto-opened menu that owns `<enter>` would break prompt submission: the user types
`#review(` at the end of a finished prompt, presses Enter to send, and gets `count=`
instead. The codebase already solved this for the `@` artifact menu — the
`unowned_bare_at` guard in the `enter` branch of `_prompt_text_area_key_handling.py`
declines to accept when the token is empty and `_completion_selection_moved` is False,
clearing the menu and letting the keypress fall through to submit.

Generalize the same rule, keyed on **how the menu opened**, mirroring the existing
`_placeholder_completion_trigger` field that already records `"auto"` / `"manual"` for
the lifetime of an open placeholder menu:

- **`manual`** — opened by `<ctrl+t>`, by the new `<ctrl+n>` / `<ctrl+p>` opener, or
  chained from an accept the user just made. Owns Enter immediately. An explicit request
  is an explicit handover.
- **`auto`** — opened by typing. Does **not** own Enter while the token is empty _and_
  the selection has not moved. Typing one character, or pressing `<ctrl+n>` / `<ctrl+p>`
  once, hands Enter to the menu.

Chained menus counting as `manual` is what keeps the round trip working: Enter on
`enabled=` opens the `true` / `false` menu, and the next Enter accepts `true` rather
than submitting a half-written argument.

### Declaration order, never alphabetical

Drop the alphabetical sort. Emit `entry.inputs` in declaration order, filtered by
`used_arg_names` and the typed prefix.

Declaration order is the order the hint panel shows, the order `input_signature` shows
on the xprompt row that was just accepted, and the order the Rust
`build_xprompt_arg_name_candidates` already emits to external editors over LSP. It is
also the only _stable_ order: alphabetical reshuffles when an xprompt author renames an
input, so muscle memory built on "the second row is the model" silently breaks.

Required-first was considered and rejected for the same reason — it is a second ordering
to remember, and requiredness is already carried on every row by the `?` marker and the
dim palette.

### Row anatomy and the border subtitle

One line per candidate, so the panel's `_content_line_count` height reservation and the
`text-wrap: nowrap` CSS contract both hold without change:

```
▸ path=       path            file to review
  enabled=    bool  =true     turn on
  count=      int   =3
  label=      str             a free-form label
```

- `name=` in `#D7AF87` (required) or `dim #D7AF87` (optional), bold when selected — the
  `=` is part of the insertion and part of what is shown, which is what makes the row
  read as the thing it will paste.
- the declared type, dim.
- `=default` for an optional input that has one, `?` for an optional input that does not
  — reusing `_default_suffix`.
- the description, dim, truncated to the remaining width.

Widths come from a `xprompt_arg_name` entry on the existing `_RowLayout` in
`_prompt_input_bar_completion_panel_content.py`, measured with a
`xprompt_arg_name_label_width` helper in the same shape as `placeholder_label_width` and
`vcs_ref_label_width`. Degrade by column as the panel narrows: description first, then
the default/`?` suffix, then the type. `name=` is never truncated — it is the payload.

The **selected** row's full description goes in the panel's `border_subtitle`, exactly
as `model_completion_subtitle` does for the model menu, so a long description is
readable without any row growing a second line.

Title becomes the reference whose arguments are open — `#review args` — read off new
candidate metadata rather than the generic `xprompt arg names`.

### The argument loop is user-driven

`(` → menu → `<ctrl+n>`/`<ctrl+p>` → Enter → value menu → Enter → type `,` → menu again.
Typing the comma re-resolves an `xprompt_arg_name` context with the previous keyword now
in `used_arg_names`, so the auto-open fires again with the remaining inputs.

Auto-inserting `, ` after a value accept was considered and rejected: it writes text the
user did not type, and it is wrong for the common case of a single keyword argument. One
character of user input buys a correct decision.

### Boundary note

`sase/memory/rust_core_backend_boundary.md` puts shared completion behavior in
`sase-core`, and the Rust core does already own an equivalent
`build_xprompt_arg_name_candidates` that the xprompt LSP serves. This tale deliberately
does **not** move ACE onto that binding: `classify_completion_context` and the arg-name
builder are not exposed through `sase_core_rs` today, so adopting them means a core
change, a published floor, and a version bump — the exact sequence that blocked epic
`sase-rj` for weeks. What this tale does instead is remove the one behavioral divergence
between the two implementations (the alphabetical sort), leaving ACE and the LSP
agreeing on content and order while differing only in presentation. Everything added
here is ACE presentation and keystroke routing over the existing Python detection layer;
no new grammar and no new parsing rule is introduced, so there is nothing new to keep in
sync.

## Out of scope

- Moving ACE onto a `sase_core_rs` binding for arg-name context and candidates. Worth
  doing; needs a core release first. Related: the LSP's `XpromptArgumentName` arm in
  `crates/sase_xprompt_lsp/src/server.rs` passes `&Default::default()` for
  `used_arg_names`, so external editors re-offer keywords the document already uses — a
  real defect in `sase-core` that this tale does not touch.
- `xprompt_arg_path` auto-open, and any change to `auto_file_paths`.
- Colon-form (`#review:…`) argument entry, which stays positional.
- The `%directive` argument menus, which have their own Rust-backed contract.

## Implementation

Work in the `sase` repository.

### Candidates and metadata

In `src/sase/ace/tui/widgets/_xprompt_arg_assist_models.py`, add a frozen slotted
`XPromptArgNameMetadata` carrying the reference text (`entry.insertion`) and the
`XPromptInputHint`, and export it from `xprompt_arg_assist.py`.

In `src/sase/ace/tui/widgets/_file_completion_xprompt_args.py`, change
`_build_named_arg_completion_candidates` to attach that metadata and to **stop sorting**
— iterate `ctx.entry.inputs` in declaration order, skipping `ctx.used_arg_names` and
non-matching prefixes as it does today. `PromptSoftCompletion` only reads `insertion`,
so the soft-suggestion path in `prompt_completion.py` is unaffected; confirm that with
the existing `test_soft_xprompt_arg_name_and_bool_value_suggestions`.

### Auto-open

In `src/sase/ace/tui/widgets/_file_completion_open.py`, widen
`_try_auto_xprompt_arg_completion` to accept `xprompt_arg_name` and `xprompt_arg_value`
in addition to `xprompt_arg_agent`, and record the trigger as `"auto"`. Keep it behind
`settings.auto_xprompt_menu` where its caller `_try_auto_prompt_reference_completion`
already places it, and keep it returning False when no candidates resolve so the static
hint still wins.

`sase/memory/tui_perf.md` governs this path. It stays allocation-light and disk-free:
`_get_xprompt_arg_completion_context` is already called on every keystroke by
`_structured_completion_claims_cursor` and by the soft-completion builder, and the new
candidate build is a loop over a tuple that is usually under ten elements. Add no new
catalog fetch — on a cold catalog `_get_xprompt_arg_assist_entries` returns an empty
list and the menu simply does not open, which
`test_cold_cache_auto_completion_does_not_build_catalog_sync` is the template for
pinning.

### Trigger ownership

Add `_xprompt_arg_completion_trigger: str | None` to `PromptTextArea`'s state in
`src/sase/ace/tui/widgets/prompt_text_area.py` and to the `TYPE_CHECKING` block in
`_file_completion_base.py`, mirroring `_placeholder_completion_trigger`. Set it to
`"manual"` in `_try_xprompt_arg_completion_tab`, in the new `<ctrl+n>` / `<ctrl+p>`
opener, and in the accept-chain; set it to `"auto"` in the auto-open; clear it in
`_clear_file_completion`.

In the `enter` branch of `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`,
extend the existing `unowned_bare_at` guard into one predicate covering both the `@`
case and the new one: an open menu does not own Enter when its trigger is `"auto"`, its
token is empty, and `_completion_selection_moved` is False. Keep the artifact-ref arm's
behavior byte for byte; its tests must not change.

### Opening from `<ctrl+n>` / `<ctrl+p>`

In the same file, insert a branch before the two `_handle_vcs_mru_cycle_key` branches:
when `not self._file_completion_active`, the vim mode is insert, and
`_get_xprompt_arg_completion_context()` returns a context whose `completion_kind` is
`xprompt_arg_name`, open the menu with trigger `"manual"` and set
`_file_completion_index` to `0` for `ctrl+n` or `len(candidates) - 1` for `ctrl+p`. Set
`_completion_selection_moved` so the first Enter accepts. Fall through to MRU cycling
when no context resolves or no candidates survive.

### Accept and chain

In `src/sase/ace/tui/widgets/_file_completion_accept.py`, route an `xprompt_arg_name`
accept through `ctx.value_start` / `ctx.value_end` from
`_get_xprompt_arg_completion_context()` rather than the generic `_get_token_context()`
tail, matching what `_try_xprompt_arg_completion_tab` already does. The generic path
happens to agree on the cases probed (`#review(count=1, la`, `#review(  c`), but the two
derive the replacement span differently and only one of them is the span the context
object computed; using the context's span removes the coincidence.

After the replacement, re-resolve the context and, when the new `completion_kind` is one
the panel can serve (`xprompt_arg_value`, `xprompt_arg_agent`, `xprompt_arg_path`), open
that menu with trigger `"manual"`. When it is `xprompt_arg_type_hint` — a free-text
`str` input, which `build_xprompt_arg_completion_candidates` correctly returns nothing
for — clear the menu and restore the static hint through the existing
`_refresh_xprompt_arg_hint_from_cursor()` call. Chaining into the path menu here is
correct even though path menus are not auto-opened: the user asked for this one by
accepting the keyword.

### Rendering

In `src/sase/ace/tui/widgets/_prompt_input_bar_completion_rows_simple.py`, add
`append_xprompt_arg_name_completion_row` and `xprompt_arg_name_label_width`, styled from
the `_REQUIRED_INPUT_STYLE` / `_OPTIONAL_INPUT_STYLE` / `_DEFAULT_STYLE` constants
already in `_xprompt_arg_assist_inputs.py` (export them or a small accessor rather than
re-declaring the hex values — one palette, one definition). Re-export both from
`_prompt_input_bar_completion_rows.py`.

In `_prompt_input_bar_completion_panel_kinds.py`, split today's `arg_completion` flag so
`xprompt_arg_name` is distinguishable from `xprompt_arg_value`; keep `xprompt_arg_value`
on the existing flat-yellow branch, which suits a two-row `true` / `false` list.

In `_prompt_input_bar_completion_panel_content.py`, add the `xprompt_arg_name` width to
`_RowLayout` and dispatch the new row renderer, ahead of the `kinds.arg_completion`
branch.

In `_prompt_input_bar_completion_panel_labels.py`, return `f"{reference} args"` as the
title when the rows carry `XPromptArgNameMetadata`, and add an
`xprompt_arg_name_completion_subtitle(rows, selected_index, inner_width)` in the shape
of `model_completion_subtitle`, wired into the `border_subtitle` chain in
`show_file_completions`.

## Verification

`sase/memory/lint_and_test.md` governs the gates: run `just check` before finishing, and
`just check-full` through the `/sase_monitor` skill before landing, since this touches
the shared completion panel that many suites render.

New widget tests in `tests/ace/tui/widgets/`, extending
`test_xprompt_arg_value_completion.py` and `test_xprompt_arg_assist.py` and following
the `CompletionTestApp` pattern in `tests/ace/tui/widgets/_completion_helpers.py`:

1. Typing `(` after a multi-input xprompt opens the keyword menu; `<ctrl+n>` /
   `<ctrl+p>` move the selection; Enter inserts `name=`.
2. Auto-opened menu, empty token, selection unmoved: Enter submits the prompt and does
   not insert a keyword.
3. Same menu after one typed character: Enter inserts.
4. Same menu after one `<ctrl+n>`: Enter inserts.
5. `<ctrl+t>`-opened menu at an empty token: Enter inserts immediately.
6. `<ctrl+n>` with no menu open at `#review(` opens the menu on the first row and leaves
   a seeded VCS tag untouched; `<ctrl+p>` opens on the last row. Pair with a test that
   `<ctrl+n>` at a _positional_ slot still cycles the MRU.
7. Rows are in declaration order, asserted with inputs whose declaration order and
   alphabetical order differ.
8. Row content: type, `?` / `=default`, and description present; an optional row uses
   the dim palette; a narrow panel drops description before type and never truncates
   `name=`.
9. `border_subtitle` shows the selected row's description and follows `<ctrl+n>`.
10. Title is `#review args`.
11. Accepting a `bool` keyword chains into the `true` / `false` menu and that menu owns
    Enter; accepting a `str` keyword opens no menu and restores the static hint;
    accepting a `path` keyword opens the path menu.
12. Already-used keywords are excluded after a comma, and a leading-whitespace clause
    (`#review(  c`) replaces only the typed token.
13. Colon form still shows the static hint or the positional path menu — no keyword
    menu.
14. Single-input and repeatable-first-input entries still resolve to a value kind and
    are unchanged.
15. `auto_xprompt_menu: false` suppresses the auto-open while `<ctrl+t>` still works.
16. Cold catalog opens no menu and builds no catalog synchronously.

Extend `tests/ace/tui/widgets/test_prompt_completion_height.py` to cover the new rows,
since the panel's reserved height depends on every row staying one line.

Add `tests/ace/tui/visual/test_ace_png_snapshots_xprompt_arg_completion.py` following
`test_ace_png_snapshots_history_word_completion.py`, with dark and light snapshots of
the open keyword menu over a four-input xprompt. Accept the goldens with
`--sase-update-visual-snapshots` and run `just test-visual`. Review the images before
accepting: `name=` should be the most legible thing in the row, the optional rows should
read as clearly secondary without becoming unreadable, and the panel should still look
like the hint panel it replaced. If it does not, the palette split is wrong and belongs
back in the rendering step.

Finally, confirm no keystroke regression with `SASE_TUI_PERF=1` against the p95 target
in `sase/memory/tui_perf.md`, typing a keyword argument into a prompt that already has a
warm catalog.
