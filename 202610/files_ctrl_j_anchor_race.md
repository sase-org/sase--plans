---
tier: tale
title: Fix the Files Ctrl-J page-anchor scroll race
goal:
  Complete sase-1ou.4 with a production scroll-race repair, deterministic regression
  coverage, and comparable isolated and loaded stress evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1ou.4
proposed_with: codex/gpt-6-astra
bead: sase-1ou.4
create_time: 2026-10-27 10:41:11
status: wip
---

- **PARENT:**
  [202610/backlog_epic_2_top_five_task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202610/backlog_epic_2_top_five_task_beads.md)
- **BEAD:** sase-1ou.4

# Fix the Files Ctrl-J page-anchor scroll race

Complete phase `sase-1ou.4` of `sase-1ou` by repairing the production scroll race
tracked by `sase-1a7`. Ctrl-J in a Files spread must put the selected page's titled
separator at the top and keep it there after the queued layout and content work has
settled. Preserve the existing five-second assertion and page/body distinction.

## Scope and prior work

This is one bounded TUI repair, its deterministic regression coverage, and the parent
epic's stress proof. Use one medium coding agent. The assigned phase has no earlier
attempt or remaining-work notes, so there is no unfinished phase implementation to split
into a child epic. Long verification commands use `/sase_monitor` and its continuation
contract.

Read these audited references before implementing:

- `sase bead read sase-1ou.4 -r "Need current phase scope and notes"`.
- `sase bead read sase-1a7 -r "Need current anchor-scroll failure evidence"`.
- `sase artifact read plan:202610/backlog_epic_2_top_five_task_beads.md "Need parent constraints and deck-anchor acceptance"`.
- Through `/sase_memory_read`, read `ace.md`, `tui.md`, `tui_perf.md`,
  `lint_and_test.md`, and `sase_beads.md`.

The parent decision `dormant_flakes = close` is final and concerns the parent lander;
this phase must not repeat that triage. No memory edits are authorized. Do not change
`sase/memory/**`, create beads, close other beads, or manually change phase status. The
user's phase-worker instruction to close only `sase-1ou.4` controls over the parent
design's instruction to close `sase-1a7`; deliver the task's closing evidence to the
parent lander in the phase note. Never close `sase-1ou` or an ancestor plan.

Planning inspected clean base `e608de2db6c2406bf614d53f31c34c774dbd47c0`:

- `7dd39ff95f` already landed the repair described by closed phase `sase-1mk.3`.
  `panel_spread.py` increments `_files_transition_generation`, enables the trailing
  reserve, defers the jump, checks that `max_scroll_y` can accommodate the anchor, and
  confirms the live position. It abandons the request after eight attempts.
- `panel_files.py::_apply_files_transition` already guards its deferred `_restore`
  against that generation. The existing pilot deliberately releases an obsolete
  transition restore after the Ctrl-J request. Preserve that coverage.
- The shared `wait_for_anchor_scroll` already re-reads the live target. Merely changing
  the test to follow a moving anchor is also an earlier, landed repair.
- The task records post-fix Python 3.12 CI recurrences on October 23 and 25, and the
  parent records two more on October 26. Prior green stress runs are insufficient.
- Six fresh focused invocations passed on local Python **3.14.7**: one invocation, then
  a five-invocation loop, all with `-q -p no:randomly`. These are smoke evidence, not a
  before/after failure-rate proof or a Python 3.12 result.
- There is no current entry for this exact node in
  `tests/reproducible_flake_baseline.txt`, and `sase bead epic-symbols sase-1ou.4`
  reported no entries. Recheck both at completion.

## 1. Establish the remaining interleaving

Use this unchanged test node:

```text
tests/ace/tui/widgets/decks/test_deck_spread_pilot.py::test_files_ctrl_j_scrolls_page_anchor_to_top
```

Before changing production behavior, record the actual base SHA, interpreter, dependency
versions, and load conditions. Run at least 50 isolated invocations of the node and five
repetitions of `tests/ace/tui/widgets/decks/` with `-n 6`, with a bounded concurrent CPU
load source for the loaded runs. Count this node separately from other directory
failures. Preserve exact commands and per-attempt results so the same loops can run
after the fix. Use `/sase_monitor` for these long commands; prepare its prompt to
continue diagnosis, not to declare completion on exit.

The primary recurrence is on Python 3.12. Reproduce and validate in that interpreter as
well as the normal checkout environment. If a separate environment is needed, create it
inside the checkout using the repository's supported virtualenv setup and the pinned
core binding; do not globally install SASE or bypass binding validation. Do not
interpret Python 3.14-only passes as disproving the 3.12 reports.

Capture a bounded event trace around a failed jump: requested page, current/queued
generation, callback type and attempt, active mode and subject, anchor row/width and
layout generation, reserve size, `scroll_y`, `scroll_target_y`, and `max_scroll_y`.
Capture which callback last wrote the Files scroll container. Use temporary diagnostics
or failure-only test output, not permanent hot-path logging.

Inspect these concrete interactions:

1. `decks/panel_spread.py::_cycle_files_spread`, `_schedule_files_scroll`,
   `_apply_files_scroll`, and `_confirm_files_scroll`: distinguish an exhausted retry
   sequence, deferred scroll execution, and an overwrite after confirmation.
2. `decks/panel_files.py::_apply_files_transition` and Files probe/resize callbacks:
   determine whether newer work incorrectly invalidates the user's navigation or changes
   its anchor/extent after the landing was accepted.
3. `decks/files_spread.py::_publish_section_layout` and `prompt_panel/_section_view.py`:
   anchors and reserve geometry are published at different points; an after-refresh
   callback is not proof that both are current.
4. `file_panel/_content.py::_schedule_scroll_anchor_restore` and `_apply_scroll_anchor`:
   `_get_scroll_container` returns the parent without a visibility check, while
   `decks/panel.py::_sync_files_views` hides the paged file view inside the same
   container used by the spread. A delayed paged read/restore can therefore be a
   competing scroll writer. This is a code-derived hypothesis, not yet a captured
   spontaneous failure.

Turn the observed ordering into a deterministic test by holding and releasing real
callbacks or worker completion at the relevant boundary. Keep the original pilot
unchanged as an independent end-to-end check. If the natural loop stays green, report
that rate accurately and require an induced, realistic interleaving that fails on
unchanged production code before claiming the race is understood.

## 2. Repair production scroll ownership and settling

Make the smallest fix supported by the trace and failing regression. This work is
Textual presentation state and belongs in this repo. Do not redesign core backend
behavior or introduce a Python fallback for shared Rust logic.

The resulting behavior must satisfy these invariants:

- Only the currently displayed Files mode can move its shared viewport. A stale
  paged-view content/anchor callback cannot reset an active spread's scroll; returning
  to paged mode must still restore its page-local reading offset.
- The newest navigation request owns its target. Older transition, subject, or
  navigation callbacks cannot override it. Subject replacement, mode changes, and widget
  teardown invalidate outstanding work.
- The jump resolves the live titled-separator row after usable anchors and reserve
  extent exist. Applying it and observing it must account for Textual's deferred scroll
  semantics. A later relevant layout publication cannot silently lose an outstanding
  request.
- If retries are the demonstrated failure, coalesce pending navigation and resume it
  from an appropriate existing layout/refresh signal. Do not merely increase the
  eight-attempt cap, spin indefinitely, or introduce a wall-clock delay. Cancel the
  pending request when its owner is superseded or removed.
- Completion does not pin the viewport forever: subsequent ordinary user scrolling
  remains effective. Keep callbacks thin and nonblocking; no I/O or sleeps on the event
  loop and no render-triggered refresh cycle.

Prefer existing generation and layout notification mechanisms. Touch `panel_spread.py`,
`panel_files.py`, `files_spread.py`, or the file-panel anchor controller only as needed
for the demonstrated cause. If the shared section mixin must change, cover the affected
Main/section navigation too.

## 3. Pin the behavior with meaningful regression tests

Add focused tests in the existing deck/file-panel test areas. Use real widgets and
controlled callbacks for ordering-sensitive assertions; do not assert implementation
details such as the exact number of retries.

- The demonstrated late layout or competing restore interleaving fails before the
  production fix and passes after it, with separator at `scroll_y` and body at
  `anchor + 1` after the delayed work is released.
- A stale transition restore remains harmless, preserving the current pilot's case.
- If paged restore ownership changes, cover both a delayed hidden paged callback and
  normal visible paged restoration; switching back must retain page-local offset.
- Exercise a superseding request and subject/mode invalidation for any new pending
  state, then confirm ordinary manual scrolling works after navigation settles.

Keep the five-second wait, assertions, and production policy intact. No skips, xfails,
timeout increases, arbitrary pauses, symvision suppressions, or broadened flake-baseline
entries. Add diagnostic values to failures if useful; do not make the wait helper scroll
or repair the view itself.

## 4. Prove the repair and record the result

Repeat the exact isolated and loaded loops from step 1 on the fixed tree: at least 50
isolated runs and five directory runs at `-n 6` with the same concurrent load source.
Report actual before/after failures over attempts by interpreter and load condition,
including the deterministic regression result. Stop and clean up only the load processes
created for this proof. Retain logs and a concise evidence report with audited artifact
commands; avoid adding a general stress framework.

Run the whole `test_deck_spread_pilot.py` file and affected file-panel/section tests.
Run `sase tool run check`, the governed `just check` recipe. Run `just fmt` or
`just fix` before handing final checks to a monitor. Do not run `just check-full`.

For rendered-output or layout changes, read `tui_screenshot.md` and run the targeted
Files deck PNG snapshots via `just fix-tui-screenshots` selectors, including
`tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views_files.py`. Inspect the
manifest and every affected golden; a partial run is not complete visual evidence.

Recheck the exact node in `tests/reproducible_flake_baseline.txt`. If it remains absent,
add no suppression or unnecessary fixed-at directive. If an entry appeared since
planning, retire it according to the parent's fixed-at contract, with truthful fix
identity/time. Never invent a future commit SHA or landing timestamp; give the parent
lander the finalization-dependent bookkeeping explicitly if needed. Run
`.venv/bin/python tools/selection_health --fail-on-new-flake` and record this node's
status separately from other owners' exceedances.

Investigate new check failures. For a failure reproduced identically on a clean base,
record `PROPOSED FOLLOW-UP:` on `sase-1ou.4`, citing an existing owner when known, and
continue to close this phase when its own work is verified. Do not repair unrelated
epic-owned failures or create task beads.

## Completion

Append evidence to `sase-1ou.4`: the captured cause, affected paths, regression
before/after, stress rates and commands, runtime/load conditions, check results,
artifact references, and any baseline disposition. Give the `sase-1ou` lander enough
evidence to close `sase-1a7`; leave that task and all ancestors for the lander.

Immediately before closing, run `sase bead epic-symbols sase-1ou.4`. Resolve any entries
or re-key legitimate future work to a still-open parent/later phase; never leave a
closed phase's symbol allowances behind. Close **only** `sase-1ou.4` with
`sase bead close sase-1ou.4 --note "<verified behavior, regression/stress results, and check evidence>"`.
Use `/sase_final` as the last action of a normal completion, allowing host finalizers to
create commits. A successful monitor handoff follows its own turn-ending rules.
