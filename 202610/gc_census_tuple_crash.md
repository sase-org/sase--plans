---
tier: tale
title: Stop the GC census from crashing TUI workers with tuple SystemError
goal:
  The opt-in GC census no longer crashes TUI worker threads with tupleobject.c
  SystemError on Python 3.12/3.13, and a failed LLM-calls panel fetch no longer exits
  the TUI.
size: small
proposed_by: bbugyi200.athena.15y
proposed_with: claude/opus
create_time: 2026-10-29 15:58:53
status: wip
---

# Plan: Fix the TUI `SystemError: tupleobject.c ... bad argument to internal function` crash

## Problem

On apollo, the TUI keeps failing with
`SystemError: ../Objects/tupleobject.c:909: bad argument to internal function`. It has
exited the TUI twice (2026-10-28 20:25 and 2026-10-29 19:31; both logged as
`textual.worker.WorkerFailed`). Three other runs hit the same error in background tasks
that log failures and keep going, so the TUI stayed up:

- `sase-agents-capacity-refresh` (twice)
- the agents artifact delta refresh

Each of those three tracebacks ends at `tuple(<generator>)` in
`refresh_runner_slot_context` (`src/sase/ace/tui/models/agent_runner_slots.py`). The
fatal 19:31 crash came from the LLM-calls panel's background fetch. Its Rich traceback
shows a frame whose local is a 40-item tuple of `ToolCallEntry` objects. That matches
the `tuple(entry for entry in entries if ...)` helpers in
`src/sase/ace/tui/llm_calls/sources.py` (`_entries_for_dir` /
`_entries_for_cached_dir`).

## Root cause (verified)

The fault is in the opt-in GC census (`SASE_TUI_GC_CENSUS=1`), not in the call sites
that raise:

1. On CPython 3.11 through 3.13, `tuple(<iterator without a length hint>)` goes through
   `PySequence_Tuple`. That function allocates a 10-slot tuple that the GC tracks, fills
   it while running the generator's Python code, and grows it with `_PyTuple_Resize`.
   `_PyTuple_Resize` raises `PyErr_BadInternalCall()` (the `tupleobject.c:909` message
   on Ubuntu's 3.12.3) whenever the tuple's refcount is not exactly 1.
2. `GCPolicy._run_idle_census` in `src/sase/ace/tui/util/gc_policy.py` runs on the UI
   thread. Its last step is `_top_types(get_objects())`. `gc.get_objects()` returns a
   list that holds a strong reference to every tracked object, including tuples that
   worker threads are still building. `_top_types` then walks about 360k objects in
   Python for about 1 s, and the GIL switches to worker threads every 5 ms during that
   walk.
3. Any worker thread that reaches item 11 of a `tuple(genexpr)` during that window finds
   refcount 2 and gets the `SystemError`. The TUI runs many such workers
   (`asyncio.to_thread` agent loads and capacity refreshes, plus the Textual
   `thread=True` workers).

Evidence collected on apollo:

- `~/.sase/logs/tui.log` has five `tupleobject.c:909` failures. Each is within 0.2–0.9 s
  of a `tui_gc_census` row in `~/.sase/logs/tui_stalls.jsonl` (census duration 1.0–2.4
  s). The census rows begin 2026-10-28 20:15, when the TUI was first launched as
  `SASE_TUI_PERF=1 SASE_TUI_GC_CENSUS=1 sase tui` (per zsh history), and the crashes
  begin after that.
- A standalone repro reproduces it exactly: one thread loops `gc.get_objects()` +
  iterate while the main thread loops `tuple(gen_of_40_dicts)`. It raises
  `SystemError: ../Objects/tupleobject.c:909` on apollo's 3.12.3 and the equivalent
  error on 3.11.13 and 3.13.13. It never fails on 3.14.7.
- A single-thread probe confirms why: while a generator feeds `tuple()`,
  `gc.get_objects()` returns the half-built tuple on 3.11, 3.12, and 3.13, and returns
  nothing on 3.14. On 3.14 the half-built tuple is no longer visible to the GC.
- athena's TUI runs on Python 3.14.7 and has zero occurrences. apollo's uv tool install
  uses the system Python 3.12.3. `pyproject.toml` declares `requires-python = ">=3.12"`,
  so 3.12 and 3.13 are both supported and both affected.

The `gc.DEBUG_SAVEALL` garbage half of the census is safe: `gc.garbage` holds only
unreachable objects. A tuple that another thread is still building is a GC root, because
its single reference lives on that thread's C stack.

## Changes

Before editing, read the `tui` and `docs_writing` reference memories (via
`/sase_memory_read`), and read `lint_and_test` before finishing.

### 1. Gate the survivor census on a safe interpreter

In `src/sase/ace/tui/util/gc_policy.py`:

- Add a small module-level predicate, `_survivor_census_safe() -> bool`, that returns
  `sys.version_info >= (3, 14)`. Tests must be able to monkeypatch it. Give it a short
  comment explaining that on CPython < 3.14, `gc.get_objects()` exposes tuples that
  other threads are still building, and holding them makes those threads'
  `_PyTuple_Resize` fail with `SystemError`.
- In `GCPolicy._run_idle_census`, call `get_objects()` only when
  `_survivor_census_safe()` is true. When it is false:
  - skip `get_objects()` entirely;
  - emit `survivor_top: []` and `survivor_count: None`;
  - add `survivor_census: "skipped_unsafe_python"` to the payload (use
    `survivor_census: "ok"` when the survivor census ran).

  Keep the garbage half, the rate limiting, the `gc_trigger("idle")` tagging, and the
  row shape otherwise unchanged. Nothing in `src/` or `tools/` reads `tui_gc_census`
  rows, so the added key is safe.

- Log one warning per `GCPolicy` instance (track it with a private instance flag) the
  first time a census skips the survivor half. The warning should say that survivor type
  counts need Python 3.14+ and that only collected-garbage types are recorded. Never
  raise from this path.
- Update the module docstring and the `_run_idle_census` docstring to say that the
  survivor census needs Python 3.14+.

### 2. Stop LLM-calls panel fetch failures from exiting the TUI

In `src/sase/ace/tui/widgets/llm_calls_panel.py`, both
`self.run_worker(fetch_task, thread=True)` calls (`_start_background_fetch` and
`refresh_llm_calls`) use Textual's default `exit_on_error=True`. Any exception in the
fetch therefore exits the whole app, and the panel's existing `WorkerState.ERROR` branch
("Error fetching tool calls") never runs in production.

- Pass `exit_on_error=False` to both calls. Many other TUI workers already do this, for
  example `src/sase/ace/tui/modals/machines_pane.py`.
- In the `WorkerState.ERROR` branch of `on_worker_state_changed`, log the failure with a
  module logger (`logging.getLogger(__name__)`) before rendering the existing error
  text. For example, call `log.warning("LLM calls fetch failed", exc_info=error)` with
  `event.worker.error`. Do this for both the stale-refetch and current-agent paths.
  Leave the rendered text and the stale/refetch behavior unchanged.

### 3. Docs

In `docs/contribute/perf-runbook.md`, update both `SASE_TUI_GC_CENSUS` descriptions: the
"Heap sampler" paragraph near line 745, and the GC policy paragraph near line 1017 that
starts "Set `SASE_TUI_GC_CENSUS=1` ...". State that the surviving-types half of the row
needs Python 3.14+. On 3.12 and 3.13 it is skipped, and the row records
`survivor_census: "skipped_unsafe_python"`, because `gc.get_objects()` on those versions
can crash worker threads that are building tuples. Follow `docs_writing` conventions. Do
not edit `CHANGELOG.md`; release-please generates it.

### 4. Tests

- `tests/ace/tui/util/test_gc_policy.py`:
  - Give the `_Collector` stub a `get_objects()` method that records calls and returns a
    small list. Existing tests must keep passing.
  - Add a test with `ENV_CENSUS=1` and `_survivor_census_safe` monkeypatched to return
    `False`. After an idle census, check that `get_objects` was never called, the
    emitted row has `survivor_census == "skipped_unsafe_python"`, `survivor_top == []`,
    and `survivor_count is None`, and `garbage_top` is still present.
  - Add the mirror test with the predicate returning `True`. Check that `get_objects`
    was called once and `survivor_census == "ok"`, with `survivor_count` equal to the
    stub list's length.
  - Add a test that the skip warning is logged only once across two census runs. Use
    `caplog` and the existing rate-limit pattern from
    `test_census_rate_limited_on_idle`.
  - Make the existing census tests deterministic across interpreters by monkeypatching
    the predicate wherever they assert on survivor fields.
- `tests/ace/tui/widgets/` (extend `test_llm_calls_panel_subject_guard.py` or add a
  sibling file that uses the same `MagicMock` `run_worker` pattern):
  - Check that both `_start_background_fetch` and `refresh_llm_calls` call `run_worker`
    with `exit_on_error=False`.
  - Check that a `WorkerState.ERROR` event for the current, non-stale worker renders the
    "Error fetching tool calls" text and logs the worker error.

Do not add a test that races real threads against `gc.get_objects()`. It would depend on
timing and be flaky. The predicate tests plus the doc comment record the hazard.

## Verification

- `just check` must pass. It runs inside `sase tool run`; see `lint_and_test`. Do not
  run `just check-full`.
- Targeted run:
  `pytest tests/ace/tui/util/test_gc_policy.py tests/ace/tui/widgets/test_llm_calls_panel_subject_guard.py`
  (plus any new test file).

## Out of scope

- Rewriting `tuple(genexpr)` call sites throughout the codebase. The victims are
  correct, and the census was what violated CPython's invariant.
- Changing which Python apollo's uv tool install uses. Global installs are human-only.
  Reinstalling sase on apollo with Python 3.14 would bring back the survivor census
  there, but that is an operator choice outside this change.
- Setting `exit_on_error=False` on other `thread=True` workers. Only the LLM-calls panel
  worker is implicated in the observed fatal crash.
