---
tier: tale
title: Mark future-scheduled Obsidian tasks Blocked
goal:
  Future-scheduled inline Obsidian tasks are reconciled to `[?]` by the property picker
  and CLI and appear in blocked.md.
create_time: 2026-09-09 19:53:14
status: wip
---

# Mark future-scheduled Obsidian tasks Blocked

## Goal

Treat a valid task-level Dataview schedule later than the effective local day as a
derived blocking condition. A recognized open Obsidian task with
`[scheduled:: YYYY-MM-DD]` (or the equivalent parenthesized Tasks metadata form) must
use the custom Blocked status `[?]` when `YYYY-MM-DD` is tomorrow or later. The
Ctrl+Shift+P property picker should apply that status immediately when it writes a
future inline `scheduled` value, and `bob task-status-hooks` should authoritatively
reconcile the same rule across the vault.

Keep project-level `scheduled: YYYY-MM-DD` frontmatter out of this task-level rule. The
navigation plugin deliberately maps `scheduled` on a project's `^prj` lifecycle task to
project frontmatter, removes stale inline fields, and manages `#hide` visibility for the
project; that existing project scheduling contract should remain unchanged.

## Existing behavior and constraints

- `bob task-status-hooks` already derives `[?]` from open `[dependsOn:: ...]` targets,
  gives that state precedence over Pomodoro ranks and stale-rank cleanup, validates the
  installed Blocked status before writing, and recovers a no-longer blocked task to its
  final Pomodoro-derived rank or Ready.
- Its task scanner already recognizes valid trailing Tasks/Dataview date metadata
  syntactically, but it does not retain `scheduled` on `TaskLine` or use it as a
  blocking reason. The command's selected daily-note date is already the effective
  anchor used for deterministic `BOB_DAY_FILE`/`BOB_NOW` behavior.
- Bob Navigation Hotkeys already has local-date helpers, calendar-date validation, an
  open-task status set, a helper that rewrites supported open statuses to `[?]`, and
  atomic single-line/counted property write paths. Counted property sessions include
  terminal tasks, so status reconciliation must change only supported open `#task`
  checkboxes while still applying the selected property to every requested task.
- `~/bob/blocked.md` currently filters out future schedules in `TQ_extra_instructions`,
  and its `is blocked` Tasks predicate means “has an open dependency”; it does not
  select a `[?]` task that is blocked solely by its scheduled date. Both query details
  must change for scheduled-only Blocked tasks to appear.
- The linked `bob-plugins` repository is the source of truth. Any implementation change
  there must be followed by `bob plugins sync` so the managed plugin files are deployed
  to the vault.

## Implementation

### 1. Extend `bob task-status-hooks` from dependency blocking to combined derived blocking

In `src/native/task_status_hooks.rs`:

1. Retain a calendar-valid scheduled date on parsed task metadata/`TaskLine`. Continue
   accepting the scanner's supported square-bracket and parenthesized Tasks metadata
   forms, whitespace normalization, trailing metadata order, and
   global-filter/fence/directory exclusions. Do not treat malformed shapes or
   nonexistent calendar dates as future schedules.
2. Compare each scheduled date with the command's effective daily anchor.
   `scheduled > anchor` is future; today and earlier are not. This makes “tomorrow or
   later” deterministic under `BOB_DAY_FILE` and `BOB_NOW` and avoids
   time-of-day/time-zone boundary comparisons.
3. Model the task's blocking state as the union of:
   - at least one recognized open `dependsOn` target; and
   - a valid future scheduled date. Feed that combined state into the existing
     transition precedence. Ready, Next, and In Progress become Blocked when either
     reason exists; an already Blocked task stays Blocked while either reason remains;
     terminal, non-task, and unrecognized/custom statuses remain untouched. When neither
     reason remains, recover `[?]` to the final Pomodoro-derived Next/In-Progress rank
     or Ready exactly as the dependency-only path does today.
4. Generalize Blocked change reporting without breaking existing dependency detail: keep
   the current `open_dependency_ids`/`unresolved_dependency_ids` JSON fields and add the
   future scheduled date (nullable) to each Blocked/unblocked record. Human output
   should identify `scheduled: YYYY-MM-DD`, open dependency IDs, or both, and
   future-scheduled changes must count in the existing `marked_blocked`/summary totals.
   Continue validating the installed Blocked status before any planned Blocked or
   unblock write.
5. Update the command help, `README.md`, and `docs/task-status-hooks.md` to define the
   date boundary, combined-reason precedence, recovery behavior, dry-run/JSON reporting,
   and the project-frontmatter exclusion.

Add focused unit and CLI integration coverage in `src/native/task_status_hooks.rs` and
`tests/cli.rs` for:

- metadata parsing with bracket/parenthesized fields, flexible spaces/order, invalid
  shapes, and impossible dates;
- yesterday/today versus tomorrow/later at a fixed daily anchor;
- Ready, Next, and In-Progress promotion to `[?]`, already-Blocked idempotency, and
  terminal/unknown preservation;
- scheduled-only, dependency-only, and combined blocking reasons;
- removal or maturation of a schedule while another dependency still blocks;
- recovery to Ready, Next, or In Progress only after every blocking reason is gone;
- dry-run non-mutation, apply output/JSON reason fields, Blocked-status validation, and
  second-run idempotency.

### 2. Make Ctrl+Shift+P block inline tasks in the same guarded edit

In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`:

1. Add a small, testable local-date predicate for a valid inline `scheduled` value later
   than today, reusing the existing calendar validation and local-date helpers.
2. In both the bare single-task setter and `planCountedBulletPropertyBatch`, compose the
   property upsert with the existing open-task-to-`[?]` rewrite before committing. Apply
   it only when:
   - the selected property name is exactly `scheduled`;
   - the selected value is a valid date after the picker/write path's local today;
   - the target is an inline real `#task`, not an ordinary bullet or a
     project-frontmatter `^prj` target; and
   - the checkbox is a supported open status (`[ ]`, `[*]`, `[/]`, or already `[?]`).
     Done, canceled, unknown/non-task statuses keep their checkbox while still receiving
     the chosen property.
3. Preserve the counted path's stale-session rejection, CRLF/formatting preservation,
   caret/viewport behavior, and all-or-nothing editor transaction. A status-only change
   (the future property already matches but the task is not `[?]`) must still count as a
   changed target. Return/report a blocked-task count so notices make the additional
   reconciliation visible.
4. Do not eagerly unblock on selecting today/past or deleting `scheduled`: the picker
   cannot safely distinguish another dependency/Pomodoro-derived reason. The
   authoritative `task-status-hooks` pass performs recovery.

Expand `scripts/test-navigation-hotkeys.cjs` to cover single and counted selections at
the today/tomorrow boundary, ready/next/in-progress/already-blocked/terminal statuses,
ordinary bullets, mixed counted batches, status-only changes, stale atomic aborts, line
endings/caret behavior, and the existing mixed project-frontmatter plus inline-task
batch. Export only the minimal helper needed by the test harness. Bump the Bob
Navigation Hotkeys patch version and update its README table description to mention
future-schedule blocking.

### 3. Make `blocked.md` include both kinds of Blocked task

Edit `~/bob/blocked.md`:

1. Remove only the Query File Defaults instruction that excludes schedules after today.
   Keep the existing template, self-note, `#hide`, grouping, sorting, and layout
   filters.
2. Change the Tasks query from `is blocked` to
   `(is blocked) OR (status.name includes Blocked)`. The first branch preserves
   dependency-derived Tasks blocking even before status reconciliation; the second
   includes `[?]` tasks whose sole reason is a future schedule.
3. Update the note's prose to describe both dependency-blocked and future-scheduled
   tasks.

Update the real-vault Tasks parity ground truth in `tests/tasks_real_vault_parity.rs` to
mirror the note: do not reject future schedules, and select either dependency-blocked
tasks or tasks whose registered status name is Blocked, while retaining the other note
filters. Add/adjust deterministic Tasks query fixtures if needed so a future-scheduled
`[?]` task with no `dependsOn` is covered independently of the opt-in real-vault test.

### 4. Validate and deploy

Run targeted tests first, then the full checks:

- In `bob-cli`: targeted `cargo test` filters for task-status-hooks and Tasks query
  parity, followed by `cargo fmt --check`, `cargo clippy --all-targets --all-features`,
  and `cargo test`.
- In `bob-plugins`: `node --test scripts/test-navigation-hotkeys.cjs`, then `npm test`
  and `npm run validate`.
- Query a deterministic fixture/note through `bob query --tasks-note blocked.md` and
  assert that a future-scheduled scheduled-only `[?]` task is present, while the
  existing exclusions still hold. If the opt-in live-vault parity environment is
  available, run it with a fixed `BOB_NOW`.
- Run `bob plugins sync` after the linked plugin changes pass, then confirm Bob
  Navigation Hotkeys reports synced at the bumped version. Do not run a mutating
  live-vault `task-status-hooks` pass as part of automated validation; use `--dry-run`
  for an optional real-vault preview.

## Acceptance criteria

- Choosing tomorrow or any later valid date for inline `scheduled` with bare or counted
  Ctrl+Shift+P changes each supported open target to `[?]` in the same editor
  transaction.
- `bob task-status-hooks` marks every recognized open, task-level future schedule `[?]`,
  reports the scheduled reason, composes correctly with dependency blocking, and is
  idempotent.
- Today/past, malformed, project-frontmatter-only, terminal, non-task, and
  unknown-status cases are not newly blocked.
- A `[?]` task is recovered only when neither a future schedule nor an open dependency
  remains, preserving the final Pomodoro rank where applicable.
- `~/bob/blocked.md` renders scheduled-only `[?]` tasks as well as dependency-blocked
  tasks, including future dates.
- All bob-cli and bob-plugins checks pass, and the bumped Bob Navigation Hotkeys files
  are synced to the vault.
