---
tier: tale
title: Add a bare `#` Pomodoro-note marker to `bob capture`
goal:
  A trailing bare `#` token in `bob capture` text captures the item as a plain-text
  sub-bullet note on the current (or next future) Pomodoro in the Bob daily note, with
  matching `bob capture-parse` support, help text, README docs, and tests.
size: medium
proposed_by: bbugyi200.athena.05m
create_time: 2026-09-09 19:59:55
status: wip
---

# Plan: bare `#` Pomodoro-note marker for `bob capture`

## Goal

Teach the `bob capture` grammar one new terminal marker: a bare `#` token at the end of
a line. When present, the capture item is written as an ordinary plain-text sub-bullet
(`- <body>`, no `#task`, no `[created::]`, no block ID) beneath the **current** Pomodoro
in the Bob daily note, falling back to the **next future** Pomodoro when no Pomodoro is
currently running.

`bob capture-parse` must report the new mode and span so live editors (bob-mac-capture,
the Hammerspoon capture UI) keep a usable preview.

## Background: what exists today

Read these before writing code. Everything below was verified against the tree at the
time this plan was authored; re-check line numbers, they drift.

### Grammar (`src/native/capture_language.rs`, ~4.5k lines)

This module is the single source of truth for capture-text classification. `capture.rs`
layers execution on it; `capture_parse.rs` layers a span-aware read-only editor view on
the _same_ functions. Never fork the rules.

Key items:

- `enum CaptureKind { Task, TaskWithBlockId, Bullet, Pomodoro, SubBullet }` — the mode a
  parsed item resolved to.
- `struct RouteToken { route: String, kind: CaptureKind }` — one recognized `@...`
  token.
- `fn parse_capture_item(...)` (~line 480) — splits an item into a parent line plus
  authored child lines, calls `resolve_line` on each, and folds the per-line results
  into `AggregateMarkers`.
- `fn resolve_line(tokens, leading, detect_route, parse_clip_markers)` (~line 602) — one
  physical line's marker resolution. It:
  1. calls `extract_terminal_markers` (strips trailing `s:<N>`, `p:<N>`, `%...` on
     either side of a trailing route token),
  2. calls `reject_legacy_bullet_markers`,
  3. returns early with the body when `detect_route` is false,
  4. lets a _leading_ route token win (parent line only),
  5. calls `validate_special_terminal_markers_line`,
  6. otherwise lets a _trailing_ route token route the preceding body.
- `struct AggregateMarkers` + `fn absorb` (~line 668) — four item-wide slots
  (route/mode, schedule, priority, clipboard). A second line resolving the same slot is
  `duplicate_marker_error(...)`. The route slot's label is currently
  `"route/mode marker (@route)"`.
- `fn is_route_marker(token)` (~line 1152) — the predicate `extract_terminal_markers`
  uses to find the route position in the terminal region. This is why `%`/`s:`/`p:`
  compose with `@route` in either order.
- `fn reject_legacy_bullet_markers(tokens, allow_route)` (~line 1173) — **the current
  blocker.** It rejects (a) any final token starting with `#`, and (b) a final plain
  `@route` preceded by a `#...` token. Bare `#` hits branch (a) today and fails with
  `legacy_marker_error()`:
  `"bullet section markers must be appended to an @route token; use @foo#bar instead of #bar @foo"`.
- `enum EditorMode { Task, Bullet, PomodoroTask, SubBullet, Incomplete }`,
  `enum Need { Route, Section, BlockId, PomodoroId, Task }`, `enum SpanKind { ... }` —
  the `capture-parse` wire vocabulary.
- `fn legacy_bullet_marker_diagnostic(tokens)` (~line 2114) — the editor-side mirror of
  `reject_legacy_bullet_markers`; emits code `legacy_bullet_marker`.

### Execution (`src/native/capture.rs`, ~4.7k lines)

- `fn plan_capture_item(...)` (~line 495) — per-item planning. Builds `capture_line`
  from `parsed.kind`, assembles `capture_block` (capture line + authored children +
  clipboard children + schedule log), resolves
  `relative_target(parsed.route.as_deref())` (route → `<route>.md`, no route →
  `mac_inbox.md`), then dispatches on `parsed.kind` to one of three planners.
- `fn plan_capture_with_pomodoro_link(...)` (~line 1146) — the `@route:<id>` path.
  Writes the task to the routed note **and** inserts `- [[<route>#^<id>]]` into the
  daily note's ledger.
- `fn insert_pomodoro_block_link(contents, block_link)` (~line 1711) — **the function to
  refactor.** It finds the `## Pomodoros` section via
  `pomodoro::pomodoros_section_range`, walks non-fenced, non-indented lines, collects
  `open` entries (`pomodoro::open_ledger_task`) and `timed` entries
  (`pomodoro::task_time_range`), then:
  - errors `"Bob daily note has multiple open timed Pomodoros"` if `timed > 1`,
  - selects `timed.first()` else `open.first()`, erroring
    `"Bob daily note has no eligible open Pomodoro"` if neither exists,
  - computes the insertion offset with `task_block_end(&lines, selected)` (end of the
    selected entry's whole child block),
  - derives the child indentation from the entry's existing children, then a nearby
    ledger child, then `"  "`,
  - inserts `{indent}- {block_link}` preserving line endings and returns a `Placement`.
- `fn plan_sub_bullet_capture(...)` (~line 1220) — the `@route+<id>` path. Note how it
  indents a _multi-line_ `capture_block` by prefixing every line with the computed
  indentation; the `#` path needs the same treatment.
- `struct CaptureWritePlan { placement, pomodoro, sub_bullet }` (~line 1117).
- `struct CaptureItemResult` (~line 2495) — the JSON/human result shape, including
  `routed`, `route`, `route_label`, `relative_target`, `target`, `kind`, `day_file`,
  `block_link`, `pomodoro_link_placement`, `parent_line`, `parent_text`,
  `parent_status_symbol`, `parent_status_name`.
- `fn print_human_item_success(...)` (~line 2559) — prints the target label
  (`route_label` if non-empty, else `relative_target`), an optional
  `under [<sym>] <parent_text>` line (requires _both_ `parent_status_symbol` and
  `parent_text`), the capture line, children, and an optional `linked <day_file>` block.
- `CaptureError::usage(...)` exits 2; `CaptureError::io(...)` exits 1.

### Ledger vocabulary (`src/native/pomodoro.rs`)

Confirmed against this project's glossary (`sase glossary read Pomodoro`):

- A Pomodoro is a checkbox item in the `## Pomodoros` section of the daily note.
- **Past**: has a timespan `(0900-0930)` and is checked → closed.
- **Current**: has a timespan and is unchecked → open.
- **Future**: has an empty `()` instead of a timespan and is unchecked → open.

So "the current, else the next future Pomodoro" is _exactly_ the existing
`timed.first()`-else-`open.first()` selection in `insert_pomodoro_block_link`. Reuse it;
do not invent a second selection rule.

Useful helpers already exported: `pomodoro::day_file_for(bob_dir)` (honours
`BOB_DAY_FILE`, else `<bob-dir>/YYYY/YYYYMMDD.md`), `pomodoros_section_range`,
`open_ledger_task`, `task_time_range`.

## Grammar specification

A **Pomodoro-note marker** is the whitespace-delimited token `#` — exactly one
character, no suffix. It occupies the same _route/mode slot_ as the `@...` token family,
so it can never coexist with a route marker in one capture item.

### Recognition

1. The marker is recognized only in a line's **terminal marker region**: it must be the
   last remaining token after `extract_terminal_markers` has stripped trailing `s:<N>`,
   `p:<N>`, and `%...` tokens.
2. A **leading** `#` is not a marker. `# foo` keeps today's behaviour (literal body
   text); the user asked for an end-of-line marker only.
3. A `#` in the middle of a line stays literal, exactly like a `#tag` does today.
4. `#<anything>` (e.g. `#Ideas`) keeps today's `legacy_bullet_marker` rejection. Only
   the bare, one-character token changes meaning.
5. Like `@route`, the marker may sit on the parent line **or** on an authored child line
   of the same item; it configures the whole item either way, and is stripped from the
   line it was typed on.
6. A line whose only remaining token is `#` fails with `missing_text_error()`
   (`"task text is required; pass TEXT or pipe it on stdin"`), so `bob capture '#'` is a
   usage error rather than a capture of the literal text `#`.

### Composition

| Marker                                                        | With `#`     | Rationale                                                                                                                                                |
| ------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `%`, `%<N>`, `%<header>`, `--clip[=H]`                        | **allowed**  | Clipboard children render beneath the note bullet exactly as they do for `@route+<id>` sub-bullet capture. Falls out of the existing machinery for free. |
| `s:<N>`                                                       | **rejected** | A Pomodoro note is a plain bullet, not a task; `[scheduled::...]` is meaningless on it.                                                                  |
| `p:<N>`                                                       | **rejected** | Same reason, plus `p:<N>` also writes a Schedule Log.                                                                                                    |
| `@route`, `@route#Sec`, `@route:id`, `@route^id`, `@route+id` | **rejected** | Two conflicting destinations.                                                                                                                            |
| `--route` / `--section` / `--task` / `--task-ref`             | **rejected** | Same; these force a destination.                                                                                                                         |

Because `#` claims the route/mode slot, the _cross-line_ conflict (`parent @work` +
child `- note #`) is already caught by `AggregateMarkers::absorb`. Update that slot's
label from `"route/mode marker (@route)"` to `"route/mode marker (@route or #)"`.

The _same-line_ conflict needs an explicit check, because `extract_terminal_markers`
will happily pick `@work` as the route and leave `#` sitting in the body. Both orders
must fail:

- `note # @work` — `#` is not the final token, but the line resolves a route.
- `note @work #` — `#` is the final token, but a route marker remains in the leading
  (parent line only) or trailing position of the rest.

Error messages (all `CaptureError::usage`, exit 2):

- route conflict:
  `"the '#' Pomodoro-note marker cannot be combined with an @route marker"`
- forced-route conflict:
  `"the '#' Pomodoro-note marker cannot be combined with --route"`
- schedule conflict: `"the '#' Pomodoro-note marker cannot be combined with 's:<N>'"`
- priority conflict: `"the '#' Pomodoro-note marker cannot be combined with 'p:<N>'"`

Note on the forced-route case: `--route` sets `detect_route = false`, which makes
`resolve_line` return early before route detection. Today a trailing `#` under `--route`
still fails, via `reject_legacy_bullet_markers`'s first branch. Keep it failing, but
with the dedicated message above instead of the legacy one. (`--section`, `--task`, and
`--task-ref` all already require `--route`, so naming `--route` in the message is
sufficient and stays accurate.)

## Implementation

### 1. `src/native/capture_language.rs` — grammar

- Add `CaptureKind::PomodoroNote`.
- Change `RouteToken.route` from `String` to `Option<String>` so the new kind can occupy
  the route/mode slot without inventing a fake route name. Update the four `RouteToken`
  constructors and the two consumers in `parse_capture_item`
  (`match aggregate.route { Some(token) => (token.route, token.kind), None => (None, CaptureKind::Task) }`).
- Add `fn is_pomodoro_note_marker(token: &str) -> bool { token == "#" }`.
- `is_route_marker`: return true for the bare `#` token, so `extract_terminal_markers`
  treats it as the route position and `%`/`s:`/`p:` compose around it in either order.
- `reject_legacy_bullet_markers`: exempt the bare `#` token from branch (a). Branch (b)
  (`#... @route`) is unchanged for non-bare `#...`; the bare-`#` + route case is handled
  by the new conflict check below, which must run first so the user gets the precise
  message.
- `resolve_line`:
  - When `detect_route` is false and the final token is `#`, return the forced-route
    conflict error.
  - When the final token is `#`: pop it; if nothing remains, return
    `missing_text_error()`; if the first token (parent line only) or the new final token
    `is_route_marker`, return the route conflict error; otherwise return
    `LineOutcome { body, markers, route: Some(RouteToken { route: None, kind: CaptureKind::PomodoroNote }) }`.
  - When a route was resolved by the leading or trailing branch and any remaining body
    token is the bare `#`, return the route conflict error.
- `parse_capture_item`: after the aggregate is folded, if the resolved kind is
  `PomodoroNote` and `aggregate.scheduled_offset` or `aggregate.priority_level` is set,
  return the schedule/priority conflict error. Doing it here (rather than per line)
  covers markers spread across lines.
- Update `duplicate_marker_error` call site label as described above.

### 2. `src/native/capture_language.rs` — editor view

- Add `EditorMode::PomodoroNote` with label `"pomodoro_note"`.
- Add `SpanKind::PomodoroNote` with label `"pomodoro_note"`, emitted for the `#` token's
  byte span.
- `needs` is empty for this mode; `route`, `section`, and `block_id` stay `null`. There
  is no partially-typed form of a one-character marker, so no `Incomplete` handling is
  required.
- `legacy_bullet_marker_diagnostic` must not fire for the bare `#`. Since it delegates
  to `reject_legacy_bullet_markers`, exempting the token there is enough — but add a
  test pinning it.
- Add editor diagnostics so a live editor keeps a usable parse instead of a hard
  failure, mirroring the four execution errors under one stable code
  `pomodoro_note_conflict` (message = the corresponding execution message, range = the
  offending token's span).

### 3. `src/native/capture.rs` — execution

- Refactor `insert_pomodoro_block_link` into
  `fn insert_pomodoro_child_block(contents: &str, block: &str) -> Result<(String, Placement, usize, String), CaptureError>`
  returning the updated contents, placement, the selected entry's 0-based line index,
  and its ledger task text. It must indent **every** line of a multi-line `block` with
  the computed child indentation (copy the pattern from `plan_sub_bullet_capture`).
  Reimplement `insert_pomodoro_block_link` as a thin wrapper that passes
  `format!("- {block_link}")`, so the `@route:<id>` path keeps byte-identical behaviour
  and its existing tests keep passing.
- Add `fn plan_pomodoro_note_capture(planner, day_file, capture_block)`:
  - fail with
    `CaptureError::io(format!("Bob daily note does not exist: {}", day_file.display()))`
    when the file is missing (same message the `@route:<id>` path uses),
  - read through `planner.read_existing(&day_file)` so batched items see earlier planned
    edits to the same daily note,
  - call `insert_pomodoro_child_block`,
  - `planner.stage(&day_file, updated)`,
  - return a `CaptureWritePlan` carrying a new
    `pomodoro_note: Option<PomodoroNoteDetails { day_file: String, pomodoro_line: usize, pomodoro_text: String }>`
    field.
- `plan_capture_item`:
  - `capture_line` for `PomodoroNote` =
    `format_sub_bullet_line(&parsed.body, None, None)` → `- <body>`. Priority and
    scheduled are impossible by grammar, so pass `None` explicitly rather than threading
    them.
  - Resolve the target before `child_indent_unit` is called, branching on the kind: for
    `PomodoroNote` the target is `pomodoro::day_file_for(&request.bob_dir)`, and
    `relative_target` is that path stripped of `bob_dir` when it is inside the vault,
    else the absolute path (`BOB_DAY_FILE` may point outside the vault). Every other
    kind keeps `relative_target(parsed.route.as_deref())` exactly as today. This makes
    the authored-children and clipboard-children indentation derive from the daily note,
    which is correct.
  - Dispatch `CaptureKind::PomodoroNote` to `plan_pomodoro_note_capture`.
  - Result fields: `kind: "pomodoro_note"`, `routed: false`, `route: null`,
    `route_label: ""`, `target`/`relative_target` = the daily note, `day_file` = the
    daily note (parity with `pomodoro_task`),
    `block_link`/`pomodoro_link_placement`/`block_id` = `null`, `parent_line` = the
    1-based ledger line, `parent_text` = the ledger entry's task text,
    `parent_status_symbol`/`parent_status_name` = `null` (the ledger checkbox is not an
    Obsidian task and has no configured status name).
- `capture_kind_label`: add the `pomodoro_note` arm.
- `print_human_item_success`: the `under ...` line currently requires both
  `parent_status_symbol` and `parent_text`. Relax it so `parent_text` alone prints
  `  under <parent_text>`; keep the styled `[<sym>]` prefix when the symbol is present.
  This is how the user learns _which_ Pomodoro received the note. Keep colour usage
  consistent with the surrounding printer.
- `plan_capture_to_target` has a defensive `SubBullet` invariant arm; add the matching
  `PomodoroNote` arm rather than letting the match go non-exhaustive silently.

### 4. Help text and docs

Per `sase/memory/cli_rules.md`: `-h|--help` output must stay clear, complete,
consistent, and scannable, and listed subcommands/options stay alphabetical. No new
options are added here, so ordering is unaffected.

- `bob capture` `long_about`: add a paragraph for the `#` marker immediately after the
  `@<route>+<block-id>` paragraph and before the `#<section-prefix>` paragraph. State:
  bare trailing `#`; renders `- <body>` with no `[created::]`, no `#task`, no block ID;
  appends at the end of the selected Pomodoro's child block in the daily note
  (`BOB_DAY_FILE` or `<bob-dir>/YYYY/YYYYMMDD.md`); selects the single open timed
  Pomodoro, otherwise the first open one; composes with `%`/`--clip`; is rejected
  alongside `s:<N>`, `p:<N>`, `@route`, and `--route`.
- Amend the existing sentence "Standalone terminal '#...' markers are no longer accepted
  and fail with a usage error." to make clear that a bare `#` is now the Pomodoro-note
  marker while `#<prefix>` standalone stays rejected.
- `bob capture` `after_help` Examples: add
  `bob capture remembered to bump the timeout #` and
  `bob capture paste the failing output % #`.
- `bob capture-parse` `long_about` and its `Modes:` line: add `pomodoro_note`.
- `README.md`: add a subsection to the capture documentation (near the
  `@route+<block-id>` and section-bullet material, around lines 214-330) covering the
  marker, the selection rule, the resulting line, and the composition table.

## Testing

### `src/native/capture_language.rs` unit tests

- bare trailing `#` resolves `CaptureKind::PomodoroNote`, route `None`, and the marker
  is stripped from the body.
- `#` on an authored child line configures the item.
- composes with `%` in both terminal orders (`note % #`, `note # %`).
- `note # s:1` / `note p:2 #` → schedule/priority conflict errors; also with the markers
  on different lines of one item.
- `note # @work`, `note @work #`, `@work note #` → route conflict error.
- `-r`/`--route` forced route + trailing `#` → forced-route conflict error.
- `#Ideas` still yields `legacy_marker_error` / the `legacy_bullet_marker` diagnostic;
  update `editor_reports_legacy_bullet_markers_without_failing` (~line 2945) only if its
  cases are affected — they use `#bar`, so they should not be.
- `#` mid-body stays literal; leading `# foo` stays literal.
- `#` as the only token → `missing_text_error`.
- editor parse: mode `pomodoro_note`, one `pomodoro_note` span with the right byte
  range, empty `needs`, no diagnostics; and one `pomodoro_note_conflict` diagnostic
  case.

### `tests/cli.rs` integration tests

- **Existing test must be rewritten**: the case around line 6825 asserts
  `bob capture jot this #` exits 2 with `"must be appended to an @route token"`. Bare
  `#` is now valid, so convert it into a positive test (or move the negative assertion
  to `#Ideas`) — do not delete the coverage.
- Current Pomodoro: daily note with `- [ ] Focus (0900-0930)` plus an existing child →
  the note lands at the end of that entry's child block with matching indentation, and
  the routed-note/inbox files are untouched.
- Next future Pomodoro: `- [x] Done (0900-0930)` followed by `- [ ] Next ()` → the note
  lands under `Next`.
- Multiple open timed entries → exit 1, `"multiple open timed Pomodoros"`, daily note
  byte-identical.
- No `## Pomodoros` section → `"has no Pomodoros section"`; no open entry →
  `"no eligible open Pomodoro"`; missing daily note → `"Bob daily note does not exist"`.
  All leave every file unchanged.
- `--dry-run` reports the plan and writes nothing.
- JSON shape: `kind == "pomodoro_note"`, `routed == false`, `route` absent, `day_file`
  set, `parent_line`/`parent_text` set, `block_link` and `pomodoro_link_placement`
  absent.
- Human output includes the `under <ledger text>` line and the rendered `- <body>`.
- Batch: a two-item draft where both items end in `#` puts both notes under the same
  Pomodoro in source order (proves the in-memory planner composes).
- Authored children: `Parent #\n- child` renders the child one further indentation unit
  beneath the note.
- `%` composition writes clipboard children beneath the note (drive the clipboard with
  `BOB_CLIPBOARD_CMD`, as the existing clip tests do).
- CRLF daily note keeps CRLF line endings (there are existing `\r\n` ledger tests around
  line 2001 to copy from).
- `bob capture-complete` with the cursor on a trailing bare `#` returns a successful
  empty result (no new completion context is introduced).
- `bob capture-parse -f json` on `note #` reports mode `pomodoro_note` and the
  `pomodoro_note` span.

### Gate

`just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
`cargo test`) must pass.

## Judgment calls to confirm at review

1. **`s:<N>` / `p:<N>` are rejected rather than silently rendered.** A Pomodoro note is
   a plain bullet, so a `[scheduled::]` or `[priority::]` property on it is inert.
   Erroring is the honest behaviour, but it is a choice — if the project owner prefers,
   the alternative is to render them like the `@route+<id>` sub-bullet path does.
2. **`%` clipboard capture is allowed.** It composes for free and "capture what I just
   copied onto this Pomodoro" is a plausible use.
3. **The note is appended at the end of the Pomodoro's child block**, after any existing
   Task Links — identical to where `@route:<id>` puts its block link.
4. **The daily note is reported as the capture target** (`routed: false`,
   `target`/`relative_target` = the daily note) rather than as a side effect of an inbox
   capture, because nothing is written anywhere else.
5. **No new marker is added for "leading `#`".** The request specified end-of-line only.

## Out of scope

- `bob-mac-capture`: it delegates grammar, completion, and live preview to
  `bob capture-parse` / `bob capture-complete` over JSON, so it picks the new mode up
  without a code change. The new `mode` and span `kind` values are additive, so
  `SCHEMA_VERSION` stays `1`. Flag the new enum values to that repo's owner rather than
  editing it here.
- chezmoi's Hammerspoon capture grammar (`tests/hammerspoon/task_capture_spec.lua`, the
  source of the `lua_*` parity tests in `capture_language.rs`) never knew about `#`, so
  the existing parity tests stay valid. Teaching the Lua UI to highlight the new marker
  is separate follow-up work in that repo.
- `bob capture-complete` gains no new completion context; there is nothing to complete
  about a one-character marker.
