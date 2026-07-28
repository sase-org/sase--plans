---
tier: tale
title: Fix ace TUI crash on queued agents with explicit runner/priority waits
goal:
  Selecting a QUEUED agent that declared an explicit runner or priority wait renders its detail header instead of
  crashing `sase ace`, the duplicated `build_wait_lanes` call site that caused the regression is collapsed into one, and
  a regression test covers the path.
create_time: 2026-07-28 18:20:17
status: done
---

- **PROMPT:** [202607/prompts/queued_wait_lane_crash.md](prompts/queued_wait_lane_crash.md)
- **AGENTS:**
  - [bbugyi200.athena.no--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.no.md#member-code)
  - [bbugyi200.athena.no--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.no.md#member-plan)
- **COMMITS:**
  - [212472e](https://github.com/sase-org/sase/commit/212472e3acc6c84d639c269c8110000bf1cfa49a) — fix(ace): render
    explicit waits for queued agents

# Plan: Fix ace TUI crash on queued agents with explicit runner/priority waits

## Problem

`sase ace` crashes with:

```
TypeError: build_wait_lanes() missing 1 required keyword-only argument: 'tribe_wait_bindings'
```

## Root cause

This is a **semantic merge conflict** between two commits that each passed `just check` in isolation but were never
checked together:

1. `ed04c42f2` (`feat(ace): display tribe wait bindings`, 17:48) added a **required** keyword-only parameter
   `tribe_wait_bindings` to `build_wait_lanes()` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py:132-140`, and updated the single call site that existed
   at the time.

2. `d8c2f5019` (`feat(agents): classify all runner-slot waiters as queued`, 18:08) added a **second, new** call site
   inside the `QUEUED_STATUS` branch of `_append_wait_field()` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py:223-233`. That branch was authored against
   the pre-`ed04c42f2` signature, so it omits `tribe_wait_bindings`.

The two call sites now disagree: `_agent_display_header_metadata.py:243-250` (the general branch) passes
`tribe_wait_bindings=tribe_wait_bindings`; the queued branch at line 225 does not.

### Confirmed on the current tree (master @ `641229f89`)

Reproduced directly:

```
$ .venv/bin/python -c "
from tests.ace.tui.widgets._agent_display_helpers import make_agent
from sase.ace.tui.widgets.prompt_panel._agent_display_parts import build_header_text
a = make_agent(status='QUEUED', wait_runners=2, wait_runners_explicit=True,
               slot_requested_at='2026-07-28T12:00:00Z',
               runner_slot_queue_position=1, runner_slot_queue_size=2)
build_header_text(a, cheap=True)"
TypeError: build_wait_lanes() missing 1 required keyword-only argument: 'tribe_wait_bindings'
```

`mypy` already flags it, and it is the **only** error in the tree:

```
$ .venv/bin/mypy
src/sase/.../_agent_display_header_metadata.py:225: error: Missing named argument
  "tribe_wait_bindings" for "build_wait_lanes"  [call-arg]
Found 1 error in 1 file (checked 2473 source files)
```

So this is an isolated, single-site defect — no other latent breakage from those commits.

### User-visible trigger

`_append_wait_field()` reaches the broken call only when **both** hold:

- `agent.status == QUEUED_STATUS`, **and**
- `wait_agent.wait_runners_explicit or wait_agent.wait_priority_explicit` is true (the guard at
  `_agent_display_header_metadata.py:221-222` returns early otherwise).

That is: focusing a queued agent that declared an explicit `%runners` / `%priority` wait in the ace Agents tab crashes
the TUI.

### Why tests did not catch it

`tests/ace/tui/widgets/test_agent_wait_section.py::test_queued_state_keeps_single_line_queue_field` is the only test
covering the queued header branch, and it builds its agent with `wait_runners=2` but **without**
`wait_runners_explicit=True` and without `wait_priority_explicit=True`. It therefore hits the early `return None` at
line 222 and never reaches the `build_wait_lanes` call below it. No test in `tests/ace/tui/widgets/` exercises queued +
explicit through `build_header_text`.

## Fix

Do **not** just add the missing kwarg. Two identical-tail call sites are what allowed the signatures to drift; collapse
them so a future required parameter can only be added in one place.

### Step 1 — collapse the duplicated call site

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`, rewrite `_append_wait_field()` so
`build_wait_lanes()` is called exactly once and the section-emitting tail (build `ResponsiveWaitSection`, record
`start`, `append_text`, record `responsive_ranges[WAIT_SECTION_ID]`, return the section) appears exactly once.

The queued branch differs from the general branch in only two ways:

1. It first appends the single-line `Queue: ...` field (lines 197-220) — keep that exactly as is, including its position
   **before** the wait lanes in `text`.
2. It keeps only the `"runners"` lane.

So the structure becomes: render the `Queue:` line and set a "runners-only" flag inside the `QUEUED_STATUS` branch
(preserving the early `return None` at line 222 when neither `wait_runners_explicit` nor `wait_priority_explicit` is
set), then fall through to one shared `build_wait_lanes(...)` call that passes **all five** keyword arguments including
`tribe_wait_bindings=tribe_wait_bindings`, apply the runners-only filter when the flag is set, then run the shared emit
tail.

Behavior must be byte-identical to the pre-regression intent:

- Queued + explicit → `Queue:` line followed by only the `[runners]` lane.
- Queued + explicit but no runners lane produced → `Queue:` line only, return `None`.
- Queued + not explicit → `Queue:` line only, return `None`.
- Non-queued → all lanes, no `Queue:` line.
- `responsive_ranges[WAIT_SECTION_ID]` must still span exactly the wait block and must **exclude** the `Queue:` line
  (i.e. `start` is captured after the `Queue:` line is appended).

Note that in the queued path `tribe_wait_bindings` cannot change the rendered output, because the only lane it
influences (`"tribes"`) is filtered out. Passing it is about restoring a type-correct, single-source-of-truth call — not
about changing queued rendering.

Keep the `"runners"` lane tag as a plain string literal matching the existing style in `_agent_wait_section.py` unless
the surrounding code already uses a named constant.

### Step 2 — add the regression test

In `tests/ace/tui/widgets/test_agent_wait_section.py`, add a test next to
`test_queued_state_keeps_single_line_queue_field` that goes through `build_header_text` with a queued agent that
**does** set `wait_runners_explicit=True`, e.g.:

```python
make_agent(
    status="QUEUED",
    waiting_for=["suppressed"],
    wait_runners=2,
    wait_runners_explicit=True,
    slot_requested_at="2026-07-28T12:00:00Z",
    runner_slot_queue_position=1,
    runner_slot_queue_size=2,
)
```

Assert that it does not raise, that the `Queue:` line is present, that the `[runners]` lane **is** rendered, and that
the suppressed non-runner lanes (e.g. `[agents]`) are **not**. This test fails with the reported `TypeError` on the
current tree and passes after Step 1.

Add a second case covering queued + `wait_priority_explicit=True` (the other half of the line 221 guard) if `make_agent`
supports it — check `tests/ace/tui/widgets/_agent_display_helpers.py` for the available fields and skip this sub-case if
the helper cannot express it.

## Verification

Run from the workspace checkout root (`just install` first — sase workspaces are ephemeral and dependencies may be
stale):

```bash
just install
.venv/bin/mypy   # must report 0 errors; confirms the call-arg error is gone
just test        # or at minimum:
.venv/bin/pytest tests/ace/tui/widgets/test_agent_wait_section.py \
                 tests/ace/tui/widgets/test_agent_queue_section.py \
                 tests/ace/tui/test_agent_runner_slots.py -q
just check       # required before reporting completion (file changes were made)
```

Sanity-check the fix reproduces no longer by rerunning the repro snippet from the Root cause section; it must print a
header instead of raising.

`just check` also runs `symvision` and `toobig`. If the refactor pushes `_agent_display_header_metadata.py` or
`_append_wait_field` past a size gate, prefer extracting the shared emit tail into a small module-private helper (e.g.
`_emit_wait_section`) rather than reintroducing a duplicated `build_wait_lanes` call.

## Out of scope

- Do not give `tribe_wait_bindings` a `None` default in `build_wait_lanes`. It is required on purpose; that requirement
  is exactly what let `mypy` catch this, and defaulting it would convert future drift into silent wrong rendering
  instead of a type error.
- No changes to `_agent_wait_section.py` logic, tribe-binding resolution, or the runner-slot queue model are needed.
