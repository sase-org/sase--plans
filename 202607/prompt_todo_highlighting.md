---
tier: tale
title: Beautiful TODO highlighting for prompt drafts
goal: 'Uppercase TODO annotations in ACE prompt inputs are unmistakable on screen,
  remain discoverable when restored outside the viewport or into a prompt stack, and
  update instantly without compromising prompt editing or TUI responsiveness.

  '
create_time: 2026-07-22 07:20:49
status: wip
---

- **PROMPT:** [202607/prompts/prompt_todo_highlighting.md](prompts/prompt_todo_highlighting.md)

# Plan: Beautiful TODO highlighting for prompt drafts

## Context and product outcome

Long prompts are often stashed and restored across writing sessions. ACE currently restores their text faithfully, but
there is no visual language for unfinished notes; because a restored prompt is focused at its end, an earlier reminder
can also be outside the viewport. Add a prompt-native TODO annotation treatment that makes visible markers beautiful and
obvious while a small title chip reports TODOs anywhere in the loaded prompt stack.

This is presentation-only behavior in `PromptTextArea` and `PromptInputBar`. Stash serialization, restoration ordering,
prompt text, submission payloads, and external-editor files must remain byte-for-byte unchanged. Consequently, no Rust
core or prompt-stash format change is needed: every path that constructs or rebuilds a `PromptTextArea`—initial prompt,
history load, external-editor return, and mounted or fresh-bar stash restore—should acquire the behavior automatically.

## Annotation contract

- Treat exact uppercase `TODO` as the intentional marker. Match it only at identifier boundaries so prose/code such as
  `TODOS`, `TODO2`, and `preTODO` is not annotated; lowercase `todo` remains ordinary text and avoids noisy accidental
  matches.
- Support the forms people naturally leave in drafts without imposing a new syntax: `TODO`, `TODO: ...`,
  `TODO(owner): ...`, Markdown bullets/checklists containing those forms, and multiple markers across lines or panes.
- Highlight standalone markers consistently wherever they occur, including Markdown literal/code regions. The contract
  stays easy to predict—uppercase `TODO` always means an annotation—and avoids a stash-specific or parser-context rule.
- Count matched annotations, not merely affected lines. Removing or changing the last marker must remove all TODO chrome
  immediately; no stored metadata can become stale.

## Visual design and interaction

- Give the marker header (`TODO`, plus an immediately attached owner and separator when present) a high-contrast, bold
  warning treatment that reads like a compact annotation badge. Give the rest of that TODO line a quieter warm treatment
  so the task is scannable without drowning out the surrounding Markdown. Derive colors from the active ACE theme and
  verify contrast in both Textual dark and light themes instead of hard-coding a dark-theme palette.
- Add a concise `TODO N` capsule to the prompt bar's border title whenever any pane contains annotations. Aggregate the
  whole stack rather than only the active pane, so an off-screen TODO or one in a compact inactive restored pane is
  still immediately visible. Keep the existing prompt/agent/binding/mode/Jinja title information intact and omit the
  capsule entirely at zero.
- Establish deliberate overlay precedence: ordinary Markdown, Jinja, placeholder, code, and xprompt syntax remain the
  base; TODO annotation styling sits above that base; active search matches, yank feedback, selections, and the cursor
  remain unambiguous above or through TODO styling. In particular, do not let a persistent TODO background conceal a
  selection or active search result.
- Preserve the current restore focus and scroll behavior. The title capsule provides awareness for off-screen work;
  loading a stash must not unexpectedly jump the cursor away from the draft's end or steal focus to the first TODO.

## Implementation shape

1. Add a focused TODO annotation helper/mixin alongside the existing prompt highlight overlays. Keep marker detection as
   small pure logic that returns character spans for the badge/header, line body, and aggregate count; convert through
   the established UTF-8-safe span path so non-ASCII text before a TODO cannot shift the rendered highlight.
2. Integrate the overlay into `PromptTextArea`'s cooperative theme-registration and highlight-map chain at the explicit
   precedence described above. Reuse the shared overlay byte/line ceilings, add a cheap `"TODO"` fast path, and cache
   scan results by document text so repeated paints, title refreshes, and cursor movement do not rescan an unchanged
   prompt. The keystroke/render path must remain synchronous, deterministic, disk-free, and free of new timers or
   background work.
3. Expose the cached annotation count to `PromptInputBar` and compose a theme-aware total-stack title capsule. Refresh
   it through the existing text-change, pane focus/rebuild, add/remove/reorder, history/editor load, and stash-restore
   title lifecycle; aggregate mounted pane caches rather than repeatedly scanning every pane on each keypress. Ensure
   initial construction and the brief pre-mount/rebuild window have a correct state without depending on filesystem
   data.
4. Document the marker contract, title count, theme behavior, and stash-restore experience in the ACE prompt-input
   documentation near prompt stashes and syntax highlighting. Make clear that the annotation is visual only and the
   literal prompt is what gets stashed and submitted.

## Verification and acceptance

- Add pure/unit coverage for boundary rules, owner/colon forms, multiple markers, Unicode-prefix byte offsets, cache
  invalidation/reuse, empty and oversized prompts, and the no-marker fast path.
- Add mounted widget coverage that asserts TODO overlay styles register and update across theme changes; editing a
  marker in or out updates both spans and count; search, selection, cursor, and yank states remain legible; and existing
  Markdown/Jinja/xprompt/code highlighting is not discarded.
- Add prompt-bar integration coverage for initial text, an off-screen marker, multi-pane aggregation, pane removal and
  reorder, and both stash restore paths (append into an existing bar and mount a fresh restored bar). Assert the title
  retains agent, mode, binding/dirty, and Jinja adornments while adding/removing the TODO capsule.
- Add deterministic PNG snapshots showing a realistic restored long draft with visible TODO forms and the total-count
  capsule in dark and light themes, plus a stacked case where at least one counted TODO is in an inactive pane. Review
  actual/expected/diff artifacts before accepting any intentional golden updates so the final treatment is balanced,
  readable, and visually dominant without looking like an error state.
- Run focused TODO/prompt widget tests while iterating, then `just test-visual` for the dedicated PNG suite and the
  repository-mandated `just check` after `just install`. The feature is complete when restored TODOs appear immediately,
  counts stay correct through live editing and stack operations, prompt text remains untouched, both themes pass visual
  review, and the guarded/cached overlay introduces no I/O or asynchronous work on the typing/render path.

## Non-goals and future room

Do not add TODO persistence metadata, automatic navigation, submission blocking, configuration, or related keywords such
as `FIXME`/`NOTE` in this change. The focused `TODO` contract establishes a coherent visual system first; navigation or
a configurable annotation vocabulary can build on the pure detector later if real usage calls for them.
