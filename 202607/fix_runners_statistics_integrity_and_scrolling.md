---
tier: tale
title: Correct Runners statistics and scrolling
goal: 'The Admin Center Runners view reports runner-slot occupancy from valid bounded
  runs and genuinely live agents, excludes abandoned or never-started artifacts, and
  supports configurable Ctrl+D/Ctrl+U half-page scrolling throughout the Statistics
  pane.

  '
create_time: 2026-07-22 06:26:31
status: done
---

- **PROMPT:** [202607/prompts/fix_runners_statistics_integrity_and_scrolling.md](prompts/fix_runners_statistics_integrity_and_scrolling.md)

# Plan: Correct Runners statistics and scrolling

## Context and confirmed failure modes

The Runners view is intentionally a presentation of the shared Rust statistics contract, not a launch-count chart. A
runner is an `ace-run` user agent that participates in global runner-slot admission, and occupancy is the time integral
of those agents' active, non-human-wait intervals.

Reproducing the screenshot's exact 24-hour window against the current artifact index yields a 30-runner peak, a 25.16
average, and a constant floor of 25 runners. Those 25 intervals all come from old records dated July 8–20 that have
`run_started_at` but no `stopped_at`, no usable `done.json`, no live marker, and dead or reused PIDs. The Rust interval
builder currently treats every terminal-less record as live through the query end, so abandoned artifacts become
permanent carry-in runners. The same query also reports 391 invalid intervals because never-started historical artifacts
pass the broad overlap-candidate filter even though a record without `run_started_at` never held a runner slot.

The scroll failure is independent. `StatisticsPane` owns a `VerticalScroll`, but its focused keymap contains only
view/range/group/project/refresh/help actions. No Statistics binding or action routes `Ctrl+D` and `Ctrl+U` to
`#statistics-body-scroll`, so the app-level bindings fall through to unrelated detail-pane behavior behind the Admin
Center.

## Shared runner-occupancy correction

Implement the data fix in the linked `sase-core` repository, where the concurrency definition is shared by every
frontend. Preserve the current half-open interval, human-wait subtraction, project filtering, event sweep, dense
distribution, and bounded trend contracts.

Refine runner candidate classification so a record without a valid `run_started_at` is a never-admitted/waiting artifact
rather than an invalid active interval. Reject it before interval aggregation, and keep the cached-column fast path so
irrelevant historical rows are not decoded merely to prove that they never ran.

For a structurally eligible record with a start but no valid terminal boundary, cap it at the query end only when
current host evidence verifies that the agent is still live. Align this proof with the established
runner-admission/listing semantics: validate the recorded PID, reject explicit stops, dead/zombie processes, and clear
PID-reuse mismatches, and use the appropriate workspace claim or home running marker as corroborating identity where
available. Factor the liveness decision behind a small injectable boundary so aggregation tests are deterministic and
the domain math remains independent of `/proc` fixtures. A terminal-less record that cannot be verified as live must not
contribute occupancy; surface it through the existing partial-snapshot diagnostics unless a narrowly scoped,
backward-compatible diagnostic field is demonstrably needed. Do not repair, delete, or synthesize terminal markers in
the user's artifact history.

Retain correct historical behavior for records with a real terminal after the selected end, carry-in records that began
before the selected start, currently live records, and yielded plan/question windows. Re-check all conservation
identities after filtering: distribution durations equal the analysis span, weighted distribution duration equals
runner-seconds, busy plus idle equals the span, and trend totals agree with the exact distribution. Keep fixed empty
ranges valid and preserve the all-time effective-start contract.

Add Rust coverage through both focused interval tests and the real scanner/index query path for:

- an abandoned terminal-less record with a dead or mismatched PID, proving it does not create an all-window occupancy
  floor;
- a verified live record, proving legitimate live/carry-in clipping still works;
- old records without `run_started_at`, proving they neither occupy a slot nor inflate invalid-interval diagnostics;
- mixed stale, live, terminal, question-yielded, project-filtered, and simultaneous boundary records; and
- the existing exact distribution/trend bounds and conservation checks.

Keep the Python query facade thin. Update PyO3/binding smoke coverage only as needed to lock in the corrected payload
behavior, and avoid moving liveness or concurrency math into Textual rendering.

## Statistics-pane scrolling contract

Extend the focused `StatisticsPaneKeymaps` scope with configurable half-page scroll down/up actions whose built-in
defaults are `ctrl+d` and `ctrl+u`. Update the keymap metadata, bundled default configuration, JSON schema,
registry/default/conflict tests, effective-help bindings, and customized-key coverage together so the default
configuration remains the source of truth. These pane-local actions may overlap app-level bindings because they are
active only while Statistics owns focus; a focused custom-range input must retain its normal text-editing behavior.

Implement the actions on `StatisticsPane` by querying the already-mounted `#statistics-body-scroll` and moving it by
half of the visible scrollable region, with a minimum one-row movement, clamping delegated to Textual, and animation
disabled for immediate key-to-paint feedback. Do no I/O, model rebuilding, or statistics reload on the keystroke path.
The behavior should work for every overflowing Statistics view, including the long Runners occupancy and busiest-slice
sections, without changing `g`, which is already the configured group action.

Advertise the effective scroll keys concisely in the Statistics footer and contextual controls, using width-aware or
combined copy so the supported narrow layout does not collide. Add an integration test that mounts a long Runners
result, focuses the pane, presses the default `Ctrl+D` and `Ctrl+U`, and asserts the body scroll position moves down and
back up without switching views or reloading data. Repeat dispatch coverage with customized scroll bindings, and verify
that unrelated Admin Center panes and the custom-range input do not receive the Statistics action.

## End-to-end acceptance and verification

Use a deterministic index fixture modeled on the reproduced dataset: several stale open records, valid completed
overlaps, a verified live record, and never-started records in the same 24-hour window. Assert the corrected
peak/average/distribution from known intervals rather than merely asserting that the peak is below today's limit.
Through the Python view model and Runners renderer, verify that summary cards, timeline, occupancy rows, busiest slices,
and skipped-data caveats all describe the same corrected snapshot.

Re-run the screenshot window as a local read-only sanity check after rebuilding the binding and confirm that the 25
abandoned artifacts no longer establish a constant floor, while agents genuinely live at query time remain represented.
Inspect the 120x40 and 90x30 Statistics/Runners PNG output after the footer/control change; accept only intentional
snapshot updates and confirm the lower Runners sections are reachable by keyboard at both sizes.

In `sase-core`, run `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
`cargo test --workspace`. In the main SASE workspace, run `just install` before checks, then focused statistics
keymap/interaction/view/binding tests, `just test-visual`, and the required `just check`. Review both repository diffs
for accidental version, schema, generated, snapshot, memory, or unrelated changes.

## Scope boundaries

- Do not mutate historical artifacts or paper over the defect with a UI-side cap at `max_running_agents`.
- Do not count hooks, Axe runners, serial family bookkeeping, queued agents, or non-agent workflow steps.
- Do not infer a historical capacity-limit timeline from today's configuration.
- Do not add filesystem/process work to render, resize, timer, or key handlers; the existing threaded/coalesced
  Statistics load remains the only refresh route.
- Do not edit SASE memory or generated instruction files.
