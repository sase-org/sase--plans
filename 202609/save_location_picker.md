---
tier: epic
title: Location-first picker for new mini-xprompts and snippets
goal: 'Opening a mini-xprompt (Ctrl+G Ctrl+X, Ctrl+G x, gx) or snippet (Ctrl+G Ctrl+T,
  Ctrl+G t, gt) target pane first shows a fast location picker. One keypress chooses
  the file or directory that will store it, and Enter accepts a default whose reason
  is shown. The name step then shows the chosen location and can go back to change
  it. Keys typed while the picker is still loading are kept and applied, never dropped
  or sent to the prompt pane.

  '
phases:
- id: picker
  title: Shared save-location picker modal and choice model
  depends_on: []
  size: medium
  description: 'picker: build the pure choice builders (hotkeys, default rules, badges,
    previews) and the SaveLocationPickerModal with loading and type-ahead support,
    CSS, docs section, unit tests, and PNG snapshots.'
- id: xprompt-flow
  title: Mini-xprompt location-first flow
  depends_on:
  - picker
  size: medium
  description: 'xprompt-flow: route the mini-xprompt request through the picker, lock
    the destination in MiniXPromptNameModal with a Shift+Tab back step and namespace
    seeding, retire the old destination cycling, update tests, snapshots, and the
    mini-xprompt docs paragraph.'
- id: snippet-flow
  title: Snippet location-first flow and Ctrl+G Ctrl+T alias
  depends_on:
  - picker
  size: medium
  description: 'snippet-flow: add the Ctrl+G Ctrl+T alias, route the snippet request
    through the picker, lock the destination in SnippetNameModal with arrows moving
    matches and Shift+Tab going back, update tests, snapshots, help, and snippet docs.'
proposed_by: bbugyi200.athena.0u8
create_time: 2026-09-29 18:58:43
status: done
bead_id: sase-1cu
---

- **PROMPT:** [prompts/202609/save_location_picker.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/save_location_picker.md)
- **BEAD:** [sase-1cu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cu/README.md)

# Plan: Location-first picker for new mini-xprompts and snippets

## Why and what exists today

- `Ctrl+G Ctrl+X` / `Ctrl+G x` / `gx` call `request_mini_xprompt_target_pane()`
  (`src/sase/ace/tui/widgets/_prompt_input_bar_mini_xprompt_pane.py`). That posts
  `MiniXPromptTargetRequested`, which
  `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mini_xprompt_pane.py` handles.
  The handler awaits the catalog and last-used loads, then pushes
  `MiniXPromptNameModal`. The destination is a small panel that only `Ctrl+N`/`Ctrl+P`
  can change. Its default comes from `default_mini_xprompt_destination()`, and
  Tab-completing a name silently moves it.
- `Ctrl+G t` / `gt` call `request_snippet_target_pane()`, which posts
  `SnippetTargetRequested`. `_prompt_bar_snippet_pane.py` handles it and pushes
  `SnippetNameModal`, where `↑`/`↓`/`Ctrl+N`/`Ctrl+P` cycle the destination. Nothing
  binds `Ctrl+G Ctrl+T` today. The `t` row in `_PROMPT_G_PREFIX_BINDINGS`
  (`_prompt_input_bar_g_prefix_actions.py`) has no `ctrl_g_aliases`, while `x` has
  `("ctrl+x",)`.
- Reliability bug: both handlers await disk I/O before any modal is on screen. Keys
  typed right after the chord (`^G ^X review`) land in the prompt pane.

The user asked for this: when a new xprompt or snippet is started from these keymaps,
first show a panel that picks the storage file or directory. One keypress should pick
it, and Enter should accept a good default. The panel must be intuitive, reliable, and
beautiful.

## Design

### Flow (both kinds)

```
chord ──▶ ① Location picker ──(hotkey / ↵)──▶ ② Name step ──(↵)──▶ target pane (unchanged)
               ▲                                   │
               └────────────── ⇧Tab ───────────────┘        Esc at any step cancels and
                                                             refocuses the origin pane
```

- The picker is pushed **synchronously** in the request handler, before any `await`.
  Every load happens in a background task (see `tui_perf.md`: keep the pump free). The
  loaded data then goes to the picker.
- Never skip the picker, even when only one destination is writable. The muscle memory
  `^G ^X p review` must always mean the same thing. If the picker were skipped, the `p`
  would become part of the name.
- Every entry point for a kind goes through the same request method: `gx`, `Ctrl+G x`,
  `Ctrl+G Ctrl+X`, the mini save-review "retarget" choice, `gt`, `Ctrl+G t`, and the new
  `Ctrl+G Ctrl+T`. Retargeting or renaming an open pane shows the picker too, with the
  pane's current location as the default.
- Non-goals: the whole-stack save panel (`gX` / `Ctrl+G X`, `UnifiedXPromptSaveModal`),
  the Snippets panel add form, and the `sase snippet add` CLI keep their own destination
  UIs. No `sase-core` changes: the picker and its default policy are presentation logic
  built on the existing Python discovery helpers (`load_unified_save_locations`,
  `load_snippet_config_locations`, `resolve_snippet_save_target`, `save_state`). Those
  helpers already own the equivalent logic today.

### Picker anatomy (visual spec)

This is a centered `ModalScreen`, a visual sibling of `DispatchTargetPickerModal`:
`thick $primary` border, `$surface` background, `padding: 1 2`, width about 72% (min 64,
max about 104 columns), `height: auto`, and a max height of about 85% with the list
scrolling inside. There is one `OptionList` with one-line rows grouped under dim
`── Section ──` header rows (disabled options, as in `XPromptLocationModal`). A footer
has a live preview line and a hints line.

```
┃   New mini-xprompt · where should it live?          ● Location › ○ Name   ┃
┃                                                                            ┃
┃   ── Project · sase ─────────────────────────────────────────────────────  ┃
┃ ▌ p  📁 Project xprompts     ./sase/xprompts/               ★ last used    ┃
┃   P  📄 Project config       ./sase/sase.yml                               ┃
┃   1  📁 Project, personal    ~/sase/xprompts/sase/          new            ┃
┃   ── Home ───────────────────────────────────────────────────────────────  ┃
┃   h  📁 Home xprompts        ~/sase/xprompts/               chezmoi        ┃
┃   H  📄 User config          ~/.config/sase/sase.yml        chezmoi        ┃
┃   2  📄 sase_work.yml        ~/.config/sase/sase_work.yml                  ┃
┃   ── Plugins & built-in (6) · + to show ─────────────────────────────────  ┃
┃                                                                            ┃
┃   → ./sase/xprompts/<name>.md · called as #sase/<name> · 24 xprompts here  ┃
┃   p P 1 h H 2 pick · ↵ last used · j/k move · esc cancel                   ┃
```

```
┃   New snippet · where should it live?               ● Location › ○ Name   ┃
┃   ── Project · sase ─────────────────────────────────────────────────────  ┃
┃   p  📄 Project config       ./sase/sase.yml                has ⇥ todo     ┃
┃   ── Home ───────────────────────────────────────────────────────────────  ┃
┃ ▌ h  📄 User config          ~/.config/sase/sase.yml        ★ default      ┃
┃   1  📄 sase_work.yml        ~/.config/sase/sase_work.yml                  ┃
┃   → ~/.config/sase/sase.yml · ace.snippets.<trigger> · 12 snippets here    ┃
┃   p h 1 pick · ↵ default · j/k move · esc cancel                           ┃
```

Styling reuses the house palette so the picker looks native:

- hotkey: `bold #00D7AF`, the which-key teal from `_prompt_input_bar_g_prefix_hints.py`
- label: `bold #87D7FF`
- path: dim
- `★ <reason>`: `bold #FFD700`
- `● current`: `#87D7FF`
- `has #name` / `has ⇥ trigger`: `#D7AF5F`
- `new`: `italic #FFD700`
- `chezmoi`: dim italic
- disabled rows: fully dim, with `·` in the key column and the reason in italic
  `#D7AF5F`

Icons: 📁 directory, 📄 YAML config (already used by `XPromptLocationModal`; the
snapshot renderer bundles Noto Emoji).

Titles:

- `New mini-xprompt · where should it live?`
- `Retarget #<name> · where should it live?`
- `New snippet · where should it live?`
- `Rename ⇥ <trigger> · where should it live?`

The `● Location › ○ Name` stepper is repeated in the name step with the steps flipped,
and the completed step there shows the chosen location's label.

### Rows, sections, hotkeys

Hotkeys are mnemonic, stable across runs, and scope-first: **`p` = project, `h` = home**
in both pickers. In the xprompt picker, Shift gives the config-file variant of the same
scope. Digits number the remaining Project/Home rows in display order.

| Picker  | Key     | Row (discovery label)                                                                                        |
| ------- | ------- | ------------------------------------------------------------------------------------------------------------ |
| xprompt | `p`     | `Project sase/xprompts/` directory (namespaced `#<project>/…`)                                               |
| xprompt | `P`     | `Project sase/sase.yml`                                                                                      |
| xprompt | `h`     | `Home ~/sase/xprompts/` directory                                                                            |
| xprompt | `H`     | `User sase.yml`                                                                                              |
| xprompt | `1`–`9` | other Project/Home rows: `Project home (<project>)` shown as "Project, personal", `User sase_*.yml` overlays |
| snippet | `p`     | `Project sase/sase.yml`                                                                                      |
| snippet | `h`     | `User sase.yml`                                                                                              |
| snippet | `c`     | configured `ace.snippet_config_path` when it is outside the discovered files (own top section "Configured")  |
| snippet | `1`–`9` | `User sase_*.yml` overlays                                                                                   |

Rules:

- Section order is Configured (snippets only), then `Project · <project>`, then `Home`,
  then `Plugins & built-in` (xprompts only). Omit empty sections.
- Match canonical rows by module-level label constants exported beside the code that
  creates them (`get_all_xprompt_locations`, `load_snippet_config_locations`). Do not
  duplicate string literals.
- Canonical rows (`p`/`P`/`h`/`H`, snippet `p`/`h`) are always shown. When not writable
  they are dimmed with their `disabled_reason` and no hotkey, so users see why their
  usual choice is unavailable. Non-canonical rows that are not writable are hidden.
- Writable plugin and built-in (dev) rows go in `Plugins & built-in`. They never get a
  hotkey and start collapsed behind one summary row. `+`, or Enter or a click on the
  summary row, toggles the section. It starts expanded when the default or current row
  is inside it. Plain users never see clutter, and a sase developer can still reach
  `default_xprompts/`.
- Rows after the ninth digit get no hotkey; users can still reach them with j/k and ↵.
- The hints line is generated from the hotkeys actually present.

### Default (↵) and highlight

The default row carries a `★ <reason>` badge. The highlight starts on it, and ↵ always
picks the highlighted row. Only selectable rows qualify.

- **xprompt**:
  1. `★ current`: the open pane's location when retargeting.
  2. `★ last used`: `load_last_used_locations()["xprompt"]`.
  3. `★ default`: the Project directory, unless the prompt context is home mode.
  4. `★ default`: the Home directory.
  5. The first selectable row.
- **snippet**:
  1. `★ current`: the open snippet pane's location when renaming.
  2. `★ configured`: `ace.snippet_config_path` when
     `resolve_snippet_save_target().source == "configured"`. This is an explicit durable
     preference, and it matches how the `gX` save panel already preselects it.
  3. `★ last used`: `load_last_used_locations()["snippet"]`.
  4. `★ default`: the resolved default target (user `sase.yml`, chezmoi-aware).
  5. The first selectable row.
- Compare last-used and current paths against both a row's location path and its
  resolved write path. With chezmoi these can differ, and the snippet pane records
  `write_path`.
- If `resolve_snippet_save_target()` reports a `fallback_reason`, the footer shows a
  warning line, for example `ace.snippet_config_path unusable: <reason> — using <path>`.
- When the picker reopens from the name step (⇧Tab), the highlight starts on the
  location that was in use. `★` stays on the default. Rows that already define the typed
  name show `has #name` / `has ⇥ trigger`.
- `last used` is still recorded only after a successful save (existing behavior), never
  on pick.

### Footer preview

Each choice carries precomputed preview text for the footer:

- xprompt directory: `→ <dir>/<name>.md · called as #<ns>/<name> · N xprompts here`
- xprompt config: `→ <file> · xprompts.<name>`
- snippet: `→ <file> · ace.snippets.<trigger> · N snippets here`

When a name is known (retarget or ⇧Tab), use the concrete path from
`destination_target_for_name(...)` instead of the `<name>` placeholder. `N` is
`len(row.names)`.

### Loading, type-ahead, and failure contract (the reliability core)

- The modal opens in a **loading** state with the stepper, title, and a dim
  `Finding destinations…` row. It exposes `set_choices(choices, *, highlight_id=None)`
  and `set_load_error(message)`.
- Before choices arrive:
  - `Esc`, or `q` when no pick is pending, dismisses `None`.
  - The first hotkey character or `Enter` becomes the single pending pick.
  - After a pending pick exists, printable characters go into a `typeahead` buffer, and
    `Backspace` edits that buffer. Every other key is ignored.
- `set_choices` resolves the pending pick immediately:
  - A pending hotkey that maps to a selectable row, or a pending `Enter` (which takes
    the highlighted default), dismisses with `SaveLocationPick(choice_id, typeahead)`.
  - An unknown or disabled hotkey leaves the picker open and clears the typeahead. The
    footer then shows the reason in error style, e.g. `✗ no destination on 2` or
    `✗ ./sase/sase.yml: migrate legacy project config first`.
- After choices are loaded:
  - Hotkeys dismiss at once.
  - `j`/`k`, `↑`/`↓`, and `Ctrl+N`/`Ctrl+P` move the highlight and skip headers and
    disabled rows.
  - `Enter` or a click picks the highlighted row.
  - Other printable keys are ignored. They never mean "accept the default", because a
    name could start with a hotkey letter.
- The orchestrator stores all loaded data before it calls `set_choices`. That way the
  dismiss callback, which may run synchronously inside `set_choices`, can push the name
  step with no further awaits, so later keystrokes reach the name input and not the
  prompt pane. The name modal's initial text is the existing initial name plus the
  typeahead (cursor at end), which is what the user would have gotten by waiting.
- Load everything the name step needs (catalog or snippet data plus last-used) before
  calling `set_choices`. Then no await separates the pick from the name step.
- If the origin pane disappears during loading, the orchestrator dismisses the picker
  and shows the existing `Prompt pane is no longer available - … discarded` warning. On
  a load failure it calls `set_load_error` and sends an error notify, and `Esc` closes.
  When there are no writable destinations, the picker says so and `Enter` does nothing.
- On cancel or on a failed origin check, ignore late load results.

### Name step (both modals)

- The modal receives the chosen destination and treats it as locked. The
  `Ctrl+N`/`Ctrl+P` destination cycling is removed.
- `Ctrl+N`/`Ctrl+P` and `↑`/`↓` move the match highlight in both modals.
- `⇧Tab` dismisses with `ChangeSaveLocationRequest(text=<current field value>)`. The
  orchestrator reopens the picker and then returns to the name step with that text.
- The header shows the stepper `✓ <location label> › ● Name`. The right-hand panel
  becomes a compact "Saving to" panel with the resolved file path, the namespace and
  chezmoi notes, and a dim `⇧Tab change location` line.
- Hints become `tab complete · ↑↓ matches · ⇧tab location · enter open · esc cancel`.

## Phase `picker`: shared save-location picker modal and choice model

Files (new unless noted):

- `src/sase/ace/tui/modals/save_location_choices.py` holds pure, Textual-free builders
  that are safe to call off-thread:
  - `SaveLocationChoice`: frozen dataclass with `choice_id` (the location path),
    `hotkey` (`str | None`), `section`, `kind` (`directory`/`config`), `label`,
    `display_path`, `badges` (default reason, current, `has …`, `new`, `chezmoi`),
    `disabled_reason`, `preview`, `is_default`, and `collapsed_group` (bool).
  - `SaveLocationPick(choice_id: str, typeahead: str = "")`.
  - `ChangeSaveLocationRequest(text: str)`.
  - `xprompt_location_choices(rows: Sequence[UnifiedSaveLocation], *, last_used_path, current_path, home_mode, project, name)`.
  - `snippet_location_choices(locations: Sequence[SnippetConfigLocation], *, resolved_target: SnippetSaveTarget, names_by_path: Mapping[str, frozenset[str]], last_used_path, current_path, project, trigger)`.
  - Each builder returns the choices plus the default id. It implements every hotkey,
    section, visibility, default, badge, and preview rule above.
- `src/sase/ace/tui/modals/mini_xprompt_target_catalog.py` (edit): rename
  `_destination_defines_name` to a public `destination_defines_name`, export it, and use
  it for `has #name` badges.
- `src/sase/ace/tui/modals/xprompt_location_modal.py` and
  `src/sase/xprompt/snippet_targets.py` (edit): export the discovery-label constants
  used for canonical-row matching.
- `src/sase/ace/tui/modals/save_location_picker_modal.py` holds
  `SaveLocationPickerModal(ModalScreen[SaveLocationPick | None])`. Its constructor takes
  `kind` (`"xprompt" | "snippet"`), `title`, and optional initial
  `choices`/`highlight_id`. It implements rendering, key handling (`on_key` matches
  hotkeys on `event.character`, case-sensitive), the `+` collapse toggle, the loading,
  type-ahead, and error states, and `set_choices` / `set_load_error`. It stays
  presentation-only and does no I/O. Export it from
  `src/sase/ace/tui/modals/__init__.py` like its siblings.
- `src/sase/ace/tui/styles.tcss` (edit): add a `SaveLocationPickerModal` block next to
  the `#dispatch-target-*` rules. Do not put it inside the mini or snippet name-modal
  blocks, which later phases edit in parallel.
- `docs/ace.md` (edit): add a `### Save location picker` section immediately before
  `### Authoring a snippet from the prompt bar`. It covers the flow, the hotkey table,
  default precedence with `★` reasons, ⇧Tab back, type-ahead, and the collapsed plugin
  section. Later phases link to this anchor.

Tests:

- `tests/ace/tui/modals/test_save_location_choices.py` covers:
  - canonical letters and Shift variants
  - digits in display order, and extras after nine getting no key
  - disabled canonical rows shown with reason and no key; hidden non-canonical disabled
    rows
  - plugin and built-in rows collapsed, with no hotkeys, and expanded when they hold the
    default or current row
  - both default precedence ladders, including home mode, stale last-used paths, chezmoi
    write-path matching, and the configured outside-discovery snippet row with key `c`
  - `has …` badges and previews with and without a name
- `tests/ace/tui/modals/test_save_location_picker_modal.py` (pilot) covers:
  - a hotkey dismisses with the right id
  - ↵ picks the highlighted default
  - j/k/↑/↓/^N/^P movement skips headers and disabled rows
  - a disabled hotkey shows the reason and stays open
  - Esc and `q` dismiss `None`
  - `+` toggles
  - loading: a buffered hotkey, a buffered ↵, a buffered unknown key (stays open with
    message), typeahead capture including Backspace, and Esc while loading
  - `set_load_error`
  - the empty state
  - a click selects
- PNG snapshots in a new
  `tests/ace/tui/visual/test_ace_png_snapshots_save_location_picker.py`, modeled on
  `test_ace_png_snapshots_snippet_name.py`, which pushes the modal directly with fixture
  choices:
  - `save_location_picker_xprompt_120x40`: ★ last used, chezmoi badge, digit overlay,
    collapsed plugin summary
  - `save_location_picker_snippet_120x40`: configured row, `has ⇥` badge
  - `save_location_picker_loading_120x40`
- This phase does not rewire any chord. The existing name modals and flows stay
  untouched, so every existing test keeps passing.

## Phase `xprompt-flow`: mini-xprompt location-first flow

- `src/sase/ace/tui/widgets/_prompt_input_bar_messages.py`: give
  `MiniXPromptTargetRequested` a `current_location_path: str | None = None`.
  `request_mini_xprompt_target_pane()` fills it from the open mini target's
  `location_path`. Edit only this message class; the snippet phase edits its own class
  in the same file.
- `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mini_xprompt_pane.py`: rewrite
  the request handler as a small flow object or state holder:
  - Validate the origin, then push `SaveLocationPickerModal(kind="xprompt")`
    immediately.
  - Spawn a retained background task, using the existing
    `_spawn_mini_xprompt_pane_task`. The task first loads `load_unified_save_locations`
    and `load_last_used_locations`, then builds the catalog with
    `load_mini_xprompt_target_catalog(project, locations=rows)` so rows are not
    discovered twice. It then builds the choices off-thread, re-checks the origin, and
    calls `set_choices`.
  - On pick: map `choice_id` back to the `UnifiedSaveLocation` row and push
    `MiniXPromptNameModal` synchronously. The name text is the initial name, rebased
    onto the destination namespace, plus the typeahead.
  - On `ChangeSaveLocationRequest`: reopen the picker, rebuilding choices off-thread for
    the typed name, with the highlight on the location in use.
  - On a `MiniXPromptNameResult`: keep the existing `_apply_mini_xprompt_name_result`
    path.
  - On `None` at any step: `refocus_pane_id(origin)`.
  - Use the prompt context's `is_home_mode` for `home_mode`.
- Namespace seeding: add a small pure helper, unit-tested and next to
  `validate_name_for_destination`. When the destination has a namespace and the text
  lacks `<ns>/`, it adds the prefix; when moving away from a namespaced destination, it
  strips that destination's prefix. Result: `p` then `review` yields `sase/review`, not
  a validation error.
- `src/sase/ace/tui/modals/mini_xprompt_name_modal.py`:
  - Take a required `destination: UnifiedSaveLocation` and drop `last_used_path`.
  - Remove the `next_destination`/`prev_destination` actions and bindings, and rebind
    `ctrl+n`/`ctrl+p` to match navigation.
  - Tab completion fills the name only and no longer moves the destination.
  - Add `shift+tab` on the screen and forward it from `_MiniXPromptNameInput` /
    `_MiniXPromptMatchList`. It dismisses `ChangeSaveLocationRequest`, so the result
    type becomes `MiniXPromptNameResult | ChangeSaveLocationRequest | None`.
  - Add the stepper header, the "Saving to" panel copy, and the new hints.
  - When the typed name has an editable definition in another destination, append
    `· ⇧tab to pick <label>` to the fork verdict.
- `mini_xprompt_target_catalog.py`: delete `default_mini_xprompt_destination`, which the
  picker's default rules replace, together with its tests. Symvision flags unused
  symbols.
- `styles.tcss`: edit only rules inside the `MiniXPromptNameModal` block.
- Tests:
  - Update `tests/ace/tui/modals/test_mini_xprompt_name_modal.py`: drop the destination
    cycling tests; add locked destination, ⇧Tab result, ^N/^P matches, and Tab not
    moving the destination.
  - Add `tests/ace/tui/actions/test_prompt_mini_xprompt_location_flow.py`, driving the
    real app mixin as `test_prompt_save_snippet_pane.py` does:
    - `^G ^X` (and `gx`) shows the picker before the loaders finish, using patched slow
      loaders
    - `p` opens the name step on the Project row
    - ↵ takes the `★` default
    - the fast sequence `p` `r` `e` `v` pressed during loading ends with the name field
      `sase/rev` and nothing typed into the prompt pane
    - ⇧Tab returns with `has #…` and the highlight on the current row, then `h` rebases
      the name
    - Esc in either step restores the origin pane focus and cursor
    - retargeting an open mini pane defaults to its location (`★ current`)
    - the origin pane vanishing during loading closes the picker with the warning
  - Keep `test_prompt_save_mini_xprompt_pane.py` and
    `test_prompt_stack_mini_xprompt_pane_lifecycle.py` passing.
- Snapshots: regenerate the `mini_xprompt_name_*` PNGs (new header, panel, and hints)
  and inspect every diff. Add `mini_xprompt_location_flow_picker_120x40`, captured
  through the real chord.
- Docs: `docs/ace.md` only, in the paragraph beginning "`gx` opens or retargets one
  focused mini-xprompt pane". State that the location picker comes first and link
  `#save-location-picker`. Do not edit keymap tables, `docs/prompt.md`, or
  `binding_common.py`; the snippet phase owns those.

## Phase `snippet-flow`: snippet location-first flow and `Ctrl+G Ctrl+T`

- Alias: in `_prompt_input_bar_g_prefix_actions.py`, add `ctrl_g_aliases=("ctrl+t",)` to
  the `t` binding. The `^G` prefix handler runs before insert-mode `Ctrl+T` completion
  in `_prompt_text_area_key_handling.py`, so completion is unaffected. Confirm this with
  a test. Update:
  - `help_modal/binding_common.py` row: `gt / Ctrl+G t / Ctrl+G Ctrl+T`
  - `tests/ace/tui/widgets/test_prompt_g_prefix_hint_*.py`: alias dispatch, and the `^G`
    hint row showing `^Gt / ^G^T`
  - `tests/test_keymaps_display_help.py`

  `src/sase/default_config.yml` needs no change, because the prompt `^G` table is not
  config-driven. Verify this and say so in the commit.

- `SnippetTargetRequested` gets `current_location_path: str | None = None`, which
  `request_snippet_target_pane()` fills from the open snippet target's `write_path`.
  Edit only this message class.
- `src/sase/xprompt/snippet_targets.py`: promote `SnippetNameModal._target_for_location`
  to a public `snippet_save_target_for_location(location) -> SnippetSaveTarget`, and use
  it in both the modal and the orchestrator.
- `_prompt_bar_snippet_pane.py`: apply the same flow rewrite as the xprompt phase:
  - Push `SaveLocationPickerModal(kind="snippet")` immediately.
  - The background task keeps the existing loader seams (`_resolve_snippet_target`,
    `_load_snippet_locations`, `_load_derived_snippet_catalog`). It adds last-used and
    per-path trigger names (`names_for_location("snippet_config", path)`) loaded
    off-thread, builds the choices off-thread, and calls `set_choices`.
  - On pick: map the choice to a `SnippetSaveTarget` (the resolved target for the
    configured row, otherwise `snippet_save_target_for_location`). Push
    `SnippetNameModal` with that target and `initial_trigger` + typeahead.
  - ⇧Tab and Esc behave as in the xprompt phase.
  - Keep the modal's `locations` equal to the discovered files in first-wins order.
    Never add the configured row to them, because `snippet_collision` treats an
    outside-discovery destination specially.
- `src/sase/ace/tui/modals/snippet_name_modal.py`:
  - The target is locked. Remove `_destination_choices`, `_move_destination`, and the
    destination bindings.
  - `↑`/`↓`/`Ctrl+N`/`Ctrl+P` move the match highlight, mirroring the mini modal's
    `_move_match`.
  - Add ⇧Tab → `ChangeSaveLocationRequest`, plus the stepper header, "Saving to" panel,
    and new hints. Keep the `fallback_reason` note in the panel.
- `styles.tcss`: edit only rules inside the `SnippetNameModal` block.
- Tests:
  - Update `tests/ace/tui/modals/test_snippet_name_modal.py`: remove destination
    cycling; add locked target, match navigation, and ⇧Tab.
  - Update `test_gt_new_snippet_loop_…` in
    `tests/ace/tui/actions/test_prompt_save_snippet_pane.py` so it passes through the
    picker (press `h` or ↵). Patch the last-used loader so the default is deterministic.
  - Add `tests/ace/tui/actions/test_prompt_snippet_location_flow.py` covering:
    - `^G ^T` and `gt` open the picker instantly
    - `★ configured` beats `★ last used`
    - the configured outside-discovery row takes `c`
    - fast type-ahead `h` `t` `o` `d` `o` ends with trigger `todo`
    - ⇧Tab round trip with `has ⇥ todo`
    - renaming defaults to `★ current`
    - Esc and origin-vanished paths
- Snapshots: regenerate `snippet_name_collision_120x40` and inspect it. Add
  `snippet_location_flow_picker_120x40` through the real chord. Check that the `^G` hint
  snapshot, if any, shows the new alias.
- Docs:
  - `docs/ace.md` keymap rows for `Ctrl+G t` and `gt`: add `Ctrl+G Ctrl+T` and mention
    the picker.
  - "Authoring a snippet from the prompt bar": the new step 1 is "Choose where", linking
    `#save-location-picker`. Remove the `↑`/`↓` destination-cycling text, and
    rename-in-place now shows the picker first.
  - `docs/prompt.md`: the gx/gt sentence mentions the location picker and
    `Ctrl+G Ctrl+T`.
  - `docs/configuration.md` `ace.snippet_config_path`: it is the `★ configured` default
    of the snippet location picker and outranks last-used.

## Verification (every phase)

- Before finishing, read `lint_and_test.md` with `/sase_memory_read`, and also
  `symvision.md` if lint flags unused or private symbols. Run `just check` through
  `sase tool run` as that note directs.
- Read `tui.md`, `tui_perf.md`, and `tui_screenshot.md` before the UI work. Regenerate
  goldens with targeted `just fix-tui-screenshots -- <selectors>` via `/sase_monitor`.
  Inspect every created or updated PNG before finalizing: generation is not approval.
- Capture a live `sase screenshot` of the real chord flow in the phase that wires it,
  and confirm the picker looks right at 120x40 in dark mode (and light mode, if the
  snapshot suite covers it).
- No `sase-core` or `sase-core-revision.txt` changes are expected. If a phase finds it
  needs shared backend behavior, it records a `PROPOSED FOLLOW-UP:` note on its bead
  instead of widening scope.
