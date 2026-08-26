---
tier: tale
title: Scope Admin Center tab memory to one TUI session
goal:
  The Admin Center remembers the last-visited section only for the current ACE process,
  a fresh process starts with no remembered section, and the Config tab opens on its
  first sub-tab instead of always opening XPrompts.
size: medium
proposed_by: bbugyi200.athena.0e8
---

- **AGENTS:**
  - [bbugyi200.athena.0e8](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0e8.md)
- **COMMITS:**
  - [a3b69bd](https://github.com/sase-org/sase/commit/a3b69bd85bdbb83c24a457aad126e0891df233f7)
    — feat(tui): make Admin Center tab memory session-scoped

# Plan: Scope Admin Center tab memory to one TUI session

## Problem

Two separate defects make ACE look like it "restores where I left off last time", which
is not what the user wants:

1. **The Admin Center's last-visited section survives across ACE processes.**
   `src/sase/ace/tui/modals/config_center_state.py` writes the `(current, alternate)`
   tab pair to `~/.sase/ace_admin_center_last_tab.txt` and reloads it at startup. So on
   the very first `#` press of a brand-new ACE process, the landing page already
   advertises "resume Procs" (or whatever section was open days ago), and a repeated `#`
   jumps straight into it.

2. **The Config tab always opens on XPrompts.** `ConfigHubSessionState.active_subtab`
   defaults to `"xprompts"` (`src/sase/ace/tui/modals/config_hub_session.py:72`), and
   `ConfigHubPane` repeats that literal as its fallback
   (`src/sase/ace/tui/modals/config_hub_pane.py:124`). XPrompts is sixth in
   `CONFIG_SUBTAB_ORDER`, whose first entry is `misc` (labelled **All**). This is a
   stale default, not persistence: commit `1382a43d8` nested the old top-level XPrompts
   section under Config and picked `"xprompts"` to preserve continuity with the section
   it replaced. That continuity no longer matters.

The two defects are independent — fixing (1) does not change (2) — but they produce the
same complaint, so this plan fixes both.

## Goal

- Within one ACE process, navigating to an Admin Center section and reopening the panel
  still resumes that section, and the two-slot alternate jump still works.
- A brand-new ACE process starts with **no** remembered section: the landing page shows
  its existing no-history copy ("`#` resumes after your first section visit") and a
  repeated `#` is inert until a section is visited.
- Opening Config without an explicit target lands on the first sub-tab (**All** /
  `misc`) in a fresh process, and lands on the last Config sub-tab visited in this
  process otherwise.
- Nothing in ACE writes tab or sub-tab position to disk any more.

## Non-goals (deliberately excluded)

Read this list before widening scope; each item was checked and left alone.

- **All other Admin Center sub-tabs are already session-scoped and already default to
  their first entry.** `ProjectsSessionState.active_subtab` defaults to `"projects"`
  (first of `projects, repos, workspaces`) and `UpdatesSessionState.active_subtab`
  defaults to `"core"` (first of `core, plugins, agent-clis`). `AdminCenterSessionState`
  is constructed once per app in `init_runtime_state`, so it dies with the process. No
  change needed.
- **The Artifacts sub-tab default stays `"stitches"`.** `_artifact_tab_model.py` sets
  `DEFAULT_ARTIFACTS_SUBTAB = "stitches"` while `FIXED_ARTIFACTS_SUBTAB_ORDER` starts
  with `"agents"`, so it is also "not the first tab" — but it is a reactive on `AceApp`,
  so it is already session-scoped (the actual complaint), the visible order is
  plugin-extensible via `resolve_artifacts_subtabs()`, and `_app_watchers.py` has
  `patches`-specific behavior keyed off it. Changing it is a separate product decision;
  flag it to the user rather than folding it in here.
- **The top-level ACE tab is not persisted.** It comes from `args.tab` through
  `ace_handler.py` into `AceApp(initial_tab=...)`.
- **Non-tab durable state stays as it is**: `~/.sase/grouping_mode.txt` (Agents-tab
  grouping), `ace_agents_fold_state.json` (group folds), `last_selection.txt`,
  `last_query.txt`, `query_selections.json`. The user asked about tabs/sub-tabs, and
  these are deliberately durable preferences.
- **No feature flag.** Per `sase/memory/sase_flags.md`, a flag is for behavior that is
  not ready or for a deprecated branch that callers must migrate off. This is a
  user-requested correction with no branch worth keeping reachable.
- **Do not delete the user's existing state files.**
  `~/.sase/ace_admin_center_last_tab.txt` simply stops being read; a stale
  `~/.sase/admin_center_tab.json` from an older implementation is already orphaned.
  Leave both on disk.
- **Do not edit `tests/reproducible_flake_baseline.txt`.** Its `fixed-at:` entry for
  `test_config_center_state.py::test_save_atomically_replaces_existing_state` is a
  historical evidence-retirement record with attached provenance prose; it becomes inert
  when the node disappears, and rewriting it would damage that narrative. Verify with
  `just selection-health` that it stays green.

## Design decisions

- **Delete the persistence layer rather than gating it.** The in-memory
  `AdminCenterTabHistory` (`config_center_history.py`) already models everything the UI
  needs — `current`, `alternate`, and `remember()`. Keeping a write-to-nowhere code path
  would leave the async writer, generation counters, and exit flush as dead weight.
  Delete the module, its test file, and the writer machinery, and keep the pure history
  dataclass untouched.
- **The `AdminCenterPersistenceMixin` collapses to a small in-memory recorder.**
  `_remember_admin_center_tab` and `_on_admin_center_tab_activated` stay, because
  `ConfigCenterModal` calls back on every successful activation and
  `_open_config_center` reads `_last_admin_center_tab` / `_admin_center_history`.
  Everything below them (`_save_admin_center_tab_now`, `_start_admin_center_tab_writer`,
  `_run_admin_center_tab_save_loop`, `_flush_admin_center_tab_state`, the six
  `_admin_center_tab_*` bookkeeping fields) goes away. Keep
  `_ensure_admin_center_persistence_state()` — direct-mixin tests rely on it — reduced
  to seeding the two surviving attributes. Consider renaming the module/mixin
  (`_admin_center_tab_memory.py` / `AdminCenterTabMemoryMixin`) since "persistence" is
  no longer accurate; if renamed, update the import in `actions/base.py:17,28`.
- **Derive the Config default from the catalog instead of hardcoding a second literal.**
  Replace the `"xprompts"` default with the first entry of the active order.
  `config_subtab_order()` reads the pinned feature-flag snapshot and must not run at
  import time, so it cannot be a plain dataclass field default. Use
  `field(default=None)` plus a `subtab()` accessor, or a `default_factory` that is safe
  at construction time — `AdminCenterSessionState()` is built in `init_runtime_state`,
  after flags are pinned, so a `default_factory` calling `config_subtab_order()[0]` is
  acceptable; verify no module-import-time construction exists before choosing it.
  Whatever shape is chosen, the literal `"xprompts"` fallback in
  `config_hub_pane.py:124` becomes `self._subtab_order[0]`.
- **This is a net startup win, not a risk.** `init_runtime_state` currently does a
  synchronous `open()`/`read()` before the event loop starts; after this change it does
  none. That is consistent with rule 9 in `sase/memory/tui_perf.md`.

## Implementation

### Part 1 — make Admin Center tab memory session-only

1. Delete `src/sase/ace/tui/modals/config_center_state.py`. Leave
   `config_center_history.py` exactly as it is.
2. Rewrite `src/sase/ace/tui/actions/_admin_center_persistence.py` down to the in-memory
   recorder described above. Drop the `asyncio`, `logging`, and `spawn_pump_free_task`
   imports that become unused. Keep the `_last_admin_center_tab` /
   `_admin_center_history` pair in sync, since `_open_config_center` reads both and a
   PNG snapshot test assigns `_last_admin_center_tab` directly.
3. In `src/sase/ace/tui/actions/_state_init_runtime.py` (lines ~52–68): drop the
   `load_admin_center_tab_history` import, initialize
   `self._admin_center_history = AdminCenterTabHistory()`, set
   `self._last_admin_center_tab = None`, and delete the six `_admin_center_tab_*` writer
   fields. Update the comment block above it, which currently claims resume comes from
   "this or a previous ACE process".
4. In `src/sase/ace/tui/actions/lifecycle.py`: remove `_flush_admin_center_tab_state`
   from the `_flush_then_do_quit` gather list (~line 224) and from the capability tuple
   in `_request_controlled_exit` (~line 254). `_flush_agents_fold_state` stays and keeps
   that path alive.
5. In `src/sase/ace/tui/modals/config_center_catalog.py`, update the
   `validated_center_tab` docstring: the `xprompts` → `config` mapping is no longer
   about "persisted resume state". Keep the mapping itself as a cheap guard for stray
   callers, but stop describing it as reaching XPrompts "through the hub's default
   child", which stops being true in Part 2.

### Part 2 — Config opens on its first sub-tab

6. In `src/sase/ace/tui/modals/config_hub_session.py`, replace the
   `active_subtab: ConfigSubTab = "xprompts"` default with a catalog-derived first
   entry, per the design decision above.
7. In `src/sase/ace/tui/modals/config_hub_pane.py:124`, replace the trailing
   `or "xprompts"` with `or self._subtab_order[0]`.
8. Confirm no production caller depends on the old default. Verified while planning: the
   only `ConfigHubEntry(...)` constructions are `snippets`, `memory` (×2), and `launch`;
   nothing passes `subtab="xprompts"`, and no keymap opens XPrompts directly. XPrompts
   remains reachable by number key inside Config.

### Part 3 — tests

9. Delete `tests/ace/tui/test_config_center_state.py` (the whole file tests the deleted
   module).
10. `tests/ace/tui/test_config_center_resume.py`: delete the three disk-coupled tests —
    `test_persisted_tab_seeds_home_and_repeated_opener_resume`,
    `test_invalid_persisted_tab_keeps_no_history_home`,
    `test_blocked_write_keeps_navigation_responsive_and_persists_latest`,
    `test_write_failure_is_nonfatal_and_same_tab_can_retry` — and drop the
    `config_center_state` imports. Keep every in-process test
    (`..._home_first_then_repeated_opener_resumes`,
    `..._switching_changes_resume_target...`, `..._direct_entry_establishes...`, the
    opener-binding tests). **Add** one test that is the actual acceptance criterion:
    after visiting a section and quitting, a _new_ `AcePage` opens the Admin Center with
    `_last_admin_center_tab is None`, the landing shows the no-history hint, and a
    repeated `#` mounts no working pane.
11. `tests/ace/tui/test_config_center_alternate_tab.py`: rewrite
    `test_persisted_file_contains_both_lines_in_order` as an in-memory assertion on
    `app._admin_center_history` (or delete it if
    `tests/ace/tui/test_config_center_history.py` already covers pair ordering), remove
    the `config_center_state` import and the `_admin_center_tab_completed_generation`
    wait at line ~263. `test_seeded_alternate_survives_close_and_reopen` stays — it is
    same-process.
12. `tests/ace/tui/actions/test_lifecycle_quit_confirm.py`: remove the two
    `_flush_admin_center_tab_state` stubs (lines ~39 and ~151) and rework
    `test_controlled_exit_waits_for_admin_center_flush` onto `_flush_agents_fold_state`,
    or delete it if the fold-flush case is already covered.
13. Re-seed the Config sub-tab explicitly wherever a test relied on the old default.
    Prefer
    `ConfigCenterModal(initial_tab="config", config_entry=ConfigHubEntry(subtab="xprompts"))`
    or a pre-set `session_state.config_hub.active_subtab`, mirroring the existing
    `_switch_to("misc")` idiom:
    - `tests/ace/tui/test_config_hub_pane.py` —
      `test_opening_config_constructs_only_the_active_child` and the
      `calls == ["xprompts", ...]` expectations.
    - `tests/ace/tui/test_config_hub_pane_navigation.py` — many sites; the bracket-cycle
      and number-prefix expectations shift with the new landing sub-tab, so recompute
      them from `CONFIG_SUBTAB_ORDER` rather than editing literals one by one.
    - `tests/ace/tui/test_config_hub_pane_launch_flags.py` (~lines 189–217).
    - `tests/ace/tui/test_xprompt_browser_filter.py:182`,
      `tests/ace/tui/test_xprompt_browser_jump.py:227`,
      `tests/ace/tui/test_xprompt_browser_load_keymap.py:87,334` — these open
      `ConfigCenterModal(initial_tab="config")` and expect the XPrompts pane; seed the
      sub-tab instead.
    - `tests/ace/tui/test_admin_center_selection_resume.py` — the
      `_ResumeCase("xprompts", "1", move_key="ctrl+n")` case needs an explicit
      `_switch_to("xprompts")`, exactly mirroring the `"config"` case's
      `_switch_to("misc")` at line ~309 (and the same wait on reopen at ~337). The
      existing `"config"` case's explicit switch becomes a harmless no-op.
14. **Add** a test that Config lands on `CONFIG_SUBTAB_ORDER[0]` with a fresh
    `AdminCenterSessionState`, and a test that a Config sub-tab visited earlier in the
    same process is what a later open lands on.

### Part 4 — sweep

15. Grep for stragglers before finishing:
    `rg 'config_center_state|_admin_center_tab_(durable|queued|save|completed)|_flush_admin_center_tab_state|_save_admin_center_tab_now'`
    over `src` and `tests` must come back empty.

## Verification

- `just install` first if this workspace is cold, then `just check`.
- `just check-full` through `/sase_monitor` before landing: this touches the Admin
  Center, the Config hub, and app lifecycle, so scoped selection is not a sufficient
  gate.
- `just test-visual` for
  `tests/ace/tui/visual/test_ace_png_snapshots_config_center_home.py`. No rebaseline is
  expected: that test assigns `page.app._last_admin_center_tab` explicitly for both the
  `None` and `"procs"` parameterizations, and tests are already isolated from the real
  `~/.sase` by the `_isolate_sase_home` fixture, so the goldens never depended on real
  persisted state. If a golden does move, stop and report it rather than accepting it
  blindly.
- `just selection-health` stays green (see the flake-baseline non-goal).
- Optional but recommended:
  `pytest -s -m slow tests/ace/tui/bench_admin_center_open.py`. Its assertions are
  structural, not wall-clock, but the Config hub now mounts **All** (`ConfigPane`, which
  builds a config inventory) instead of XPrompts on first entry. If first-paint cost
  after `#`→`1` regresses noticeably, say so in the handoff — do not silently revert the
  default.
- Manual smoke: start `sase ace`, press `#` — the landing hint must read "resumes after
  your first section visit", and a second `#` must do nothing. Enter Config: it must
  show **All**. Move to XPrompts, close, reopen — Config must return to XPrompts. Quit
  ACE, restart, press `#`,`1` — Config must be back on **All**, and no new mtime on
  `~/.sase/ace_admin_center_last_tab.txt`.

## Risks

- **Test churn is the bulk of this work**, concentrated in Part 3 step 13. Nine test
  files reach the XPrompts pane implicitly. Seed the sub-tab explicitly rather than
  reordering the catalog, which would ripple into number keys, the tab strip, and the
  numbered-link tests.
- **`config_subtab_order()` must not run at import time** (its own docstring says so).
  If a `default_factory` turns out to be reachable before flags are pinned, fall back to
  a `None` sentinel resolved in `ConfigHubPane.__init__`.
- **Losing the exit-flush path entirely** would be wrong: `_flush_then_do_quit` still
  has to await `_flush_agents_fold_state`. Only the Admin Center entry is removed from
  it.
