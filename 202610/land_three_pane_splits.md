---
tier: tale
title: Finish and land the three-pane splits epic (sase-1eu)
goal:
  Finish the remaining sase-1eu work found during land verification, then close the
  epic. That work is restoring the red pager tests, fixing the pager ctrl+w arm, closing
  the deck contract gaps, adding the missing Pilot coverage, making the ctrl+shift
  chords actually arrive through tmux, splitting the oversized files, finishing the
  docs, and retiring the leftover epic-symbol entries.
size: medium
proposed_by: bbugyi200.athena.sase-1eu.land
bead: sase-1eu
create_time: 2026-10-02 23:26:05
status: wip
---

- **PARENT:**
  [202610/three_pane_splits.md](https://github.com/sase-org/sase--plans/blob/main/202610/three_pane_splits.md)
- **BEAD:**
  [sase-1eu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eu/README.md)

# Finish and land the three-pane splits epic (sase-1eu)

## Context

Epic **sase-1eu** ("Three-pane splits for the Agents deck and the pager") has all eight
phases closed (sase-1eu.1 … sase-1eu.8). Its plan is `plan:202610/three_pane_splits.md`;
run `sase bead read sase-1eu -r "<why>"` for the path. Read that plan's **UX contract**,
**Keys**, **Beauty spec**, **Guardrails for every phase** and **Epic acceptance bar**
before starting.

The land agent's verification at master `ba91bf93c1` found the shared `PaneGrid` model,
the deck and pager adapters, the new keys, persistence, the flag removal and most docs
in place. It also found the gaps below. This tale finishes them **and performs the epic
closeout in the same turn**. Nothing resumes the landing after this tale, so the
closeout steps at the end are mandatory.

The land agent already triaged every child `PROPOSED FOLLOW-UP:` note and recorded the
outcomes on sase-1eu, so do not re-triage them. Phase workers' rules still apply: record
any new discovered work not caused by this epic with `/sase_new_task`.

Sase-1es.6 (`54427ed47c`, Line-API `ScrollView` pager body) landed **after** the epic's
pager phases. The body widget is now `PagerBodyScroll` (`#pager-body-scroll`), and the
body API is `_invalidate_body_paint` / `_invalidate_body_layout` /
`_ensure_body_layout`.

If tests fail with "compiled profile digest does not match payload" or a missing
`sase_core_rs` attribute, the workspace venv is stale. Run `just install` first.

## Work

Do the steps in this order. Steps 1–3 fix behavior and master-red tests and come first.

### 1. Restore the red tests on master (caused by this epic or its merge with sase-1es.6)

- `tests/pager/test_app_three_panes.py` lines ~274, 296, 317, 589 and 612 still assert
  `len(<view>.query("#pager-body")) == 1`. Sase-1es.6 deleted that widget. Change each
  to `"#pager-body-scroll"`, as sase-1es.6 did in `tests/pager/test_app_split.py`.
  - Strengthen these survivor assertions (they are sase-1er's regression coverage). The
    survivor must render non-empty body content, and a `j` press must actually move its
    scroll (`scroll_y` or the reading line changes), not just the footer.
- `tests/ace/tui/actions/test_view_files_pager_split_keys.py` (~lines 96–98) still
  expects `|` on a two-pane split to rotate (`BELOW` → `BESIDE`). Since unflag-docs, `|`
  nests a third pane and `ctrl+t` turns.
  - Rewrite the sequence to the current contract.
  - Extend `_DECK_ACTIONS` and the host's `BINDINGS` with the new deck actions:
    `toggle_deck_focus_reverse`, `swap_deck_panel_next`, `swap_deck_panel_prev`,
    `close_deck_panel` and `turn_deck_layout`, each on its default keys.
  - Assert that the ACE modal pager handles `ctrl+b`, `ctrl+shift+f/b`, `>`, `<`,
    `ctrl+shift+d`, `ctrl+x` and `ctrl+t` itself, and that the Agents deck never splits
    and no deck action fires.

### 2. Pager `ctrl+w` arm fixes (`src/sase/pager/_screen_split.py` and label landing)

The contract says label landing, `Esc`, and **any focus or structure change** clear the
preview. Results are **never redirected**.

1. **Two-pane vanished target redirects.** `show_in_other_view` decides `split` from the
   _current_ pane count. When a two-pane arm's target closes before the async result
   lands, the grid is single, so the code opens a brand-new split instead of canceling.
   The stale `_armed_other_target` is also never cleared.
   - Fix: if an arm captured a target (`_armed_other_target` set, or the view's pending
     "other" action came from an arm), consume it first. If the captured pane is gone,
     cancel with "The other pane closed before the link landed." regardless of the
     current pane count.
   - Only the unarmed single-pane path may open a split.
   - Add a Pilot test: two panes, arm `ctrl+w`, close the target, land the result.
     Assert no new pane, the message shown, and the capture cleared.
2. **Preview sticks when a label lands without a pane result.** The preview and the
   capture clear only inside `_take_armed_target`. These outcomes never reach it:
   - the attached-handler path (`_screen_actions_labels.py` ~151);
   - URL copy (~156);
   - an unresolved ref (`_screen_actions_resolve.py` ~264);
   - a media target (~276).

   Fix: clear the arm (preview **and** capture) on every terminal outcome of an armed
   label, for example through one `_disarm_other()` helper called from each exit path.
   Add Pilot tests for a URL label and an unresolvable label under `ctrl+w`.

3. **Structure or focus change keeps a stale arm.** `_apply_split_state` drops the
   preview frame through `set_pane_role`, but the host capture and the view's pending
   "other" action survive, so the footer still announces a target. Fix: focus, swap,
   turn, resize, close and nest while armed cancel the arm completely (preview, capture,
   pending action, footer). Add a Pilot test: three panes, `ctrl+w`, then `ctrl+t`. The
   footer no longer shows the armed target, and the next label follows in place.

### 3. Deck contract gaps (`src/sase/ace/tui/widgets/decks/`, `_agent_detail_deck_*`)

1. **Three-panel resize must clamp to the minimums.** `decks/layout.py` `step_ratio`
   steps without a fit check, so `{` on a C3 main panel at a 100×34 area leaves it 30
   columns wide, under `MIN_DECK_PANEL_WIDTH` (40). Mirror the pager (`_screen_split.py`
   resize, ~795):
   - with three panels, compute the candidate grid;
   - if
     `fits(candidate, w, h, min_width=MIN_DECK_PANEL_WIDTH, min_height=MIN_DECK_PANEL_HEIGHT)`
     fails, keep the state unchanged, with no toast.

   Thread the area extent the way `turn_deck_layout` does (`_deck_area_extent`). Add a
   unit and a Pilot test.

2. **Search passthrough keys.** Add these to the `ids` tuple in
   `actions/agents/_deck_search_host.py` (~33–47), so they exit a committed deck search
   the way `\`, `|` and `ctrl+f` do:
   - `toggle_deck_focus_reverse`
   - `swap_deck_panel_next`
   - `swap_deck_panel_prev`
   - `close_deck_panel`
   - `turn_deck_layout`

   Add a test: with a committed search, `ctrl+x` closes the panel and leaves no
   committed search on a hidden widget.

3. **Registry pairs.** Add the two pairs the epic plan required to
   `_CONTEXTUAL_APP_DUPLICATES` in `keymaps/registry.py`:
   - `frozenset({"swap_deck_panel_prev", "start_ancestor_mode"})`
   - `frozenset({"swap_deck_panel_next", "start_child_mode"})`

   Each needs a "Tab-disjoint" comment. Confirm the partners' availability is really
   tab-disjoint. Add a validation test: a user override `swap_deck_panel_next: ">"` (raw
   `>`) is accepted, not reverted with a warning.

4. **Close moves Textual focus to the survivor.** `close_deck_panel` in
   `_agent_detail_deck_layout.py` (~432) only refreshes chrome; after a close
   `app.focused` is `None`. After applying the state, focus the new focused panel's
   visible scroll widget, as the click and `ctrl+f` paths do. Add a Pilot assertion that
   `app.focused` lies inside the surviving focused panel after closing each of the three
   panels.
5. **Erase prunes the erased session.** In `decks/layout.py`, the erase transition
   (~179–184) keeps the erased pane's `DeckPanelState` in `panels`, while
   `close_deck_panel` prunes it. Prune it on erase too, so `panels` keys always equal
   `grid.panes`, and assert that invariant in tests.
6. **Remove the last two-pane remnants.**
   - Delete the dead `_zoom_half_glyph` fallback in `decks/titles.py` (~48–60, it uses
     `1 - panel_index`). Callers always supply the glyph from `position_glyph`.
   - Make `DeckArea.panel(pane_id)` (`decks/area.py` ~54–72) resolve through an explicit
     pane-ID → `DeckPanel` map built at compose time, rather than `query_one` by DOM ID
     with a hard-coded `(0, 1, 2)`.

   Keep widget identity: no remounts.

Leave the zoomed-from-single split key alone. It restores and then splits, as the
`_open_deck_split` docstring documents. No hidden panel exists there, so nothing is
lost, and the A8 "only restore" rule targets zoomed-from-split states.

### 4. Missing Pilot coverage from the epic acceptance bar

Use the full app or the existing deck Pilot harnesses (for example
`tests/ace/tui/widgets/decks/test_deck_three_panels.py` and the deck split Pilot tests).
Assert widget identity (`is`) and no mount or unmount events wherever structure changes.

- **Deck:**
  - Real key presses for `ctrl+b`, `ctrl+shift+f`, `ctrl+shift+b`, `>`, `<`,
    `ctrl+shift+d`, `ctrl+x` and `ctrl+t` reach their actions on a two- and a
    three-panel deck.
  - Swap keeps `DeckPanel` identity **and** each panel's scroll offset.
  - Click focus works after a swap.
  - Each of the three panels can be closed, and erase works with main focused and with a
    pair panel focused, through key presses (today these are covered only at model
    level).
  - Rapid repeated split, swap and close keys leave a valid state.
  - The Artifacts tab keeps `<` / `>` and the Beads pane keeps `ctrl+t` (tab-disjoint
    availability).
- **Pager:**
  - Swap and turn keep each view's identity **and** its scroll or reading position.
    Today only identity is asserted.

### 5. Make the ctrl+shift chords actually arrive through tmux (terminal-chain completion)

The land agent proved on this host (tmux 3.5a, Textual 8.0.1) with a private tmux server
under a pty. It fed kitty's CSI-u bytes (`ESC[102;6u` for ctrl+shift+f) into the tmux
client and logged what the pane program received:

- With the committed chezmoi config (`extended-keys on`, default
  `extended-keys-format xterm`) and Textual's only request (kitty `ESC[>1u`, which tmux
  3.5a ignores), the app receives bare `0x06` / `0x02` / `0x04`. So `ctrl+shift+f/b/d`
  arrive as `ctrl+f/b/d`. `extended-keys always` (mode 1) also flattens ctrl+letter.
- Only when the app requests modifyOtherKeys mode 2 (`ESC[>4;2m`) does tmux forward the
  shift.
  - With `extended-keys-format xterm` it sends `ESC[27;6;102~`, which Textual's
    `_re_extended_key` cannot parse.
  - With `extended-keys-format csi-u` it sends `ESC[102;6u`, which Textual parses as
    `ctrl+shift+f`.
- Under mode 2 + csi-u, other keys arrive in exactly the CSI-u forms Textual already
  handles under kitty directly:
  - Tab `\t`, Enter `\r`, Esc and Backspace unchanged;
  - shift+tab `ESC[9;2u`, alt+f `ESC[102;3u`, ctrl+space `ESC[32;5u`, ctrl+x
    `ESC[120;5u`;
  - `>`, `<`, `|` and `\` plain; f12 `ESC[24~`.

Do this:

1. **chezmoi.** Open the linked repo with `sase repo open chezmoi -r "<why>"` (use only
   the printed path, and read its `AGENTS.md`). In `home/dot_config/tmux/tmux.conf`, add
   `set -s extended-keys-format csi-u` next to the existing `set -s extended-keys on`
   line, and update the comment. Keep the kitty `no_op` maps: kitty documents `no_op` as
   "passing it to the program running inside it". Do not run `chezmoi apply`. This repo
   change is a declaration obligation.
2. **sase: request modifyOtherKeys=2 only inside tmux.** Add a small Textual driver hook
   used by both `AceApp` (`src/sase/ace/tui/app.py`) and the standalone pager app
   (`SasePager`, `src/sase/pager/app.py`):
   - When the real Linux driver enters application mode **and** `TMUX` is set, write
     `\x1b[>4;2m`.
   - When it leaves application mode (exit **and** suspend, e.g. running `$EDITOR`),
     write `\x1b[>4;0m`, so the shell in the pane never receives CSI-u afterwards.
   - Re-send it on resume.
   - Headless/test drivers and non-tmux sessions are untouched.

   A `LinuxDriver` subclass selected through `driver_class` / `get_driver_class` is one
   way. Keep it tiny, put it in a module that imports no heavy UI code, and do not
   regress the pager cold-path import-weight tests.
   - Read `sase_flags.md` first, and follow it if it requires a flag for this
     user-reaching encoding change.
   - Unit-test that the sequences are written only when `TMUX` is set, and that stop or
     suspend writes the reset.

3. **Verify with the private-tmux pty harness** (do not touch the live tmux session).
   - Start `tmux -L <tmpname> -f <conf>` under a Python `pty.fork()` with
     `TERM=xterm-kitty`, using the new chezmoi config lines.
   - Run a tiny Textual key logger that uses the new driver hook. Write `ESC[102;6u`,
     `ESC[98;6u`, `ESC[100;6u` and `ESC[111;6u` to the pty master, and confirm the app
     logs `ctrl+shift+f/b/d/o`.
   - Confirm `ctrl+f`, `j`, `>`, Tab, shift+tab, Enter, Esc, Backspace, ctrl+space and
     f12 still log their usual names.
   - After the app exits, confirm a raw reader in the same pane receives legacy bytes
     again.
   - Kill only the private server.
   - Record the results and the updated manual checklist as a note on sase-1eu.1:
     `chezmoi apply`, then press the chords inside tmux inside kitty in ACE and the
     pager.

### 6. File sizes (plan guardrail: stay under the 700-line `toobig` tier)

- Split `src/sase/ace/tui/widgets/_agent_detail_deck_layout.py` (802 lines) by moving
  the pane-key actions (focus, swap, close, turn, ratio) into a sibling mixin module.
  The epic plan named this split.
- Split `src/sase/pager/_screen_split.py` (800 lines), for example by moving the
  `ctrl+w` arm/preview/target code into a sibling mixin.
- Bring `_app_action_availability.py` (717) and `keymaps/registry.py` (703) under 700 if
  that is a trivial extraction. Otherwise leave them and say so in the close note.
- No behavior change; keep `_lint-toobig` green.

### 7. Docs, help and docstrings

- `src/sase/ace/tui/actions/_debug_leaks.py:9`: the docstring still says `ctrl+shift+d`.
  Make it `f12`.
- `docs/ace.md`:
  - ~5272–5274 says a layout key while zoomed "ends the zoom without restoring".
    Describe restore-only for zoomed-from-split.
  - ~5600–5610: the picker text says "fill the other panel" and shows glyph-less hints.
    Describe the MRU target and the glyph hint ("show in the ◲ bottom-right panel").
  - Add the seven-geometry diagram from the epic plan's UX contract.
  - Fix the keys-table row (~1391) broken by an unescaped `|`; use `\|` inside the
    table.
- `docs/configuration.md`: add rows for `swap_deck_panel_next` / `swap_deck_panel_prev`
  (`ctrl+shift+f,greater_than_sign` / `ctrl+shift+b,less_than_sign`) and
  `close_deck_panel` (`ctrl+shift+d,ctrl+x`).
- `docs/pager.md`: add the terminal note (chords need the kitty → tmux CSI-u chain from
  step 5; `>` / `<` / `ctrl+x` always work), modeled on `docs/ace.md` ~5770. Update the
  `docs/ace.md` terminal note the same way.
- Help: the Agents help modal deck rows (`modals/help_modal/agents_bindings.py`) and the
  pager help Panes block (`_trail_chrome_help.py`) each lead with the one-sentence split
  grammar. That sentence is: `\` / `|` erase a full-span divider, else draw one through
  the focused pane, else turn a three-pane layout.
- If a help or footer golden changes, inspect it and regenerate it only through
  `just fix-tui-screenshots -- <selectors>`. Single- and two-pane goldens must otherwise
  stay byte-identical.

### 8. Retire the leftover epic-symbol entries

`sase bead epic-symbols sase-1eu` lists four entries, at `Justfile` ~399–402:
`Geometry`, `GridSpec`, `geometry` and `main_pane` in
`src/sase/ace/tui/util/pane_grid.py`. Their only consumers outside `pane_grid.py` are
tests, which do not count. Read `symvision.md` with `/sase_memory_read`, then:

- `geometry`, `Geometry` and `main_pane`: privatize them (`_geometry`, `_Geometry`,
  `_main_pane`) and update the in-file callers and the tests that import them; tests may
  import private names. If one is truly dead, delete it and its tests instead.
- `GridSpec`: give it a real consumer by annotating the adapters' `grid_spec(...)`
  results or parameters with `GridSpec`, in `decks/area.py` and the pager grid-apply
  code. Privatize it if that is not natural.
- Delete the four `--epic-symbol 'sase-1eu(...)'` lines from the `Justfile`. Re-key
  nothing: no open bead needs them.
- Regenerate the golden transition table only if the notation changes, and say why.

### 9. Acceptance-bar records (go in the close note and a sase-1eu note)

- **Agents j/k p95 with three panels vs two, against the 16 ms budget.** Extend an
  existing bench (`tests/ace/tui/bench_tui_deck_view.py` or
  `tests/ace/tui/bench_tui_jk.py`) with a visible-panel-count scenario (2 vs 3). Run it
  with `SASE_TUI_PERF=1` and record p50/p95 for both.
  - If p95 misses 16 ms, fix it within the existing debounce and coalescing patterns.
  - Otherwise file the gap with `/sase_new_task`.
- **Live look passes.** For both surfaces, run real content at 80x24, 120x40 and 205x65
  (the `tui.md` / `tui_screenshot.md` reference memories describe `sase screenshot`):
  - Agents: long replies, diffs, FINAL and streaming.
  - Pager: long markdown, diffs and time bands.

  Judge frame strength, the `◰`–`◳` glyph weight and narrow-pane truncation, and attach
  PNGs and findings as a sase-1eu note. If the live harness cannot reach the pager,
  render the T shapes at those sizes through the visual-test harness instead, and say
  so.

### 10. Verification

- Read `lint_and_test.md`, `tui.md` and `symvision.md` with `/sase_memory_read`.
- Run the targeted suites:
  - `tests/pager/` (non-visual);
  - `tests/ace/tui/widgets/decks/`;
  - `tests/ace/tui/util/test_pane_grid*.py`;
  - the keymap tests;
  - `tests/ace/tui/actions/test_view_files_pager_split_keys.py`.
- Run the deck and pager visual suites in check mode. Agents:
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`. Pager:
  `tests/pager/visual`.
- Then run `sase tool run check`. Do **not** run `just check-full`.
- Known unrelated failures, already filed (do not fix them here):
  - time-band goldens drift (sase-1f8);
  - intermittent history-golden chrome loss (sase-1f9);
  - `test_prompt_key_io_probe_counts_main_thread_calls` (sase-1eq);
  - the load flakes sase-1f0, sase-1al, sase-1f5, sase-1f6 and sase-1f7.

### 11. Commit message (release notes)

The unflag-docs commit (`c62e4f1491`) shipped without the release-please callouts the
epic plan required. Write this turn's commit message as a conventional
`feat(ace-pager): ...` subject. At the bottom of the body, add these extra
conventional-commit lines, which release-please reads as separate entries (do not use
`!` or `BREAKING CHANGE`):

```
feat(ace): the other split key now nests a third pane (T layouts) on the Agents deck and in the pager
feat(ace): ctrl+t turns a split; the other split key no longer rotates
feat(ace): same-key unsplit on the Agents deck keeps the focused panel
feat(ace): debug_leak_snapshot moved from ctrl+shift+d to f12
feat(ace): ctrl+shift+f/b/d chords now reach SASE inside tmux (requires the updated tmux extended-keys config)
```

### 12. Closeout (mandatory, same turn, last)

1. Close **sase-1er** (pager close and unsplit detach the surviving view) as fixed by
   this epic:
   `sase bead close sase-1er --note "<fixed by 5dff14eab4/e076ff435c; survivor regression tests in tests/pager/test_app_split.py and test_app_three_panes.py green>"`.
2. Run `sase bead epic-symbols sase-1eu`. It must list nothing. Otherwise finish step 8;
   re-key an entry only if a still-open bead truly needs it.
3. Close the epic:
   `sase bead close sase-1eu --note "<what was verified: steps 1–9, test and check results, bench numbers, live-look note>"`.
   - Never use `--force` to make the close succeed.
   - If the close is rejected for leftover epic-symbol entries, finish that cleanup and
     close again.
4. Run `just symvision` (through `sase tool run` if the recipe guard requires it) and
   confirm it is clean of sase-1eu entries.
5. Set `status: done` in the frontmatter of the epic's plan file
   `plan:202610/three_pane_splits.md`. Use the PLAN path that `sase bead read sase-1eu`
   prints.
6. Sase-1eu has no `parent_bead`, so the landing ends here.

## Guardrails

- Presentation code stays in Python; nothing moves to `sase-core`.
- No pane remounts on any structural key. Single- and two-pane goldens stay
  byte-identical unless a step above explicitly changes help or footer text.
- No I/O in key handlers; keep the coalesced off-thread deck-state save.
- Every Agents keymap change updates `default_config.yml`, `keymaps/app_keymaps.py`,
  keymap metadata, palette rows and the help modal together.
- `PaneGrid` semantics do not change. If you must change them, update the golden
  transition table in the same change and explain why.
