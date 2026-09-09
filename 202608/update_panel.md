---
tier: epic
status: done
title: The ,U Update panel — scoped, cached, Admin-Center-free updates
goal: "Pressing ,U opens a fast, keyboard-first Update panel rendered entirely from
  already-fetched update evidence. Each option runs its scoped update as a background
  proc that asks for the same y/n confirmation ACE asks today, and no option opens the
  SASE Admin Center.

  "
phases:
  - id: evidence
    title: Cached update evidence and the panel state projection
    depends_on: []
    size: small
    description: "evidence: stash the periodic UpdateStatus on the app, share the
      existing update/agents accent palette, and add a pure projection that turns the
      two cached snapshots into the panel's four option rows.

      "
  - id: preview
    title: Pane-free, scope-aware update preview
    depends_on: []
    size: medium
    description: "preview: introduce UpdateLeg/UpdateScope, collect preview inputs
      without the Updates pane, and teach the comprehensive preview and its confirm
      rendering to cover only the selected legs.

      "
  - id: procs
    title: App-level update execution and proc submission
    depends_on:
      - preview
    size: medium
    description: "procs: move comprehensive execution, proc submission, completion, and
      restart onto an app-level mixin, run the preview itself as a tracked proc, and
      make submission and summaries scope-aware.

      "
  - id: panel
    title: The UpdatePanel modal
    depends_on:
      - evidence
    size: medium
    description: "panel: build the keyboard-first UpdatePanel modal, its rows, chips,
      styling, and exports, driven only by the projected panel state.

      "
  - id: wire
    title: Wire ,U to the panel and the scoped flow
    depends_on:
      - evidence
      - preview
      - procs
      - panel
    size: medium
    description: "wire: repoint the ,U leader chord at the panel, run the selected scope
      as a preview proc into the y/n confirm modal, and refresh every label that
      describes the chord.

      "
  - id: retire
    title: Retire the Admin Center auto-update path
    depends_on:
      - wire
    size: medium
    description: "retire: delete the auto_update and captured-provider plumbing that
      only existed for the old ,U dispatch, and drop the comprehensive mixin from the
      Updates pane.

      "
  - id: visual
    title: Visual snapshots and final verification
    depends_on:
      - panel
      - wire
    size: small
    description:
      "visual: add PNG goldens for the populated and never-checked panel states and run
      the exhaustive verification lane."
proposed_by: bbugyi200.athena.080
bead_id: sase-r1
create_time: 2026-09-09 19:52:04
---

- **PROMPT:**
  [prompts/202608/update_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/update_panel.md)
- **BEAD:**
  [sase-r1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-r1/README.md)

# Plan: The `,U` Update panel

## Context

`,U` (leader `update_sase`) currently calls `action_update_sase_shortcut()` in
`src/sase/ace/tui/actions/base.py:138`. That opens the **SASE Admin Center** on the
Updates tab with `auto_update=True` and the captured provider names from the last
automatic check. The pane then performs a _full live inventory load_ (uv-tool probe,
plugin catalog, PyPI enrichment, agent-CLI detection with `--version` subprocesses,
agents-sync reconcile) before it can build a preview, show the y/n confirm modal, and
submit one tracked proc that runs all three legs.

Two problems: the chord is slow because it waits on a live load, and it is
all-or-nothing — there is no way to update just providers or just the agents cache from
the chord.

The evidence needed to _describe_ the available work is already fetched periodically:

- `UpdateToastMixin` (`src/sase/ace/tui/actions/update_toast.py`) runs an automatic
  check on a timer and produces an `sase.updates.UpdateStatus` with `components` (host /
  core / plugin) and `provider_candidates`. Today it throws all of that away except the
  provider names (`_automatic_update_provider_names`).
- `AgentsSyncActionsMixin` (`src/sase/ace/tui/actions/agents_sync.py`) runs its own
  timer and already keeps the whole snapshot in `self._agents_sync_last_status`.

So the panel can be rendered from memory with zero I/O.

## The design

### The panel

A modal chooser, `UpdatePanel`, opened by `,U`. It is pure presentation over an
immutable `UpdatePanelState`; it performs no I/O of its own.

```
╭─ ↑ Update ─────────────────────────────── checked 4m ago ─╮
│ ┌───────────────────────────────────────────────────────┐ │
│ │ e  Everything                             ↑ 6 available│ │
│ │    SASE, providers, and published agents in one       │ │
│ │    tracked update.                                    │ │
│ │                                                       │ │
│ │ s  SASE, core & plugins                   ↑ 4 available│ │
│ │    Upgrade the sase host package, sase-core, and      │ │
│ │    every installed plugin.                            │ │
│ │    sase 1 · sase-core 1 · plugins 2                   │ │
│ │                                                       │ │
│ │ p  Providers                              ↑ 2 available│ │
│ │    Update every installed LLM / agent CLI provider.   │ │
│ │    claude, codex · 1 needs manual steps               │ │
│ │                                                       │ │
│ │ a  Agents                                  ✓ up to date│ │
│ │    Import agent hoods your other machines published.  │ │
│ └───────────────────────────────────────────────────────┘ │
│  e s p a select · j/k move · ⏎ run · r re-check · q close  │
╰───────────────────────────────────────────────────────────╯
```

Row anatomy: line 1 is `key badge · title · right-aligned status chip`; line 2 is the
always-present description; line 3 appears **only** when the cached evidence has a
breakdown worth showing. Rows grow only when they have something to say.

**Rows** (order is fixed; `Everything` is highlighted on open so `,U` `⏎` reproduces
today's muscle memory in three keystrokes):

| key | title                | description                                                           | scope          |
| --- | -------------------- | --------------------------------------------------------------------- | -------------- |
| `e` | Everything           | SASE, providers, and published agents in one tracked update.          | all three legs |
| `s` | SASE, core & plugins | Upgrade the sase host package, sase-core, and every installed plugin. | SASE leg       |
| `p` | Providers            | Update every installed LLM / agent CLI provider.                      | provider leg   |
| `a` | Agents               | Import agent hoods your other machines published.                     | agents leg     |

Keys are lowercase-only and mutually distinct (`e`/`s`/`p`/`a`), so no shift-slip can
promote a narrow choice into the heavy one. `j`/`k` move, `⏎` runs the highlighted row,
mouse click runs a row, `r` re-checks, `q`/`escape` closes.

**Status chips** reuse the palette the top bar already established, so the panel reads
as the same system as the badges it explains:

| state           | chip                | style                                |
| --------------- | ------------------- | ------------------------------------ |
| work available  | `↑ N available`     | row accent, bold                     |
| nothing pending | `✓ up to date`      | green, dim                           |
| never checked   | `· not checked yet` | dim                                  |
| check failed    | `! check failed`    | red; the source error becomes line 3 |

Row accents come from the existing indicators: SASE `#AF87FF` (`#FFAF5F` when a pending
core rebuild is known), Providers `#00D7FF`, Agents `#5FD787`, Everything `$primary`.
Glyphs `↑` for updates and `⇅` for the agents row, matching the badges.

**Freshness** is disclosed in the border subtitle: `checked 4m ago`,
`never checked — press r`, or `re-checking…` while an `r`-triggered check is in flight.
When the newest of the two snapshots is older than 30 minutes the subtitle renders in
the amber `#FFAF5F` accent.

### The flow

1. `,U` builds `UpdatePanelState` from two in-memory fields and pushes `UpdatePanel`. No
   worker, no disk, no network. This is the fast path the request asks for.
2. Choosing a row dismisses the panel immediately and submits an **`update-preview`
   proc** (`_submit_session_worker`, no exclusive scopes, `dedup_key="update-preview"`).
   The proc collects the inputs the preview needs and builds the scoped preview.
3. On completion the proc's callback pushes the existing `PluginActionConfirmModal` —
   the same `y`/`n` confirmation shown today — containing only the selected legs'
   sections.
4. Confirming submits the tracked update proc exactly as today: same exclusive scopes,
   same completion toast, same post-update receipt and restart behavior.
5. Cancelling does nothing; no scope was ever claimed.

The Admin Center is never pushed at any point.

### Design decisions

- **Rows are never disabled.** A stale or missing snapshot must not lock the user out. A
  row whose cached evidence says `up to date` is still selectable; the preview proc runs
  and reports the truthful no-op through the existing toast.
- **Snapshot gating is preserved.** The provider leg keeps intersecting the captured
  provider names with live inventory and never broadens scope. The SASE leg treats
  itself as already-current _only_ when the cached snapshot exists, both its
  `core_source` and `plugin_source` are `successful`, and `component_count == 0`;
  otherwise it builds the real preview rather than trusting silence.
- **`r` re-check is opt-in.** It reuses the app's existing periodic paths
  (`_schedule_automatic_update_check(periodic=True)` and
  `_schedule_agents_sync_status_check(recompute=True)`) and refreshes the open panel in
  place. The default path stays cache-only.
- **One update at a time.** Every scope claims all three exclusive scopes
  (`sase-update`, `agent-cli-update`, `agents-sync`). Narrowing them would let a SASE
  update finish and restart ACE underneath a concurrent agents import. Updates are rare;
  serializing them is the reliable choice.
- **No feature flag.** This lands complete and ready: it is not a disabled beta, not an
  early landed path, and no deprecated branch has to stay reachable — the Admin Center
  Updates tab keeps its own `u` / `A` / `a` actions and stays reachable from the top-bar
  updates badge and the Admin Center opener.
- **No Rust core change.** Nothing new enters the shared backend domain. Every phase
  re-composes update services that already live in Python (`sase.updates`,
  `sase.dev_update`, `sase.agent_clis`, `sase.agents_sync`, `sase.uv_tool`); the new
  code is TUI composition and presentation only.

### Naming note

`action_open_updates_panel` already exists and opens the Admin Center's **Updates** tab.
The new modal is the **Update panel** (`UpdatePanel`, singular). Keep the existing
action name untouched; do not rename it as part of this epic.

---

## Cached update evidence and the panel state projection

Give the app the evidence the panel needs and the pure function that shapes it.

**Stash the full status.** `src/sase/ace/tui/actions/update_toast.py` currently derives
only `_automatic_update_provider_names` in `_apply_startup_update_status`. Add
`_automatic_update_status: UpdateStatus | None` beside it, assigned from the same
UI-thread application point so both fields stay consistent. Keep
`_automatic_update_provider_names` exactly as-is — `action_update_sase_shortcut` and
several tests read it. Declare the new attribute in the mixin's class-level annotations,
in `src/sase/ace/tui/actions/_state_init.py`, and initialize it to `None` in
`src/sase/ace/tui/actions/_state_init_runtime.py` next to the existing initializer.

**Share the accent palette.** `_UPDATES_ACCENT`, `_CORE_UPDATE_ACCENT`,
`_AGENT_CLI_ACCENT` and `_UPDATE_GLYPH` are private to
`src/sase/ace/tui/widgets/updates_indicator.py`; `_AGENTS_SYNC_ACCENT` and
`_AGENTS_SYNC_GLYPH` are private to `src/sase/ace/tui/widgets/agents_sync_indicator.py`.
Move them into a new `src/sase/ace/tui/widgets/update_accents.py` as public names
(`UPDATES_ACCENT`, `CORE_UPDATE_ACCENT`, `AGENT_CLI_ACCENT`, `AGENTS_SYNC_ACCENT`,
`UPDATE_GLYPH`, `AGENTS_SYNC_GLYPH`) and import them from both indicators and, later,
the panel. Keep module-private aliases in the two indicator modules if that avoids
churning their tests, but the single definition must live in the new module — the panel
and the badges must never drift apart.

**Project the state.** Add `src/sase/ace/tui/update_panel_state.py`: pure, no Textual
import, no I/O. It defines:

- `UpdateOptionChip` — a small frozen record: `kind` (`available` / `current` /
  `unknown` / `failed`), `text`, and the count.
- `UpdateOptionRow` — `scope`, `key`, `title`, `description`, `chip`, `detail` (the
  optional line-3 breakdown), and `accent`.
- `UpdatePanelState` — `rows: tuple[UpdateOptionRow, ...]`, `freshness_label: str`,
  `stale: bool`, `rechecking: bool`.
- `build_update_panel_state(status, agents_snapshot, *, now, rechecking=False)`.

Projection rules:

- SASE row count is `status.component_count`; the detail line groups `status.components`
  by `role` into `sase N · sase-core N · plugins N`, using only the roles that are
  non-zero. When `status.has_core_update`, the row accent switches to
  `CORE_UPDATE_ACCENT` and the detail line gains ` · core rebuild`.
- Providers row count is `status.agent_cli_count`; the detail line lists candidate
  display names (truncate past four with `+N more`) and appends `· N need manual steps`
  when `status.manual_agent_cli_count` is non-zero.
- Agents row count is
  `sum(project.pending_foreign_count for project in agents_snapshot.projects)`; the
  detail line names the distinct source machines when the count is non-zero.
- Everything row count is the sum of the other three. Its chip is `failed` if any
  contributing source failed, `unknown` if every contributing source is unknown.
- A row is `failed` when its backing `UpdateSourceStatus` has an `error`; the error text
  becomes the detail line. SASE draws on both `core_source` and `plugin_source`;
  Providers on `agent_cli_source`; Agents on any `ProjectSyncStatus.error`.
- A row is `unknown` when its backing source has never completed
  (`UpdateSourceStatus.known` is false), or when `status` / `agents_snapshot` is `None`.
- `freshness_label` is derived from the **newer** of the two `checked_at` values via a
  compact relative formatter (`just now`, `4m ago`, `2h ago`, `3d ago`); `stale` is true
  past 30 minutes; both are `None`/true-stale when no snapshot exists at all.

Tests: `tests/ace/tui/test_update_panel_state.py`, table-driven over the shapes that
matter — everything current, mixed counts, a core rebuild, a failed provider source, a
never-checked app, a `None` agents snapshot, manual-only providers, stale vs fresh
timestamps. Extend `tests/ace/tui/test_update_toast_automatic_checks.py` to assert the
new `_automatic_update_status` field is stashed and replaced on each successful check.

---

## Pane-free, scope-aware update preview

Today the comprehensive preview is a mixin method on `PluginsBrowserPane` that reads
`self._agent_cli_statuses`, `self._uv_tool`, `self._offline`, `self._loading` and calls
`self._sase_up_to_date()` and `self._make_sase_update_preview()`. Make the preview a
function of explicit inputs so the app can build it without the pane.

**Scope vocabulary.** Add `src/sase/ace/update_scope.py` (next to
`src/sase/ace/comprehensive_update.py`, importable from both TUI and non-TUI code):

- `UpdateLeg(StrEnum)`: `SASE`, `PROVIDERS`, `AGENTS`.
- `UpdateScope(StrEnum)`: `EVERYTHING`, `SASE`, `PROVIDERS`, `AGENTS`, with a `legs`
  property returning the `frozenset[UpdateLeg]` it selects and `ALL_LEGS` as a module
  constant.

**Request and preview carry the scope.** In
`src/sase/ace/tui/modals/plugins_browser_comprehensive_update_models.py`, add
`scope: UpdateScope = UpdateScope.EVERYTHING` to `ComprehensiveUpdateRequest`, and add a
`selected_legs` property to `ComprehensiveUpdatePreview` delegating to
`request.scope.legs`. Gate the runnable properties on selection: `sase_runnable`,
`provider_runnable`, and `agents_runnable` each return `False` when their leg is not
selected. An unselected leg is _not_ a skip and _not_ "already current" — it simply does
not appear.

**Collect inputs off the pane.** Add `src/sase/ace/tui/update_preview_inputs.py` with a
frozen `UpdatePreviewInputs` (`uv_tool`, `agent_cli_statuses`, `agent_cli_error`,
`offline`, `cached_status`) and `collect_update_preview_inputs(*, cached_status, legs)`.
It runs off-thread and does the minimum the selected legs require:

- `UpdateLeg.SASE` → `probe_uv_tool()` from
  `src/sase/ace/tui/modals/plugins_browser_loading.py`.
- `UpdateLeg.PROVIDERS` → `collect_agent_cli_statuses(refresh=False, offline=False)`,
  capturing any exception as `agent_cli_error` rather than raising.
- `UpdateLeg.AGENTS` → nothing here; the agents leg already reads
  `get_agents_sync_status(revalidate_only=True)` inside the preview builder.

Skipping the collection a leg does not need is what keeps a Providers-only or
Agents-only selection quick.

**Make the builder a function.** Extract the body of
`ComprehensiveUpdateActionsMixin._start_comprehensive_update_preview`'s inner `task()`
into a module-level
`build_comprehensive_update_preview(request, inputs, *, already_refreshed_roots=())` in
`src/sase/ace/tui/modals/plugins_browser_comprehensive_update_preview.py`. Behavior
changes only where the pane state used to supply an answer:

- The agents leg runs only when `UpdateLeg.AGENTS` is selected.
- The provider leg runs `plan_captured_providers` only when `UpdateLeg.PROVIDERS` is
  selected, unchanged otherwise.
- `sase_current` is no longer `self._sase_up_to_date()`. Derive it from the cached
  status: current when `inputs.cached_status` is not `None`, both `core_source` and
  `plugin_source` are `successful`, and `component_count == 0`. When the cached status
  is missing or a source failed, do not claim current — build the real preview.
- The remaining SASE-leg logic (`NotUvToolInstall` blocker, `load_receipt_for_summary`,
  `make_sase_dev_update_preview`, `dev_update_blocking_reason`, the managed-argv
  exception) moves across unchanged, taking `inputs.uv_tool` instead of `self._uv_tool`.
  `make_sase_dev_update_preview` is already a module-level function in
  `plugins_browser_dev_update.py`, so call it directly.

**Scope the confirm rendering.** `sase_preview_section`, `provider_preview_section`, and
`agents_preview_section` stay as they are, but the caller assembles only the selected
legs' sections. Add scope-aware modal copy so a narrow choice does not read like a wide
one:

| scope        | title                       | intro                                                                      |
| ------------ | --------------------------- | -------------------------------------------------------------------------- |
| `EVERYTHING` | Update everything           | (today's text, unchanged)                                                  |
| `SASE`       | Update SASE, core & plugins | Confirm the SASE, core, and plugin work below.                             |
| `PROVIDERS`  | Update providers            | Confirm the exact provider update commands below; they run sequentially.   |
| `AGENTS`     | Import published agents     | Confirm the cached agent hoods to import. Only the captured cache is used. |

Also scope `_handle_comprehensive_noop`: its "no captured updates remain" and
"everything is already current" messages must name the chosen scope, and the
manual-provider branch must not try to switch an Admin Center sub-tab when no Admin
Center is mounted — guard on `_switch_to_subtab` being present (it already uses
`getattr`) and fall back to a toast that points at the Admin Center Updates tab.

Tests: extend `tests/ace/tui/test_plugins_browser_pane_comprehensive_update.py` and
`..._confirmation.py`, and add a new module covering the extracted function directly —
one case per scope asserting which legs are planned, which sections render, and that an
unselected leg is absent rather than reported as skipped.

---

## App-level update execution and proc submission

Move the second half of the flow off the pane so nothing needs the Admin Center mounted.

**New app mixin.** Add `src/sase/ace/tui/actions/update_run.py` with
`UpdateRunActionsMixin`, mixed into the ACE app alongside `UpdateToastMixin` and
`AgentsSyncActionsMixin`. It owns:

- `_submit_update_preview_proc(request)` — submits an `update-preview` proc via
  `_submit_session_worker` with `display_name="plan update"`, a scope-derived `cl_name`,
  `dedup_key="update-preview"`, **no** `exclusive_scopes` (planning is read-only), and a
  `duplicate_message` of "An update is already being planned." Its body calls
  `collect_update_preview_inputs` then `build_comprehensive_update_preview`, and returns
  the preview as the `TrackedProcResult` payload.
- `_on_update_preview_complete(completion)` — on a `None` payload, toast the error; on a
  non-runnable preview, run the scoped no-op handler; otherwise push
  `PluginActionConfirmModal` on the **app** with the scoped sections and copy.
- `_submit_scoped_update_task(preview)` — today's `_submit_comprehensive_update_task`,
  with a scope-derived `display_name` / `cl_name`, the unchanged `dedup_key`
  `"comprehensive-update"`, and the unchanged three `exclusive_scopes`.
- `_on_scoped_update_complete(completion)` — today's
  `_on_comprehensive_update_complete`, unchanged apart from dropping the pane-only
  `self._agent_cli_results` bookkeeping and the trailing `self._start_load(force=False)`
  (there is no pane to reload). It keeps the indicator revalidations, the agents async
  refresh, the update receipt + pending toast, and the restart.

**Move the execution and restart helpers.** `ComprehensiveUpdateExecutionMixin`
(`plugins_browser_comprehensive_update_execution.py`) reads only `self._uv_tool` and
`self._run_sase_update_summary` / `self._execute_dev_update`, both of which are
`@staticmethod` shims in `plugins_browser_operations.py` that just forward to
module-level functions. Convert the mixin's methods to module-level functions taking the
`uv_tool` explicitly (`execute_provider_leg(preview)`,
`execute_sase_leg(preview, uv_tool)`, `execute_agents_leg(preview)`), and have the new
app mixin call them. Move `_restart_after_update` / `_restart_after_update_when_ready` /
`running_background_procs` out of `plugins_browser_sase_update_procs.py` into a shared
module the pane and the app mixin both import — they already operate purely on
`self.app`, so the extraction is mechanical.

**Scope the summary.** Add `selected_legs: frozenset[UpdateLeg]` to
`ComprehensiveUpdateResult` in `src/sase/ace/comprehensive_update.py`, defaulting to all
three so existing construction sites keep today's behavior. Teach
`comprehensive_update_summary` to emit only the selected legs' lines, and have the
executor record an unselected leg's SASE status as `SKIPPED` with reason `not selected`
so the payload stays truthful even though the line is suppressed.

Tests: extend
`tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py` for the
extracted functions, and add a new module for the app mixin covering: a preview proc
that yields a runnable preview pushes the confirm modal; a non-runnable preview toasts
instead; a confirmed modal submits with the expected `display_name` / `cl_name` /
`dedup_key` / `exclusive_scopes` per scope; a duplicate submission is rejected with the
existing message; a `code_changed` result restarts and a non-changing one toasts; a
scoped summary omits unselected legs.

---

## The UpdatePanel modal

Add `src/sase/ace/tui/modals/update_panel.py`. Pure presentation: it takes an
`UpdatePanelState`, renders it, and dismisses with a result. It must not import
`sase.updates` or `sase.agents_sync` and must not touch the filesystem.

- `UpdatePanelResult` — frozen, `scope: UpdateScope`.
- `UpdatePanel(ModalScreen[UpdatePanelResult | None])` with
  `__init__(self, state: UpdatePanelState)`.
- `BINDINGS`: `e` / `s` / `p` / `a` choose their row; `j` / `k` move; `enter` chooses
  the highlighted row; `r` requests a re-check; `q` and `escape` cancel.
- `compose()` yields a `Container#update-panel-container` holding an
  `OptionList#update-panel-list` and a `Static#update-panel-hints` footer, following
  `agent_cleanup_panel_modal.py` as the structural precedent (it is the closest existing
  keyboard-first chooser and gives mouse selection for free through
  `on_option_list_option_selected`).
- Each `Option` id is the `UpdateScope` value; each prompt is a
  `Table.grid(expand=True)` with two columns — left `key badge + title`, right-justified
  chip — followed by the dim description row and, when present, the dim accent detail
  row. If grid expansion misbehaves inside `OptionList`, fall back to a `rich.text.Text`
  with the chip padded to the option's content width; the right-aligned chip is the
  requirement, the mechanism is not.
- Border title is `↑ Update`; border subtitle is `state.freshness_label`, rendered in
  `CORE_UPDATE_ACCENT` when `state.stale`, and replaced by `re-checking…` when
  `state.rechecking`.
- `set_state(state)` re-renders rows in place while preserving the highlighted index, so
  a background re-check can land without moving the cursor under the user.
- `r` posts an `UpdatePanel.RecheckRequested` message rather than doing work itself; the
  app handles it. The panel stays I/O-free.
- Footer hint: `e s p a select · j/k move · ⏎ run · r re-check · q close`.

Style it in `src/sase/ace/tui/styles.tcss` next to the `AgentCleanupModal` block: a
`thick $primary` container border, `$surface` background, `1 2` padding, width `78` with
`max-width: 92%`, an inner list bordered `solid $secondary`, and a centered
`$text-muted` hint line. Export `UpdatePanel` and `UpdatePanelResult` through the lazy
map and `__all__` in `src/sase/ace/tui/modals/__init__.py`, and add the matching
re-export lines to `src/sase/ace/tui/modals/__init__.pyi`.

Tests: `tests/ace/tui/test_update_panel.py` — each letter key dismisses with the right
scope; `enter` on the default highlight yields `EVERYTHING`; `j`/`k` move and `enter`
follows; `escape` and `q` dismiss `None`; `r` posts the re-check message without
dismissing; `set_state` preserves the highlight; a never-checked state still renders
four selectable rows.

---

## Wire `,U` to the panel and the scoped flow

Repoint the chord and refresh everything that describes it.

**The action.** Rewrite `action_update_sase_shortcut` in
`src/sase/ace/tui/actions/base.py`. It must stay allocation-only — build
`UpdatePanelState` from `self._automatic_update_status` and
`self._agents_sync_last_status`, then `push_screen(UpdatePanel(state), callback)`. No
worker, no `_open_config_center`. The callback ignores `None` and otherwise calls
`_submit_update_preview_proc(ComprehensiveUpdateRequest(provider_names=…, scope=result.scope))`,
passing the same `_automatic_update_provider_names` projection the old code passed.

**Re-check.** Handle `UpdatePanel.RecheckRequested` on the app: call
`_schedule_automatic_update_check(periodic=True)` and
`_schedule_agents_sync_status_check(recompute=True)`, mark the open panel `rechecking`,
and push a fresh state into it when each check completes. Wire that through the two
existing completion points (`_apply_startup_update_status` and
`_complete_agents_sync_status_check`) via a single small `_refresh_open_update_panel()`
helper that no-ops when the panel is not the active screen.

**Labels.** The action id `update_sase` and the key `U` do not change, so
`src/sase/default_config.yml` and `src/sase/ace/tui/keymaps/mode_keymaps.py` need no
edit. Update the human text in:

- `src/sase/ace/tui/commands/_mode_commands.py` — `"update_sase"` becomes
  `"Update panel (SASE, providers, agents)"`.
- `src/sase/ace/tui/widgets/_keybinding_modes.py:414` — the leader footer entry becomes
  `"update panel"`.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`, `.../patches_bindings.py`,
  `.../axe_bindings.py` — the `update_sase` row label becomes
  `"Update panel (SASE, providers, agents)"`.

Tests: update `tests/ace/tui/test_log_panel_keymap.py`'s
`test_update_sase_shortcut_opens_updates_with_auto_update` to assert the panel is pushed
with the projected state instead of the Admin Center; keep
`tests/ace/tui/test_leader_keymap_dispatch.py` passing (the dispatch contract is
unchanged). Add coverage that a chosen scope submits an `update-preview` proc carrying
that scope, and that the chord does no I/O — assert the cached-status accessors are not
called during dispatch.

---

## Retire the Admin Center auto-update path

With `,U` off it, the auto-update plumbing has no caller. Remove it rather than leaving
a dead second entry point into the same flow.

- `src/sase/ace/tui/actions/base.py` — drop the `auto_update` and
  `comprehensive_provider_names` parameters from `_open_config_center` and from its
  remaining call sites.
- `src/sase/ace/tui/modals/config_center_modal.py` — drop the two constructor parameters
  and the `_auto_update` / `_comprehensive_provider_names` attributes.
- `src/sase/ace/tui/modals/config_center_catalog.py:83` — stop forwarding them into the
  pane factory.
- `src/sase/ace/tui/modals/plugins_browser_pane.py` — drop `auto_update_on_load`,
  `comprehensive_provider_names`, `_comprehensive_update_request`,
  `_starting_comprehensive_request`, and `_comprehensive_update_plan_worker`; drop
  `ComprehensiveUpdateActionsMixin` from the class bases.
- `src/sase/ace/tui/modals/plugins_browser_workers.py` — remove the
  `_auto_update_on_load` handoff block and its error-path resets. Keep
  `_reusable_fresh_editable_roots` and the updates-indicator revalidation; the pane's
  own `u` action still uses them.
- `src/sase/ace/tui/modals/plugins_browser_sase_update.py` — remove the
  `_starting_comprehensive_request` branch from `_start_sase_update_preview`.
  `_close_admin_center_after_sase_update` stays: the pane's own `u` action still closes
  the Admin Center after a confirmed self-update.
- `plugins_browser_comprehensive_update.py` keeps only what the extraction left behind;
  if nothing remains after the `preview` and `procs` phases, delete the module and its
  re-export shims.

The Updates pane must keep working exactly as before for its own keys: `u` (core +
plugins), `A` (agent CLIs), `a` (sync agents), and every install/uninstall/mode-switch
action.

Tests: relocate or delete the three `test_plugins_browser_pane_comprehensive_update*.py`
modules according to where their subject landed; remove pane tests that construct the
pane with `auto_update_on_load=True`; keep the Admin Center construction tests green.
Run `just check` and fix any symvision unused-symbol findings the deletions expose.

---

## Visual snapshots and final verification

Add PNG goldens under `tests/ace/tui/visual/`, following
`test_ace_png_snapshots_agents_sync_indicator.py` for structure and
`test_ace_png_snapshots_config_center_plugin_actions.py` for a modal capture:

- `update_panel_pending_120x40.png` — every row populated: a SASE row with a core
  rebuild, a provider row with a manual-steps caveat, a populated agents row, and a
  fresh `checked … ago` subtitle.
- `update_panel_unchecked_120x40.png` — no snapshots at all: four `· not checked yet`
  rows and the `never checked — press r` subtitle.

Both fixtures build an explicit `UpdatePanelState` and push the panel directly, so the
goldens stay deterministic and never depend on real update checks. Generate them with
`just test-visual --sase-update-visual-snapshots` and confirm a clean re-run.

Finish with the exhaustive lane. `just check-full` outruns a single agent turn, so run
it through `/sase_monitor` (`sase monitor start --command 'just check-full' …`) with a
`--next` action rather than inline, and remember `just install` first — the workspace's
virtualenv may be stale.

## Verification

- `just install`, then `just check` after each phase.
- `just test-visual` for the `visual` phase.
- `just check-full` through `/sase_monitor` before the epic lands.
- Manual smoke in ACE: `,U` paints instantly with no visible load; `⏎` reproduces the
  old all-legs confirmation; `s`, `p`, and `a` each confirm only their own section; `n`
  cancels cleanly; `y` submits one proc visible in the Admin Center Procs tab; the SASE
  Admin Center never appears at any point in the flow.
