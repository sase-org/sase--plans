---
tier: tale
size: small
title: Route bare `#` Pomodoro notes to the last completed Pomodoro
goal:
  "When `bob capture <text> #` runs with no current Pomodoro, the note lands under the
  last completed Pomodoro in the daily note instead of the first future one, falling
  back to the first future Pomodoro only when the ledger has no completed entry. The
  `@<route>:<block-id>` ledger-link path keeps its existing open-only selection."
proposed_by: bbugyi200.athena.06v
create_time: 2026-09-09 19:59:55
status: wip
---

# Plan: route bare `#` Pomodoro notes to the last completed Pomodoro

## Goal

Change the ledger-entry selection used by the bare trailing `#` Pomodoro-note marker
(added in `f171a7e`, earlier today) so that it picks, in order:

1. the **current** Pomodoro (the single open entry with a recognized time range),
2. otherwise the **last completed** Pomodoro in the daily note,
3. otherwise the **first future** Pomodoro,
4. otherwise fail without writing anything.

Today step 2 does not exist: with no current Pomodoro the note goes straight to the
first open (future) entry, and a ledger with only completed entries fails outright.

The `@<route>:<block-id>` path, which inserts a `[[route#^id]]` Task Link into the
ledger, keeps today's open-only rule (current, else first future). A link to a _newly
created, not-yet-done_ task belongs on a Pomodoro that is still open; a note is a record
of work that already happened, which is why it should attach to the Pomodoro that just
finished.

## Background: what exists today

Verified against the tree at authoring time (`e04d1a1`). Line numbers drift — re-check
them.

### The one selection helper both paths share

`src/native/capture.rs:1809` `insert_pomodoro_child_block(contents, block)` is the
single implementation. It returns
`(updated_contents, Placement, selected_line_index, selected_ledger_text)` and is called
from exactly two places:

- `src/native/capture.rs:1794` `insert_pomodoro_block_link(contents, block_link)` — a
  thin wrapper that passes `format!("- {block_link}")`. Called by
  `plan_capture_with_pomodoro_link` (`src/native/capture.rs:1238`) for `@route:<id>`.
- `src/native/capture.rs:1289`, inside `plan_pomodoro_note_capture`
  (`src/native/capture.rs:1275`) for `CaptureKind::PomodoroNote` (the bare `#`).

Its body (`src/native/capture.rs:1812-1885`):

1. resolves the `## Pomodoros` section with `pomodoro::pomodoros_section_range`
   (`src/native/pomodoro.rs:280`);
2. walks the section skipping fenced lines (`markdown::fenced_lines`) and indented lines
   (`is_indented_line`, `src/native/capture.rs:2561`), so nested checkboxes are never
   candidates;
3. collects `open` entries via `pomodoro::open_ledger_task`
   (`src/native/pomodoro.rs:338`, rejects both `[x]` and `[-]`) and, among those,
   `timed` entries via `pomodoro::task_time_range` (`src/native/pomodoro.rs:369`);
4. errors `"Bob daily note has multiple open timed Pomodoros"` when `timed.len() > 1`;
5. selects `timed.first()`, else `open.first()`, else errors
   `"Bob daily note has no eligible open Pomodoro"`;
6. re-parses the selected line with `open_ledger_task` (with an `expect`) to produce the
   returned ledger text;
7. computes the insertion offset with `task_block_end` (`src/native/capture.rs:2494`,
   end of the entry's whole child block, blank lines included when more indented content
   follows), derives the child indentation from the entry's existing children, then from
   a nearby ledger child (`nearby_child_bullet_indentation`), then `"  "`, indents
   **every** line of a multi-line `block`, and inserts it preserving the document's line
   endings.

Steps 1-4 and 6-7 are all correct for the new behavior. Only step 5 (and the wording of
one error message) changes, and it must change for the `#` path only.

### Ledger vocabulary

From `sase glossary read Pomodoro`: a Pomodoro is a checkbox item in the daily note's
`## Pomodoros` section. **Past** Pomodoros have a timespan and are checked. **Current**
Pomodoros have a timespan and are open. **Future** Pomodoros have an empty `()` and are
open. So the three-step rule above maps onto existing helpers with no new parsing:

- current → the single `open` + `timed` entry,
- last completed → the last entry matched by `pomodoro::completed_ledger_task`
  (`src/native/pomodoro.rs:356`, `[x]` only), already `pub(crate)` and used by the
  `bob_pomodoro` status path,
- first future → `open.first()` in the branch where no `timed` entry exists (in that
  branch every open entry is untimed, i.e. future).

### What already returns the selection to the caller

`PomodoroNoteDetails { day_file, pomodoro_line, pomodoro_text }`
(`src/native/capture.rs:1192`) feeds the result's `parent_line` (1-based) and
`parent_text` (`src/native/capture.rs:786-791`), which the human printer renders as
`under <parent_text>` (`src/native/capture.rs:2706`). `parent_status_symbol` and
`parent_status_name` stay `null` for `pomodoro_note` results — README:541-546 documents
that deliberately, "since the ledger checkbox is not an Obsidian task". This plan keeps
that contract; see [Judgment calls](#judgment-calls-to-confirm-at-review).

### Sibling implementations that must not change

- `src/native/task_status_hooks.rs:576-606` has its own ledger scan (`scan_pomodoros`)
  for the `[/]` in-progress hook, including the same `"multiple open timed Pomodoros"`
  invariant. It is unrelated to capture and stays untouched
  (`docs/task-status-hooks.md:446` documents it).
- `bob-mac-capture` was grepped for `pomodoro_note` / `next future` / `first open` and
  has no reference to the selection rule; it consumes `bob capture-parse` /
  `bob capture-complete`, neither of which reports ledger selection. No cross-repo
  change is needed.

## Specification

For a bare trailing `#` capture, within the daily note's `## Pomodoros` section,
considering only non-fenced, non-indented (top-level) checkbox entries:

| Ledger state (no current Pomodoro unless stated)        | Selected entry                                      |
| ------------------------------------------------------- | --------------------------------------------------- |
| Exactly one open timed entry                            | that entry (**current**) — unchanged                |
| More than one open timed entry                          | error, `multiple open timed Pomodoros` — unchanged  |
| No open timed entry, one or more `[x]` entries          | the **last** `[x]` entry in document order          |
| No open timed entry, no `[x]`, one or more open entries | the **first** open entry (first future) — unchanged |
| No `[x]` and no open entry (empty, or only `[-]`)       | error, `no eligible Pomodoro`                       |
| No `## Pomodoros` section                               | error, `no Pomodoros section` — unchanged           |
| Daily note missing                                      | error, `Bob daily note does not exist` — unchanged  |

Notes on the rule:

- "Last completed" is **document order**, not latest end time. The ledger is written
  top-to-bottom as the day progresses, and `pomodoro::latest_ledger_pomodoro`
  (`src/native/pomodoro.rs:243`) already treats document order as chronological order.
  No time parsing or sorting is introduced.
- A `[x]` entry qualifies whether or not it carries a `(HH:MM-HH:MM)` range. The
  checkbox is the authoritative "completed" signal, and `completed_ledger_task` does not
  require a range.
- Cancelled `[-]` entries qualify as neither open nor completed, so they are skipped
  entirely — same as today.
- The timed-ambiguity error still wins over every fallback: two open timed entries mean
  "which Pomodoro is current?" is unanswerable, and guessing would silently attach notes
  to the wrong session.
- Nothing else about the `#` marker changes: grammar, conflicts, rendering (`- <body>`,
  no `#task`, no `[created::]`, no block ID), insertion point (end of the selected
  entry's child block), indentation reuse, line-ending preservation, `--dry-run`,
  batching, `%`/`--clip` composition, and the JSON field set are all as-is.

### Error message

The `#` path's "nothing to attach to" message becomes
`"Bob daily note has no eligible Pomodoro"` (dropping `open`), because a completed entry
is now eligible and the old wording would be wrong. The `@route:<id>` path keeps
`"Bob daily note has no eligible open Pomodoro"` verbatim, so its tests
(`src/native/capture.rs:3966`, `tests/cli.rs:4228`) and README:376 stay accurate.

## Implementation

### 1. `src/native/capture.rs` — selection policy

- Add a small copy `enum` next to `insert_pomodoro_child_block`, e.g.:

  ```rust
  /// Which ledger entry a daily-note child insertion attaches to.
  #[derive(Debug, Clone, Copy, PartialEq, Eq)]
  enum PomodoroSelection {
      /// Current Pomodoro, else the first future one. Used by `@<route>:<id>`
      /// ledger links, which must attach to a Pomodoro that is still open.
      CurrentOrFuture,
      /// Current Pomodoro, else the last completed one, else the first future
      /// one. Used by the bare `#` Pomodoro-note marker.
      CurrentOrLastCompleted,
  }
  ```

  Give it a private helper for the empty-ledger message (`CurrentOrFuture` →
  `"Bob daily note has no eligible open Pomodoro"`, `CurrentOrLastCompleted` →
  `"Bob daily note has no eligible Pomodoro"`) so the two strings live next to the
  variants that produce them.

- Change `insert_pomodoro_child_block` to take `selection: PomodoroSelection` as its
  final parameter and update its doc comment to describe both policies.

- In its scan loop, keep collecting `open` and `timed` exactly as today, and add a
  `completed` bucket in the same pass:

  ```rust
  if let Some(task) = pomodoro::open_ledger_task(line) {
      open.push((index, task));
      if pomodoro::task_time_range(task).is_some() {
          timed.push((index, task));
      }
  } else if let Some(task) = pomodoro::completed_ledger_task(line) {
      completed.push((index, task));
  }
  ```

  Keep `timed` populated **only** from open entries, so a completed entry with a
  timespan can never trip the ambiguity error.

- Keep the `timed.len() > 1` check ahead of selection, then select:

  ```rust
  let (selected, selected_text) = match selection {
      PomodoroSelection::CurrentOrFuture => timed.first().or(open.first()),
      PomodoroSelection::CurrentOrLastCompleted => {
          timed.first().or(completed.last()).or(open.first())
      }
  }
  .copied()
  .ok_or_else(|| CaptureError::io(selection.no_entry_message()))?;
  ```

  Carrying the parsed task text in the buckets lets the returned ledger text come from
  whichever parser matched the line and removes the existing re-parse plus `expect` at
  `src/native/capture.rs:1852`. If the buckets are kept as bare indices instead, the
  text must be resolved as
  `open_ledger_task(line).or_else(|| completed_ledger_task(line))` — never
  `open_ledger_task` alone, which would panic on a completed selection.

- Everything after selection (`task_block_end`, indentation, multi-line indent,
  `insertion_text_preserving_line_endings`, `Placement`) is untouched.

- `insert_pomodoro_block_link` passes `PomodoroSelection::CurrentOrFuture`;
  `plan_pomodoro_note_capture` passes `PomodoroSelection::CurrentOrLastCompleted`.

Nothing else in `plan_pomodoro_note_capture`, `CaptureWritePlan`, `PomodoroNoteDetails`,
`CaptureItemResult`, or the human printer changes: `parent_line` and `parent_text`
already describe whichever entry was selected.

### 2. `src/native/capture.rs` — `bob capture` help text

Per `sase/memory/cli_rules.md`, `-h|--help` must stay clear, complete, and scannable; no
options are added or reordered here.

- In the bare-`#` paragraph of `long_about` (`src/native/capture.rs:165-172`), replace
  "Capture prefers the single open timed entry in its Pomodoros section and otherwise
  uses the first open entry" with the three-step rule: the single open timed entry,
  otherwise the last completed entry, otherwise the first open entry. Keep the sentence
  about appending at the end of that entry's child block and the composition sentence.
- Leave the `@<route>:<block-id>` paragraph (`src/native/capture.rs:139-147`) exactly as
  it is — the wording there is still true for that path.
- No `after_help` example changes: `bob capture remembered to bump the timeout #`
  already illustrates the marker, and selection is not visible in an example line.

### 3. `README.md`

- In the bare-`#` section (README:418-425), rewrite the selection sentences to the
  three-step rule, and state that a ledger with only completed entries now succeeds
  (attaching to the last one) instead of failing. Keep the surrounding text about the
  daily-note file selection, the rendered bullet, the insertion point, and the
  composition table.
- The sentence "The daily note is selected the same way as `@<route>:<block-id>`
  captures" (README:416-418) must keep referring to _which file_ is used. Make sure the
  rewritten paragraph cannot be read as claiming the two paths share an entry-selection
  rule; say plainly that unlike `@<route>:<block-id>`, a Pomodoro note may attach to a
  completed entry.
- Leave the `@<route>:<block-id>` selection paragraph (README:366-377) unchanged.
- No change to the `pomodoro_note` JSON documentation (README:544-551): the field set,
  including the omitted `parent_status_symbol`/`parent_status_name`, is unchanged.

## Testing

### `src/native/capture.rs` unit tests

The existing `pomodoro_link_*` tests (`src/native/capture.rs:3910`, `:3940`, `:3966`,
`:3982`) must keep passing untouched apart from the added argument in any direct calls —
they are the regression net proving the `@route:<id>` path did not move. Add note-policy
tests calling
`insert_pomodoro_child_block(..., PomodoroSelection::CurrentOrLastCompleted)` directly:

- **Current wins over a completed entry**: a ledger with `- [x] Done (0900-0930)` above
  an open timed `- [ ] Focus (1000-1030)` selects `Focus`; the same fixture under
  `CurrentOrFuture` also selects `Focus` (both policies agree while a Pomodoro runs).
- **Last completed wins over a future entry**: `- [x] First (0900-0930)`,
  `- [x] Second (1000-1030)`, `- [ ] Next ()` selects `Second`, and the inserted child
  lands between `Second` and `Next` with `Placement::Inserted`; assert the full updated
  document, not just the index.
- **Completed entry with existing children**: the child is appended after the completed
  entry's last existing child, reusing that child's indentation.
- **Policy divergence on one fixture**: the same completed-plus-future ledger selects
  the completed entry under `CurrentOrLastCompleted` and the future entry under
  `CurrentOrFuture`.
- **First future when nothing is completed**: `- [ ] Next ()`, `- [ ] Later ()` selects
  `Next` under both policies.
- **Returned ledger text comes from the completed parser**: the third tuple element is
  the completed entry's 0-based line index and the fourth is its text without the
  `- [x] ` prefix (this is what would panic if the `expect` on `open_ledger_task` were
  left in place).
- **Cancelled and nested entries are not completed entries**: a ledger of
  `- [-] Cancelled ()` plus `  - [x] Nested (0900-0930)` fails with
  `"no eligible Pomodoro"`; an empty `## Pomodoros` section fails the same way.
- **Fenced lookalikes are ignored**: a `[x]` entry inside a fenced block is not
  selectable (mirror `pomodoro_section_scan_ignores_fenced_lookalikes`).
- **Ambiguity still wins**: two open timed entries plus a completed entry fail with
  `"multiple open timed Pomodoros"` under `CurrentOrLastCompleted`.
- **CRLF preserved** when inserting under a completed entry.

### `tests/cli.rs` integration tests

- **Rewrite `capture_bare_terminal_marker_falls_back_to_next_future_pomodoro`**
  (`tests/cli.rs:6942`). Its fixture is exactly the behavior change:
  `## Pomodoros\n\n- [x] Done (0900-0930)\n- [ ] Next ()\n`. Rename it to
  `..._falls_back_to_the_last_completed_pomodoro`, assert
  `parent_text == "Done (0900-0930)"`, `parent_line == 3`, and a resulting document with
  `  - note this` between the `Done` and `Next` lines. Do not delete the coverage.
- **Add** `capture_bare_terminal_marker_falls_back_to_the_first_future_pomodoro`: a
  ledger of `- [ ] Next ()\n- [ ] Later ()\n` still lands under `Next`
  (`parent_text == "Next ()"`), proving the third step survives.
- **Add** a two-completed-entries case (either as a JSON assertion or folded into the
  rewritten test) proving the **last** `[x]` entry wins, not the first.
- **Update
  `capture_bare_terminal_marker_preflight_failures_leave_daily_note_untouched`**
  (`tests/cli.rs:7014`). Its `"no-open"` case uses
  `## Pomodoros\n\n- [x] Done (0900-0930)\n`, which now succeeds. Replace it with a
  genuinely ineligible ledger — `## Pomodoros\n\n- [-] Cancelled ()\n` and/or an empty
  `## Pomodoros` section — and expect `"Bob daily note has no eligible Pomodoro"`. Keep
  the `missing-section` and `missing-daily-note` cases as they are.
- **Unchanged, must keep passing**:
  `capture_bare_terminal_marker_writes_pomodoro_note_under_current_pomodoro` (`:6858`),
  `..._human_output_shows_the_ledger_entry` (`:6916`),
  `..._multiple_open_timed_pomodoros_is_io_error` (`:6981`), `..._dry_run_...`
  (`:7078`), `..._batch_items_land_under_the_same_pomodoro_in_order` (`:7113`),
  `..._authored_children_nest_one_level_deeper` (`:7144`),
  `..._clipboard_marker_writes_children_beneath_note` (`:7177`),
  `..._keeps_crlf_line_endings` (`:7217`) — all use a single open timed entry.
- **Unchanged, must keep passing**:
  `capture_pomodoro_preflight_failures_leave_both_notes_untouched`
  (`tests/cli.rs:4228`), whose `"no-open"` case (`- [x] (1330-1400) Done` →
  `"no eligible open Pomodoro"`) is the proof that `@route:<id>` did not inherit the new
  fallback. If the message helper is factored wrongly, this test is what catches it.
- **Optional but cheap**: extend the batch test with a completed-only ledger to confirm
  two `#` items in one batch stack under the same completed entry in source order. The
  inserted children are indented, so they are skipped by the top-level scan and the
  second item re-selects the same entry.

### Gate

`just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
`cargo test`) must pass.

## Judgment calls to confirm at review

1. **`@<route>:<block-id>` keeps the open-only rule.** The request named the `#` syntax
   only, and a Task Link to a brand-new task on a _finished_ Pomodoro would misreport
   when the work happened. This is why the policy is a parameter rather than a rewrite
   of the shared helper.
2. **Document order decides "last completed"**, not the largest end time. Simpler,
   allocation-free, and consistent with `pomodoro::latest_ledger_pomodoro`. A ledger
   whose entries are out of chronological order would pick the bottom-most `[x]`.
3. **Cancelled `[-]` entries are never selected.** They record work that did not happen.
4. **A `[x]` entry with no time range still counts as completed.**
5. **Two open timed entries remain a hard error** rather than falling back to the last
   completed entry.
6. **The JSON contract is unchanged.** No `pomodoro_selection` field and no
   `parent_status_symbol` for `pomodoro_note` results, per the deliberate decision
   recorded at README:544-551. The human `under <parent_text>` line already names the
   chosen entry, and a completed Pomodoro shows a real timespan where a future one shows
   `()`. If distinguishing _current_ from _last completed_ at a glance turns out to
   matter, that is a separate, additive change.

## Out of scope

- `src/native/task_status_hooks.rs` ledger selection and `docs/task-status-hooks.md`.
- `bob-mac-capture` and the Hammerspoon capture UI: no grammar, mode, span, or JSON
  field changes, and neither documents the ledger-selection rule.
- `src/native/capture_language.rs`, `capture_parse.rs`, and `capture_complete.rs`: the
  `#` grammar, the `pomodoro_note` mode and span, and completion contexts are untouched.
- Retroactively moving notes already captured under a future Pomodoro.
