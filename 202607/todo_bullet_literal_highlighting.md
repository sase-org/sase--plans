---
tier: tale
title: Extend TODO bullet highlighting and ignore code literals
goal: Highlight every source line belonging to a dash-list item that begins with TODO
  while keeping inline and fenced code examples visually inert and out of the TODO
  count.
create_time: 2026-07-23 10:19:23
status: wip
---

- **PROMPT:** [202607/prompts/todo_bullet_literal_highlighting.md](prompts/todo_bullet_literal_highlighting.md)

# Plan: Extend TODO bullet highlighting and ignore code literals

## Context and outcome

The prompt input widget currently recognizes bounded uppercase `TODO` headers anywhere in a prompt. A colon-terminated
header receives the running-gold marker style and applies the warm italic `todo.body` style only through the remainder
of its physical source line. That leaves a Markdown list item's lazy or indented continuation lines unstyled, as shown
in the supplied screenshot: `- TODO: … have` is styled, but the following `made updates.` source line is not. The same
scanner also treats examples inside backtick code spans and fenced code blocks as real annotations, including them in
the prompt border's `TODO N` pill.

Change the presentation-only TODO overlay so an existing dash-list item whose first content is the exact uppercase
`TODO:` header receives `todo.body` styling from immediately after the colon through the end of that Markdown list item.
This must cover lazy continuation lines, explicitly indented continuations, and later paragraphs that the Markdown
parser assigns to the item, while stopping before a sibling item or content outside the item. Preserve the dash itself
as the structural `bullet.dash` marker rather than recoloring it as TODO body text. Nested list content is part of the
owning TODO item, but nested structural dashes should likewise retain their bullet-marker treatment.

Keep the scope precise: only a dash-list item's first content token being the exact `TODO:` form activates multiline
extension. `TODO(owner):`, checklist prefixes such as `- [ ] TODO:`, and a marker later in an item's prose retain the
existing same-line annotation behavior. Existing bounded forms outside code (`TODO`, `TODO:`, `TODO(owner)`, and
`TODO(owner):`) otherwise remain supported, including the rule that only colon-terminated headers style following prose.
Ordinary quotation marks are not code delimiters and therefore continue to allow TODO matching.

Treat inline backtick code spans and both closed and live unclosed fenced code blocks as literal zones. A `TODO` header
whose start lies in one of those zones must produce neither `todo.header` nor `todo.body` spans and must not contribute
to the whole-stack `TODO N` count. Use the launch parser's established code-literal semantics, including multi-backtick
inline spans and backtick or tilde fences, so prompt execution and prompt presentation agree. Do not alter submitted,
stashed, restored, history-loaded, or editor-returned prompt text.

## Detection and overlay design

Refactor `src/sase/ace/tui/widgets/_todo_highlight.py` around two related inputs:

1. Scan bounded TODO headers only outside ranges returned by the existing
   `sase.xprompt._literal_zones.code_literal_ranges()` helper. Retain the current uppercase/boundary rules, overlay
   byte/line ceilings, and exact-text cache. Preserve the cheap `"TODO"` fast path, avoid running literal-zone work when
   the prompt has no possible code delimiters, and walk the sorted literal ranges and regex matches linearly rather than
   testing every match against every range.
2. Derive dash-list item boundaries from the `PromptTextArea` document's already-maintained incremental Markdown
   tree-sitter tree. Prepare/reuse a small `list_item` query instead of reparsing Markdown or introducing another parser
   on the typing path. For an item whose first content is exactly `TODO:`, convert the captured UTF-8 points/range into
   the character-offset contract used by `_append_highlight_span`, clamp the parser's end position to the real prompt
   length, and extend that annotation's body through the owning item. When an annotation is not such an item starter,
   keep its current end-of-line/next-marker behavior.

Keep the queried list-item boundaries and computed annotations cached by exact prompt text so title reads and repeated
highlight-map builds do not repeat work. Inactive or not-yet-mounted panes still need a correct literal-aware count from
their raw stack text; list-item parsing is unnecessary for counting because multiline extension does not create
additional annotations. If Markdown list querying is unavailable, fail safely to the existing same-line body range
rather than breaking editing or extending a highlight into unrelated prose.

Maintain the established overlay ordering: base Markdown, code/xprompt/Jinja, persistent TODO presentation, then
transient search, yank, selection, and cursor treatments. Ensure `todo.header` remains visually above the extended body,
re-layer `bullet.dash` above `todo.body` where their ranges overlap, and keep search/yank/selection/cursor chrome above
both. The current theme colors and dark/light behavior do not change.

## Tests and visible contract

Expand `tests/ace/tui/widgets/test_prompt_todo_highlight.py` with focused pure-scanner and mounted-widget coverage for:

- inline code using single and longer backtick delimiters, closed backtick/tilde fences, and a live unclosed fence,
  asserting that literal TODO text yields no header/body spans while active markers before or after the literal zones
  still do;
- literal-aware counts, including mixed real/literal markers, and preservation of the fast path and overlay size caps;
- a `- TODO:` item with the screenshot's unindented lazy continuation, indented continuation lines, a blank-separated
  paragraph, UTF-8 text, and a following sibling bullet, asserting every item-content row is styled and the sibling is
  not;
- negative extension cases for `TODO:` later in an item, `TODO(owner):`, checklist-prefixed TODO text, and unrelated
  prose, while confirming their existing eligible same-line styling remains intact;
- nested item content plus overlay precedence, verifying structural dashes and TODO headers remain distinct and search,
  yank, selection, and cursor feedback still win.

Update `tests/ace/tui/widgets/test_prompt_todo_title.py` so initial construction, live editing, prompt stacks, history,
stash/restore, and editor rebuild paths consistently omit inline/fenced literal examples from `TODO N` without losing
real offscreen annotations.

Revise the TODO visual fixture in `tests/ace/tui/visual/_ace_prompt_png_snapshot_helpers.py` to visibly include a
multiline `- TODO:` item matching the reported screenshot, an adjacent ordinary bullet that establishes the stop
boundary, and inline/fenced TODO examples that remain rendered as code rather than annotations. Update the count
assertion in `tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`, regenerate only the affected
dark/light TODO PNG goldens under `tests/ace/tui/visual/snapshots/png/`, and inspect the actual/expected/diff artifacts
to confirm the continuation text gains the existing body style while literal examples and unrelated bullets do not.

Update the prompt TODO documentation in `docs/ace.md`: replace the statement that TODO treatment applies inside Markdown
code regions, document literal exclusion and count behavior, and describe exact `- TODO:` multiline item extension and
its boundary. Keep the presentation-only and overlay-priority guarantees.

## Validation and handoff

Before handoff, run `just install`, the focused TODO/bullet/title widget tests, and the targeted prompt-highlighting PNG
snapshot tests for both themes. Inspect visual diffs before accepting intentional golden changes. Then run the
repository-required `just check`, which also exercises the full visual snapshot suite. Confirm `git diff --check` is
clean and that no prompt payload, parser expansion, or stack persistence behavior changed.
