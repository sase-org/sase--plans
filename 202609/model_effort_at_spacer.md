---
tier: tale
title: Typing @ after a %model value and space opens effort completion
goal:
  In the ACE prompt input, pressing @ right after `%m:<model> ` replaces the space with
  @ and opens the effort-level completion menu.
size: small
proposed_by: bbugyi200.athena.0rc
create_time: 2026-09-24 16:13:20
status: wip
---

# Plan: `@` after `%m:<model> ` swallows the space and opens effort completion

## Goal

In the ACE prompt input (INSERT mode), when the cursor sits right after a completed
colon-form `%model` value followed by a single space — for example `%m:gpt-6-sol |` —
typing `@` should **replace that space with `@`** (giving `%m:gpt-6-sol@|`) and then
open the effort-level completion menu (`none`, `minimal`, `low`, `medium`, `high`,
`xhigh`, `max`). This mirrors the existing one-shot xprompt-spacer rewrite, where `,` /
`:` replaces the trailing space after an accepted `#name ` completion.

The most common way to reach that state is accepting a `==model` / `=alias` shortcut,
which already rewrites the token to `%m:gpt-6-sol ` (with a trailing space) and leaves
the cursor after the space. Typing `%m:gpt-6-sol` by hand and then a space should behave
the same way.

## What already exists (no changes needed)

- **Effort rows for `%m:<model>@`.** The Rust directive classifier already reports
  `%m:gpt-6-sol@` (and `%m:@large@`, `%model:x@`) as a `directive_argument` clause with
  `directive_name == "effort"`, and `build_directive_clause_candidates` returns the
  seven effort levels for it (checked via `classify_directive_completion`).
- **Auto-open after the edit.** `_open_auto_reference_completion_after_change("@")` in
  `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py` →
  `_try_auto_prompt_reference_completion` (`_file_completion_open.py`) tries
  `_try_auto_directive_arg_completion()` first (gated by
  `ace.prompt_completion.auto_directive_menu`), which opens those effort rows before the
  `@` artifact-reference menu gets a chance.
- **Model-token detection.** `classify_directive_completion(line, col)` in
  `_directive_completion_tokens.py` (row-local, backed by `sase_core_rs`) classifies
  `%m:gpt-6-sol` at its end as `kind="directive_argument"`, `directive_name="model"`,
  `syntax_form="colon"`, `value_role="model"`, `token="gpt-6-sol"`, with `end` at the
  token end. It returns `None` for invalid directive contexts such as inline code
  (`` `%m:opus ``), and classifies an already-suffixed `%m:gpt-6-sol@high` as
  `directive_name="effort"`, so such text is naturally excluded.

This is presentation-only keystroke handling. It reuses the shared Rust classifier
instead of reimplementing directive parsing, so it stays on the Python side of the Rust
core boundary (the same place as the existing xprompt spacer logic).

## Design

### Stateless check at the `@` keystroke

Use a stateless, cursor-local check when `@` is pressed, not a pending "armed" flag like
`PendingXPromptCompletionSpacer`. The request covers both a completion-inserted space
and a hand-typed space, and a text check handles both without new state to invalidate.
The guards below keep it narrow enough that ordinary `@` typing elsewhere is unaffected.

Fire only when **all** of these hold:

1. The key event's character is `@`.
2. The widget is in INSERT mode. Put the check after the NORMAL/visual-mode early
   returns in `_on_key`, next to the existing `#@` trigger block, so this holds
   structurally.
3. The owning prompt bar exists and `bar._mode == "prompt"`. Skip feedback and other
   modes, matching `_try_auto_prompt_reference_completion`.
4. There is no selection (`start == end`).
5. On the current line, with `col` the cursor column: `col >= 2`,
   `line[col - 1] == " "`, and `line[col - 2]` is not whitespace (exactly one space
   between the token and the cursor).
6. The cursor is at end of line or directly before whitespace
   (`col == len(line) or line[col].isspace()`). This prevents turning `%m:opus |fix bug`
   into `%m:opus@fix bug`.
7. `classify_directive_completion(line, col - 1)` returns a clause with
   `kind == "directive_argument"`, `directive_name == "model"`,
   `syntax_form == "colon"`, `value_role == "model"`, a non-empty `token`, and
   `clause.end == col - 1` (the model token ends exactly at the space). Parenthesized
   `%model(...)` forms are out of scope.

When the check fires:

- Stop and prevent-default the event.
- Replace the single space with `@` through the existing `_replace_absolute_range`
  helper. It goes through `_replace_via_keyboard`, so the edit is undoable and
  dot-repeat bookkeeping stays consistent, and it leaves the cursor after the `@`.
- Then run the same follow-up as the xprompt-spacer path:
  `self._refresh_file_completion_from_cursor()` followed by
  `self._open_auto_reference_completion_after_change("@")`. This opens the effort menu
  when `auto_directive_menu` is on. When that setting is off, the space is still
  swallowed and the menu stays closed; manual `Ctrl+T` still opens effort rows there.
  This matches `test_optional_agent_spacer_colon_respects_disabled_auto_menu`.

### Code placement

- Add a small pure helper module, for example
  `src/sase/ace/tui/widgets/_model_effort_spacer.py`, with
  `find_model_effort_spacer(line: str, col: int) -> int | None`. It returns the column
  of the space to replace when guards 5–7 hold, else `None`. Keeping it pure makes it
  cheap to unit test and keeps `_prompt_text_area_key_handling.py` (already ~480 lines)
  from growing much. Check the cheap string guards (5, 6) before the Rust classifier
  call so the per-keystroke cost of ordinary `@` typing is a few character comparisons
  (see the TUI perf note).
- In `_prompt_text_area_key_handling.py`, inside the existing
  `if event.character == "@":` block (before the `#@` detection, which needs
  `line[col - 1] == "#"` and so cannot conflict), add a mixin method call such as
  `self._try_model_effort_spacer_rewrite()`. It applies guards 3–4, calls the helper,
  and performs the replacement and completion refresh. Return early when it handles the
  event. Add the method to the mixin's `TYPE_CHECKING` protocol stubs as needed, or
  implement it directly in the mixin. Follow whichever pattern the surrounding mixin
  uses for `_consume_xprompt_completion_spacer`.
- The top-of-`_on_key` pending xprompt spacer only consumes `,` / `:`, so `@` already
  falls through to the new code unchanged.

### Accepted trade-off

A prompt that starts with `%m:opus ` followed immediately by an `@` artifact reference
(`%m:opus @plan…`) now gets `%m:opus@` plus the effort menu. The user explicitly asked
for this; recovery is to undo (`Ctrl+Z` / NORMAL `u`) or type the space back. Mention
this in the docs sentence.

## Tests

Add `tests/ace/tui/widgets/test_model_effort_spacer.py`, modeled on
`tests/ace/tui/widgets/test_xprompt_completion_spacer.py` and
`tests/ace/tui/widgets/test_model_explicit_completion.py` (reuse `CompletionTestApp`
from `._completion_helpers` and the `ModelExplicitCompletionTestApp` /
`_model_entries()` setup where useful).

Pure helper unit tests (`find_model_effort_spacer`):

- `"%m:gpt-6-sol "` at end → returns the space column.
- `"%model:@large "`, `"%m:codex/gpt-6-sol "`, and `"fix it %m:opus "` → match.
- No match: `"%m:gpt-6-sol@high "` (already has effort), `"%m: "` (empty token),
  `"%m:opus  "` (two spaces), `"%model(opus "` (parenthesized), ``"`%m:opus "`` (inline
  code), `"%effort:high "`, `"hello "`, cursor not directly after the space, and a
  cursor followed by non-whitespace text (`"%m:opus |fix"`).

Pilot (async) tests:

- Type `%m:gpt-6-sol` then space then `@` → text is `%m:gpt-6-sol@`, cursor at end,
  `_file_completion_active` is true, `_completion_kind == "directive_arg"`, and
  candidate insertions are the seven effort levels.
- `==gpt` + `Ctrl+F` (gives `%m:gpt-5.6-sol ` in the existing fixture) then `@` →
  `%m:gpt-5.6-sol@` with the effort menu open.
- With `auto_directive_menu=False`, the space is still replaced but no menu opens.
- Typing `@` after plain prose plus a space (`hello @`) still inserts a literal `@` with
  the space kept, so the artifact-reference behavior is unchanged.
- Undo after the rewrite restores `%m:gpt-6-sol `.
- Feedback-mode bar: `@` is inserted literally (no rewrite), if the existing fixtures
  make this cheap to set up.

## Docs

- `docs/ace.md`, in the **Model shortcuts** bullet of the prompt-completion section
  (around the `==model` description that says the shortcut leaves the cursor after a
  trailing space): add a sentence saying that typing `@` directly after a colon-form
  `%m:<model> ` / `%model:<model> ` value and its single trailing space (at end of line
  or before whitespace) replaces the space with `@` and opens the effort-level menu
  (subject to `auto_directive_menu`). Note the undo escape hatch for a following `@`
  reference.
- Optionally cross-reference it from the paragraph that describes the xprompt
  trailing-space rewrites.
- No keymap or `src/sase/default_config.yml` change is needed; no new configuration
  option is added.

## Verification

- `just fmt`, then `just check` (run through `sase tool run` per the lint-and-test
  memory). Do not run `just check-full`.
- Manually sanity-check in the TUI if convenient: `==gpt`, `Ctrl+F`, `@` should show the
  effort menu, and `Ctrl+F` on `high` should give `%m:gpt-…@high`.
