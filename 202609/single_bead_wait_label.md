---
tier: tale
size: small
title: Show the sole waited-on bead ID in agent rows
goal: Make a WAITING agent's single bead dependency identifiable in the Agents list
  while preserving status cues, other wait summaries, and responsive refreshes.
proposed_by: bbugyi200.athena.0hl
status: done
---

# Single bead wait labels

## Outcome and scope

When a WAITING row's effective wait source has exactly one entry in `waiting_for_beads`
and an empty `waiting_for`, replace the bead count with its complete ID. The motivating
screenshot (`~/tmp/screenshots/20260909_183205.png`) shows `(WAITING ◐1)` in the list
while the detail pane already identifies `sase-yz`. The new list text is
`(WAITING ◐ sase-yz)`.

This is a small tale: one coding agent can make a bounded presentation change and verify
it. No backend/domain behavior changes are needed. Existing wait resolution, dependency
counts, cache loading, and release conditions remain the source of truth; the change
belongs in the Python TUI presentation layer.

## Visual and behavioral contract

| Effective wait state                                     | Status parenthetical      |
| -------------------------------------------------------- | ------------------------- |
| One in-progress bead, no agent targets                   | `(WAITING ◐ sase-yz)`     |
| One open bead, no agent targets                          | `(WAITING ○ sase-yz)`     |
| One closed bead still displayed while release catches up | `(WAITING ● sase-yz)`     |
| One bead with a confirmed unknown/unavailable status     | `(WAITING ? sase-yz)`     |
| One bead before status enrichment is available           | `(WAITING sase-yz)`       |
| One in-progress bead plus a five-minute relative wait    | `(WAITING ◐ sase-yz +5m)` |
| Two in-progress beads, no agent targets                  | `(WAITING ◐2)`            |
| One running agent and one in-progress bead               | `(WAITING ▶1 ◐1)`         |

Preserve the amethyst `WAITING` label and dim parentheses. Use the canonical bead status
glyph and its existing bold status color from `src/sase/bead_status_presentation.py`;
this also covers claimed, ready, and snoozed beads. Render the ID in `#FF87D7`, matching
the existing wait-target identity color in the detail pane. Keep one space between
status, glyph, and ID. The ID's color stays stable when the bead status changes. Use
Rich `Text.append` with literal text, without interpreting IDs as markup.

Keep the full ID, including project prefix and dotted suffixes. Add no new icon, label,
brackets, animation, keybinding, or setting. Keep the compact single-line row layout and
existing pane-edge clipping. At ordinary pane widths the example ID must be fully
visible. Long IDs at narrow widths may clip using the existing row policy; retain the
full underlying text and the full ID in the detail pane. Do not introduce a special
abbreviation scheme or redesign row width allocation.

Eligibility uses the actual fields on `wait_display_agent(agent)`, not summed counts or
the row's own associated bead/name. Any entry in `waiting_for` disqualifies the special
label, including completed/missing agents and group references such as `@epic` and
reserved `@default`. Group references can produce zero ordinary-agent counts, so
`counts.agents.has_any` alone is insufficient. Multiple bead entries remain count-based
even when only one has a warm status. Use the existing normalized dependency lists; do
not add display-only deduplication or change wait semantics.

Timers do not disqualify a single-bead label. Preserve the current relative and
absolute-time annotations and their ordering. All other status branches, warning
markers, mixed dependency summaries, list totals, and detail-pane behavior retain their
current formatting and meaning.

## Implementation

1. In `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`, derive the optional
   sole bead ID in the existing WAITING branch from the effective wait source. Keep the
   remaining status/timer code intact.
2. Add a focused presentation helper in `src/sase/ace/tui/wait_status_presentation.py`,
   for example `format_wait_dependency_summary(counts, *, single_bead_id=None)`. The row
   renderer passes the eligible ID; all other cases delegate to the existing
   `format_wait_dependency_status_counts` unchanged. Preserve that count formatter and
   its established count ordering for existing callers/tests.
3. For an eligible ID, use the already-supplied `WaitDependencyStatusCounts.beads` to
   obtain the sole status token; reuse `_wait_bead_status_token` within its module. For
   this case exactly one warm bead contributes a count of one. Empty/absent counts mean
   that only the ID is known: show the ID without inventing an unknown glyph. An
   explicit unknown bead count uses the established amber `?`. If supplied counts
   contradict the singleton contract, retain the count formatter rather than associate
   an arbitrary status with the ID.
4. Keep this helper memory-only. Do not call bead storage, cache resolvers,
   subprocesses, file reads, or new async tasks from rendering. No Rust API, persisted
   schema, or count-dataclass changes are required.
5. Confirm the existing refresh path remains sufficient: `_agent_list_build.py` passes
   counts projected from a cached status snapshot;
   `actions/agents/_loading_bead_warmup.py` recomputes counts and selectively patches
   affected rows. `widgets/_agent_list_render_cache.py` already includes the effective
   `waiting_for`, `waiting_for_beads`, and dependency counts in `agent_render_key`.
   Reuse these inputs and prove that both ID-only changes and status-only changes
   invalidate cached text, including inherited wait sources. Extend production
   cache/refresh code only if a focused regression demonstrates a missing input; no
   full-list refresh should be added for this feature.

## Verification and acceptance

Extend the existing behavioral tests instead of building a new harness:

- `tests/ace/tui/widgets/test_agent_list_wait_dependency_status.py`: update the
  singleton count expectation and cover exact text plus Rich style spans for all six
  canonical bead statuses and unknown. Cover cold/absent counts (ID present, no `?`),
  timers, and a row whose `wait_display_source` has different dependencies from the
  displayed row. Verify mixed waits, multiple beads, time-only waits, and non-WAITING
  rows preserve their current output. Include a bead plus a group target with zero
  ordinary-agent counts and two beads with only one warm status.
- `tests/ace/tui/widgets/test_agent_render_key_wait_state.py`: verify switching only the
  bead ID changes the key even when counts match, for both direct and inherited wait
  sources; verify transitions between singleton and mixed waits. Existing status-count
  key checks should continue passing.
- Keep `tests/ace/tui/test_agent_wait_dependency_status_counts.py` passing to prove the
  underlying counts, ordering, and cold-versus-unknown distinction are intact. Add
  narrowly scoped helper tests there if needed, without duplicating the row test matrix.
- Reuse `tests/ace/tui/test_agents_bead_warmup.py` to verify that enrichment supplies
  fresh counts through selective row patching. Add one focused check connecting a cold
  singleton label to its warmed label; keep the ID stable while the glyph
  appears/changes. Existing no-blocking and no-unnecessary-rebuild checks remain the
  guard for this path.

Add focused PNG coverage alongside
`tests/ace/tui/visual/test_ace_png_snapshots_agents_waiting.py`, using the existing
`AcePage`, loader patches, seeded bead-status cache, and convergence helpers. Use
deterministic local fixtures and clean the cache afterward. Capture normal 120x40 and
narrow 90x32 terminal sizes with a short singleton ID, a dotted long ID, and a
mixed-wait control. Ensure selected and unselected singleton rows are represented and
the detail pane shows the selected target. Inspect the generated PNGs for readable
ID/status contrast, exact spacing, tidy one-line rows, expected clipping, and no layout
overlap; accept only intentional golden changes. Normal width must show the motivating
short ID in full. The narrow fixture must keep the full ID available in the detail pane
even when list content is clipped.

Before coding, read project guidance with
`sase memory read tui_perf.md lint_and_test.md --reason "Implement and verify single-bead wait labels"`.
Run focused tests while developing, then required `just check`. Run the dedicated visual
lane explicitly, since `just check` excludes PNG tests:

```bash
just test -- tests/ace/tui/widgets/test_agent_list_wait_dependency_status.py tests/ace/tui/widgets/test_agent_render_key_wait_state.py tests/ace/tui/widgets/test_agent_list_wait_timing.py tests/ace/tui/test_agent_wait_dependency_status_counts.py tests/ace/tui/test_agents_bead_warmup.py
just check
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents_waiting.py
```

Include any new split test modules in the focused commands. Use
`--sase-update-visual-snapshots` only for intentional snapshot creation/updates, inspect
the results, then rerun the affected visual tests without that flag. Use the
`sase_monitor` skill for long-running verification, as project guidance requires.
Completion means the example renders as `(WAITING ◐ sase-yz)`, cold status loading never
hides the known ID or fabricates an unknown state, selective updates stay accurate, the
other wait forms retain their behavior, and required checks plus inspected visual
snapshots pass.
