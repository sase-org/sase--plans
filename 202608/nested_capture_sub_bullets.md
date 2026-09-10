---
tier: tale
title: Add nested authored sub-bullets to Bob capture
goal:
  Let capture drafts use exactly two source spaces to nest one authored bullet beneath
  the preceding authored child, consistently in bob-cli and bob-mac-capture.
size: medium
proposed_by: bbugyi200.athena.022.f0
create_time: 2026-09-09 20:00:14
status: wip
---

# Add nested authored sub-bullets to Bob capture

## Goal

Extend Bob's existing physical-line-aware capture syntax from one flat authored-child
level to a bounded two-level hierarchy. A draft such as:

```text
Prepare the launch review
- Confirm the rollout owner
  - Send the owner the final date
- Attach the final checklist @work p:1
  - Verify the links
```

must create the ordinary captured task/bullet plus two first-level authored children,
with each two-space-indented item rendered as a child of the immediately preceding
first-level authored child. Keep bob-cli authoritative for parsing and rendering, and
make the linked bob-mac-capture client decode, preview, submit, and document the richer
draft without implementing a second capture grammar.

This is a medium tale because one agent can implement and validate the coordinated
contract end to end, but the work crosses the shared Rust execution/editor/completion
grammar, hierarchy-aware rendering and JSON output, CLI documentation and integration
coverage, and the Swift client's version-tolerant models and fake-bob boundary.

## Required syntax and hierarchy

- The first physical line remains the captured parent. Every later nonblank line may be
  either a first-level authored bullet at column zero or a second-level authored bullet
  prefixed by exactly two ASCII spaces. At either level, accept `-`, `*`, or `+`
  followed by at least one space or tab, strip that source marker/separator, normalize
  body whitespace as today, and render the canonical `- <body>` marker.
- Support exactly one authored nesting level. One space, three or more spaces, a leading
  tab, four-space/deeper nesting, wrapped prose, and ordinary continuation text remain
  invalid and fail `bob capture` before any write. Update `invalid_child_line` wording
  to name both accepted shapes rather than describing every continuation as flat.
- A nonempty two-space-indented item belongs to the nearest preceding valid, nonempty
  column-zero authored bullet. If no such item has appeared, execution fails with a
  line-numbered orphan error and `capture-parse` reports a stable
  `orphaned_nested_bullet` diagnostic over that physical line. Starting a new
  column-zero bullet changes the owner for subsequent nested items.
- Preserve interactive placeholder behavior at both levels: blank lines, `- `/`* `/`+ `,
  and ` -`/` *`/` +` with no body produce no item or diagnostic. Ignored blank or
  placeholder rows do not clear the most recent first-level owner, so a later nonempty
  nested item still attaches to that owner; an empty nested placeholder is harmless even
  before an owner exists.
- Preserve LF, CRLF, and bare-CR input parity, Unicode body text, mixed source bullet
  markers, and source order. Render the hierarchy depth-first: an owning first-level
  item, all of its nested items, then the next first-level item. Authored items as a
  whole stay before clipboard children and the priority schedule log; clipboard and
  schedule-log items remain children of the captured parent rather than accidentally
  nesting under the last authored item.
- Capture-wide terminal `s:<N>`, `p:<N>`, `%...`, and route/mode markers may appear on a
  valid nested item exactly as they may on a current flat authored child. Strip them
  from that item's body, retain the one-marker-per-slot rule across the whole draft,
  reject an item emptied only by marker removal, preserve forced-route literal behavior,
  and continue allowing a later line to resolve the overall route/mode. Only the first
  physical line keeps leading-route syntax.
- Cursor completion must recognize a trailing marker in a valid nested item's body,
  compute replacement ranges against the original UTF-8 input, and return no completion
  while the caret is on its two-space/list-marker prefix. Malformed or orphaned lines do
  not become an alternate completion grammar.

## Rendering and wire compatibility

- Represent authored items internally as body plus semantic depth instead of inferring
  hierarchy again during output. Choose the destination note's existing tab-or-two-space
  indentation unit exactly once. Render first-level authored items with one unit and
  nested items with two units; a fresh note keeps today's tab fallback. For a
  `sub_bullet` capture beneath an existing task, the existing parent-task indentation is
  still prefixed to the complete captured block, so both authored levels remain relative
  to the newly captured bullet.
- Keep `bob capture --format json` backward compatible: `sub_bullets` remains the
  optional source-ordered array of exact rendered Markdown lines, now including both
  first- and second-level indentation. Human output and dry-run print the same exact
  lines, and the field remains omitted when the draft emits no authored item.
- Keep `capture-parse` schema version 1 and its optional `sub_bullets: [String]` field
  as the source-ordered normalized bodies expected by existing clients. Add an optional
  parallel `sub_bullet_depths: [1 | 2]` field with exactly one entry per emitted
  `sub_bullets` item, so editors can preserve hierarchy without an incompatible type
  change. Omit both arrays when no authored items exist. Older clients may ignore the
  additive field; bob-mac-capture should synthesize depth `1` for every parsed body when
  talking to an older bob that omits it, and should reject or safely ignore mismatched
  depth/body counts rather than indexing unsafely.
- Do not add capture modes, needs, completion contexts, or semantic span kinds for list
  indentation. The raw input owns the list prefix; existing marker and wikilink spans
  continue to point only at recognized syntax within each body.

## Implementation

1. Refactor the shared grammar in `src/native/capture_language.rs` around an explicit
   authored-item type carrying normalized body and depth. Replace the flat
   `strip_bullet_marker` assumption with one canonical continuation-line classifier used
   by execution, editor parsing, and completion. Track the latest emitted first-level
   owner while walking physical lines; enforce exact zero/two-space syntax, placeholder
   handling, orphan rejection, marker aggregation, and absolute byte offsets in that one
   pass rather than reparsing indentation in each caller. Extend the unit matrices for
   valid mixed hierarchies, owner changes, placeholders/blanks, orphan/deeper/bad-indent
   errors, CRLF/bare-CR, Unicode, capture-wide markers, execution/editor parity, and
   nested-line completion.

2. Update `src/native/capture.rs` to render each authored item from its depth and the
   target-selected indentation unit, preserving depth-first source order and the current
   captured-parent/authored/clipboard/schedule-log block order. Ensure ordinary tasks,
   section bullets, ID-only tasks, Pomodoro-linked tasks, and captures beneath existing
   tasks all place the nested lines at the correct relative depth. Keep line-ending
   preservation, dry-run, placement, duplicate-ID preflight, and single-/two-note atomic
   write behavior unchanged. Extend human/JSON serialization tests and command help.

3. Update `src/native/capture_parse.rs`, `src/native/capture_complete.rs`,
   `tests/cli.rs`, and the bob-cli `README.md`. Serialize and document
   `sub_bullet_depths` additively at schema version 1, render hierarchy legibly in human
   parse output, and document exact source indentation, ownership, placeholders,
   rejected deeper indentation, marker behavior, target-selected output indentation, and
   the unchanged JSON contract. Add end-to-end tests for stdin/argv, fresh tab-indented
   and existing two-space-indented targets, dry-run/no-write failures, all capture
   kinds, exact JSON lines/depths, parse diagnostics/ranges, and completion on nested
   lines, plus flat-draft regressions.

4. Open the linked `bob-mac-capture` repository through `sase repo open bob-mac-capture`
   before reading or editing it. Extend `CaptureParseResponse` with version-tolerant
   `sub_bullet_depths` decoding while retaining `[String]` decoding for both parse and
   capture success. Keep preview driven by bob-cli's exact rendered `sub_bullets` lines,
   pass the original multiline draft unchanged for parse/live-preview/explicit-preview/
   submit, and do not reproduce indentation validation or owner association in Swift.
   Keep the existing Ctrl-J top-level row shortcut unchanged; users author a nested row
   with a newline and the documented two spaces rather than introducing an unrelated
   key-binding change in this feature.

5. Extend bob-mac-capture's `Tests/Fixtures/fake-bob`, CaptureCore/process-client tests,
   panel-model tests, and `README.md` with one draft containing multiple first-level
   items and corresponding nested items. Prove the full draft remains one argv element,
   parse decoding retains bodies plus depths, live/explicit preview renders the nested
   Markdown lines verbatim, submission keeps the hierarchy, older responses without the
   new field remain compatible, and no new completion/highlighting behavior is inferred
   in Swift.

## Validation

- In bob-cli, run focused Rust unit and CLI integration tests for capture-language line
  classification, capture execution/rendering, capture-parse diagnostics/JSON,
  capture-complete nested-line cursor behavior, and regression coverage for existing
  flat authored children, clipboard children, schedule logs, ID-only, Pomodoro, section,
  and existing-task sub-bullet modes. Then run `just all` for formatting, Clippy, and
  the full suite, followed by `git diff --check`.
- In bob-mac-capture, syntax-check the fake-bob fixture, then run `just format-lint`,
  `just test`, and `just build`. If the host still lacks a selected Apple
  developer/Xcode toolchain, report that environmental limitation explicitly after
  running every available fixture/source check rather than treating it as a feature
  regression.
- Re-read bob-cli's capture/capture-parse/capture-complete help and README contracts and
  bob-mac-capture's keyboard, draft, and preview documentation after validation. Confirm
  that examples, accepted indentation, diagnostic names, depth values, fallback
  behavior, and exact rendered output agree across both repositories.

## Acceptance criteria

- A draft `Parent\n- First\n  - First detail\n- Second\n  - Second detail` writes each
  detail beneath its corresponding authored child, using two destination indent units
  for the nested line and never attaching it to the captured parent or wrong sibling.
- Existing flat drafts behave byte-for-byte as before. Blank/placeholder lines stay
  harmless, while an orphaned nonempty nested item and every unsupported indentation
  depth fail before writes and produce stable line-specific editor diagnostics.
- Markers and completion work from nested bodies with correct UTF-8 ranges and the same
  capture-wide precedence/duplicate rules as first-level bodies.
- Human output, dry-run, `capture` JSON, and the target note agree on exact nested
  lines; `capture-parse` retains its string array and adds aligned depths without
  raising schema version 1 or breaking older clients.
- bob-mac-capture accepts the draft, displays the exact hierarchy in live and explicit
  preview, submits it unchanged, and remains compatible with bob versions that omit the
  additive depth field, without adding a Swift-side capture parser.
- All available formatting, lint, build, and test checks pass, with unrelated or
  toolchain-only failures clearly distinguished from regressions.
