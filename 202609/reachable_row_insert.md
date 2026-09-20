---
tier: epic
title: Make the Agents-tab row insert reachable for real arrivals and close sase-13i.4
parent_bead: sase-142
goal: "A real agent node arriving in the @epic tribe panel on athena records
  display_row_insert, not display_panel_rebuild, and an apply that changes no rendered
  row in a panel repaints nothing in that panel. Both claims are proved by the existing
  frame-level harness extended to the row shapes real arrivals actually have, and
  re-proved by a landed-SHA soak on athena with deliberately created nodes, which closes
  sase-13i.4.

  "
phases:
  - id: panel-scoped-rebuild-gates
    title: Decide rebuild scope per panel instead of per roster
    depends_on: []
    size: medium
    description: "panel-scoped-rebuild-gates: make the three whole-roster predicates in
      _try_refresh_agents_display_incremental attribute a rebuild to the panel keys they
      concern instead of rebuilding every panel, keeping the reason observable per panel
      and covering it with a sibling-panel-change harness scenario.

      "
  - id: reachable-insert-for-real-rows
    title: Admit the row shapes real arrivals actually have
    depends_on:
      - panel-scoped-rebuild-gates
    size: medium
    description: "reachable-insert-for-real-rows: decide on evidence whether a workflow
      family that arrives whole can be inserted in place, implement that answer under
      the existing insert-equals-rebuild contract, and either prove the shape reaches
      display_row_insert or name the topology invariant that blocks it.

      "
  - id: quiet-applies
    title: Stop applies that change no rendered row from repainting
    depends_on:
      - panel-scoped-rebuild-gates
    size: medium
    description: "quiet-applies: scope each panel's paint key to its own rows so a
      fold-count change in one panel stops repainting the others, settle the agent-list
      column in-frame when a removal collapses a panel, and re-measure the residual
      same-occupancy rebuilds by reason.

      "
  - id: reverify-arrivals-on-athena
    title: Re-soak with real arrivals, prove the insert, and close sase-13i.4
    depends_on:
      - reachable-insert-for-real-rows
      - quiet-applies
    size: medium
    description:
      "reverify-arrivals-on-athena: re-run the landed-SHA athena soak with probes that
      can actually reach the insert, assert a nonzero display_row_insert and the rest of
      the trace criteria, close sase-13i.4 with that evidence, and route the
      consolidated tui_perf.md update through the memory-write skill."
proposed_by: bbugyi200.athena.sase-142.land
create_time: 2026-09-20 17:09:24
status: wip
---

- **PROMPT:**
  [prompts/202609/reachable_row_insert.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/reachable_row_insert.md)
- **PARENT:**
  [202609/epic_panel_new_node_flicker.md](https://github.com/sase-org/sase--plans/blob/main/202609/epic_panel_new_node_flicker.md)

# Make The Agents-Tab Row Insert Reachable For Real Arrivals And Close sase-13i.4

## Problem

Epic `sase-142` landed three commits that fixed the @epic panel flicker the user
reported, and a landed-SHA soak on athena confirmed the user-visible claim: zero
`stale_grouping_mode` fallbacks where there had been 232 of 234 applies, zero
panel-count dips, `agent-list-panel-epic` present in all 52 `refresh_panel_widgets`
spans, and the frame-level invariants holding across all 20 arrival windows (7 of them
@epic). The panel is no longer blanked, the highlight and scroll survive, and the column
no longer resizes a frame after the rows.

But the mechanism that was supposed to produce that result never ran. In 2284 seconds
with 7 real @epic arrivals, `display_row_insert` recorded **0 successes and 0
attempts**. All 7 arrivals, and all 50 non-startup rebuilds, were
`display_full_rebuild`. `sase-142` therefore shipped `AgentList.try_insert_rows` —
roughly 320 lines of widget code behind 457 lines of tests — as a path that is
unreachable in production, and `sase-13i.4`, which `sase-142` phase 4 was titled to
close, is still open. This epic finishes that.

Three separate gates stand between a real arrival and the insert.

### Gate 1 (confirmed): rebuild scope is decided for the whole roster, not per panel

`_try_refresh_agents_display_incremental`
(`src/sase/ace/tui/actions/agents/_display.py`, ~lines 297-308) computes one diff over
the entire published roster and returns `False` — sending every panel down
`_refresh_agents_display(list_changed=True)` — as soon as any of three whole-roster
predicates fires:

```
if diff.duplicate_identity:                      -> panel_membership_change
if BY_STATUS and _by_status_display_membership_changed(previous)  -> status_membership_change
if diff_touches_workflow_tree(diff, previous, self._agents):      -> workflow_tree_change
```

Only after all three pass does `_try_refresh_agents_display_incremental_impl` reach
`PanelPatchMixin._try_insert_panel_rows` and the per-panel insert. On athena those three
accounted for 50 of 50 non-startup rebuilds (`status_membership_change` 22,
`workflow_tree_change` 21, `panel_membership_change` 7). A status change in `@default`
rebuilds `@epic`, even though nothing in `@epic` moved.

`_by_status_display_membership_changed` is the clearest case: it compares
`grouping_tree_keys_for_display` over the _whole_ roster, so a new BY_STATUS bucket
appearing in one panel invalidates every panel. The per-panel insert already re-derives
and re-checks the banner topology it needs (`try_insert_rows` declines on a changed
banner/spacer sequence), so the global check is doing work the panel-level gate repeats.

This also explains the carry-forward measurement `sase-142` was asked to report: 21 of
50 rebuilds had **unchanged panel occupancy** and a non-`stale_grouping_mode` reason.
Those are exactly the applies where one panel's structure changed and the others were
rebuilt for it.

### Gate 2 (confirmed): real arrivals are workflow families, which the insert declines

Agents launched through `sase run` on this host carry a `#git` / `#gh` workflow block,
so they arrive as `AgentType.WORKFLOW` families and render as a parent row plus
descendants (observed: `home (DONE) x5`). `try_insert_rows` bails on clan containers,
clan members and workflow parents, and `_is_plain_leaf_row` declines them, because their
synthetic projection and descendant topology are the caller's to rebuild.

That gate is correct as written for a _changing_ tree. It is not obviously correct for a
family that arrives whole and touches no existing row. Phase
`reachable-insert-for-real-rows` decides that question on evidence rather than assuming
either answer.

### Gate 3 (confirmed): the paint key folds global state into every panel

`_panel_paint_key` (`src/sase/ace/tui/actions/agents/_display_panel_widgets.py`) folds
the global `fold_counts` — and the visible / fully-expanded parent key sets — into every
panel's key. An arrival that changes fold counts in `@epic` therefore invalidates
`@default`'s paint key and re-runs `update_list` on a panel whose rows did not change.
Measured at frame level by `sase-142.1` (note #2) in the `clan_member` (idx 3) and
`second_clan` (idx 6) windows. The in-place insert bails on clan rows, so `sase-142.3`
could not remove it.

## What not to do

- Do not delete a gate to reach the insert. Each of the three predicates exists because
  some real topology change makes a stale panel wrong. The work is to evaluate each one
  against the panel it actually concerns, not to stop evaluating it. A change that makes
  `display_row_insert` nonzero by admitting an unsafe insert has failed, not succeeded.
- Do not weaken `_agent_display_widgets_match_grouping_mode()`. `sase-142.2` fixed its
  input, not the guard; a panel holding rows built under a previous mode must still
  force the rebuild.
- Do not relax a harness invariant or loosen a marker to make a phase pass.
  `tests/ace/tui/test_epic_panel_arrival_frames.py` currently holds all six invariants
  as plain assertions with no strict-xfail markers remaining, and it must still do so at
  the end of every phase here. A phase that needs a new invariant adds one.
- Do not restate the epic's criterion as "full rebuild with stable widgets" without
  first doing the work in `panel-scoped-rebuild-gates` and
  `reachable-insert-for-real-rows` and reporting what is still unreachable and why. That
  restatement is a legitimate outcome, but only as a measured conclusion; it is not a
  shortcut past the measurement.
- Do not change `../sase-core`. This is Textual presentation state, widget row mutation,
  and TUI loader glue.
- Do not treat the repo's pre-existing red baseline as this epic's work. Verify a
  failure reproduces on clean `HEAD` before attributing it, and do not repair it here.
  Known at the time of writing: `test_capacity_gate_to_admission` (x2, `sase-13q`),
  `test_lazy_tier2_reconcile_apply::...rearms` (`sase-13n`), `test_app_import_budget`
  (`sase-13r`).
- Do not run `just check-full`. `just check` is the verification recipe; a `just check`
  pass with a `just check-full` failure is a test-infrastructure bug to file, not
  remaining work.
- Do not edit `sase/memory/tui_perf.md` inside phases 1-3. Phase
  `reverify-arrivals-on-athena` routes the single consolidated update through the
  memory-write skill, and earlier phases record `PROPOSED FOLLOW-UP:` notes instead.
- Do not create task beads from inside a phase. Phase workers record
  `PROPOSED FOLLOW-UP:` notes on their own phase bead.

## Phases

### Phase panel-scoped-rebuild-gates: Decide rebuild scope per panel instead of per roster

Make the three whole-roster predicates in `_try_refresh_agents_display_incremental`
(`_display.py` ~297-308) answer "which panels must rebuild", not "must everything
rebuild".

Keep the cheap global short-circuit: when no predicate fires, the path is unchanged.
When one does, attribute it to the panel keys it concerns, put only those keys in
`panel_rebuild_keys`, and let `_refresh_affected_panel_widgets` paint the rest
incrementally. Derive the attribution from the panel index that
`_agent_panel_index()`/`_panel_keys_per_agent()` already build; do not add a second
partition of the roster.

- `diff.duplicate_identity` -> the panel(s) holding the duplicated identity.
- `_by_status_display_membership_changed` -> restrict the
  `grouping_tree_keys_for_display` comparison to each panel's own slice, so a new bucket
  in one panel does not invalidate the others. The identity loop already skips
  identities with no rendered row (`sase-142.3`); keep that.
- `diff_touches_workflow_tree` -> the panel(s) whose workflow tree actually changed.

Record the per-panel attribution in the existing `_refresh_trace.py` fallback reasons so
a rebuild still names the panel and the reason it was rebuilt; a partial rebuild must be
as observable as the global one was.

Extend `tests/ace/tui/_epic_arrival_frames.py` with a scenario the current harness has
no coverage for: a structural change in a sibling panel (a `@default` status-bucket
move) concurrent with no change at all in `@epic`. The invariant is that `@epic` records
no `update_list` and no `render_collapsed` in that window.

**Exit criterion.** An apply whose only structural change is confined to one panel
rebuilds that panel and leaves every other panel's widget untouched, asserted at frame
level in the harness; all six existing invariants still pass as plain assertions; and
the phase's bead note reports, per predicate, which panel set it now attributes to and
which case (if any) still has to fall back globally.

### Phase reachable-insert-for-real-rows: Admit the row shapes real arrivals actually have

Answer, on evidence, whether a workflow family that arrives whole can be inserted in
place, and implement the answer.

Start from the measurement: on athena every `sase run` arrival was an
`AgentType.WORKFLOW` family rendering as a parent plus descendants, and
`_is_plain_leaf_row` declined it. Determine whether an arriving family whose rows are
all new — no existing row changes position, no existing parent gains or loses a child —
is separable from the tree mutation the existing gate is written against.

If it is, extend `try_insert_rows` to admit that shape, mirroring the discipline the
existing gates already establish: decline before mutating; leave `_row_entries`,
`_row_by_agent_idx`, `_row_by_agent_attempt`, `_banner_at_row`, `_banner_row_by_key`,
`_row_render_ctx` and `_row_tier_styles` exactly as a full rebuild would; renumber
option ids; re-render banner chips by swapping, never mutating, the shared cached
banners. The existing equivalence test in
`tests/ace/tui/widgets/test_agent_list_try_insert_rows.py` — an insert leaves the widget
identical to a rebuild — is the contract, and every new shape must be added to it.

If it is not separable, say so precisely and record why, then state which arrival shapes
_can_ reach the insert on this host and how `reverify-arrivals-on-athena` should produce
one. That is a legitimate result; a phase note that names the exact topology invariant
that blocks it is worth more than an unsafe insert.

Also confirm what `sase-142.3` left implicit: `display_row_insert` declines record the
gate as `fallback_reason`, so after this phase a soak can tally _why_ an arrival still
rebuilt. Add the fallback tally recipe to `docs/perf_runbook.md` beside the existing
Agents-tab paint-frames jq recipe.

**Exit criterion.** Either the harness records `display_row_insert` for an arriving
workflow family with every invariant holding as a plain assertion and the
insert-equals-rebuild equivalence test covering that shape, or the phase's bead note
states the specific topology invariant that makes it unsafe, names the arrival shapes
that do reach the insert, and specifies the probe shape the verification phase must
launch.

### Phase quiet-applies: Stop applies that change no rendered row from repainting

Close the "an apply that changes nothing repaints nothing" residue that `sase-142` left
measured but unfixed.

**Scope the paint key to its own panel.** `_panel_paint_key`
(`_display_panel_widgets.py`) folds global `fold_counts` and the visible /
fully-expanded parent key sets into every panel's key, so an arrival that changes fold
counts in `@epic` re-runs `update_list` on `@default`. Restrict each panel's key to the
fold counts and parent key sets of the rows that panel actually holds. Proposed by
`sase-142.1` note #2, measured there at frames idx 3 and idx 6.

**Settle the column when a removal collapses a panel.** `_try_remove_agent_rows`
(`_display_panel_patches.py`, ~line 253) calls `render_collapsed()` when a tribe's last
visible rows go, which zeroes that panel's `_content_requested_width`, and then leaves
the column resize to the `WidthChanged` handler a pump cycle later — the same two-step
shape `sase-142.3` removed for arrivals, on the path it did not cover. Call
`_settle_agent_list_container_width` there. Note this interacts with `8ae9ac1eab` (a
retired tribe key requests `list_changed=True`, which already settles in-frame); the gap
is the fast path that keeps the panel mounted. Proposed by `sase-142.3` note #4, and
explicitly unmeasured — the harness has no removal scenario, so add one before fixing
it.

**Re-measure the residue.** `sase-142.4` recorded 21 of 50 non-startup rebuilds on
unchanged panel occupancy. Phase `panel-scoped-rebuild-gates` should remove most of
them. Re-run the analyzer over a fresh local capture and report what remains, per
reason, so `reverify-arrivals-on-athena` has a prediction to test rather than a hope.

**Exit criterion.** A removal that collapses a panel moves the column width in the same
frame as the rows, asserted by a new harness removal scenario; an arrival in one panel
records no `update_list` and no `render_collapsed` on any sibling panel whose rows did
not change; and the phase's bead note reports the predicted post-fix count of
same-occupancy rebuilds by reason.

### Phase reverify-arrivals-on-athena: Re-soak with real arrivals, prove the insert, and close sase-13i.4

Repeat the `sase-142.4` verification on a landed SHA with the two defects that spoiled
it removed.

Reuse what already exists rather than rebuilding it: the pinned-clone soak method, the
separate traced `sase tui` in its own tmux session under the host's real persisted state
(`by_status`, committed query `NOT machine:apollo`, split tribe panels,
`SASE_TUI_TRACE=1`), and the stdlib analyzer that inlines the CI harness invariant
checkers. Those are recorded on `sase-142.4` note #1 with their paths. Confirm the
running TUI's imported SHA before measuring (`tui_perf.md` rule 15), leave the user's
interactive TUI alone, and use `/sase_monitor` for the wait.

Two things `sase-142.4` could not do, which this phase must:

- **Launch probes that can actually reach the insert.** Every probe it launched picked
  up `#git:home` and rendered as a workflow family. Launch the arrival shape phase
  `reachable-insert-for-real-rows` named as reachable — a plain-leaf probe with no
  workflow block if families stayed unreachable, families if they did not — staggered so
  arrivals are not batched into one apply.
- **Not lose probes to the admission bug.** `sase-142.4` had only 2 of 6 probes run,
  because `%w(time=...)` waits never elapse after the coordinator process changes
  (recorded as a DISCOVERED ISSUE on `sase-s6`, which owns that coordinator). Do not
  depend on time-based staggering unless that is fixed by then; stagger by launching
  separately instead, and confirm each probe actually started before relying on it.

Assert from the trace: a nonzero `display_row_insert` count on real @epic arrivals; zero
`stale_grouping_mode`; no `agents.refresh_panel_widgets` span missing
`agent-list-panel-epic`; no panel-count dip; the paint-log invariants holding across
every arrival; and the same-occupancy rebuild count matching what `quiet-applies`
predicted, with every remainder explained by reason. Capture a `sase screenshot` pair
around at least one arrival.

Then close `sase-13i.4` with the evidence, after running
`sase bead epic-symbols sase-13i.4` and resolving any leftovers. If a criterion still
does not hold, do not close it: record the measurement and say plainly which criterion
failed.

Finally, route the one consolidated `sase/memory/tui_perf.md` update through the
memory-write skill, covering what `sase-142` and this epic together established: rule 6
should name `try_insert_rows` beside `patch_row` / `try_remove_rows`; a `STARTING` agent
that becomes rendered is an arrival (a row insert), not a `BY_STATUS` bucket move; the
agent-list column settles in-frame via `_settle_agent_list_container_width`, with the
`WidthChanged` handler kept only for out-of-band changes; and rebuild scope is decided
per panel, so one panel's structural change no longer rebuilds its siblings. This was
proposed by `sase-142.3` note #1 and deliberately deferred to here so the note describes
the finished behavior rather than an intermediate state.

**Exit criterion.** A landed-SHA soak on athena of at least 30 minutes with deliberately
created @epic arrivals records a nonzero `display_row_insert`, every assertion above
holds or is reported as failed with its measurement, and `sase-13i.4` is closed with
that evidence or left open with a named failing criterion.
