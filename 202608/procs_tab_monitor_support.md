---
tier: epic
title: First-class sase monitors on the Admin Center Procs tab
goal: "A monitor proc is unmistakable on the Procs tab: it wears the orange monitor
  gear, streams its live output like the agent metadata panel does, names the sase agent
  it belongs to, and opens that agent on the Agents tab with <enter> — while the tab
  header reports running plain procs and running monitors as blue and orange gear
  counts.

  "
phases:
  - id: observer-monitor-fields
    title: Observer carries the monitor's log path and agent name
    depends_on: []
    size: small
    description:
      "observer-monitor-fields: carry log_path and shell_name on ObservedProc and read a
      proc's tail from its own log path, so artifacts-owned monitor logs stream instead
      of reading empty."
  - id: monitor-row-presentation
    title: Monitor rows wear the gear and name their agent
    depends_on:
      - observer-monitor-fields
    size: medium
    description:
      "monitor-row-presentation: mark monitor rows with the orange gear in the list and
      output header, show the monitor's agent name, and render durable tails through the
      shared cached ANSI renderer."
  - id: header-gear-counts
    title: Blue and orange gear counts in the tab header
    depends_on:
      - observer-monitor-fields
    size: small
    description:
      "header-gear-counts: extract the top-bar gear chip into a shared helper and render
      scope-aware running plain-proc and monitor counts in the Procs tab title."
  - id: agent-jump
    title: <enter> opens the monitor's agent on the Agents tab
    depends_on:
      - monitor-row-presentation
    size: medium
    description:
      "agent-jump: add the <enter> keymap that resolves a monitor row to its agent and
      reveals it on the Agents tab, with honest failure messages and help/hints
      coverage."
  - id: docs-and-visual-suite
    title: Documentation and visual snapshot suite
    depends_on:
      - header-gear-counts
      - agent-jump
    size: small
    description:
      "docs-and-visual-suite: rewrite the Procs Tab documentation for monitor rows,
      counts, and the jump key, then rebaseline and extend the Procs PNG goldens."
proposed_by: bbugyi200.athena.04g
status: done
bead_id: sase-o9
create_time: 2026-09-09 19:51:12
---

- **PROMPT:**
  [prompts/202608/procs_tab_monitor_support.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/procs_tab_monitor_support.md)
- **BEAD:**
  [sase-o9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-o9/README.md)

# Plan: First-class sase monitors on the Admin Center Procs tab

## Context and problem

A `sase monitor start` supervisor is a durable proc like any other: `sase.monitor.start`
submits it through `submit_proc_request()` with `proc_id=<monitor_id>`,
`origin="monitor"`, `shell_name=<member agent name>`, and
`log_path=<artifacts_dir>/live_reply.md`. It therefore already appears on the Admin
Center Procs tab (`src/sase/ace/tui/modals/procs_pane*.py`) — but it appears as an
anonymous command row that tells the user nothing about what it really is:

- **A monitor is indistinguishable from a plain proc.** Every other SASE surface marks
  monitors with the orange gear: the top bar (`MonitorIndicator`, `#FFAF5F`), the Agents
  tab (`_agent_list_render_agent.py` prefixes monitor rows with `⚙ `), and the help
  modal legend (`⚙ Monitor shell (label)`). The Procs tab is the one place that drops
  the mark, so nothing signals that the row is a family member with an agent behind it.
- **A monitor's output never streams.** `ObservedProc` drops `Proc.log_path`, and
  `ProcObserver._read_log_tail()` calls `read_proc_log_tail(proc_id, DETAIL_LOG_LINES)`
  without it — so it always reads the store-owned `~/.sase/procs/logs/<proc_id>.log`.
  Monitors submit with `log_owner=ARTIFACTS_LOG_OWNER` and write to
  `<artifacts_dir>/live_reply.md` instead, so that read finds nothing and a running
  monitor shows a permanent `Working...` with an empty body. The agent metadata panel
  renders the same log fine (`_agent_display_step_render._update_monitor_display` →
  `render_axe_output(..., "ansi")`), which is exactly the gap the user hits: the output
  exists, this tab just looks in the wrong place.
- **The tab header hides the split the top bar already makes.** `ProcProjection`
  computes `active_count` and `active_monitor_count`, and the top bar renders them as a
  blue `⚙ N` chip and an orange `⚙ N` chip. The Procs tab title says only
  `Procs · this session  [3 running · 12 done]`, so the tab that exists to explain procs
  is less informative than the one-line indicator above it.
- **The agent behind a monitor is unreachable.** The proc row carries `shell_name` (the
  monitor's member agent name, e.g. `acme--mon`) and its `proc_id` _is_ the `monitor_id`
  that `Agent.monitor_id` records, so the linkage is one hop away — but the pane neither
  shows the name nor offers a way to get to the agent.

This epic is presentation and read-model work on an existing surface. No `sase monitor`
/ `sase proc` CLI behavior, proc lifecycle, kill semantics, or Rust core wire changes;
the sibling `sase-core` repo is not touched.

## Design principles

1. **One monitor visual language, everywhere.** The gear glyph and hue are already
   canonical (`MONITOR_GLYPH = "⚙"`, `MONITOR_GLYPH_COLOR = "#FFAF5F"` in
   `src/sase/monitor_state.py`; `bold #FFAF5F` on the Agents tab). The Procs tab adopts
   them verbatim rather than inventing a monitor treatment, so a monitor looks the same
   in the top bar, the Agents tab, and here.
2. **Hue is the lane; the glyph is the family.** Blue `#48CAE4` means "ACE-owned proc",
   orange `#FFAF5F` means "monitor". The same gear in two hues, already used by
   `ProcIndicator` / `MonitorIndicator`, is the whole legend — the tab header reuses
   that exact chip so the counts read as the same object the user already knows from the
   top bar.
3. **Counts answer the question the visible list asks.** Header counts are computed from
   the pane's current scope rows, so pressing `a` (this session ↔ all sessions) moves
   the counts with the list. A header that disagreed with the rows under it would be
   worse than no header at all.
4. **Never imply data that is not there.** A tail is labeled a tail. A monitor whose
   agent row is not on the Agents tab says so plainly instead of jumping somewhere
   approximate. Both lanes of the gear count are always rendered — a missing chip must
   never be readable as "unknown".
5. **The render path stays pure.** Every new value is either already on the observed row
   or resolved once per list rebuild from in-memory state. No stat, glob, config read,
   or store read per row or per keystroke (`sase/memory/tui_perf.md`, rules 1 and 8).
   Log tails keep being read on the observer thread, for the selected row only.
6. **Reuse the machinery that already works.** The durable tail renders through the same
   cached ANSI renderer as the agent panel; the jump reuses the Agents-tab reveal
   contract (`prepare_agent_navigation_target` / `reveal_agent_navigation_target`) that
   handles folds, panels, and filtered targets, rather than assigning `current_idx` by
   hand.

## Target layout

Procs tab with a running monitor selected:

```
Procs · this session   ⚙ 3  ⚙ 1   [4 running · 12 done]
┌─ Procs ──────────────────────────┐┌─ Output — just check-full ────────────────┐
│ ● ⚙ just check-full      12s ago  ││ ⚙ just check-full  |  0:12  ace·sase#14   │
│      acme--mon · Working...       ││ $ /bin/sh -c just check-full              │
│ ● sync sase-42            3s ago  ││ agent  acme--mon                          │
│      Working...                   ││ Working...                                │
│ ✓ mail sase-41            2m ago  ││ ───────────────────────────────────────── │
│      Mailed to reviewers.         ││ ruff .................. Passed            │
│ ⊘ ⚙ pytest -x           14m ago   ││ mypy ................... Passed           │
│      hotfix--mon-0 · Killed       ││ pytest tests/ace ......                   │
└───────────────────────────────────┘└───────────────────────────────────────────┘
   j/k: move  a: scope  d/D: dismiss  K: kill  e: edit  y: copy  ⏎: agent  ': jump  …
```

Reading order: what scope am I in and how much is running in each lane → which procs,
which of them are monitors and whose → the selected proc's identity, command, agent, and
live output → the keys that apply to the selection.

## Phase designs

### Observer carries the monitor's log path and agent name

`src/sase/ace/tui/proc_observer.py`:

- Add two additive fields to `ObservedProc`, both defaulted so every existing
  constructor (including `src/sase/ace/tui/repro/replay.py` and the test/visual
  fixtures) keeps working unchanged:
  - `log_path: str = ""` — the row's authoritative combined-log path (`Proc.log_path`).
    Store-owned rows repeat the store path; a monitor carries
    `<artifacts_dir>/live_reply.md`.
  - `shell_name: str | None = None` — `Proc.shell_name`, the named proc shell. Document
    on the field that for a monitor row (`origin == MONITOR_PROC_ORIGIN`) this is the
    monitor's **member agent name** (`acme--mon`), which is exactly what `docs/ace.md`
    already states about named proc shells.
- Populate both in `_store_proc_row()`.
- Change `_read_log_tail()` to take the row's log path and forward it:
  `read_proc_log_tail(proc_id, DETAIL_LOG_LINES, log_path=log_path or None)`.
  `read_proc_log_tail` already resolves `None` to the store-owned path and already reads
  the rotated `.1` sibling the same way `read_monitor_log_tail` does, so an
  artifacts-owned monitor log needs no second code path. Pass `proc.log_path` from
  `_store_proc_row()`.
- Add a small public predicate next to the existing `is_monitor_shell_row()` if the
  presentation phases need one for a row's monitor agent name; do not duplicate the
  `origin == MONITOR_PROC_ORIGIN` test in the pane modules.
- Leave `_snapshot_signature()` alone: it already includes `row.output`, so a growing
  monitor tail publishes a new snapshot and the pane repaints on its existing 0.25 s
  tick. Confirm this by test rather than by reasoning.

This phase alone makes a running monitor's output stream on the tab (the pane already
renders `task.output` for store-backed rows), so it is a complete, shippable increment
on its own.

**Tests** (`tests/ace/tui/test_proc_observer.py`): a monitor-shaped `Proc` (origin
`monitor`, `log_owner` artifacts, `log_path` pointing at a temp `live_reply.md`,
`shell_name="acme--mon"`) selected as the detail proc yields an `ObservedProc` whose
`output` is that file's tail, including content only present in the rotated `.1`
sibling; a store-owned row still reads `~/.sase/procs/logs/<id>.log`; `shell_name` and
`log_path` round-trip; appending to the monitor log changes the published snapshot
signature.

### Monitor rows wear the gear and name their agent

All rendering stays in `src/sase/ace/tui/modals/procs_pane_render.py` (plus a new
focused sibling module if that file would grow unwieldy — the pane is already split that
way).

- **Gear marker.** Import `MONITOR_GLYPH` / `MONITOR_GLYPH_COLOR` from
  `sase.monitor_state` and style the glyph `bold #FFAF5F`, byte-identical to
  `_agent_list_styling._MONITOR_GLYPH_STYLE`. In `task_row_label()` the gear sits
  between the status glyph and the label (`● ⚙ just check-full`), the same slot the
  existing `◆ detached` marker uses; in `output_header()` it prefixes the label. A row
  is never both a monitor and a detached row, but the marker helpers must remain
  independent so a future row kind cannot silently swallow either.
- **Agent name.** Monitor rows show the monitor's member agent name:
  - Resolution order, per rebuild, no I/O: (1) the loaded Agent row whose `monitor_id`
    equals the proc row's `proc_id` — use its already-presented `presented_agent_name`;
    (2) fall back to `ObservedProc.shell_name` presented through
    `present_agent_name(name, snapshot)` with **one** `AgentIdentitySnapshot.current()`
    captured per rebuild and threaded in, never called per row; (3) if neither exists,
    render no name rather than a placeholder.
  - Build the `monitor_id -> agent` index once per `_rebuild_list()` pass in
    `procs_pane_selection.py` from `getattr(self.app, "_agents", ())`, and hand it to
    the render helpers as a plain mapping. The render helpers stay pure functions of the
    row plus that mapping so they remain directly unit-testable, and a host app without
    `_agents` (the pane test harness) degrades to the `shell_name` fallback.
  - Placement: list row secondary line becomes `   acme--mon · Working...` — name in
    `#FFAF5F`, separator dim, existing phase/message text unchanged and still dim; when
    there is no secondary text the line carries just the name. Output header gains an
    `agent  acme--mon` line directly under the `$ <command>` line, label dim and name in
    `#FFAF5F`.
- **Durable tail rendering.** Route the store-backed tail through
  `sase.ace.tui.util.axe_log_renderer.render_axe_output(f"proc:{task.proc_id}", text, "ansi")`
  — the renderer the agent metadata panel already uses for `live_reply.md`. It caches on
  `(size, tail hash)`, so an append-only monitor log re-parses only the changed tail
  instead of re-running `Text.from_ansi` over the whole tail every 0.25 s tick, and it
  applies `cap_ansi_output`'s tail-biased byte cap with its own explicit truncation
  notice. Keep `BodyCache` for the in-memory `ProcLogLine` path (local placeholder rows)
  exactly as it is; verify by test that a capped render still ends with the newest bytes
  and that the notice is rendered.
- **Honest tail labeling.** When a store-backed row's tail comes back at the
  `DETAIL_LOG_LINES` cap, prefix the body with a dim italic
  `… showing the last 400 lines …` (built from the constant, not a literal) so a
  streaming view is never mistaken for the whole log.

**Tests** (`tests/ace/tui/test_procs_pane.py` and a render-focused sibling):
`task_row_label()` and `output_header()` emit the gear with the canonical style for a
monitor row and omit it for a plain row; the agent name comes from the Agent index when
present and from `shell_name` when absent, and is absent entirely when neither is; a
monitor row with a live tail renders that tail in the output pane and keeps rendering
appended lines across refresh ticks; the tail notice appears only at the cap; the
detached-row marker and session chip are unchanged.

### Blue and orange gear counts in the tab header

- **Shared chip helper.** Extract the private `_gear_chip()` from
  `src/sase/ace/tui/widgets/proc_indicator.py` into a new dependency-free module
  `src/sase/ace/tui/proc_gear_chips.py` exporting the two lane hues
  (`PROC_GEAR_HUE = "#48CAE4"`, and the monitor hue re-exported from
  `sase.monitor_state.MONITOR_GLYPH_COLOR` rather than re-spelled) and one chip builder.
  `ProcIndicator` and `MonitorIndicator` import it and keep their current rendering
  byte-for-byte — their existing hide-at-zero behavior in the top bar is unchanged.
- **Zero-state variant.** The builder grows an explicit zero treatment used only by the
  pane: a filled chip (`bold #1a1a1a on <hue>`) for a nonzero count and an unfilled
  `dim <hue>` `⚙ 0` for zero. The top bar keeps hiding at zero (it is an ambient badge);
  the Procs tab always renders both lanes, because on the tab that explains procs a
  missing chip would read as "unknown" rather than "none" (principle 4).
- **Title.** `_title_text()` in `src/sase/ace/tui/modals/procs_pane_selection.py`
  returns a Rich `Text`: `Procs · <scope>   ⚙ N  ⚙ M   [R running · D done]`, where `N`
  counts scope rows that are active and **not** monitor shells, `M` counts scope rows
  that are active monitor shells, and the bracketed totals keep their current meaning
  (`N + M == R` by construction; assert that relationship in a test). Counts derive from
  `self._tasks` — already scope filtered — using the existing `is_active()` and
  `is_monitor_shell_row()` predicates, so `a` moves the counts with the list. Update the
  `Label` construction in `procs_pane.py` and `_update_title()` to carry the `Text`
  through.

**Tests** (`tests/ace/tui/test_procs_pane_selection.py` or a sibling): counts split
monitor and non-monitor active rows; zero lanes render the dim variant and nonzero the
filled variant; toggling scope with `a` changes the counts alongside the row set; a
finished monitor is excluded from the running count; the chip builder is shared (the
top-bar widgets still render their existing content and still hide at zero).

### `<enter>` opens the monitor's agent on the Agents tab

- **Action.** Add `action_open_monitor_agent()` to the Procs pane (a new focused sibling
  module, `procs_pane_agent_jump.py`, keeping `procs_pane.py` a facade) plus an
  `("enter", ...)` entry in `ProcsPane.BINDINGS`. Because `TaskList` inherits
  `OptionList`'s `enter -> select` binding, also handle `OptionList.OptionSelected` on
  the pane and route it to the same action (the precedent is
  `agent_run_log_modal.on_option_list_option_selected`); this is what makes both
  `<enter>` and a mouse click work. The action is inert while jump mode (`'`) is active
  and inert — silently — on a non-monitor row.
- **Target resolution.** Scan `getattr(self.app, "_agents", ())` for
  `agent.monitor_id == <row proc_id>` (the proc id _is_ the monitor id) and take that
  agent's `identity`. If there is no match, notify
  `No agent row for <name> on the Agents tab` (naming the resolved agent name when
  known) and do nothing else — the plan deliberately does **not** guess a nearby row
  such as the family root.
- **Navigation.** Dismiss the Admin Center through the modal's own close path so the
  active tab is still remembered (`#` reopens on Procs), then set
  `current_tab = "agents"` and reveal the target through the app's
  `_reveal_agent_row()`, which already expands fold ancestors, picks the owning panel,
  saves a jump anchor, and acknowledges unread state. Sequence the reveal after the
  dismissal (e.g. `call_after_refresh`) so no modal screen is on the stack when the
  Agents tab mutates — `_handle_member_jump_key` deliberately bails while a
  `ModalScreen` is active, and the reveal path should be entered under the same
  condition.
- **Honest failures.** `_reveal_agent_row()` reports failures through
  `_notify_member_reveal_failure()` with member-roster wording ("Member roster changed;
  jump cancelled"), which is wrong for this caller. Add an optional subject argument
  (defaulting to today's wording so no existing call site changes behavior) so the Procs
  jump reports the monitor agent — e.g. "Monitor agent is no longer visible" for the
  not-visible/filtered failures. One notification per failed jump, never two.
- **Discoverability.** The pane hints line gains a conditional `⏎: agent` token, shown
  only when the selected row is a monitor with a resolvable agent — the Admin Center
  analogue of the ace conditional-keymap convention (`src/sase/ace/CLAUDE.md`). Keep the
  token short so the single-line hints bar still fits at 120 columns; verify at the
  narrowest width the tab already supports. Add `Enter — Open the monitor's agent` to
  `ADMIN_CENTER_TASKS_SECTION` in `src/sase/ace/tui/modals/help_modal/binding_common.py`
  (the `?` popup must stay in sync with every option change).

**Tests** (`tests/ace/tui/test_procs_pane.py` / a jump-focused sibling): `<enter>` on a
monitor row whose agent is loaded dismisses the modal, switches to the Agents tab, and
lands on that agent; `<enter>` on a monitor row with no matching agent notifies once and
leaves the modal open; `<enter>` on a plain proc row does nothing; `<enter>` during jump
mode is consumed by jump mode; the click path (`OptionSelected`) reaches the same
action; the hints line shows the token only for monitor rows with a resolvable agent;
the shared reveal-failure wording change does not alter the existing member-jump
notifications.

### Documentation and visual snapshot suite

- **`docs/ace.md`, "Procs Tab".** Add a short _Monitors on this tab_ subsection: monitor
  rows carry the orange `⚙` (same mark as the Agents tab and the top bar), name their
  member agent, stream `live_reply.md` live, and open that agent with `<enter>`; monitor
  procs are unattributed, so they are visible in both scopes; `K` on a monitor stops the
  supervisor through the proc-shell stop path, which settles the family and runs any
  `--next` action. Extend the status icon table with the gear, and document the header
  counts (blue = running procs excluding monitors, orange = running monitors, both
  scoped to the tab's current scope). While editing this section, correct two drifted
  statements it already contains: the tab is opened with `3`, not `5` (`_TAB_SPECS` in
  `config_center_catalog.py`), and running procs refresh on the pane's 0.25 s tick, not
  "every second".
- **PNG goldens** (`tests/ace/tui/visual/`): extend
  `_ace_config_center_procs_helpers.py` with a running monitor row (orange gear, agent
  name, streaming tail) and a finished monitor row, then rebaseline
  `config_center_procs_tab_120x40.png` and add one monitor-focused golden capturing the
  header chips, the marked rows, and the streamed output pane. Run `just test-visual`
  with `--sase-update-visual-snapshots` and inspect each actual/expected/diff artifact
  under `.pytest_cache/sase-visual/` before accepting — snapshot updates are deliberate,
  never blind. `config_center_home_resume_procs_*.png` show the landing page and should
  not change; if they do, investigate rather than rebaseline.
- **Final consistency pass** across the tab at 120 and ~90 columns: gear hue identical
  to the Agents tab, chip styling identical to the top bar, `·` separators consistent,
  hints line unclipped, no meaning carried by color alone (the gear glyph and the agent
  name always accompany the hue).

## Testing

- Every phase runs `just check` before handing off; the combined tree runs
  `just check-full` through `/sase_monitor` (never inline) before landing, per the
  repo's two-speed verification rule.
- Unit and pilot coverage is listed per phase above. Coverage must include the degraded
  paths, not only the happy ones: no `_agents` on the host app, a monitor row with no
  `shell_name`, a monitor whose log file does not exist yet, and a monitor whose agent
  is filtered off the Agents tab.
- Visual: `just test-visual` in the final phase, with each accepted diff inspected.
- Perf sanity: confirm no new I/O on a render or keystroke path (the tail read stays on
  the observer thread, for the selected row only), and spot-check tab interaction with
  `SASE_TUI_PERF=1` that j/k on the Procs tab stays responsive with a large streaming
  monitor selected.

## Performance and architecture constraints

- Presentation and read-model only. No `sase-core` / `sase_core_rs` change, no proc wire
  change, no new store fields: every value this epic surfaces is already on the durable
  `Proc` row (`log_path`, `shell_name`, `origin`) or already in the ACE agent model
  (`Agent.monitor_id`).
- The observer keeps doing all disk work on its daemon thread and keeps reading a log
  tail only for the selected detail row. The pane's existing 0.25 s tick, `BodyCache`,
  selection guard, and jump-hint invalidation paths are reused as-is; no new refresh
  code path is introduced.
- Agent lookup is an in-memory scan built once per list rebuild — never per row, never
  per keystroke, never on the observer thread.
- No feature flag. Each phase lands a complete, enabled slice of presentation for an
  existing surface, and no phase leaves a half-built user-reaching path behind a
  default-off branch; `sase/memory/sase_flags.md` reserves flags for behavior that
  reaches users before it is ready, which is not the case here.

## Risks and mitigations

- **`shell_name` is not monitor-specific.** Any `sase proc run --shell NAME` row carries
  one. Mitigated by gating every monitor treatment on `origin == MONITOR_PROC_ORIGIN`
  via the existing `is_monitor_shell_row()` predicate, never on the presence of
  `shell_name`.
- **Reading an artifacts-owned log path.** The path comes from the durable proc row this
  process already trusts, and `read_proc_log_tail` performs a bounded tail read; no
  deletion path changes (`delete_proc_logs` still refuses non-store-owned logs), so a
  monitor's `live_reply.md` remains owned by its agent family.
- **Jump lands nowhere.** The Agents tab may filter the monitor row out. Mitigated by
  reusing the reveal contract that reports exactly why, plus a single honest
  notification and no fallback guess.
- **Snapshot churn.** The header change alters the one existing Procs golden. The final
  phase owns that rebaseline explicitly and inspects the diff artifacts.
- **Hints line overflow.** The conditional `⏎: agent` token lengthens an already long
  single line. Mitigated by keeping the token minimal and verifying rendering at the
  narrow width in the final pass.
