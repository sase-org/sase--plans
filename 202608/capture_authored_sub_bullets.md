---
tier: epic
status: done
title: First-class authored sub-bullets in Bob capture
goal:
  Bob capture accepts a one-line parent followed by flat authored bullets, applies
  capture-wide markers from the end of any input line, writes the children with native
  Obsidian indentation, and gives the macOS capture app polished bullet editing and
  exact hierarchical preview behavior.
phases:
  - id: line-aware-capture
    title: Line-aware capture grammar and Markdown output
    depends_on: []
    size: medium
    description:
      "line-aware-capture: make bob-cli own a line-preserving capture model,
      line-terminal marker extraction, authored-child rendering, compatible JSON and
      stdin contracts, documentation, and exhaustive Rust coverage across every capture
      mode."
  - id: mac-bullet-editor
    title: Native bullet editing and hierarchical preview
    depends_on:
      - line-aware-capture
    size: medium
    description:
      "mac-bullet-editor: consume bob-cli's authored-child contract in bob-mac-capture,
      add Ctrl-J bullet insertion and empty-bullet Backspace behavior through the native
      text system, render exact nested preview lines, and verify the complete macOS
      interaction."
proposed_by: bbugyi200.athena.00w.f0.f0.w0
create_time: 2026-09-09 19:59:48
---

- **PROMPT:**
  [prompts/202608/capture_authored_sub_bullets.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/capture_authored_sub_bullets.md)

# First-class authored sub-bullets in Bob capture

## Outcome

A capture draft has one parent line followed by zero or more flat Markdown bullets:

```text
Prepare the launch review
- Confirm the rollout owner
- Attach the final checklist @work p:1
```

The trailing `@work` and `p:1` configure the whole capture even though they appear on a
child line. They are removed from that line, and the target note receives one coherent
Obsidian block (using the target note's normal child indentation):

```markdown
- [ ] #task Prepare the launch review [created::<date>] [priority::<value>]
      [scheduled::<date>]
  - Confirm the rollout owner
  - Attach the final checklist
  - 🗓️ **SCHEDULE LOG**
    - _<date>_ — <roll reason>
```

The same child syntax works when the parent is an ordinary note bullet, a
Pomodoro-linked task, or itself a captured sub-bullet beneath an existing task. The
macOS app makes the common path delightful: Ctrl-J starts the next `- ` row, Backspace
removes an unused `- ` row in one action, syntax completion works on the active line,
and preview shows the exact parent/child hierarchy that Bob will write.

This is an epic because the authoritative grammar and vault mutation belong in
`bob-cli`, while native editing and presentation belong in the linked `bob-mac-capture`
repository. The macOS phase deliberately depends on the CLI phase so it consumes a
settled wire contract instead of independently interpreting capture syntax.

## Repositories and ownership

- Phase `line-aware-capture` works in the primary `bob-cli` checkout. The central files
  are `src/native/capture_language.rs`, `src/native/capture.rs`,
  `src/native/capture_parse.rs`, `src/native/capture_complete.rs`, `tests/cli.rs`, and
  `README.md`.
- Phase `mac-bullet-editor` works in the linked `bob-mac-capture` repository. Before
  reading or editing it, use the `/sase_repo` skill:

  ```sh
  sase repo open bob-mac-capture -r "Add native authored-sub-bullet capture support"
  ```

  Use only the path printed by that command. The main touch points are
  `CaptureModels.swift`, `CapturePanelModel.swift`, `CaptureKeyCommandRouter.swift`,
  `CapturePanelController.swift`, `CapturePanelView.swift`, their existing test suites,
  the fake-bob fixture, and `README.md`.

- Swift never parses capture grammar or writes the vault. `bob capture-parse`,
  `bob capture-complete`, and `bob capture --dry-run/--format json` remain the only
  sources for syntax, completion, preview, and rendered Markdown.

## Product and grammar contract

### Physical lines and authored bullets

1. Normalize CRLF and bare CR only for semantic line splitting; keep original UTF-8 byte
   offsets for editor spans and completion ranges.
2. The first physical line is the parent. After capture-wide markers are removed, it
   must contain nonempty text and is normalized internally exactly as today's
   single-line body is normalized.
3. Every later nonblank line must be one flat unordered Markdown item beginning at
   column zero with `-`, `*`, or `+`, followed by at least one space or tab. Strip that
   source marker and separating whitespace, normalize the item body, and render it as
   `- <body>`. Preserve inline Markdown and checkbox text. The mac app emits canonical
   `- `, while the other two markers make pasted/CLI input unsurprising and match Bob's
   existing flat clipboard-list vocabulary.
4. Blank continuation lines and marker-only placeholder rows such as `- ` produce no
   child. This keeps the transient state created by Ctrl-J previewable and harmless.
   Reject any nonblank continuation prose, indented/nested item, wrapped item, or item
   that becomes empty only after capture markers are removed. Report the physical line
   and a stable ranged diagnostic rather than silently folding or dropping user text.
5. This feature creates one flat authored-child level only. It does not infer nested
   input hierarchy. A capture whose overall kind is already `sub_bullet` nests the new
   captured line under the selected existing task and nests authored children one
   additional level beneath that captured line.

### Markers on any line

Treat supported markers as capture-wide directives even when their terminal occurrence
is on an authored-bullet line:

- `s:<N>`, `p:<N>`, `%`, `%N`, and `%header` may occur in the contiguous terminal marker
  region of any physical line.
- A trailing `@route`, `@route#section`, `@route:block-id`, or `@route^block-id` may
  occur at the end of any line and composes with the other line-terminal markers in the
  same orders accepted today. Preserve the established leading-route form on the first
  line.
- Markers in the middle of a line remain literal. `--route`, `--section`, `--task`,
  `--task-ref`, `--clip`, and `--no-clip` retain their current override/literal
  semantics across all lines.
- Resolve all recognized occurrences into one capture. A second occurrence of the same
  global kind (route/mode, schedule, priority, or clipboard), including a conflicting
  occurrence on another line, is ambiguous and must fail with a precise diagnostic and
  no write. Do not silently choose a line or leak an apparently consumed directive into
  child text.
- A directive removed from a child line affects the parent capture; it never schedules,
  prioritizes, routes, or attaches clipboard content to just that child. State this in
  CLI and app documentation.

Build this once as a line-aware model in `capture_language.rs`. Execution parsing,
editor parsing, and completion must share line/token classification and marker
selection; do not add a multiline parser beside the existing authoritative grammar.
Spans remain ordered, non-overlapping, half-open UTF-8 byte ranges into the raw input.
Add a bullet-marker span kind if useful for native editor styling, but keep unknown
additive kinds safe for older clients.

### Rendering, ordering, and compatibility

- Reuse the existing target-aware indentation policy: prefer the target note's dominant
  tab-or-two-space child unit and fall back to the tab Obsidian uses by default. Render
  each authored item exactly one unit beneath the newly captured parent. Preserve the
  target document's line endings.
- Assemble a capture block in this stable order: parent line, authored children,
  clipboard children/attachments, then the priority-roll schedule log. Existing
  sub-bullet planning must prefix the complete inner block so grandchildren retain the
  correct relative depth.
- Keep capture JSON backward compatible: `text` and `task_line` remain the normalized
  parent body and rendered parent line. Add an optional/omitted-when-empty `sub_bullets`
  array containing the exact rendered authored-child lines, including their
  target-selected indentation. Human output prints those lines directly beneath
  `task_line`, before clipboard and schedule-log lines.
- Add `sub_bullets` to `capture-parse` as normalized child bodies (without source list
  markers) so the editor can understand the semantic draft without re-parsing it. This
  is additive to schema version 1; document the distinction between semantic parse
  bodies and exact rendered capture-result lines.
- Change piped-stdin handling in `capture`, `capture-parse`, and `capture-complete` from
  one `read_line` call to reading the complete UTF-8 stream. Embedded newlines in a
  single argv value remain intact; multiple argv values are still joined with spaces.
  Update help/error text and README examples for multiline stdin and shell quoting.

## Phase `line-aware-capture`: CLI grammar and output

### Implementation

1. Refactor `capture_language.rs` around raw physical lines with byte origins. Represent
   the normalized parent body, normalized authored child bodies, global marker values,
   capture kind/route, marker spans, and diagnostics in one shared parse result. Adapt
   the existing terminal-marker walker rather than copying schedule, priority,
   clipboard, route, Pomodoro, or selected-task classifiers.
2. Make the editor parser expose all recognized directive spans on all lines and give
   malformed continuation lines, empty-after-marker items, and duplicate global
   directives stable diagnostic codes/ranges. Update cursor-aware completion to select a
   valid leading/trailing marker on the physical line containing the cursor, so
   `- context @ca|` completes routes without treating `@ca` as middle-of-draft prose.
   Preserve leading-marker precedence and current incomplete `@`, `@#`, `@:`, and `@^`
   picker states.
3. Extend `ParsedCaptureText` with authored child bodies. In `capture.rs`, determine the
   child indent whenever authored children, clipboard content, or a schedule log need
   it; render authored children; and pass all child sources through a single block
   assembler in the required order. Ensure fresh targets, existing tab/two-space notes,
   CRLF notes, forced modes, Pomodoro coordinated writes, and existing-task sub-bullet
   writes all use the same path.
4. Extend human and JSON output without changing the meaning of existing fields. Keep
   dry-run exact and side-effect free. A parse/validation error, duplicate marker,
   invalid priority, clipboard failure, or coordinated Pomodoro/sub-bullet preflight
   failure must leave every note and planned attachment untouched.
5. Read complete stdin in all three capture commands and update command help,
   `README.md`, JSON contract documentation, examples, and the retired one-line wording.
   Explain accepted flat bullet syntax, capture-wide marker scope, indentation,
   ordering, empty placeholders, invalid multiline shapes, and behavior for each capture
   kind.

### Tests

Add focused unit and CLI integration coverage for:

- LF, CRLF, and bare-CR drafts; Unicode before markers; exact cross-line UTF-8 spans;
  complete stdin reads; and argument values containing embedded newlines.
- `-`, `*`, and `+` children, inline Markdown and checkbox preservation, blank and `- `
  placeholders, first-line missing text, nonbullet continuation prose, nested/indented
  or wrapped items, and items emptied by marker extraction.
- Every terminal marker kind at the end of the parent and each child line, valid mixed
  marker order, middle-of-line literals, forced-route/section/task behavior, invalid
  marker diagnostics, duplicate/conflicting global directives, and parse/execution
  equivalence.
- Completion on parent and child lines at real UTF-8 cursor offsets, including a marker
  on an earlier child rather than only at end of draft.
- Exact Markdown and JSON for task, ordinary note bullet, Pomodoro task, and selected
  existing-task sub-bullet captures; fresh/tab/two-space indentation; line endings;
  authored-child/clipboard/schedule-log ordering; dry run; and all-or-nothing failure.
- Backward compatibility for ordinary single-line captures and omission of `sub_bullets`
  when empty.

Run from `bob-cli`:

```sh
cargo test capture_language
cargo test --test cli capture
just all
just install-smoke
git diff --check
```

## Phase `mac-bullet-editor`: native interaction and preview

### Model and process contract

1. Extend `CaptureParseResponse` and `CaptureCommandSuccess` in `CaptureCore` to decode
   the additive `sub_bullets` fields. Capture-result decoding must tolerate the field
   being absent so the app still produces a useful parent-only preview during benign
   executable version skew.
2. Preserve the draft as one argv element for parse, completion, preview, and submit.
   Ensure the model supplies the actual caret's UTF-8 byte offset, not an assumed
   end-of-draft offset, when editing or completing a marker on any line. Continue to
   validate every server-provided byte range before highlighting or replacement.
3. Style an additive bullet-marker span subtly if the CLI provides it. All route,
   section, Pomodoro, selected-task, schedule, priority, and clipboard highlighting and
   completion still come from Bob's spans/needs; Swift must not classify line-terminal
   markers itself.

### Native editing behavior

1. Split the existing generic newline key command so Ctrl-J is a dedicated
   `insertBulletNewline` action while Shift-Return and Option-Return retain ordinary
   newline insertion. Through the editable `NSTextView` first responder and native text
   system, Ctrl-J replaces the current selection with `\n- `, dismisses completion,
   lands the caret after the space, participates in undo, and works in the middle or at
   the end of a draft without directly rewriting `CapturePanelModel`.
2. Add a Backspace command path that intervenes only for an editable text view with a
   collapsed selection on a physical line whose complete content is exactly `- `. Delete
   that row and its preceding newline as one native edit (or just the row if it is the
   first line), place the caret at the prior line boundary, and dismiss stale
   completion. All other selections, modifiers, lines, and delete behavior pass through
   untouched to AppKit.
3. Keep IME/marked-text and accessibility behavior native: use text-view change hooks
   and replacement APIs, not synthetic key events or string surgery in the SwiftUI
   model. Preserve the editor's autosizing/six-visible-line scroll behavior and the
   panel's persistent footer.

### Exact, beautiful preview

Render a compact monospaced Markdown block containing `task_line` followed by the exact
`sub_bullets` lines in source order and indentation. Do not flatten the array into prose
or cap it at the current two-line parent preview. Let the existing auxiliary overflow
region scroll when the panel's screen-height clamp binds, while destination, placement,
kind, schedule, and relative-target metadata remain concise and visually secondary.

When available, keep clipboard and schedule-log child lines in their established order
after authored children so explicit Preview mirrors the full block. Continuous
`--no-clip` preview must retain its existing no-clipboard guarantee and literal-marker
notice. Preview errors never mutate or discard the draft.

### Tests and documentation

- Extend process-client/fake-bob tests with multiline argv recording and parse,
  completion, live-preview, explicit-preview, and submit responses containing
  `sub_bullets`.
- Add decoding tests for present, empty, absent, and unknown additive JSON fields.
- Extend key-router/controller tests for Ctrl-J versus Shift/Option-Return; selection
  replacement; caret placement; undo-capable text-view changes; Backspace at final and
  middle `- ` rows; first-line handling; and pass-through for nonempty bullets,
  selections, modifiers, noneditable views, and unrelated responders.
- Add model/view-focused tests for actual cursor offsets on earlier lines and for the
  ordered preview line model. Avoid screenshot-pixel assertions; manually evaluate the
  rendered hierarchy, spacing, scroll/clamp behavior, light/dark appearance, Dynamic
  Type, and VoiceOver reading order.
- Update the keyboard table and runtime/live-preview documentation. Remove the obsolete
  statement that multiline editing is only whitespace-normalized affordance and show a
  complete parent-plus-children example with line-terminal markers.

Run the repository's full checks on a macOS 26 host:

```sh
just format-lint
just build
just test
just bundle
```

The current implementation workspace is Linux and cannot compile the AppKit/SwiftUI
target; do not report these macOS checks as passed unless they run on macOS. Require the
repository's macOS CI to pass. With an installed build and the matching new `bob`
executable, manually verify:

1. Ctrl-J after the parent and after a middle child creates `- ` and leaves the caret in
   the right place; Backspace on that unused row removes it cleanly; ordinary Backspace
   and Shift/Option-Return remain native.
2. A marker typed at the end of any child highlights and completes at the real caret,
   changes the global destination/properties in preview, and disappears from the
   rendered child.
3. Preview visually nests several authored bullets (including Markdown and checkbox
   text), stays readable in light/dark mode, and scrolls without hiding the editor or
   Capture button at the screen-height limit.
4. Return and Command-Return write the exact previewed hierarchy for task, note-bullet,
   Pomodoro, and selected-existing-task destinations; failures retain the full multiline
   draft.
5. Authored children plus clipboard content and a rolled priority produce the documented
   parent → authored children → clipboard children → schedule-log order with the note's
   indentation and line endings.

## Scope constraints

- Do not add nested authored-list syntax, continuation paragraphs, ordered lists,
  per-child routing/properties, or batch creation of multiple parents.
- Do not move grammar, Markdown formatting, target discovery, preview truth, clipboard
  access, random scheduling, or vault mutation into Swift.
- Do not change existing clipboard-list classification limits or attachment/snippet
  behavior while extracting shared helpers.
- Do not introduce a new CLI flag or a breaking rename/removal of existing JSON fields.
- Do not make live preview read the clipboard, and do not weaken coordinated-write or
  dry-run guarantees.
- Do not edit a deployed copy of `bob-mac-capture`; work only in the linked source
  repository opened through `/sase_repo`.
