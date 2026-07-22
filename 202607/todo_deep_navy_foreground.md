---
tier: tale
title: Use a terminal-safe deep navy foreground for TODO markers
goal: Keep TODO headers and count capsules readable on running gold without relying
  on the terminal's ANSI black palette entry.
create_time: 2026-07-22 11:01:30
status: wip
---

- **PROMPT:** [202607/prompts/todo_deep_navy_foreground.md](prompts/todo_deep_navy_foreground.md)

# Use a terminal-safe deep navy foreground for TODO markers

## Problem

ACE currently renders TODO marker headers in the prompt editor and the matching `TODO N` border-title capsule as exact
black (`#000000`) on the shared running gold (`#FFD700`). Although the nominal colors have excellent contrast, Rich can
degrade exact black to the terminal's ANSI black palette entry. A customized terminal palette may display that entry as
a much lighter color, such as salmon, which makes the marker difficult to read on gold.

Replace exact black with deep navy `#00005F`. It remains visually distinct from the gold surface, has a WCAG contrast
ratio of approximately 12.85:1 against `#FFD700` (above the 7:1 AAA threshold for normal text), maps away from ANSI
black when Rich degrades output to 256 or standard colors, and remains suitable for both dark and light ACE themes
because the chip background is fixed.

## Scope and constraints

- Apply the same foreground to inline TODO headers and the whole-stack `TODO N` count capsule through the existing
  shared `todo_theme_colors()` path.
- Keep `RUNNING_COLOR` (`#FFD700`) as the background and preserve the current theme-aware colon-terminated body-note
  color.
- Do not change TODO parsing/counting, prompt contents, marker boldness, theme switching, or the precedence of cursor,
  selection, search, and yank styles.
- Update only the documentation and PNG goldens whose stated or rendered TODO foreground changes.

## Implementation

1. In `src/sase/ace/tui/widgets/_todo_highlight.py`, change the shared TODO marker foreground constant from exact black
   to `#00005F`. Keep the color centralized so `_register_todo_text_area_theme()` and
   `PromptInputBar._todo_chip_markup()` continue to receive identical colors from `todo_theme_colors()`.

2. In `tests/ace/tui/widgets/test_prompt_todo_highlight.py`, update the palette, theme-switch, and final rendered-cell
   expectations to deep navy, and rename the black-specific render test accordingly. Add an explicit relative-
   luminance/contrast assertion for the marker foreground against `RUNNING_COLOR`, requiring at least 7:1, so future
   palette edits cannot retain the right wiring while silently becoming unreadable. Continue checking every rendered
   cell of `TODO(owner):` in both built-in themes to prove the final overlay style, not just the helper return value.

3. In `tests/ace/tui/widgets/test_prompt_todo_title.py`, update the rendered border-title capsule expectation to
   `#00005F` while retaining the assertions that the running-gold background, bold weight, count, and theme-independent
   style remain unchanged. This verifies that the title and editor paths stay in sync through the shared palette helper.

4. Update the TODO-marker description in `docs/ace.md` to call out the exact deep navy-on-running-gold treatment and its
   terminal-palette motivation. Leave the unrelated Agents-tab unread-count black-on-gold documentation unchanged.

5. Regenerate the affected TODO PNG snapshots under `tests/ace/tui/visual/snapshots/png/` for restored dark, restored
   light, and stacked prompts. Inspect the resulting images and visual diff artifacts to confirm that headers and count
   capsules are clearly legible, use the same navy on gold, and do not alter body notes or unrelated prompt chrome.

## Validation

1. Run `just install` before project checks, as required for an ephemeral SASE workspace.
2. Run the focused TODO widget tests:
   `pytest tests/ace/tui/widgets/test_prompt_todo_highlight.py tests/ace/tui/widgets/test_prompt_todo_title.py`.
3. Regenerate intentional PNG changes with `just update-visual-snapshots`, then run `just test-visual` without the
   update flag and confirm it passes cleanly.
4. Review `git diff --check`, the source/test/doc diff, and the three changed PNGs to ensure the scope is limited to the
   planned foreground adjustment.
5. Run the mandatory full `just check` suite and resolve any failures before handing the implementation back.

## Acceptance criteria

- Inline TODO headers and the `TODO N` capsule render as bold `#00005F` text on `#FFD700` in both dark and light themes.
- The foreground/background contrast assertion is at least 7:1; the selected pair measures approximately 12.85:1.
- Exact ANSI black is no longer used for TODO markers, avoiding the user's customized black palette entry.
- TODO body-note styling and all higher-priority interaction highlights retain their existing behavior.
- Focused widget tests, a clean non-updating visual snapshot run, and `just check` all pass.
