---
tier: epic
title: Stop the @epic tribe panel flickering when new nodes join it
goal: 'A new agent node joining the @epic tribe panel produces exactly one visual
  transition: the panel widget is never blanked, its highlight and scroll position
  survive, the agent-list column geometry settles in the same frame as the rows, and
  an apply that changes nothing repaints nothing. The claim is proved by a deterministic
  frame-level harness in CI and re-proved by a landed-SHA soak on athena that actually
  creates new nodes.

  '
phases:
- id: flicker-frame-harness
  title: Deterministic frame-level repro for a node joining @epic
  depends_on: []
  size: medium
  description: 'flicker-frame-harness: build the per-refresh paint log and the no-blank/one-transition
    invariants, drive a new node into an @epic clan panel through the real apply pipeline
    under the host''s grouping, query and collapse shape, and land it green with a
    strict xfail on every invariant the current tree violates.

    '
- id: collapsed-panel-mode
  title: Stop collapsed panels forcing a full rebuild on every apply
  depends_on: []
  size: small
  description: 'collapsed-panel-mode: record the active grouping mode when a panel
    paints collapsed so the stale_grouping_mode guard stops sending 232 of 234 applies
    down the full-rebuild path, without weakening the guard for panels that hold rows
    from an earlier mode.

    '
- id: row-insert-without-blanking
  title: Add rows in place and settle the column in one frame
  depends_on:
  - flicker-frame-harness
  - collapsed-panel-mode
  size: medium
  description: 'row-insert-without-blanking: give AgentList an in-place row insert
    to mirror try_remove_rows so an added node stops clearing and re-emitting the
    whole panel, and make the agent-list container width settle inside the refresh
    that changed the rows instead of one pump cycle later.

    '
- id: verify-new-nodes-on-athena
  title: Prove it on athena with real node arrivals and close sase-13i.4
  depends_on:
  - flicker-frame-harness
  - collapsed-panel-mode
  - row-insert-without-blanking
  size: medium
  description: 'verify-new-nodes-on-athena: soak a landed SHA under the host''s real
    state while deliberately creating nodes that join the @epic tribe, assert the
    harness invariants from traces and live captures, then close sase-13i.4 with the
    evidence or record precisely why it still cannot close.'
proposed_by: bbugyi200.athena.sase-13i.4.f0.f0
create_time: 2026-09-20 12:14:18
status: wip
bead_id: sase-142
---

- **PROMPT:** [prompts/202609/epic_panel_new_node_flicker.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/epic_panel_new_node_flicker.md)
- **BEAD:** [sase-142](https://github.com/sase-org/sase--beads/blob/main/pages/sase-142/README.md)

# Stop The @epic Tribe Panel Flickering When New Nodes Join It

## Problem

Epic `sase-13i` closed three phases (`sase-13i.1`, `sase-13i.2`, `sase-13i.3`) and its
verify phase `sase-13i.4` is still open after two failed verification attempts. The
`2 panels → 1 panel → 2 panels` dip the epic was written against is gone: commit
`a8869ac679` made the load-apply boundary project clan containers and fleet rows before
fold filtering, and the 32-minute soak
(`~/.sase/perf/sase-13i.4-soak4-final-tui_trace.jsonl`, 101,207 records, 234
`agents.refresh_panel_widgets` spans) recorded zero `2 → 1` transitions and zero spans
missing `agent-list-panel-epic`.

The user still sees flickering, and reports it happens **mostly when new nodes have to
be added to the `@epic` tribe panel**. That is exactly the case the soak never
exercised. In those 1,943 seconds the trace contains **zero**
`widget.agent_list.update_list` spans: no panel's row set ever changed, so the soak
measured an idle tab and concluded stability. The published roster did move (24 → 27
agents across the run) without a single panel repaint, which is itself worth explaining,
but the decisive gap is that the "a node joins a panel" path was never observed.

Two defects in that path are confirmed below. A third — what the added row does to the
column's width and to the panel's scroll/highlight position — is strongly implicated but
must be measured, not assumed, which is why the first phase is a measurement harness
rather than a fix.

### Defect 1 (confirmed): one collapsed panel puts every apply on the full-rebuild path

`AgentList.render_collapsed()` (`src/sase/ace/tui/widgets/agent_list.py:257`) resets the
widget's row state and sets `_panel_collapsed = True`, but it never assigns
`_grouping_mode`. Only the rebuild path does
(`src/sase/ace/tui/widgets/_agent_list_build_rebuild.py:147`). A panel that has only
ever painted collapsed therefore keeps the `GroupingMode.STANDARD` default from
`agent_list.py:114`.

`_agent_display_widgets_match_grouping_mode()`
(`src/sase/ace/tui/actions/agents/_display.py:347`) requires _every_ mounted `AgentList`
to match `self._grouping_mode`. Under the host's `by_status` grouping a single collapsed
panel makes it return `False`, and `_try_refresh_agents_display_incremental`
(`_display.py:281`) records `stale_grouping_mode` and falls back to
`_refresh_agents_display(list_changed=True)`.

Reproduced deterministically against the existing `_DisplayDiffApp` harness with three
panels (`@default`, `@epic`, a collapsed `@job`) under `BY_STATUS`:

```
agent-list-panel:       mode=BY_STATUS collapsed=False
agent-list-panel-epic:  mode=BY_STATUS collapsed=False
agent-list-panel-job:   mode=STANDARD  collapsed=True
_agent_display_widgets_match_grouping_mode() -> False
apply with a new @epic node   -> fallbacks: ['stale_grouping_mode']
apply with no change at all   -> fallbacks: ['stale_grouping_mode']
```

The live trace agrees: 232 of the soak's 234 applies recorded `stale_grouping_mode`
(`fleet_refresh` 84, `auto_refresh` 61, `inflight_poll` 45, `watcher` 26, `notification`
9, `tier1_index_revalidate` 6, `dismissed_index_sync` 1). The incremental display path
that `sase-127`, `sase-12p` and `sase-13i.2` were built to reach is effectively dead on
this host, and the epic's "zero full rebuilds on unchanged occupancy keys" criterion
cannot be met while this holds. `sase-13i.4` note #6 raised this as an unproven
hypothesis; the probe above proves it.

The full-rebuild path is not itself a blank — `_panel_content_is_unchanged`
(`src/sase/ace/tui/actions/agents/_display_panel_widgets.py:251`) still suppresses the
repaint of panels whose rows did not move. But it repaints _chrome_ on every panel,
re-runs `_sorted_widget_panel_keys`, `_reorder_agent_list_widgets`,
`_apply_panel_heights` and `_focus_focused_panel_widget` every time, and it means the
one path that could ever add a row cheaply is never taken.

### Defect 2 (confirmed): adding one row clears and re-emits the whole panel

Both surviving paths converge on the same widget call for an added row:

- full rebuild: `_refresh_panel_widgets_impl` →
  `_paint_panel_widget(skip_content=False)` → `widget.update_list(...)`;
- incremental: `_try_refresh_agents_display_incremental_impl` sees
  `diff.has_collection_changes`, puts the panel key in `panel_rebuild_keys`, and
  `_refresh_affected_panel_widgets` → `_paint_panel_widget` → `widget.update_list(...)`.

`update_list` delegates to `build_list`, which begins with `widget.clear_options()` and
wipes `_row_entries`, `_banner_at_row`, `_row_render_ctx`, `_row_tier_styles`,
`_row_by_agent_attempt`, `_row_by_agent_idx` and `_banner_row_by_key`
(`_agent_list_build_rebuild.py:120-129`), then re-emits every option.

Probed on the clean two-panel case, one new `@epic` node produced
`display_panel_rebuild` with `epic.update_list_calls == 1` and
`default.update_list_calls == 0`: sibling panels are already protected, but the panel
the user is watching is blanked and rebuilt for a single added row.

The asymmetry is explicit in the code. `try_remove_rows`
(`src/sase/ace/tui/widgets/_agent_list_build_patching.py:26`) and `patch_row` (`:169`)
exist; there is no insert counterpart, so removals and content changes have a fast path
and additions do not.

### Defect 3 (implicated, must be measured before it is fixed)

Two consequences of the clear-and-re-emit are visible-frame candidates and are the
reason this epic opens with a harness:

- **Out-of-band column resize.** `build_list` ends by recomputing
  `_content_requested_width` and `_refresh_requested_width()` (`agent_list.py:283`)
  _posts_ `AgentList.WidthChanged`. `on_agent_list_width_changed`
  (`src/sase/ace/tui/actions/_event_widgets.py:275`) handles that message in a later
  pump cycle and sets `#agent-list-container.styles.width`. A node whose rendered row is
  wider than the current widest row therefore repaints the rows in one frame and resizes
  the whole agent-list column — and the detail panel beside it — in the next.
  `render_collapsed()` zeroes `_content_requested_width` on the same code path, so a
  panel that collapses can also drive the negotiated width down and back up.
- **Highlight and scroll reset.** `clear_options()` discards the widget's option list
  and the per-row trackers that `update_highlight` and `patch_agent_row` depend on. How
  much of the highlight and scroll position survives the re-emit, and in which frame it
  is restored, is not asserted anywhere today.

Neither claim is stated as established fact. Phase `flicker-frame-harness` measures both
and reports what it finds; phase `row-insert-without-blanking` fixes what the
measurement confirms and records a `PROPOSED FOLLOW-UP:` for anything it refutes.

### Why the previous attempts missed this

Every prior verification asserted on aggregate counters — panel counts, `panel_keys_for`
membership, published row counts, `finalize_plan` outcomes, `active_search` fallbacks.
Those all pass on an idle tab, and they passed in the last soak. Nothing in the repo
asserts what a panel _looks like_ across consecutive refreshes while its row set is
changing, and no deterministic test drives a node arrival into an occupied tribe panel.

## What not to do

- Do not weaken `_agent_display_widgets_match_grouping_mode()` by deleting the check or
  by exempting panels wholesale. It exists so a grouping cycle cannot leave a previous
  mode's banners under the new mode's label. A panel painted collapsed holds no rows and
  no banners, which is why recording the mode it painted under is safe; a panel holding
  rows from the old mode must still force the rebuild.
- Do not debounce, delay, or animate the transition, and do not paint a spinner. This
  epic removes visual transitions; it does not hide them behind latency.
- Do not reach for merged tribe panels (`_agent_panels_grouped`) as the fix. One widget
  hides the symptom and keeps the defect.
- Do not change `../sase-core`. Everything here is Textual presentation state, widget
  row mutation, and TUI loader glue in this repo. Core hardening stays the follow-up it
  already is on `sase-13i.4` note #5.
- Do not add a second renderer or a bespoke screenshot path. Live captures go through
  `sase screenshot`, and golden PNGs stay under `tests/ace/tui/visual/snapshots/png/`
  maintained by `just fix-tui-screenshots`.
- Do not treat the pre-existing red baseline as this epic's work. `sase-13i.4` note #8
  lists it: 20 `no-untyped-def` mypy errors in `src/sase/main/ace_tmux*.py`, symvision
  private-import failures in `src/sase/memory/selector_models.py` and
  `src/sase/main/ace_tmux_support.py`, and six tests that fail identically on clean
  `HEAD`. Verify a failure reproduces on clean `HEAD` before attributing it there, and
  do not repair it here.
- Do not edit `sase/memory/tui_perf.md` inside a phase. The rules this epic touches
  (rule 5 fast path, rule 6 selective updates, rule 12 programmatic `OptionList` guards,
  rule 15 stale imported code) already cover it; the land agent routes any addition
  through the memory-write skill, and phases record `PROPOSED FOLLOW-UP:` notes instead.
- Do not create task beads from inside a phase. Phase workers record
  `PROPOSED FOLLOW-UP:` notes on their own bead.

## Phases

### Phase flicker-frame-harness: Deterministic frame-level repro for a node joining @epic

Build the measurement this epic has been missing, and land it red.

**Paint log.** Add a per-refresh observation record for the Agents tab, enabled in tests
and behind the existing `SASE_TUI_TRACE=1` switch in the app, that captures for every
completed agents-display refresh: the ordered mounted `AgentList` widget ids, each
widget's `id()` object identity, `option_count`, `_panel_collapsed`, `_grouping_mode`,
`_requested_width` and resolved `styles.height`, plus `#agent-list-container`'s resolved
width, the highlighted option index and scroll offset of each panel, and the refresh's
`source` / `display_cost` / `fallback_reason`. Reuse
`src/sase/ace/tui/actions/agents/_refresh_trace.py` for the source/cost fields and
`src/sase/ace/tui/util/trace.py` for emission; do not add a third trace channel.

**Scenario.** Drive a node arrival through the real apply pipeline, not by assigning
`app._agents`. Start from the host's shape: `GroupingMode.BY_STATUS`, split tribe
panels, committed query `NOT machine:apollo`, three panels (`@default`, `@epic` carrying
a clan container with members, and a collapsed sibling), and a selection parked inside
`@epic` below the fold. Then apply, in separate applies: a new member joining the
existing `@epic` clan; a new node that creates a second clan container in `@epic`; and a
node that arrives `STARTING` and becomes rendered on the following apply. Cover the case
where the new row is wider than every existing row, because that is the
width-negotiation trigger.

**Invariants.** Assert over the paint log, not over aggregate counters:

1. No mounted panel's `option_count` ever decreases across an apply whose roster only
   gained rows, and no panel is observed with zero options between two observations that
   both had rows.
2. Every panel widget keeps its `id()` across the arrival; `agent-list-panel-epic` is
   never unmounted and remounted.
3. `#agent-list-container`'s resolved width changes at most once per arrival and moves
   monotonically to its new steady value — no grow-then-shrink and no shrink-then-grow.
4. The selected identity stays selected and `@epic`'s scroll offset still shows it.
5. An apply that changes no rendered row produces no `update_list`, no
   `render_collapsed` and no width or height change on any panel.
6. No panel paints collapsed while its slice has rows, and no panel reports a
   `_grouping_mode` other than the app's.

Where a Textual `Pilot` app is needed for resolved geometry rather than the
`_DisplayDiffApp` fake, use one; `_DisplayDiffApp` stubs `_refresh_agents_display_impl`
(`tests/ace/tui/_agent_display_diff_helpers.py:199`), so it cannot see the full-rebuild
path's painting and must not be used to assert anything about it. Add one live
`sase screenshot` capture recipe to `docs/perf_runbook.md` for the human-verifiable
before/after, and keep it out of the golden lane.

**Keep the tree green.** The harness demonstrates failures that later phases fix, but it
must not turn an unrelated agent's `just check` red in the meantime. Land each invariant
that currently fails as
`@pytest.mark.xfail(strict=True, reason="sase-13i @epic panel flicker: <invariant>")`,
so the suite stays green _and_ the moment a fix lands the strict xfail turns red and
forces the marker's removal. Invariants that already pass land as plain assertions. The
`row-insert-without-blanking` phase removes every marker it fixes; any marker still
standing at the end of the epic is a finding the land agent triages, not a marker to
delete.

**Exit criterion.** The harness lands committed, green, with a strict-xfail marker on
every invariant the current tree violates, and the phase's bead note names each such
invariant, the observation index it fails on, and the code path that produced the
offending frame. A phase that cannot make any invariant fail must say so explicitly and
record what it observed instead — that outcome would refute the diagnosis above and is a
legitimate, valuable result, not a failure of the phase.

### Phase collapsed-panel-mode: Stop collapsed panels forcing a full rebuild on every apply

Fix Defect 1. `_paint_panel_widget` already knows the active `grouping_mode`; pass it
into `render_collapsed(grouping_mode=...)`
(`src/sase/ace/tui/widgets/agent_list.py:257`) and have the widget store it alongside
`_panel_collapsed`, so a collapsed panel reports the mode it actually painted under.
Keep the parameter keyword-only with no default so no caller can silently skip it. Then
let `_panel_content_is_unchanged`
(`src/sase/ace/tui/actions/agents/_display_panel_widgets.py:251`) stop repainting a
collapsed panel whose recorded mode already matches — today the mismatch drops it
through to `render_collapsed()` on every single apply.

Do not touch `_agent_display_widgets_match_grouping_mode()` itself. With the mode
recorded correctly the guard passes on its own, and it must keep failing for a panel
that holds rows built under a previous mode.

Cover with tests:

- an apply with a collapsed sibling panel under `BY_STATUS` records no
  `stale_grouping_mode` fallback and no `display_full_rebuild`;
- cycling `BY_STATUS → STANDARD` while a panel is collapsed still forces the full
  rebuild, because the expanded panels hold the old mode;
- a panel that collapses and then expands paints its rows under the app's current mode;
- an apply that changes nothing calls neither `update_list` nor `render_collapsed` on
  any panel.

Reuse the `_DisplayDiffApp` fake for the fallback-reason assertions, which is what it is
good at, and add the widget-level assertions against a real `AgentList`.

**Exit criterion.** The probe shape from the Problem section — three panels, one
collapsed, `BY_STATUS` — reports `_agent_display_widgets_match_grouping_mode() is True`
and stays on the incremental path across both a changed and an unchanged apply.

### Phase row-insert-without-blanking: Add rows in place and settle the column in one frame

Fix Defect 2, and whatever Phase `flicker-frame-harness` confirmed of Defect 3. Read
that phase's bead notes first; they decide the second half of this phase's scope.

**In-place insert.** Add `try_insert_rows(widget, ...)` to
`src/sase/ace/tui/widgets/_agent_list_build_patching.py` as the mirror of
`try_remove_rows`, returning `False` to fall back to the existing `update_list` rebuild
whenever the fast path is unsafe. Mirror the gates `try_remove_rows` already establishes
rather than inventing new ones: bail on grouping modes outside
`{STANDARD, BY_STATUS, BY_MACHINE}`; bail when the insert would add or remove a
`BY_STATUS` bucket banner or change `rendered_group_keys`; bail on clan containers, clan
members and workflow parents, whose synthetic projection and descendant topology the
caller must rebuild; bail when an existing row's rendered width would have to change,
because column alignment is computed across the emitted rows in `build_list`. Keep the
widget's per-row trackers consistent — `_row_entries`, `_row_by_agent_idx`,
`_row_by_agent_attempt`, `_banner_at_row`, `_banner_row_by_key`, `_row_render_ctx`,
`_row_tier_styles` — since `patch_row`, `try_remove_rows` and `update_highlight` all
read them. Banner chip counts may heal on the next full refresh, exactly as
`try_remove_rows` already documents.

Wire it from the display layer beside the existing remove path in
`src/sase/ace/tui/actions/agents/_display_panel_patches.py`, so
`_try_refresh_agents_display_incremental_impl` attempts an insert before adding the
panel key to `panel_rebuild_keys`. Record a `display_row_insert` cost in
`_refresh_trace.py` alongside `row_patch` and the existing panel costs, and keep the
fallback observable.

**One-frame geometry.** If the harness confirmed the out-of-band column resize, settle
the container width inside the refresh that changed the rows: compute the negotiated
width from the mounted panels' `_requested_width` values at the end of
`_refresh_panel_widgets_impl` and `_refresh_affected_panel_widgets`, in the same call
stack that painted them, and leave `on_agent_list_width_changed` in place for genuinely
out-of-band changes. Do not let a collapsed panel's zeroed `_content_requested_width`
drag the negotiated width below what the expanded panels need. Keep this synchronous and
cheap — it runs on the refresh path, under `tui_perf.md` rules 1, 2 and 6.

If the harness confirmed a highlight or scroll reset that the in-place insert does not
already eliminate, fix it here; if it refuted either sub-claim, implement nothing for it
and record a `PROPOSED FOLLOW-UP:` note saying so.

**Exit criterion.** Every invariant from `flicker-frame-harness` passes as a plain
assertion with its strict-xfail marker removed — never by relaxing the invariant or by
loosening the marker — and a node joining `@epic` records `display_row_insert` rather
than `display_panel_rebuild` for the ordinary non-clan case. Run
`just fix-tui-screenshots` and inspect every golden change individually before accepting
it; generation is not approval.

### Phase verify-new-nodes-on-athena: Prove it on athena with real node arrivals and close sase-13i.4

Re-run the verification `sase-13i.4` asks for, on a **landed** SHA — note #5 lists "ran
on the uncommitted working tree" as one of the two reasons that bead is still open.
Confirm the running TUI's imported SHA before measuring (`tui_perf.md` rule 15); an
editable install keeps executing the code it imported at start.

Soak a separate traced `sase tui` in its own tmux session against the host's real
persisted state — `by_status`, committed query `NOT machine:apollo`, split tribe panels
— with `SASE_TUI_TRACE=1`, for at least 30 minutes. Leave the user's interactive TUI
alone. Use `/sase_monitor` for the wait rather than blocking the turn.

Unlike every previous soak, **make nodes arrive**. Idle observation is what let the last
run report success while recording zero `widget.agent_list.update_list` spans. Launch
several short, cheap agents that land in the `@epic` tribe — staggered, so arrivals are
not batched into one apply — through `/sase_run`, and capture a `sase screenshot` pair
around at least one arrival for a human-verifiable before/after.

Assert from the trace: zero `stale_grouping_mode` fallbacks; zero `display_full_rebuild`
on applies with unchanged occupancy keys; a nonzero count of `display_row_insert`,
proving arrivals were actually observed; no `agents.refresh_panel_widgets` span missing
`agent-list-panel-epic` after it is first seen; no panel-count dip; and the paint-log
invariants from `flicker-frame-harness` holding across every arrival.

Then close `sase-13i.4` with `sase bead close sase-13i.4 --note "<what you verified>"`,
after running `sase bead epic-symbols sase-13i.4` and resolving any leftovers. If a
criterion still does not hold, do not close it: record the measurement and a
`PROPOSED FOLLOW-UP:` note, and say plainly which criterion failed. Note that
`sase-13i.4` is assigned to the `sase-13i.4` agent name; if the close is refused on
assignment grounds, record the evidence as notes and hand the close to this epic's land
agent rather than forcing it.

Carry forward, unresolved by this epic and still worth a note: the 701
`display_full_rebuild` observations in the last soak were dominated by
`stale_grouping_mode`, so re-measure that count after `collapsed-panel-mode` lands and
report whether any non-`stale_grouping_mode` rebuild remains on unchanged occupancy
keys.
