---
tier: tale
title: Bind Space to repeat last VCS xprompt and remove Run agent (home)
goal: In the ACE TUI, Space prefills the prompt with the last launched VCS xprompt
  (blank home prompt when none), Ctrl+Space is unbound by default, and the start_agent_home
  action is gone.
size: medium
proposed_by: bbugyi200.athena.0p0
status: done
---

# Move "Repeat last VCS xprompt" from `<ctrl+space>` to `<space>`; remove "Run agent (home)"

## Goal

In the ACE TUI, bare `<space>` currently runs `start_agent_home` (opens an empty
home-workspace prompt bar) and `<ctrl+space>` (canonical key `ctrl+@`) runs
`start_agent_from_patch` (opens the prompt bar pre-filled with the most recently
launched VCS xprompt, e.g. `#gh:owner/repo `).

After this change:

- `<space>` runs `start_agent_from_patch` (repeat last launched VCS xprompt).
- `<ctrl+space>` is unbound by default.
- The `start_agent_home` action is removed entirely (binding, keymap field, default
  config key, command-palette entry, help rows, availability lists). Users who want an
  empty home prompt press `<space><ctrl+u>` (the prompt bar's readline `ctrl+u` clears
  to start of line), or use `+` and pick the home entry.

No feature flag: this is a keymap default the user explicitly asked to change, and keys
remain user-configurable. A stale `start_agent_home` key in a user's config is already
handled by the registry's "Unknown keymap action(s) in config (ignored)" warning path
(`src/sase/ace/tui/keymaps/registry.py`); do not add a legacy alias for it.

Keep the `ctrl+space` / `ctrl+at` → `ctrl+@` aliases in
`src/sase/ace/tui/keymaps/key_validation.py` (`_CTRL_SPACE_KEY`, `_KEY_ALIASES`) so
users can still bind Ctrl+Space themselves; their tests stay.

Do not touch leader-mode `,<space>` (`leader_mode.keys.agent_from_cl`), which is a
different action.

## Design decisions

1. **Empty-MRU fallback.** Today `action_start_agent_from_patch` warns "No previously
   launched VCS xprompt" and does nothing when the VCS xprompt MRU is empty. Since
   `<space>` becomes the primary launch key (and the onboarding / quick-start cards
   advertise it for first-time users who have never launched a VCS xprompt), change the
   empty-MRU branch to open the plain home prompt bar instead:
   `self._show_prompt_input_bar_for_home()` with no arguments, and no warning toast.
   This keeps `<space>` useful on a fresh install and makes `<space><ctrl+u>` always a
   working escape hatch. Leave `action_start_last_vcs_xprompt_in_editor` (`ctrl+g`)
   warning behavior unchanged.

2. **Drop stale row-dependent gating on `start_agent_from_patch`.** The action no longer
   depends on the selected Patch/agent row (it reads the global MRU), yet several
   availability gates still treat it as a row action. Left alone, `<space>` would be
   dead on an empty Agents tab (exactly the onboarding case), on remote-agent rows, and
   in the palette when no Patch row is selected. Remove those gates so the action is
   available on every tab regardless of selection. Keep the mid-prompt guard (see
   below).

## Changes

### Bindings / keymaps

- `src/sase/ace/tui/bindings.py`: delete the `Binding("space", "start_agent_home", ...)`
  line; change the `start_agent_from_patch` binding key from `"ctrl+@"` to `"space"`.
  Update the adjacent comment ("Run agent from Patch (Patches view only)") to describe
  it as "Repeat last launched VCS xprompt (all tabs)".
- `src/sase/default_config.yml` (~line 724): remove `start_agent_home: "space"` and set
  `start_agent_from_patch: "space"`. Per the repo gotcha, this file must match the
  in-code defaults.
- `src/sase/ace/tui/keymaps/app_keymaps.py`: remove the `start_agent_home: str` field
  (and any default / from-dict / to-dict plumbing that references it in that file).
- `src/sase/ace/tui/keymaps/metadata.py`: remove the `("start_agent_home", ...)` entry.
- `src/sase/ace/tui/keymaps/mode_keymaps.py`: no change expected; confirm the leader
  `agent_from_cl: "space"` default is unaffected. Also check any keymap-conflict /
  duplicate-key validation does not now flag `space` (app) vs. the modal-local `space`
  toggle bindings — those are separate binding scopes and were already coexisting with
  `start_agent_home: "space"`, so no conflict is expected.

### Actions

- `src/sase/ace/tui/actions/agent_workflow/_entry_custom.py`:
  - Delete `action_start_agent_home`.
  - In `action_start_agent_from_patch`, replace the empty-MRU warn-and-return with a
    plain `self._show_prompt_input_bar_for_home()` call (decision 1). Update the
    docstring accordingly. Keep the legacy-override indirections untouched.
- `src/sase/ace/tui/actions/artifacts.py` (~line 106): remove `"start_agent_home"` from
  the allowed app-action set.

### Availability gating

- `src/sase/ace/tui/_app_action_availability.py`:
  - Remove `"start_agent_from_patch"` from `_LOCAL_AGENT_ROW_ACTIONS` (it is not a row
    action; removing it drops both the remote-row disable and the prompt-owns-keys
    branch for it).
  - Keep the dedicated guard
    `if action == "start_agent_from_patch" and ... app._prompt_input_active(): return False`
    — it is now the only thing preventing a remount from clobbering an in-progress
    prompt. Rewrite its comment: the action is now bound to printable `space`, which the
    focused TextArea normally swallows (and the vim layer swallows unhandled printable
    NORMAL keys), but the action-level guard still covers rebinding to a non-printable
    key and every focus position inside the bar.
- `src/sase/ace/tui/commands/_availability_agents.py`: remove
  `"app.start_agent_from_patch"` from `_REQUIRES_AGENT` and
  `_REMOTE_AGENT_LOCAL_COMMANDS`.
- `src/sase/ace/tui/commands/_availability_artifacts.py`: replace
  `"app.start_agent_home"` (~line 60, the always-available set) with
  `"app.start_agent_from_patch"`, and remove `"app.start_agent_from_patch"` from the
  "needs a selected Patch row" set (~line 320).
- `src/sase/ace/tui/commands/_app_metadata_actions.py`: delete the `start_agent_home`
  palette entry; change the `start_agent_from_patch` entry to title "Repeat last
  launched VCS xprompt", tab scope `ALL_TABS` (instead of `CL_AGENTS`), and give it the
  search aliases the home entry had plus a few obvious ones, e.g.
  `("home", "repeat", "last vcs xprompt", "run agent")`.
- Grep for any other command-catalog / availability tables that list
  `start_agent_from_patch` as Patches/Agents-only and make them all-tab.

### Help, onboarding, quick-start copy

- `src/sase/ace/tui/modals/help_modal/patches_bindings.py`, `agents_bindings.py`,
  `axe_bindings.py`: delete the `(d(a.start_agent_home), "Run agent (home)")` rows. In
  `agents_bindings.py` "Agent Actions", put the `start_agent_from_patch` "Repeat last
  launched VCS xprompt" row where the home row was (if it is only listed under "General"
  there, leave one listing, not two). Keep the existing "Repeat last launched VCS
  xprompt" rows elsewhere.
- `src/sase/ace/tui/widgets/tab_quickstart.py` (~line 217): use
  `app.start_agent_from_patch`; reword to e.g. "Launch an agent: repeats your last VCS
  xprompt, or opens a blank home-workspace prompt the first time."
- `src/sase/ace/tui/widgets/agent_onboarding.py` (~line 243): use
  `app.start_agent_from_patch`; reword similarly ("open the prompt bar (pre-filled with
  your last VCS xprompt, if any; `Ctrl+U` clears it) and describe a task ...").
- `src/sase/ace/tui/widgets/vim_text_area.py` (~line 131 docstring): replace
  `start_agent_home` with `start_agent_from_patch`.

### Docs

- `docs/ace.md` "Workflows and Agents" table (~line 1085): replace the `Space` and
  `Ctrl+Space` rows with a single row:
  `` `Space` | Prefill the prompt with the most recently launched VCS xprompt (blank home prompt if none; `Space` then `Ctrl+U` for a blank prompt) ``.
  Grep `docs/` (including `docs/llms.md` "Home Mode" section ~line 2951) for other
  mentions of Space launching a home-mode agent or of Ctrl+Space and update them.

### Tests

Update, don't just delete, coverage:

- `tests/test_keymaps_registry_loading.py`: default `start_agent_from_patch == "space"`;
  the `ctrl+space` user-config canonicalization test stays (it sets the key explicitly).
  Fix the `agent_from_cl != "ctrl+@"` assertion if its meaning changes.
- `tests/test_keymaps_app_bindings.py`: rename/adjust
  `test_build_app_bindings_uses_ctrl_space_agent_binding` → asserts `space` maps to
  `start_agent_from_patch` and no binding targets `start_agent_home`.
- `tests/test_command_catalog.py`: `test_start_agent_from_patch_command_uses_ctrl_space`
  → key sequence `("space",)`; replace `test_start_agent_home_command_uses_bare_space`
  with an assertion that `app.start_agent_home` is absent from the catalog.
- `tests/ace/tui/test_show_agent_run_log_keymap.py`:
  `by_key["space"] == "start_agent_from_patch"`.
- `tests/test_keymaps_e2e.py` (~line 212): press `space` and record
  `action_start_agent_from_patch`; drop the `action_start_agent_home` monkeypatch.
- `tests/test_keymaps_display_help_key_display.py`: keep the `ctrl+@` display tests
  (display helper still supports it); update
  `test_help_modal_displays_ctrl_space_agent_shortcuts` to expect `Space` and no "Run
  agent (home)" row.
- `tests/ace/tui/widgets/test_tab_quickstart.py`: switch overrides from
  `start_agent_home` to `start_agent_from_patch`.
- `tests/ace/tui/test_artifacts_scaffold.py`,
  `tests/ace/tui/widgets/test_vim_normal_key_containment.py`,
  `tests/ace/tui/test_prompt_input_collection_launch.py`,
  `tests/ace/tui/test_prompt_bar_stack_submit_handlers.py`: press `space` instead of
  `ctrl+@` where the test is about the default binding; keep the mid-prompt gating tests
  meaningful (a pressed `space` inside a focused prompt must insert/remain in the prompt
  and must not remount it; the action must report unavailable while the prompt is
  mounted). Rename `ctrl_space` identifiers to `space`/`repeat_vcs` where it clarifies.
  If `tests/reproducible_flake_baseline.txt` lists a renamed test id, update the entry.
- `tests/ace/tui/test_entry_points_vcs_prefix_selection.py` /
  `test_entry_points_vcs_prefix_prompt_history.py`:
  `test_ctrl_space_warns_when_mru_empty` → assert the empty-MRU path mounts a blank home
  prompt bar (no initial text) and emits no warning. Keep the `ctrl+g` empty-MRU warning
  test as is (update its docstring reference to `<ctrl+space>`).
- Add a test that `start_agent_from_patch` is available on the Agents tab with no agents
  / no selection and on a remote-agent row, and in the command palette on the Patches
  tab with no Patch row selected.
- Add a test that `app.start_agent_home` / `start_agent_home` in a user keymap config is
  ignored with the unknown-action warning (no crash).
- Update any visual/PNG snapshot or golden help-modal text that shows "Run agent (home)"
  or `Ctrl+Space`.

Finish with a repo-wide grep for `start_agent_home`, `Run agent (home)`,
`Run Agent (Home)`, `ctrl+@`, `Ctrl+Space`, and `ctrl_space` to catch stragglers (the
remaining hits should only be the key-alias/display-helper code and their tests).

## Verification

- Run `just check` (per the repo's lint/test memory note; do not run `check-full`).
- Manually (or via the TUI test harness): on each tab, `space` pre-fills the last VCS
  xprompt; with an empty MRU it opens a blank home prompt; `space` then `ctrl+u` yields
  a blank prompt; `ctrl+space` does nothing by default; typing a space inside the prompt
  bar inserts a space and does not remount the bar.
