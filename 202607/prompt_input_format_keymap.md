---
tier: tale
title: Explicit prompt formatting keymap
goal: 'The active ACE prompt input pane can be formatted on demand with gf in NORMAL
  mode or Ctrl+G f in INSERT mode, using the same canonical Markdown formatting policy
  as launch-time agent prompts without blocking or losing concurrent edits.

  '
create_time: 2026-07-16 08:09:48
status: done
prompt: 202607/prompts/prompt_input_format_keymap.md
---

# Plan: Explicit Prompt Formatting Keymap

## Context and current findings

ACE prompt inputs intentionally use Textual soft wrapping and preserve the raw buffer while the user types. Automatic
Prettier reflow was removed because it caused subprocess latency, cursor movement, and unsolicited whitespace changes.
An explicit formatting command fits that design: the buffer changes only when the user asks for it.

The canonical launch-time formatting stage is currently in
`src/sase/llm_provider/preprocessing.py::preprocess_prompt_late()`. It calls
`sase.file_references.format_with_prettier()` with `AGENT_PROMPT_WRAP_WIDTH` (80), which invokes Prettier as Markdown
with `--prose-wrap=always` and safely returns the original text when Prettier is disabled, missing, times out, or fails.
Other Markdown artifacts keep the formatter's 120-column default. The input action must reuse the 80-column agent prompt
policy; it must not call the full preprocessing pipeline, because that would also expand xprompts, execute command
substitutions, process file references, render Jinja, strip comments, and otherwise change prompt semantics rather than
merely formatting the editable source.

Prompt-local `g` continuations are hardcoded and centrally declared in
`src/sase/ace/tui/widgets/_prompt_input_bar_g_prefix_actions.py`. The same table drives dispatch and the which-key-style
hint panel. Bare `g` reaches that table from NORMAL mode before unclaimed continuations fall through to native Vim
commands; INSERT mode reaches it through the existing `Ctrl+G` prefix. These prompt-local keys are not part of the
configurable app keymap registry, so `src/sase/default_config.yml` and the command palette do not need entries.

## Product behavior

- `gf` in NORMAL mode formats the text in the pane that is active when the command is invoked.
- `Ctrl+G f` in INSERT mode performs the same action and leaves the pane in INSERT mode. Because NORMAL mode already
  exposes the same prompt-local `Ctrl+G` surface, `Ctrl+G f` may remain a harmless symmetric NORMAL-mode alias; `gf` is
  the primary NORMAL binding.
- In a multi-pane prompt stack, only the selected pane is formatted. Other panes, their ordering, shared frontmatter,
  and stack selection are unchanged.
- The action also works for the single `PromptTextArea` used by feedback and approve-prompt bars; those are prompt input
  widgets even though they are not multi-agent stacks.
- Empty or already-formatted text is a no-op. The existing formatter fallback remains non-destructive when Prettier is
  unavailable, disabled, or fails.
- Formatting is an explicit, single undoable edit. It preserves the current Vim mode and maps the cursor/selection to
  the equivalent logical position after reflow. It does not become a Vim dot-repeat mutation.
- If the user edits the captured pane, rebuilds/unmounts it, or starts a newer format request while Prettier is running,
  the stale result is discarded rather than overwriting newer state. Moving focus alone does not retarget the request:
  it still belongs to the pane selected at invocation.
- Visual mode is unchanged; neither binding is added there.

## Design

### 1. Make the agent-prompt formatting policy a named shared entry point

Add a small formatting-only helper alongside `format_with_prettier()` in `src/sase/file_references.py`, named to
distinguish it from full prompt preprocessing (for example, `format_agent_prompt_markdown(text)`). It should be the
single place that selects `AGENT_PROMPT_WRAP_WIDTH` and delegates to the existing Prettier helper, preserving its
Markdown parser, prose-wrap behavior, underscore normalization, environment opt-out, timeout, and failure fallback.

Change `preprocess_prompt_late()` to call this named helper at its existing formatting stage, without changing the
surrounding protected-region, command, file-reference, Jinja, or comment-processing order. The TUI action will call the
same helper. This removes duplicated knowledge of the prompt width and ensures future changes to prompt formatting
affect launch-time and on-demand formatting together, while saved artifacts continue to use the 120-column default.

### 2. Add a widget-scoped asynchronous formatting action

Keep the editing lifecycle on `PromptTextArea`, preferably in a focused mixin such as `widgets/_prompt_format.py`,
following the existing prompt preview/jump worker pattern:

- Snapshot the target widget, source text, and a monotonically increasing request id when the action starts. Clear
  completion, search, and prefix UI that would otherwise be anchored to the old buffer.
- Run the shared synchronous formatter and any diff/cursor-mapping work in a worker thread (`run_worker` plus
  `asyncio.to_thread`, or the equivalent established widget-worker pattern). Never invoke `subprocess.run()` on
  Textual's event loop; the shared helper permits a ten-second timeout.
- After the worker yields, re-check the request id, mount identity, and exact source text. Re-read the target's current
  cursor/selection and Vim/read-only state at application time so cursor movement or a mode change during the await is
  not rolled back. Discard stale results and retain newer edits.
- Apply a changed result as one whole-buffer `_replace_via_keyboard()` edit, temporarily relaxing `read_only` when the
  current mode is NORMAL, then restore the mode/read-only contract. Map cursor and selection endpoints through the
  old/new diff and clamp them to the formatted document.
- Let the existing `TextArea.Changed` path synchronize the pane model, dirty marker, line numbers, completion context,
  and prompt-bar height; explicitly refresh only where the event does not already cover it.
- Handle unexpected worker errors with a concise toast and leave the buffer untouched. An unchanged result can report
  that formatting made no changes without claiming whether the cause was already-formatted content or the canonical
  formatter's safe fallback.

This keeps formatting local and reversible, without reviving the removed automatic formatter scheduler or tying the wrap
width to terminal size.

### 3. Route both requested key sequences through the canonical prefix table

Extend `_PROMPT_G_PREFIX_BINDINGS` with an `f` entry that delegates to a `PromptInputBar` action such as
`format_active_prompt()`. That bar action captures `active_text_area()` once and asks that exact widget to format
itself; it does not serialize or format the whole stack.

Add the corresponding availability and label methods so the shared table also renders `f -> format prompt` in the
transient `g` / `Ctrl+G` hint panel. The entry should be available for a non-empty active `PromptTextArea` in prompt,
feedback, and approve-prompt modes. Dispatch remains a swallowed no-op when the pane is empty or otherwise unavailable,
matching the existing prompt-prefix contract. Update nearby enumerating comments/docstrings so `gf` is listed among the
prompt-owned `g` continuations while native `gg`, `ge`/`gE`, and `gu`/`gU`/`g~` continue to fall through unchanged.

No `default_config.yml`, app `KeymapRegistry`, footer, or command-catalog change is needed: this is a prompt-local modal
editing command, not a configurable global app action. It is unconditional within its input context, so it belongs in
the help and prefix hint surfaces rather than the conditional footer.

### 4. Keep help and visual discovery in sync

Add `gf / Ctrl+G f` with a concise “Format current prompt” description to `PROMPT_INPUT_SECTION` in
`src/sase/ace/tui/modals/help_modal/binding_common.py`. Keep the description within the help modal's 32-character limit
and preserve the fixed box-width rules.

The new hint row intentionally changes the prompt `g`-prefix panel, and the help row intentionally changes the keymaps
help screen. Inspect and update the affected PNG goldens only for those expected additions:

- `prompt_stack_g_prefix_hints_120x40.png`
- `help_keymaps_changespecs_120x40.png`

## Test plan

### Shared formatter contract

Extend `tests/test_format_with_prettier.py` to prove that the named agent-prompt helper selects 80 columns and that
`preprocess_prompt_late()` routes through it. Preserve coverage for the 120-column artifact default and for missing,
disabled, failed, and timed-out Prettier fallbacks. Update existing formatter mocks only where the new named call
boundary requires it.

### Prompt widget behavior

Add focused PromptInputBar tests with a deterministic mocked formatter:

- NORMAL `gf` formats the active pane, remains in NORMAL mode, and does not interfere with native
  `gg`/`ge`/case-operator continuations.
- INSERT `Ctrl+G f` produces the same formatted text and remains in INSERT mode.
- A multi-pane stack changes only the pane selected at invocation; focus changes during the worker do not retarget the
  result, and frontmatter/other panes stay byte-for-byte unchanged.
- Feedback and approve-prompt bars can format their single pane.
- The edit is one undo step, cursor/selection mapping is stable across whitespace normalization and line reflow, and an
  unchanged result creates no edit.
- A controlled slow formatter proves the key handler/event loop remains responsive. Edits made while it runs, a newer
  request, or a pane rebuild/ unmount cause the old result to be discarded.
- Missing/disabled/failing formatter behavior leaves the buffer intact, and formatting remains explicit: ordinary typing
  still relies only on virtual wrapping and never schedules Prettier.

### Dispatch, help, and snapshots

Update `tests/ace/tui/widgets/test_prompt_g_prefix_hints.py` for the new `f` entry on bare-`g` and `Ctrl+G` surfaces,
its label/availability, feedback-mode visibility, dispatch routing, hint closure, and the continued fallthrough of
Vim-owned continuations. Update `tests/test_keymaps_display_help.py` to assert the new help row across the main ACE
tabs. Regenerate the two intentional PNG goldens above and re-run the dedicated visual suite.

## Verification

After implementation:

1. Run `just install` first in the ephemeral workspace.
2. Run the formatter, prompt-prefix, new prompt-formatting, help, virtual-wrap, and relevant Vim `g`-continuation tests.
3. Run `just test-visual`, using `--sase-update-visual-snapshots` only to accept the two inspected hint/help additions,
   then re-run without the update flag.
4. Run the required full `just check`.
5. Manually smoke a long Markdown prompt in a single pane and a multi-pane stack: verify NORMAL `gf`, INSERT `Ctrl+G f`,
   undo, mode/cursor stability, active-pane targeting, and responsive typing/navigation while formatting runs.

## Boundary and out of scope

This is presentation-layer prompt editing plus reuse of an existing Python formatter; no shared backend domain behavior
or Rust wire/API change is needed. The work does not restore automatic formatting, format every pane/frontmatter at
once, run xprompt/file/Jinja preprocessing from the editor action, make prompt-local Vim keys user-configurable, or
change the 120-column policy for saved Markdown artifacts.
