---
tier: tale
title: Reveal a hidden prompt-bar spare on space
goal:
  Plain home space reveals one pre-mounted inert prompt bar, and every other prompt mode
  still fresh-mounts a new instance per session.
size: medium
proposed_by: bbugyi200.athena.sase-1ex.11
bead: sase-1ex.11
create_time: 2026-10-03 08:59:53
status: wip
---

- **PARENT:**
  [202610/prompt_space_and_project_cycle_latency.md](https://github.com/sase-org/sase--plans/blob/main/202610/prompt_space_and_project_cycle_latency.md)
- **BEAD:**
  [sase-1ex.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ex/sase-1ex.11.md)

# Plan: Reveal a hidden prompt-bar spare on space

Implement epic phase `space-hot-spare` (bead `sase-1ex.11`) from
`plan:202610/prompt_space_and_project_cycle_latency.md`. `<space>` on the plain home
prompt becomes a reveal of one idle-mounted bar. Feedback, approve, relaunch, markdown,
frontmatter, and read-only prompts keep today's fresh mount. Do not close the parent
epic `sase-1ex` or any ancestor. Do not create beads. Record extra work as
`PROPOSED FOLLOW-UP:` notes on `sase-1ex.11`.

Before editing, read `tui.md` and `tui_perf.md` with `sase memory read`. Before
finishing, read `lint_and_test.md` and `symvision.md` the same way. Use the names master
already has (`vcs_macro_mru` versus `vcs_xprompt_mru`). Do not edit
`src/sase/default_config.yml`. No feature flag.

## Gate

Run the slow bench first, with every earlier phase already landed:

```bash
pytest -s -m slow tests/ace/tui/bench_prompt_bar_keys.py
```

If both the first `<space>` and the steady `<space>` printed p95 are at most 60 ms, skip
the spare. Note the numbers on `sase-1ex.11` and close the bead. The research expects
this gate to fail, because compose, CSS, and reflow are about 90 ms. A missed target
later is a recorded reason plus a follow-up, never a guardrail trade.

## Spare lifecycle

Add `app._prompt_bar_spare: PromptInputBar | None`, initialized next to
`_active_prompt_bar` in `src/sase/ace/tui/actions/_state_init_runtime.py`. Enable
scheduling only on `AceApp` (`_prompt_bar_spare_enabled = True`). Harnesses that compose
`PromptBarMountMixin` without that flag (`PromptLifecycleApp`, `RealBarLaunchApp`) stay
spare-free, so today's "any `PromptInputBar` in the DOM means the prompt is active"
tests keep their meaning.

After `_mount_state_loads_done` becomes true, and again from `_detach_prompt_bar` once
the outgoing bar is removed, schedule one attempt. Re-arm a short timer until all of
these hold: startup loads are done, the nav gate is idle, `_prompt_input_active()` is
false, no modal owns input, and no spare is already mounted. Then mount at most one
fresh `PromptInputBar`.

Construct that bar with no `id`, `display=False`, and `can_focus_children=False`. Mount
it with `self.mount` on the app, the same parent today's bar uses. `styles.tcss` already
styles `PromptInputBar` by type and has no `#prompt-input-bar` rule, so no TCSS
conversion is required. Verified against Textual's `DOMNode.id` setter: an unset id may
be assigned once, even after mount, but the assignment does not register the id in the
parent `NodeList._nodes_by_id`, so `DuplicateIds` would miss it. Do not assign an id
after mount. The spare stays id-less for its whole life, including after reveal. Lookups
keep using `mounted_prompt_bar` / `_active_prompt_bar`.

Mark the instance with an explicit `_is_prompt_spare` flag set at construction.
`display=False` alone is not the marker. Clear the flag only when this instance is
revealed.

Never recycle a used bar. Revealing clears `_prompt_bar_spare`. Dismissal removes that
instance through today's synchronous detach and cancelled-history save. The next idle
moment mounts a different instance.

A non-spare mount must drop any leftover spare first, synchronously, before the new bar
is inserted. Cover `_show_prompt_input_bar_for_home` and the other fresh-mount sites.
Also discard from a non-spare `on_mount`, so a direct
`app.mount(PromptInputBar(id="prompt-input-bar"))` (visual PNG helpers) does not leave
two bars. `query_one("#prompt-completion")` and `query_one("#frontmatter-panel")` walk
depth-first and would otherwise hit the spare's children.

## Inert while hidden

`PromptInputBarLifecycleMixin.on_mount` today publishes `app._active_prompt_bar` and
then focuses, places the cursor, marks `prompt_space` paint, sets title and subtitle,
adds mode classes, and runs warm-ups. Move that body into a new `activate()` method.

- A spare's `on_mount` does not publish `_active_prompt_bar` and does not call
  `activate()`. It does not focus, watch the theme, schedule the xprompt stale-check
  worker, or run `_schedule_deferred_mount_warmups`.
- A fresh, non-spare bar calls `activate()` from `on_mount`, so feedback, approve,
  relaunch, markdown, frontmatter, and read-only mounts are unchanged.
- `activate()` is idempotent. A second call does not double-watch the theme or
  double-schedule warm-ups.

`_prompt_input_active()` stays false while only the spare exists, so the countdown tick,
auto-refresh, watcher deferral, and `start_agent_from_patch` availability stay as they
are today. `check_app_action` already disables that action only when
`_prompt_input_active()` is true.

The catalog refreshers in `_startup_prompt_catalog.py`
(`_refresh_visible_prompt_semantic_surfaces` and
`_refresh_visible_prompt_catalog_surfaces`) skip a text area whose own `display` is
false. A child of a `display: none` parent still reports `display=True`. Skip text areas
that sit inside an inert spare, in those two loops and in any other
`query(PromptTextArea)` loop that would mutate a merely mounted area. Loops that already
no-op unless a completion menu is open can stay as they are.

## Plain `<space>` reveal

Only the plain home path in `action_start_agent_from_patch`
(`src/sase/ace/tui/actions/agent_workflow/_entry_custom.py`) may reveal the spare. Keep
the existing prefill branches: warm snapshot, empty MRU, cold snapshot plus late
prefill, and the no-snapshot loader fallback.

When a ready inert spare is mounted and this call is the plain home prompt (no
`as_xprompt_markdown`, frontmatter inputs, binding, read-only target, selected pane, or
cursor):

1. Begin the session with `_setup_home_prompt_context`, which mints a new `session_id`
   via `begin_prompt_session`.
2. Seed the spare from the same prefill the fresh path would have passed as
   `initial_value`. MRU head text is a single pane (`+project ` or a blank string).
   Write it with the active text area's `load_text` and `_sync_state_from_widgets`, the
   same loader `try_apply_pending_space_prefill` already uses. Do not recompose the
   spare.
3. Reveal: `display=True`, `can_focus_children=True`, clear `_is_prompt_spare`, clear
   `_prompt_bar_spare`.
4. Set `_active_prompt_bar`.
5. Call `activate()`. That focuses, moves the cursor to the end, refreshes title,
   subtitle, and classes, marks the in-flight `prompt_space` sample, and runs the
   warm-ups.

Do this synchronously inside the key handler so `record_pending_space_prefill` still
snapshots the post-activate cursor. The cold branch still opens a blank bar and records
the pending prefill. Late apply already goes through `mounted_prompt_bar`, which prefers
`_active_prompt_bar`.

If no spare is ready, call today's `_show_prompt_input_bar_for_home` unchanged. That
includes a second `<space>` that arrives before the replacement spare has mounted.

Any `_show_prompt_input_bar_for_home` call that passes markdown, frontmatter, a binding,
a read-only target, a pane, or a cursor discards the spare and fresh-mounts with
`id="prompt-input-bar"`.

## Dismissal

Keep `_detach_prompt_bar`: cancelled-history save for the outgoing session, synchronous
node removal, focus transfer, and clearing `_active_prompt_bar`. The revealed bar is
removed and never returned to the spare slot. Scheduling the next spare happens only
after that removal, and only when the enable flag is set.

## Docs

Update the one `prompt_space` sentence in `docs/perf_runbook.md` so the model timestamp
is described as landing in `activate()` (reveal or fresh mount), not only on first
mount. Leave the final before/after table to phase `acceptance`.

## Tests

Add `tests/ace/tui/test_prompt_bar_hot_spare.py` on `AceApp` (the enable flag is on).
Cover:

- The spare mounts once after startup idle, stays `display=False`, has no id, and is not
  `_active_prompt_bar`. `_prompt_input_active()` is false. `check_app_action` still
  allows `start_agent_from_patch`. No warm-up worker or timer is running for the spare.
- Revealing matches a fresh `_show_prompt_input_bar_for_home` for text, prompt context
  (`display_name`, `history_sort_key`), title, subtitle, cursor, and classes.
  Parametrize a warm prefill, an empty MRU, and a cold snapshot whose late prefill lands
  on the untouched revealed bar.
- Each activation gets a new `_prompt_session.session_id`.
- Cancel saves history once, for that session. Submit still launches.
- Rapid `<space>` / `escape` does not collide ids, does not reuse the dismissed widget,
  and falls back to a fresh mount when the replacement spare is not ready yet.
- After dismissal and the idle wait, a different spare is mounted.
- Quitting with a spare mounted leaves `app.workers` finished and no pump-free tasks, in
  the spirit of `test_ace_page_fast_startup_is_structurally_quiet`.

Update an existing assertion only when it treated "any `PromptInputBar`" as "prompt
session open" on a real `AceApp` / `AcePage` that now holds a hidden spare. The
replacement assertion is: no displayed, non-spare bar, and `_prompt_input_active()` is
false. Do not change `PromptLifecycleApp` parity tests unless they start seeing a spare.

## Verification

- `sase tool run check`
- Prompt-bar and Agents-layout ACE PNG suites in check mode, no golden updates. A
  changed golden is a bug. Run them through `sase monitor` if the run would outlast the
  turn:

```bash
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents.py tests/ace/tui/visual/test_ace_png_snapshots_prompt_word_completion.py tests/ace/tui/visual/test_ace_png_snapshots_frontmatter_panel.py tests/ace/tui/visual/test_ace_png_snapshots_vcs_project_completion.py
```

Include the other `tests/ace/tui/visual/test_ace_png_snapshots_prompt_*.py` and
`test_ace_png_snapshots_agents*.py` modules that mount a prompt bar.

- Rerun `pytest -s -m slow tests/ace/tui/bench_prompt_bar_keys.py` and note first and
  steady `<space>` p50/p95/max on the bead. The expected band is about 50–60 ms. If p95
  stays above 60 ms, record the numbers and add
  `PROPOSED FOLLOW-UP: overlay-dock the prompt bar — reveal still misses 60 ms after the hidden spare`.
  Do not implement overlay docking.
- Run `sase bead epic-symbols sase-1ex.11`. Re-key any `--epic-symbol` line that names
  this phase to a still-open bead (the parent epic or a later phase). `sase bead close`
  refuses while leftovers remain.
- Close only `sase-1ex.11` with
  `sase bead close sase-1ex.11 --note "<what you verified>"`.
