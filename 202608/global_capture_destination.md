---
tier: epic
status: done
title: Global capture destination declarations
goal: "Bob capture drafts can declare one shared task or parent-task destination with @@
  syntax, item-local markers override it reliably, and bob-cli plus Bob Mac Capture
  provide coherent parsing, completion, diagnostics, preview, notifications, and atomic
  writes.

  "
phases:
  - id: global-destination-protocol
    title: Authoritative bob-cli grammar, execution, and protocol
    depends_on: []
    size: medium
    description:
      "global-destination-protocol: implement the shared draft envelope, inherited
      execution semantics, additive editor/output protocol, documentation, and Rust
      coverage."
  - id: mac-global-destination-experience
    title: Bob Mac Capture integration and global-aware presentation
    depends_on:
      - global-destination-protocol
    size: medium
    description:
      "mac-global-destination-experience: consume the finalized protocol for completion,
      diagnostics, preview, notifications, accessibility, documentation, and Swift
      coverage."
proposed_by: bbugyi200.athena.0c9.w0
create_time: 2026-09-09 20:00:08
---

- **PROMPT:**
  [prompts/202608/global_capture_destination.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/global_capture_destination.md)

# Global capture destination declarations

## Goal

Add one visually clear, document-scoped destination declaration to Bob's capture draft
grammar and carry it through the authoritative Rust protocol and Bob Mac Capture without
duplicating grammar in Swift. A draft headed by `@@foo` sends every otherwise-unrouted
item to `foo.md`; a draft headed by `@@foo+a-id` inserts every otherwise-unrouted item
as its own direct child beneath task `^a-id` in `foo.md`. Any item-local destination or
mode marker wins for that item.

The result should feel like a small header for the whole draft: easy to discover through
completion, obvious in highlighting and preview, strict about mistakes, atomic at write
time, and concise in success notifications.

## Product and protocol contract

### Canonical syntax and placement

- A global declaration is exactly one whitespace-free token on the first nonblank
  physical line of the draft. Leading blank lines remain harmless. The first capture
  item may start on the immediately following line or after any run of blank separator
  lines, so both of these are valid:

  ```text
  @@foo
  First task

  Second task
  ```

  ```text
  @@foo

  First task

  Second task
  ```

- `@@<route>` supplies task mode and the routed note for every item that has no local
  destination/mode marker.
- `@@<route>+<block-id>` supplies sub-bullet mode, note, and parent task for every item
  that has no local destination/mode marker. Every capture parent is inserted as a
  separate direct child of the selected task in source order; authored children remain
  nested beneath their own capture parent. Route and block-ID validation are identical
  to the existing `@route` and `@route+block-id` grammar.
- The header is metadata, never a capture item: it is absent from item counts, semantic
  capture text, preview blocks, notifications, and note contents. A draft containing
  only a header fails with an actionable “add a capture item” diagnostic/error.
- Reserve `@@` for this document-scoped grammar. A second declaration, a declaration
  after capture content, extra text/markers on the declaration line, or unsupported
  global forms such as `@@foo#Ideas`, `@@foo^id`, and `@@foo:id` are errors rather than
  literal task text. Literal `@@...` text can still be protected as inline code.
- Do not combine a textual global declaration with destination-forcing CLI options
  (`--route`, `--section`, `--task`/`--task-ref`, or `--task-section`); report the two
  competing document-wide controls instead of silently choosing one. Clipboard,
  scheduling, priority, dry-run, formatting, and stdin behavior remain composable.

### Inheritance and override precedence

Resolve each real item in this order:

1. An item-local route/mode marker wins. At minimum this covers the requested `@bar` and
   `@bar+b-id`; apply the same coherent rule to every existing local marker (`@bar#...`,
   `@bar^...`, `@bar:...`, and the bare Pomodoro-note `#`).
2. Otherwise inherit the complete `@@...` declaration.
3. Otherwise retain today's `mac_inbox.md` task default.

Local scheduling, priority, clipboard, and authored-child markers remain item-scoped. A
local override is not a duplicate route diagnostic merely because a global header
exists. Existing batch planning remains transactional, and later items must continue to
see earlier staged edits to the same note/parent.

### Editor and JSON contract

- Refactor the shared capture-language layer around one draft/envelope parse that
  identifies the optional header and the real capture items while preserving original
  UTF-8 byte offsets, CRLF/bare-CR line handling, and physical line numbers. Execution,
  `capture-parse`, `capture-complete`, and same-note wikilink completion must all
  consume this representation.
- Keep schema version 1 and make the wire changes additive. `capture-parse` exposes an
  optional `global_destination` object with the declaration range, effective mode,
  route, block ID, and `needs`; its top-level/`items` route, mode, and block-ID fields
  describe each item's effective destination after inheritance. Real item indices and
  ranges exclude the header but continue to index the original draft.
- Add distinct semantic spans for the global route and global sub-bullet route/block ID
  (plus the existing interactive-placeholder treatment for incomplete separators), so
  clients can highlight and cache-complete them without guessing from raw punctuation.
  Diagnostics remain structured, ranged, and nonfatal in `capture-parse`, while
  `bob capture` returns the matching strict usage error.
- `bob capture` success JSON adds an optional top-level `global_destination` summary
  (`mode`, `route`, and optional `block_id`). Preserve all legacy single-item fields and
  the ordered `captures` array exactly; the header must not create an extra result.
- `capture-complete` supports the route component of `@@`, `@@fragment`, and
  `@@route+...`, plus the existing-task component of `@@route+fragment`. Replacement
  ranges exclude both `@` sigils and the `+`, all offsets remain draft-global UTF-8 byte
  offsets, and `--all-tasks` continues to offer missing-ID tasks for the app's Add ID
  workflow. Completion elsewhere remains scoped to the cursor's real item. An inherited
  global route also becomes the current note for same-note wikilink heading/block
  completion unless that item overrides it.

### Bob Mac Capture presentation

- Swift continues to submit the untouched draft and treats bob-cli as the only grammar
  authority. Decode the additive global metadata tolerantly so older `bob` binaries
  remain usable.
- Map the new global span kinds into the existing destination/block-ID semantic palette,
  request completion for them, and make cached route completion strip the complete `@@`
  sigil. Server task completion, including the missing-ID prompt and stale-safe byte
  replacement, must work on `@@route+...` exactly as it does on local `@route+...`.
- Present a compact scope summary in preview/destination details: `All items → foo.md`
  or `All items → foo.md · under ^a-id`. Continue to show every exact preview block in
  order, and identify destinations that differ from the shared declaration as local
  overrides rather than implying the whole batch moved.
- Make notifications global-aware but result-driven. A same-destination global batch is
  concise (for example, `2 tasks · foo.md`; a shared-parent batch also says
  `under ^a-id`) and does not repeat the same arrow/filename on every body line. Items
  that override the header name their actual destination. Open Note/Open Notes actions
  are still derived from the ordered unique targets returned by `bob capture`, so mixed
  override batches open every real destination and a shared batch opens only once. Never
  include the raw `@@...` token in notification text or diagnostics history.

## Compatibility and boundaries

- No new CLI option or subcommand is introduced. Existing drafts without `@@` keep their
  parse, completion, JSON, file mutation, and notification behavior.
- Do not add global section, authored-ID, Pomodoro-link, schedule, priority, or
  clipboard declarations in this change. The only global destination forms are `@@route`
  and `@@route+block-id`.
- Do not duplicate capture splitting or marker validation in Swift. App behavior is
  driven by bob-cli parse/completion/output metadata and actual capture results.
- Preserve atomic batch rollback, note indentation/line endings, managed-log insertion,
  duplicate-ID checks, privacy redaction, keyboard/accessibility behavior, and the
  single-process batch submission model.

## Phase 1: Authoritative bob-cli grammar, execution, and protocol

Implement the contract in the `bob-cli` repository.

### Work

1. Introduce a typed optional global destination/declaration in
   `src/native/capture_language.rs` and refactor draft splitting so all parser consumers
   share header detection, real-item boundaries, effective inheritance, diagnostics,
   spans, and UTF-8 ranges. Keep item-local markers authoritative over the inherited
   kind/route.
2. Thread the declaration through strict batch parsing and `src/native/capture.rs`.
   Apply it before per-item planning without weakening the existing in-memory planner,
   sub-bullet insertion rules, duplicate block-ID validation, or transactional commit.
   Add the top-level success metadata and a useful human summary without changing the
   legacy item result shape.
3. Extend `src/native/capture_parse.rs` and `src/native/capture_complete.rs` with the
   additive JSON/editor contract, cursor-aware global route/task fields, inherited
   same-note wikilink routing, and actionable malformed/misplaced/conflicting-header
   diagnostics.
4. Update concise CLI help, `README.md`, and `docs/capture.md` with the header examples,
   override precedence, supported/unsupported forms, completion behavior, and atomic
   semantics.
5. Add focused unit and CLI integration coverage for:
   - header placement with/without a blank line, leading blanks, CRLF, Unicode offsets,
     a header-only draft, and malformed/duplicate/misplaced declarations;
   - `@@foo` batches, local `@bar` and other local-mode overrides, output item counts,
     ordered same-note staging, dry-run, stdin, human output, and JSON compatibility;
   - `@@foo+a-id` direct-child ordering, authored subtrees, managed-log placement,
     missing/ambiguous parent errors, local `@bar+b-id` and plain-route overrides, and
     rollback after a later failure;
   - parse metadata/effective item fields/spans/diagnostic ranges and the forced-option
     conflict;
   - route and task completion on the header, `--all-tasks` missing-ID candidates,
     global replacement ranges, local-item completion, separator behavior, and inherited
     same-note wikilink completion.

### Validation

Run `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`.
Exercise representative `bob capture`, `capture-parse`, and `capture-complete` commands
against temporary vault fixtures and confirm failed mixed/global batches leave every
fixture byte-for-byte unchanged.

## Phase 2: Bob Mac Capture integration and global-aware presentation

Consume the finalized bob-cli protocol in the linked `bob-mac-capture` repository.

### Work

1. Add tolerant `CaptureCore` models for parse/success global metadata and the new
   spans; keep absent fields compatible with older bob binaries. Update the fake-bob
   fixture to emit realistic global parse, completion, preview, capture, override, and
   failure responses without teaching Swift to parse the grammar.
2. Extend `CapturePanelModel` completion gating and cached replacement handling for both
   global route span families. Verify route completion, task completion, the Add block
   ID prompt, accepted UTF-8 replacement/cursor ranges, stale-response protection, parse
   diagnostic status, and effective inherited/local-override preview behavior.
3. Update semantic span mapping, preview/destination presentation, accessibility labels,
   and VoiceOver announcements with the compact shared-scope summary and explicit local
   overrides. Keep exact Markdown preview blocks supplied by Bob.
4. Pass the global summary alongside normalized capture results to
   `NotificationService`; implement the concise same-scope/shared-parent wording while
   preserving ordered semantic text, scheduled dates, privacy, and result-derived unique
   target actions for mixed overrides.
5. Document the grammar and UI in the app `README.md`. Add `CaptureCore`, model/view,
   and notification tests for tolerant decoding, new palette/span mappings, cached and
   server completion, missing-ID assignment, ranged diagnostics, same-target task
   batches, shared-parent note batches, mixed local overrides, exact target opening, and
   exclusion of raw declaration syntax from notification/diagnostic history.

### Validation

Run `just format-lint`, `just build`, and `just test` in `bob-mac-capture`. Re-run the
relevant bob-cli protocol tests with the app fixture expectations, then manually smoke
the panel with `@@foo`, `@@foo+a-id`, and a mixed override draft to inspect completion,
highlighting, preview hierarchy, VoiceOver labels, success notification wording, and
Open Note/Open Notes routing.

## End-to-end acceptance examples

Given:

```text
@@foo
First task

Second task @bar

Third task
```

Bob produces exactly three capture results: tasks one and three land in `foo.md`, task
two lands in `bar.md`, the header appears in neither note, and any failure rolls back
both targets. Preview and notification count three items and two destinations;
completion on `@@fo` replaces only `fo`, while completion on `@ba` remains item-local.

Given:

```text
@@foo+a-id
First note
- authored detail

Second note

Independent task @bar

Different parent @bar+b-id
```

Bob inserts the first and second capture parents as ordered sibling children of `^a-id`
in `foo.md`, keeps the authored detail beneath the first capture, creates an ordinary
task in `bar.md`, and inserts the fourth item beneath `^b-id` in `bar.md`. The app shows
the shared parent plus two overrides, notifications expose only semantic capture text,
and Open Notes opens `foo.md` then `bar.md` once each.
