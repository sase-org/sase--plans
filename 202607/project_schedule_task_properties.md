---
tier: tale
title: Replace project
goal:
  "A project note's frontmatter schedule propagates into per-task Dataview `[scheduled::
  YYYY-MM-DD]` fields and derived `[?]` Blocked checkboxes instead of `#hide` tags on
  ordinary tasks."
create_time: 2026-09-09 19:53:20
status: wip
---

# Replace project `#hide` visibility with task-level scheduled properties

## Goal

Stop using the `#hide` tag as the mechanism that suppresses ordinary tasks in a
scheduled project note. When a project note's frontmatter declares a valid
`scheduled: YYYY-MM-DD`, every **open** ordinary task in that note should instead carry
a task-level Dataview schedule `[scheduled:: YYYY-MM-DD]` equal to the project's date,
unless the task already declares a valid schedule on the same day or later. Those tasks
should then be Blocked (`[?]`) while the date is in the future and recover to their
normal rank once the date is due, removed, or superseded.

The user-visible outcome is unchanged (tasks in a future-scheduled project stay off
`dash.md`), but the reason becomes a real, inspectable task property instead of an
opaque tag, and those tasks become first-class Blocked tasks that appear in `blocked.md`
alongside dependency-blocked tasks.

## Audit findings and constraints

Read this section before implementing; one premise in the request is inaccurate.

- **`bob task-status-hooks` does not manage `#hide` at all.**
  `grep -n hide src/native/task_status_hooks.rs` returns nothing. The two components
  that add/remove `#hide` from a project note's tasks in response to frontmatter
  `scheduled:` are `bob projects sync` (`src/native/projects.rs`, `TaskVisibilityPolicy`
  / `ProjectChange::ReconcileTaskVisibility` / `task_visibility_edits`) and Bob
  Navigation Hotkeys' `Ctrl+Shift+P` project path (`planProjectScheduleVisibility` /
  `normalizeTaskHideTag` in `plugins/bob-navigation-hotkeys/main.js`). This plan changes
  those two, not `task_status_hooks.rs`.
- **The task-level half of the desired behavior already exists and is bidirectional.**
  Commits `9bb625a` and `456f9d2` (bob-cli) and `5759320` and `2c8059c` (bob-plugins)
  made a valid inline `[scheduled:: YYYY-MM-DD]` later than the effective anchor a
  derived blocking reason. `bob task-status-hooks` already marks such tasks `[?]`, keeps
  them Blocked while any reason remains, and recovers them to Ready/Next/In Progress
  once every reason is gone. The plugin already has `isFutureInlineScheduledValue`,
  `isDueInlineScheduledValue`, `blockObsidianTaskCheckboxStatus`,
  `reconcileBlockedScheduledTaskLine`, and the `buildTargetScheduledRecoveryByLine`
  vault snapshot. **This plan should compose those existing pieces, not reimplement
  blocking or recovery.**
- Today's schedule policy touches _every_ Markdown task line in the note (open, done,
  canceled, nested, ordered-list, blockquoted), adds exactly one whole-token `#hide`
  when the date is future, and removes all `#hide` tags from ordinary tasks when the
  date is due. `^prj` is special-cased through `TaskVisibilityPolicy::include_prj`: it
  is hidden with everything else when future, and unhidden when due only if it is the
  note's only task.
- The `^prj` lifecycle task has a **second, unrelated** `#hide` contract:
  `bob projects sync` removes `#hide` from an open `^prj` only when the project has no
  non-hidden open tasks and no open sub-projects, so the project surfaces on `dash.md`.
  That rule is guarded off entirely for projects with a valid frontmatter schedule
  (`if project.scheduled.is_none()` in `plan_project_sync_at`). **This plan keeps
  `#hide` as the mechanism for the `^prj` task and keeps that guard.**
- `^prj` must never carry an inline `[scheduled:: ...]` field:
  `ProjectChange::RemoveScheduled` and the plugin's
  `removeAllBulletProperties(lines[cursorLine], "scheduled")` both strip it, because
  frontmatter is the sole project schedule. Propagation must therefore exclude the
  `^prj` task.
- `#hide` has other consumers that must keep working: `dash.md` and `blocked.md` filter
  on it, Bob Navigation Hotkeys refuses to add a dependency to a `#hide` target, and
  Block ID Prompt skips `#hide` tasks in its block-link picker. Those consumers are not
  changed; they simply stop seeing project-scheduled ordinary tasks as hidden.
- `dash.md` already excludes the replacement state twice over —
  `filter by function !task.scheduled.moment || task.scheduled.moment.isSameOrBefore(moment(), "day")`
  and `is not blocked`, plus `status.type is TODO` in its sections — so a
  `[scheduled:: <future>]` `[?]` task stays off the dashboard without `#hide`.
- `blocked.md` selects `(is blocked) OR (status.name includes Blocked)` and filters
  `#hide`. Project-scheduled tasks will therefore start appearing there. That is
  intended and consistent with the `future_scheduled_tasks_blocked` contract; no query
  change is needed.
- `bob task-status-hooks` gives derived Blocked precedence over Pomodoro-derived Next/In
  Progress. A task in a future-scheduled project that is pulled into today's Pomodoro
  will therefore stay `[?]` instead of being promoted to `[*]`/`[/]`, where a `#hide`
  task used to be promoted normally. This is an accepted consequence: reschedule the
  task (or the project) to work on it.
- `projects.rs` treats every checkbox mark other than `x`, `X`, and `-` as open, so
  `[?]` already counts as an open task for `open_task_count`/`open_unhidden_count`.
  Blocked project tasks therefore keep `^prj` hidden through the ordinary surfacing rule
  on unscheduled projects with no extra work.
- `bob-plugins` is the linked source-of-truth repo; validated plugin changes must be
  deployed with `bob plugins sync`. Neither `bob projects sync` nor
  `bob task-status-hooks` is wired into `bob nightly` or any cron/chezmoi job; both are
  run on demand.

## Design decisions

State these in the docs as well, because they define the new contract.

1. **The inline `scheduled` field is the propagated state; the `[?]` checkbox is derived
   from it.** `bob projects sync` writes task-level `[scheduled:: ...]` fields and never
   writes a checkbox character. `bob task-status-hooks` remains the single CLI owner of
   `[?]` and already reconciles it in **both** directions from those fields. This avoids
   duplicating the Blocked-status registry validation and the vault-wide
   dependency/Pomodoro context inside `projects.rs`, and avoids a `projects sync` that
   could block but never unblock. _Consequence:_ after `bob projects sync` alone, a
   newly propagated task is scheduled but not yet `[?]` (still hidden from `dash.md` by
   the schedule filter), and a matured task is still `[?]` until the next
   `bob task-status-hooks` run. `bob projects sync` must print a hint pointing at
   `bob task-status-hooks` whenever it changes task schedules. _Rejected alternative:_
   forward-only `[?]` writes in `bob projects sync`. It would need
   `validate_blocked_status`/`TasksSettings` exported from `task_status_hooks.rs` and
   would still be unable to unblock, producing a confusing one-way command.
2. **The interactive path does the whole job immediately.** `Ctrl+Shift+P` on a `^prj`
   task already rewrites the entire note in one guarded transaction, and the plugin
   already owns both a blocker and a recovery snapshot, so it applies the propagation
   _and_ the `[?]`/recovery reconciliation at once — the behavior the request asks for
   at the point of interaction.
3. **`#hide` stays the mechanism for the `^prj` lifecycle task only**, both for schedule
   suppression (future date forces exactly one `#hide` on `^prj`) and for the
   pre-existing dash-surfacing rule. Ordinary tasks in a project with a valid
   frontmatter schedule get all whole-token `#hide` tags removed, at every checkbox
   status. That removal doubles as the migration path for tags the old policy wrote.
4. **Skip rule.** A task keeps its own schedule when it already declares a valid
   `YYYY-MM-DD` value on the project's date or later. Absent, malformed, and earlier
   values are replaced with the project's date. A task carrying more than one
   `scheduled` field is left untouched and reported as a warning.
5. **Unscheduling.** `Ctrl+D` on the project-backed `scheduled` item removes the
   propagated fields, because it still knows the date being removed; it strips inline
   `scheduled` fields whose value is exactly that date from ordinary open tasks and
   recovers those tasks. `bob projects sync` cannot do this — once the frontmatter key
   is gone the project is indistinguishable from one that was never scheduled — so a
   schedule removed outside the picker leaves matured-on-date task schedules behind.
   Document that, and prefer `Ctrl+D` for unscheduling.

## The new contract

For a **non-terminal** project note with a valid frontmatter `scheduled: P`, let `today`
be the local current date (`BOB_NOW` override honored):

| Line                                                                                      | `P > today`                                             | `P <= today`                                                                       |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Open ordinary task (` `, `*`, `/`, `?`) with no valid schedule, or a valid schedule `< P` | set `[scheduled:: P]`; strip `#hide`                    | set `[scheduled:: P]`; strip `#hide`                                               |
| Open ordinary task with a valid schedule `>= P`                                           | leave schedule; strip `#hide`                           | leave schedule; strip `#hide`                                                      |
| Open ordinary task with two or more `scheduled` fields                                    | leave line unchanged; warn                              | leave line unchanged; warn                                                         |
| Ordinary task with a done/canceled/unknown mark                                           | strip `#hide` only                                      | strip `#hide` only                                                                 |
| `^prj` lifecycle task                                                                     | force exactly one `#hide`; never add an inline schedule | leave `#hide` as-is unless `^prj` is the note's only task, in which case remove it |
| Frontmatter, fenced code, non-task lines                                                  | untouched                                               | untouched                                                                          |

Derived `[?]` follows from the task-level schedule through the existing rules:
`bob task-status-hooks` marks a recognized open task `[?]` while its schedule is later
than the daily anchor and recovers it once no derived reason remains. `Ctrl+Shift+P`
applies the same transition synchronously.

Invalid frontmatter schedules remain per-file scan errors and leave the file untouched.
Terminal (`done`/`canceled`) projects are still excluded from schedule reconciliation.

## Implementation

### 1. Replace the `#hide` visibility policy in `bob projects sync`

In `src/native/projects.rs`:

1. Replace `TaskVisibilityPolicy` with a schedule-propagation policy. It needs the
   project date `P`, whether `P` is future, and whether `^prj` is the note's only task
   (the existing `include_prj` input). Keep the change plan/apply split:
   `plan_project_sync_at` decides and counts, `apply_project_changes` produces
   `TextEdit`s.
2. Rework `task_visibility_edits` into a schedule-reconciliation edit builder that, for
   each real task line after the frontmatter and outside fences (reuse
   `parse_task_line`, `markdown_fence_line`, `line_spans`, `is_valid_prj_task_line`):
   - for the `^prj` line, apply only the `#hide` rule from the contract table above
     (this is today's `include_prj` behavior, unchanged);
   - for every other task line, remove all whole-token `#hide` spans (`tag_spans` +
     `remove_tag_spans_from_line`);
   - for every other task line whose mark is one of ` `, `*`, `/`, `?`, upsert
     `[scheduled:: P]` unless the line already declares a valid schedule on `P` or
     later, or declares more than one `scheduled` field.
3. Detect an existing task schedule from both the square-bracket and parenthesized
   Dataview forms so a task written as `(scheduled:: 2026-08-01)` is not given a
   duplicate bracketed field. Generalize `inline_field_span` / `inline_field_value` (or
   add a sibling that scans both delimiters and returns every occurrence, so the
   duplicate-field warning is possible). Always _write_ the bracketed form. Emoji Tasks
   dates (`⏳ 2026-08-01`) are not recognized by the CLI's Blocked derivation and stay
   out of scope; say so in the docs.
4. Parse candidate values with the same strictness as `parse_project_schedule`: exactly
   `YYYY-MM-DD` and a real calendar date. Anything else is "no usable schedule" and is
   replaced.
5. Insert a new field immediately before a trailing `^block-id` when one exists,
   otherwise at the end of the trimmed line — this is exactly
   `task_hide_tag_insertion_offset`, so rename it to a metadata-neutral name and reuse
   it. Replacing an existing field must reuse its span so surrounding text is untouched.
   Preserve indentation, list markers, blockquote prefixes, descriptions, other inline
   fields, tags, block IDs, and CRLF line endings.
6. Replace `ProjectChange::ReconcileTaskVisibility` and `SyncEvent::TaskVisibility` with
   a schedule-oriented change and event carrying the project date, the number of task
   lines that received or updated a schedule, and the number of task lines that had
   `#hide` removed. Human output should read like
   `ok roadmap  scheduled 4 tasks 2026-08-16  frontmatter scheduled is future` and
   `ok roadmap  removed #hide from 4 tasks  task schedules replace #hide`, with the
   summary counter renamed from `task visibility updated` to `task schedules updated`.
   Add the ambiguous-`scheduled`-field warning as a normal per-project warning
   (non-fatal, not auto-fixed). When any task schedule changed, print one hint line
   telling the user that `bob task-status-hooks` reconciles the `[?]` markers.
7. Leave the `^prj` surfacing rule, sub-projects ledger, status reconciliation,
   `RemoveScheduled`, warnings, exit codes, and the `project.scheduled.is_none()` guard
   exactly as they are.

Add unit coverage beside the existing `projects.rs` tests and CLI coverage in
`tests/cli.rs` with `BOB_NOW`-pinned fixtures for:

- future and due/past project dates, and the boundary at the scheduled date itself;
- tasks with no schedule, an earlier schedule, an equal schedule, a later schedule, a
  malformed schedule, a parenthesized schedule, and duplicate `scheduled` fields;
- ` `, `*`, `/`, `?`, `x`, `X`, `-`, and unknown/custom marks;
- legacy `#hide` tags removed from ordinary tasks at every status, including multiples
  on one line, while `#hidden`/`#hideaway` and other tags survive;
- `^prj` keeping exactly one `#hide` when future, being unhidden when due and sole, and
  never receiving an inline schedule;
- nested, ordered-list, and blockquoted tasks; fenced examples; frontmatter; and CRLF
  files;
- terminal projects and invalid frontmatter dates leaving files untouched;
- dry-run writing nothing, the new human/summary output, and a second run being a no-op;
- an end-to-end fixture where `bob projects sync` followed by `bob task-status-hooks`
  produces `[?]`, and where re-running both after the date matures recovers the tasks.

### 2. Keep `bob projects list`'s `SHOWN` column honest

`open_unhidden_count` currently drives both the `^prj` surfacing rule and the `SHOWN`
column. Once ordinary tasks stop carrying `#hide`, `SHOWN` would report the full open
count for every future-scheduled project.

Split the two uses: keep `open_unhidden_count` exactly as it is (open `#task`, not
`^prj`, no `#hide`) as the input to the surfacing rule, and add a separate
dash-visibility count for the `SHOWN` column that additionally excludes a task whose
checkbox is `?` or whose valid task-level schedule is later than the current local date.
Do not feed the new count into `plan_project_sync_at`; changing the surfacing input
would make `^prj` surface for projects whose tasks are merely dependency-blocked, which
is out of scope. Cover both counts in
`projects_list_scans_project_notes_and_renders_counts` and document the `SHOWN`
definition in `docs/projects.md`.

### 3. Propagate schedules from `Ctrl+Shift+P` and clean up on `Ctrl+D`

In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`:

1. Replace `normalizeTaskHideTag`/`planProjectScheduleVisibility` with a
   schedule-propagation planner that implements the same contract table as the CLI. It
   takes the note content, the project date, `today`, and an optional per-line recovery
   map, and returns the rewritten content plus counts. Reuse `getRealMarkdownTaskLines`,
   `isProjectLifecycleTaskLine`, `getWholeTaskTagSpans`/`removeTextSpans`,
   `parseBulletPropertyFields`, `upsertBulletProperty`, `validateProjectScheduledDate`,
   and `getObsidianTaskCheckboxStatus`. Keep the existing `#hide` handling for the
   `^prj` line only.
2. Compose status reconciliation with the existing helpers rather than new logic:
   `blockObsidianTaskCheckboxStatus` for a future project date, and
   `reconcileBlockedScheduledTaskLine(line, recovery)` for a due/past date or a delete.
   A task whose own later schedule was preserved must still be blocked, and must never
   be recovered by a due project date.
3. Give `planProjectScheduledUpdate` and `planProjectScheduledDelete` an optional
   `recoveryByLine` option and thread it into the planner. `planProjectScheduledDelete`
   additionally strips inline `scheduled` fields from ordinary open tasks when the value
   is exactly the frontmatter date being removed, leaving any other value alone, then
   reconciles those lines. Both keep stripping any inline `scheduled` from the `^prj`
   line and keep their existing frontmatter edits and `cursorLineDelta` behavior.
4. Make the runtime paths snapshot-aware, following the pattern already used by
   `setBulletPropertyValue`: `setProjectNoteScheduledValue` and
   `deleteProjectNoteScheduledValue` compute the candidate task line indices from the
   guarded preimage,
   `await buildTargetScheduledRecoveryByLine(this.app, filePath, content, lines, today)`
   when the operation can recover (due/past date, or delete), then **revalidate** the
   write context (active file, editor, full preimage, expected `^prj` line and expected
   frontmatter value) before applying one `cm.setValue` transaction. A future date needs
   no snapshot. Preserve caret/viewport behavior and CRLF.
5. Update `planCountedBulletPropertyBatch`'s project-frontmatter branch for the renamed
   plan fields and pass the recovery map through; keep the rest of the counted contract
   (stale-session rejection, single transaction, changed /unchanged target accounting,
   `taskLineDelta` mapping, mixed project-plus-inline batches) intact. An ordinary
   inline target in the same counted batch that also receives the project's propagated
   schedule must end up with exactly one `scheduled` field and one status decision.
6. Rework the notices from the hide/show wording to the new one, e.g.
   `scheduled → 2026-08-16; scheduled 4 tasks; marked 4 Blocked` and, on a due date or
   delete, the existing `scheduledRecoveryNoticeSuffix` counts
   (`recovered N Ready/Next/In Progress`, `N still Blocked`,
   `N deferred to bob task-status-hooks`). Keep the conservative deferral behavior: when
   the snapshot cannot prove recovery, apply the schedule edits and leave `[?]` alone.
7. Bump the Bob Navigation Hotkeys patch version, and update its `manifest.json`
   description and the root `README.md` plugin table row to say that project schedules
   propagate to task-level `scheduled` properties and Blocked status instead of `#hide`.

Extend `scripts/test-navigation-hotkeys.cjs` with pure-planner and runtime coverage
mirroring the CLI test list above, plus: fixed today/tomorrow/yesterday boundaries;
single and counted `^prj` selections; recovery to Ready, Next, and In Progress; a
remaining open dependency keeping a task `[?]`; unreadable/ambiguous snapshots
deferring; `Ctrl+D` removing only exactly-matching propagated fields; mixed
project-plus-inline counted batches; stale-source aborts writing nothing; caret/viewport
preservation; and CRLF notes.

### 4. Documentation

- `docs/projects.md`: rewrite the scheduling bullets in **Sync Rules**, the `SHOWN`
  definition, the `### Scheduling from the ^prj task` section, and the example output.
  State the contract table, the skip rule, the duplicate-field warning, that `^prj`
  keeps `#hide`, that `bob task-status-hooks` owns `[?]`, and that removing `scheduled`
  outside `Ctrl+D` leaves propagated fields behind.
- `README.md` (bob-cli): update the `bob projects` paragraphs that describe
  `#hide`-based schedule visibility.
- `docs/task-status-hooks.md`: replace the paragraph stating that project frontmatter
  "does not participate" and "continues to own that field and its `#hide` behavior" with
  the new relationship — project frontmatter is propagated into task-level schedules by
  `bob projects sync` and the property picker, and `task-status-hooks` then treats those
  fields exactly like any other inline schedule. Note that Blocked outranks Pomodoro
  promotion for such tasks.
- Command help/`long_about` for `bob projects sync` must match the new behavior.
- `bob-plugins/README.md` plugin table row and `manifest.json` description, as above.
- No `~/bob/dash.md` or `~/bob/blocked.md` query changes; call out explicitly in
  `docs/projects.md` that project-scheduled tasks now appear in `blocked.md`.

### 5. Migration, validation, and deploy

1. **Preview the vault migration before touching it.** Run `bob projects sync --dry-run`
   and review the per-project counts; this is a large one-time rewrite across every
   scheduled project. The vault is committed by `bob nightly`'s `bulk-git-commit`, so
   confirm the working tree is clean (or committed) first, then apply, then run
   `bob task-status-hooks --dry-run` and review before applying it.
2. **Audit stranded tags.** Projects whose `scheduled` frontmatter was removed while the
   old policy had applied `#hide` keep those tags, and no command can distinguish them
   from hand-authored ones. Produce a list (for example with `bob query` or a grep for
   task lines carrying `#hide` without a trailing `^prj`) and let the user decide; do
   not bulk edit them.
3. In `bob-cli`: targeted `cargo test` filters for `projects` and `task_status_hooks`,
   then `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
   `cargo test`).
4. In `bob-plugins`: `node --test scripts/test-navigation-hotkeys.cjs`, then `npm test`
   and `npm run validate`.
5. Run `bob plugins sync` only after the plugin suite and manifest validation pass, then
   confirm Bob Navigation Hotkeys reports `synced` at the bumped version.
6. Verify with a deterministic fixture vault (fixed `BOB_NOW`/`BOB_DAY_FILE`) that a
   future-scheduled project's tasks are absent from a `bob query --tasks-note dash.md`
   run and present in a `blocked.md` run, and that both flip once the date matures. Do
   not run a mutating live-vault `bob task-status-hooks` pass as part of automated
   validation.

## Acceptance criteria

- `bob projects sync` on a non-terminal project with a valid frontmatter `scheduled: P`
  gives every open ordinary task a `[scheduled:: P]` field unless it already declares a
  valid schedule on `P` or later, removes every whole-token `#hide` from ordinary tasks
  at any status, and adds no inline schedule to `^prj`.
- `^prj` still carries exactly one `#hide` while `P` is in the future, is unhidden when
  `P` is due and `^prj` is the note's only task, and its dash-surfacing rule is
  otherwise unchanged.
- A subsequent `bob task-status-hooks` run marks those tasks `[?]` while `P` is later
  than the daily anchor and recovers them to Ready, Next, or In Progress once `P` is due
  and no other derived blocking reason remains.
- `Ctrl+Shift+P` choosing a project date applies the propagation and the `[?]`/recovery
  decision in one guarded editor transaction, and `Ctrl+D` removes the frontmatter
  property, the exactly-matching propagated task fields, and the resulting Blocked
  markers.
- Tasks with a later own schedule, duplicate `scheduled` fields, terminal or unknown
  checkbox marks, fenced examples, frontmatter, and non-task lines are not incorrectly
  rewritten; invalid frontmatter dates still leave the file untouched with a per-file
  error.
- `bob projects list` still reports `0` in `SHOWN` for a future-scheduled project.
- Repeated `bob projects sync` and `bob task-status-hooks` runs are idempotent; dry runs
  write nothing.
- All bob-cli and bob-plugins checks pass, docs describe the new contract, and the
  bumped Bob Navigation Hotkeys is synced source-identically into the vault.
