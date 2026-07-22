---
tier: tale
title: Auto-continue prompt bullets with normal-mode o
goal: Make prompt NORMAL-mode o open a correctly indented hyphen bullet when the cursor
  is anywhere in an existing Prettier-wrapped bullet item.
create_time: 2026-07-22 10:36:43
status: done
---

- **PROMPT:** [202607/prompts/prompt_o_auto_bullets.md](prompts/prompt_o_auto_bullets.md)

# Auto-continue prompt bullets with normal-mode `o`

## Outcome

When `o` is pressed in NORMAL mode inside the prompt input widget, opening a line below a hyphen bullet or one of that
bullet's physical continuation lines should insert a sibling `- ` marker at the owning bullet's indentation and leave
the cursor after the marker in INSERT mode. This must work for nested bullets and for continuation lines created by the
prompt's Prettier formatting, while preserving ordinary Vim `o` behavior everywhere else.

## Current behavior and constraints

- `VimNormalEditingMixin._handle_normal_edit_key()` currently implements `o` for the shared `VimTextArea` tower by
  moving to the physical line end, inserting only `\n`, entering INSERT mode, and beginning insert-text capture after
  the structural newline. `PromptTextArea`, configuration editors, and other reusable Vim text areas inherit that path.
- The live `PromptInputBar` constructs `PromptTextArea` with Markdown highlighting, but focused normal-mode tests also
  exercise it without a syntax-aware document. Bullet ownership therefore must not depend on Tree-sitter being installed
  or enabled.
- Prettier renders a hyphen bullet's wrapped prose on subsequent physical lines indented beneath the marker; nested
  bullets and their continuation lines add another indentation level. The feature needs to recover the closest
  containing bullet marker rather than inspecting only the current line.
- Existing bullet highlighting recognizes only a leading, space-indented `- ` marker. Keep editing behavior aligned with
  that contract: do not silently broaden this change to ordered lists, `*`/`+` markers, block quotes, or tab-indented
  markers.
- The TUI keypress path must remain in-memory and side-effect free. Do not invoke Prettier, parse files, perform I/O, or
  start asynchronous work to decide what `o` inserts.
- `o` is an internal Vim command, not an `ace.keymaps` configuration entry, so no default-keymap configuration change is
  expected. Confirm this while implementing.

## Behavioral contract

1. On a line whose leading marker is `- `, `o` inserts `\n` followed by the same leading spaces and `- `.
2. On a contiguous physical continuation line, scan backward for the nearest containing `- ` marker. A candidate marker
   contains the current line only when every intervening nonblank line remains indented beneath the marker's content
   column. Insert the new marker at that candidate's marker indentation. This must distinguish a nested bullet
   continuation from a continuation that has dedented back to an outer bullet.
3. A blank-line boundary, a dedented ordinary paragraph, a fenced-code boundary, or an unsupported marker
   ends/disqualifies bullet ownership. In those cases, preserve the existing bare-newline `o` behavior.
4. Apply this only to `PromptTextArea` and only to lowercase `o`. Uppercase `O`, bare/shared `VimTextArea`, single-line
   editors, and other hosts retain their current behavior.
5. Preserve Vim mutation semantics: the automatically inserted newline/prefix is the structural part of `o`; insert
   capture starts after that prefix so user-typed text is not double-prefixed by `.`, undo remains coherent with the
   existing `o` edit grouping, and the replayed `o` re-evaluates bullet context at its destination.

## Implementation plan

1. Add a small pure prompt-bullet editing helper under `src/sase/ace/tui/widgets/`.
   - Recognize a line-start marker made of zero or more ASCII spaces followed by exactly `- `.
   - Given document lines and a cursor row, return the sibling marker prefix (`<indent>- `) or no prefix.
   - Walk backward only through the relevant contiguous in-memory block, tracking the minimum intervening indentation so
     the nearest structurally containing marker wins at the correct nesting level.
   - Treat blank/dedent and Markdown fence lines as boundaries, and keep tabs and other list-marker forms outside the
     contract.
   - Keep this logic separate from color/highlight rendering so it can be tested without mounting Textual.

2. Introduce a narrow host hook for the shared NORMAL-mode open-below command.
   - Change the lowercase `o` branch in `_vim_normal_editing.py` to ask the host for the text to insert after the
     current physical line, with a default of the existing `\n`.
   - Add the default hook to `VimTextArea` (or the existing host-hook layer) so reusable editors remain behaviorally
     unchanged.
   - Override the hook in the prompt-specific mixin/class to call the pure bullet helper and return `\n<prefix>` when
     ownership is found.
   - Compute the prefix before mutating the document, place the cursor after the inserted prefix, and continue calling
     `_record_insert_mutation_start()` only after the structural insertion so dot-repeat capture remains correct.

3. Add focused unit and widget integration coverage.
   - Parameterize the pure helper over a top-level bullet line, a Prettier-wrapped continuation, a nested bullet and
     nested continuation, a continuation dedented back to an outer bullet, and multiple sibling bullets so the nearest
     valid owner is selected.
   - Cover negative boundaries: ordinary prose, a blank paragraph boundary, a dedented line after an old list, fenced
     code, horizontal rules/tight dashes, `*`/`+`/ordered markers, and tab indentation.
   - Through `PromptPage`, assert that `o` on direct and wrapped top-level/nested bullets inserts the expected marker,
     enters INSERT mode, and places the cursor after `- `; assert a plain line still opens blank.
   - Add regression assertions that `O` is unchanged and that a bare `VimTextArea` still inserts only a newline even
     when its text resembles a bullet.
   - Exercise `o` followed by typed text, `Escape`, `u`, and `.` so the prefix participates in the existing undo
     structure and dot-repeat produces exactly one correctly indented marker while re-evaluating destination context.

4. Update the prompt NORMAL-mode documentation in `docs/ace.md`.
   - Describe lowercase `o` as auto-continuing a containing hyphen bullet at its indentation in prompt panes, including
     physical Prettier continuation lines.
   - State that non-bullet lines retain ordinary open-below behavior; leave the `O` description unchanged.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run the focused pure-helper and prompt normal-mode test modules, including the existing dot-repeat coverage affected
   by `o`.
3. Run `just check` for the repository-required full lint, type-check, test, and snapshot validation.
4. If any visual snapshot changes unexpectedly, inspect the generated visual diff artifacts rather than accepting new
   goldens; this behavior change should not require an intentional snapshot update.

## Acceptance criteria

- `o` on either the marker line or any supported Prettier-wrapped continuation of a top-level or nested hyphen bullet
  opens a sibling `- ` at the owning bullet's indentation.
- Bullet detection does not leak across blank/dedent/fence boundaries or activate for unsupported list syntax.
- Plain prompt lines, uppercase `O`, and non-prompt Vim text areas retain existing behavior.
- Cursor placement, INSERT transition, undo, and dot-repeat remain consistent and do not duplicate the automatic marker.
- Documentation matches the shipped behavior, focused tests pass, and `just check` succeeds.
