---
tier: tale
title: Show the current shell runtime on agent family rows
goal: "Sequential agent-family container rows show the currently running shell's elapsed
  runtime before the family's total runtime, so users can see both values without
  expanding the family.

  "
size: medium
proposed_by: bbugyi200.athena.0bo
---

- **AGENTS:**
  - [bbugyi200.athena.0bo](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bo.md)
- **COMMITS:**
  - [184fa9a](https://github.com/sase-org/sase/commit/184fa9aed8b71a950393c8d1eba67bcee6141766)
    — feat(ace): show current family shell runtime

# Plan: Show the current shell runtime on agent family rows

## Context

ACE currently gives an active row a `🏃‍♂️` marker followed by one elapsed value. For an
agent-family container, that value comes from `compute_row_runtime()` and is the union
of the loaded family members' run intervals, with approval and human-wait gaps excluded.
The concrete shell rows revealed by expanding the family each show their own elapsed
runtime, which forces users to expand a family just to learn how long its current shell
has been running.

The agent-list renderer already has all required state in memory through the family
container and its normalized `runtime_children`, and active time text is already updated
through `AgentList.patch_active_runtime_rows()`. This change must reuse those paths: it
must not add disk access, subprocess work, a new timer, or a full-list rebuild to the
render path.

## User-visible contract

- When a sequential family has a concrete in-flight shell with a measurable runtime,
  render the family suffix as `🏃‍♂️ <current-shell-runtime> / <family-total-runtime>`. The
  current-shell value is always to the left of `/`; the existing aggregate family value
  remains on the right.
- Render that same suffix on the family container whether its member rows are collapsed
  or expanded.
- Treat both agent shells and family-attached monitor proc shells as concrete shells. If
  a monitor is the current in-flight shell, its own elapsed runtime is the left-hand
  value.
- Preserve the existing aggregate-runtime definition and compact duration formatting. In
  particular, the right-hand value remains the interval union rather than a sum that
  double-counts overlap.
- If a family has no in-flight shell, or the active shell has not recorded a usable
  `BEGIN`/run-start timestamp, omit the current-shell segment and `/` rather than
  presenting waiting or unknown time as live execution.
- Leave standalone agent rows, workflow-step rows, clan containers, execution-neutral
  parallel-family rows, completed-family timestamps, unread markers, user-paused
  markers, file-change pencils, and right-edge alignment unchanged.

## Implementation

### 1. Resolve the current concrete shell from the family projection

Extend `src/sase/ace/tui/models/agent_family_members.py` with a small, pure helper that
returns the current in-flight shell only for a sequential-family container. Build on
`concrete_family_shell_rows()` so the lookup shares the existing rules for promoted
roots, concrete workflow planner steps, serial continuations, nested monitors,
synthetic/parallel exclusions, ordering, deduplication, and cycle safety. Reuse
`agent_row_is_in_flight()` for the execution predicate and select the newest in-flight
entry in canonical chain order as a deterministic safeguard against a transient snapshot
containing more than one active candidate.

Keep this lookup in-memory and linear in the already-loaded family subtree. Return no
shell for clans, parallel projections, non-container rows, or families whose only
pending member is queued, dependency-waiting, or paused for a user.

### 2. Compute one shell's own runtime without re-aggregating its descendants

Refactor `src/sase/ace/tui/models/agent_time.py` to expose a focused display helper for
a concrete shell's leaf runtime. It should reuse `_leaf_runtime_interval()` and the
existing compact-duration formatter so `BEGIN` handling, plan handoffs, question
continuations, retries, terminal timestamps, and running monitors retain exactly the
same semantics as their expanded shell rows.

The helper must deliberately ignore `runtime_children`; otherwise asking for a promoted
root shell or a shell that owns a nested monitor would accidentally return the family or
monitor aggregate again. Keep `compute_row_runtime()` unchanged as the source of the
family's total runtime. Use one captured reference time for both calculations so the two
displayed values cannot straddle different seconds when a caller omits `now`.

### 3. Compose the two values in the existing runtime suffix

Update `src/sase/ace/tui/widgets/_agent_list_render_layout.py` so
`build_runtime_suffix()` obtains the current concrete shell for a sequential family and
places its leaf elapsed value before the aggregate elapsed value. Associate the existing
gold live marker with the current-shell segment and render a literal, spaced separator,
yielding plain text such as `🏃‍♂️ 3m05s / 8m42s`.

Retain the current single-value live rendering as a fallback when there is no eligible
current-shell duration. Do not disturb the timestamp prefix, unread/failed/user-paused
marker precedence, file-change suffix composition, compact-duration styles, or
`assemble_padded_option()` alignment behavior. The longer suffix should participate in
the existing target-width calculation exactly like any other runtime suffix.

### 4. Keep cached and selectively patched rows coherent

Audit `src/sase/ace/tui/widgets/_agent_list_render_cache.py` against every field used to
select and time the current shell. Extend the explicit runtime signature where needed so
a family-row cache entry changes when the selected active shell changes, a monitor
becomes active or settles, or the active shell's timing metadata changes. Preserve the
existing per-second `now` quantization for ticking rows.

Continue to rely on `runtime_suffix_ticks()` and `AgentList.patch_active_runtime_rows()`
for live updates. Do not introduce a new refresh route: the family total and
current-shell duration must advance through the same in-place `patch_agent_row()`
operation, preserving option count, marks, fold state, and right-edge alignment.

### 5. Cover runtime semantics, rendering, caching, and the collapsed experience

Add focused tests in the existing agent-family/runtime suites:

- In `tests/ace/tui/models/test_agent_family_members.py`, cover current-shell selection
  for an active promoted root, a later serial continuation, and a nested running
  monitor; also cover no-active-shell, queued/waiting-only, parallel-family, and
  transient multiple-active-candidate cases.
- In `tests/ace/tui/widgets/test_agent_list_runtime_compute.py` (or the nearest focused
  runtime-model module), prove that leaf runtime computation ignores descendant
  aggregation while preserving the expanded shell row's timing semantics.
- In `tests/ace/tui/widgets/test_agent_list_runtime_rendering.py`, assert the exact
  `🏃‍♂️ <current> / <total>` order and spacing for active families, including a family
  whose root is the current shell and one whose current shell is a nested monitor.
  Assert that completed/paused families and non-family active rows retain their current
  suffixes.
- In `tests/ace/tui/widgets/test_agent_list_runtime_patching.py`, prove one in-place
  tick advances both values without rebuilding rows, and that the suffix is identical
  for collapsed and expanded family-container rendering.
- In the render-cache tests, mutate a family's active-shell choice and timing metadata
  to prove the cached suffix invalidates instead of reusing the previous member's
  runtime.
- Add or adapt a focused fixture in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py` so a running family
  visibly carries the two-value suffix while collapsed, then expand it and verify the
  family row keeps the same suffix alongside the member rows. Update only the affected
  PNG golden(s).

### 6. Document the family-specific suffix

Update the runtime-suffix description in `docs/ace.md` to show the two-value active
family format and explain that the left value is the current concrete shell while the
right value is the aggregate family interval. Cross-reference or add the same concise
rule in `docs/agent_families.md` near the ACE family-row behavior so users understand
that expanding a family is no longer required to inspect the active shell runtime.

## Verification

1. Run `just install` before repository checks so the workspace has the current Python
   and Rust dependencies.
2. Run the focused family projection, runtime compute/render, cache, patching, and PNG
   snapshot tests while iterating. Accept the intentional targeted visual golden with
   `--sase-update-visual-snapshots`, then rerun it under exact pixel comparison.
3. Run `just test-visual` to catch unintended Agents-tab layout changes in other PNG
   snapshots.
4. Run `just check` for the required whole-repository lint gates and diff-scoped test
   lane. Escalate to monitored `just check-full` only if `just check` reports an unusual
   selection/escalation or the final diff enters the repository's broadening set.

## Acceptance criteria

- A collapsed or expanded sequential-family container with an actively running shell
  shows `🏃‍♂️ <current-shell-runtime> / <family-total-runtime>` at the right edge.
- The left value matches the runtime shown on that shell's expanded row at the same
  reference time, and the right value matches the family's pre-change aggregate total.
- Family-attached running monitors are eligible current shells; waiting, queued,
  human-paused, settled, synthetic, and parallel rows are not.
- Both values tick through selective row patching with correct cache invalidation and
  without new I/O, timers, event-loop work, or full-list rebuilds.
- Rows outside the active sequential-family case retain their existing markers,
  timestamps, duration semantics, alignment, and visual appearance.
- Focused tests, visual snapshots, and `just check` pass.

## Non-goals

- Changing how family total runtime is aggregated or moving that existing aggregation
  across the Rust/Python boundary.
- Adding the two-value presentation to CLI/JSON output, clan rows, parallel families, or
  standalone agents.
- Showing queue time, dependency-wait time, or human-response wait time as the current
  shell runtime.
