---
tier: tale
title: Fall back from snippet Tab actions to shifting the current list item
goal:
  Prompt Tab and Shift+Tab perform useful snippet work first, then shift the supported
  list item under the cursor from anywhere on its marker line when no snippet action
  succeeds.
size: medium
proposed_by: bbugyi200.athena.02s
create_time: 2026-08-15 15:27:15
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.02s](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.02s.md)
- **COMMITS:**
  - [f86373a](https://github.com/sase-org/sase/commit/f86373aeddab1d2e53b0336d6b999d3c87fb302b)
    — feat(ace): run snippet Tab actions before list shifts

# Plan: Fall back from snippet Tab actions to shifting the current list item

## Goal and current behavior

Make prompt INSERT-mode `Tab` and `Shift+Tab` indent or dedent the current supported
list item from anywhere on its marker line, but only after the key has failed to do
useful snippet work. A successful snippet expansion or tabstop jump must still win.

The current behavior is split across three places:

- `PromptTextAreaKeyHandlingMixin._on_key()` consumes both keys, gives an active snippet
  session blanket priority, and otherwise asks the ordered- and hyphen-list planners to
  shift the current line before `Tab` attempts snippet expansion or tabstop advance.
- `plan_prompt_bullet_shift()` recognizes a supported space-indented `- ` line but
  rejects a cursor past the marker's content column.
- `plan_prompt_ordered_shift()` similarly rejects a cursor past an ordered marker's
  content column before applying its existing parent-aware block move and renumbering
  rules.

That marker-region guard is why the structural behavior disappears once the cursor
enters item text. The dispatch order also does not express the desired fallback
contract: a blanket "session active" check treats a retreat at the first stop or an
advance from the final stop as useful even though no jump is available, while
marker-region shifting can precede a possible snippet expansion.

## Behavior contract

This remains prompt-local INSERT-mode behavior. NORMAL and VISUAL/V-LINE modes keep
their existing vim operators, app-level tab switching remains suppressed while the
prompt owns the keys, and both keys remain consumed when every candidate action is a
no-op.

For a collapsed selection, dispatch each key in this order:

| Key         | First useful action                                                                                           | Structural fallback                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `Tab`       | Expand the trigger word immediately before the cursor; if none expands, jump to the next live snippet tabstop | If neither action succeeds, indent/nest the supported list marker on the cursor's logical line   |
| `Shift+Tab` | Jump to the previous live snippet tabstop                                                                     | If no previous stop exists, dedent/unnest the supported list marker on the cursor's logical line |

The snippet operations' boolean result, rather than the mere existence of a snippet
session, decides whether structural fallback is allowed. Preserve the existing lifecycle
side effects: advancing from the final tabstop may end that session before the list
fallback runs, while retreating from the first tabstop leaves the session active and the
subsequent list edit remaps its offsets through the existing edit funnel. Preserve
xprompt completion-spacer handling ahead of normal dispatch: a valid jump still removes
the one-shot spacer and returns; when there is no jump, keep the spacer text and
continue through ordinary snippet/list fallback.

When a selection is active, do not shift a list item; retain today's snippet handling
and consumed no-op behavior. Do not extend ownership to a separate physical continuation
line: the eligible line itself must begin with one of the already-supported markers.
Soft-wrapped display rows are naturally part of the same logical line and therefore work
wherever the cursor appears in that line.

For hyphen bullets, retain all existing semantics except cursor eligibility:

- Recognize only the existing space-indented `- ` family, excluding tight or
  tab-indented dashes and unsupported Markdown marker families.
- `Tab` inserts the existing two-space `INDENT_UNIT` at line start.
- `Shift+Tab` removes up to one unit, and remains a no-op at zero indentation.
- Move the cursor by the inserted/removed prefix width so it stays anchored to the same
  item text even when it is in the middle or at the end of that text.
- Shift only the marker's own logical line, preserving the current single-edit, undo,
  snippet-offset, completion-state, and dot-repeat bookkeeping.

For ordered items, likewise remove only the marker-region cursor restriction. Keep the
existing structural contract intact: `Tab` needs a preceding parent, moves the item and
its owned block to that parent's content column, and renumbers the source and
destination runs; `Shift+Tab` needs an enclosing parent, moves the owned block outward,
and renumbers. Preserve the cursor's relative content position across marker-width
changes and keep the entire operation one `TextEdit`/undo checkpoint.

## Implementation

1. In `src/sase/ace/tui/widgets/_prompt_text_area_key_handling.py`, rewrite the
   INSERT-mode Tab branch around successful actions:
   - retain event consumption, soft-completion clearing, and the pending xprompt-spacer
     fast path;
   - for `Tab`, try expansion and then advance in their current relative order,
     returning only when either succeeds;
   - for `Shift+Tab`, try retreat and return only when it succeeds;
   - after those attempts, and only for a collapsed selection, ask the ordered planner
     first and the hyphen planner second, then apply any returned edit through
     `_apply_planned_text_edit(..., remap_dot_capture=True)`;
   - otherwise return as a consumed no-op. Remove the now-unneeded blanket
     `snippet_session_active` gate/type stub from this mixin if it has no other caller.

2. In `src/sase/ace/tui/widgets/_prompt_bullet_editing.py`, let
   `plan_prompt_bullet_shift()` plan from any valid offset on the direct marker line
   instead of rejecting columns beyond the content column. Update its docstring to
   describe the line-wide cursor contract. Reuse the existing line/marker recognition
   and cursor-delta calculation; do not add document scans or widget state.

3. In `src/sase/ace/tui/widgets/_prompt_ordered_shift_editing.py`, remove the analogous
   content-column rejection and update the planner documentation. Leave parent lookup,
   bounded scans, owned-block movement, delimiter style, destination numbering, run
   renumbering, and cursor remapping unchanged.

4. Update the prompt-input section of `docs/ace.md` so its key table states
   snippet-first fallback, and replace the marker-region/"use NORMAL mode inside
   content" prose with the new whole-marker-line eligibility rule for both list
   families. Keep the exclusions, one-edit semantics, hyphen single-line scope, and
   ordered owned-block behavior explicit. Update the concise prompt-input help row in
   `src/sase/ace/tui/modals/help_modal/binding_common.py` (and its assertion) to
   communicate "snippet action, else list shift" within the modal's 32-character
   description limit.

No `src/sase/default_config.yml` edit is needed: these are hard-coded prompt-widget
interpretations of the physical keys, not changes to configured
`ace.keymaps.app.next_tab`/`prev_tab`. No Rust-core change is needed because the list
planners and key dispatch are presentation-only Textual behavior; the existing
Rust-backed snippet session transitions remain the source of truth for whether a tabstop
move succeeds.

## Tests

Extend the focused pure-planner and mounted-widget coverage rather than creating a
second dispatch abstraction.

1. In `tests/ace/tui/widgets/test_prompt_bullet_editing_helpers.py`, replace the
   past-content-column rejection cases with indent and dedent plans from the middle and
   end of populated hyphen-bullet text. Assert exact edit ranges and cursor offsets,
   including a later document row and a partially indented dedent, while retaining
   existing invalid-marker and out-of-range coverage.

2. In `tests/ace/tui/widgets/test_prompt_bullet_insert_editing.py`, exercise
   `Tab`/`Shift+Tab` with the cursor inside and at the end of populated bullet text.
   Assert resulting text, cursor anchoring, INSERT mode, one-step undo, and existing
   dot-capture behavior. Keep a selected bullet non-structural.

3. In `tests/ace/tui/widgets/test_prompt_ordered_shift_editing.py`, replace the
   past-content-column no-op/planner assertions with mounted and pure-planner cases that
   nest and unnest from inside ordered-item content. Include an owned continuation/child
   block and a renumber that changes marker width so the existing block and cursor
   invariants are proven under the expanded cursor range.

4. Add dispatch-precedence regressions using the existing snippet-capable test apps and
   `SnippetSessionState` fixtures:
   - a known trigger inside either supported list line expands without shifting the
     list;
   - `Tab` with a reachable next stop jumps without shifting;
   - `Shift+Tab` with a reachable previous stop retreats without shifting;
   - `Tab` at the final stop, where advance returns false and closes the session, falls
     back to shifting the current list item;
   - `Shift+Tab` at the first stop, where retreat returns false but the session remains
     active, falls back to shifting and leaves the remapped session usable;
   - an unknown/non-trigger word with no useful tabstop falls back to shifting. Keep the
     xprompt completion-spacer jump tests passing to prove their earlier fast path is
     unchanged.

5. Update `tests/ace/tui/modals/test_help_modal.py` for the revised compact help
   wording. No PNG snapshot update should be necessary because the row length and help
   layout contract remain bounded; inspect any visual failure instead of accepting it
   automatically.

## Verification

1. Run `just install` first, as required for an ephemeral workspace.
2. Run the focused prompt suites for bullet helpers/editing, ordered shifting, snippet
   expansion, xprompt completion spacers, and the help modal.
3. Run the repository Markdown formatter for `docs/ace.md`, inspect the diff for only
   intentional prose/layout changes, and run `git diff --check`.
4. Run the required `just check`. If its scoped lane broadens, escalates, or reports
   unusual test selection, run `just check-full` only through `/sase_monitor` with a
   follow-up action, per project instructions.
5. Inspect the final diff and rerun the targeted behavioral matrix as needed, confirming
   the keypress path remains synchronous, memory-only, and bounded; no config,
   Rust-core, memory, generated instruction, or visual-snapshot files should change.
