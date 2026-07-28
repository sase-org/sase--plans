---
tier: tale
title: Remove an auto-inserted xprompt spacer before a typed comma
goal:
  Completing a no-input or optional-only xprompt and immediately typing a comma produces a punctuation-tight reference
  such as `#foo,` without regressing optional-argument colon completion or ordinary user-authored spacing.
create_time: 2026-07-28 17:36:50
status: done
---

- **PROMPT:** [202607/prompts/xprompt_completion_comma_spacer.md](prompts/xprompt_completion_comma_spacer.md)
- **AGENTS:**
  - [bbugyi200.athena.nh--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nh.md#member-code)
  - [bbugyi200.athena.nh--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nh.md#member-plan)
- **COMMITS:**
  - [5568411](https://github.com/sase-org/sase/commit/5568411c9545b2c34bd112fb30f0d39afd4aacc2) — fix(tui): tighten
    commas after xprompt completions

# Plan

## Context

ACE's prompt input completion skeleton appends a trailing space when an xprompt has no required inputs. This covers both
a true no-input xprompt and an optional-only xprompt (including the requested single-optional-input case). If
punctuation already follows the completion target, `_xprompt_arg_assist_skeletons.py` suppresses that space, but
punctuation typed after acceptance currently lands after it and produces text such as `#foo ,`.

There is already a safe one-shot rewrite for a closely related case: after an optional-only xprompt completes to
`#foo `, typing `:` replaces the recorded spacer with `:`. The pending state records the completion's exact reference
and spacer offsets, verifies that the cursor and text are unchanged before editing, and is populated by all four xprompt
acceptance paths:

- direct single-candidate and completion-panel acceptance;
- soft-completion acceptance;
- prompt selector / smart-snippet insertion.

That state is intentionally restricted to optional-only entries because `:` introduces an optional argument; a no-input
xprompt must continue to preserve its space before a typed colon. The comma behavior differs: comma is punctuation and
should remove the completion-added space for either no-input or optional-only entries.

## Implementation

1. Generalize the pending optional-spacer model and helpers into a pending xprompt-completion spacer.
   - Rename the model, prompt-text-area field, mixin type declarations, exports, reset sites, note helper, and consume
     helper so their names describe both no-input and optional-only completions.
   - Record pending state only for entries with no required inputs and only when the accepted skeleton actually left the
     cursor immediately after an exact `entry.insertion + " "` sequence. Keep the existing offset, cursor,
     spacer-character, and reference-identity checks so this feature cannot strip an unrelated or user-authored space.
   - Preserve whether the completed entry has optional inputs in the pending state (or equivalent logic), allowing `:`
     only for optional-only entries while allowing `,` for both supported spacer-producing shapes.
   - Keep required-input completions, including the required-text `#name:: ` skeleton, outside this rewrite.

2. Extend the prompt text area's one-shot key handling.
   - When the first key after a recorded completion spacer is `,`, replace the spacer with the comma, consume the
     original key event, leave the cursor after the comma, and run the normal post-change completion/hint refresh
     against the final text.
   - Retain the existing optional-only `:` rewrite and automatic argument-menu behavior.
   - Clear the pending state on every other key or whenever validation fails, allowing Textual's normal insertion path
     to run unchanged. Cursor movement, intervening input, altered reference text, or an absent spacer must therefore
     preserve the space.

3. Update the focused xprompt spacer regression suite and helper-level coverage.
   - Cover immediate comma replacement for true no-input and optional-only xprompts, producing `#name,`.
   - Exercise the acceptance plumbing across direct `Ctrl+T`, completion-panel, soft-completion, and
     selector/smart-snippet paths so every source of an auto-inserted spacer records equivalent pending state.
   - Retain explicit assertions that optional-only `:` still becomes `#name:`, a no-input colon still remains `#name :`,
     and colon-triggered argument completion still opens only where applicable.
   - Add negative coverage showing that an intervening key or moved/invalidated cursor/reference makes a later comma
     insert normally, and that completions accepted before existing punctuation do not create pending spacer state.
   - Update test/module terminology and any input-shape predicate tests to reflect the generalized state while
     preserving the distinction between no-required-input and optional-only entries.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused prompt-widget tests:
   - `.venv/bin/pytest -q tests/ace/tui/widgets/test_xprompt_optional_spacer.py`
   - `.venv/bin/pytest -q tests/ace/tui/widgets/test_xprompt_arg_assist.py tests/ace/tui/widgets/test_xprompt_completion.py tests/ace/tui/widgets/test_prompt_live_completion.py`
3. Run the mandatory full repository check with `just check`.

## Non-goals

- Do not change xprompt parsing or shorthand semantics; this is an input-widget editing convenience around a spacer
  inserted by completion.
- Do not strip arbitrary spaces before commas or change punctuation typed without a freshly recorded xprompt completion.
- Do not alter keymaps or completion configuration.
