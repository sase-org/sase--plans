---
create_time: 2026-07-12 09:41:05
status: done
prompt: 202607/prompts/prompt_stash_panel_leader_keymap.md
tier: tale
---
# Prompt Stash Panel Leader Keymap Plan

## Goal

Add a configurable ACE leader shortcut, `,@` by default, that always opens the prompt stash panel from any main tab.

The intended behavior is:

- With exactly one stashed entry, bare `@` keeps its current convenience behavior: restore the entry immediately,
  popping an unpinned entry or retaining a pinned entry.
- With exactly one stashed entry, `,@` opens the stash panel instead, so the user can inspect, pin, edit, or otherwise
  navigate that row without restoring it first.
- With multiple stashed entries, `,@` follows the same picker path as bare `@`, including ordering, selection, restore,
  pin, delete, prompt-mode guards, and home-prompt mounting behavior.
- With no entries, `,@` reuses the existing empty-stash notification rather than opening an empty modal.

This is a TUI presentation/navigation change only. The existing prompt-stash persistence and Rust core APIs do not need
to change.

## Current Behavior

- `restore_prompt_stash` is an app-level action bound to bare `@`. It calls
  `_open_prompt_stash_panel(auto_restore_single=True)`, which auto-restores a one-entry stash and opens
  `StashedPromptsModal` for a multi-entry stash.
- Prompt-local `Ctrl+G p` posts `PromptInputBar.RestoreRequested`, which calls `_open_prompt_stash_panel()` without the
  auto-restore flag and therefore opens the panel even for one entry.
- The app leader mode has no stash action. Its former `restore_prompt_stash` / `,P` entry is deliberately retired and
  filtered from user overrides, so reviving that action identifier would blur the distinction between restore and
  panel-only behavior.
- Leader subkeys are configurable in both `LeaderModeKeymaps` and `src/sase/default_config.yml`, dispatched by
  `LeaderModeMixin`, and surfaced dynamically in the leader footer and the three tab-specific help sections.
- The stash snapshot read already runs through `asyncio.to_thread()`, satisfying the TUI responsiveness rule; the new
  key path should reuse it and must not add a synchronous count/read on the event loop.

## Design

1. Add a distinct built-in leader action named `open_prompt_stash`, defaulting to Textual key name `at` (`@`).
   - Add the same entry to `LeaderModeKeymaps.keys` and `ace.keymaps.modes.leader_mode.keys` in
     `src/sase/default_config.yml`, keeping bundled configuration as the source of truth and typed defaults aligned.
   - Keep the old `restore_prompt_stash` leader identifier in `_RETIRED_LEADER_KEYS`, so stale `,P` overrides remain
     ignored. The app-level `restore_prompt_stash` binding for bare `@` remains unchanged.
   - Preserve the existing uniqueness validation for default leader subkeys; `at` does not collide with another leader
     key.

2. Give the panel-only flow an explicit app entry point and route leader dispatch to it.
   - In `PromptBarStashMixin`, add an async panel-opening action that delegates to `_open_prompt_stash_panel()` with
     `auto_restore_single=False` (the existing default). Do not duplicate snapshot reads or modal construction.
   - In `LeaderModeMixin._dispatch_leader_key()`, match `open_prompt_stash`, remember it like other leader commands, and
     schedule the async action through Textual's callback/event-loop mechanism before restoring the normal footer.
   - This keeps the synchronous key handler responsive while the shared helper performs disk I/O off-thread. It also
     makes leader repeat re-run the same panel-only action after the modal has closed, consistent with ordinary leader
     actions.
   - Leave `action_restore_prompt_stash()` unchanged so bare `@` continues passing `auto_restore_single=True`.

3. Make the new shortcut discoverable without reading the stash synchronously.
   - Add an unconditional `@ prompt stash` (or equivalently clear panel-oriented label) entry to the leader footer on
     all tabs. Empty-store handling already provides a useful toast, so there is no need to restore the old
     `has_stashed_prompts` footer gate or perform a keypress-time disk read.
   - Add the configured leader sequence to the Leader Mode sections in PRs, Agents, and AXE help. Keep the existing bare
     `@ Restore stashed prompt` entries, using distinct labels so users can understand the auto-restore versus
     panel-opening choices.
   - Resolve both the leader prefix and subkey from the keymap registry so user overrides remain accurate in footer/help
     displays.

## Tests

Add or update focused coverage for the behavioral split and all keymap surfaces:

- `tests/test_keymaps_defaults.py` and `tests/test_keymaps_registry_loading.py`
  - Assert `LeaderModeKeymaps` and the loaded default registry contain `open_prompt_stash: "at"`.
  - Preserve the regression that stale leader `restore_prompt_stash` overrides are dropped.
  - Keep the default leader-subkey uniqueness test green.

- `tests/ace/tui/test_leader_keymap_dispatch.py` and its fake-app helper
  - Assert leader `at` exits leader mode, schedules the panel-only action, remembers the raw subkey, and refreshes the
    footer.
  - Assert repeat-last re-dispatches the panel-only action if repeat behavior is covered explicitly.

- `tests/ace/tui/actions/test_prompt_stash_restore_open.py`
  - Add direct coverage for the new panel-only action with one entry: it pushes `StashedPromptsModal`, does not restore
    into the prompt bar, and does not pop the store.
  - Retain the existing bare-`@` single-entry tests as the regression proving auto-restore behavior did not change.
  - Reuse the existing multiple-entry test/helper to prove both entry points produce the same modal contents and order.

- `tests/test_keymaps_e2e.py`
  - Exercise the real `comma`, `at` sequence and prove it dispatches the panel-opening action, while bare `at` still
    dispatches the restore action.

- `tests/ace/tui/widgets/test_keybinding_footer_restore_stash.py`, `tests/ace/tui/test_leader_keybinding_footer.py`, and
  `tests/test_keymaps_display_help.py`
  - Replace the obsolete “leader never shows restore stash” expectation with coverage that all tabs show the new
    panel-oriented `,@` shortcut.
  - Assert configured leader prefix/subkey overrides flow through footer/help text.
  - Continue distinguishing the bare `@` restore label from the `,@` panel label.

No visual snapshot update should be accepted automatically. If the added leader-footer item changes a PNG golden,
inspect the actual/expected/diff artifacts and update the snapshot only if it accurately represents this intentional
discoverability change.

## Verification

After implementation, install the workspace dependencies as required, run the focused keymap, dispatch, stash-action,
footer, help, and end-to-end tests above, then run the repository-wide required check:

```bash
just install
just check
```

If `just check` reports an intentional ACE visual snapshot delta, inspect it and run the dedicated visual suite with the
approved golden update workflow before rerunning `just check`.

## Risks and Guardrails

- **Semantic regression in bare `@`:** avoid changing `action_restore_prompt_stash()` or the `auto_restore_single=True`
  branch; tests must preserve pinned and unpinned one-entry behavior.
- **Accidentally reviving legacy `,P`:** use the new `open_prompt_stash` identifier and retain the retired-key filter.
- **Event-loop blocking:** reuse `_open_prompt_stash_panel()` and its `asyncio.to_thread()` snapshot read; do not query
  the stash synchronously from leader dispatch or footer construction.
- **Async callback lifetime/errors:** schedule the async action through the established Textual callback path and cover
  the real key sequence end to end rather than relying only on a synchronous fake.
- **Keymap drift:** update the typed leader defaults and `default_config.yml` together, and derive UI display strings
  from the loaded registry.
