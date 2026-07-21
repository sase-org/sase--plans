---
tier: tale
title: Runners Statistics experience
goal: The Admin Center Statistics pane gains a responsive Runners view that makes
  runner occupancy, concurrency trends, current-limit context, and busiest slices
  accurate, discoverable, and legible at both wide and narrow sizes.
bead: sase-8j.3
parent: sase/repos/plans/202607/runners_statistics.md
create_time: 2026-07-21 17:38:23
status: done
---

- **PROMPT:** [202607/prompts/runners_statistics_experience.md](prompts/runners_statistics_experience.md)

# Plan: Runners Statistics experience

## Context and boundaries

The shared Rust response and the frozen Python runner view models already expose the effective analysis window, exact
summary metrics, occupancy rows, bounded trend slices, skipped-data counts, and the current global runner limit captured
inside the existing off-thread Statistics load. This work is the presentation phase: it must not reproduce interval or
concurrency math, add artifact/config reads to rendering, create another refresh path, or imply that today's global
limit was the historical/project-specific limit.

Keep the existing worker-backed, coalesced Statistics loader and the current range, project, refresh, keyboard, mouse,
scroll, and help interactions. Runner rendering must remain pure and bounded by the response's occupancy and trend
collections. A fixed idle window and a carry-in runner with no launches are valid Runners results even when the
launch-oriented aggregate view is empty; only an unavailable runner payload should use runner-specific no-data guidance.

## Statistics navigation and responsive tabs

Extend the reusable panel-tab entry with an optional compact label and teach the existing width-sensitive strip to
select that label while preserving active styling, uppercase treatment, separators, centering, and click hit ranges. Opt
the Statistics strip into a measured compact threshold, using concise but unambiguous labels (including `Plans/Q`) so
all eight views fit at the supported narrow size without changing other tab-strip consumers.

Add `Runners` immediately after `Runs` to the Statistics view literal, order, labels, descriptions, legends, dispatcher,
and help completeness surfaces. Verify keyboard cycling and mouse selection agree with the new order and reuse the
already-loaded composite result rather than causing a reload.

## Runners composition and responsive presentation

Add a focused, pure runner presentation helper/renderer and wire it into the Statistics body. Present five consistent
summary cards for Peak, Average, Busy, Runner time, and Current limit. Include exact peak/busy durations and shares,
derive time at/above today's limit only from the provided occupancy rows, label the limit as a global current reference
when project-scoped, and call out an observed peak above that reference without clipping the display.

Render a fixed-zero-baseline concurrency timeline whose scale is `max(observed peak, current limit)`: cyan average fill,
magenta peak markers, and a gold current-limit reference/legend must keep average, peak, and current capacity visually
distinct. Bound the chart to the returned slices and the available terminal width while retaining the selected window's
start/end context. Add an exact occupancy table containing every returned runner count, human duration, percentage, and
a proportional bar; style zero quietly, ordinary occupancy with the cyan/magenta family, and at/above-current-limit
comparisons with gold/red caveated as a present-day comparison.

Add a deterministic Busiest slices table sorted by peak, then average, then time, showing slice label, peak, average,
busy duration, and runner time. Use a balanced wide composition for timeline/distribution where legible and stack all
sections in the existing vertical scroll at narrow widths. Repaint only when a responsive layout threshold changes,
keeping resize and message-pump work bounded and free of I/O.

## State, explanations, and fixtures

Special-case Statistics emptiness so an available idle or carry-in Runners payload renders even when
`StatisticsViews.empty` is true. Provide distinct guidance for an absent/partial Runners payload, retain the existing
loading and error behavior, and surface skipped malformed/invalid data without turning a partial valid snapshot into a
failure.

Update the inline legend and contextual help to plainly define runner eligibility, overlap/carry-in/live clipping,
yielded human-wait exclusions, zero occupancy, time-weighted averages, all-time effective coverage, project scoping,
skipped malformed data, and the current-global-limit caveat. Extend deterministic test and visual fixtures with
populated, wholly idle, carry-in, over-limit, project-filtered, and missing-runner payloads, without coupling UI tests
to live configuration or artifacts.

## Verification and visual acceptance

Add focused tests for compact/full tab labels and click ranges, eight-view keyboard/mouse navigation, no-reload view
switching, runner-specific empty semantics, summary and caveat copy, fixed-baseline chart output, exact occupancy rows,
deterministic busiest-slice ordering, help completeness, and bounded wide/narrow rendering. Add dedicated populated
Runners PNG snapshots at 120x40 and 90x30, inspect their actual/diff artifacts, and deliberately update only existing
Statistics snapshots whose tab strip or help content changed.

Run `just install` before repository checks, then the focused Statistics/tab tests and dedicated visual suite. Re-run
visual tests after accepting only intentional snapshot changes, run `just check`, and inspect the final diff for
unrelated files, accidental generated changes, or changes outside this phase.
