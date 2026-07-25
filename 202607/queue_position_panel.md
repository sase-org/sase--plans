---
tier: tale
title: Show a queued agent's spot in the line in the metadata panel
goal: Selecting a slot-waiting agent shows its admission rank, how many waiters will
  genuinely be admitted before it, and a compact ladder of the queue around it, all
  derived from the same ordered queue that assigns the rank.
create_time: 2026-07-25 12:55:55
status: done
---

- **PROMPT:** [202607/prompts/queue_position_panel.md](prompts/queue_position_panel.md)

# Plan: Show a queued agent's spot in the line

## Context

Selecting a `QUEUED` agent renders one metadata line:

```
Queue: #1 of 2 · requested 3m ago · 0/10 runners
```

In the 120x40 golden that line already wraps onto two rows of the detail pane. It states a rank and then buries it under
two unrelated facts, so the number that matters reads as one clause among three. Worse, a bare rank is not
self-explanatory: `#6 of 12` does not say who is in front, why they are in front, or whether the agents in front will
actually start first — and under this admission gate, several of them frequently will not.

`sase/core/runner_slots/_admission.py::may_start` admits **the first waiter in (priority, FIFO) order whose own
threshold is currently satisfied**. Waiters ahead whose threshold is not satisfied are skipped, not blocking. The
troubleshooting page already states this ("An older low-threshold waiter does not block a later launch whose higher
threshold currently permits it to run"), but nothing in the UI shows it. A user looking at `#6 of 12` cannot tell that
two of the five ahead are drain barriers that will not beat them.

There is also no view of the queue anywhere in ACE. The runners modal lists running processes only. The Agents list
shows `#N/M` per row but in list order, not queue order. So the rank is the single thread the user has, and it is thin.

This matters more now that `w` edits wait priority on a parked row. Choosing a priority without seeing the queue is
guesswork.

## Goal

When a slot-waiting agent is selected, the metadata panel should make its spot in the line **legible at a glance**: the
rank, how many agents will genuinely be admitted first, and the shape of the queue immediately around it.

## Design

Two pieces: a tightened `Queue:` field that leads with the spot, and a compact `QUEUE` ladder section that gives the
spot its context.

### The `Queue:` field

```
Queue: #6 of 12 · 3 ahead · 5m in queue
```

- `#6` bold in the queued cornflower `#5F87FF`, ` of 12` in the same color unbolded — the existing treatment.
- `3 ahead` is the new fact and the heart of this change. See "Who is actually ahead" below. When the count is zero,
  render `at the front` instead of `0 ahead`.
- `5m in queue` replaces `requested 5m ago`: same value from `slot_requested_at`, shorter, and it reads as a duration
  rather than an event.
- Drop the trailing `· {in_use}/{cap} runners` clause. It moves into the section heading, where it belongs with the rest
  of the queue's global context, and dropping it is what keeps this line on one row in the narrow pane.

The `Wait:` line for explicit-threshold `WAITING` rows is unchanged. Its `queue #N of M` clause stays as-is; this change
adds the section for those rows but does not rewrite their field.

### Who is actually ahead

**Rule: an entry ranked above the selected agent is ahead of it if and only if its threshold is greater than or equal to
the selected agent's threshold.**

This follows directly from `may_start`. Threshold here is `wait_runners`, the existing-runner threshold: `cap - 1` for
implicit global-cap waiters, the authored `N` for `%wait(runners=N)`. An entry ahead with an equal or higher threshold
becomes eligible no later than the selected agent does, so the gate takes it first. An entry ahead with a stricter
threshold — the drain barrier case — becomes eligible only after a deeper drain, so the gate skips past it and it never
delays the selected agent.

Treat a missing `wait_runners` as `0`, matching `RunnerSlotWaiter`'s own fallback for invalid threshold values in
`_admission.py`. That makes the unknown case park deepest rather than silently counting as ahead.

`ahead` is therefore exactly derivable from data already on the rows, needs no projection into the future, and is
correct at the instant it is rendered. It is the one derived number in this design, and it is worth deriving because it
is the number the user is actually trying to compute in their head.

### The `QUEUE` section

Rendered immediately after the metadata fields — before the family roster — whenever the selected agent has a
`runner_slot_queue_position`. That covers `QUEUED` rows and explicit-threshold `WAITING` rows alike: both hold a real
rank today, and giving one a ladder while the other gets a bare number would be an arbitrary hole.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
❖ QUEUE · 12 waiting · 10/10 runners
   #1  zz.alpha       ≤0    12m
   … +2 more
   #4  ku.beta              8m
   #5  zt.gamma             6m
 #6  ku--code          p5   5m
   #7  qq.delta             4m
   … +5 more
```

**Heading.** Same visual grammar as the `NEIGHBORS` and `MEMBERS` rosters — a `━` rule, the `❖` mark, an underlined
title, and a dim suffix — accented in the queued cornflower. Rendered through `append_section_heading` so the section
gets a navigation anchor like every other section. Either extract the heading helper out of `_member_roster.py` or
mirror it; the requirement is that it looks like a sibling of the existing rosters, not a new kind of thing.

The runner counts in the heading come from the selected agent's own `runner_slots_in_use` and `wait_runners + 1` — the
same two values the `Queue:` field renders today. Deliberately not from `RunnerCapacitySnapshot`, whose capacity fields
are the neutral fallback when the loader has no effective limit; sourcing from the row keeps the heading correct on both
paths.

**Entry lines.** Rank, name, qualifiers, queue time. Fixed left-aligned columns, name truncated to fit the pane.

- Rank for the selected agent is a reverse-video chip, `#6` styled `bold black on #5F87FF`, exactly the numbered-chip
  treatment `_member_roster.py` uses. That plus a bold name is how ACE already marks a selected row, so the highlighted
  line reads as "this one" without a label. No `← you` annotation: the panel is already about this agent, its name is in
  the `Name:` field directly above, and the words cost real width in a ~50-column pane.
- Rank for entries genuinely ahead: plain cornflower.
- Rank for entries ahead but parked deeper (threshold below the selected agent's): amethyst `#AF87FF`, dim name. These
  are visually demoted because they are not in this agent's way, and the amethyst deliberately matches `WAITING` — they
  are held by their own condition, which is precisely what `WAITING` means everywhere else in ACE.
- Rank for entries behind: dim.

**Qualifiers.** Two optional columns, each rendered only when it carries information:

- `≤N` in amethyst when the entry has an explicit authored threshold.
- `p{N}` dim when the entry's normalized priority differs from the `10` default.

The suppression rule is the point: when every waiter sits at the default priority the column vanishes entirely, and when
one does not, `p5` appears next to the entry that jumped the line and explains the ordering on the spot. Decide per
entry, never per window, so scrolling the window never changes whether a column exists.

**Queue time.** Compact duration since `slot_requested_at`, dim, via the existing `format_compact_duration`. Compute
`local_now()` once per section render and pass it down so every row is measured against the same instant.

**Window.** Deterministic, at most seven entry rows:

- If the queue holds seven entries or fewer, render all of them with no gaps.
- Otherwise render rank 1, the two entries immediately ahead of the selected agent, the selected agent, and the two
  immediately behind — deduplicated and in rank order — with `… +K more` in dim italic standing in for each skipped
  span, matching the existing roster tail idiom.

Rank 1 is always present because the front of the line is exactly where a drain barrier shows up, and "nothing is moving
because rank 1 is `≤0`" is the single most useful thing the section can tell someone.

The section does not fold and does not scroll. The panel's fold machinery is only wired for family containers and lane
owners, so making this foldable for arbitrary rows would mean new fold plumbing for a section that is capped at seven
lines. Note the cap in the docs and leave the full-queue view to the Agents list.

### Explicit non-goals

Call these out in the change description so they read as decisions rather than omissions:

- **No ETA and no "you're next" prediction.** Both would require modeling agent runtimes or future arrivals. A
  confidently wrong time estimate is worse than no estimate, and this design's whole claim is that what it shows is true
  right now.
- **No wire or CLI change.** `runner_slot_queue_position` and `runner_slot_queue_size` keep their current meaning; the
  section is a presentation of data `sase agent list --json` already exposes. Nothing in `waiting.json`, the Rust
  agent-scan wire, or the integrations projection changes.
- **No jump targets.** Queue entries are not numbered into the panel's digit-jump allocator. That allocator is shared
  across the family roster and neighbors, and adding a third claimant is a separate change.
- **No new keybinding or ace option**, so the `?` help modal needs no update.

## Making the spot true

The feature's only claim is that the displayed spot is the real spot. Three things enforce that.

**One queue, computed once.** Add a frozen `RunnerQueueEntry` (identity, presented name, threshold,
`wait_runners_explicit`, normalized priority, `slot_requested_at`, status) and a
`RunnerCapacitySnapshot.queue: tuple[RunnerQueueEntry, ...] = ()`. Populate it in `refresh_runner_slot_context` from the
`waiters` list that is already sorted there to assign `runner_slot_queue_position` — reuse that same traversal, do not
add a pass or a second sort. Populate it on both return branches, including the neutral-limit one: the queue is row
context, like the positions themselves, not a capacity measurement.

The snapshot already flows through the loading boundary to `_agent_runner_capacity` on the app. The panel reads it
through `getattr(app, "_agent_runner_capacity", None)` at the three `build_header_text` call sites that already resolve
`lane_neighbors` this way (`_agent_display.py`, `_agent_display_render.py`, `_agent_display_hints.py`); a small shared
accessor in the prompt-panel package is worth adding rather than triplicating the `getattr`.

The renderer must never sort, filter, or re-derive membership. It locates the selected agent's entry by identity and
slices. Test the consequence directly: every rendered rank equals that agent's own `runner_slot_queue_position`.

**One comparator.** Three copies of the admission sort key exist today and must agree for any of this to be honest:

- `sase/core/runner_slots/_admission.py::_waiter_sort_key` — the real gate
- `sase/ace/tui/models/agent_runner_slots.py::_waiter_sort_key`
- `sase/integrations/agent_list_entries.py::_runner_slot_waiter_sort_key`

They agree only by coincidence, and the TUI copy uses `raw_suffix` where the other two use the record timestamp. Export
one ordering key from `sase.core.runner_slots` and have all three delegate to it. Verify — do not assume — that the
TUI's `raw_suffix` is the same string as the record timestamp the other two use, and pin it with a test that runs
equivalent fixtures through all three paths and asserts identical order.

**Show what the gate does.** The `ahead` rule and the parked-deeper demotion are the visible form of `may_start`. Cover
them with a fixture where a drain barrier is ranked above the selected agent: it must not count toward `ahead`, and it
must render amethyst rather than cornflower.

## Verification

- Unit-test the window selection: a queue of seven or fewer renders whole; a long queue renders rank 1, the ±2
  neighbors, and correct `… +K more` counts for each gap; the selected agent is always present.
- Unit-test rank and `ahead` against thresholds: equal thresholds count as ahead, stricter thresholds do not, a missing
  `wait_runners` is treated as `0`, and `ahead == 0` renders `at the front`.
- Unit-test qualifier suppression: an all-default-priority queue renders no `p{N}` column; a single non-default entry
  renders `p{N}` on that entry only; `≤N` appears only on explicit-threshold entries.
- Test that an agent without a `runner_slot_queue_position` renders no section, and that the section's ranks match
  `runner_slot_queue_position` on the corresponding rows.
- Add the cross-path ordering parity test described above.
- Visual: keep `runner_slot_wait_agents()` as the short-queue case (two waiters, whole queue rendered, no gaps) and
  regenerate its two goldens for the reworded field and the new section. Add a second fixture with roughly nine waiters
  — a `≤0` drain barrier at rank 1, a non-default-priority entry, and enough implicit waiters that both `… +K more` gaps
  appear — select a middle `QUEUED` row, and add a golden for it. Inspect the PNGs; this is a change about how something
  looks, so the goldens are the acceptance evidence.
- The existing runner-slot visual test asserts `requested 3m ago`; update it to the new wording along with any other
  assertion on the old field text.
- Run the focused runner-slot, agent-status, loader, and integrations suites, then `just install` followed by
  `just check`, and `just test-visual`.

## Risks and constraints

- The section renders on the cheap immediate header path that repaints on every `j`/`k`. It must stay pure and
  in-memory: no filesystem access, no awaits, no re-scan, and no work proportional to the full agent list. Reading a
  precomputed tuple and slicing at most seven entries is the entire per-render cost, and it must stay that way.
- `RunnerCapacitySnapshot` gains a field. It is a frozen dataclass with a defaulted addition, so existing construction
  sites are unaffected, but check equality-based assertions in the loader and apply-boundary suites.
- Holding presentation data rather than `Agent` references in `RunnerQueueEntry` is deliberate: selective refresh
  rebuilds rows, and a snapshot holding live references could render a stale name against a fresh rank.
- Amethyst for parked-deeper entries sits next to cornflower in the same column. Confirm against the regenerated goldens
  that the two are separable at a glance; if not, dim the parked rank further rather than moving off the established
  `WAITING` color.
- New module-level symbols must be used or private, or Symvision will fail the build.
