---
create_time: 2026-07-08 21:53:40
status: done
prompt: 202607/prompts/distinct_update_stash_badges.md
tier: tale
---
# Plan: Distinguish update and prompt-stash top-bar badges

## Product context

ACE's top bar can show both:

- **Updates available**: `↑ <count>`, currently styled with the Updates/Admin Center violet accent `#AF87FF`.
- **Stashed prompts**: `❄ <count>`, also currently styled with the same `#AF87FF` background.

The glyphs are different, but both badges are compact count pills in the same indicator cluster. Reusing the same violet
background makes the two states too similar at a glance, especially when the user is scanning the bar rather than
reading each glyph.

The goal is a narrow visual fix: keep the same placement, visibility rules, counts, click/key behavior, and tooltips,
but make one badge color distinct enough that users can identify update-vs-stash state immediately.

## Current architecture

Relevant repo-relative files:

- `src/sase/ace/tui/app.py`
  - Composes the top-bar cluster in this order: task, updates, model indicators, stashed prompts, notifications.
  - No layout change is needed.
- `src/sase/ace/tui/widgets/updates_indicator.py`
  - Defines `_UPDATES_ACCENT = "#AF87FF"`.
  - Renders the badge with `style=f"bold #1a1a1a on {_UPDATES_ACCENT}"`.
  - Opens the Updates panel on click.
- `src/sase/ace/tui/widgets/stashed_prompts_indicator.py`
  - Renders the badge with a hard-coded `style="bold #1a1a1a on #AF87FF"`.
  - Shows/hides based on stash count and exposes stash count/pinned count for prompt hints.
- `src/sase/ace/tui/actions/update_toast.py`
  - Uses the Updates tab accent for update-toast content, with `#AF87FF` as fallback.
- `src/sase/ace/tui/modals/config_center_modal.py`
  - Defines the Updates tab accent as `#AF87FF`.
- `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py`
  - Covers the stashed-prompts top-bar badge PNG snapshot.
- `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`
  - Also renders the stashed-prompts badge in the prompt-stack g-prefix visual snapshot.

Key observation: the update badge's violet is part of a broader Updates tab/update-toast identity, while the
prompt-stash badge's violet is local to one widget plus comments/docs. Therefore the least disruptive change is to
recolor the prompt-stash badge, not the update badge.

## Design decision

Use a green-teal stash accent, recommended value **`#00D7AF`**, and keep black foreground text (`#1a1a1a`).

Rationale:

- It is visually far from the update violet `#AF87FF`.
- It avoids the immediate top-bar neighbors' existing roles:
  - task badge cyan `#48CAE4`;
  - default model override gold `#D7AF5F`;
  - notifications orange/gold `#FF8700` / `#FFD700`;
  - alias override violet `#AF87FF`.
- It has strong contrast with the current dark text. A quick relative-luminance check gives black-on-`#00D7AF` contrast
  above 11:1, so the compact count stays readable.
- The green-teal meaning fits "saved/restorable draft" better than moving update status away from the established
  Updates/Admin Center accent.

This is intentionally only a top-bar stash-badge color change. Prompt-stash modal row colors can stay as-is unless the
badge change exposes a broader stash palette inconsistency later.

## Implementation plan

1. Update `src/sase/ace/tui/widgets/stashed_prompts_indicator.py`.
   - Introduce a local constant such as `_STASH_ACCENT = "#00D7AF"` near the imports.
   - Change `_build_content()` to render `style=f"bold #1a1a1a on {_STASH_ACCENT}"`.
   - Update the class docstring from "violet accent" to "green-teal accent" (or equivalent wording) so the docs match
     the new visual language.

2. Keep `src/sase/ace/tui/widgets/updates_indicator.py` unchanged.
   - Its violet remains the Updates/Admin Center accent.
   - No behavior, tooltip, click handling, count logic, or top-bar order changes.

3. Update visual-test comments that describe the old stash color.
   - `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py` currently says the stash badge uses a "violet
     accent"; revise that wording to the new color family.
   - Check `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py` for any stale color wording if touched by the
     snapshot update.

4. Refresh affected PNG snapshots intentionally.
   - Expected snapshots:
     - `stashed_prompts_indicator_badge_120x40.png`
     - likely `prompt_stack_g_prefix_hints_120x40.png`, because it also sets the stash indicator count before rendering.
   - If the update affects additional snapshots, inspect the diffs and only accept those where the only meaningful
     change is the stash badge background.

## Testing and verification

Because this changes source files in this repo, run the normal setup and final check:

1. `just install`
2. Targeted non-visual checks:
   - `just test tests/ace/tui/test_update_toast.py tests/ace/tui/test_top_bar_order.py`
   - Add a small widget-level assertion for the stash badge style if practical, for example asserting
     `StashedPromptsIndicator._build_content(3)` contains the new background style.
3. Targeted visual update/verification:
   - First run the affected visual tests to see the expected failures:
     `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py`
   - Re-run with `--sase-update-visual-snapshots` only after confirming the diffs are limited to the stash badge color:
     `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_prompt_stash.py tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py --sase-update-visual-snapshots`
   - Re-run the same visual tests without the update flag.
4. Final required repo check: `just check`.

## Risks and mitigations

- **Risk: green-teal reads too close to the task badge cyan.** Mitigation: inspect the visual snapshots with both badges
  present where possible. If the badge still feels too close, switch the stash accent to a warmer but non-notification
  color such as `#87D75F` before accepting snapshots.
- **Risk: broader stash surfaces still use violet.** Mitigation: this plan intentionally scopes only the top-bar
  disambiguation the user reported. Modal row markers can stay stable unless review shows the mixed palette is
  confusing.
- **Risk: visual snapshot churn beyond the intended badge.** Mitigation: do not accept unrelated PNG diffs. Investigate
  any text/layout drift separately.
