---
tier: tale
title: Inherit VCS xprompt when adding prompt panes
goal:
  New prompt panes created with g- or Ctrl+G - begin with the active pane's explicit VCS
  xprompt workflow when one is present.
size: small
proposed_by: bbugyi200.athena.01j
create_time: 2026-08-14 13:41:52
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.01j](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.01j.md)
- **COMMITS:**
  - [402b3b6](https://github.com/sase-org/sase/commit/402b3b65ad162ff9ab978342196b4ed3c4807b33)
    — feat(ace): inherit VCS xprompt tag for added panes

# Plan: Inherit VCS xprompt when adding prompt panes

## Context and decisions

Both prompt-local keymaps already converge on
`PromptInputBarStackNavigationMixin.add_bottom_pane()`. That action synchronizes the
live prompt widgets into `PromptStackState`, then unconditionally appends an empty item.
In contrast, the Ctrl+Space launch entry point opens a prompt prefilled with a canonical
`#<workflow>:<ref> ` prefix. The add-pane action should provide the same useful starting
point without changing either key binding or the launch context.

Treat the currently active launchable prompt pane as the source of truth. After its live
text has been synchronized, use the existing registered-workflow lexical parser to
extract an explicit leading VCS xprompt tag, including the supported placement after
leading `%` directives and a tag at end-of-input. Seed the new pane with the trimmed tag
followed by one space, matching the Ctrl+Space prefill shape. Do not copy the rest of
the prompt or its directives, search for arbitrary embedded tags, infer a workflow from
project state, or materialize the implicit `#git:home` default. If the active prompt has
no explicit leading VCS tag, retain the current empty-pane result. If the pinned snippet
target pane has focus, treat it as a non-launchable editor rather than a workflow
source: insert the new agent pane above it with empty initial text.

Keep this behavior in the TUI stack-navigation action rather than changing
`PromptStackState.append_bottom()`: other callers use that state primitive to restore
specific pane contents and must remain verbatim. The extraction must remain a cheap,
read-only operation over the already synchronized text; it must not resolve VCS refs,
read project files, or invoke provider operations on the keypress path. Existing
append-at-bottom, snippet-pane ordering, focus, insert-mode entry, prompt-mode guard,
and transient-completion cleanup behavior remain unchanged.

## Implementation

1. Update `src/sase/ace/tui/widgets/_prompt_input_bar_stack_navigation.py` so
   `add_bottom_pane()` derives the new pane's initial text from the selected agent pane
   after `_sync_state_from_widgets()`. Reuse the canonical xprompt VCS-tag extractor
   with an end-of-input sentinel, normalize a match to exactly `<tag><space>`, and pass
   that value to `append_bottom()`; otherwise pass the empty string. Explicitly bypass
   extraction when the selected item is the snippet target. Keep parsing local to this
   action so stack-state mutation remains generic and stash/snippet restore paths are
   unaffected.

2. Extend `tests/ace/tui/widgets/test_prompt_stack_keymaps_add_pane.py` to exercise the
   shared behavior through both public key paths: NORMAL-mode `g-` and INSERT-mode
   `Ctrl+G -` must append and focus an insert-mode pane prefilled with the active pane's
   explicit VCS tag. Add coverage proving that a pane without an explicit tag still
   creates an empty pane, that a leading tag following prompt directives is inherited
   without copying those directives or body text, and that a multi-pane stack inherits
   from the selected/current pane rather than an earlier pane. Cover the snippet-focused
   edge so its body is not inherited and the new agent pane stays above the pinned
   snippet. Retain the existing no-op and retired-key regression assertions.

3. Run `just install`, then the focused add-pane widget test module for quick feedback.
   Finish with `just check`, as required for SASE repository changes. The change does
   not touch the broadening set and does not require `just check-full` or PNG snapshot
   updates unless scoped verification reports an escalation.

## Acceptance criteria

- `g-` in NORMAL mode and `Ctrl+G -` in INSERT mode append a bottom prompt pane whose
  initial text is the active pane's explicit registered VCS xprompt tag followed by a
  single space.
- Only the VCS tag is inherited; prompt directives and task text are not duplicated.
- A bare prompt, including one that would later normalize implicitly to `#git:home`,
  still produces an empty new pane.
- In a stack whose panes use different VCS refs, the selected pane determines the
  inherited tag.
- A focused snippet target is not used as a VCS source, and it remains pinned below the
  newly added agent pane.
- The new pane remains selected, focused, and in INSERT mode, and non-prompt modes
  remain no-ops.
- Focused tests and `just check` pass.
