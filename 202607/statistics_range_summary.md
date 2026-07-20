---
tier: tale
title: Clarify the Statistics selected time range
goal: 'The SASE Admin Center Statistics tab shows an immediately understandable summary
  of the selected time range alongside its existing exact date-time boundaries, across
  presets, custom ranges, refreshes, and loading states.

  '
create_time: 2026-07-20 12:46:20
status: wip
prompt: 202607/prompts/statistics_range_summary.md
---

# Plan: Clarify the Statistics selected time range

## Context and outcome

The Statistics pane currently renders only `StatsRange.label`, an exact interval such as
`2024-07-01 18:20 EDT – 2024-07-08 18:20 EDT`, in an already dense one-line heading. The exact boundaries are useful for
auditing timezone and inclusive/exclusive behavior, but they are slow to scan and do not tell the user whether the
selection was a preset, a rolling custom duration, a calendar period, or an open-ended range.

Give every successfully resolved range two complementary descriptions:

- a concise, selection-aware label shown first, such as `Today`, `Last 24 hours`, `Last 7 days`, `All time`,
  `Last 14 days`, `March 2026`, `Jul 1–3, 2026`, or `Since Jul 1, 2026`; and
- the existing exact local date-time interval, unchanged, shown immediately after the concise label.

This is presentation behavior. It must not change timestamps, calendar/DST handling, bucket selection, query payloads,
auto-refresh cadence, or the Rust statistics boundary.

## Range description model

Extend the immutable range result in `src/sase/stats/ranges.py` with a dedicated concise display label while preserving
the existing absolute `label` and `start_ts`/`end_ts` contract. Compute the concise value when presets and custom inputs
are resolved rather than reverse-engineering it in the TUI from elapsed seconds; otherwise equivalent durations such as
the `7d` preset and a custom `1w` input lose their user-facing selection semantics, and calendar/DST ranges can be
mischaracterized.

Use the existing preset metadata as the source of truth for scan-friendly preset names: `Today`, `Last 24 hours`,
`Last 7 days`, `Last 30 days`, `Last 90 days`, and `All time`. For custom inputs, centralize a pure formatter alongside
the parser with these rules:

- spell relative units with correct singular/plural grammar (`Last 1 hour`, `Last 2 days`, `Last 4 weeks`);
- render a completed month as its calendar name and year (`March 2026`), and identify a currently capped month as being
  to date;
- compact closed date spans without losing the year or introducing ambiguity across month/year boundaries; and
- render open-ended ranges as `Since <date>`. If a requested future endpoint is capped at now, describe the effective
  range rather than implying that future activity is included.

Keep the exact interval generation and its current closed-end display convention intact. Update public exports,
fixtures, and construction sites deliberately so every `StatsRange` used by the pane has both labels instead of relying
on a misleading fallback.

## Statistics pane presentation

Refactor `src/sase/ace/tui/modals/statistics_pane_rendering.py` so the main heading remains responsible for the current
view/group, project filter, and load/update status, while a dedicated range renderable presents:

`Range: <concise label>  ·  <exact date-time interval>`

Add that one-line range row in `src/sase/ace/tui/modals/statistics_pane.py` between the heading and view tabs, and style
it in `src/sase/ace/tui/styles.tcss`. Emphasize the concise label, retain a clearly legible exact interval, and place
the concise text first so a constrained-width ellipsis preserves the most useful identification. Keep the row
non-wrapping and consistent with the pane's centered header hierarchy.

Update the heading and range row together whenever range state is selected or re-resolved: initial composition, preset
cycling, custom submission, periodic/manual refresh, loading, success, empty, and error states must never show a stale
summary paired with new absolute boundaries. Reconcile the extra row with the body/loading/error height budgets so the
view strip, data panels, and key hints remain visible at supported modal sizes. The render path must remain pure and
cheap: no I/O, timestamp queries, subprocesses, new timers, or extra worker loads.

## Tests and visual verification

Expand `tests/stats/test_ranges.py` to lock down both descriptions while retaining all existing timestamp and timezone
assertions. Cover every preset, relative custom units including singular/plural cases, completed and current months,
same-month and cross-month/year closed spans, open-ended spans, future-end capping, and DST-sensitive absolute labels.

Extend `tests/ace/tui/test_statistics_pane.py` to assert that the dedicated range row contains the concise and exact
labels on initial load and updates immediately and coherently when cycling presets and accepting a custom range. Confirm
that refresh/load state changes preserve both labels and that query calls still receive the same resolved timestamps.

Update the deterministic range fixture and all affected Statistics PNG goldens under
`tests/ace/tui/visual/snapshots/png/`. Inspect the overview, runtime, projects, drill-down, empty, and loading images to
verify the new hierarchy is readable, the exact range remains visible, and the added row does not clip panels or hints.
Add a focused custom-range visual case only if the existing default-range golden plus interaction assertions do not make
the transition visually reviewable.

Run the focused range and Statistics-pane tests, then `just test-visual` for the intentional snapshot changes. Because
implementation files will change, run `just install` as required for an ephemeral workspace and finish with
`just check`. Any accepted golden update should be limited to the Statistics layout described here.

## Acceptance criteria

- A user can identify the active range at a glance without parsing timestamps.
- The exact configured-timezone start and end date-times remain visible next to the concise description.
- Preset, relative, calendar, open-ended, and capped selections receive accurate and deterministic concise labels.
- Range changes and refreshes cannot leave concise and exact labels out of sync.
- Statistics query boundaries, refresh behavior, and responsiveness are unchanged.
- Focused behavioral tests, Statistics visual snapshots, and the repository-wide check all pass.
