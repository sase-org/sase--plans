---
tier: tale
title: Monitor gear iconography across ACE nodes, containers, and the top bar
goal:
  One reliable amber gear identifies monitor shells at every scale — row, subtree badge,
  and top-bar chip — and bash/python workflow steps share a single command glyph.
size: medium
proposed_by: bbugyi200.athena.048
create_time: 2026-08-16 16:04:52
status: wip
---

# Monitor Gear Iconography: One Glyph, Three Scales

Give monitor shells a single, reliable, beautiful icon that reads the same at every
scale — row mark, subtree badge, and session chip — and collapse the two separate
workflow-step emoji into one command glyph.

## Goal

Four user-requested outcomes, plus the correctness fixes they expose:

1. Agent **families** and **clans** show a monitor icon with a count when their subtree
   contains running monitors.
2. The top-right blue gear stops counting monitors; a second, amber gear beside it
   counts running monitor shells.
3. One monitor icon everywhere, improved over today's `⏱`.
4. Agent **bash** and **python** workflow steps share one glyph instead of `🐚` / `🐍`.

## Design decisions and why

### D1. The monitor icon becomes `⚙` (U+2699 GEAR), amber `#FFAF5F`

Today's monitor glyph is `⏱` (U+23F1). Under this machine's fontconfig,
`fc-match monospace:charset=23F1` resolves to **VL Gothic — a CJK font**. `⏱` is also an
emoji-family codepoint that many terminals render double-width. So the current icon is
off-metric, off-weight, and font-lottery-dependent. That is the "improve the look"
opportunity: it is a real rendering defect, not a taste preference.

`⚙` (U+2699) resolves to **DejaVu Sans Mono**, is `East_Asian_Width=Neutral`,
`rich.cells.cell_len == 1`, and carries default _text_ presentation — so it takes the
Rich foreground color instead of being replaced by a color-emoji bitmap. It renders as
one crisp, styleable cell.

It is also the honest glyph: a monitor **is** a proc shell (`sase monitor start` submits
a proc with `origin="monitor"`), and the top bar already draws procs as a gear. Sharing
the gear across both lanes and separating them by hue states the true relationship —
"gears are background work; blue is ACE's own, amber is your supervised commands." Blue
vs. amber is also the most colorblind-safe hue axis available.

### D2. One glyph at three scales

The same `⚙` is the whole visual grammar:

| Scale         | Where                           | Form              |
| ------------- | ------------------------------- | ----------------- |
| Row mark      | a monitor row in the agent list | `⚙ ` before label |
| Subtree badge | family / clan container row     | `⚙N`              |
| Session chip  | top bar                         | `⚙ N` on amber    |

Collapse a family showing `⚙2`, expand it, and you see exactly two `⚙` rows. The badge
is a promise the expansion keeps. That self-consistency is the point of the redesign.

### D3. Bash and python steps share `❯` (U+276F)

`🐚` and `🐍` are emoji: two cells wide, rendered from a color-emoji font that ignores
the row's Rich style, and the source of the child-row alignment the visual snapshot
suite already pins. `❯` is one cell, resolves to DejaVu Sans Mono, and is _the_ terminal
prompt glyph — it reads as "a command runs here" with no legend needed. The existing
per-step-type colors (bash amber `#FFAF5F`, python green `#87D787`) keep carrying the
type distinction, which is what the user asked for: one icon, color does the rest.

Trade-off, accepted deliberately: `❯` is already `MEMBER_ENTRY_CURSOR_GLYPH` in the
member-roster detail panel. Different widget, different position (leading cursor column
vs. step-prefix), and the roster never lists workflow steps — so the two never appear in
the same list. No other agent-list glyph is `❯`, and the obvious alternatives are worse:
`▸` already means "name-root banner" _in this same widget_, and `»` reads as "more", not
"execute".

### D4. Splitting the top-bar count fixes two real bugs

The blue gear counts `ProcProjection.active_count`, which today includes monitor
proc-shells (monitors submit with `session_id=None`, so `_is_relevant` admits them). The
same count feeds two other places, and in both of them counting monitors is simply
wrong:

- `QuitOptionsModal` renders "**N procs will be stopped.**" A monitor is a _detached_
  supervisor that survives ACE exit — it will not be stopped.
- `running_background_procs()` blocks an ACE self-update restart. A detached monitor has
  no reason to block an ACE restart.

Both are corrected by the same split. This is in scope because it is the same call site
the user asked to change; leaving them would make the two gears disagree with the modals
that read the same projection.

### D5. Monitors still are not agents

The `[S… R… Q…]` status chip counts agents; `concrete_agent_statuses()` already filters
`row.is_monitor`. The new `⚙N` badge sits _beside_ that chip rather than adding a letter
inside it, so the documented invariant — "a family with one agent and one monitor shell
is a one-agent family that ran one command" — stays intact and visible.

## Visual specification

**Agent-list monitor row** (unchanged except the glyph):

```
  └─ ⚙ just check-full (MONITORING)          12:04:31 · 3m12s
```

**Family / clan container row** — badge follows the status chip, precedes the `◆` bead
glyph, zero-suppressed:

```
acme (RUNNING) ×3 [R2 D1] ⚙1 ◆ sase-nb.5 acme
                          ^^^ bold #FFAF5F
```

**Top bar** — two chips, monitor chip immediately right of the proc chip, each hidden at
zero:

```
 Agents  Artifacts  AXE            ⚙ 2  ⚙ 1  ⇧3  …
                                   ^blue ^amber
```

- proc chip: `" ⚙ N "`, `bold #1a1a1a on #48CAE4` (existing, count now excludes
  monitors)
- monitor chip: `" ⚙ N "`, `bold #1a1a1a on #FFAF5F` (new)

**Workflow steps** — one glyph, two colors:

```
  └─ ❯ diff (RUNNING)        ❯ in bold #FFAF5F (bash)
  └─ ❯ setup (DONE)          ❯ in bold #87D787 (python)
```

## Implementation

### Step 1 — Shared monitor constants

`src/sase/monitor_state.py` is the sanctioned lightweight shared monitor module (its
only import is `sase.plan_chain`). The agent-list render path must not import the
monitor supervisor stack — see the comment at `_agent_list_styling.py:105-110` — so the
constants go here, not in `sase/monitor/`.

Add and export in `__all__` (keep the list sorted):

- `MONITOR_GLYPH = "⚙"`
- `MONITOR_GLYPH_COLOR = "#FFAF5F"`
- `MONITOR_TIMEOUT_GLYPH = "⧖"`
- `MONITOR_PROC_ORIGIN = "monitor"`

Move `MONITOR_PROC_ORIGIN` here from `src/sase/monitor/proc_adapter.py:42`: change that
module to import it from `sase.monitor_state` while keeping its existing `__all__` entry
so `sase/monitor/start.py:58` keeps working unchanged. This gives the ACE observer
thread the discriminator without pulling in the supervisor stack.

### Step 2 — Agent-list glyph constants

`src/sase/ace/tui/widgets/_agent_list_styling.py`:

- `_MONITOR_GLYPH = MONITOR_GLYPH` (imported from `sase.monitor_state`); keep
  `_MONITOR_GLYPH_STYLE` / `_MONITOR_ROW_STYLE` as they are.
- Replace the `_STEP_TYPE_GLYPHS` body with a single shared glyph:
  ```python
  # Bash and python steps are both "a command the machine runs"; the
  # per-step-type color carries the distinction, so one prompt chevron is
  # enough — and unlike the old 🐍/🐚 emoji it is one cell wide and takes
  # the row's Rich style.
  _STEP_RUN_GLYPH = "❯"
  _STEP_TYPE_GLYPHS: dict[str, str] = {"python": _STEP_RUN_GLYPH, "bash": _STEP_RUN_GLYPH}
  ```
  Keeping the dict shape means the `_agent_list_render_agent.py:192` call site is
  untouched and `agent`/`parallel`/`prompt_part` steps stay glyph-free as documented.
- Add `_MONITOR_COUNT_GLYPH_STYLE = "bold #FFAF5F"` for the container badge.

### Step 3 — Running-monitor subtree count

`src/sase/ace/tui/models/agent_family_members.py` already owns `agent_row_is_in_flight`.
Add beside it and to `__all__`:

```python
def running_monitor_count(agent: Agent) -> int:
    """Count in-flight monitor shells anywhere beneath one container row."""
```

Requirements:

- Walk `(*row.runtime_children, *row.followup_agents)` recursively. Both lists are
  populated for containers: `_agent_status_apply.py:348` attaches family members to
  `followup_agents`, `_agent_ordering.py:196` attaches them to `runtime_children`, and
  `_agent_tree.py:298` fills a clan container's `runtime_children` with its direct
  members — so a clan's monitors are reached at depth 2.
- Count a row when `row.is_monitor and agent_row_is_in_flight(row)`. Reusing
  `agent_row_is_in_flight` (rather than a bare `monitor_state == "running"` check) means
  the badge and the row agree about what "running" means, including its `stop_time`
  guard.
- Do **not** count the container itself.
- Dedupe by `row.identity` and cycle-guard by `id(row)` — `Agent` is mutable and
  unhashable, and `runtime_children`/`followup_agents` overlap, so both guards are
  required. Mirror the traversal shape of `agent_tribe_summary._descendant_rows`.

### Step 4 — Render the badge

`src/sase/ace/tui/widgets/_agent_list_render_agent.py` — mirror the existing
`parallel_family_counts` threading exactly, so the count is computed once per row and
reused by both the cache key and the renderer:

- Add `running_monitors: int | None = None` to `format_agent_option` and
  `cached_format_agent_option`.
- In `format_agent_option`, resolve it when `None`:
  ```python
  monitor_count = (
      running_monitor_count(agent) if running_monitors is None else running_monitors
  )
  ```
  Only render for container rows:
  `agent.is_clan_container or is_sequential_family_container(agent)` —
  `is_sequential_family_container` also covers legacy roots that predate the explicit
  `is_family_container_row` marker.
- Emit immediately after the `family_chip` block and before the `_BEAD_GLYPH` block:
  ```python
  if monitor_count and is_container_row:
      text.append(" ")
      text.append(f"{_MONITOR_GLYPH}{monitor_count}", style=_MONITOR_COUNT_GLYPH_STYLE)
  ```
- In `cached_format_agent_option`, compute `running_monitor_count(agent)` once and pass
  it to both `agent_render_key(...)` and `format_agent_option(...)`.

`src/sase/ace/tui/widgets/_agent_list_render_cache.py`:

- Add `running_monitors: int | None = None` to `agent_render_key`, resolve it the same
  way, and include the resolved integer in the returned key tuple.

Perf note (TUI rule 6/8): this adds one bounded subtree walk per _container_ row per
cache miss, on the same order as the existing `clan_member_counts` call, and it is
memoized by the same LRU. Nothing new touches disk.

### Step 5 — Carry proc origin into the observer projection

`src/sase/ace/tui/proc_observer.py`:

- `ObservedProc`: add `origin: str = ""`.
- `_store_proc_row`: pass `origin=proc.origin` (`Proc.origin` is a required field on the
  durable row, `src/sase/procs/models.py:36`).
- Add a module-level predicate:
  ```python
  def is_monitor_shell_row(row: ObservedProc) -> bool:
      """Return whether an observed row is a `sase monitor start` proc shell."""
      return row.origin == MONITOR_PROC_ORIGIN
  ```
- `ProcProjection`: add `active_monitor_count: int = 0` and a
  `active_monitor_rows(*, all_sessions: bool = False)` helper filtering `active_rows()`
  through `is_monitor_shell_row`. Keep `active_count` as the **total** so existing
  readers do not silently change meaning; the split happens at each call site.
- Set `active_monitor_count` alongside `active_count` at both `replace(...)` sites
  (`compose_proc_projection` and the snapshot builder around line 453).

Session-local overlay rows (`_session_overlay_rows`) are ACE-owned by construction and
default to `origin=""`, so they stay on the blue side. Correct: they are not monitors.

### Step 6 — Two chips in the top bar

`src/sase/ace/tui/widgets/proc_indicator.py` — factor the chip body and add the sibling
widget in the same module so the gear contract lives in one place:

```python
_GEAR = MONITOR_GLYPH  # same glyph for both lanes; hue is the lane

def _gear_chip(count: int, background: str) -> Text:
    if count <= 0:
        return Text("")
    return Text(f" {_GEAR} {count} ", style=f"bold #1a1a1a on {background}")
```

- `ProcIndicator` keeps `#48CAE4`.
- New `MonitorIndicator(Static)` with the same `set_count` contract, background
  `#FFAF5F`, and a docstring stating it counts running monitor shells (detached
  supervisors that outlive ACE).

Wiring:

- `src/sase/ace/tui/widgets/__init__.py` — add
  `"MonitorIndicator": (".proc_indicator", "MonitorIndicator")` to the lazy map and to
  `__all__`, respecting the keep-sorted lint gate.
- `src/sase/ace/tui/widgets/__init__.pyi` — add the re-export line.
- `src/sase/ace/tui/_app_layout.py` — `yield MonitorIndicator(id="monitor-indicator")`
  immediately after `ProcIndicator`, and add the import.
- `src/sase/ace/tui/styles.tcss` —
  `#monitor-indicator { width: auto; content-align: right middle; }`, placed next to the
  `#proc-indicator` block.

### Step 7 — Feed the split counts

`src/sase/ace/tui/actions/proc_actions.py`, `_update_proc_indicator`: read the effective
projection once and update both widgets — blue gets
`active_count - active_monitor_count`, amber gets `active_monitor_count`. Keep the
existing broad `except Exception: pass` guard so a missing widget never breaks a proc
callback, and make sure a failure to find one widget still lets the other update.

Rename the method to `_update_proc_indicators` only if every call site in this file is
updated; otherwise leave the name and just widen the body.

### Step 8 — Stop monitors from masquerading as blocking procs

- `src/sase/ace/tui/actions/lifecycle.py:194` `_count_running_tasks`: return
  `active_count - active_monitor_count`. The `QuitOptionsModal` line it feeds reads "N
  procs will be stopped", which was never true of a detached monitor supervisor.
- `src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py:317`
  `running_background_procs`: filter monitor shells out of `active_rows()`. A detached
  monitor must not block an ACE self-update restart. Keep the docstring accurate
  ("...that must finish before ACE can restart" — now genuinely true).

### Step 9 — Consistency sweep for the remaining `⏱`

The agent list already badges a monitor timeout with `⧖`
(`_agent_list_render_agent.py:408`) while two other monitor surfaces use `⏱`. Unify on
`MONITOR_TIMEOUT_GLYPH`:

- `src/sase/main/monitor_render.py` — `STATUS_DISPLAY["timeout"]`.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py` —
  `_STATE_DISPLAY["timeout"]`.

Leave `src/sase/ace/tui/widgets/_axe_dashboard_render.py:65` (AXE chop timeout — a
different domain), `src/sase/output.py:204` (a transient CLI spinner, not a node icon),
and `src/sase/chops/report.py` / `docs/axe.md` (a user-authored bullet-glyph allowlist)
alone.

### Step 10 — Help modal legend

`src/sase/ace/tui/modals/help_modal/agents_bindings.py`, "Agent Row Glyphs" section:

- `("⏱", "Monitor shell (label)")` → `("⚙", "Monitor shell (label)")`
- add `("⚙N", "N running monitors in subtree")`
- add `("❯", "Bash / python step")`

Descriptions stay within the 32-character help-modal limit and the 57-character box
width documented in `src/sase/ace/CLAUDE.md`.

### Step 11 — Docs

- `docs/ace.md`
  - ~line 1793: the monitor-shell paragraph — `amber ⏱ glyph` → `amber ⚙ glyph`; keep
    the `⚠` / `⚑` badge sentences.
  - the "Agent Row Glyphs"-adjacent table above it: swap the monitor entry and add the
    `⚙N` subtree badge row.
  - ~line 1840: rewrite the `🐍 / 🐚` paragraph for the single `❯` glyph, keeping the
    colorblind-scanning rationale (now carried by glyph _presence_, with color as the
    type signal) and the unchanged agent/parallel/`prompt_part` sentence.
  - ~line 3255 "Proc Indicator": state that the blue gear counts ACE's own procs and
    **excludes** monitor shells; add a short "Monitor Indicator" paragraph for the amber
    gear, noting it is hidden at zero and that monitors survive ACE exit.
- `docs/monitors.md` ~line 253 "In the ACE TUI": `amber ⏱ glyph` → `amber ⚙ glyph`, and
  add one sentence that a collapsed family or clan carries a `⚙N` badge for its running
  monitors.

## Testing

Update existing assertions:

- `tests/ace/tui/widgets/test_agent_list_monitor_rows.py` — `⏱` → `⚙` (2 assertions).
- `tests/ace/tui/widgets/test_agent_list_provider_emoji_badges.py:182` —
  `"🐚 diff (RUNNING)"` → `"❯ diff (RUNNING)"`.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py:190` — the
  `("sase", "main", "setup", "diff", "🐚 ", "🐍 ")` token tuple becomes `"❯ "`. This
  test also asserts child-connector and bolt column alignment; the step glyph narrowing
  from two cells to one shifts those columns, so re-derive the expectations from the
  actual render rather than hand-editing offsets.

Add:

- `running_monitor_count` unit tests: a family root with one running and one completed
  monitor returns 1; a clan container aggregates across members at depth 2; a cyclic /
  duplicated child graph terminates and counts each monitor once; a plain agent row
  returns 0.
- Agent-list render tests: a family container row with running monitors renders `⚙N`
  after the status chip; a container with only settled monitors renders no badge; a
  non-container row never renders the badge; the badge does not alter the `[S… R…]`
  agent chip.
- `ProcProjection` tests: `active_monitor_count` counts only `origin="monitor"` rows;
  `active_count` still totals; `_store_proc_row` carries `origin` through.
- Indicator tests: blue chip shows `total - monitors`, amber shows monitors, each hides
  at zero (extend or mirror the existing proc-indicator coverage).
- `_count_running_tasks` and `running_background_procs` exclude monitor shells.

PNG goldens: the step-glyph width change moves child-row columns. Regenerate with
`just test-visual -- --sase-update-visual-snapshots`, then re-run `just test-visual`
clean and eyeball `agents_auto_approve_workflow_child_alignment_120x40.png` (and any
other golden the run reports) to confirm the tighter alignment is an improvement, not a
regression.

## Out of scope

- The Admin Center Procs pane and Runners modal row rendering (status glyphs only today;
  no type icon to unify).
- `src/sase/output.py`'s CLI spinner stopwatch and AXE's chop-timeout glyph.
- The pre-existing `_FILE_CHANGE_GLYPH = "✏️"` variation-selector width hazard — worth a
  separate task bead, not this change.
- Any change to which rows _are_ monitors, monitor lifecycle, or proc lifecycle.

## Verification

`just install`, then `just check`. The glyph and widget changes touch the agent-list
render path and the widget registry, so finish with `just check-full` through
`/sase_monitor` (`sase monitor start --command 'just check-full' …` with a `--next`
action) before landing, plus `just test-visual` for the regenerated goldens.

Manual pass in `sase ace`: start a monitor (`sase monitor start --command 'sleep 120'`),
confirm the amber gear appears in the top bar while the blue gear's count drops by one,
confirm the owning family row shows `⚙1` while collapsed and one `⚙` row when expanded,
and confirm a workflow with bash and python steps renders `❯` in amber and green with
the child connectors aligned.
