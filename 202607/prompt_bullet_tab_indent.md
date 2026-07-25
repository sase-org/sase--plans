---
tier: tale
title: Indent and dedent prompt bullets with Tab and Shift+Tab
goal: 'In the prompt input widget, INSERT-mode Tab indents the hyphen bullet under
  the cursor by one two-space unit and Shift+Tab removes one unit, without disturbing
  snippet expansion, tabstop navigation, or any other prompt key.

  '
create_time: 2026-07-25 08:25:09
status: done
---

- **PROMPT:** [202607/prompts/prompt_bullet_tab_indent.md](prompts/prompt_bullet_tab_indent.md)

# Plan: Indent and dedent prompt bullets with Tab and Shift+Tab

## Context and outcome

While typing a hyphen bullet list in the ACE prompt input, there is no way to nest the bullet being written without
leaving INSERT mode. Prompt INSERT-mode `Tab` currently only expands a snippet trigger word and then advances a pending
snippet tabstop; on a bullet marker it does nothing, because there is no trigger word before the cursor. `Shift+Tab` is
entirely unbound in the prompt: the app-level `prev_tab` action is deliberately suppressed while a vim text area is
focused, so the key falls through to the Textual screen's focus-cycling binding and only appears inert because the
prompt text area refocuses itself after every blur.

Give both keys the obvious list-editing meaning while the cursor sits at the beginning of a bullet line: `Tab` indents
that bullet by one two-space unit and `Shift+Tab` removes one unit. The motivating case is a freshly opened marker — a
line holding exactly `- ` with the cursor immediately after it — where `Tab` should turn the line into ` -` and
`Shift+Tab` should turn it back.

Everything else about the prompt stays as it is. Snippet expansion, tabstop advance, manual and soft completion, xprompt
argument hints, auto-pairing, the prompt stack, and every vim mode keep their current behavior.

## Behavior contract

**Eligible line.** Only a hyphen bullet participates, using the prompt's existing marker rule: zero or more leading
spaces, a `-`, then a space. Reuse that rule from the module that already owns prompt bullet ownership so the new keys
agree exactly with bullet-dash highlighting and with `Ctrl+J` / `o` / `O` bullet continuation. A tab-indented dash, `*`
/ `+` / ordered / blockquote markers, a thematic break, and a tight `-x` dash are not bullets and are never indented.
Physical continuation lines of a wrapped bullet carry no marker of their own and are also excluded.

**Eligible cursor.** The selection must be collapsed and the cursor column must be at or before the marker's content
column (leading spaces plus two). That is the whole "beginning of the line" region: column zero, anywhere inside the
leading indentation, on the dash, and the common resting place immediately after `- `. This region can never hold a
snippet trigger word — every character before the cursor there is a space or the dash — so admitting it costs no
existing `Tab` behavior.

| Situation (INSERT mode, collapsed selection)        | `Tab`                                       | `Shift+Tab`                                   |
| --------------------------------------------------- | ------------------------------------------- | --------------------------------------------- |
| Cursor in the marker region of a `- ` bullet        | Insert two spaces at column 0               | Remove up to two leading spaces from column 0 |
| Same, but the line has zero (or one) leading spaces | Insert two spaces                           | Remove the one space, or no-op at zero        |
| Cursor past the marker's content column             | Today's snippet expansion / tabstop advance | Consumed no-op                                |
| Non-bullet line, or a selection is active           | Today's snippet expansion / tabstop advance | Consumed no-op                                |
| A snippet expansion left tabstops queued            | Advance to the next tabstop                 | Consumed no-op                                |

Fix the removal amount and the insertion amount to the same two-space unit the prompt's vim `>>` / `<<` operators
already use, including their "remove up to one unit" dedent rule, rather than introducing a second notion of shift
width. Take that unit from the existing vim transform helpers.

**Cursor and scope.** The cursor keeps its position relative to the line's content: indenting moves it two columns
right, dedenting moves it left by however many spaces were actually removed, floored at column zero. Only the cursor's
own line changes — continuation lines and nested descendants stay where they are, matching `>>`.

**Key consumption.** Consume `Tab` and `Shift+Tab` in prompt INSERT mode even when neither indent nor snippet work
applies. `Tab` already behaves this way. Consuming `Shift+Tab` is a deliberate, small behavior change: the key stops
reaching the screen's focus-cycling binding, so the prompt no longer takes a blur/refocus round trip that the widget
silently undoes. NORMAL and VISUAL modes are untouched — vim `>>`, `<<`, and visual `>` / `<` already own indentation
there, they remain the way to indent a bullet whose content the cursor is already inside, and the suppressed app-level
tab-switch bindings stay suppressed while the prompt is focused.

**Edit bookkeeping.** A single keypress produces a single text edit, so one NORMAL-mode `u` restores the line exactly.
The edit inserts or removes text at the start of the line, which can sit before the offset where the current INSERT
session began capturing text for dot-repeat; remap that capture offset by the edit's delta, the way whole-buffer prompt
formatting already remaps it, so a later `.` replays the text the user typed instead of a slice shifted by the indent.

## Implementation shape

Add one pure planner to the module that already holds prompt hyphen-bullet helpers, following the convention the
auto-pair and Jinja pair helpers established: take the current document text plus an absolute cursor offset, decide
eligibility from the line containing that offset, and return either the planned replacement (range, replacement text,
resulting cursor offset) or nothing when the keypress does not apply. Keep it free of widget state so it is unit
testable, and reuse the module's existing marker and boundary patterns instead of writing new ones. The planner must be
public, since the widget mixin in another module imports it and a test-only consumer would not satisfy the unused-symbol
linter.

Dispatch from the existing INSERT-mode tab branch of the prompt key-handling mixin, extending it to also match the
shifted key, and apply the plan through the mixin's existing planned-edit helper so soft completion, manual completion,
xprompt argument hints, and the completion-context notification are torn down exactly as they are for the other
structural insert-mode edits. Keep the bullet decision ahead of snippet lookup: snippet lookup asserts that the host app
is the ACE app, so a bullet indent that short-circuits before it also stays testable in the isolated prompt harness.
Preserve the current relative order of snippet expansion and tabstop advance for every non-bullet case.

All of the work is in-memory and bounded by the current line; no rescans of the document, no refresh passes, and no
awaited work on the keypress path.

## Tests and documentation

Cover the pure planner with parametrized cases: top-level and nested markers; the cursor at column zero, inside the
leading indentation, on the dash, and at the content column; a marker-only `- ` line; one-space and zero-space dedents;
the cursor past the content column; plain prose; a tab-indented dash; `*`, `+`, ordered, and blockquote markers; a tight
`-` with no following space; and a marker on a later row of a multi-line document so offset mapping is exercised in both
directions.

Cover the key dispatch with the isolated prompt harness in INSERT mode: the motivating `- ` case indenting to ` -` with
the cursor after the marker and `Shift+Tab` returning it; repeated `Tab` presses accumulating indentation; a dedent from
a deeply indented nested bullet; `Shift+Tab` on an unindented bullet and on prose leaving text, cursor, and mode
untouched; an active selection falling through instead of indenting; and a NORMAL-mode `u` after one indent restoring
the original line. Add a queued-tabstop case proving tabstop advance still wins, and a case using the app-shaped host
the existing snippet tests rely on to prove a trigger word later on a bullet line still expands.

Document the pair in the `docs/ace.md` prompt INSERT-mode key table — amend the `Tab` row and add a `Shift+Tab` row —
and extend the bullet-editing prose that follows it with the eligibility rule, the two-space unit, the single-line
scope, and the pointer to `>>` / `<<` for indenting from inside a bullet's content. Add one row to the help modal's
prompt-input section, within its description width limit. No keymap configuration change is needed: prompt-widget keys
are hard-coded rather than part of `ace.keymaps`, and the app's `next_tab` / `prev_tab` defaults keep `tab` /
`shift+tab`.

Because workspaces are ephemeral, run `just install` first, then the targeted prompt bullet, snippet, and help tests,
then the repository-required `just check`.
