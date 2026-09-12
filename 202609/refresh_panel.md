---
tier: epic
title: Unified ACE Refresh panel on R
goal: "One `R` gesture opens a Refresh panel that replaces both the immediate tab
  refresh and the `,y` full-history refresh, adds a provider usage-window refresh and an
  everything sweep, shows each option's real freshness, and keeps the old gestures
  reachable behind a sunset flag until the panel has soaked.

  "
phases:
  - id: freshness
    title: Surface freshness recorder
    depends_on: []
    size: small
    description: 'freshness: add the in-memory per-surface "last reloaded" recorder,
      stamp it from the auto-refresh sweep and the manual agents/artifacts/axe refresh
      paths, and expose a pure formatter for relative freshness labels.

      '
  - id: panel
    title: Refresh panel modal
    depends_on:
      - freshness
    size: medium
    description: "panel: build the RefreshPanelModal single-key chooser, its rows,
      cursor, banner, availability states, worker-loaded usage freshness, styles, and
      modal exports.

      "
  - id: wire
    title: Gesture rewiring behind the refresh_panel flag
    depends_on:
      - panel
    size: medium
    description: "wire: create the sunset refresh_panel flag, route R and `,y` through
      the panel when it is on, execute each chosen option, and keep today's direct
      gestures as the off branch.

      "
  - id: finish
    title: Documentation and visual snapshot
    depends_on:
      - wire
    size: small
    description:
      "finish: document the panel in docs/ace.md, correct the stale refresh-key prose,
      add the PNG visual snapshot, and run the full verification gate."
proposed_by: bbugyi200.athena.0kb
create_time: 2026-09-12 14:56:15
status: wip
---

# Plan: Unified ACE Refresh panel on R

## Problem

Refreshing ACE is split across two unrelated gestures with no shared discovery surface:

- `R` (app-level `refresh` action, `src/sase/ace/tui/actions/base.py:484`) immediately
  refreshes whatever the current tab is — Agents (visible-inbox Tier 1), the active
  Artifacts pane, or Axe.
- `,y` (leader `full_history_refresh`,
  `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py:179` →
  `action_refresh_agents_full_history`, `src/sase/ace/tui/actions/base.py:515`)
  force-rescans Agents from full artifact history, and refuses on any other tab.

Nothing refreshes provider subscription usage on demand except the `u` key buried inside
the Providers · Usage modal, and nothing tells the user how stale any surface actually
is before they press a key.

## Outcome

`R` opens a **Refresh panel**: a centered, single-key chooser that names every refresh
ACE can perform, shows how fresh each target already is, and runs exactly one of them.

```
                    ┌─ Refresh ─────────────────────────────────────┐
                    │  auto-refresh every 10s · next in 4s          │
                    │                                               │
                    │  r  This tab · Agents           updated 12s ago│
                    │     Reload the visible inbox from the index.  │
                    │                                               │
                    │  f  Full history · Agents      last scan 2h ago│
                    │     Rescan every source artifact. Slower.     │
                    │                                               │
                    │  u  Usage windows · 3 providers   updated 6m ago│
                    │     Re-probe provider subscription limits.    │
                    │                                               │
                    │  a  Everything                        heavier │
                    │     Every surface, full history, and usage.   │
                    │                                               │
                    │  r tab · f history · u usage · a all · esc    │
                    └───────────────────────────────────────────────┘
```

## Design decisions

These are settled; implement them as written rather than re-deriving them.

### 1. Fixed option keys, not a configurable keymap scope

The panel is a transient chooser that dismisses on one keypress, so it follows the
established single-key-chooser precedent — `QuitOptionsModal`, `JumpActionModal`,
`PromptSubmitChoiceModal` all hardcode their `BINDINGS`. Configurable
`ace.keymaps.<scope>` sections exist for _browsable_ panes that stay open (`memory`,
`snippets`, `statistics`, `projects`, `machines`, `config`, `gate`). Do **not** add a
`refresh` keymap scope, a `RefreshPanelKeymaps` dataclass, or a `sase.schema.json`
keymaps entry. The gesture that opens the panel stays configurable: it remains the
app-level `refresh` action bound to `R` in `src/sase/default_config.yml`.

### 2. `R` keeps the `refresh` action id

`action_refresh` changes what it _does_, not what it is _called_. The `refresh` action
id is referenced by `PaneCapability.REFRESH`
(`src/sase/ace/tui/_artifact_tab_actions.py:47`), the command palette
(`src/sase/ace/tui/commands/_app_metadata_actions.py:102`), the help modal, four
Artifacts pane footers, and the `stitches_refresh` / `plans_refresh` / `beads_refresh` /
`files_refresh` legacy aliases in `src/sase/ace/tui/keymaps/registry.py`. Renaming it
would ripple through all of them for no gain.

### 3. Option keys and aliases

| Key | Aliases           | Option        | Runs                                                      |
| --- | ----------------- | ------------- | --------------------------------------------------------- |
| `r` | `R`, `enter`, `1` | This tab      | today's per-tab `action_refresh` body                     |
| `f` | `2`               | Full history  | today's `action_refresh_agents_full_history` body         |
| `u` | `3`               | Usage windows | `submit_usage_refresh(None, explicit=True, origin="ace")` |
| `a` | `4`               | Everything    | forced sanity sweep + full history + usage                |

`R` is deliberately an alias for the first option, so a double-tapped `R R` reproduces
the old `R` exactly. `enter` activates whichever row the cursor is on, so `j`/`k` +
`enter` is a complete alternative to the letters. `escape` and `q` cancel.

### 4. Full history works from every tab

`action_refresh_agents_full_history` currently refuses when `current_tab != "agents"`.
That guard is policy, not a technical constraint: the agents loading path already
handles off-tab loads explicitly and restores selection through `_agents_last_identity`
(`src/sase/ace/tui/actions/agents/_loading_disk.py:196`, `:392`, `:558`). Drop the guard
so `R f` means the same thing everywhere and the row never needs a disabled state. The
row subtitle names Agents as its target so the scope is never ambiguous.

### 5. An unavailable option toasts and keeps the panel open

Only the usage row can be unavailable (usage metrics disabled in config, or no eligible
providers). Pressing its key then notifies the reason and leaves the panel open rather
than dismissing into a no-op — the user can still pick something else.

### 6. Freshness is measured, never guessed

Every chip comes from a real recorded reload, and a surface that has not been reloaded
in this session renders `—`, not a fabricated age. See the `freshness` phase.

### 7. A sunset feature flag keeps both old gestures reachable

`R` changing from "do the thing" to "open a chooser" is a user-reaching deprecation, and
`sase/memory/sase_flags.md` makes a flag mandatory for a deprecated branch that must
stay reachable while users migrate. The closest precedent is `ref_sync_gesture`, a
`sunset` flag guarding a new keyboard gesture. See the `wire` phase.

## Phase: freshness — Surface freshness recorder

Add a small, self-contained recorder so the panel can state the truth about staleness.

### New module `src/sase/ace/tui/actions/event_refresh/_freshness.py`

- A module-level constant naming the recorded surfaces: `agents`, `agents_full_history`,
  `patches`, `artifacts`, `axe`, `notifications`.
- `note_surface_refreshed(app: Any, surface: str, *, now: float | None = None) -> None`
  — stamp `time.monotonic()` into an `app._surface_refreshed_mono: dict[str, float]`
  created on first use. Unknown surface names are ignored rather than raising; this runs
  on refresh completion paths and must never be able to break one.
- `surface_refreshed_age(app: Any, surface: str) -> float | None` — seconds since the
  last stamp, or `None` when never stamped in this session.
- `freshness_label(age: float | None) -> str` — pure formatter returning `—` for `None`,
  `just now` under 5s, then `12s ago`, `6m ago`, `2h ago`, `3d ago`. No I/O, no clock
  reads; the caller supplies the age.

`_surface_refreshed_mono` is intentionally in-memory only and monotonic-based: it
answers "how stale is what I am looking at right now", which is exactly the question the
panel asks, and it survives no restart because a restart reloads everything anyway.

### Stamping sites

1. `_run_auto_refresh_body` (`.../event_refresh/_auto_refresh.py:159`) already collects
   a `reloaded: list[str]` of surface names from `_run_auto_refresh_surfaces`. Stamp
   each name in that list once, inside the existing `finally:` block that writes the
   trace fields. One loop, no new probing, no new I/O.
2. `action_refresh` (`actions/base.py:484`) — stamp the surface it just scheduled
   (`agents`, `patches` or `artifacts`, or `axe`) at request time.
3. The full-history path — stamp `agents_full_history` from
   `action_refresh_agents_full_history`.

Stamping at request time in (2) and (3) is deliberate: the panel reports "you asked for
this that long ago", which is what a user reasoning about whether to press again needs,
and it cannot drift if a scheduled load is coalesced away.

### Verification

- New `tests/ace/tui/test_refresh_freshness.py`: `freshness_label` boundaries (`None`,
  0s, 4.9s, 5s, 59s, 60s, 3599s, 3600s, 24h), unknown-surface no-op,
  `surface_refreshed_age` before and after a stamp.
- Extend the existing auto-refresh test module that drives `_run_auto_refresh_body` to
  assert reloaded surfaces get stamped and a skipped (unchanged) surface does not.
- Symvision will report the new public functions as unused until `panel` consumes them.
  Add `--epic-symbol <this epic's bead id>(note_surface_refreshed)` and the same for
  `surface_refreshed_age` and `freshness_label` to the symvision invocation in the
  `Justfile`; the `panel` phase removes those entries.
- `just check`.

## Phase: panel — Refresh panel modal

### New module `src/sase/ace/tui/modals/refresh_panel_modal.py`

```python
type RefreshChoice = Literal["this_tab", "full_history", "usage", "everything"]
```

`RefreshPanelModal(ModalScreen[RefreshChoice | None])`, constructed with plain data so
it is unit-testable without an app:

- `tab_label: str` — e.g. `"Agents"`, `"Artifacts › Beads"`, `"Axe"`.
- `rows: tuple[RefreshRow, ...]` where `RefreshRow` is a frozen dataclass of `choice`,
  `key`, `aliases`, `title`, `target`, `subtitle`, `chip`, `tone`,
  `unavailable_reason: str | None`.
- `auto_refresh_label: str | None` — the header line.
- `initial_choice: RefreshChoice` — where the cursor starts, default `"this_tab"`.
- `banner: str | None` — the migration line the `wire` phase passes for `,y`.
- `load_usage_status: Callable[[], UsageRowStatus]` — injected loader, defaulted to the
  real one, so tests supply a stub.

### Layout and styling

Compose the same shape as `JumpActionModal`: a `Container` carrying
`duration-choice-container` with `id="refresh-panel-container"`, a
`duration-choice-title` label, an optional header `Static`, one `Static` per row, a
spacer, and a footer `Static`. Add to `src/sase/ace/tui/styles.tcss`:

- `RefreshPanelModal` in the existing shared `align: center middle` selector list next
  to `QuitOptionsModal`.
- `#refresh-panel-container { width: 72; }`, `#refresh-panel-title { width: 100%; }`.
- `.refresh-panel-header` / `.refresh-panel-footer` using `$text-muted`, matching
  `.jump-action-footer`.
- `.refresh-panel-row-selected` for the cursor row: `background: $boost;` plus the
  existing `duration-choice-row` metrics so the row does not reflow when selected.

Each row renders as two lines, following the `JumpActionModal._render_choice` idiom: a
bold key, a bold title with a dim ` · <target>` suffix, the freshness chip padded to the
right edge of the 72-cell container, and a dim subtitle on the second line. Use
`rich.markup.escape` on every interpolated value. Tones reuse the existing
`duration-choice-tone-primary` (row `r`) and `duration-choice-tone-accent` (row `a`, the
heavy one); an unavailable row renders `[dim]` throughout with its reason in place of
the chip. Add no new emoji or wide glyphs — the repo has a glyph audit
(`tests/ace/tui/visual/_glyph_audit.py`).

### Behavior

- `BINDINGS`: `r`/`R`/`1` → `this_tab`, `f`/`2` → `full_history`, `u`/`3` → `usage`,
  `a`/`4` → `everything`, `enter` → the cursor row,
  `j`/`k`/`down`/`up`/`ctrl+n`/`ctrl+p` → move the cursor, `escape`/`q` → cancel. All
  `show=False`, like the sibling choosers.
- Cursor movement re-renders the four row `Static`s in place (never rebuilds the screen)
  and skips nothing — an unavailable row is still selectable so `enter` on it produces
  the same explanatory toast as its letter.
- Choosing an available option calls `self.dismiss(choice)`. Choosing an unavailable one
  calls `self.notify(reason, severity="warning")` and returns without dismissing.
- `action_cancel` dismisses `None`.

### Usage row loading must not block first paint

`on_mount` paints immediately with the usage row showing `checking…`, then starts a
single `run_worker(..., thread=True, exit_on_error=False)` — the same shape
`ProviderUsageModal` uses (`models_panel_usage_modal.py:180`). The worker returns a
`UsageRowStatus` frozen dataclass of `provider_count: int`, `chip: str`,
`unavailable_reason: str | None`, computed from `get_usage_metrics_settings().enabled`,
`eligible_usage_providers()`, and `load_usage_view_snapshot()`. Derive the chip from the
newest per-window age using `age_label` from `sase.llm_provider.usage.presentation`, the
same helper `models_panel_usage_rendering` already uses, so the panel and the Usage view
never disagree. `on_worker_state_changed` patches only the usage row. Cancel the worker
in `on_unmount`.

This keeps the panel off the event loop and off the message pump, per rules 1 and 2 of
`sase/memory/tui_perf.md`. Nothing else in `compose`/`on_mount` may touch disk:
`eligible_usage_providers()` reaches provider metadata and CLI readiness checks, so it
belongs in the worker and nowhere else.

### Registration

Add `RefreshChoice`, `RefreshPanelModal`, `RefreshRow`, and `UsageRowStatus` to
`src/sase/ace/tui/modals/_export_table.py`, `modals/__init__.py`'s `__all__`, and
`modals/__init__.pyi`, keeping each list sorted.

### Verification

- New `tests/ace/tui/test_refresh_panel_modal.py`: each letter and numeric alias returns
  the right `RefreshChoice`; `R` returns `this_tab`; `enter` follows the cursor; `j`/`k`
  move and wrap consistently with the sibling modals; `escape` returns `None`; an
  unavailable usage row notifies and does not dismiss; `initial_choice` positions the
  cursor; `banner` renders when supplied and is absent otherwise; a stubbed
  `load_usage_status` patches the usage row after the worker settles; the row chips use
  `freshness_label` output.
- Drop the three `--epic-symbol` entries added by `freshness` from the `Justfile`, and
  add one for `RefreshPanelModal` until `wire` pushes it.
- `just check`.

## Phase: wire — Gesture rewiring behind the refresh_panel flag

### Create the flag first

```bash
sase flag new refresh_panel -k sunset -z small \
  --when-enabled 'R opens the Refresh panel, whose single-key options run the current tab refresh, the Agents full-history rescan, a provider usage-window refresh, or all three, and ,y opens that panel with the cursor on Full history.' \
  --when-disabled 'R immediately refreshes the current tab and ,y immediately runs the Agents full-history refresh, exactly as they behaved before the panel landed.' \
  --remove-when 'The Refresh panel has shipped as the default for a full release and no user has asked to restore the immediate-R gesture.'
```

Paste the printed registry entry into `src/sase/feature_flags/registry.py` (member on
`FeatureFlag` plus the `FeatureFlagDefinition`, both in existing alphabetical position)
and run `just sync-feature-flags-schema`.

### `action_refresh` (`src/sase/ace/tui/actions/base.py`)

Split today's body out into `_refresh_current_tab_surfaces()`, which keeps the existing
per-tab dispatch verbatim plus the `freshness` stamping, and returns the surface label
used for `tab_label`. Then:

- Flag **off**: `action_refresh` calls `_refresh_current_tab_surfaces()` and notifies
  `"Refreshed"` — byte-for-byte today's behavior.
- Flag **on**: `action_refresh` builds the rows and pushes `RefreshPanelModal` with a
  callback that dispatches the choice.

Row construction lives in a new `src/sase/ace/tui/actions/refresh_panel.py` mixin so
`base.py` does not grow further, and reads only in-memory state: `current_tab`,
`current_artifacts_pane_key`, `refresh_interval`, `_countdown_remaining`, and
`surface_refreshed_age`.

### Choice dispatch

- `this_tab` → `_refresh_current_tab_surfaces()`, notify `"Refreshed"`.
- `full_history` → the existing `action_refresh_agents_full_history` body with the
  `current_tab != "agents"` guard removed (design decision 4), notify
  `"Refreshing Agents from full history"`.
- `usage` → `run_worker(thread=True)` calling
  `submit_usage_refresh(None, explicit=True, origin="ace")`; on completion notify from
  the `UsageRefreshReceipt`: started providers, `"Usage refresh already running"` when
  the receipt started nothing, and the disabled reason when usage metrics are off. Never
  call it on the UI thread.
- `everything` → set `self._last_full_sanity_refresh = 0.0` and call
  `self._on_auto_refresh()`, which is the existing forced full-reconcile path that
  ignores every per-surface dirty gate (`event_refresh/_auto_refresh.py:181`), then run
  the `full_history` and `usage` steps. Notify `"Refreshing everything"`. Reusing the
  sanity sweep rather than hand-rolling a new fan-out is required by rule 5 of
  `sase/memory/tui_perf.md`.

### `,y` migration

In `agent_workflow/_leader_mode.py` at the `full_history_refresh` branch:

- Flag **off**: unchanged — call `action_refresh_agents_full_history()`.
- Flag **on**: open the Refresh panel with `initial_choice="full_history"` and a banner
  reading `,y lives here now — press f`. The leader key stays registered so the off
  branch keeps working; it is retired for real only when the flag is removed.

Also gate the flag-on presentation so discovery points at `R`:

- `src/sase/ace/tui/widgets/_keybinding_modes.py:388` — omit the `full history refresh`
  leader footer hint.
- `src/sase/ace/tui/commands/_mode_commands.py:64,118` — omit the `full_history_refresh`
  command-palette entry.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py:315` — replace the `,y` row
  with the `R` Refresh-panel row; add the same row to the app-level bindings section so
  it shows on every tab.
- `src/sase/ace/tui/commands/_app_metadata_actions.py:102` — retitle the `refresh`
  command to `Open Refresh panel` when the flag is on, keeping `Refresh tab` off.

### Verification

Both states are mandatory (`sase/memory/sase_flags.md`). New
`tests/ace/tui/test_refresh_panel_dispatch.py`, parameterized over the flag:

- Flag on: `R` pushes `RefreshPanelModal`; each returned choice calls the right
  collaborator (assert on the scheduling methods, not on disk); `full_history` works
  from Artifacts and Axe, not only Agents; `usage` runs off the UI thread; `everything`
  zeroes `_last_full_sanity_refresh` before triggering the sweep; `,y` pushes the panel
  with the cursor on Full history and the banner set; the leader footer, command
  palette, and help omit `,y`.
- Flag off: `R` refreshes immediately and pushes no screen; `,y` calls
  `action_refresh_agents_full_history` directly; the `,y` footer/palette/help entries
  are present.

Update the existing suites that assert on these surfaces:
`tests/test_command_palette_wiring.py`, `tests/test_command_palette_e2e.py`,
`tests/ace/tui/test_leader_keymap_dispatch.py`, and
`tests/ace/tui/_leader_keymap_helpers.py`. Drop the `RefreshPanelModal` `--epic-symbol`
entry from the `Justfile`. `just check`.

## Phase: finish — Documentation and visual snapshot

### `docs/ace.md`

- Add a **Refresh panel** subsection near the existing refresh narrative (around
  line 1458) describing `R`, the four options, their keys and aliases, the freshness
  chips, and the `refresh_panel` flag as the escape hatch back to the old gestures.
- Lines 1461 and 1465 still say manual refresh is `y`; that is stale since the sase-m6.9
  keymap unification moved refresh to `R`. Correct both while rewriting the paragraph.
- Line 1469 tells the reader to use `,y`; point it at `R` → `f` instead.
- Line 2435's leader-mode table row for `,y` is removed, and the app-level key table
  gains the `R` Refresh-panel row.
- Run `just fmt-md` so the tables stay prettier-clean at the configured print width.

### Visual snapshot

Add `tests/ace/tui/visual/test_ace_png_snapshots_refresh_panel.py`, modeled directly on
`test_ace_png_snapshots_jump_action.py`: start an `AcePage`, push a `RefreshPanelModal`
built from fixed literal rows (fixed chips, fixed provider count, fixed auto-refresh
label — no live clock or disk reads, so the golden is deterministic),
`await page.expect_modal("RefreshPanelModal")`, `await wait_for_visual_idle(page)`, then
`assert_page_png(page, "refresh_panel_120x40", ...)`. Add a second case with the `,y`
banner and the cursor on Full history, since that is the migration surface most likely
to regress. Generate the goldens with `just update-visual-snapshots` and confirm both
render correctly before committing them.

### Final gate

The change touches the ACE refresh paths, the keymap presentation surfaces, docs, and
the visual suite, so this phase closes with `just check-full` run through
`/sase_monitor` (`sase/memory/lint_and_test.md`), plus `just test-visual`.

## Out of scope

- No `ace.keymaps.refresh` config scope (design decision 1).
- No change to the auto-refresh cadence, the surface-token gating, or
  `ace_refresh_tokens`.
- No change to what any individual refresh actually loads; this epic changes how a
  refresh is chosen, not how it runs.
- No new CLI surface.
