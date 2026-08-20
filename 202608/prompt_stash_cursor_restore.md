---
tier: tale
title: Restore prompt-stash pane and cursor position
goal:
  Prompt stashes preserve the active pane and cursor and restore them in mounted or
  newly created prompt input bars without regressing legacy rows.
size: medium
proposed_by: bbugyi200.athena.08p
create_time: 2026-08-20 13:32:00
status: wip
---

# Plan: Restore prompt-stash pane and cursor position

## Goal and user-visible contract

When ACE stashes a prompt draft, persist the active prompt pane and its cursor location
with the stash row. Restoring that row must focus the corresponding restored pane and
place the cursor at the same logical zero-based `(row, column)` within the text that was
actually persisted. This applies to `Ctrl+S` current-pane stashes, `gs` / `Ctrl+G s`
bundle stashes, restart recovery, stash-before-loading-an-xprompt, and `gS` pinned-stash
replacement. Restore must work both when drafts are appended to an already-mounted
prompt bar and when the app mounts a fresh home prompt bar.

Keep the current interaction semantics around the new position:

- Restored prompts still enter INSERT mode; this change remembers pane and cursor
  position, not the prior Vim mode or selection range.
- A bundle remembers the one pane that was active when the bundle was captured. Its
  cursor pane index is relative to the bundle's persisted, non-empty segments, not the
  original stack and not the existing `pane_index` ordering field.
- A current-pane stash always targets its sole persisted segment. A frontmatter-only
  draft may target its empty pane at `(0, 0)`.
- If the active pane is omitted because it is empty while another pane is captured, omit
  cursor metadata and retain the existing restore-at-the-end behavior.
- When several stash rows are restored together, preserve the current oldest-first
  append order. The final restored row determines focus: use that row's saved
  pane/cursor when present; otherwise focus its final appended pane at the end, matching
  today's behavior. Earlier rows' cursor metadata must not unexpectedly pull focus away
  from the final row.
- Rows written by older SASE versions and failed-launch recovery rows have no editor
  position. They remain fully readable and restore with the current
  last-pane/end-of-text fallback.

## Backward-compatible core wire contract

The prompt-stash JSONL schema is shared backend behavior, so extend it in the linked
`sase-core` repository rather than inventing Python-only persistence.

1. In `crates/sase_core/src/prompt_stash/wire.rs`, add a small serializable cursor
   record containing bundle-local `pane_index`, `row`, and `column`, and add it to
   `PromptStashEntryWire` as optional metadata. Use unsigned, zero-based coordinates,
   `serde(default)`, and omission for `None` so old JSONL rows deserialize unchanged and
   new cursorless rows keep the old JSON shape. This is an additive entry field, so keep
   the prompt-stash envelope schema version unchanged. Re-export the cursor wire beside
   the existing prompt-stash records and update the stale entry documentation to
   describe the canonical single-row bundle representation.
2. Let the existing append/read/pop/pin/rewrite operations and PyO3 conversion carry the
   field through unchanged; do not add a second store API or perform migration rewrites.
   Extend Rust parity tests in `crates/sase_core/tests/prompt_stash_store_parity.rs` to
   prove a cursor-bearing row survives append/read, pop, pin, and rewrite paths,
   serializes under the intended nested key, and that a legacy row without the key
   defaults to no cursor rather than `(0, 0)`.
3. In `src/sase/core/prompt_stash_wire.py`, mirror the nested cursor record and optional
   entry field. Rehydrate absent metadata as `None`, include present metadata in the
   facade payload, and export the new type. Extend
   `tests/test_core_facade/test_prompt_stash.py` for Python dict round trips, old-row
   defaults, fake-binding payloads, and a real-extension round trip. Do not manually
   edit crate versions or the `sase-core-rs` dependency window; release-plz and the
   existing core release reconciler own those versions.

## Capture and persistence

1. Extend `StashedPromptPane` in
   `src/sase/ace/tui/widgets/_prompt_input_bar_stack_models.py` with presentation-side
   cursor metadata and an explicit active marker. In
   `PromptInputBar.capture_stashable_panes()` and `stash_active_pane()`, first
   synchronize live `PromptTextArea` state, then capture the active pane identity and
   cursor alongside each persisted pane body. Keep snippet panes excluded and keep the
   existing frontmatter and pane-order behavior.
2. Because stash bodies are stored with surrounding whitespace stripped, normalize the
   cursor against the stripped body rather than copying the original widget coordinates
   blindly. Implement a pure, tested conversion through an absolute character offset:
   subtract the removed leading prefix and clamp cursors in removed leading/trailing
   whitespace to the start/end of the persisted body, then convert back to
   `(row, column)`. This keeps multiline cursor restoration exact and guarantees valid
   coordinates without changing the established stripped-text contract.
3. In `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash.py`, convert the
   captured active pane into the optional bundle-local cursor wire while building the
   immutable `PromptStashEntryWire`. Thread the same metadata through pinned-stash
   updates so `gS` replaces text, frontmatter, and cursor atomically while retaining the
   target row's id, timestamp, project, source, `pinned`, and existing ordering
   metadata. Leave failed-launch stashing cursorless because there is no live editor
   position to preserve.
4. Add focused capture and handler tests under
   `tests/ace/tui/widgets/test_prompt_stash_capture.py`,
   `tests/ace/tui/actions/test_prompt_stash_handler.py`, and
   `tests/ace/tui/actions/test_prompt_stash_update.py` covering a single multiline pane,
   a bundle whose active pane is not the last pane, leading/trailing whitespace
   normalization, frontmatter-only capture, pinned replacement, and the
   empty-active-pane fallback.

## Restore transport and widget focus

1. Replace the lossy `(text, frontmatter)` restore tuples in
   `src/sase/ace/tui/prompt_stash_entries.py` with an internal typed restore payload
   that expands each canonical bundle row into panes while marking at most one pane as
   the saved focus target. Validate the saved bundle-local index against the parser
   output; ignore an invalid index, and defer row/column bounds checking to the prompt
   widget where the final Textual document exists.
2. Update `PromptBarStashRestoreMixin` in
   `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore.py` to pass the
   enriched panes through both restore branches. For a mounted prompt bar, append all
   panes and identify the new item corresponding to the final restored row's target. For
   a fresh bar, pass the final target's parsed pane index and cursor through
   `_show_prompt_input_bar_for_home()` along with the existing xprompt-markdown text. If
   the final row lacks usable metadata, deliberately use the legacy
   final-pane/end-of-text fallback.
3. Extend `PromptInputBar.restore_stashed_entries()` to rebuild with the existing
   `PromptFocusRestore` mechanism when a saved target exists. This reuses item-id focus
   restoration and `_clamp_cursor_location()` after the new widgets have mounted.
   Preserve blank-leading-pane removal and frontmatter adoption, taking any removed
   placeholder into account when resolving the target item.
4. Add optional initial selected-pane/cursor arguments to the home-bar mount and
   `PromptInputBar` construction path. Seed the parsed stack selection before
   composition, then on mount clamp and apply the requested cursor instead of always
   calling `_cursor_to_end()`. Keep all non-stash callers on defaults so history,
   xprompt, relaunch, feedback, and ordinary empty-bar behavior remain unchanged.
5. Expand tests in `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py`, its
   restore harness, `tests/ace/tui/widgets/test_prompt_stash_restore_keymap.py`, and
   focused prompt-input initialization tests to cover: exact multiline cursor
   restoration into an empty mounted bar; append into a non-empty bar; a bundle
   targeting a middle pane; fresh-home-bar mounting; placeholder removal; frontmatter
   adoption; multiple-row final-row precedence; coordinate clamping; and
   legacy/cursorless rows still focusing the final pane at its end.

## Documentation, performance, and verification

Update the prompt-stash section of `docs/ace.md` to state that manual/restart stashes
remember the active pane and cursor, while legacy and failed-launch rows restore at the
end. No keymap, modal layout, or visual styling changes are needed, so PNG golden
updates are out of scope unless implementation reveals an unintended rendering delta.

Keep capture and restore UI mutations synchronous and in-memory; all JSONL access must
continue through the existing off-event-loop Rust facade tasks. Do not add disk work,
subprocesses, timers, or new refresh paths to typing or cursor movement handlers.

Verification order:

1. In the linked `sase-core` repository, run focused prompt-stash tests during
   development and then its required `just check`, which includes the PyO3 binding
   tests.
2. In the main repository, run `just install` so the local core extension is rebuilt
   from the linked checkout.
3. Run the focused core-facade, prompt-stash capture/update/restore, and prompt-input
   widget tests named above.
4. Run the required main-repository `just check`. If its scoped selector escalates or
   reports unusual selection, run `just check-full` through `/sase_monitor` as required
   by project instructions.

## Acceptance criteria

- A manually stashed multiline prompt restores to the same logical line and column in
  INSERT mode.
- A bundled stash restores focus to the pane that was active at capture time, including
  a non-final pane, in both an existing prompt bar and a newly mounted home prompt bar.
- Stripping surrounding whitespace cannot shift the saved position or create an
  out-of-range cursor.
- Updating a pinned stash updates its saved pane/cursor together with its content.
- Old JSONL rows, failed-launch rows, invalid target indices, and cursorless rows
  preserve the existing safe final-pane/end-of-text behavior without migration or data
  loss.
- Rust store/binding checks, focused Python/TUI tests, and the repository-wide required
  checks pass without introducing event-loop I/O or visual/keymap regressions.
