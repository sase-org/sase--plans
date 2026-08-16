---
tier: tale
title: Add alias agent history to Launch Control
goal:
  Pressing `H` on an alias-bearing Launch Control row opens a responsive, context-aware
  history modal that shows bounded prior runs, provenance, model and prompt context, and
  supports paging, refresh, hidden-run toggling, prompt preview, copying, navigation,
  and jump hints without changing Launch Control state.
size: medium
proposed_by: bbugyi200.athena.sase-n8.6
bead: sase-n8.6
create_time: 2026-08-16 14:38:38
status: wip
---

- **PROMPT:**
  [prompts/202608/launch_control_alias_history_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/launch_control_alias_history_panel.md)
- **PARENT:** [202608/launch_control_alias_history.md](launch_control_alias_history.md)
- **BEAD:**
  [sase-n8.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-n8/sase-n8.6.md)

# Plan

## Scope and established contracts

This tale implements only phase `panel` of
`plan:202608/launch_control_alias_history.md`. The prerequisite phases already provide:

- `sase.llm_provider.alias_history.load_alias_history`, whose typed view models carry
  ordered alias groups, per-group totals/truncation, status rollups, project display
  names, provenance labels, alias trails, prompt snippets, run identity, and the
  `cached` / `revalidate` freshness knob;
- `llm_provider.model_alias_history_limit`, used by the adapter when no explicit limit
  is supplied; and
- artifact-index schema 22 plus the Python wire/facade. The TUI must call the adapter
  rather than read the SQLite index or reclassify provenance itself.

This phase does not add visual PNG fixtures or goldens (phase `visual` owns those),
change the core query contract, alter history retention, or change alias resolution. The
panel treats the configured limit as a display window and makes truncation visible.

## 1. Resolve Launch Control rows into history requests

Add `models_panel_history.py` beside the other focused Launch Control mixins and compose
it into `ModelsPanel`.

- Add `("H", "alias_history", "History")` to `ModelsPanel.BINDINGS`; lowercase `h`
  remains bucket-back.
- Resolve an `AliasView` to its bare alias and current provider/model/effort display
  context.
- Resolve a `LaunchModelSettingRow` only when `row.snapshot.referenced_alias` is
  present. A concrete launch target produces a clear warning toast and opens nothing.
- Resolve a `BucketView` to every member alias in stable member order, so history works
  on collapsed buckets as well as on drilled-in aliases.
- Reject default-effort, runner-limit, and big-epic-threshold rows with a row-specific
  warning explaining that the setting is not an alias.
- Push the history modal without rebuilding or mutating the parent panel. Add `H` only
  to context-aware footer variants whose selected row supports history (alias,
  alias-backed launch model, and bucket), keeping scalar and concrete-model footers
  honest.

Represent the entry context with a small immutable request/title dataclass so the modal
does not retain mutable Launch Control rows. It should carry aliases, source label,
ownership, and the effective model summary available at entry time.

## 2. Add modal state and one off-thread load path

Add a focused state module for the modal's immutable snapshot and load request. It wraps
`load_alias_history` and records:

- requested aliases and entry context;
- current `limit_per_alias` (initially `None`, allowing the adapter to use config),
  `include_hidden`, and `freshness`;
- the returned `AliasHistoryView`; and
- stable selectable run keys derived from group alias plus artifact directory, so the
  same run can stay highlighted across reloads.

The modal opens immediately with a disabled loading option. Every initial load and
re-query uses one `_start_load` method with
`run_worker(..., thread=True, exclusive=True, group="alias-history-load")`; no query,
filesystem read, or parsing runs in an action/message/render method. A replacement load
cancels the prior worker, and `on_unmount` cancels outstanding workers.

On success, rebuild the options from the immutable result while preserving the prior run
key when it still exists, using the synchronous `_updating_highlight` guard around
programmatic `OptionList.highlighted` changes. On error, leave the modal usable, render
an honest failure/empty row, and show a warning toast. Ignore cancelled/stale worker
results.

All re-query actions use this same path:

- `Ctrl+K` doubles the current effective limit (using the returned/configured limit
  after the first load) and preserves the highlighted run;
- `r` reloads with `freshness="revalidate"` for that request, then returns normal
  subsequent requests to cached freshness;
- `.` toggles hidden runs and reloads while preserving a surviving selection.

## 3. Build the history modal, rendering, and actions

Add a modal module and a pure rendering module. Compose the modal from
`KeyedPaneEntryJumpMixin[str]`, `OptionListNavigationMixin`, and `ModalScreen`, adapting
the existing pane-entry jump hooks to selectable run keys. Route apostrophe-mode keys
through the same normalized jump-key state machine used by Launch Control.

The modal uses Launch Control's visual grammar: title/summary, `OptionList`, fixed
two-line-or-more detail strip, and context footer.

- Title: identify the alias or bucket, preserve the tan ownership accent for user-owned
  sources, include the effective provider/model/effort when a single entry supplied it,
  and report total recorded, returned, and done/failed/running counts.
- Rows: newest-first runs exactly in adapter order, with status marker, relative time,
  agent/workflow identity, configured project display name, provider-themed model badge
  plus effort, and the adapter's `direct`, `default`, `via @...`, or `unrecorded` chip.
  Multiple aliases receive disabled group headers and the existing single-spacer
  convention; headers/spacers/loading/empty rows are never jump targets.
- Detail strip: render the recorded alias trail to the concrete provider/model and
  effort, state the known launch origin, and show a non-speculative legacy explanation
  when provenance is unrecorded. Follow with the prompt snippet and available project,
  workspace, bead, start/duration, retry, Patch, hidden, and xprompt context without
  inventing missing values.
- Empty state: name the requested alias/group and explain that provenance-aware history
  is recorded for newer runs, without claiming facts the query contract cannot prove
  about unrelated index rows.
- Footer: advertise navigation/jump, Enter prompt, `y` copy, `Ctrl+K` load more, `r`
  refresh, `.` hidden toggle, and close keys; indicate current hidden state and whether
  more rows are available.

Bindings are `j`/`k`/arrows/`Ctrl+N`/`Ctrl+P`, apostrophe jump, Enter, `y`, `Ctrl+K`,
`r`, `.`, and `Esc`/`q`. Enter reads the selected run's `raw_xprompt.md` off-thread and
opens `PreviewPanelModal` with a Markdown `PreviewPayload`; missing/unreadable prompts
produce a warning without closing the history modal. `y` derives a durable agent
reference through the existing `reference_for_agent_name` helper and sends `@agent:...`
through `schedule_copy_delivery`; runs without a durable agent name warn instead of
copying a guessed reference.

Use `provider_styles.py` for model badges and existing duration/time helpers where their
semantics match. Keep render helpers pure and defensive around absent legacy fields.

## 4. Style and document the surface

Add `AliasHistoryModal` and `#alias-history-*` rules to `src/sase/ace/tui/styles.tcss`
beside Provider Routing. Share its centered modal, double-border, width/chrome budget,
bounded option-list viewport, detail border, and footer treatment, with enough height
for grouped rows and wrapped details at narrow viewports.

Extend `docs/ace.md`'s Launch Control section with:

- `H` in the main key table;
- what the history panel answers and its row/detail anatomy;
- the four provenance labels and the honest pre-recording meaning of `unrecorded`;
- the display-window versus retention distinction; and
- the modal key table for prompt preview, copy, paging, refresh, hidden toggle,
  navigation/jump, and close.

## 5. Test the contract at pure and mounted levels

Add focused history rendering/state tests plus mounted Textual tests using the existing
Models-panel app helpers and deterministic clocks. Cover:

- the uppercase `H` binding and footer visibility on alias, alias-backed launch-setting,
  bucket, concrete launch-setting, and scalar rows;
- opening from an alias, launch setting, and collapsed bucket, including ordered grouped
  output, while rejected rows only toast;
- all four provenance row labels, provider/model/effort/project cells, trail and
  unrecorded details, title rollups, truncation, grouped headers/spacers, and empty
  state;
- immediate loading paint and threaded worker success, failure toast, cancellation on
  unmount, and selection preservation;
- `Ctrl+K` limit doubling, `r` revalidation, hidden toggle, and the fact all three reuse
  the same load seam;
- normal navigation plus adaptive jump over selectable runs only;
- Enter opening the full prompt preview and warning for a missing prompt;
- `y` scheduling a durable `@agent:` reference and warning for a run without one; and
- `Esc`/`q` returning to the original Launch Control instance with its active bucket,
  rows, highlight, and change flags unchanged.

Keep phase `visual`'s PNG scenarios out of this phase. Update existing keymap/footer
assertions and add geometry assertions where needed to catch clipped title/detail/footer
content.

## Verification

1. Run the focused history, Models-panel keymap/action/layout, and
   documentation-adjacent tests during development.
2. Run `just install` because this numbered workspace may have stale dependencies.
3. Run the required repository gate with `just check`; if its scoped selection escalates
   or reports unusual selection, follow project guidance and use the monitored
   exhaustive lane rather than running `just check-full` inline.
4. Re-run the focused mounted modal tests after formatting/fixes and inspect
   `git diff --check` plus the final diff for accidental PNG/golden, default-keymap, or
   unrelated changes.
