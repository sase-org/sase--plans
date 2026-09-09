---
tier: tale
title: Recover due scheduled Obsidian tasks from Blocked
goal:
  When an inline task schedule becomes due or is removed, recover Blocked tasks to Ready
  or their recent Pomodoro-derived rank without overriding another blocking reason.
create_time: 2026-09-09 19:53:10
status: wip
---

# Recover due scheduled Obsidian tasks from Blocked

## Goal

Make future-schedule blocking genuinely bidirectional. A recognized `[?]` Obsidian task
whose valid task-level `[scheduled:: YYYY-MM-DD]` date is today or earlier must stop
being Blocked once it has no other derived blocking reason. It should recover to Ready
(`[ ]`) when it has no recent activity rank, Next (`[*]`) when it is directly reachable
from a recent Pomodoro reference, or In Progress (`[/]`) when the existing
Pomodoro/transclusion graph provides that stronger rank.

Apply the rule authoritatively in `bob task-status-hooks` and immediately when
Ctrl+Shift+P changes or deletes an inline `scheduled` property. Preserve the existing
rolling-daily contract: the command's selected current daily note and newest existing
earlier canonical daily note are the two recent-activity sources. In the plugin, local
today and the newest canonical daily note before today provide the equivalent
interactive context. “Previous” deliberately means the newest earlier daily note rather
than requiring the literal calendar day before the anchor, so gaps, weekends, and year
boundaries keep working.

Do not store or guess a hidden pre-Blocked status. A directly referenced task that is
currently `[?]` has no active rank to preserve and therefore recovers to Next. In
Progress is selected only when the existing monotonic dependency transclusion graph
supplies an In-Progress request, such as reachability from an already-In-Progress task.
Project frontmatter `scheduled:` on a `^prj` lifecycle task remains governed by the
existing project visibility workflow and is outside this inline task rule.

## Audit findings and constraints

- Commit `9bb625a` already made `bob task-status-hooks` unblock `[?]` when a schedule
  reaches the daily anchor and no open Dataview dependency remains. Current
  open-Pomodoro reachability restores Next/In Progress; otherwise the task becomes
  Ready.
- The CLI's previous-daily references currently protect only tasks that are already
  `[/]`. Because a scheduled task is `[?]`, a task referenced only in the previous daily
  note currently becomes Ready rather than Next/In Progress when its date matures.
- Bob Navigation Hotkeys 1.13.12 intentionally implements only the forward half. Its
  single and counted setters turn supported open inline tasks into `[?]` for future
  dates, but its fixed-boundary test proves that selecting today leaves `[?]` unchanged.
  Deleting `scheduled` also leaves `[?]` unchanged.
- A due schedule is not sufficient by itself to unblock a task: an open
  `[dependsOn:: ...]` target must continue to keep the parent `[?]`. Terminal, non-task,
  and unrecognized/custom task statuses remain untouched.
- The current CLI separates active promotion roots from rolling recent activity. The
  follow-up must not let historical references start promoting ordinary Ready tasks or
  prevent the existing scoped stale-In-Progress cleanup.
- Editor writes must retain the plugin's stale-source guards, CRLF and formatting
  preservation, caret/viewport behavior, and one-transaction counted semantics. Vault
  reads needed to decide recovery happen before the guarded source transaction and must
  include unsaved open Markdown buffers where available.
- `~/bob/blocked.md` already selects `(is blocked) OR (status.name includes Blocked)`
  without excluding future dates. No query change should be needed: recovered tasks
  disappear when their `[?]` marker is removed, while tasks with a remaining open
  dependency stay present.
- `bob-plugins` is the linked source-of-truth repository. Validated plugin changes must
  be deployed with `bob plugins sync`.

## Implementation

### 1. Give `task-status-hooks` a recovery-only recent rank

In `src/native/task_status_hooks.rs`:

1. Keep the existing active `desired` map based on surviving block links beneath current
   open Pomodoros. It remains the only map used for ordinary Ready/Next/In-Progress
   promotion and vault-wide Next clearing.
2. Build a second recovery map from the already-resolved, non-retired
   current-plus-previous recent-activity roots, traversing the same eligible task
   transclusion graph with the same strongest-rank, merge, and cycle rules as
   `desired_statuses`. Combine it with the active map by taking the stronger rank.
3. Use that combined rank only when a recognized `[?]` task has no future inline
   schedule and no recognized open Dataview dependency. A directly recent Blocked root
   defaults to Next; a stronger reachable graph request may recover it to In Progress; a
   task absent from the recovery map returns to Ready. Leave all non-Blocked promotion
   and cleanup decisions on the original active/recent inputs.
4. Continue giving remaining derived blocking reasons precedence over recovery. A
   due/past or deleted schedule does not unblock a task while any dependency ID resolves
   to a recognized open task. Future scheduling likewise keeps the task Blocked
   regardless of recent rank.
5. Preserve daily-note safety: the previous daily note remains read-only and excluded
   from task-status writes; retired links do not count; same-note, aliased, embedded,
   and cross-note block links retain current resolution behavior; a sectionless newest
   earlier daily contributes no recovery roots and does not fall through to an older
   note.
6. Update command help, `README.md`, and `docs/task-status-hooks.md` to distinguish
   active promotion from recovery-only recent rank, define direct Next versus
   graph-derived In Progress, and explain that no previous status is persisted.

Add focused unit and CLI integration coverage in `src/native/task_status_hooks.rs` and
`tests/cli.rs` for:

- yesterday/today/past schedule boundaries under a fixed dated `BOB_DAY_FILE`, plus
  schedule deletion represented by absent metadata;
- `[?]` recovery to Ready with no recent reference, Next from a non-retired current or
  newest-previous direct reference, and In Progress through a stronger transclusion
  path;
- open Pomodoro, completed Pomodoro, aliases/embeds, retired links, missing daily dates,
  a sectionless previous daily, and the newest-earlier-note selection across gaps;
- a remaining open Dataview dependency or future schedule overriding every recovery
  rank;
- no promotion of an ordinary `[ ]` task referenced only in the previous daily, no
  mutation of the previous daily, dry-run non-mutation, human/JSON unblock reporting,
  and second-run idempotency.

### 2. Compute an interactive recovery snapshot in Bob Navigation Hotkeys

In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`, add testable helpers and a
small runtime snapshot builder that mirror only the task-status facts needed by
scheduled recovery:

1. Parse the installed Obsidian Tasks settings from
   `.obsidian/plugins/obsidian-tasks-plugin/data.json` when available, including the
   global filter and single-character status types. Use the same conventional fallback
   statuses as the CLI. Treat an unreadable or ambiguous status registry conservatively:
   never turn `[?]` into an active status when the remaining-block decision cannot be
   proven.
2. Snapshot Markdown files from `app.vault.getMarkdownFiles()`, preferring each open
   editor's unsaved content over `cachedRead`. Override the active source path with the
   exact editor preimage being guarded by the property picker. Parse real
   globally-filtered task lines, trailing block IDs, `[id:: ...]`, `[dependsOn:: ...]`,
   and eligible child task-transclusion edges with the CLI's file-scoped identity and
   note-resolution rules.
3. Select today's canonical `<year>/<YYYYMMDD>.md` and the newest existing earlier
   canonical daily note. From their `## Pomodoros` sections, collect non-retired block
   references from recognized open or completed Pomodoros. Resolve aliases, embeds,
   same-note links, and explicit paths to path-plus-block identities; ignore fenced
   examples, canceled Pomodoros, ordinary/heading links, ambiguous note targets, and
   struck retired links.
4. Compute, once per single or counted picker action:
   - which target task lines retain an open Dataview dependency after the proposed
     property edit; and
   - the recovery-only Next/In-Progress rank reachable from the two daily ledgers and
     eligible transclusion graph. Keep unknown, duplicate, stale, or unreadable facts
     conservative rather than incorrectly unblocking.
5. Expose the decision to the existing pure single/count planning helpers as immutable
   per-target recovery metadata. Do not write any non-selected task or daily note. A
   selected Blocked task with no usable trailing block ID can still recover to Ready
   after proving it has no remaining derived block, but it cannot receive a
   Pomodoro-derived rank.

Keep the helper surface local to Bob Navigation Hotkeys; do not import private code from
Bob Ledger Tools or another plugin.

### 3. Compose due-schedule recovery into Ctrl+Shift+P writes

Update the bare and counted property mutation paths:

1. For an inline `scheduled` set, evaluate the post-edit value. A valid date after local
   today keeps the existing Ready/Next/In-Progress-to-Blocked behavior. A valid date
   today or earlier makes an existing `[?]` task eligible for recovery. Deleting inline
   `scheduled` has the same recovery behavior because it removes the scheduling reason.
2. Before recovering `[?]`, require the interactive snapshot to prove that no open
   Dataview dependency remains. Recover to the snapshot's recent rank or Ready. If
   another open dependency remains, leave `[?]` unchanged. Do not change Done, canceled,
   non-task, unknown/custom, or already-active Ready/Next/In-Progress checkboxes.
3. Continue excluding project-frontmatter `scheduled` on `^prj`. Mixed counted batches
   may update project frontmatter and inline tasks together, but only ordinary inline
   targets participate in task status recovery.
4. Convert the runtime single/count setters and schedule deletion paths to await the
   snapshot where needed. After every asynchronous read, revalidate the active file,
   editor, full source preimage, and counted target session before applying one guarded
   editor transaction. A status-only recovery counts as a changed target even when the
   selected property value was already present.
5. If the snapshot cannot safely prove recovery because a required vault read, status
   definition, identity, or dependency resolution is ambiguous, still apply the
   requested property edit but leave `[?]` unchanged and include a concise notice that
   status reconciliation was deferred to `bob task-status-hooks`. Report recovered
   Ready, Next, In-Progress, still-Blocked, and deferred counts in single/count notices
   without changing unrelated property notices.
6. Bump the Bob Navigation Hotkeys patch version and update its manifest/README
   description to state that inline schedules are reconciled in both directions.

Expand `scripts/test-navigation-hotkeys.cjs` with pure-helper and runtime coverage for:

- fixed yesterday/today/tomorrow boundaries and deleting `scheduled`;
- single and counted `[?]` recovery to Ready, direct recent Next, and graph-derived In
  Progress;
- current and newest-previous daily selection, date gaps/year boundaries,
  open/completed/canceled Pomodoros, retired links, aliases/embeds, same-note/cross-note
  resolution, fenced content, and sectionless daily notes;
- dependency-only and combined schedule/dependency blocking, closed/missing/duplicate
  IDs, custom Tasks statuses, and conservative unreadable/ambiguous fallback;
- project `^prj`, terminal/custom statuses, missing block IDs, mixed project-plus-inline
  batches, CRLF, status-only edits, notices, unsaved open buffers, source changes during
  asynchronous reads, caret/viewport preservation, one counted transaction, and no
  partial write after a stale guard.

### 4. Document, validate, and deploy

1. Keep `~/bob/blocked.md` unchanged unless validation exposes a regression. Its
   existing status-name branch should remove recovered tasks naturally and retain tasks
   that remain dependency-blocked.
2. In `bob-cli`, run targeted task-status-hooks tests, then `cargo fmt --check`,
   `cargo clippy --all-targets --all-features`, and `cargo test`.
3. In `bob-plugins`, run `node --test scripts/test-navigation-hotkeys.cjs`, then
   `npm test` and `npm run validate`.
4. Exercise deterministic fixtures with fixed dates to prove the same due/past `[?]`
   task becomes Ready, Next, or In Progress in the CLI and property-picker planners,
   while an open dependency keeps it in `blocked.md`.
5. Run `bob plugins sync` only after the linked plugin suite and manifest validation
   pass, then confirm the deployed Bob Navigation Hotkeys version and source match the
   linked repository. Do not run a mutating live-vault `task-status-hooks` pass during
   validation; an optional real-vault check must use `--dry-run`.

## Acceptance criteria

- `bob task-status-hooks` changes a recognized scheduled `[?]` task to `[ ]` when its
  valid date is today/earlier (or scheduling metadata is absent), no open dependency
  remains, and neither recent daily ledger reaches it.
- The same task recovers to `[*]` when directly reached from a non-retired
  current/newest-previous Pomodoro reference, or to `[/]` only when a stronger eligible
  transclusion path requests In Progress.
- Previous-daily recovery applies only to Blocked tasks whose reasons cleared:
  historical references do not promote ordinary Ready tasks, and the previous daily note
  is never modified.
- Bare and counted Ctrl+Shift+P set/delete operations make the same
  Ready/Next/In-Progress recovery decision in their guarded editor transaction, while
  future schedules still become `[?]`.
- A remaining future schedule or recognized open Dataview dependency keeps the task
  `[?]`; project lifecycle, terminal, non-task, unknown/custom, retired-link, ambiguous,
  and stale cases are not incorrectly reopened.
- Existing `blocked.md` behavior remains correct without a query change, all bob-cli and
  bob-plugins checks pass, and the bumped navigation plugin is synced source-identically
  into the vault.
