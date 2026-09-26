---
tier: epic
title: 'Close the `:` Command Line landing gaps: hide/hop deadlocks, key and source
  bugs, stale goldens'
goal: 'The `:` Command Line from epics sase-17x and sase-17x.13 hides, hops, and reopens
  without wedging the app. Every configurable key routes through `ace.keymaps.command_line`.
  Completion sources return fresh, complete candidates. No state mutation or disk
  write runs on the wrong thread. `just lint` passes the line-count gate. Every golden
  this epic changed is regenerated and inspected, and a live walkthrough succeeds
  end to end, including a screenshot export after hide and reopen.

  '
parent_bead: sase-17x.13
phases:
- id: split-completion
  title: Bring screen_completion.py under the line-count limit
  depends_on: []
  size: small
  description: 'split-completion: extract a cohesive helper module from the 1039-line
    `screen_completion.py` so `just lint` (toobig, enforced by the master-gate CI
    job) passes again, keeping symvision and import paths green.'
- id: hide-hop-deadlocks
  title: Fix the hide and palette-hop deadlocks and the lost loop hops
  depends_on: []
  size: medium
  description: 'hide-hop-deadlocks: stop awaiting `dismiss()` from the panel''s own
    key handler (Esc on an empty line and the `;` hop both wedge today). Route that
    Esc through `hide_panel`. Deliver the Procs focus target without `call_from_thread`
    on the loop, and refresh every screen that reopened while the grammar load was
    still running.'
- id: key-routing
  title: History walk, menu keys, and compact key hints
  depends_on:
  - split-completion
  - hide-hop-deadlocks
  size: medium
  description: 'key-routing: reset the history walk cursor on each new walk. Decouple
    menu movement from the history bindings and fix the menu hint. Render compact
    `esc` / `^R` key names. Drive the remaining key contracts through real key presses.'
- id: completion-sources
  title: Fresh caches, project and cd resolution, and the keystroke probe
  depends_on:
  - key-routing
  size: medium
  description: 'completion-sources: bypass the provider disk cache after a block finishes
    and drop in-flight stale fetches. Resolve `cd +<label>` and `+home`, and merge
    the provider into project slots. Offer `cd -` and dotfiles. Stamp the perf probe
    at the key. Close the listed source test gaps.'
- id: ui-thread-state
  title: Tip marker write, restored-block append, and history counts
  depends_on:
  - hide-hop-deadlocks
  size: small
  description: 'ui-thread-state: move the palette-tip marker write off the loop. Append
    restored blocks on the UI thread only. Remember each run once in memory. Add the
    writes-chip and off-screen-tail pilot tests.'
- id: goldens-walkthrough
  title: Regenerate epic goldens and take the live walkthrough
  depends_on:
  - completion-sources
  - ui-thread-state
  size: medium
  description: 'goldens-walkthrough: fix the doc-peek visual setup and tighten the
    never-awaits test. Regenerate and inspect the 22 goldens this epic changed. Resolve
    sase-18o. Take the live walkthrough, including a screenshot export after hide
    and reopen.'
proposed_by: bbugyi200.athena.sase-17x.13.land
create_time: 2026-09-25 08:41:36
status: done
bead_id: sase-17x.13.10
---

- **PROMPT:** [prompts/202609/command_line_landing_gaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/command_line_landing_gaps.md)
- **PARENT:** [202609/command_line_landing_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202609/command_line_landing_fixes.md)
- **BEAD:** [sase-17x.13.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17x/sase-17x.13.10.md)

# Plan: Close the `:` Command Line landing gaps

## Context

Epic **sase-17x.13** ("Finish the `:` Command Line") closed all nine of its phases. Its
land agent then audited master at `ea130c678` against that epic's plan. Read the plan
with `sase bead read sase-17x.13 -r "<why>"`, which shows the linked plan file. Most of
the work is real and tested, and the command-line and completion suites pass (581
passed). But the audit found the problems below. The land agent reproduced each one with
a throwaway pilot test or a targeted visual run, or confirmed it by reading the code.
This plan covers only that remaining work. The sase-17x.13 land agent closes
sase-17x.13, and then sase-17x, after this plan lands.

The most serious finding is that the panel **wedges on Esc**. `CommandLineInput._on_key`
runs `await self.screen.dismiss(None)` for Esc on an empty INSERT line, and
`hop_to_palette` runs `await self.dismiss(None)` for `;`. Both run inside a message
handler of a widget on the screen being popped. The pop waits for that handler to
finish, and the handler waits for the pop, so neither ever completes. Textual's own
`ScreenError` guard misses this case because the active message pump is the input
widget, not the screen.

A pilot reproduced both:

- After `page.press("escape")` on an empty line, `press` never returns. The dismissed
  `CommandLineScreen` stays attached, and app teardown hangs.
- After `page.press("semicolon")` the panel is gone, but `CommandPaletteModal` never
  opens.

This explains two earlier reports. Phase sase-17x.13.4 recorded that the `;` press
wedges Pilot. Phase sase-17x.13.9 recorded that `sase screenshot --window` timed out
after a hide and reopen. A repo-wide grep finds no other `await …dismiss(` /
`await …pop_screen(` in `src/sase/ace/tui`.

**Out of scope.** These are tracked elsewhere; do not duplicate them:

- **sase-18n**: narrow top-bar usage PNG timeouts.
- **sase-18y**: session-panel fold-levels PNG timeout.
- **sase-194**: nondeterministic scroll in the Admin Center agent-CLI/plugins PNGs. This
  covers `config_center_updates_mark_all_clis`, `config_center_agent_cli_install_detail`
  and `config_center_plugins_community_detail`.
- **The stale `agents_retry_e2e_plan_session_countdown` golden**, noted on
  sase-17m.5.1.6.
- **sase-14o**: bead-store host isolation.
- **sase-13a**: zsh `sbd` flake.
- **sase-195**: the tui_perf rule 12 wording.
- **`tests/tool/test_settlement.py` over the line limit.** It is not from this epic. The
  toobig_split routine owns file splits (sase-18h).

**Rules for every phase.**

- Master moves quickly. Re-derive each finding at HEAD before fixing it; the line
  numbers below come from `ea130c678` and will drift.
- Read `sase memory read tui.md tui_perf.md -r "<why>"` before changing TUI code. The
  keystroke path stays synchronous, in-memory and prompt-free (tui_perf rules 1, 2, 11
  and 12).
- Put a regression test on every fix. Drive the real widget or pilot path wherever
  possible.
- A pilot test for a transition that could deadlock must bound itself:
  `asyncio.wait_for(page.press(...), timeout=...)` or an equivalent. A regression must
  then fail, not hang the lane.
- Run `just fix`, then `sase tool run check`. Only `goldens-walkthrough` regenerates
  goldens; earlier phases may leave rendered drift for it.
- Keep public import paths stable (`sase.ace.tui.command_line.screen`, …). Symvision
  counts only non-test consumers. Do not leave a public helper whose only callers are
  tests, and do not import private helpers across modules (see sase-180).
- Paths are relative to `src/sase/ace/tui/command_line/` unless another path is given.

## Phase sections

### `split-completion`: Bring screen_completion.py under the line-count limit

`screen_completion.py` is 1039 lines. Commit `3c6e8f04d` (sase-17x.13.9) pushed it past
the 1000-line `toobig` limit. `just lint` runs toobig, and so does the master-gate CI
job (`.github/workflows/master-gate.yml`), so master's lint job is red. (`just check`
skips toobig.) The completion code is contended, so split it first.

- Skip this phase if the toobig_split routine has already split the file at HEAD.
- Otherwise, move one cohesive group into a sibling module, for example the keystroke
  perf probe (`_record_keystroke_probe` and its helpers) or the path and `cd` candidate
  helpers.
  - The screen mixin keeps thin call sites.
  - Aim well under 1000 lines (the 850 warning line), so later phases have headroom.
- Keep symvision clean: no cross-module imports of `_private` names, and no public
  helper whose only callers are tests.
- Verify with `.venv/bin/toobig src 1000 850 700`, `just symvision` and
  `sase tool run check`. `tests/tool/test_settlement.py` also fails toobig; it is out of
  scope.

### `hide-hop-deadlocks`: Fix the hide and palette-hop deadlocks and the lost loop hops

- **Esc hide.** In `input.py` `_on_key`, the empty-line Esc branch must call
  `self.screen.dismiss(None)` without awaiting it. Store the draft first, as
  `action_hide_panel` does.
- **Esc goes through the keymap.** Route that branch through `keymaps.hide_panel` with
  `binding_matches_key`, not the literal `"escape"`.
  - With `hide_panel` rebound, Esc on an empty INSERT line falls through to vim.
  - With it `unbound`, nothing in INSERT hides the panel.
  - Keep Esc-with-text entering NORMAL, and keep Esc in an active menu leaving the menu.
- **`;` hop.** `hop_to_palette` (`screen_navigation.py`) must not await `dismiss()`.
  Dismiss, then open the palette after the pop is queued, for example with
  `app.call_later(app.action_open_command_palette)`.
  - The panel's draft, cursor and selection must survive, so the palette `:` hop can
    return to them.
  - `_on_key` must no longer await `hop()`. After this phase its only remaining await is
    `super()._on_key(event)`.
- **Procs focus target.** In `src/sase/ace/tui/modals/config_center_modal.py`:
  - `_open_initial_tab` runs on the app loop (a plain `run_worker(coro)`).
  - It calls `_post_proc_focus_target`, which calls `app.call_from_thread`. That raises
    on the app thread, and a debug log swallows the error.
  - As a result, the Command Line `p` key (`block_procs`) opens Admin Center → Procs but
    never focuses the proc.
  - Fix: call `_deliver_proc_focus_target()` directly after the await. Keep a threaded
    hop only if a real non-loop caller exists (check). Otherwise remove
    `_post_proc_focus_target`.
- **Grammar readiness on reopen.** Each open pushes a new `CommandLineScreen`. While the
  first load is pending, `ensure_command_line_grammar_loaded` (`grammar.py`) returns
  `False` without registering the new screen's `on_ready`. Only the closed screen's
  callback fires, so the reopened panel stays on `indexing commands…` until a keystroke.
  Fix both sides:
  - Keep a list of pending `on_ready` callbacks on the app and call each one once the
    load settles.
  - `_on_grammar_ready_from_worker` (`screen.py`) must no-op when its screen is no
    longer mounted or active.
- Tests (pilot, with real key presses and bounded waits):
  - Esc on an empty line hides the panel, and the old screen is fully detached
    (`is_attached` is false). Open and hide once more.
  - Esc with `hide_panel` rebound does not hide the panel.
  - `;` on an empty line opens `CommandPaletteModal` through `page.press`, and the draft
    survives a `:` hop back.
  - `p` on a selected block focuses that proc in the Procs pane through the real opener,
    not a stub.
  - A reopen during a pending grammar load leaves `indexing commands…` once the load
    lands, with no keystroke.

### `key-routing`: History walk, menu keys, and compact key hints

- **The history walk reuses the old cursor.** `history_step` (`screen_navigation.py`)
  sets `self._history.anchor = self._walk_anchor` before calling `walk()`. That defeats
  the anchor-change reset in `history.py` `walk()`.
  - Example: `↑↑` on `bead`, then typing `git` (one match), then `↑` does nothing.
  - Example: on the same prefix, the first `↑` after an edit skips the newest entry.
  - Fix: remove the pre-assignment, or reset the cursor whenever a new `_walk_anchor`
    starts.
- **Menu keys are tied to the history keys.** Menu movement in `command_line_handle_key`
  (`screen_completion.py`) keys off `history_prev` / `history_next`. Rebinding history
  therefore breaks menu `↑`/`↓`.
  - Move the menu with the fixed menu keys: `up`/`down`, `ctrl+p`/`ctrl+n`, and Tab /
    shift+Tab. `docs/ace.md` and `src/sase/default_config.yml` already document these as
    fixed.
  - Fix `COMMAND_LINE_MENU_HINTS` (`screen_constants.py`). It says `esc normal`, but Esc
    in the menu only leaves the menu.
- **Compact key names.** `_compact_key_display` (`screen_constants.py`) renders
  `key_display_name`, so the border hints read `Ctrl+R search · escape hide`. The UX
  mock reads `^R search · esc hide` (proposed by sase-17x.13.8).
  - Render `escape` as `esc`, and `ctrl+<k>` as `^<K>`.
  - Keep the arrow glyphs and the omission of `unbound` keys.
  - Update the hint assertions.
- **Key-press tests** for contracts that are currently tested only through helpers:
  - `→` accepts the ghost at the end of the line.
  - `↑`/`↓` filter by prefix, using several entries, some of which do not match.
  - `R` on a declined block reruns with `-y`. On any other block, `R` shows the notice
    (assert the notice).
  - `i`, `a` and `:` return a selected block to INSERT.
  - Grammar readiness goes through the real loader, using an in-process
    `CommandLineGrammar` as `test_chrome_layout.py` does, not a direct
    `_on_grammar_ready_from_worker()` call.

### `completion-sources`: Fresh caches, project and cd resolution, and the keystroke probe

- **The provider disk cache survives a finished block.** `candidates_for`
  (`src/sase/completion/candidates/providers.py`) reads `<kind>[__<project>].tsv` from
  `sase.completion.candidates.cache`.
  - Entries are valid by source-file mtime plus TTL: 5 s for `pending_plan`, 30 s
    otherwise.
  - For `pending_plan`, only `notifications.jsonl` is mtime-checked.
  - `invalidate_provider_cache` (`screen_completion.py`) clears only the screen's
    `ProviderCache`, and the refetch then reads the disk file. So `plan approve X`
    followed by `plan approve <Tab>` can still offer `X` (proposed by sase-17x.13.5).
  - Fix: give the TUI's post-finish refetch a way past the disk layer, either a
    `use_disk_cache=False` path through `candidates_for` or an invalidate hook in
    `cache.py`. Also drop any in-flight fetch started before the invalidation, for
    example with a generation counter.
  - Today, an active menu skips the refetch entirely. After the fix, the next render
    must refetch.
- **`cd +<project>` rejects offered names.** Completion offers project **labels**
  (`ProjectDisplaySnapshot.label_for`) and `home`. `_resolve_project_checkout`
  (`builtins.py`) matches `record.project_name` exactly and passes `include_home=False`.
  Resolve through the same display snapshot so labels map back to names, include `home`,
  and keep the lookup off the UI thread.
- **Project slots miss projects with no loaded agent.** The in-memory reader
  (`sources.py` `_project_candidates`) only sees projects of loaded agents and the
  current project. `needs_provider_fetch` skips the provider whenever in-memory rows
  exist.
  - For `project`, always merge the debounced provider fetch, with in-memory rows first.
  - Keep in-memory-first with provider fallback for `proc`.
- **Small source gaps:**
  - `cd -` appears only after `-` is typed (`screen_completion.py`). Offer it in the
    empty-argument menu.
  - Path completion always skips dotfiles (`sources.py`). List them when the typed
    basename starts with `.`.
- **Perf probe.** `_record_keystroke_probe` stamps its start at `_refresh_completion`
  entry, so the Key → `TextArea.Changed` queue delay is not measured.
  - Stamp the start on key receipt (`CommandLineInput._on_key`) and carry it to the
    paint.
  - Keep the probe gated on `SASE_TUI_PERF=1` with the `SASE_TUI_PERF_PATH` override.
  - Keep a strong reference to any task it schedules, or use `call_after_refresh`
    without `create_task`.
- **Test gaps:**
  - The `‹N marked›` row for a variadic **agent** slot, driven through a mounted screen
    with `app._marked_agent_order`, not the bare mixin.
  - The path scan runs in the debounced worker, not on the keystroke path (assert the
    thread or the call site).
  - `cd` completes directories as well as `+project` and `-`.
  - Proc and project slots really fetch from the provider (use a stub provider) when app
    state is empty.
  - Path rows reach the popup for a `--cwd` slot through `complete()`.
  - The disk-cache bypass after a finish.
  - The labelled and `home` `cd +` resolution.

### `ui-thread-state`: Tip marker write, restored-block append, and history counts

- **Tip marker write on the loop.** `_maybe_show_palette_moved_tip` (`screen.py`) calls
  `mark_palette_moved_tip_shown()` on the UI thread, which does a `mkdir` and a `touch`
  (`palette_moved_tip.py`). The earlier phase moved only the marker read off-thread. Do
  the write in a thread worker too, and keep the session cache so a reopen does no disk
  I/O.
- **Restored-block append from a worker thread.** `ensure_block_for_proc` (`restore.py`)
  appends to `session.blocks` from the Procs-jump worker thread, which races UI-thread
  iteration.
  - Read the proc off-thread.
  - Build and append the block, and trim to 200, on the UI thread only.
  - `modals/procs_pane_agent_jump.py` and `screen.py` are the callers.
- **Double-counted history.** A proc command is remembered in memory at submit
  (`screen_submission.py`) and again at exit (`exits.py`), so its in-memory `count`
  drifts from the store. Remember each run once, and let the exit only update
  `last_exit` and `last_used`.
- Tests:
  - The marker write happens off the loop thread.
  - The restored-block append happens on the UI thread.
  - One run gives one count.
  - A mounted panel shows the `⚠ writes` chip for `tool stop` and `plan approve`. Today
    only the spec contract asserts `writes`.
  - A running block that is scrolled out of view, not only one with `display=False`, is
    not tailed.

### `goldens-walkthrough`: Regenerate epic goldens and take the live walkthrough

- **Doc-peek visual setup.** `test_command_line_doc_peek_png_snapshot`
  (`tests/ace/tui/visual/test_ace_png_snapshots_command_line.py`) seeds `_popup_state`
  and calls `popup.show_items()` directly, and leaves the input empty. The golden
  therefore shows `close` candidates under an empty line, with the card at column 0 and
  no real footer. Type `bead cl` and render through `screen._render_popup`.
- **Visual helper cleanup.** Remove the visual `_open_panel`'s patch of
  `CommandLineHistory.refresh`, which the screen no longer calls.
- **Never-awaits test.** In `test_completion_popup.py`, also cover `_render_popup` and
  `_render_signature`. Assert that `CommandLineInput._on_key` awaits nothing except
  `super()._on_key`. After `hide-hop-deadlocks`, dismiss and hop are no longer awaited.
- **Regenerate** with `just fix-tui-screenshots -- <selectors>`. Hand long runs to
  `/sase_monitor`. Inspect every update against the UX mock. These are the 22 goldens
  this epic changed:
  - All 16 older `command_line_*` goldens. Chrome-layout (`403586e27`) and
    completion-fixes were never regenerated. `completion_popup_160x40` and
    `indexing_120x40` are current, but recheck them after the key-hint change.
  - `agents_onboarding_120x40`, `agents_onboarding_no_plugins_120x40`,
    `agents_decks_single_empty_120x40`, `changespecs_onboarding_120x40` and
    `changespecs_onboarding_no_match_120x40` (the `:` Command Line row from
    `bde335de5`).
  - `help_guide_agents_120x40`: its guide text comes from `agent_onboarding.py`, so only
    the scrollbar thumb moves.
- **Leave other drift alone.** Do not regenerate goldens owned by the out-of-scope beads
  above.
  - Close sase-18o with a note naming the regenerated goldens.
  - Note on sase-16a whether its help drift is gone.
  - Run a full `just test-visual` at the end. Report every remaining failure or drift
    and its owner in the phase close note.
- **Live walkthrough.** Read `sase memory read tui_screenshot.md -r "<why>"`, then use
  `sase screenshot` on the checkout build:
  1. Press `:`, type `bead show `, press Tab and accept, then run the command.
  2. Press Esc to hide the panel, then reopen it. Take a screenshot export after the
     reopen. It must succeed; it timed out in sase-17x.13.9.
  3. Press `;` on an empty line; the Command Palette must open.
  4. Use the palette `:` hop, which must return to the kept draft.
  5. Press `p` on a block; its proc must be focused in Procs.
  6. Check `↑` history and `v` on a running block.

  Record what you saw in the phase close note.
