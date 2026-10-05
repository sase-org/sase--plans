---
tier: tale
title: Copy the highlighted stashed prompt with y
goal:
  Add a Stash-local y shortcut that copies the highlighted prompt body while keeping the
  panel open and stash state intact.
size: small
proposed_by: bbugyi200.apollo.58
create_time: 2026-10-05 12:02:55
status: wip
---

# Copy the highlighted stashed prompt with `y`

## Goal and scope

Add a lowercase `y` shortcut to the Stash panel that copies the highlighted stashed
prompt to the clipboard. Implement this as one focused change shared by the Prompts
overlay's Stash pane and the standalone stash picker.

This is a `tale`, sized `small`: one coding agent can add the binding and action, update
discoverability, and verify the interaction using existing helpers. No store, wire
schema, Rust binding, CLI, or memory changes are needed. This is TUI interaction glue
around an existing clipboard service, within the project's Python/Rust boundary.

## Required behavior

- `y` copies the currently highlighted row, regardless of other rows' restore or delete
  marks. It does not copy every marked row or automatically choose the newest row.
- Copy the exact `PromptStashEntryWire.text` string. Preserve multiline content,
  whitespace, Unicode, raw references, and bundle separators (`\n---\n`); a bundled row
  copies all its stored prompt bodies in their stored order. Do not copy the truncated
  row label or the humanized/styled preview, expand macros, split and rejoin the bundle,
  or append preview metadata.
- The separately stored `frontmatter` field is excluded, matching the existing Trash-row
  copy payload in
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore_trash.py`. This is
  a prompt body copy, not a new draft-export format. A selected row with an empty body
  copies that exact empty string; guard on the presence of an entry, not text
  truthiness.
- Copying leaves the panel open, keeps its highlight and scroll position, and preserves
  pins, staged restore/delete marks, stash contents, and the originating prompt bar. It
  emits no restore, delete, pin, or lifecycle request.
- An empty list or missing/invalid highlight is a safe no-op with no clipboard request
  and no success notification.
- Navigation and closing remain responsive during clipboard delivery. The text belongs
  to the row selected when `y` was pressed, even if selection changes before delivery
  finishes.
- Use existing clipboard success/OSC 52 reporting and manual-copy fallback. Clipboard
  failure must not consume a stash or dismiss the stash panel; closing a fallback modal
  returns to it.
- The shortcut is scoped to Stash. History filter typing, History's `Ctrl+Y`, Trash's
  `Ctrl+Y`, and main-screen bindings retain their existing behavior.

## Implementation

All paths below are relative to the repository root. Modal filenames without a full
prefix are under `src/sase/ace/tui/modals/`.

1. Add `("y", "copy_prompt", "Copy")` to the shared `STASH_BINDINGS` in
   `stash_messages.py`, following the existing plain-letter binding convention. Both
   `StashPane` (`stash_pane_widget.py`) and `StashedPromptsModal`
   (`stashed_prompts_modal.py`) already consume this list. Keep the binding on these
   hosts, rather than adding an overlay-wide or app-wide action.

2. Add a synchronous `action_copy_prompt()` to `StashControllerMixin` in
   `stash_controller.py`. Resolve `_highlighted_entry()` once and return if it is
   `None`. Pass its captured `entry.text` to the public
   `sase.ace.tui.actions.clipboard.schedule_copy_delivery` helper, using `self.app` as
   owner, `copied_label="stashed prompt"`, and a descriptive task name such as
   `sase-copy-stashed-prompt`. Retain the helper's default modal failure policy. Follow
   the neighboring history copy action's import pattern, but do not call
   `_emit_result()` or dismiss the panel.

   The helper already moves system clipboard work to a thread, schedules it outside
   Textual's serial message pump, tracks the task on the app, attempts OSC 52 within its
   size limit, and offers exact text in `CopyFallbackModal`. Do not directly call a
   subprocess, a store facade, or `copy_to_clipboard()` from the new action, and do not
   await clipboard completion inside a handler. No new message type or app handler is
   necessary.

3. Advertise `y copy` in both branches of `StashControllerMixin._hint_text()`
   (trash-enabled and permanent-delete modes). The overlay already obtains its footer
   from this method through `StashPane._hint_text()` and `PromptsModal._footer_text()`.
   Keep both footer lines legible at 100- and 120-column terminal widths, including the
   overlay's `@ newest` prefix. Update the standalone modal's key documentation as
   appropriate.

4. Add a concise context-qualified row such as
   `("y (Stash)", "Copy highlighted prompt")` to the shared `PROMPT_INPUT_SECTION` in
   `help_modal/binding_common.py`. Preserve the help popup's existing box dimensions and
   description limits. Document the shortcut, exact body payload, and panel-stays-open
   behavior alongside the Stash controls in `docs/ace.md` under Prompt Stacks.

   `src/sase/default_config.yml` and `src/sase/ace/tui/keymaps/registry.py` currently
   have no Stash keymap scope: these controls live in `STASH_BINDINGS`. No configuration
   change is necessary for this local addition, and an unused YAML key must not be
   introduced. If the implementation checkout has since migrated Stash into a
   configurable scope, use that scope and update its bundled default to `y` and its
   associated registry tests.

## Verification and acceptance

Read `tui.md`, `tui_perf.md`, `tui_screenshot.md`, and `lint_and_test.md` through
`sase memory read` before implementation/verification. Use the existing Textual test
hosts in `tests/ace/tui/modals/stashed_prompts_modal_test_helpers.py` and
`tests/ace/tui/modals/test_prompts_modal.py`. Add focused behavior coverage, for example
in `test_stashed_prompts_modal_copy.py`, and extend overlay/help tests where
appropriate. Stub clipboard transports; never touch the developer's real clipboard or
stash store.

- Drive actual `y` keypresses in both stash hosts. Navigate away from the newest row,
  stage marks on other rows, and verify that only the highlighted row's exact text is
  delivered. Check that the panel, selection, rows, marks, and pin state remain intact
  and no dismissal/lifecycle result is emitted.
- Exercise a multiline bundled body containing whitespace and Unicode, with separate
  frontmatter and raw reference text. Verify exact payload equality, including
  separators and trailing newline, and exclusion of frontmatter and UI metadata. Cover
  missing selection and the selected-empty-body contract.
- In the overlay, switch to History and type `y` in its filter; ensure no hidden Stash
  copy fires. Verify that returning to Stash still enables `y`, and that the Stash
  shortcut is inactive while Trash is focused. Use existing history fixtures to avoid
  real history I/O.
- Include one integration test with clipboard delivery held behind a controllable event:
  press `y`, then navigate before releasing delivery. Verify navigation progresses and
  the clipboard receives the original selection's text. Always release and drain the
  task during test cleanup; avoid timing sleeps.
- Exercise transport failure from the Stash action: the existing selectable fallback
  contains the exact text and its dismissal returns to the unchanged stash panel. Reuse
  existing transport coverage in `tests/ace/tui/actions/test_clipboard_delivery.py` for
  the remaining OS/OSC 52 permutations rather than duplicating it.
- Update the relevant shared-help assertion in
  `tests/test_keymaps_display_help_panels.py` and verify the stash hints advertise
  `y copy` in both modes.

Run the focused stash, overlay, clipboard-delivery, and help tests appropriate to the
changed files, then `just fix` (or at least `just fmt`) and `sase tool run check`, the
required wrapped `just check` recipe. Do not run `just check-full`.

Since footer output changes, run `just fix-tui-screenshots` with targeted selectors
after `--` for the Stash cases in
`tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py` and
`tests/ace/tui/visual/test_ace_png_snapshots_prompts_overlay.py`, plus any affected help
snapshots. Inspect the retained report and changed PNGs, especially the 100-column
footers; a `partial` report does not establish that skipped goldens are current. Keep
golden updates limited to the intended rendered changes. Use `/sase_monitor` for long
verification commands and have its continuation inspect mutating visual results before
finalization.

Done means `y` delivers the exact highlighted stash body through the shared nonblocking
clipboard path, preserves stash/editor state, is discoverable in the footer and help,
and passes the focused behavior checks, required check recipe, and reviewed targeted
visual coverage.
