---
tier: tale
title: Numbered Artifacts tabs and star-prefixed saved PR queries
goal: 'Artifacts sub-tabs are visibly numbered and directly reachable by digit, while
  saved PR queries move behind a beautiful PR-only star menu with reliable single-key
  slot selection.

  '
create_time: 2026-07-16 13:05:15
status: done
prompt: 202607/prompts/artifacts_numbered_tabs_saved_queries.md
---

# Plan: Numbered Artifacts tabs and star-prefixed saved PR queries

## Outcome

Make the Artifacts surface faster to navigate and easier to learn without overloading bare digits:

- Render the canonical Artifacts order as **1 PRs, 2 Commits, 3 Bugs, 4 Plans** and let those digits jump directly to
  the corresponding sub-tab whenever the top-level Artifacts tab is visible.
- Retire bare `0`–`9` saved-query loading. On the **PRs sub-tab only**, `*` opens a saved PR query chooser; one
  following digit loads that slot. The same chooser also supports mouse/Enter selection and `j`/`k` navigation.
- Keep the tab strip, command palette, help popup, user documentation, cached saved-query state, and visual language in
  agreement so the feature is discoverable and has one consistent mental model.

This is a focused `tale`: the behavior spans several tightly coupled TUI surfaces, but it should land atomically from
one coding agent. There is no Rust-core work because tab presentation and Textual key dispatch are frontend concerns.

## Existing architecture and constraints

- `ArtifactsView` owns the canonical `ARTIFACTS_SUBTAB_ORDER`, a reusable `PanelTabStrip`, and pane lifecycle routing.
  `ArtifactsMixin` already exposes direct actions for PRs, Commits, Bugs, and Plans, as well as bracket cycling.
- `PanelTabStrip` already has the numbered rendering used by the SASE Admin Center. Artifacts currently opts into
  uppercase-active labels but not numbers.
- App bindings are declared twice by design: `src/sase/ace/tui/bindings.py` is the class-level fallback, while
  `keymaps/loader.py` builds the runtime registry bindings. The ten saved-query digits are currently fixed bindings in
  both paths, outside `AppKeymaps`.
- Saved queries are cached in `app._saved_queries` at startup and refreshed after in-app save/delete operations.
  `SearchQueryPanel` already uses that cache to avoid disk reads in a hot render path. The new chooser must receive a
  snapshot of this cache rather than perform filesystem I/O on the Textual event loop.
- `AceApp.check_action` and command-palette availability both enforce Artifacts sub-tab scoping. Both layers must agree:
  key dispatch may not open the query chooser from Agents, AXE, Commits, Bugs, or Plans, and numbered sub-tab jumps may
  not unexpectedly switch to Artifacts from another top-level tab.
- Per the ACE guidelines, behavior changes must be reflected in the `?` help popup. Per the TUI performance guidance,
  handlers must remain I/O-free and sub-tab switching must continue through the existing lifecycle/refresh path.

## Interaction design

### Numbered Artifacts sub-tabs

Use `ARTIFACTS_SUBTAB_ORDER` as the only number-to-tab source of truth:

| Key | Sub-tab | Accent retained |
| --- | ------- | --------------- |
| `1` | PRs     | aqua            |
| `2` | Commits | gold            |
| `3` | Bugs    | red             |
| `4` | Plans   | violet          |

Enable `show_numbers=True` on the existing Artifacts `PanelTabStrip`. This deliberately reuses the Admin Center's quiet
number styling: the active number takes the tab accent, inactive numbers remain dimmer than their labels, the active
label remains uppercase, separators and centering are unchanged, and the full `number + label` cell remains clickable.

The digit bindings are fixed position shortcuts, like the Admin Center's numbered tabs. Bind `1`–`4` to the existing
`show_artifacts_*` actions in both fallback and runtime binding builders. Gate those actions to the visible top-level
Artifacts tab, while allowing them on every Artifacts sub-tab. Do not bind `5`–`0`: after this change they are ordinary
unhandled keys rather than hidden saved-query shortcuts. Keep `[` / `]` cycling as a complementary path.

Direct command-palette jumps keep their current ability to select a named Artifacts sub-tab, but their displayed key
sequences become `1`–`4` instead of keyless entries. Programmatic command execution may still enter Artifacts by name;
the visible digit keymaps themselves must never do so from Agents or AXE.

### Saved PR query menu

Treat `*<slot>` as a two-step keyboard interaction implemented with a modal, not a timed prefix mode:

1. From PRs, press `*`.
2. A compact centered chooser appears immediately from the cached saved-query snapshot.
3. Press one populated slot digit (`1`–`9`, then `0` in display order) to load it and dismiss the chooser.

The modal approach is intentionally explicit and reliable: there is no prefix timeout, no transient app mode to leak
across tab changes, and no ambiguity about what the second key will do. It also creates enough visual space to show the
actual queries instead of asking users to memorize slots.

Add a configurable app action such as `open_saved_query_picker`, defaulted to Textual's canonical key name for `*` in
`default_config.yml` and the fallback bindings. This keeps the app-level prefix compatible with the existing keymap
registry and key-display helpers; the modal's ten digit selectors remain fixed because they are the persisted slot
identifiers shown in the UI. Gate the action explicitly to `current_tab == Artifacts` and
`current_artifacts_subtab == "prs"`.

The new modal should:

- Receive an immutable/copy snapshot of `app._saved_queries` and the canonical active PR query. It must not call
  `load_saved_queries()` while composing, rendering, or handling a key.
- Render only populated slots in the familiar `1`–`9`, `0` order, with a strong but restrained gold/aqua query-themed
  frame, slot keycaps, syntax-highlighted query text, and an `active` badge on the query currently applied. Initially
  highlight the active query when present, otherwise the first populated slot.
- Accept a populated digit in one keypress, `j`/`k` or arrows plus Enter, and mouse selection. `q`/Escape cancels
  without changing query state.
- Claim all ten modal-local digits. Pressing an empty slot keeps the chooser open and emits a concise warning rather
  than closing or leaking the digit to an app binding.
- Show a designed empty state when no queries exist, including the existing save syntax (`/`, then `#<slot> <query>`)
  and a close hint. `*` should therefore always produce understandable feedback instead of silently doing nothing.
- Return a slot id to the app callback, which invokes the established saved-query application/history/selection path.
  Make that path consult the in-memory saved-query cache so the chooser and loader agree on the snapshot and the new
  interaction adds no synchronous saved-query file read. Preserve the existing `action_load_saved_query_0..9` methods as
  compatibility/command-palette dispatch points unless a mechanical generic dispatcher can replace them without changing
  external behavior.

Opening the chooser should not itself alter query history, selection, or tab state. Loading the already-active query
should retain the current no-duplicate-history behavior. A malformed cached query should close or remain safe according
to the existing loader contract and surface the existing error notification rather than crash the modal callback.

## Binding, palette, help, and documentation coherence

Update every surface that currently teaches bare saved-query digits:

- Remove the ten fixed `load_saved_query_*` bindings from `DEFAULT_BINDINGS` and the runtime binding builder; replace
  them with the four fixed Artifacts digit jumps. Add the configurable `*` picker action to `AppKeymaps`, binding
  metadata, the default config, fallback bindings, command metadata, and keymap source-of-truth tests.
- Update saved-query command catalog entries to display the configured two-key sequence (`*1`, ..., `*0`) and scope them
  to PRs in availability logic. Their command-palette executor may continue to call the established slot actions. Update
  catalog documentation and tests that currently call these "digit commands" or assume all-tab availability.
- Remove HelpModal's modal-local bare digit loaders and generated `load_query_*` methods; a digit pressed in Help must
  no longer load a query. In the help contents, change `0-9 Load saved query` to the configured prefix sequence and
  render saved slots as `[*N]` (or the configured prefix equivalent) while preserving the fixed 57-character box and
  32-character description constraints.
- Update the PR search-query panel's saved-slot badge from `[N]` to the prefixed affordance so the always-visible state
  reinforces the new shortcut without requiring Help.
- Update `docs/ace.md`, which currently promises bare digits and direct loading from Help, to describe the PR-only
  chooser, numbered Artifacts navigation, keyboard/mouse selection, and the unchanged query-save syntax.
- Do not add these global actions to the conditional footer: direct tab navigation and query-menu opening are stable
  surface-level commands and belong in the numbered strip/help documentation under the footer convention.

## Implementation areas

1. **Bindings and scoping** — update the fallback/runtime binding definitions, keymap schema/defaults/metadata,
   `AceApp.check_action`, the non-PR Artifacts allowlist, and command-palette availability/catalog metadata.
2. **Artifacts presentation** — opt the Artifacts strip into numbering and keep number mapping derived from
   `ARTIFACTS_SUBTAB_ORDER`; retain the existing watcher and pane lifecycle paths for actual switches.
3. **Saved-query chooser** — add a focused modal module, exports, styles, cached snapshot wiring, selection callback,
   active/empty states, and cache-backed slot application.
4. **Discoverability** — update Help, the search-query badge, command palette labels/sequences, and `docs/ace.md`.
5. **Regression and visual coverage** — add behavioral tests, a dedicated PNG snapshot for the chooser, and update only
   the Artifacts/Help/PR snapshots whose visible numbering or prefixed badges intentionally change.

## Verification

### Binding and interaction tests

- Assert runtime and fallback bindings contain `1`–`4 -> show_artifacts_*`, contain the configured `*` picker action,
  and contain no bare digit `load_saved_query_*` binding. Replace brittle binding-count expectations with semantic
  assertions where practical.
- From each Artifacts sub-tab, press every valid digit and verify the correct `ContentSwitcher` pane, active strip cell,
  activation/deactivation counts, and lazy first-load behavior. Verify `[` / `]` wrapping remains unchanged.
- From Agents and AXE, press `1`–`4` and `*`; verify neither the top-level tab nor any query changes. From Commits,
  Bugs, and Plans, verify digits still jump among Artifacts but `*` does not open the chooser. Verify focused text
  inputs can still type digits and `*` because the app bindings are non-priority and context gated.
- Verify command palette entries show the numbered sub-tab shortcuts and prefixed saved-query sequences, and that
  saved-query/picker commands are unavailable outside PRs while named programmatic Artifacts jumps remain correct.

### Chooser tests

- Cover deterministic `1`–`9`, `0` ordering, active-row styling/initial highlight, long-query truncation without broken
  syntax styling, and cached-snapshot construction with disk loading monkeypatched to fail.
- Cover digit, Enter, mouse, and `j`/`k` selection; Escape/`q` cancellation; empty-slot warning without dismissal; and
  the no-saved-queries empty state.
- Integration-test `*` then a populated digit from PRs: the modal closes, the canonical query changes, results reload,
  prior selection/history semantics are preserved, and selecting the already-active query does not add history.
- Prove direct bare digits no longer invoke saved query actions, including from the Help modal, and verify the Help
  panel renders the configured prefix affordance within its formatting limits.

### Visual and performance checks

- Add a PNG snapshot at a representative terminal size with several saved queries, one active query, varied syntax, and
  a long entry. Add or reuse an empty-state snapshot if it catches materially different layout behavior.
- Run the Artifacts, PR onboarding/search, and Help visual suites first; inspect actual/expected/diff artifacts before
  accepting intentional goldens. The numbered strip should remain centered at supported narrow widths, and the chooser
  must keep its title, list, and footer visible without clipping.
- Confirm chooser open and digit-to-dismiss use only cached data and immediate widget work. The selected query may use
  the established reload path, but no new filesystem read, subprocess, JSON parse, sleep, or full Artifacts rebuild may
  run synchronously merely to display the menu.

### Required gate

Run `just install`, then `just check`. Run `just test-visual` (and update snapshots only after inspecting intentional
diffs). If navigation hot paths are touched beyond simple reactive assignment, also exercise
`tests/ace/tui/bench_artifacts_jk.py` and preserve the documented p95 target.

## Non-goals and compatibility

- Do not change the saved-query file format, slot numbering, save/delete syntax, query parser, or startup fallback to
  the last/first saved query.
- Do not persist a new prefix-mode state; the chooser exists only while its modal screen is open.
- Do not remove direct saved-query action methods or palette execution without first preserving their callers and tests.
  The breaking user-facing change is specifically the bare `0`–`9` keymap, now replaced by numbered Artifacts navigation
  plus `*<slot>` on PRs.
- Do not alter Admin Center numbering or generic `PanelTabStrip` rendering; Artifacts opts into the already-established
  component behavior.
