---
tier: tale
title: Bring P view transitions within the D10 budgets
goal: "Each P transition on the 5,000-line and 14,000-line Reply fixtures meets the D10
  key-to-paint budgets, or a measured guard the UI explains is in place, and the numbers
  are recorded on bead sase-1b1.8.2.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1b1.8.2
bead: sase-1b1.8.2
create_time: 2026-09-27 15:13:57
status: wip
---

- **PARENT:**
  [202609/deck_views_landing_remainder.md](https://github.com/sase-org/sase--plans/blob/main/202609/deck_views_landing_remainder.md)
- **BEAD:**
  [sase-1b1.8.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1b1/sase-1b1.8.2.md)

# Plan: Bring P view transitions within the D10 budgets

Implement phase bead `sase-1b1.8.2` only. Close that bead after the verification below.
Do not close epic `sase-1b1.8` or plan bead `sase-1b1`. Do not create beads. Record
out-of-scope work as `PROPOSED FOLLOW-UP:` notes on `sase-1b1.8.2`.

This is presentation-only Textual work. Nothing belongs in `sase_core`. Scope stays Main
and Files. Tools and FINAL stay automatic: no badge, and `P` stays unavailable. Keep
`tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py::test_final_panel_shows_no_badge_and_no_cycle`
green. Read `tui.md` and `tui_perf.md` with `/sase_memory_read` before editing TUI code,
and `lint_and_test.md` before finishing.

## What a loaded-host profile already showed

Do not treat the wall-clock numbers below as the acceptance result. They were one sample
each at loadavg about 13.7. Re-measure on a quieter host with the bench. The shape of
the cost is what to trust:

`DeckPanelViewMixin.set_view_policy` returns in about 2–7 ms. `measure_card_rows` does
not run for a fixed view (that skip already exists in `decide_document_block_mode` /
`forced_deck_mode`). `show_document` and `_apply_section_content` run once per press and
return in under 2 ms. There is no second compose to delete.

The time is the refresh after the press, inside `SectionTrackingVisual`
(`src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py`):

- Cold page-cards on the 5,000-line Reply: `get_height` consumed a full Rich render
  (about 930 ms) and `render_strips` built a `Strip` per line (about 410 ms). Paint was
  about 1.7 s, and the 1 s stall watchdog fired.
- Cold page-cards on the 14,000-line Reply: `render_strips` alone was about 1.7 s and
  paint was about 2.7 s. The watchdog fired again.
- Warm repeat of the same view hits `_section_strip_cache` / `_section_height_cache`.
  `render_strips` then costs under 1 ms. Textual still walks every cached line
  (`Strip._apply_link_style`: about 15k calls at 5,000 lines, about 42k at 14,000). That
  walk is inside Textual. Do not fork Textual.
- Page blocks stays around 50 ms. Spread to page blocks is already inside the budget.
  The misses are transitions whose body is the whole Reply (page blocks to page cards,
  page blocks to spread, spread to page cards).

`RichVisual.get_height` and `render_strips` both consume `Console.render` for the entire
reply. `SectionTrackingVisual.get_height` then calls `_anchors_for_rich_visual`, which
consumes `Console.render` again, and `render_strips` does the same on a miss. The bench
warms every transition before sampling, so its samples are the warm cache-hit path. The
pathological test still counts stall-watchdog rows during that warmup, so a cold
14,000-line strip build on the event loop fails the test even when the warm samples are
fast.

One profile window also spent about 500 ms in `probe_version` → `subprocess.run`,
reached from `live_session_ids` on a Textual timer. That is not the steady per-press
cost (page blocks stayed near 50 ms on the same run). If a quiet-host deck-view profile
still shows it inside the paint window, record a `PROPOSED FOLLOW-UP:` note. Do not fix
the proc observer in this phase.

## Work, in order

Stop at the first step whose quiet-host remeasure meets D10. D10, as asserted by
`tests/ace/tui/bench_tui_deck_view.py`:

- Standard Reply (10 turns × 500 lines): p50 at most 150 ms and p95 at most 300 ms
  key-to-paint for every ordered transition.
- Pathological Reply (10 turns × 1,400 lines): max under 1,000 ms and no stall-watchdog
  row, including warmup.
- Forced Files spread (20 files × 2,000 lines) already passes on the landing host.
  Re-record it. Do not redesign the Files probe.

The bench is `slow`. Run it by path, at least twice, under `/sase_monitor`. Note the
host load next to the numbers.

### 1. Collapse the cold double walk

In `SectionTrackingVisual`, one cold consumption of the reply must produce the height,
the strips, and the anchors. Cache them under the existing digest / width / height /
style key. `get_height` and `render_strips` then hit that cache. Collect anchors with
`_anchors_for_strips` from the strips just built, instead of a second
`_anchors_for_rich_visual` console render.

This visual is also the prompt-panel section tracker. Pixels, anchor rows, and scroll
positions stay the same. Do not change the cache key so that two digests alias. The
caches are process-global and capped at 8. Do not virtualize the scroller and do not
patch Textual's `_apply_link_style`.

`panel_view.py` is 827 lines. The hard limit is 1,000. Put any new transition helper in
a new module under `src/sase/ace/tui/widgets/decks/`. Leave `measure_card_rows` alone.

Re-measure. A single 14,000-line strip build is still about 1.7 s, so expect the
pathological watchdog to keep failing if that build stays on the event loop. If both
fixtures already meet D10, skip steps 2 and 3.

### 2. Badge-first body, off the pump

Use this only when step 1 leaves a budget missed. It is the D10 "badge paints first"
mitigation, applied to Main view changes only.

Keep the synchronous prefix of `_apply_main_view_change`: reuse the pending reading
anchor, bump `_view_generation`, store the pending anchor, decide the mode, store
`_render_mode`, and `refresh_chrome`. The badge for the destination view must be what
the next frame paints. `effective_layout` may flip in this prefix, as it does today.

Do not call `show_document` / `Static.update` in that prefix. Build the reply strips off
the UI thread with `spawn_pump_free_task` (`src/sase/ace/tui/util/pump_tasks.py`) and
`asyncio.to_thread`. Use a private `rich.console.Console`. Do not touch Textual widgets,
the app console, or any widget state from that thread.

Apply the prebuilt strips on the UI thread only when `_view_generation` still matches. A
newer press makes the stale apply a no-op. Then run today's
`_restore_spread_view_target` / `_restore_paged_view_target` for that same generation.
Cancel the in-flight task at teardown with `cancel_pump_free_tasks`.

The bench registers `call_after_refresh(mark_painted)` after `set_view_policy` returns.
Do not register a `call_after_refresh` that builds or applies the body, or that callback
runs before `mark_painted` and the budget still includes it. The first frame is the
badge. The Rich walk must not run on the event loop, so the stall watchdog does not
fire. Applying prebuilt strips on the UI thread has to stay under the 1 s stall
threshold. If a quiet-host profile shows the apply itself stalling, chunk that apply. Do
not move the Rich walk back onto the loop.

`tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py` calls `set_view_policy` three
times with no await between them and requires the second and third calls to reuse the
pending anchor. Keep anchor capture and the generation bump synchronous so that test
still holds. `_settle` in that file only waits for `effective_layout` plus two pauses.
Two pauses are not a body-completion barrier. Extend the wait so assertions run after
that generation's body has been applied or dropped. Keep the block, offset, pin, and
rapid-press assertions.

Leave automatic subject changes on `show_main_document` synchronous. The deferral is the
user view-change path (`set_view_policy` → `_apply_main_view_change`), which is what the
bench calls. Files forced spread stays on its off-thread probe.

The bench's between-sample `_settle` must wait until that generation's deferred body has
been applied or dropped, so one sample's UI-thread apply does not fall inside the next
sample's paint window. Leave `mark_painted` on the first refresh after the key. That
refresh is the badge frame. Do not move the paint mark to body completion.

Re-measure. If D10 holds, skip step 3.

### 3. Measured guard, last

Use this only when a budget is still missed after steps 1 and 2. Add an explicit size
guard the UI explains. A fixed inline layout above a line threshold measured from the
quiet-host numbers may show a short badge status or one toast. The threshold comes from
those numbers, not a guess. Never silently ignore the press, and never show a layout
other than the one the badge names (D5). Document the guard in the "Deck Views" section
of `docs/ace.md`. Re-measure and record the numbers.

## Verification

Record p50/p95/max for every standard and pathological transition, plus the Files
forced-spread keypress and after-probe times, in a `sase bead note` on `sase-1b1.8.2`.
Include the host load. If the host stays too busy for a stable sample, say so and keep
the best quiet run.

These stay green:

- `tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py` (all six ordered
  transitions, the bottom pin, rapid P-P-P, partial documents).
- `tests/ace/tui/widgets/decks/test_deck_view_files_pilot.py`, `test_deck_view_keys.py`,
  and the deck chrome and title suites.
- The six `agents_deck_view_*` goldens:
  `just fix-tui-screenshots --check -- tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`.
  If a mitigation changes pixels on purpose, regenerate with a targeted
  `just fix-tui-screenshots` and inspect every golden.

Then `sase tool run check`. Known master-red failures called out on the epic plan (FINAL
deck tests, import budget, finalizer symvision, memory README drift,
`test_agent_completion`, the header-panel scroll test, the Files Ctrl+J flake) are not
this phase. If `just check` fails the same way on the clean base tree, note it as a
`PROPOSED FOLLOW-UP:` and close anyway.

Before closing, run `sase bead epic-symbols sase-1b1.8.2`. Re-key any `--epic-symbol`
Justfile line that this phase owns onto a still-open bead (the parent epic or a later
phase). Close only `sase-1b1.8.2`.
