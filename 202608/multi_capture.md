---
tier: epic
status: done
title: Multi-item capture for Bob Mac Capture
goal:
  One Bob Mac Capture draft atomically captures every blank-line-separated task or note,
  with intuitive native editing, complete preview and notification feedback, and
  backward-compatible Bob protocols.
phases:
  - id: bob_batch_protocol
    title: Add Bob's batch grammar, protocol, and atomic transaction
    depends_on: []
    size: medium
    description:
      "bob_batch_protocol: implement shared item boundaries, cursor-aware parse and
      completion, additive ordered output, and all-or-nothing multi-file capture."
  - id: mac_batch_integration
    title: Integrate batch results and native editor behavior in the mac app
    depends_on:
      - bob_batch_protocol
    size: medium
    description:
      "mac_batch_integration: decode and present aggregate results, add reliable
      indentation-aware Control-J edits, and preserve one-process submission semantics."
  - id: mac_notification_polish
    title: Deliver complete, polished single and batch notifications
    depends_on:
      - mac_batch_integration
    size: small
    description:
      "mac_notification_polish: render richer ordered notification summaries and route
      singular or plural open actions to every unique captured destination."
proposed_by: bbugyi200.athena.024.w1
create_time: 2026-09-09 20:00:14
---

- **PROMPT:**
  [prompts/202608/multi_capture.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/multi_capture.md)

# Multi-item capture for Bob Mac Capture

## Goal

Let one Bob Mac Capture draft create multiple Obsidian tasks or notes. A run of one or
more whitespace-only physical lines separates capture items; each nonempty item retains
Bob's existing parent-plus-authored-sub-bullets grammar. Keep the single-item experience
compact and backward compatible, make Control-J an intuitive bridge from authored
bullets to the next item, and show a complete, polished preview and notification for
every committed result.

This is a coordinated `bob-cli` / `bob-mac-capture` protocol change. The mac app must
continue delegating capture grammar, preview, completion, and vault mutation to Bob; it
must not split the draft into independent subprocess submissions, because a late failure
would otherwise leave a partially captured batch and invite duplicate resubmission.

## Product and protocol decisions

- Treat one or more blank or whitespace-only physical lines as one separator. Ignore
  leading/trailing separator runs and empty groups; a draft with no non-whitespace item
  still fails as missing text. Within an item, the first nonblank line is its parent and
  subsequent contiguous `-`/`*`/`+` rows use the existing zero- or two-space authored
  child grammar. A blank line therefore ends the current child hierarchy, and the next
  item begins with a new parent line.
- Parse, complete, dry-run, and submit the same ordered item boundaries. Completion is
  scoped to the item containing the UTF-8 cursor, and every span, diagnostic, and
  replacement range remains expressed in byte offsets of the complete original draft.
- Preserve the exact existing JSON shape for a single capture. For a multi-capture
  success, keep the first result's legacy top-level fields so older clients still
  recognize that the mutating command succeeded, and add an ordered `captures` array
  containing every per-item result. Add ordered per-item summaries/ranges to
  `capture-parse` without bumping schema version 1; retain aggregate top-level spans and
  diagnostics for tolerant existing clients. Failures retain
  `{ "ok": false, "error": ... }` and identify the offending item/physical line without
  echoing the whole draft.
- Plan the complete batch against an in-memory view of every affected note (and daily
  ledger), then stage and commit all unique outputs as one transaction. Later items see
  earlier planned edits to the same target, so insertion order, duplicate block-ID
  checks, sub-bullet lookup, and Pomodoro links match a successful sequential capture
  without exposing intermediate state. Any parse, clipboard, validation, staging, or
  replace failure leaves notes, ledgers, and newly-created clip assets at their original
  state; `--dry-run` performs the same planning with no writes.
- Apply command-level forcing options consistently to every item. Resolve per-item
  trailing markers independently. Derive stable item-specific priority-roll randomness
  from `BOB_PRIORITY_ROLL_SEED` plus source order so live preview, explicit preview, and
  submission agree without coupling every `p:<N>` item to the same roll. Reserve clip
  asset names across the batch so same-timestamp items cannot collide.
- Control-J remains a native text edit. Normally it inserts a canonical `- ` row at the
  caret, copying exactly the current authored row's supported indentation (zero or two
  ASCII spaces). If the current physical line contains only optional whitespace, one
  Markdown bullet marker (`-`, `*`, or `+`), and optional whitespace, replace that
  placeholder with exactly one blank separator and put the caret at the beginning of the
  following line. Reuse an existing line terminator when present so the shortcut never
  creates two blank separator rows accidentally. Multi-line selections keep the normal
  replacement behavior rather than triggering the placeholder special case.
- Preview every item in source order, with a compact count and visually separated
  destination/block groups inside the existing scrollable auxiliary region. Preserve
  Bob's exact rendered block lines and avoid nested scrolling. Status and VoiceOver
  announcements say `Captured` for one item and include the item count for a batch.
- Notifications use the semantic parent text rather than raw draft syntax. One result is
  titled `Task captured` or `Note captured`, names its destination, and includes its
  captured text (plus a scheduled date when present). A batch is titled with its item
  count, summarizes task/note and destination counts, and includes one ordered body line
  per capture; application code must not truncate or replace entries with an ellipsis.
  Notification click/actions and Command-Return open de-duplicated destination paths in
  source order, using singular/plural action labels as appropriate.

## Compatibility and safety invariants

- A single-line or authored-sub-bullet-only draft produces the same Markdown, JSON,
  human output, completion behavior, and app presentation it does today, apart from the
  intentionally richer success notification.
- One Bob invocation owns one batch. The app clears and dismisses its draft only after
  decoding a successful aggregate response; any command or transport failure retains the
  full draft and reports no partial success.
- Multi-item JSON is additive and ordered. An older mac client paired with the newer Bob
  sees the legacy top-level first result and therefore clears a successfully committed
  batch instead of reporting a decode failure that could cause duplicate resubmission.
- Blank-line parsing accepts LF, CRLF, and bare CR with correct global UTF-8 ranges.
  Separator rows themselves have no marker completion or highlighting.
- Logs and errors preserve the existing captured-text redaction guarantees. The success
  notification is the only newly authorized surface for semantic capture text.

## Phase 1: Add Bob's batch grammar, protocol, and atomic transaction

Work in `bob-cli`.

1. Refactor `src/native/capture_language.rs` around a shared source-range-aware item
   splitter used by strict execution parsing, editor parsing, and cursor completion.
   Parse each item with the established single-capture grammar, translate local spans
   and diagnostics to global UTF-8 ranges, and preserve item/physical-line context in
   strict errors. Update the blank-line, placeholder, nested-child, CR/LF, multiple
   marker, and cursor-boundary tests; specifically retire tests that treat a true blank
   row as ignorable inside one authored-child hierarchy while preserving marker-only
   placeholder behavior.
2. Extend `src/native/capture_parse.rs` with additive ordered item summaries/ranges and
   aggregate global spans/diagnostics while retaining schema version 1 and the legacy
   top-level view. Update `src/native/capture_complete.rs` so marker and same-note
   wikilink completion use the cursor's active item and return draft-global replacement
   ranges; return an empty success while the cursor is on a separator.
3. Refactor `src/native/capture.rs` into parse/plan/commit stages that can plan all
   items against evolving in-memory note and ledger snapshots. Generalize the current
   one- or two-file replacement code into a staged multi-file transaction with rollback,
   and fold clip assets into the same cleanup guarantees. Detect within-batch duplicate
   IDs, preserve source-order insertion when items share a target, reserve asset names,
   and make seeded priority rolls deterministic per item. Keep dry-run on the exact same
   planner with its commit step disabled.
4. Keep single-success JSON unchanged. Add the compatibility-preserving `captures` array
   and readable numbered human output only when multiple items are present; keep
   existing failure JSON and enrich errors with bounded item/line location. Document
   separator, option, output, preview, clipboard, and transactional semantics in
   `README.md` and command help.
5. Add focused unit and CLI integration coverage in `src/native/capture_language.rs`,
   `src/native/capture.rs`, `src/native/capture_parse.rs`,
   `src/native/capture_complete.rs`, and `tests/cli.rs` for mixed task/note routes,
   multiple items targeting one and several files, authored children on each item,
   trailing/consecutive separators, completion in later items, legacy single JSON, multi
   JSON ordering, seeded schedules, clip-name collisions, dry-run, validation failure
   before mutation, staged-write failure rollback, and no orphaned temporary or
   attachment files.

Validation:

- `cargo fmt --check`
- `cargo clippy --all-targets --all-features`
- `cargo test`
- Manually compare `bob capture-parse`, `bob capture-complete`, and
  `bob capture --dry-run --format json` for a single draft and a two-item draft,
  including a cursor/marker in the second item.

## Phase 2: Integrate batch results and native editor behavior in the mac app

Work in the linked `bob-mac-capture` repository, opened through `sase repo open`.

1. Extend `Sources/CaptureCore/CaptureModels.swift` to decode Bob's additive parse-item
   metadata and multi-success `captures` array while normalizing legacy single responses
   to a one-item collection. Preserve tolerant decoding of old Bob output and all
   current per-item helpers such as exact preview block lines. Cover legacy, additive,
   malformed, and mixed multi-result fixtures in `CaptureModelTests`.
2. Update `BobProcessClient`, its fake Bob fixture, and process-client tests so full
   multi-line drafts continue traveling as one argv value and parse, completion, live
   preview, explicit preview, and submit all exercise the aggregate contract. Do not add
   client-side item splitting or multiple mutating subprocess calls.
3. Replace the ad-hoc Control-J insertion in `CapturePanelController` with a pure,
   deterministic edit resolver that returns replacement text/range and final selection.
   Implement bullet-only placeholder-to-separator behavior, reuse an existing line
   delimiter when necessary, and copy the current valid zero/two-space bullet
   indentation for ordinary insertion. Keep completion dismissal, native undo/IME, and
   accessibility behavior. Add matrix tests for `-`/`*`/`+`, leading/trailing
   whitespace, top-level/nested/current-parent rows, EOF and middle-of-draft rows,
   collapsed and multi-line selections, and unrelated/noneditable responders.
4. Teach `CapturePanelModel` to retain the aggregate success, produce count-aware
   destination/status/VoiceOver summaries, clear only after aggregate success, and open
   each unique target in source order for Command-Return. Preserve the full draft on all
   failures and keep the single-item state behavior unchanged.
5. Redesign `PreviewPane` to render a one-item compact view or an ordered multi-item
   stack with count, kind/destination metadata, dividers/cards, and each result's exact
   `previewBlockLines`. Let the panel's existing auxiliary overflow region own scrolling
   and add accessibility labels that expose item position/count and destination. Update
   model/view layout tests and the runtime/keyboard/preview contract in `README.md`.

Validation:

- `just format-lint`
- `just build`
- `just test`
- On macOS, exercise a two-item draft with a nested authored child, verify live and
  explicit previews match Bob's dry-run order, verify Control-J at top-level/nested rows
  and at both placeholder depths, and verify Return versus Command-Return success and
  failure retention behavior.

## Phase 3: Deliver complete, polished single and batch notifications

Work in the linked `bob-mac-capture` repository, opened through `sase repo open`.

1. Introduce a pure notification presentation builder in `NotificationService.swift`
   that maps ordered capture results to singular/plural title, destination and kind
   summary, full ordered body lines, scheduled details, and de-duplicated target paths.
   Use friendly `Task`/`Note` labels for Bob's task, Pomodoro-task, bullet, and
   sub-bullet kinds, with a safe fallback for future kinds.
2. Register singular and plural notification categories (`Open Note` / `Open Notes`),
   store backward-compatible single-target metadata plus an additive target-path array,
   and route a notification click or action to every unique Obsidian URL in source
   order. A result with no usable target remains informative without exposing a broken
   action.
3. Pass the full normalized success collection from `CapturePanelModel`, and expand
   `NotificationServiceTests` and model tests to cover richer single task/note content,
   same-target and cross-target batches, mixed kinds, scheduled items, stable ordering,
   destination de-duplication, category/action selection, legacy user-info routing, and
   the guarantee that every captured semantic text appears once in the batch body.
4. Update the README notification runtime contract and perform a signed installed-app
   smoke test: single task, single note, same-target batch, cross-target batch,
   foreground banner, Notification Center expansion, singular/plural actions, and
   successful opening of all intended notes.

Validation:

- `just format-lint`
- `just build`
- `just test`
- `just bundle` followed by the signed installed-app notification smoke matrix above.

## Epic completion criteria

- The same draft can contain at least two independently routed task/note captures,
  separated by a whitespace-only blank line, with authored first- and second-level
  bullets retained under the correct parent item.
- Bob either commits every item and related ledger/clip output or leaves the vault
  unchanged, and dry-run/live preview/reporting preserve exact source order.
- Control-J inherits supported indentation and converts any whitespace-wrapped empty
  Markdown bullet row into exactly one separator plus a caret-ready next parent line.
- The panel preview, status, accessibility announcements, Command-Return opening, and
  success notification cover all results, while a single result remains compact and is
  more informative than today's destination-only notification.
- Both repositories' full format, lint, build, and test suites pass, and the signed
  macOS smoke matrix confirms the real notification and keyboard experience.
