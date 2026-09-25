---
tier: epic
title: "Finish the `:` Command Line: fix landing-audit bugs and spec gaps"
goal: "The `:` Command Line from epic sase-17x behaves as its plan specifies. Every key
  in the Keys table works and is configurable under `ace.keymaps.command_line`. No
  action crashes or silently fails. Completion reaches every candidate and every entity
  kind. No synchronous disk I/O runs on the UI thread. The chrome and popup match the UX
  specification. The missing PNG goldens exist, and CI builds a sase-core that exposes
  `CommandLineGrammar`.

  "
phases:
  - id: core-pin
    title: Move the sase-core CI pin past CommandLineGrammar
    depends_on: []
    size: xsmall
    description:
      "core-pin: ratchet `sase-core-revision.txt` to sase-core's remote HEAD so CI
      builds a core that exposes `CommandLineGrammar`, then prove the pinned-bindings
      check passes."
  - id: worker-hops
    title: Fix call_from_thread misuse on the app loop
    depends_on: []
    size: small
    description:
      "worker-hops: stop calling `call_from_thread` from coroutines that already run on
      the app loop. This fixes the `v` pager crash on an unloaded tail and the popup
      that never leaves `indexing commands…`. Sweep the package for the same pattern and
      add pilot tests on the real paths."
  - id: key-behavior
    title: Make every Keys-table key behave as specified
    depends_on:
      - worker-hops
    size: medium
    description:
      "key-behavior: wire ↑/↓ prefix-filtered history. Offer ghost text only at the end
      of the line. Scope NORMAL mode and Block-nav key capture to their contexts. Add
      `ctrl+f` menu accept. Limit `R` to confirmation-declined blocks. Keep the draft on
      the palette `:` hop. Drive keys through the real input in the tests."
  - id: keymap-config
    title: Apply the ace.keymaps.command_line scope
    depends_on:
      - key-behavior
    size: medium
    description:
      "keymap-config: build the panel's bindings and input routing from
      `CommandLineKeymaps`, add the Block-nav actions to the scope, and render hints
      from live key names. Update the schema, the docs and the onboarding touchpoints."
  - id: completion-fixes
    title: Popup, provider-footer and cache correctness
    depends_on:
      - key-behavior
    size: medium
    description:
      "completion-fixes: make every candidate reachable through a scrolling popup window
      and fix the highlight echo guard. Scope and clear the provider footer. Honor
      per-kind TTLs and invalidate on finish. Recheck the cursor on async results. Fix
      the hint and heading text."
  - id: entity-sources
    title: Proc, project, marked, path and cd completion sources
    depends_on:
      - completion-fixes
    size: medium
    description:
      "entity-sources: feed proc and project slots from real app state with provider
      fallback. Read Agents-tab marks. Add off-thread path/dir and `cd` completion. Rank
      the selected agent's plan first."
  - id: policy-io
    title: Foreground interpreter, writes chips, and UI-thread I/O
    depends_on:
      - keymap-config
    size: medium
    description:
      "policy-io: run foreground commands with the TUI's interpreter and fix the missing
      writes classifications. Move history, the tip marker, kill, the Procs jump store
      read and tail reads off the UI thread. Keep history in session-held memory that
      updates on exit."
  - id: chrome-layout
    title: Border chrome and floating popup
    depends_on:
      - completion-fixes
      - policy-io
    size: medium
    description:
      "chrome-layout: move the title, context chip, key hints and running count onto the
      frame borders, recomposed on resize. Float the popup over the transcript, anchored
      above the input at the replace-span column."
  - id: goldens-perf
    title: Goldens, perf probe, and remaining test gaps
    depends_on:
      - entity-sources
      - chrome-layout
    size: medium
    description:
      "goldens-perf: add the missing completion-popup goldens and regenerate the
      affected suites. Fix the host-path and import-budget stragglers if still red. Make
      the keystroke probe measure key-to-paint. Close the listed test gaps and take a
      live walkthrough."
proposed_by: bbugyi200.athena.sase-17x.land
parent_bead: sase-17x
create_time: 2026-09-24 20:28:36
status: wip
---

- **PROMPT:**
  [prompts/202609/command_line_landing_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/command_line_landing_fixes.md)
- **PARENT:**
  [202609/command_line_panel.md](https://github.com/sase-org/sase--plans/blob/main/202609/command_line_panel.md)

# Plan: Finish the `:` Command Line

## Context

Epic **sase-17x** (`:` Command Line) reported all 12 phases done. Its land agent then
audited the code at master `55936f429` against the epic's plan (read it with
`sase bead read sase-17x -r "<why>"`, which shows the linked plan file). The feature
works, but that audit found crashes, keys that do nothing, empty completion sources,
UI-thread disk I/O, and spec items that were never built. The epic's own tests pass
because they call helpers directly and never drive these paths. The land agent
reproduced each bug listed below with a throwaway pilot test or confirmed it by reading
the code.

This plan covers only the remaining work. The sase-17x land agent closes sase-17x after
this plan lands. The flag bead sase-181 is already closed.

**Out of scope** (tracked elsewhere, do not duplicate):

- **In-flight epic sase-18f** ("Return just check to green") phase **sase-18f.4** owns
  three things: the command-line visual fixture host path
  (`test_visual_fixture_host_paths`), the TUI import budget
  (`test_app_import_stays_under_startup_budget`), and the missing-`Z`-zoom help tests.
  The `goldens-perf` phase rechecks the first two and fixes them only if they are still
  red when it runs.
- **sase-17u** (orphaned agents-sidecar objects blocking `sase final prepare`).

**Rules for every phase.**

- Master moves quickly, so re-derive each finding at HEAD before fixing it. Line numbers
  below are from `55936f429` and will drift.
- Read `sase memory read tui.md tui_perf.md -r "<why>"` before changing any TUI code.
  The keystroke path stays synchronous, in-memory and prompt-free (tui_perf rules 1, 2,
  11 and 12).
- Put a regression test on every fix. Where possible the test drives the real widget or
  pilot path, not only a helper.
- Run `just fix`, then `sase tool run check`. Run
  `just fix-tui-screenshots -- <selector>` for rendered changes, and inspect every
  golden created or updated.
- Keep public import paths stable (`sase.ace.tui.command_line.screen`, …). Symvision
  counts only non-test consumers, so do not leave a public helper whose only callers are
  tests.
- Paths below are relative to `src/sase/ace/tui/command_line/` unless another path is
  given.

## Phase sections

### `core-pin`: Move the sase-core CI pin past CommandLineGrammar

`sase-core-revision.txt` is `6d0d0e6d5c0e…`. That revision predates sase-core commit
`1bdadab` ("feat(command-line): CommandLineGrammar resolver and sase adapter"). sase
already calls `require_rust_binding("CommandLineGrammar")`
(`src/sase/completion/command_line_grammar.py`), so CI's "Check pinned core bindings"
step and the contract test fail on the pinned core. The sase-17x.4 and sase-17x.5
follow-ups asked for this bump. The sase-core commits are already pushed to
`origin/master`.

- Run `just ratchet-core-revision`. It moves the pin to sase-core's remote HEAD and
  exits 2 when it applies a bump, which is expected. Confirm the new SHA contains
  `1bdadab`: open the linked checkout with `sase repo open sase-core -r "<why>"` and run
  `git merge-base --is-ancestor 1bdadab <new-sha>`.
- Run the pinned-binding check the `lint` job uses (`tools/check_sase_core_rs_bindings`;
  see `docs/rust_backend.md`, "The CI source revision pin"). It must report no missing
  binding.
- Commit only the pin file. Its commit message names the bindings it unblocks.

### `worker-hops`: Fix call_from_thread misuse on the app loop

`App.call_from_thread` raises when called from the app's own thread. Two async workers
run on the app loop (plain `run_worker(coro)`, not `thread=True`) and still hop with
`call_from_thread`:

- **`v` crash.** Pressing `v` on a selected block whose tail is not loaded yet
  (`tail_loaded=False`, has a `proc_id`) stops the TUI with `WorkerFailed`. The cause is
  in `screen_navigation.py`: `_open_pager_worker` awaits `asyncio.to_thread(...)` and
  then calls `self.app.call_from_thread(self._push_block_pager, block)`. A typical
  trigger is `r` on a slow command followed by `v`. The only test pre-sets
  `tail_loaded=True` (`tests/ace/tui/command_line/test_transcript_blocks.py`).
- **Grammar never refreshes the popup.** `grammar.py`'s `_load` runs on the loop and
  calls `on_ready()`. `screen.py` `_on_grammar_ready_from_worker` then calls
  `call_from_thread(...)`, which raises, and a debug log swallows the error. The popup
  stays on `indexing commands…` until the next keystroke.

Fix: after the await, call the UI-thread function directly, re-capturing state per
tui_perf rule 4 (the block may have been removed or the screen dismissed). Or make
`on_ready` a plain UI-thread callback. Sweep `command_line/`,
`modals/procs_pane_agent_jump.py`, `actions/_proc_action_completion.py`, and the
command-line exit delivery (`exits.py`) for the same misuse: any `call_from_thread`
reachable from a coroutine on the app loop.

Tests (pilot):

- `v` on an unloaded tail pushes `PagerScreen` with the loaded text and raises no
  `WorkerFailed`.
- When the grammar handle lands while the panel is open with an empty line, the popup
  and hint leave `indexing commands…` without any keystroke.

### `key-behavior`: Make every Keys-table key behave as specified

The epic's Keys table is the contract. Each item below was reproduced at `55936f429`.

- **↑/↓ history is dead.** `history_step` (`screen_navigation.py`) has no callers. When
  the menu is inactive, the `up`/`down` branch of `command_line_handle_key`
  (`screen_completion.py`) returns `False`, so TextArea just moves the cursor. The
  bottom hint still advertises `↑↓ history`. Wire INSERT `↑`/`↓` with the menu inactive
  to walk history filtered by the typed prefix: keep the typed prefix, and restore it
  past the newest entry. Menu-active `↑`/`↓` keeps moving within the menu.
- **→ in mid-line pastes the ghost.** Typing `bead`, then `←←` and `→` produced
  `be list --status openad`. `_update_ghost` (`screen_navigation.py`) sets
  `widget.suggestion` without checking the cursor position, and Textual's `cursor_right`
  inserts any non-empty suggestion. Offer the suggestion only when the cursor is at the
  end of the line, and re-evaluate it on cursor moves (clear it elsewhere).
- **NORMAL mode swallows vim keys.** `CommandLineInput._on_key` (`input.py`, the
  `BLOCK_NAV_KEYS` / `_BLOCK_NAV_CHARS` branch) forwards
  `x p y e v r R o g G K a i j k …` to the screen in NORMAL mode even when no block is
  selected, so `x` on `bead lisst` does nothing. The table gives NORMAL only `esc`
  (hide) and `k`/`↑` (move into the transcript). The Block-nav keys apply only while a
  block is selected. Route accordingly:
  - With no block selected, every other key reaches vim.
  - With a block selected, the Block-nav set is captured and `i`/`a`/`:` return to
    INSERT.
  - `ctrl+r` is an INSERT key (history search), so in NORMAL it stays vim redo. Check
    `_POPUP_KEYS` handling.
- **`ctrl+f` menu accept.** Commit `307da2dac`, which landed during the epic, made
  `ctrl+f` the prompt bar's completion-accept key. When the Command Line menu is
  **active**, `ctrl+f` accepts the highlighted item like `⏎`. When the menu is inactive,
  it keeps its vim/emacs forward-char meaning. Document it in the `docs/ace.md` Command
  Line section.
- **`R` must be declined-only.** `rerun_selected_with_confirm_flag`
  (`screen_navigation.py`) appends `-y` to any block. The plan limits `R` to
  confirmation-declined blocks (the `⊘ declined` state from run-policies). On other
  blocks `R` is a no-op with a subtle notice. Update `test_transcript_blocks.py`, which
  currently asserts `R` on `bead list`.
- **The palette `:` hop erases the draft.** `on_input_changed` in
  `modals/command_palette_modal.py` dismisses with `command_line_prefill=""`. The
  dismiss handler in `actions/base.py` (`action_open_command_palette`'s `_on_dismiss`)
  then overwrites `session.draft`. A hop must reopen the Command Line with its existing
  draft, cursor and selection. Only the no-match fallback row prefills. Give the result
  an explicit hop-versus-prefill distinction rather than overloading `""`.

Tests. Drive keys through the mounted `CommandLineInput` with a pilot. The existing
tests exercise only the pure `CompletionPopupState`.

- ⏎ with the menu active accepts and does not submit.
- Tab / shift+Tab cycle, and `ctrl+f` accepts.
- Esc with the menu active restores the typed text.
- ↑/↓ walk the prefix-filtered history.
- → accepts the ghost only at the end of the line.
- NORMAL `x` edits the line when no block is selected.
- `R` on a non-declined block does nothing.
- The palette `:` hop keeps the draft.

### `keymap-config`: Apply the ace.keymaps.command_line scope

`CommandLineKeymaps` (`keymaps/app_keymaps.py`) and `ace.keymaps.command_line` in
`src/sase/default_config.yml` hold seven keys. `load_command_line_keymaps` loads them in
`keymaps/registry.py`, but nothing consumes them. `screen.py` hardcodes `BINDINGS`, and
`input.py` hardcodes the history, search and `;`-hop keys. `build_command_line_bindings`
was deleted as dead code in `114fbca89` for exactly this reason. The epic's plan says
the panel keys are configurable, and sase-17x.8 proposed making the Block-nav keys
configurable.

- Add the Block-nav actions to `CommandLineKeymaps` with today's defaults. Keep
  `shift+x` twins where uppercase letters are bound (see closed sase-wo).
  - `block_next` j, `block_prev` k, `block_first` g, `block_last` G
  - `block_toggle_expand` o, `block_pager` v, `block_kill` K
  - `block_rerun` r, `block_rerun_confirm` R
  - `block_edit` e, `block_copy_output` y, `block_copy_command` Y
  - `block_procs` p, `block_remove` x, `block_focus_input` i
- Build the screen's `Binding` list from the registry's `CommandLineKeymaps`. Mirror
  `build_config_hub_bindings` / `build_gate_modal_bindings` in `keymaps/bindings.py`,
  applied the way `modals/config_hub_pane.py` applies its scope.
- Route `CommandLineInput`'s key checks (history prev/next, history search, the palette
  hop, the Block-nav set from `key-behavior`) through the same keymaps. No literal key
  names may remain for configurable actions.
- The completion-menu keys (Tab, shift+Tab, `ctrl+n`, `ctrl+p`, ⏎ and `ctrl+f` accept,
  Esc-leaves-menu, → ghost accept) follow the zsh menu-select contract and may stay
  fixed. If they do, say so in `docs/ace.md` and in the config comments. Nothing
  advertised as configurable may be ignored.
- Render the hint strings in `screen_constants.py` (`; Command Palette`, `↑↓ history`,
  `^R search`, the Block hints, …) from live `key_display_name` values, and omit an
  action whose key is `unbound`.
- Keep `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`
  (`ace.keymaps.command_line`), the keymap validation and defaults tests, and
  `docs/ace.md` in sync. Remember the core-memory gotcha about `default_config.yml`.
- Docs touchpoints that `c83e3bd91` missed:
  - `docs/configuration.md` (about line 829): the custom-mode example still uses
    `prefix: ";"`, which now collides with the palette key. `docs/ace.md` already
    changed to `"B"`.
  - `widgets/tab_quickstart.py` and `widgets/agent_onboarding.py` show the live palette
    key but never mention `:` Command Line. Add it from the live key.
  - The onboarding goldens this changes are regenerated in `goldens-perf`. Run the
    targeted ones here if cheap.
- Tests:
  - A user override for a panel action and for a Block-nav action takes effect in a
    mounted pilot.
  - An `unbound` action is inactive and missing from the hints.
  - Schema and defaults are in sync.

### `completion-fixes`: Popup, provider-footer and cache correctness

Files: `popup.py`, `screen_completion.py`, `sources.py` (`ProviderCache`), and
`extras.py`.

- **Candidates past row 8 are unreachable.**
  - `CompletionPopup.show_items` keeps `items[:POPUP_MAX_VISIBLE_ROWS]` (8), but the
    state machine cycles through all of them (`bead ` has 30).
  - `highlight_index` wraps modulo 8, and the highlight echo then resets
    `_popup_state.index`. After 10 Tabs the index was 1.
  - Fix: render a scrolling 8-row window that keeps the state's index visible, and keep
    `CompletionPopupState.index` and the `OptionList` highlight in agreement. The footer
    `N of M` reflects the true index.
- **The echo guard never fires** (tui_perf rule 12). `highlight_index` sets and clears
  `_applying_programmatic_highlight` synchronously. `on_option_list_option_highlighted`
  (`screen_completion.py`) reads it later, from the queued async message, so it is
  always false. Replace the flag with a check that survives the queue. For example,
  ignore a highlighted message whose index equals the last programmatic index, or use a
  generation counter.
- **The provider footer is sticky.** `_provider_note` is reset only after a later
  successful fetch. One failure stamps `⚠ <kind> unavailable` on every later popup,
  including subcommand and option rows. Scope the note to the current slot's
  `value_kind` and clear it when the slot or kind changes. Show "unavailable" only for a
  failed fetch. An empty but successful fetch shows a neutral `no <kind>` footer. Today
  `pending_plan` with no pending plans says "unavailable" (`extras.py` provider-footer
  text).
- **The cache ignores volatile kinds.** `ProviderCache` uses a flat 15 s TTL.
  `482fb46bc` (landed during the epic) added `VOLATILE_KIND_TTL_SECONDS` in
  `src/sase/completion/kinds.py` (for example, `pending_plan` is 5 s). Use
  `VOLATILE_KIND_TTL_SECONDS.get(kind, default)`. Also invalidate the provider cache
  when a command-line block finishes (the exit-watch settle). Otherwise `plan approve X`
  followed by `plan approve <Tab>` still offers `X`.
- **Stale-result drop.** `_provider_fetch_task` re-checks the line after the await but
  not the cursor. The epic plan requires dropping the result when either changed.
- **Text fixes:**
  - The popup footer shows `⇥ accept` and `⇥ complete` together (`popup.py` footer plus
    `screen_completion.py`). Show one key hint that fits the menu state.
  - The idle hint says `441 commands` (`len(handle)`, every node) instead of the
    top-level count (63).
  - The empty state lacks the `RECENT` and `FOR <selection> · selected <kind>` section
    headings from the UX mock and uses only badges. Add the headings as non-selectable
    rows. Enter on a command row still inserts without running.
- Tests:
  - Tab through 30 items reaches item 30 with a visible highlight.
  - Programmatic highlights do not reset the index.
  - The footer clears on a slot change, and empty and failed fetches read differently.
  - The per-kind TTL applies, and the cache is invalidated on finish.
  - A cursor move during a fetch drops the result.

### `entity-sources`: Proc, project, marked, path and cd completion sources

Files: `sources.py`, `extras.py`, `builtins.py`, and `screen_completion.py` (the fetch
hookup).

- **Proc slots get nothing.** `_proc_candidates` looks for `visible_procs`, `procs` or
  `list_procs` on `app._proc_observer`, and `ProcObserver` has none of these. `proc` is
  in `IN_MEMORY_VALUE_KINDS`, so `needs_provider_fetch` never asks the existing `proc`
  provider either. Read the app's latest delivered observer projection instead: see
  `_apply_proc_observer_snapshot` in `actions/_proc_action_completion.py` and
  `ProcObserverSnapshot.projection`.
- **Project slots get nothing.** `_project_candidates` reads `_projects`, `projects` or
  `_project_names`, and nothing sets any of these on the app. The `ValueKind.PROJECT`
  provider exists (`src/sase/completion/candidates/providers.py`), but it is skipped for
  the same in-memory reason.
- Rule for every in-memory kind: when the in-memory reader yields nothing or is
  unavailable, fall back to the debounced provider fetch. When both yield candidates,
  merge them with in-memory rows first.
- **Marked rows ignore Agents-tab marks.** `marked_values_for_kind` (`extras.py`) reads
  `_artifacts_marked_targets["agents"]`, but Agents-tab marks live in
  `app._marked_agent_order`. Read the agent kind from there, keep the other kinds, and
  test that `‹N marked›` appears for a variadic agent slot.
- **There is no path or dir completion.** `sources.py` marks `path`/`dir` as native and
  never fetches them, and the Rust resolver returns no items (`proc run --cwd ` gives
  0). The epic's Sources list includes Paths. List them in the debounced worker, off the
  keystroke path: one `scandir` of the typed directory prefix, relative to the working
  context's cwd, `~`-expanded, directories suffixed with `/`, bounded, and cached per
  directory with a short TTL. Feed the results as dynamic candidates.
- **`cd` has no completion** (proposed by sase-17x.11; a `complete_cd` helper was
  dropped as dead code). `cd` completes directories through the path source,
  `+<project>` through the project source, and `-`. The `cd +project` lookup in
  `builtins.py` currently runs inline on the UI thread. Move it off-thread and apply the
  pin when it lands.
- **Selection first for plans.** `selected_entity_values` should also yield the selected
  agent's plan name (`plan_name(agent.plan_path)`, the helper `482fb46bc` uses), so
  `plan approve` / `plan reject` rank the selected agent's plan first.
- Tests:
  - Proc and project slots return candidates from app state, and fall back to the
    provider when the state is empty.
  - Agent marks produce a marked row.
  - Path completion lists a temp dir without I/O on the keystroke path (assert the fetch
    happens in the worker).
  - `cd` completes directories and projects.
  - The selected plan ranks first.

### `policy-io`: Foreground interpreter, writes chips, and UI-thread I/O

- **The foreground run uses PATH `sase`.** `_run_foreground` (`screen_submission.py`)
  runs `["sase", *tokens]`, but procs use `sase_command_argv` from
  `ace/tui/durable_ops.py` (the TUI's own interpreter). Use `sase_command_argv` for
  foreground runs too, keeping the working-context cwd. Update the policy-routing tests.
- **Writes classification** (`src/sase/completion/run_policy.py`):
  - `tool stop` kills a run but reports `writes: false`. Commit `cae16be3c` added it
    during the epic.
  - `plan approve` / `plan reject` also report `writes: false`.
  - Add them to the writes-true overrides and regenerate
    `tests/completion/snapshots/cli_spec.json`. The drift and contract tests must stay
    green.
- **Synchronous disk I/O on the UI thread** (tui_perf rule 1; the epic plan says "no
  disk I/O on open"):
  - **History.** It is re-read on every open (`screen.py` creates a per-screen
    `CommandLineHistory`). Hold it on the app-level `CommandLineSession`: load it once,
    off-thread, and reuse it on reopen.
  - **In-memory history misses new commands.** `exits.py` writes finished commands only
    to the store, so ghost text and `RECENT` miss them until reopen. Update the
    in-memory history at submit and exit, and write the store off-thread.
  - **The palette-moved tip marker** is checked on every open (`screen.py`). Read it
    once, off-thread, and cache the result on the session.
  - **`K` kill** calls `kill_store_task` inline (`screen_navigation.py`). Use a thread
    worker the way the Procs pane's kill path does, keeping `ConfirmKillModal`.
  - **The Procs ⏎ jump** reads the store inline (`modals/procs_pane_agent_jump.py`,
    `ensure_block_for_proc`). Do the read off-thread, then open the panel.
  - **Tail polling** reads logs on the loop for every running block (`transcript.py`
    tail tick). Limit it to visible running blocks while the panel is shown, as the plan
    specifies, and do the `ProcLogCursor` reads off-loop in the pump-free task.
- Tests:
  - The foreground argv uses the interpreter.
  - The writes chip appears on `tool stop` and `plan approve`.
  - Reopen does no history or marker disk read (patch the store to fail on read after
    first load).
  - A finished command shows in ghost text and `RECENT` without a reopen.
  - Kill and the Procs jump run in workers.
  - Hidden or off-screen running blocks are not tailed.

### `chrome-layout`: Border chrome and floating popup

The epic's UX specification ("Geometry and chrome") differs from the shipped layout.
`screen.py` composes the title, context chip, key hints and running count as rows inside
the frame, and the popup as a full-width in-flow row.

- Put `❯ Command Line` on the top border's left (`❯` bold `#FFD700`) and the
  working-context chip on its right. Put the context key hints on the bottom border's
  left and `N running` on its right. Use Textual `border_title` / `border_subtitle`
  alignment, or a padded composed title. Pad and recompose on resize so both ends stay
  aligned, and middle-truncate the chip to fit.
- Float the popup **over** the transcript, as in Helix. Anchor it just above the input
  row at the replace-span column, clamped to the frame. It keeps the 8-row window from
  `completion-fixes` and the doc peek beside it on terminals at least 140 columns wide.
  Use a Textual layer or overlay, not an in-flow row, so the transcript does not reflow
  as the popup opens and closes.
- Keep the frame at 96% wide (max 160 columns), with height growing to 65% and `ctrl+t`
  toggling full height.
- Tests: title and hints survive resize (pilot at two widths), the popup does not change
  the transcript's height, and the popup column tracks the replace span.

### `goldens-perf`: Goldens, perf probe, and remaining test gaps

- **Missing completion-popup goldens** (the completion-popup phase's list, proposed by
  sase-17x.9): typing with the popup open, the signature with an active slot, the
  `⚠ writes` chip, a diagnostic undercurl, and `indexing commands…`. Add them to
  `tests/ace/tui/visual/test_ace_png_snapshots_command_line.py`.
- **Regenerate** every `command_line_*` golden changed by `chrome-layout`,
  `completion-fixes` and `keymap-config`, plus the onboarding/quickstart goldens that
  `keymap-config` changed. Use `just fix-tui-screenshots -- <selectors>` through
  `/sase_monitor` when long. Inspect every creation and update against the UX mock.
- **sase-18f.4 stragglers.** Re-run `tests/ace/tui/test_visual_fixture_host_paths.py`.
  The command-line visual test embedded `cwd="/home/test/projects/sase"`: use an allowed
  synthetic owner or the renderer's home snapshot. Also re-run
  `tests/ace/tui/test_app_import_budget.py`. If the Command Line added eager import
  edges into the app import graph, defer them; do not raise the budget. Fix either one
  only if it is still red and sase-18f.4 has not landed it.
- **Perf probe.** `_record_keystroke_probe` (`screen_completion.py`) times only the
  synchronous compute. Measure keystroke to popup paint (for example, stamp on key and
  record after refresh), keep it gated on `SASE_TUI_PERF=1`, and honor the existing
  perf-path override convention. Add a test that the probe writes a sample only when
  enabled.
- **Never-awaits test.** The current test (`test_completion_popup.py`) exempts
  `command_line_handle_key` and does not cover `CommandLineInput._on_key`. Assert that
  the keystroke compute path (resolve, complete, render) is synchronous. The only awaits
  allowed are the dismiss and hop transitions.
- **Remaining test gaps:**
  - line/cursor recapture in `_provider_fetch_task`
  - `_maybe_prepend_marked_row` insertion through the screen
  - doc-peek rendering, not only `doc_peek_visible()`
- **Live walkthrough.** Read `sase memory read tui_screenshot.md -r "<why>"`, then use
  `sase screenshot` on the checkout build. Press `:` and type `bead show `, then Tab,
  run the command, hide the panel, and reopen it. Then check `p` Procs, ↑ history, `v`
  on a running block, and the palette `:` hop keeping the draft. Record what you saw in
  the phase close note.
