---
tier: tale
title: Capture clipboard bullet lists as task sub-bullets
goal:
  Recognize flat unordered Markdown lists copied to the clipboard and render each item
  as an Obsidian child bullet beneath the captured task without weakening
  structured-text safeguards.
create_time: 2026-09-09 19:53:04
status: wip
---

# Plan: Capture clipboard bullet lists as task sub-bullets

## Context and outcome

`bob capture` already classifies clipboard values in `src/native/capture_clip.rs` and
supplies rendered child lines to the task, ordinary-bullet, and Pomodoro capture paths
in `src/native/capture.rs`. Today, any multiline value containing `- `, `* `, or `+ `
list syntax is treated as Markdown-structured text and saved verbatim to a snippet file.
A clipboard containing only the three top-level bullets from the request therefore
produces a snippet link instead of three direct children.

Change that classification so a complete, flat unordered Markdown list is translated
into child bullets. With `BOB_NOW=2026-07-17`, a clipboard containing the example list,
and `bob capture 'foo bar baz @foo %'`, `foo.md` should receive:

```markdown
- [ ] #task foo bar baz [created::2026-07-17]
  - Use `@` symbol instead of `#` for tribe prefix.
  - Support expansion of families within clan.
  - Family members must be launched sequentially.
```

## Clipboard classification and rendering

- Add a focused parser/classifier for a clipboard value whose every logical line is a
  non-empty, top-level unordered list item. Recognize the standard `-`, `*`, and `+`
  markers when followed by Markdown whitespace, remove only the marker and its
  separating whitespace, and preserve each item's remaining inline Markdown and text.
  Handle a one-item list as well as lists up to the existing ten-line inline limit so a
  source marker is never doubled in the rendered child.
- Run this pure-list recognition before the generic single-line/structured-Markdown
  fallbacks, but keep attachment detection and the existing per-value validation
  guarantees intact. Reuse the existing `ClipMode::Lines` result and child-line renderer
  rather than introducing a new public JSON mode: headerless captures become direct
  sibling children, while an explicit clipboard header continues to own the items using
  the established header/nesting rules. Counted history captures should continue to
  classify each entry independently and flatten the resulting rendered lines in source
  order.
- Keep snippet preservation for values that are not entirely a safe flat list: ordered
  lists, nested or indented items, wrapped continuation lines, mixed bullets and prose,
  blank or empty items, block quotes/headings/fences, and values above the line limit.
  This prevents partial marker stripping or loss of Markdown structure. Checkbox text
  within an otherwise valid unordered item can remain part of the item body, allowing
  the existing renderer to preserve it as a nested checkbox rather than special-casing
  task syntax.

## User-facing contract

- Update the `bob capture --help` clipboard-shape description to distinguish recognized
  flat unordered lists from other Markdown-structured content while retaining the
  documented snippet fallback.
- Update the README clipboard documentation and examples to show that copied flat bullet
  lists are normalized into task sub-bullets, that source bullet markers are removed,
  and that unsupported/mixed/nested Markdown still becomes a snippet.
- Keep `task_line` as the parent line and retain the existing `clip` JSON shape. A
  recognized list reports `mode: "lines"`, with `lines` containing the exact rendered
  child bullets; no new flags, subcommands, options, files, or schema fields are
  required.

## Verification

- Extend `capture_clip` unit tests around list recognition and rendering: the requested
  dash-list shape, `*` and `+` markers, a single bullet, explicit-header nesting,
  preserved inline Markdown/checkbox bodies, and the existing ten-line boundary. Add
  negative cases for empty items, mixed prose, ordered/nested/wrapped lists, blank
  lines, and over-limit lists to prove they still select snippet mode.
- Add or extend CLI integration coverage using a deterministic `BOB_CLIPBOARD_CMD` and
  `BOB_NOW=2026-07-17`. Assert the request's routed `foo.md` result exactly, plus the
  stable JSON `mode` and rendered `lines`; exercise at least one header or
  counted-history composition path if it is not already fully established by unit
  coverage.
- Re-run the focused capture unit and CLI integration tests during development, then run
  the repository's full quality gate with `just all` (`cargo fmt --check`, Clippy for
  all targets/features, and the complete test suite).

## Risks and boundaries

The main risk is misclassifying arbitrary Markdown and silently discarding meaningful
structure. Requiring every line to match the flat-list grammar, retaining existing size
limits, and falling back atomically to a verbatim snippet for ambiguous input keeps the
change conservative. Reusing `ClipMode::Lines` avoids a compatibility break for JSON
consumers, while exact rendered-line tests protect indentation needed for Obsidian task
ownership and later capture insertion.
