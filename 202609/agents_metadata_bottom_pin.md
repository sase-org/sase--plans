---
tier: tale
title:
  Pin the Agents metadata panel to the bottom after G instead of scrolling to it once
goal:
  Pressing `G` on the Agents tab while the metadata panel is the active detail surface
  keeps that panel's viewport showing the end of the document as async enrichment,
  slow-tool ticks, and auto-refresh repaint it, instead of drifting upward as the
  document grows and jumping when a shorter repaint clamps the scroll offset; the pin
  releases on the next deliberate move away and on a document identity change.
size: medium
proposed_by: bbugyi200.athena.0k9
create_time: 2026-09-12 12:10:24
status: wip
---

# Plan: Pin The Agents Metadata Panel To The Bottom After `G`

## Context

On the Agents tab, `G` (`app.scroll_to_bottom` in `src/sase/default_config.yml:466`,
bound at `src/sase/ace/tui/bindings.py:239`) is supposed to take the user to the end of
the active detail surface. When that surface is the metadata panel
(`#agent-prompt-scroll` / `#agent-prompt-panel`), the view does not stay there: over the
next few seconds the panel drifts away from the bottom and occasionally jumps far up the
document.

### Root cause

**`G` on the Agents tab is a one-shot absolute scroll with no follow state, and the
metadata document is repainted with a different height many times per second after the
keypress.**

`action_scroll_to_bottom` (`src/sase/ace/tui/actions/navigation/_basic.py:419`) resolves
the active container and scrolls it once:

```python
elif self.current_tab == "agents":
    scroll_id = self._get_agent_detail_scroll_id()
    scroll_container = self.query_one(scroll_id, VerticalScroll)
    if scroll_id == "#agent-prompt-scroll":
        panel = scroll_container.query_one("#agent-prompt-panel", AgentPromptPanel)
        target = max(0, int(scroll_container.max_scroll_y) - panel.section_layout_reserve)
        scroll_container.scroll_to(y=target, animate=False, immediate=True)
    else:
        scroll_container.scroll_end(animate=False)
```

That writes one absolute `scroll_y` and then forgets about it. Compare the Axe tab,
which has had follow state since it was written: `action_scroll_to_bottom` sets
`self._axe_pinned_to_bottom = True` (`_basic.py:431`), and every Axe render re-applies
it (`src/sase/ace/tui/actions/axe_display/_render.py:295-307`). `_axe_pinned_to_bottom`
is the only pin flag in the tree — the Agents tab has no equivalent.

Meanwhile the metadata document is anything but static. Every one of these repaints the
panel through the single `AgentPromptPanel.update()` choke point
(`src/sase/ace/tui/widgets/prompt_panel/__init__.py:79`), with a document whose height
differs from the last one:

- **Streamed header lanes.** `_build_detail_header_summary` resolves
  `LANE_RESOLUTION_BATCHES` cheapest-first and hands each intermediate batch to the main
  thread with `call_from_thread`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_async.py:145-151`), so the SASE
  CONTEXT section grows batch by batch after the first paint
  (`_agent_display_async_agent.py:286`).
- **Enrichment workers.** Bead display, linked deltas, clan sections, and tribe sections
  each land separately in `on_worker_state_changed` (`_agent_display_async.py:172-181`),
  each adding or resizing a section.
- **The slow-tool render tick.** A 5-second `set_interval` re-renders the whole document
  from cache while any slow tool source is pending (`prompt_panel/__init__.py:29`,
  `:333-342` → `refresh_slow_tool_metadata_from_cache`).
- **Auto-refresh.** The periodic agents refresh runs `_apply_agent_detail_update` →
  `agent_detail.update_display(...)`
  (`src/sase/ace/tui/actions/agents/_display_detail_render.py:262`), re-rendering
  elapsed times, wait status, tool call rows, and reply bodies.

Textual keeps `scroll_y` as an absolute offset across all of that. It has exactly two
behaviors that matter here, and both are wrong for a user who asked to be at the end:

1. **Content grows → the viewport silently drifts up.** Nothing moves `scroll_y`, so
   every line appended below the fold pushes the real bottom further past the viewport.
   The new content the user pressed `G` to watch is the content they stop seeing.
2. **Content shrinks → `scroll_y` is clamped and never restored.** `Widget.scroll_y` is
   a reactive validated by `validate_scroll_y`, which clamps to the current
   `max_scroll_y`. A repaint that renders a shorter document (a resolved lane replacing
   a longer pending block, a fold-level summary replacing a roster, or the cheap
   header-only paint from `update_display_immediate` → `update_header_only`,
   `widgets/agent_detail.py:156`) drops `scroll_y` to the new, much smaller maximum. The
   next repaint restores the document height but not the offset, so the panel is left
   parked high in the document. This is the visible "jump"; the drift in (1) is what
   makes it recur.

Both were reproduced on a clean tree at `df33453b7`, driving the mounted harness from
`tests/ace/tui/widgets/test_prompt_panel_section_navigation_actions.py` in a 50x16
pilot: press `G`, then repaint the panel with a taller document, a much shorter one, and
the taller one again.

| step                     | `scroll_y` | `max_scroll_y` | lines adrift |
| ------------------------ | ---------- | -------------- | ------------ |
| after `G`                | 7          | 7              | 0            |
| +10 lines of new content | 7          | 17             | 10           |
| shorter repaint (clamp)  | 0          | 0              | -            |
| full document returns    | 0          | 17             | 17           |

The third and fourth rows are the jump: one shorter repaint clamps the offset to 0, and
when the document comes back the viewport is stranded at the very top of it.

Two secondary defects in the same code path make even the initial keypress unreliable,
and both must be fixed for a pin to hold:

3. **`G` reads a stale `max_scroll_y`.** The `#agent-prompt-scroll` branch above passes
   `immediate=True`, which deliberately skips the deferral that plain `scroll_end` uses.
   Textual's own `Widget.scroll_end` wraps its `_scroll_to` in `call_after_refresh`
   precisely because, in its words, "we need the refresh to work out and then figure out
   how big things are" before reading `max_scroll_y`. When `G` arrives in the same frame
   as a repaint — the common case on a live agent — the target is computed from the
   pre-layout height and lands short of the bottom.
4. **The section-navigation reserve moves the bottom.** Once `ctrl+j`/`ctrl+k` has
   called `enable_section_layout_reserve()` (`prompt_panel/__init__.py:148`),
   `get_content_height` appends a phantom trailing extent so the final section title can
   be top-aligned (`:125-146`). That reserve is recomputed from the current anchors on
   every layout, so `max_scroll_y` and the real end of the document move independently.
   Any pin must target `max_scroll_y - section_layout_reserve`, not `max_scroll_y`.

Point (4) is also why Textual's built-in `Widget.anchor()` cannot be used as-is: the
compositor pins an anchored widget to `virtual_size.height - container height`
(`textual/_compositor.py:608-616`), which includes the phantom reserve, so an anchored
metadata panel would sit in blank space below the last line.

### Why the widget, not the app, owns the pin

The Axe pin lives on the app because Axe has exactly one render function to re-apply it
from. The metadata panel has no such place: over thirty call sites across
`widgets/prompt_panel/_agent_display*.py` call `self.update(...)`, and the async workers
repaint without the app knowing. They all funnel through `AgentPromptPanel.update()`,
which is also the only code that knows the content actually changed (it early-returns on
an equal content digest, `prompt_panel/__init__.py:86-97`) and the only owner of
`section_layout_reserve`. That makes the widget the single correct place to re-apply,
and it means idle ticks that render an identical document cost nothing extra.

## Invariants

- **Idle repaints stay free.** The digest early-return in `AgentPromptPanel.update()`
  must keep returning before any pin work. An auto-refresh tick that renders an
  identical document must not schedule a callback, lay out, or scroll.
- **No new refresh path.** The pin re-applies from the existing update and layout hooks
  only. Do not add a timer, a poller, or a second refresh route
  (`sase/memory/tui_perf.md` rule 5).
- **Pump callbacks stay thin and synchronous.** The re-apply callback reads
  `max_scroll_y`, computes a target, and scrolls. No I/O, no awaits, no unbounded work
  (`tui_perf.md` rule 2). Coalesce with a scheduled flag cleared in the callback, and
  release the flag if scheduling raises.
- **One bottom-target expression.** `action_scroll_to_bottom` and the re-apply must
  compute the target from the same helper so they cannot drift apart.
- **`g` still means "top".** `scroll_to_top` and every other deliberate move keeps
  working unchanged apart from releasing the pin.
- **Other surfaces are untouched.** The Axe pin, the artifacts detail scroll, and the
  Agents file and tools scrolls keep their current behavior exactly.
- **No new keybinding.** `G` keeps its existing `app.scroll_to_bottom` action and config
  key; nothing is added to `src/sase/default_config.yml`.

## Implementation

### 1. Pin state and re-apply on `AgentPromptPanel`

In `src/sase/ace/tui/widgets/prompt_panel/__init__.py`:

- Add class-level state next to the existing section fields (`:38-53`):
  `_pinned_to_bottom: bool = False`, `_bottom_pin_reapply_scheduled: bool = False`, and
  `_bottom_pin_last_y: int = -1`.
- Add `pin_to_bottom()` / `release_bottom_pin()` methods and an `is_pinned_to_bottom`
  property. Gate `pin_to_bottom()` on
  `getattr(self, "id", None) == "agent-prompt-panel"`, matching how
  `enable_section_layout_reserve` (`:148`) already scopes itself to the real metadata
  panel, so a panel instance rendered elsewhere cannot start scrolling a container it
  does not own.
- Add `bottom_scroll_target(scroll) -> int` returning
  `max(0, int(scroll.max_scroll_y) - self.section_layout_reserve)`. This is the one
  expression for "the end of the real document", used by both the pin and
  `action_scroll_to_bottom`.
- Add `_bottom_pin_container()` returning `self.parent` when it is a `ScrollView` (it is
  `#agent-prompt-scroll`, `widgets/agent_detail.py:89`), else `None`.
- Add `_schedule_bottom_pin_reapply()`: no-op unless pinned and not already scheduled;
  otherwise set the flag and `self.call_after_refresh(self._reapply_bottom_pin)`,
  clearing the flag again if the call raises.
- Add `_reapply_bottom_pin()`:
  1. Clear `_bottom_pin_reapply_scheduled` first, so a failure cannot wedge the pin.
  2. Resolve the container; return if there is none or the pin was released meanwhile.
  3. **Release on user divergence.** If
     `int(scroll.scroll_y) != self._bottom_pin_last_y` and
     `int(scroll.max_scroll_y) >= self._bottom_pin_last_y`, someone else moved the
     viewport — a mouse wheel, a scrollbar drag, or a search-overlay restore, none of
     which route through an app action — so call `release_bottom_pin()` and return. The
     `max_scroll_y` half of the condition is what distinguishes a user scroll from
     Textual's own clamp: after a clamp the new maximum is _below_ the offset we last
     wrote, and that case must stay pinned, because it is the exact case this bug is
     about.
  4. Otherwise compute `bottom_scroll_target(scroll)`, call
     `scroll.scroll_to(y=target, animate=False, immediate=True)`, and record
     `_bottom_pin_last_y = target`.
- Call `self._schedule_bottom_pin_reapply()` at the end of `update()` (`:79`), **after**
  `super().update(content, layout=layout)` and after the digest early-return, so an
  unchanged document does no work.
- Re-apply on geometry changes too: handle `events.Resize` on the panel and call
  `_schedule_bottom_pin_reapply()`. A container resize (terminal resize, or a panel-mode
  swap between the `3fr` / `7fr` / `100%` rules in `styles.tcss:3877-3944`) changes
  `max_scroll_y` without any `update()`. Follow the existing MRO-preserving pattern in
  `_agent_display_async.py:166-171` and delegate to `super()`'s handler if one exists.
  This cannot oscillate: scrolling does not change the panel's height, and
  `#agent-prompt-scroll` sets `scrollbar-gutter: stable` (`styles.tcss:3882`), so the
  scrollbar appearing cannot reflow the content width.
- Release the pin in `prepare_section_document()` (`:56`) on the `previous != identity`
  branch, beside the existing `_active_section_identity` /
  `_section_layout_reserve_enabled` resets. Every document path routes through
  `prepare_section_document`, `prepare_section_document_for_agent`, or
  `reset_section_document` — agent selection, attempt pinning
  (`prompt_panel/__init__.py:65`), tribe documents (`_agent_display.py:102`), hint mode
  (`_agent_display_hints.py:83`), workflows (`_workflow_display.py:93`), and the empty
  state (`_agent_display_render.py:601`) — so this one line covers all of them. `G` pins
  the document the user pressed it on, not the panel forever; `j` to a different agent
  starts unpinned, exactly like the section cursor and the layout reserve already do.

### 2. Set and release the pin from the navigation actions

In `src/sase/ace/tui/actions/navigation/_basic.py`:

- `action_scroll_to_bottom` (`:419`), `#agent-prompt-scroll` branch: call
  `panel.pin_to_bottom()` and let the pin's own `call_after_refresh` re-apply perform
  the scroll, replacing the current inline `scroll_to(..., immediate=True)`. This fixes
  defect (3) — the target is now computed after the refresh, from a settled
  `max_scroll_y`, the same way `Widget.scroll_end` does it — and removes the duplicated
  target expression. The non-metadata branches (`#agent-file-scroll`,
  `#agent-tools-scroll`) keep calling `scroll_end` unchanged.
- Release the pin wherever the user deliberately moves the metadata viewport somewhere
  that is not the bottom, so the release is immediate rather than waiting for the next
  repaint to detect divergence:
  - `action_scroll_to_top` (`:400`), agents branch.
  - `action_scroll_detail_down` / `action_scroll_detail_up` (`:239`, `:256`) when the
    resolved `scroll_id` is `#agent-prompt-scroll`.
  - `action_scroll_prompt_down` / `action_scroll_prompt_up` (`:272`, `:287`).
  - `_cycle_agent_metadata_section` (`:310`), which top-aligns a section anchor.

  Add a small `_release_agent_metadata_bottom_pin()` helper on the mixin that resolves
  the panel defensively (the same `try/except` shape the surrounding methods already
  use) and calls `release_bottom_pin()`, so these are one line each.

Note that `ctrl+d` while pinned is already a no-op at the bottom and leaves `scroll_y`
unchanged; releasing there is still correct, because the user asked for a relative move
rather than "stay at the end".

Everything else — mouse wheel, scrollbar drags, and the metadata search overlay's scroll
and restore paths in `actions/agents/_metadata_search.py` — is covered by the divergence
check in step 1 and needs no call-site change. A search restore that lands back on the
pinned offset keeps following; one that lands elsewhere releases.

### 3. Help text

`src/sase/ace/tui/modals/help_modal/agents_bindings.py:73-76` currently labels the `g` /
`G` row "Scroll file panel to top / bottom", which is wrong twice over now: `G` targets
whichever detail surface is active, and on the metadata panel it now follows. Retitle it
to say the bottom scroll pins/follows on the metadata panel, in the same terse register
as the neighboring rows.

## Tests

Reuse the mounted harness that already exists for this surface. Lift the
`_MetadataNavigationApp` class out of
`tests/ace/tui/widgets/test_prompt_panel_section_navigation_actions.py:18-44` into the
shared `tests/ace/tui/widgets/_prompt_panel_section_navigation_helpers.py` (imported by
that file already) and import it from both, rather than duplicating a second harness.
Give it the extra `ctrl+u` / `ctrl+d` / `ctrl+f` / `ctrl+b` bindings the new tests need.

Add `tests/ace/tui/widgets/test_prompt_panel_bottom_pin.py`:

1. **Growth keeps the view at the end.** Press `G`, assert the panel is at
   `bottom_scroll_target`, then `panel.update(...)` a taller document and assert it is
   still at the (new, larger) bottom target — not at the stale offset. Fails on
   `master`.
2. **Shrink then regrow does not strand the viewport.** Press `G`, update with a much
   shorter document (the clamp), then update with the tall one again, and assert the
   panel is back at the bottom target rather than the clamped offset. This is the "jump"
   half of the bug and fails on `master`.
3. **An unchanged document does no work.** Press `G`, then call `panel.update()` with
   the identical renderable and assert the digest early-return still fires — no
   generation bump (`_section_generation` unchanged) and no re-scroll scheduled. Guards
   the idle cost invariant.
4. **`g` releases.** Press `G` then `g`, grow the document, and assert the panel stays
   at the top.
5. **Relative scrolls release.** Parametrize over `ctrl+u`, `ctrl+d`, `ctrl+f`,
   `ctrl+b`: press `G`, press the key, grow the document, assert the offset is whatever
   the relative scroll left behind and not the new bottom.
6. **A direct container scroll releases.** Press `G`, then call
   `scroll.scroll_to(y=0, animate=False, immediate=True)` directly to stand in for a
   mouse wheel, grow the document, and assert the panel stays at the top.
7. **A clamp does not release.** The inverse of (6), asserting the divergence check's
   `max_scroll_y` guard: after a shrink that clamps the offset, the pin is still set.
8. **A new document releases.** Press `G`, call
   `panel.prepare_section_document("other")`, update, and assert the panel is unpinned
   and at the top.
9. **The pin respects the section reserve.** With the reserve enabled by a `ctrl+j`
   first, press `G`, grow the document, and assert the resting offset is
   `max_scroll_y - section_layout_reserve` and strictly less than `max_scroll_y`.

The existing assertions at `test_prompt_panel_section_navigation_actions.py:95-105` —
that `G` lands on `max_scroll_y - section_layout_reserve` — must keep passing unchanged;
they are the regression guard for routing `G` through the new helper.

## Verification

Run `just install` first — this ephemeral workspace needed it during planning and its
`sase_core_rs` build was still in flight — then `just check` inline.

The change is confined to the ACE TUI (one widget module, one navigation mixin, one help
module, plus tests) and adds no import-graph or cross-package edges, so the scoped lane
is the right gate; `just check-full` is not required for this change unless `just check`
reports an unusual selection or escalates, in which case run it only through the
`/sase_monitor` skill with the `TESTING` / `TESTED` status pair, never inline
(`sase/memory/lint_and_test.md`).

`symvision` should stay clean without a pragma: every new method has an in-repo caller —
the navigation mixin for the public ones, `update()` and the resize handler for the
private ones.

Finally, confirm it by hand in a live ACE session, because the failure is a timing
behavior the mounted tests only approximate: open the Agents tab on a RUNNING agent
whose metadata is still enriching, press `G`, and watch for ~30 seconds across at least
one slow-tool tick and one auto-refresh. The last line of the document must stay on
screen the whole time. Then scroll up with the wheel and confirm it stays where you put
it.

## Non-goals

- **Do not pin the file or tools panels.** `G` on `#agent-file-scroll` and
  `#agent-tools-scroll` keeps its current one-shot `scroll_end`. Those documents are
  replaced wholesale rather than progressively enriched, so they do not have this bug,
  and following a diff is a different feature request.
- **Do not add a pinned/following indicator to the panel border subtitle.** It is a
  reasonable idea and the Axe pin does not have one either; adding one means touching
  the subtitle builders on a surface that already recomputes them on every refresh.
  Separate work if it is wanted.
- **Do not use Textual's `Widget.anchor()`.** Explained above: the compositor's anchor
  target includes the `section_layout_reserve` phantom extent, so it would park the view
  in blank space below the document whenever section navigation has been used.
- **Do not add a feature flag.** Per `sase/memory/sase_flags.md`, a flag is for an
  unproven opt-in beta or for keeping a deprecated branch reachable while callers
  migrate. This is a bug fix to a keystroke's behavior with no old branch anyone
  migrates off, and an agent creates a `beta` flag only as epic scaffolding. If the
  follow-behavior should be user-selectable forever, that is a config field, not a flag
  — and the user has not asked for one.
- **No Rust core change.** Per the core-backend-boundary rule, the litmus test is
  whether another frontend would need this to match the TUI. Scroll offsets,
  keybindings, and Textual layout state are presentation-only, so this stays in Python.
- **Do not change `_get_agent_detail_scroll_id`.** Which surface `G` targets when an
  agent has file content is existing, separately-decided behavior
  (`actions/navigation/_basic.py:222`); this plan only changes what happens once the
  metadata panel is the target.

## Done when

- On the Agents tab with the metadata panel active, `G` leaves the last line of the
  document on screen and keeps it there across streamed header lanes, enrichment
  workers, the 5-second slow-tool tick, and auto-refresh.
- The pin releases on `g`, on `ctrl+u`/`ctrl+d`/`ctrl+f`/`ctrl+b`, on `ctrl+j`/`ctrl+k`,
  on a mouse-wheel or scrollbar scroll away from the bottom, and on moving to a
  different agent, attempt, or tribe document.
- A repaint that renders a shorter document no longer strands the viewport high in the
  document once the full document returns.
- `grep -n "section_layout_reserve" src/sase/ace/tui/actions/navigation/_basic.py`
  returns nothing: the bottom-target expression exists once, on the panel.
- An auto-refresh tick that renders an identical document still returns early from
  `AgentPromptPanel.update()` and schedules no scroll.
- `tests/ace/tui/widgets/test_prompt_panel_bottom_pin.py` passes, and its growth and
  shrink-then-regrow cases fail on `master`.
- `just check` is clean.
