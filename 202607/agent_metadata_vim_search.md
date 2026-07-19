---
tier: epic
status: done
title: Vim-style text search for the Agents-tab metadata panel
goal: 'The Agents-tab metadata panel supports beautiful, reliable, vim-faithful incremental
  text search (`/`, `?`, `n`, `N`, `<esc>`/`q`) across every panel variant (single
  agent, clan, family, tribe, workflow, step), with easy free-form text copying from
  a jumped-to match. The previous `/` (edit query) and `?` (help) bindings move to
  `,/` and `,?` on all tabs.

  '
phases:
- id: engine
  title: Shared vim-search engine extracted from the zoom modal
  depends_on: []
  description: '''Shared vim-search engine extracted from the zoom modal'' section:
    factor the ZoomSearchMixin state machine (typing/committed modes, incremental
    preview, n/N wrap, command-line rendering, scroll-to-match math) into a host-agnostic
    controller and re-host the zoom modal on it with zero behavior change.'
- id: remap
  title: Remap edit-query and help to leader chords `,/` and `,?`
  depends_on: []
  description: '''Remap edit-query and help to leader chords'' section: retire the
    app-level `slash`/`question_mark` bindings for edit_query and show_help, re-home
    both as leader-mode keys on all tabs, and update every hint surface (help modal,
    footer, onboarding, command palette, quickstart) that displays those keys.'
- id: panel-search
  title: Inline search-mode on the agent metadata panel
  depends_on:
  - engine
  - remap
  description: '''Inline search-mode on the agent metadata panel'' section: bind `/`
    and `?` on the Agents tab to a new search-mode hosted on the AgentDetail prompt
    panel using the shared engine, covering all panel variants, with yank/selection
    support, help and footer documentation, and unit plus PNG visual snapshot coverage.'
bead_id: sase-76
---

# Plan: Vim-style text search for the Agents-tab metadata panel

## Product context

The Agents tab of `sase ace` renders all metadata for the selected row — single agents, clan summaries, family
summaries, tribe summaries, workflow documents, bash/python steps, and attempt-pinned views — into one widget:
`AgentPromptPanel` (a Textual `Static` holding a Rich renderable) inside the `#agent-prompt-scroll` `VerticalScroll` of
the `AgentDetail` composite (`src/sase/ace/tui/widgets/agent_detail.py`). That panel can be many screens tall, and today
the only find-in-page search lives in the fullscreen zoom modal (`Z`), implemented by `ZoomSearchMixin` in
`src/sase/ace/tui/modals/zoom_panel_search.py`: incremental `/` and `?` search with smartcase, live match highlighting,
a vim-style command line with `[i/N]` match counts, `n`/`N` repeat with wrap feedback ("search hit BOTTOM, continuing at
TOP"), and scroll restore on cancel.

This epic brings that exact, already-loved search experience to the inline metadata panel — no zoom required — and frees
the `/` and `?` keys to trigger it. The design goal is that a vim user's fingers already know the feature: `/foo<enter>`
`n` `n` `N` `<esc>`. Uniformity with the zoom modal's search is itself a UX feature: one search grammar everywhere.

Three keymap facts shape the work (all in `src/sase/default_config.yml` under `ace.keymaps`):

- `/` is currently `edit_query` (`slash`), an all-tab action that opens the agent filter modal on the Agents tab and
  query editors elsewhere (`src/sase/ace/tui/actions/base.py`, `action_edit_query`).
- `?` is currently `show_help` (`question_mark`), global on all tabs (`src/sase/ace/tui/actions/navigation/_modals.py`,
  `action_show_help`).
- Both must move to leader-mode chords: `,/` and `,?`, with `,?` working on **all** tabs.

## High-level design

### Interaction model (the spec)

While the Agents tab is active and no modal/prompt input owns the keyboard:

- `/` opens **forward** incremental search over the metadata panel; `?` opens **reverse** search. A one-line vim-style
  command bar appears docked at the bottom of the detail panel: sigil + query + block cursor on the left, gold `[i/N]`
  match count (or dim red "pattern not found") on the right, rendered by the shared `render_search_command_line`
  (`src/sase/ace/tui/widgets/search_command_line.py`).
- **Typing mode**: printable characters extend the query; `backspace`/`ctrl+h` edit it; matching is incremental (vim
  `incsearch`) with smartcase (existing `find_search_matches` in `src/sase/ace/tui/widgets/_vim_search.py`); all matches
  are highlighted, the current match distinctly, and the view scrolls to keep the current match visible with a few
  context rows. `<enter>` commits the search; `<esc>`/`ctrl+c` cancels and restores the pre-search scroll position. All
  other keys are consumed (typing mode owns the keyboard).
- **Committed mode** (search-mode proper): `n`/`N` jump to the next/previous match, wrapping with the vim wrap
  notifications; `/` and `?` start a fresh search from the current match; `<esc>` **or** `q` exits search-mode (the `q`
  press is consumed — it must never quit the app from search-mode); `y` yanks (see "Copying text" below), `Y` yanks the
  current match's whole line; the detail-scroll keys (`ctrl+d`/`ctrl+u`/`ctrl+f`/`ctrl+b`) scroll the search view.
  **Every other key exits search-mode and then performs its normal action** — j/k keep navigating agents, `<tab>` keeps
  switching tabs, `z`/`,`/`%` still enter their modes. This "exit + passthrough" rule is uniform by construction: no
  hand-maintained structural-key lists that drift as keymaps evolve.
- Committing with an empty query or zero matches exits search-mode with the scroll restored (vim behavior).
- The command bar's border subtitle doubles as contextual help: `[enter] accept  [esc/^c] cancel` while typing, and
  `[n/N] next/prev  [y] yank  [esc/q] close` when committed — mirroring the zoom modal.

### Search overlay architecture

Follow the zoom modal's proven overlay pattern rather than mutating the live panel:

- `AgentDetail.compose()` gains a hidden `#agent-search-scroll` (`VerticalScroll`) wrapping a `#agent-search-panel`
  (`Static`), plus the one-line `#agent-search-command` bar. On search start the prompt scroll is hidden and the search
  scroll takes its place, mirroring the prompt scroll's current layout classes (`expanded`, `layout-priority`, …) so
  geometry is pixel-identical; file/tools panels are untouched. `#agent-prompt-scroll` is visible in every
  `DetailPanelMode` (AUTO/TOOLS/INFO — see `src/sase/ace/tui/widgets/_agent_detail_panels.py`), so the swap is always
  well-defined.
- The **corpus** is captured once at search start by flattening whatever renderable the prompt panel currently displays
  (reuse `renderable_to_text` from `src/sase/ace/tui/modals/zoom_panel_rendering.py`, hoisted somewhere shared). Because
  every panel variant renders into the same widget, this covers single agents, clans, families, tribes, workflow
  documents, bash/python steps, and attempt-pinned views automatically — and any variant added later. An empty corpus
  notifies "Nothing to search" and does not enter search-mode.
- The overlay paints the corpus as one continuous `Text` (`no_wrap`, crop overflow) with the shared match/current-match
  styles, exactly like the zoom overlay, so the two surfaces look and feel identical.
- **Refresh interplay**: the hidden native prompt panel keeps receiving its normal debounced updates underneath — the
  overlay is an immutable snapshot, so no pausing of app refresh machinery is needed and exit always reveals fresh
  content. Guards keep the overlay stable: while search-mode is active, the `AgentDetail` visibility/mode plumbing
  (`_apply_panel_mode`, `on_file_visibility_changed`, `on_tools_visibility_changed`, `_expand_prompt_only`) must route
  prompt-scroll class changes to the active (search) scroll; and if the selected identity changes out from under the
  panel (background refresh removed the row) search-mode exits cleanly. User-driven selection/tab/mode changes already
  exit via the committed-mode passthrough rule.

### Copying text (the "easy to select and copy" requirement)

Two complementary paths, both enabled by the overlay being a frozen, continuous, unstyled- except-highlights `Static`:

- **Mouse selection**: Textual ≥ 3 ships app-wide text selection (`App.ALLOW_SELECT` defaults to `True`; the codebase
  never disables it). On the live panel this is impractical because content refreshes clobber the selection mid-drag; on
  the frozen overlay it just works. Click-drag to select any free-form region, then press `y` — when
  `screen.get_selected_text()` returns a selection, `y` copies it; otherwise `y` copies the current match text. `Y`
  copies the current match's full line. All copies go through the existing `copy_to_system_clipboard` helper
  (`src/sase/ace/tui/actions/clipboard/_helpers.py`) and toast a short confirmation.
- Because the overlay is a single widget with no per-section markup, terminal-native selection (e.g. shift+drag) also
  degrades gracefully.

### Keymap remap design

- **App level** (`ace.keymaps.app`): `edit_query` and `show_help` are removed as app-key bindings; two new entries
  `search_forward: "slash"` and `search_reverse: "question_mark"` are added. The new actions are gated to the Agents tab
  in `AceApp.check_action`; on other tabs `/` and `?` are deliberately inert for now (extending search to the
  ChangeSpecs/Axe detail panels is explicitly future work this design leaves room for).
- **Leader mode** (`ace.keymaps.modes.leader_mode.keys`): add `edit_query: "slash"` and `show_help: "question_mark"`.
  Dispatch branches in `_dispatch_leader_key` (`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`) simply invoke
  the existing `action_edit_query` / `action_show_help`, unguarded by tab — `action_edit_query` already branches per
  tab, and help is global, so `,/` and `,?` work on **all** tabs. The current hardcoded `question_mark` branch in leader
  dispatch (the "Tab guide moved" hint) is replaced by the real help dispatch.
- **Config hygiene**: `AppKeymaps`/`_BINDING_META` in `src/sase/ace/tui/keymaps/types.py` and the `DEFAULT_BINDINGS`
  mirror in `src/sase/ace/tui/bindings.py` change in lockstep with `src/sase/default_config.yml` (the loader hard-fails
  on drift, and the keymap sync tests enforce it). Stale user overrides for the retired app keys must degrade
  gracefully: add a retired-app-keys drop set in `src/sase/ace/tui/keymaps/loader.py` mirroring `_RETIRED_LEADER_KEYS`
  so an old `app: {edit_query: ...}` override is silently dropped with a debug log rather than producing a confusing
  warning or startup failure.

### Rust core boundary

Everything here is Textual presentation state (key handling, overlay rendering, scroll math), so it stays in this repo
per the core-boundary litmus test; no `sase-core` changes.

### Performance posture (per tui_perf rules)

- All per-keystroke work is synchronous and tiny: one regex scan plus one `Text` stylize over a corpus that was
  flattened **once** at search entry. No disk I/O, no subprocesses, no new refresh paths, no timers.
- Key handling is read-only and prompt-free; search interception happens at the top of the existing central
  `EventKeyboardMixin.on_key` chain (`src/sase/ace/tui/actions/_event_keyboard.py`), the same mechanism fold/copy/leader
  modes already use, so no new dispatch machinery is introduced.
- The overlay swap is class-toggling on existing widgets; the native panel's debounced refresh path is untouched.

---

## Phase sections

### Shared vim-search engine extracted from the zoom modal

`ZoomSearchMixin` (`src/sase/ace/tui/modals/zoom_panel_search.py`) contains ~500 lines of subtle, battle-tested behavior
— the typing/committed state machine, incremental preview, `n`/`N` repeat with wrap detection, offset↔row/col math,
scroll-context targeting, command- line rendering, and scroll restore. Duplicating it for the inline panel would fork
bugs; instead, extract the host-agnostic core into a reusable engine (e.g.
`src/sase/ace/tui/widgets/vim_search_controller.py`):

- The controller owns: mode (`off`/`typing`/`committed`), direction, query, corpus, line-start offsets, match spans,
  current selection, origin/restore offsets, last-search memory, and the pure key-interpretation logic (which key does
  what in which mode). It exposes the small verbs the hosts call (`start`, `handle_key`, `repeat`, `exit`) and reports
  effects through a narrow host protocol: fetch corpus, show/hide overlay, paint overlay text and command line, scroll
  to offset, restore scroll, focus, notify. Move the shared match/current-match style constants and the wrap-feedback
  messages here too.
- Re-host `ZoomSearchMixin` on the controller as a thin adapter that binds the protocol to the zoom widgets
  (`#zoom-search-scroll`, `#zoom-search-panel`, `#zoom-search-command`) and keeps zoom-specific behavior (refresh-timer
  pause/resume, the structural-key exit set, target switching) local. **Zero behavior change** is the acceptance bar.
- Also hoist `renderable_to_text` (or a thin wrapper) to a shared, non-modal home so the inline panel can use it without
  importing from `modals/`.

Testing: the existing suite `tests/ace/tui/test_agents_zoom_panel_search.py` must pass unchanged — it is the behavioral
safety net for the refactor. Add focused unit tests for the extracted controller itself (state machine transitions,
include-origin selection semantics, wrap reporting, offset math) so the inline host in the `panel-search` phase can lean
on them.

### Remap edit-query and help to leader chords `,/` and `,?`

Free the `/` and `?` keys and re-home their actions as leader chords, keeping every discovery surface truthful:

- `src/sase/default_config.yml`: remove `edit_query`/`show_help` from `ace.keymaps.app`; add `search_forward: "slash"`
  and `search_reverse: "question_mark"` (the actions land in the `panel-search` phase — this phase adds the config
  entries, `AppKeymaps` fields, `_BINDING_META` rows, and `DEFAULT_BINDINGS` mirrors together with no-op-safe
  `check_action` gating so the tree stays green); add `edit_query: "slash"` and `show_help: "question_mark"` under
  `modes.leader_mode.keys`.
- `src/sase/ace/tui/keymaps/types.py`: drop the two retired `AppKeymaps` fields and `_BINDING_META` rows; add the two
  new search fields/rows; extend the `LeaderModeKeymaps` defaults. `src/sase/ace/tui/bindings.py` `DEFAULT_BINDINGS`
  mirrors the same edits.
- `src/sase/ace/tui/keymaps/loader.py`: add the retired-app-keys drop set so stale user overrides of
  `app.edit_query`/`app.show_help` are dropped quietly.
- Leader dispatch (`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`): add `edit_query` and `show_help` branches
  invoking the existing actions on every tab; delete the hardcoded `question_mark` "Tab guide moved" hint branch.
- Discovery surfaces — every place that renders the old keys must show the leader chord (introduce a small shared helper
  that formats a leader chord like `,?` from the registry instead of scattering string concatenation):
  - Help modal content builders: `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py`, `agents_bindings.py`,
    `axe_bindings.py` (move `/` and `?` rows into the Leader Mode sections; respect the 57-char box rules in
    `binding_common.py`).
  - Leader footer overlay: `update_leader_bindings` in `src/sase/ace/tui/widgets/_keybinding_modes.py`.
  - Onboarding/quickstart/info surfaces that display `app.show_help` today: `widgets/tab_quickstart.py`,
    `widgets/agent_onboarding.py`, `widgets/changespec_onboarding.py`, `widgets/axe_onboarding.py`,
    `widgets/axe_info_panel.py`.
  - Command palette: remove the `edit_query`/`show_help` rows from `src/sase/ace/tui/commands/_app_metadata.py` (leader
    commands are auto-derived by `_iter_leader_commands` in `commands/_mode_commands.py` — give both ids proper
    descriptions there) and update the `app.show_help` availability allowlist entries in `commands/availability.py` to
    their leader command equivalents.

Testing: update the keymap sync/registry suites (`tests/test_keymaps_defaults.py`, `test_keymaps_app_bindings.py`,
`test_keymaps_registry_loading.py`, `test_keymaps_e2e.py`, `tests/ace/tui/test_leader_keymap_dispatch.py`,
`test_leader_keybinding_footer.py`) and add pilot coverage that `,/` opens the correct editor per tab, `,?` opens help
on all three tabs, bare `?` no longer opens help, a stale user override of the retired keys loads cleanly, and the help
modal/onboarding surfaces render the chord text.

### Inline search-mode on the agent metadata panel

Build the feature on the extracted engine and the freed keys:

- **Overlay widgets**: extend `AgentDetail.compose()` (`src/sase/ace/tui/widgets/agent_detail.py`) with the hidden
  `#agent-search-scroll` / `#agent-search-panel` pair and the `#agent-search-command` bar; add matching `styles.tcss`
  rules modeled on the zoom search CSS (`#zoom-search-*` blocks). The swap mirrors the prompt scroll's layout classes;
  the `AgentDetail` visibility/mode plumbing is taught to address "the active metadata scroll" so background file/tools
  visibility events cannot resurrect the hidden prompt scroll mid-search.
- **App wiring**: a new mixin (e.g. `actions/agents/_metadata_search.py`) hosts a `VimSearchController` bound to the
  `AgentDetail` overlay, implements `action_search_forward`/`action_search_reverse` (gated to the Agents tab in
  `check_action`), and exposes the search key handler that `EventKeyboardMixin.on_key` consults **first**, before any
  mode-prefix handling. Behavior per the interaction model above: typing mode owns the keyboard; committed mode handles
  `n`/`N`/`/`/`?`/`esc`/`q`/`y`/`Y` plus overlay scrolling for `ctrl+d`/`ctrl+u`/`ctrl+f`/`ctrl+b` (read from the
  registry, not hardcoded), and applies exit-then-passthrough for everything else. Exit paths cover cancel (restore
  scroll), close (keep position), selected-identity change, and tab/panel changes.
- **Yank & selection**: `y` copies the mouse selection when `screen.get_selected_text()` yields one, else the current
  match; `Y` copies the current match's line; both via `copy_to_system_clipboard` with a confirmation toast.
- **Docs**: add a "Search" block to the Agents help modal section (`/`, `?`, `n`, `N`, `esc`/`q`, `y`/`Y`); command bar
  subtitles carry the in-mode hints (footer stays reserved for conditional keymaps per the footer convention); add
  command palette metadata for the two new app actions in `commands/_app_metadata.py`.

Testing:

- Pilot tests (new module(s) under `tests/ace/tui/`): `/` opens typing mode and the command bar; incremental preview
  counts and current-match advance; `enter` commits; `n`/`N` wrap with wrap toasts; `?` reverse semantics; `esc` cancel
  restores scroll; `q` in committed mode exits search-mode without quitting the app; `N` no longer opens the tag modal
  and `n` no longer renames while committed; exit-then-passthrough for `j`/`k` (selection actually moves and search
  closes); corpus coverage for clan, family, and tribe summary variants (search finds text unique to each summary);
  empty-corpus notify; yank paths with a stubbed clipboard; overlay stability across a background prompt-panel refresh
  and a selected-identity change.
- PNG visual snapshots (`tests/ace/tui/visual/`): typing-mode overlay with multiple highlighted matches plus command
  bar, and committed-mode current-match state — the beauty bar for this feature is pixel-level.
- Keymap/`check_action` coverage: `/` inert on ChangeSpecs/Axe tabs; actions disabled while modals or the prompt input
  own the keyboard.

## Risks and mitigations

- **Zoom refactor regression** — mitigated by phase ordering: the extraction phase must keep the existing zoom search
  test suite green before the inline host builds on the engine.
- **Key interception conflicts** — search interception sits at the top of the one central `on_key` chain, and typing
  mode consumes everything, so mode prefixes cannot half-trigger; the exit-then-passthrough rule avoids fragile per-key
  lists.
- **Layout fights with background visibility events** — addressed structurally by routing prompt-scroll class changes
  through an "active metadata scroll" accessor while search-mode is on, with a dedicated pilot test.
- **User keymap overrides referencing retired keys** — handled by the loader drop set; old configs keep loading cleanly.
- **Large corpora** — flatten once at entry; per-keystroke regex over panel-sized text is well within the zoom search's
  proven envelope.
