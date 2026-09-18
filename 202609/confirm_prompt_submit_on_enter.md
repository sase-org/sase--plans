---
tier: tale
title: Configurable Enter confirmation for prompt submission
goal:
  Plain Enter confirms prompt launches by default while configurable opt-out and direct
  prefix keymaps remain reliable.
size: medium
proposed_by: bbugyi200.athena.0nc
create_time: 2026-09-18 18:41:02
status: wip
---

# Configurable Enter Confirmation for Prompt Submission

## Goal

Make plain `<enter>` a deliberate, consistent launch gesture in sase's TUI prompt input.
By default, pressing `<enter>` on any non-empty, launchable prompt-mode draft opens the
existing keyboard-first submission panel, including when the bar has only one ordinary
prompt pane. Pressing `<enter>` again activates the panel's primary action and launches
the prompt. Users who prefer the old immediate behavior can disable the panel with a new
configuration field:

```yaml
ace:
  prompt_submission:
    confirm_on_enter: false
```

The default is `true`. The explicit `g<enter>` (NORMAL mode) and `<ctrl+g><enter>`
(INSERT mode) routes remain direct submission accelerators and never open this general
confirmation panel.

This is presentation-only Textual behavior. Submission parsing, xprompt expansion,
remote-dispatch planning, and agent launching retain their existing backend paths, so no
`sase-core` change is needed.

## Product and interaction design

### One predictable rule for plain Enter

Apply the confirmation only when all of the following are true:

- the bar is in `prompt` mode;
- the active pane is an agent prompt rather than a snippet or mini-xprompt auxiliary
  editor;
- at least one agent prompt pane contains non-whitespace text; and
- `ace.prompt_submission.confirm_on_enter` is enabled.

This deliberately includes three shapes that currently diverge: an ordinary single-pane
prompt, a multi-pane prompt stack, and a targeted xprompt draft. It keeps specialized
semantics unchanged:

- an empty ordinary prompt still follows the existing empty-submit/cancel path without
  asking the user to confirm launching nothing;
- snippet and mini-xprompt panes keep using Enter to save their auxiliary content;
- `feedback` and `approve_prompt` bars keep submitting their mode-specific response
  directly, because they do not launch the prompt bar's agent draft;
- the existing dirty-auxiliary guard, visible-TODO warning, stale-target checks, prompt
  input collection, and remote-dispatch preflight remain authoritative after the user
  chooses to submit;
- accepting or dismissing completion UI keeps its present contract: Enter submits the
  authored text rather than accepting a completion candidate.

When confirmation is disabled, plain Enter calls the existing active-pane submit path.
For one pane this launches and closes the bar; for a stack it launches/removes the
selected pane, matching the direct-submit semantics and the behavior that predated the
stack chooser. A targeted draft launches rather than opening its launch/save chooser;
the existing `gw` / `<ctrl+g>w` and `gX` / `<ctrl+g>X` save gestures remain available.

### Extend the existing submission panel

Evolve `PromptSubmitChoiceModal` rather than adding a second confirmation-dialog style.
It already owns the beautiful double-border, theme-aware, keyboard-first visual language
for stacked and targeted submissions; making it handle the ordinary single-pane case
gives the user one recognizable launch surface.

Render context-aware content:

- **Ordinary single pane:** title `Launch prompt?`; one primary `Launch agent` row,
  explaining that one agent will start with this prompt.
- **Targeted single pane:** retain the launch, write-to-target, and save-as rows, with
  the launch row primary.
- **Multi-pane stack:** retain `Submit all` and `Submit current` (plus target-save rows
  when applicable), with `Submit all` primary because building a stack usually signals
  whole-stack intent.

Bind `<enter>` to a new `choose_primary` action: it returns `send` for a single pane and
`all` for a multi-pane stack. Keep all mnemonic keys (`s`, `a`, `c`, `w`, `X`) and
cancel keys (`escape`, `q`) as aliases. Update the primary row and compact footer to
teach Enter explicitly (for example `enter/s launch` or `enter/a all`) without making
the panel visually noisy. The first Enter therefore asks; the second Enter confirms.

The modal must confirm the draft the user actually saw. Capture a lightweight immutable
origin token before pushing it (bar/stack identity, rebuild generation, selected item,
ordered pane ids and text, frontmatter, and binding). On a non-cancel result, validate
that token before invoking the existing submission action. If the bar was unmounted or
programmatically rebuilt/changed while the modal was open, fail closed, keep the new
draft intact, refocus it when possible, and tell the user to review and submit again.
Normal modal input cannot edit the covered bar, but this protects background and
programmatic state transitions and makes the confirmation semantically reliable.

### Keep the fast path truly distinct

Do not route `g<enter>` or `<ctrl+g><enter>` through the plain-Enter decision. They
already dispatch through `PromptInputBar.submit_active_pane()` directly to
`action_submit_prompt()`, so preserve that structure and add focused regression tests
for both sequences on a single pane as well as a stack. “Direct” bypasses only the new
general confirmation panel; existing contextual safety checks such as visible TODOs,
dirty auxiliary content, and dispatch preflight still apply.

### Make the state discoverable

When confirmation is enabled, use `submit…`/`launch…` wording in the INSERT-mode border
subtitle for an ordinary single pane as well as existing stacked/targeted bars, so the
ellipsis honestly signals that Enter opens another surface. When it is disabled, retain
immediate `send` wording. NORMAL mode continues to advertise `g<enter>` as the direct
route, and the prompt-local `<ctrl+g>` hint panel continues to expose its Enter
continuation.

Update help and documentation to state:

- plain Enter opens the submission panel by default and Enter confirms its primary
  action;
- `g<enter>` / `<ctrl+g><enter>` submit the selected prompt directly;
- the exact opt-out is `ace.prompt_submission.confirm_on_enter: false`;
- disabling confirmation makes stacked Enter submit the active pane immediately.

## Implementation

1. Add a typed, defensive `PromptSubmissionSettings` parser (following
   `current_project_settings.py`) for the new `ace.prompt_submission` block. Its sole
   field is `confirm_on_enter: bool = True`; missing, malformed, or non-boolean values
   fall back to `true` rather than relying on truthiness.
2. Load those settings once during ACE startup in `_state_init_late.py`, retain them on
   `StartupMixin`, and expose a memory-only getter/fallback usable by prompt widgets.
   The Enter keystroke path must not read config files, parse YAML/JSON, or do any other
   synchronous I/O.
3. Add `ace.prompt_submission.confirm_on_enter: true` to `src/sase/default_config.yml`
   and the corresponding closed-object boolean schema to
   `src/sase/config/sase.schema.json`. This explicitly satisfies the project's default
   config rule for new TUI configuration.
4. Generalize `PromptSubmitChoiceModal` for an untargeted single pane, add dynamic
   title/row/footer copy, and make Enter choose the context's primary launch action.
   Keep the existing theme-aware shared styles and tune only the modal-specific width or
   spacing if the new single-pane composition demonstrates a visual imbalance.
5. Refactor the plain-Enter branch in `_prompt_text_area_key_handling.py` into a small,
   readable decision that consults only cached settings and bar state. Extend
   `_open_submit_choice_panel()` in `_prompt_text_area_actions.py` to cover ordinary
   single panes, capture/revalidate the origin token, and then route results through the
   existing `action_submit_prompt()` / `action_submit_prompt_stack()` methods. Leave the
   prefix dispatch table and `submit_active_pane()` direct route intact.
6. Make `_prompt_input_bar_subtitles.py` reflect configured immediate versus confirmed
   Enter behavior while preserving auxiliary-pane, mode-specific, targeted, and stack
   hints.
7. Update the Prompt Input help rows plus `docs/ace.md`, `docs/xprompt.md`, and the ACE
   configuration overview/details in `docs/configuration.md`. Remove statements that an
   untargeted single pane always sends immediately, describe Enter as the primary
   confirmation key, and document the opt-out and its stack semantics.

## Tests and acceptance criteria

Add or update focused widget tests so all of these contracts are explicit:

- With default settings, the first Enter on a non-empty ordinary single pane opens the
  submission panel and posts no `Submitted` event; the second Enter posts exactly one
  unchanged single-pane submission.
- The single-pane panel renders the intended title, primary copy, Enter hint, and cancel
  hint. Escape/q cancels, preserves text/cursor/selection, and restores focus.
- In a multi-pane panel, Enter selects the primary whole-stack action; `a`/`Ctrl+S` and
  `c` retain their current all/current results. Targeted single- and multi-pane rows
  retain launch/write/save-as behavior.
- A bar state change or unmount between opening and confirming fails closed and never
  submits replacement content.
- `g<enter>` and `<ctrl+g><enter>` directly submit one ordinary single pane and the
  selected pane of a stack without opening `PromptSubmitChoiceModal`, regardless of the
  confirmation setting.
- With `confirm_on_enter: false`, Enter immediately submits an ordinary single pane, the
  active pane of a stack, and a targeted pane without opening the panel.
- Empty prompts, snippet panes, mini-xprompt panes, feedback bars, and approve-prompt
  bars retain their current Enter behavior.
- Completion teardown, TODO confirmation (including the expected additional TODO safety
  step after the general panel), dirty auxiliary guards, frontmatter reattachment,
  whole-stack joins, and dispatch preflight continue to work.
- Subtitle assertions cover both config states and remain accurate in INSERT/NORMAL,
  stacked, targeted, and auxiliary contexts.
- Parser tests cover defaults, valid false, malformed blocks, and non-boolean values.
  Schema/default tests prove the shipped default and public schema agree and reject
  unknown or invalid prompt-submission fields.

Update the existing prompt submit-choice PNG snapshot to show Enter as the primary stack
action, and add an ordinary single-pane confirmation snapshot in both the default theme
and the existing light-theme visual harness if that harness supports this modal. The
snapshots should demonstrate balanced spacing, clear hierarchy, and readable
primary/cancel affordances at the standard 120x40 viewport; include the existing narrow
viewport coverage if modal copy or width changes can wrap there.

Run focused tests first, including the prompt stack submit/cancel, TODO, subtitle,
g-prefix routing, settings, and config-schema suites. Then run the repository's required
`just check` fast gate and the relevant visual snapshot suite/update workflow. Re-read
the canonical lint/test guidance before finishing implementation, as required after
tracked SASE files change.

## Non-goals and compatibility

- Do not change the configured keymaps: this feature changes what the existing Enter and
  prompt-prefix actions do, so no new keymap action or default binding is needed.
- Do not bypass contextual safety or launch planning on the direct prefix routes.
- Do not add a feature flag or a Python copy of backend launch logic.
- Do not live-reload this setting inside an already-running TUI; like other cached ACE
  startup settings, a config edit takes effect in the next TUI process.
