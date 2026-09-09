---
tier: tale
title: Dependency-aware blocked task status
goal: "Tasks with unresolved task dependencies are marked with a reliable, visually
  coherent Blocked status by both bob mark-next-tasks and the Obsidian dependency
  hotkey, while dashboard queries and existing task workflows remain correct.

  "
create_time: 2026-09-09 19:53:02
status: wip
---

# Plan: Dependency-aware blocked task status

## Outcome and product contract

Add a first-class Obsidian Tasks status with this shared contract:

- Markdown symbol: `[?]` (currently unused by real `#task` lines in the vault).
- Name: `Blocked`.
- Type: `ON_HOLD`, so Tasks continues to regard it as open/non-terminal and dependency
  chains can remain transitively blocked.
- Next status: Ready (`[ ]`), making a manual click an explicit return to the ordinary
  queue; the authoritative `bob mark-next-tasks` reconciliation can immediately
  re-derive Blocked if an open dependency still exists.
- Presentation: a restrained red/rose accent, soft background, normal unstruck task
  text, and a crisp `?` checkbox marker. Reuse the same `--task-status-blocked` color
  token in the dashboard BLOCKED count chip so source tasks, rendered query results, and
  summary navigation read as one visual system.

“Blocked” is derived state, not a second dependency model. A non-terminal task is
blocked exactly when at least one value in its `dependsOn` property resolves to at least
one non-terminal task with the matching `id`. Match Obsidian Tasks semantics: IDs are
vault-wide, any open duplicate is sufficient, self-dependencies block, missing IDs are
ignored, and `DONE`, `CANCELLED`, and `NON_TASK` targets do not block. Only direct
dependencies decide a task's marker; transitive effects arise naturally because a
Blocked target is itself open.

The status transition precedence is:

1. Terminal tasks remain untouched even if they retain historical dependency metadata.
2. Any recognized open task with an open dependency becomes or remains Blocked,
   overriding Ready, Next, and In Progress for the duration of the block.
3. A Blocked task with no remaining open dependency returns to the active status
   computed from the final Pomodoro graph (`[/]` or `[*]`), or to Ready (`[ ]`) when it
   is not reachable. This intentionally does not preserve a stale pre-block status in
   hidden metadata.
4. All other tasks keep the existing monotonic Next/In-Progress and unreachable-Next
   behavior.

The command remains the vault-wide source of truth. The `!` hotkey provides immediate
feedback when creating a dependency; removing a dependency stays status-neutral because
the hotkey does not have an authoritative snapshot of every remaining vault dependency.
The next `bob mark-next-tasks` run performs complete unblocking.

## Implementation

### 1. Install and style the vault status

- Extend `~/bob/.obsidian/plugins/obsidian-tasks-plugin/data.json` with the `[?]`
  Blocked status, preserving all unrelated Tasks settings and status order. Keep the
  status available in Tasks commands so it remains inspectable and editable through the
  normal UI.
- Extend `~/bob/.obsidian/snippets/task-statuses.css` with dedicated blocked
  color/background variables and selectors for symbol `?` (and, where useful, status
  name `Blocked`). Narrow the existing Next selectors that match every `ON_HOLD` task:
  after Blocked is added, status type alone can no longer distinguish green Next from
  red Blocked. Include Blocked in the shared card, source-line, checkbox, and
  non-strikethrough rules without changing Done or Canceled presentation.
- Update only the dashboard presentation hook in `~/bob/dash.md` to consume
  `--task-status-blocked` for its BLOCKED chip. Keep the three Tasks queries based on
  `is not blocked`; their semantic dependency filter remains correct and must not be
  replaced with a fragile `status.name` or `status.symbol` filter. Keep
  `~/bob/blocked.md` based on `is blocked` for the same reason.

### 2. Make `bob mark-next-tasks` reconcile dependency status

- Extend `src/native/mark_next.rs` task/settings models to retain each task's Tasks
  status type, `[id:: ...]`, and `[dependsOn:: ...]` values while preserving the current
  global-filter, fence, directory-exclusion, line-ending, and status-byte-offset
  behavior. Parse both bracket and parenthesized Dataview fields using the same
  identity/value conventions already exercised by the native Tasks implementation.
- Read the configured status registry, distinguish open types (`TODO`, `IN_PROGRESS`,
  `ON_HOLD`) from terminal types, and validate the Blocked contract before planning a
  write that needs it. A missing, duplicate, or incompatible Blocked definition must
  produce an actionable error and no partial note writes instead of emitting an unknown
  checkbox status. Preserve the existing narrower `DONE` classification used for
  completed Pomodoro-link retirement.
- After the full vault scan and final post-structural Pomodoro graph are known, build
  the vault-wide ID index and derive each task's open direct dependencies. Combine that
  result with the existing desired active-status map in one transition planner so each
  task receives at most one checkbox edit and daily-note structural/status changes still
  compose atomically.
- Teach ranked dependency propagation that Blocked is open but has no active rank.
  Incoming Pomodoro rank should flow through a blocked intermediate to its dependency
  targets, while Blocked itself wins the final displayed marker.
- Add explicit dry-run/human sections and additive JSON fields for tasks marked Blocked
  and tasks unblocked, including their actual from/to symbols and unresolved/open
  dependency IDs where useful. Include these edits in no-op detection and summaries
  without changing the meaning of existing `marked_next`, `marked_in_progress`,
  `cleared`, or reference fields.
- Update `docs/mark-next-tasks.md`, the README command summary, and help text to
  document dependency-property status reconciliation, precedence, missing/duplicate
  identity behavior, recovery from `[?]`, dry-run output, and the Tasks settings guard.

### 3. Give the `!` keymap immediate blocked feedback

- In the linked `bob-plugins` repository, update
  `plugins/bob-navigation-hotkeys/main.js` around the existing dependency-aware
  transclusion transaction. Treat `[?]` as an open task status for picker/navigation
  purposes.
- When a plain sole block link becomes a transclusion and the validated target task
  snapshot is open, add `dependsOn` and set the parent task to `[?]` in the same
  source-editor transaction. Do not block the parent for Done, Canceled,
  unknown/non-task, hidden, unresolved, invalid, or stale targets, and do not change
  parent status when removing a dependency.
- Preserve the existing atomicity rules. For cross-file targets, mark the parent only
  after the target snapshot and external write are accepted; a stale or failed external
  edit must not leave `dependsOn` or Blocked metadata behind. For counted `!`
  operations, aggregate by parent so one successfully linked open target blocks that
  parent, terminal targets do not, and repeated targets retain the strongest existing
  target-promotion behavior.
- Keep Blocked out of the active rank ordering used to promote dependency targets. A
  blocked parent can still request the minimum active rank already implied by the
  surrounding operation, but `[?]` must never be misread as stronger than Next or In
  Progress.
- Add focused helper/runtime tests in `scripts/test-navigation-hotkeys.cjs` for
  same-file and cross-file open targets, terminal targets, already-Blocked targets,
  mixed counted toggles, failed/stale external writes, hidden/invalid endpoints, and
  status-neutral unlinking. Bump the plugin patch version and update its manifest/README
  description.
- Deploy only from the linked source of truth with `bob plugins sync`; never edit the
  deployed `~/bob/.obsidian/plugins/bob-navigation-hotkeys/` copy directly. Verify the
  synced manifest, JavaScript, and styles match the source repository.

### 4. Protect compatibility and edge cases

- Update mark-next unit/integration fixtures that currently use `[?]` as a generic
  unknown status to use a different truly unknown symbol, then add dedicated fixtures
  for Blocked. Cover Ready/Next/In-Progress to Blocked, idempotent already-Blocked
  tasks, unblocking to Ready/Next/In-Progress, terminal parents and targets,
  multiple/missing/duplicate IDs, self-dependencies, dependency chains/cycles, mixed
  bracket/parenthesis metadata, CRLF, dry-run, JSON/human output, guard-rail failures,
  and composition with Pomodoro structural edits.
- Extend native Tasks parity coverage so the configured Blocked status is hydrated as
  `ON_HOLD`, remains `not done`, and participates in `is blocked`/`is blocking` exactly
  like the installed Tasks 8.0.0 behavior. Preserve the dashboard invariant: WIP, NEXT,
  and READY exclude semantically blocked tasks, while the BLOCKED count/page include
  them.
- Keep archived/generated/template and non-`#task` checkboxes out of reconciliation,
  retain idempotent whole-vault writes, and avoid changing the public CLI option
  surface.

## Validation and rollout

1. Run focused Rust tests for mark-next transitions/settings/metadata and CLI fixture
   tests, then `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `cargo test` (or the repository's `just all`).
2. In `bob-plugins`, run `npm test` and `npm run validate`, including the expanded
   navigation-hotkey coverage and manifest/version checks.
3. Run the real-vault parity test against a copied snapshot with a fixed `BOB_NOW`;
   confirm every `dash.md` Tasks block executes and independently matches WIP/NEXT/READY
   ground truth after the new status registry is loaded. Also execute the `blocked.md`
   Tasks block and verify its results match semantic dependency blocking rather than
   merely `[?]`.
4. Run `bob mark-next-tasks --dry-run --format json` against a controlled fixture that
   contains every transition, apply it, rerun for an `already in sync` no-op, close
   dependencies, and rerun to verify deterministic recovery. Then use a live-vault dry
   run to review the exact tasks that would be newly blocked/unblocked without silently
   changing them.
5. Sync the updated navigation plugin, reload Tasks/Obsidian so the new registry is
   active, and smoke-test one same-note and one cross-note `!` toggle in Obsidian.
   Confirm the parent changes to the red `[?]` treatment only for an open target, target
   promotion still works, source/query rendering agree, the dashboard chip uses the same
   accent, and WIP/NEXT/READY/BLOCKED membership is correct.

## Non-goals

- Do not replace `dependsOn` or Obsidian Tasks' `is blocked` semantics with the checkbox
  marker.
- Do not persist a hidden “previous status” field or automatically restore a historical
  In-Progress state that is no longer derivable from the Pomodoro graph.
- Do not make the `!` hotkey perform a full-vault unblocking scan; authoritative cleanup
  belongs to `bob mark-next-tasks`.
- Do not add CLI flags or migrate unrelated custom statuses, task metadata, dashboard
  layout, or plugin behavior.
