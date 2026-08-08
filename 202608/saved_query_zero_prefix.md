---
tier: tale
title: Restore direct saved-PR-query slot keys behind a "0" prefix
goal:
  Pressing `0` followed by a digit on the Artifacts tab loads that saved PR query slot
  directly (`02` -> slot 2, `00` -> slot 0), without disturbing the numbered Artifacts
  sub-tab keys or the `*` saved-query chooser.
proposed_by: bbugyi200.athena.vf
create_time: 2026-08-07 21:35:17
status: wip
---

# Plan: Restore direct saved-PR-query slot keys behind a `0` prefix

## Background

Saved PR/ChangeSpec queries used to be loadable with bare digits: `2` loaded slot 2.
Commit `835929471` ("feat(ace): add numbered artifacts and saved query picker") took
those digits away and gave them to the Artifacts sub-tabs:

- `src/sase/ace/tui/keymaps/bindings.py` and `src/sase/ace/tui/bindings.py` now build
  `Binding(str(index), f"show_artifacts_{subtab}", ...)` for
  `enumerate(ARTIFACTS_SUBTAB_ORDER, start=1)` — today that is `1`=Commits, `2`=Beads,
  `3`=Bugs, `4`=PRs, `5`=Files. These are deliberately non-configurable.
- The 10 `action_load_saved_query_<d>` methods still exist in
  `src/sase/ace/tui/actions/changespec/_query.py`, but the only remaining key path to
  them is `*` (`ace.keymaps.app.open_saved_query_picker`), which pushes
  `SavedQueryPickerModal`; that modal binds digits to `select_slot`.
- `iter_saved_query_commands()` in `src/sase/ace/tui/commands/catalog.py` therefore
  renders the palette rows for those slots as `<picker-key><digit>` (`*1` .. `*0`).
- `docs/ace.md` records the regression explicitly: "Bare digits no longer load saved
  queries".

The user wants the fast, direct numeric path back, namespaced under a `0` prefix: `02`
loads saved query slot 2, `00` loads slot 0. The `*` chooser stays exactly as it is.

## Design decisions

**1. A new configurable app keymap `start_saved_query_mode`, default `"0"`, arms a
one-shot digit selection.** This mirrors two existing, proven patterns in the codebase:

- `action_start_checkout_mode` / `_checkout_mode_active` / `_handle_checkout_key` in
  `src/sase/ace/tui/actions/workspace.py` — a prefix key that arms non-configurable
  digit sub-keys.
- The Admin Center Statistics pane's `select_view` keymap, which already defaults to `0`
  and "arms `1`-`7` selection for a numbered Statistics view"
  (`src/sase/ace/tui/modals/statistics_pane.py`, documented in `docs/configuration.md`).
  Using `0` as an arming prefix for numbered selection is already house style.

The digit sub-keys stay non-configurable (like the checkout mode's `1`-`9` and the
Artifacts sub-tab digits) because they are the slot identifiers themselves.

**2. Do NOT reuse the `open_saved_query_picker` key as the prefix.** Repointing
`open_saved_query_picker` from `asterisk` to `0` would technically make `02` work
(prefix opens the modal, `2` selects the slot), but it would delete the `*` chooser
binding the user already has and route every slot load through a pushed modal screen.
Keep `*` = chooser, add `0<digit>` = direct load. The two are complementary and the
change is purely additive.

**3. The prefix is available on the Artifacts tab (any sub-tab), and nowhere else.**

- Artifacts is where `0` is unclaimed: sub-tab digits start at `1`, so there is no
  collision now, and none until `ARTIFACTS_SUBTAB_ORDER` reaches ten entries.
- It must NOT be armed on the Agents tab. `_handle_member_jump_key` in
  `src/sase/ace/tui/actions/navigation/_member_jump.py` consumes `0`-`9` (including a
  two-digit pending form) for numbered tribe/clan/family member jumps. Scoping the
  prefix to Artifacts keeps member jump untouched.
- Allowing it from any Artifacts sub-tab (not just PRs) is deliberate and free:
  `_load_saved_query()` already switches to the Artifacts PRs sub-tab via
  `switch_to_artifacts_subtab(self, "prs")` when it is called from elsewhere. So `0` `3`
  from Commits lands on PRs with slot 3 loaded. This is intentionally looser than
  `action_open_saved_query_picker`, which stays PRs-only; the chooser is a browsing
  affordance, the digit keys are a jump.

**4. Key precedence is already correct — no binding-priority work needed.**
`EventKeyboardMixin.on_key` (`src/sase/ace/tui/actions/_event_keyboard.py`) runs before
Textual resolves `BINDINGS`. Proof in-tree: `fold_mode.keys.set_level_1` is `"1"` and
`z1` works today even though `1` is bound to `show_artifacts_commits`. So while
`_saved_query_mode_active` is set, the mixin branch consumes the digit and the sub-tab
binding never fires.

## Implementation

### Keymap plumbing

1. `src/sase/ace/tui/keymaps/app_keymaps.py` — add `start_saved_query_mode: str` to
   `AppKeymaps`, in the `# Queries` block immediately after `open_saved_query_picker`.

2. `src/sase/ace/tui/keymaps/metadata.py` — add
   `("start_saved_query_mode", "Saved Query Slots", False)` to `_BINDING_META`, right
   after the `open_saved_query_picker` entry so binding order stays grouped.

3. `src/sase/default_config.yml` — under `# Queries` in `ace.keymaps.app`, add
   `start_saved_query_mode: "0"` after `open_saved_query_picker`.

4. `src/sase/ace/tui/bindings.py` — add the matching fallback entry to
   `DEFAULT_BINDINGS` next to the `asterisk` binding:
   `Binding("0", "start_saved_query_mode", "Saved Query Slots", show=False)`.

Confirm the loader's duplicate-binding validation is happy: no other `ace.keymaps.app`
action currently uses `0`.

### Mode state and dispatch

5. `src/sase/ace/tui/actions/_state_init.py` — initialize
   `self._saved_query_mode_active: bool = False` beside `_checkout_mode_active`.

6. `src/sase/ace/tui/actions/_event_base.py` — declare `_saved_query_mode_active: bool`
   with the other mode-flag type hints.

7. `src/sase/ace/tui/actions/_event_keyboard.py` — insert a branch into the `on_key`
   chain directly after the `_checkout_mode_active` branch (it must precede the trailing
   `member_jump_handler(key)` fallback):

   ```python
   elif self._saved_query_mode_active:
       if self._handle_saved_query_key(event.key):
           event.prevent_default()
           event.stop()
   ```

8. `src/sase/ace/tui/actions/changespec/_query.py` — add to `ChangeSpecQueryMixin`, next
   to `action_open_saved_query_picker`:
   - `action_start_saved_query_mode()`: return early unless
     `self.current_tab == ARTIFACTS_TAB` (the mixin's `current_tab` literal for the
     Artifacts tab is `"changespecs"`; use the same `ARTIFACTS_TAB` constant the
     availability module uses, or the existing `"changespecs"` comparison style already
     used in this file — match whichever the surrounding code uses). Set
     `self._saved_query_mode_active = True` and refresh the footer hint (step 10).
   - `_handle_saved_query_key(key: str) -> bool`: always clear
     `_saved_query_mode_active` first (one-shot, like `_handle_bang_key`); on `"escape"`
     restore the footer and return `True`; when `key` is a single decimal digit call
     `self._load_saved_query(key)` then restore the footer and return `True`; otherwise
     restore the footer and return `True` (consume the key, matching
     `_handle_checkout_key`'s "invalid key just exits mode" behaviour).

   `_load_saved_query` already notifies `"No query saved in slot N"` for empty slots, so
   no extra empty-slot handling is needed. Textual delivers digits as `"0"`..`"9"`,
   which matches `KEY_ORDER` in `src/sase/ace/saved_queries.py` directly.

9. `src/sase/ace/tui/_app_action_availability.py` — gate the new action alongside the
   existing `show_artifacts_*` block:

   ```python
   if action == "start_saved_query_mode" and app.current_tab != ARTIFACTS_TAB:
       return False
   ```

### Footer hint

10. `src/sase/ace/tui/widgets/_keybinding_modes.py` — add
    `update_saved_query_bindings()` modelled on `update_bang_bindings()`: display label
    `"QUERY"` and rows for the populated slots read from the app's `_saved_queries`
    cache (fall back to an empty/`no saved queries` row when the cache is empty or
    absent). Call it from `action_start_saved_query_mode`; on exit,
    `_handle_saved_query_key` calls `self._refresh_current_tab()` to restore the normal
    footer, exactly as `_handle_bang_key` does. Guard the footer lookup with the same
    `try`/`except` shape the other mode-footer helpers use so a missing footer never
    breaks the key path.

### Discovery surfaces

11. `src/sase/ace/tui/commands/_app_metadata.py` — add an `APP_COMMAND_META` entry for
    `start_saved_query_mode` (label e.g. `"Load saved PR query by slot"`, category
    `"Saved Queries"`, tabs `CL_ONLY`, aliases such as
    `("saved query slot", "query slot")`). This is mandatory:
    `ensure_metadata_covers_app_keymaps()` raises `RuntimeError` at import time when an
    `AppKeymaps` field has no metadata.

12. `src/sase/ace/tui/commands/catalog.py` — in `iter_saved_query_commands`, take the
    prefix from `registry.app.start_saved_query_mode` instead of
    `registry.app.open_saved_query_picker`, so palette rows read `01`..`09`, `00` and
    now describe a real key sequence. Update the module docstring bullet ("The 10
    saved-query picker sequences (configured prefix + `0`..`9`)") to say the sequences
    are the saved-query prefix plus a slot digit. Leave `iter_digit_commands`,
    `CommandExecutor(kind="saved_query")`, and `execute_command`'s `saved_query` branch
    untouched.

13. `src/sase/ace/tui/modals/help_modal/modal.py` — pass
    `saved_query_prefix=key_display_name(km.app.start_saved_query_mode)` to
    `add_saved_queries_section`.

14. `src/sase/ace/tui/modals/help_modal/query_sections.py` — change the
    `saved_query_prefix` default from `"*"` to `"0"` and update the docstring.

15. `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py` — in the `Queries`
    section, keep the chooser row and add a slot row above it, e.g.:

    ```python
    (
        f"{d(a.start_saved_query_mode)}1-9 / {d(a.start_saved_query_mode)}0",
        "Load saved PR query slot",
    ),
    (d(a.open_saved_query_picker), "Choose saved PR query"),
    ```

    (The existing chooser row currently renders as `*1-9 / *0`; it should become the
    bare chooser key now that the digits belong to the new prefix.)

16. `src/sase/ace/tui/widgets/changespec_detail.py` — the active-slot badge builds its
    prefix from `registry.app.open_saved_query_picker` with a `"*"` fallback. Switch it
    to `registry.app.start_saved_query_mode` with a `"0"` fallback so the badge reads
    `[03]` and tells the user the key that loads it. Update the adjacent comment.

### Documentation

17. `docs/ace.md`, "Saved Queries" section (~line 2164): rewrite. State that on the
    Artifacts tab `0` followed by a slot digit loads that saved PR query directly
    (`01`-`09`, `00`), that it works from any Artifacts sub-tab and lands on PRs, that
    `Esc` or any non-digit cancels, and that `*` still opens the chooser (PRs sub-tab
    only). Delete the now-false sentence "Bare digits no longer load saved queries" and
    replace it with the accurate statement that bare digits `1`-`5` select Artifacts
    sub-tabs and the slot keys live behind `0`.

18. `docs/configuration.md`, `ace.keymaps` section: add a short note that
    `ace.keymaps.app.start_saved_query_mode` (default `0`) arms saved-PR-query slot
    selection, that its digit sub-keys are not configurable, and that it is scoped to
    the Artifacts tab. Put it near the existing `edit_query` scoping paragraph.

Grep for any other prose that claims bare digits load saved queries or that `*<digit>`
is the slot key sequence and fix it:
`rg -n 'saved quer' docs/ src/sase --glob '!**/__pycache__/**'`.

## Tests

19. `tests/test_keymaps_defaults.py` — extend the saved-query default test (or add a
    sibling) asserting `reg.app.start_saved_query_mode == "0"` and that
    `open_saved_query_picker` is still `"asterisk"`.

20. `tests/test_keymaps_app_bindings.py` — assert both the registry-built bindings and
    `DEFAULT_BINDINGS` contain `start_saved_query_mode` bound to `"0"`. The two existing
    assertions that no binding action starts with `load_saved_query` remain true and
    must stay: the slot digits are handled in `on_key`, not as bindings.

21. `tests/test_command_catalog.py` — rename/retarget
    `test_saved_query_commands_follow_configured_picker_prefix` to drive
    `{"keymaps": {"app": {"start_saved_query_mode": "P"}}}`, and add a default-registry
    assertion that the ten `saved_query.*` specs have `key_display` `01`..`09`, `00`.

22. `tests/test_command_availability_changespecs.py` — cover the new
    `app.start_saved_query_mode` command's tab scoping next to the existing
    `test_saved_query_picker_and_slots_are_pr_only` case.

23. New `tests/ace/tui/test_saved_query_slot_keys.py` — app-level behaviour, using the
    existing ACE TUI test harness (see `tests/ace/tui/test_agents_jump_to_prs_subtab.py`
    and `tests/ace/tui/test_changespecs_onboarding.py` for the harness/fixture idiom,
    and monkeypatch `sase.ace.saved_queries` file paths or seed `app._saved_queries`
    directly the way the picker tests do):
    - From the Artifacts PRs sub-tab with slot 2 populated, `0` then `2` loads slot 2's
      query.
    - From the Artifacts Commits sub-tab, `0` then `3` loads slot 3 **and** switches
      `current_artifacts_subtab` to `"prs"`.
    - `0` then `0` loads slot 0.
    - `0` then a digit for an empty slot leaves the query unchanged (warning path).
    - `2` on its own still switches to the Beads sub-tab — i.e. the prefix does not leak
      and the mode flag is cleared after one key.
    - `0` then `escape` cancels: mode flag cleared, query unchanged.
    - On the Agents tab, `0` does not arm the mode (`_saved_query_mode_active` stays
      `False`), so member-jump digit handling is unaffected.

24. `tests/ace/tui/modals/test_help_modal.py` — update the expectations for the saved
    queries section badge prefix and the new/changed `Queries` binding rows.

25. Visual snapshots:
    `tests/ace/tui/visual/test_ace_png_snapshots_saved_query_picker.py` renders
    `SavedQueryPickerModal`, which does not use the prefix, so it should be unaffected.
    However, the changespec detail `[*N]` badge (step 16) and any help-modal golden may
    appear in PNG goldens. Run `just test-visual`; if and only if a golden legitimately
    shifts because of the badge/help text, re-record it with
    `--sase-update-visual-snapshots` and say so in the commit body.

## Risks and edge cases

- **Member jump.** The only real collision risk. Mitigated by the Artifacts-only
  availability gate (step 9) plus the early-return in `action_start_saved_query_mode`.
  Test 23's last case locks this in.
- **Modal screens.** `on_key` on the app still fires while a modal is up in some paths;
  `_handle_saved_query_key` is a no-op unless the mode was armed, and the mode can only
  be armed from the Artifacts tab, so no extra modal guard is required. If review finds
  the flag can survive a modal push, clear it wherever the other mode flags are cleared.
- **`ARTIFACTS_SUBTAB_ORDER` growth.** A tenth sub-tab would produce a `"10"` binding
  and never a `"0"` one, so the prefix stays collision-free; no action needed, but a
  comment near `_ARTIFACT_SUBTAB_BINDINGS` noting `0` is reserved for saved-query slots
  is cheap insurance.
- **User rebinding.** A user who sets `start_saved_query_mode` to a key already used in
  `ace.keymaps.app` hits the loader's existing duplicate-binding validation; nothing new
  to build.

## Verification

```bash
just install
just check
just test-visual   # only if step 25 touches goldens
```

Run `just check-full` before landing, since this touches keymaps, bindings, the command
catalog, help surfaces, and docs.

Manual smoke: launch `sase ace`, go to Artifacts, press `0` (footer shows the QUERY hint
and populated slots), press `2` (slot 2 loads, PRs sub-tab active, detail badge reads
`[02]`), press `2` again (Beads sub-tab), press `*` (chooser still opens on PRs).
