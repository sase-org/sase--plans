---
tier: tale
title: Make TODO markers legible and colon-gate note highlighting
goal: 'TODO markers render as unmistakably dark text on running gold, while only colon-terminated
  TODO annotations style the prose that follows them.

  '
create_time: 2026-07-22 09:42:44
status: wip
---

- **PROMPT:** [202607/prompts/todo_colon_highlight_legibility.md](prompts/todo_colon_highlight_legibility.md)

# Plan: Make TODO markers legible and colon-gate note highlighting

## Context and outcome

The prompt editor recognizes bounded uppercase `TODO` markers and overlays a gold header plus a warm italic body style.
The supplied screenshot exposes two readability problems: the visible marker foreground is too light against the gold,
and a bare quoted `TODO` causes unrelated prose through the next marker or line ending to inherit the body-note style.
The palette helper currently intends black-on-`#FFD700`, but its coverage mostly asserts registered theme values rather
than the final rendered cells. This change should make the actual inline marker and prompt-border count capsule reliably
dark and should distinguish marker recognition from body-note activation.

Keep the work presentation-only in the ACE TUI. Prompt contents, marker counting, prompt-stack aggregation, history and
stash restoration, editor round-trips, submission, and agent lifecycle behavior remain unchanged. The render path must
continue using the existing cached, bounded scan without adding I/O, refresh work, or data-scaled processing.

## Syntax and color contract

- Continue recognizing uppercase `TODO` at the existing identifier boundaries, with the optional owner form. Lowercase
  `todo` and embedded or suffixed identifiers such as `preTODO`, `TODOS`, and `TODO2` remain ordinary text.
- Continue rendering and counting the marker itself for all four supported forms: `TODO`, `TODO:`, `TODO(owner)`, and
  `TODO(owner):`.
- Give every marker header the fixed Agents-tab running gold background (`#FFD700`) with an explicit black (`#000000`)
  foreground. Apply the same black-on-gold contract to the whole-stack `TODO N` border capsule.
- A marker activates the warm, italic body-note style only when its complete header ends in `:`. Therefore bare `TODO`
  and `TODO(owner)` stop at the marker; punctuation, closing quotes, and prose after them retain their underlying prompt
  syntax. `TODO:` and `TODO(owner):` continue styling from the colon to the next recognized marker on that line or the
  line ending.
- Preserve overlay precedence: search-current, yank feedback, selection, and cursor treatments remain visually above
  TODO styling, while TODO headers remain above ordinary Markdown, code-span, Jinja, xprompt, and alternate syntax.

## Span construction and rendering

Adjust `src/sase/ace/tui/widgets/_todo_highlight.py` so each recognized header still produces one annotation/count, but
only a colon-terminated match receives a non-empty body span. Preserve the current next-marker and line-end boundaries
for colon-terminated bodies, including mixed lines such as `TODO bare; TODO: styled note`. Keep the cached annotation
value object and UTF-8 byte-column conversion coherent with this contract; prefer representing a bare marker with an
empty body range so existing count and overlay plumbing do not split into separate detection paths.

Make the marker foreground an explicit, stable part of the shared TODO palette rather than relying on theme blending or
an indirect contrast result. Ensure theme registration, re-registration after dark/light theme changes, inline render
composition, and the `PromptInputBar` count capsule all consume that same contract. Do not weaken the theme-aware warm
body color for colon-terminated annotations.

## Documentation and regression coverage

Update `docs/ace.md` to describe the difference between a bare marker and a colon-terminated annotation, including the
optional-owner forms, and to state that marker/count text is black on running gold while only body-note color follows
the active theme.

Strengthen `tests/ace/tui/widgets/test_prompt_todo_highlight.py` and `tests/ace/tui/widgets/test_prompt_todo_title.py`
around observable behavior:

- cover bare, colon-terminated, owner, punctuation/quoted, multiple-marker, and mixed-marker lines;
- assert bare markers have empty body spans while colon-terminated bodies retain the next-marker/line-end boundary;
- inspect the generated highlight map so prose after a bare marker is not assigned `todo.body`;
- inspect final rendered cell/segment styles in dark and light themes to prove marker glyphs are `#000000` on `#FFD700`,
  not merely that the intermediate theme registry contains those values;
- retain count behavior for every recognized marker and assert the prompt-border capsule uses the same black-on-gold
  palette through theme changes; and
- retain the existing precedence checks for selection, search, yank feedback, and cursor rendering.

Extend the focused prompt-highlighting visual fixture with a screenshot-like quoted bare `TODO` followed by ordinary
prose alongside a colon-terminated annotation. Update only the affected PNG golden snapshots, inspect the actual and
diff images, and confirm that the bare suffix keeps its normal syntax while both header forms and the count pill are
visibly dark-on-gold. Keep dark/light snapshot coverage for marker legibility and the inactive prompt-stack count.

## Validation

Before repository checks, run `just install` as required for an ephemeral workspace. Then run the focused TODO widget
tests and focused PNG snapshot cases, accepting only intentional TODO golden changes. Run the complete
`just test-visual` suite to catch cross-surface rendering regressions, followed by the mandatory `just check` so final
formatting, linting, typing, SASE/plan validation, and the full test suite cover the exact final tree.
