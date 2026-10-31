---
tier: tale
title: Fix the stale prompt-bar cursor readout, then close epic sase-1q5
goal:
  The debounced prompt-bar cursor readout always converges to the cursor's real
  position, even after other code writes the subtitle directly. Epic sase-1q5 is then
  closed with its verification note, a clean Symvision whitelist, and its plan file
  marked done.
size: small
proposed_by: bbugyi200.athena.sase-1q5.land
proposed_with: claude/opus
bead: sase-1q5
create_time: 2026-10-31 00:45:09
status: wip
---

- **PARENT:**
  [202610/tui_index_lock_speedup.md](https://github.com/sase-org/sase--plans/blob/main/202610/tui_index_lock_speedup.md)
- **BEAD:**
  [sase-1q5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1q5/README.md)

# Plan: Fix the stale prompt-bar cursor readout, then close epic sase-1q5

## Context

Epic `sase-1q5` ("Restore TUI speed by ending artifact-index lock starvation") has all 8
phases closed. Its land agent verified every phase against the epic plan and the
DECISIONS. It confirmed the sase-core pushdown commit `2775bc00` is inside the
`sase-core-revision.txt` pin and found no conflicts with commits that landed after the
epic started. It triaged all 12 `PROPOSED FOLLOW-UP` notes and recorded the outcomes on
the epic bead. `sase bead epic-symbols sase-1q5` reports no entries.

One defect caused by the epic remains. Phase `sase-1q5.6` (commit `001351f5b9`,
"perf(tui): move small synchronous work off pump") debounced the prompt bar's cursor
readout. It also added a dedupe in
`src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py`:

```python
def _refresh_cursor_subtitle(self) -> None:
    """Rebuild the border subtitle only when its rendered value changes."""
    rendered = self._render_subtitle(self._subtitle_base)
    if rendered == getattr(self, "_cursor_subtitle_rendered", None):
        return
    self._cursor_subtitle_rendered = rendered
    self.border_subtitle = rendered
```

The dedupe compares against a private cache of the last value _this method_ wrote. About
ten other code paths assign `self.border_subtitle` directly:

- soft completion, next-word hints, and the mode subtitle in
  `_prompt_input_bar_completion_panel.py`;
- `refresh_search_readout` in `_prompt_input_bar_search.py`.

Those writes render the cursor position at the moment they run. Suppose such a write
lands mid-burst at an intermediate cursor position, and the cursor then returns to the
position cached at the last debounced write. The debounced callback renders a value
equal to the stale cache and skips the write. The displayed readout keeps showing the
intermediate column until the next cursor move. Typing two characters and backspacing
twice within the 150 ms debounce window, with a soft completion appearing in between, is
enough.

Reproduced on master in a pilot test:

1. Seed the bar with `hello`.
2. Set the cursor to `(0, 1)` and call `bar.refresh_search_readout()`.
3. Set the cursor back to `(0, 5)` and call `bar.refresh_cursor_readouts()`.
4. Wait 0.3 s.

The displayed subtitle (`bar._border_subtitle.plain`) still ends with `Ln 1, Col 2`, but
`bar._render_subtitle(bar._subtitle_base).plain` ends with `Ln 1, Col 6`.

The fix below was trialed by the land agent: the readout converged to `Col 6` and all 14
existing tests in `tests/ace/tui/widgets/test_prompt_input_bar_cursor_readout.py`
passed.

## Step 1: Make the dedupe check what is actually displayed

In `_refresh_cursor_subtitle`
(`src/sase/ace/tui/widgets/_prompt_input_bar_stack_rendering.py`), skip only when the
render is unchanged _and_ the widget still shows the value this method last wrote.
Textual's `border_subtitle` getter returns the stored title's console markup, so record
that markup after each write and compare against it:

```python
def _refresh_cursor_subtitle(self) -> None:
    """Rebuild the border subtitle only when the displayed value would change.

    Other subtitle writers (soft completions, next-word hints, the mode
    subtitle, the search readout) assign ``border_subtitle`` directly, so the
    skip also requires the widget to still show the markup written here last.
    """
    rendered = self._render_subtitle(self._subtitle_base)
    if rendered == getattr(
        self, "_cursor_subtitle_rendered", None
    ) and self.border_subtitle == getattr(self, "_cursor_subtitle_markup", None):
        return
    self.border_subtitle = rendered
    self._cursor_subtitle_rendered = rendered
    self._cursor_subtitle_markup = self.border_subtitle
```

- Keep the debounce in `refresh_cursor_readouts` and the `on_unmount` cancel in
  `_prompt_input_bar_lifecycle.py` unchanged.
- Do not add the cache reset to the other writers. The displayed-markup check covers
  every current and future direct writer.
- The callback runs at most once per 150 ms debounce burst, so reading the markup costs
  nothing that matters. Keep the method's docstring accurate.

## Step 2: Regression test

In `tests/ace/tui/widgets/test_prompt_input_bar_cursor_readout.py`, next to
`test_border_subtitle_debounces_and_skips_unchanged_render`, add
`test_debounced_readout_repaints_after_a_direct_subtitle_write`:

- Use `_PromptBarApp("hello")` and `app.run_test(size=(80, 24))`, then
  `await pilot.pause(0.2)`.
- Set the cursor to `(0, 1)` and call `bar.refresh_search_readout()`, a direct write
  that renders `Col 2`.
- Set the cursor back to `(0, 5)`, call `bar.refresh_cursor_readouts()`, then
  `await pilot.pause(0.3)`.
- Assert the _displayed_ subtitle matches the current render:
  `bar._border_subtitle.plain == _subtitle_text(bar).plain`, and it ends with
  `"Ln 1, Col 6"`.
- Read `_border_subtitle` (Textual's stored `Content`), not `_subtitle_text`. The
  existing helper recomputes the render and cannot see a stale display.

Confirm the new test fails on the unfixed code (stash the Step 1 change and run it
once), then passes with the fix. The existing
`test_border_subtitle_debounces_and_skips_unchanged_render` must still pass. With no
external writes, the second refresh still skips, so `_cursor_subtitle_rendered` keeps
its identity.

## Step 3: Verify

- Read `lint_and_test.md` with `/sase_memory_read` before verifying. Then run the
  focused tests:
  `.venv/bin/python -m pytest -q tests/ace/tui/widgets/test_prompt_input_bar_cursor_readout.py tests/ace/tui/widgets/test_prompt_input_bar_stack*.py`.
- Run `just check` through `sase tool run`, the only way an agent may run that guarded
  recipe; hand it to `/sase_monitor` if it will outlast your turn's foreground limit.
- Do not run `just check-full` or any global install.
- Known unrelated failures you may see are already owned by beads:
  - `tests/agents_sync/test_cli_names.py::test_repair_states_dispatches_and_reports_json`
    (`sase-1q7`);
  - the full-lane flakes in `sase-1q8`, `sase-z1`, and `sase-13a`;
  - `test_start_monitor_serializes_concurrent_starts_in_one_lane` (epic `sase-j7`).

  Do not fix them here.

## Step 4: Close out epic sase-1q5 (final step, same turn as the fix)

Do this in the same turn as Steps 1–3. Do not wait for, or order it after, this work's
own commit, push, or CI.

1. Run `sase bead epic-symbols sase-1q5`.
   - The land agent saw no entries. If any appear, resolve each per the Symvision
     epic-whitelist policy (read `symvision.md` with `/sase_memory_read`): wire it up,
     privatize it, add a non-test pragma, or delete it.
   - Re-key a Justfile `--epic-symbol` line to another bead only if a still-open later
     bead needs it.
2. Close the epic with a note that summarizes the verification:

   ```bash
   sase bead close sase-1q5 --note "Verified all 8 phases against the plan and DECISIONS (pushdown, no rollback flag, tui_perf.md untouched): stranded batch 238->3 queries; 60 s TTL, 30 s forced floor, memoized verdicts/counts; lock-busy serves same-key cache or keeps roster, first paint keeps bounded scan; sase-core 2775bc00 SQL session/name pushdown + NOCASE indexes + scoped reconcile, inside the sase-core-revision.txt pin; capacity/title and runtime-aggregation trims; persistent trim worker, off-pump ToolRun and capacity probes, debounced prompt subtitle; capacity rescan 120 s + parsed-cache reuse; freeze-report slow-load section and perf-runbook remeasure recipe. Integration: no post-epic commit touches the epic's surfaces; no TUI-process lock-busy source scan remains. Fixed the debounced subtitle dedupe staleness from sase-1q5.6 (displayed-markup check plus regression test). Follow-ups triaged in epic note (new tasks sase-1q7, sase-1q8, sase-1q9, sase-1qa; +1s on sase-1ge, sase-1gf, sase-lx, sase-199, sase-z1, sase-13a; sequencing note on sase-1nc; macro-schema proposal declined as already fixed)."
   ```

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, clean them up and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/tui_index_lock_speedup.md`. Get its absolute path from the `PLAN` line
   of `sase bead read sase-1q5 -r "Need the epic plan path for closeout"`. It currently
   reads `status: wip`; change only that field.
5. Epic `sase-1q5` has no `parent_bead`, so nothing further cascades.
