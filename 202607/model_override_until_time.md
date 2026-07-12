---
create_time: 2026-07-10 17:49:36
status: done
prompt: 202607/prompts/model_override_until_time.md
tier: tale
---
# Models Panel: Override Until a Specific Time

## Product Direction

The Models panel should let a user express an override end in the way they are already thinking about it: either as an
amount of time ("for two hours") or as a wall-clock target ("until 5:00 PM"). The new path should be visibly distinct
from custom durations, fast from the keyboard, and explicit about the timezone and resolved date before it writes any
state.

The existing model picker and duration presets remain unchanged. After choosing a model, the duration popup gains one
new mnemonic choice:

```text
                    Override Duration

  1   15 minutes              Quick model checks
  2   30 minutes              A short task
  3   1 hour                  A focused session
  4   2 hours                 A longer implementation block
  5   4 hours                 Half a day
  6   Until cleared           Persists until removed

  t   Until a specific time   Choose a local clock time or date
  c   Custom duration         Enter minutes or hours

  esc Cancel
```

Choosing `t` opens a compact, focused time-entry popup rather than overloading the custom-duration field:

```text
                    Override Until

  Time   [ 5:00 PM                                  ]

         Ends today, Fri Jul 10 at 5:00 PM EDT
         in 2h 18m  |  America/New_York

  enter Set override   |   esc Back
```

This gives the common case a low-keystroke path (`t`, type, Enter), while the preview removes the dangerous ambiguity
around "today or tomorrow," configured timezone, and daylight-saving transitions.

## Interaction and Time Semantics

- Keep numeric presets `1` through `6`, `c` for custom duration, and all current cancellation behavior stable. Add `t`
  as the memorable shortcut and visible row for an absolute target.
- Accept friendly, deterministic local-time forms such as `5pm`, `5:30 PM`, `17:30`, `today 5pm`, `tomorrow 9am`, and
  `2026-07-12 09:00`. Continue to understand compact `HHMM` for users familiar with SASE wait-time syntax. Avoid
  locale-ambiguous numeric dates such as `7/12`.
- Interpret an undated clock time as its next occurrence in the configured SASE timezone: later today when possible,
  otherwise tomorrow. An explicit `today` or explicit date that is not in the future is an error rather than silently
  changing the requested day.
- Show a live preview containing the resolved weekday/date, 12-hour local time, timezone abbreviation, configured IANA
  timezone when available, and remaining duration. The preview is the confirmation; pressing Enter persists exactly that
  instant.
- Validate daylight-saving boundaries deliberately. Reject nonexistent local times with an actionable message. When a
  wall time is ambiguous during a fall-back transition, require an offset-qualified ISO target (and explain the two
  valid offsets) rather than silently selecting the wrong occurrence.
- Keep invalid input in the popup, retain focus, and show a concise inline error. `Esc` returns to the duration choices
  so the user can choose a preset instead; a second `Esc` cancels the override flow as it does today.
- After success, notify with the exact target (for example, `@coder override: CODEX(o3) until Fri Jul 10, 5:00 PM EDT`).
  The compact Models-panel and top-bar chips continue showing time remaining, preserving their alignment and glanceable
  behavior.

## Technical Plan

1. **Add an exact-expiry operation to the existing override store.**
   - Extend `src/sase/llm_provider/temporary_override.py` with an alias-keyed setter that accepts an absolute Unix
     expiry, alongside the existing relative-duration setter.
   - Refactor both public setters through one private write path so provider/model resolution, validation, sibling
     override preservation, and atomic serialization cannot diverge.
   - Capture `created_at` once, require a finite expiry strictly later than it, and persist the caller's exact
     `expires_at` rather than recomputing it from a duration after UI latency. Preserve the current v2 JSON schema, read
     behavior, expiry boundary, cleanup, and all back-compat wrappers; no state migration is needed.
   - Export and document the new alias-keyed API. Keep this focused extension with the current temporary-override
     subsystem; `sase-core` has no model-override state or resolution surface today, so introducing an isolated Rust
     time setter would split ownership rather than create a reusable backend boundary.

2. **Model relative, absolute, no-expiry, and cancellation outcomes explicitly in the Models-panel flow.**
   - Add small typed result/sentinel values in the Models-panel duration helpers so "open the specific-time input," an
     absolute expiry, `None` (until cleared), a relative duration, and cancellation cannot be confused.
   - Teach the shared duration-choice modal to bind and render the nonnumeric `t` choice while keeping existing callers
     and shortcuts unchanged.
   - Route `t` from `DurationPickerModal` into the new time-entry popup, return to the picker on Back, and hand the
     final typed result to `ModelsPanel`. Relative choices continue using `set_alias_override`; absolute choices use the
     new exact-expiry API.

3. **Build a timezone-aware, presentation-focused absolute-time input.**
   - Add a dedicated Models-panel time helper/modal rather than coupling model overrides to
     `xprompt._directive_time.parse_absolute_time`, whose accepted syntax and `%wait(...)`-specific errors are not an
     appropriate user-facing contract for this popup.
   - Resolve inputs against `sase.core.time.get_timezone()` and an injectable clock, returning a timezone-aware target
     plus its epoch value and display text. Keep parsing pure and deterministic so rollover, explicit dates, past
     targets, configured-timezone divergence, and DST edges are straightforward to test.
   - Update the preview synchronously as text changes; this is CPU-only formatting with no filesystem or subprocess work
     on the Textual event loop.

4. **Make override writes responsive and race-safe at the UI boundary.**
   - Move Models-panel set and clear persistence off the Textual event loop using the established short background
     worker pattern. Capture the alias/model/expiry before dispatch, guard duplicate submissions, and apply
     notifications, `_changed`, and row refreshes on the UI thread only after a successful write.
   - Revalidate the absolute target in the backend so a target that expires between preview and submission fails cleanly
     without marking the panel changed. Preserve the highlighted alias when the completion refresh lands.

5. **Give the new popup a deliberate visual treatment.**
   - Extend `src/sase/ace/tui/styles.tcss` with a centered, double-border popup consistent with the duration picker,
     enough width for full date/time previews, a restrained field label, violet/green valid preview, and red inline
     validation state.
   - Keep layout height fixed across neutral, valid, and error preview states so typing does not make the popup jump.
     Include the configured timezone in the neutral hint before the user types.

6. **Document the complete workflow.**
   - Update the Models-panel section in `docs/ace.md` with the `t` choice, accepted examples, rollover rule, timezone
     preview, and Back behavior.
   - Update `docs/llms.md` with exact-expiry semantics and the new public API while making clear that the JSON schema
     remains unchanged.
   - Refresh module docstrings/copy that currently describe the picker as duration-only.

## Test and Visual Coverage

- Add pure parsing/formatting tests with a pinned clock and configured timezone for 12/24-hour inputs, compact `HHMM`,
  later-today and next-day rollover, `today`/`tomorrow`, ISO dates, invalid calendar values, explicit past targets,
  configured-timezone divergence, nonexistent DST times, and ambiguous DST times with/without an explicit offset.
- Extend temporary-override tests to prove an absolute setter stores the exact epoch, rejects past/non-finite values,
  preserves other aliases, retains atomic v2 serialization, and expires at the existing equality boundary. Keep all
  relative-duration and back-compat tests intact.
- Add modal/pilot tests for the `t` shortcut, live neutral/valid/error previews, Enter acceptance, Esc-to-picker
  behavior, stable `1`-`6`/`c` behavior, exact API dispatch, success/error notifications, and no `_changed` update on
  failure.
- Add ACE PNG snapshots for the enhanced duration chooser and the specific-time popup in neutral, valid, and error
  states. Inspect generated diffs and accept goldens only when the layout, hierarchy, timezone treatment, and color
  states are intentional.
- Run focused model-override, Models-panel, shared-duration-modal, timezone, and visual tests during development. Then
  run `just install`, `just test-visual`, and the required `just check` before handoff.

## Non-Goals

- No natural-language date parser or new third-party date/time dependency.
- No changes to model-resolution precedence, already-running agents, top-bar countdown semantics, or persistent alias
  editing.
- No on-disk schema bump or migration, and no partial migration of the existing model-override subsystem into
  `sase-core`.
- No date-picker calendar widget; the keyboard-first text entry plus exact preview is the focused interaction for this
  change.
