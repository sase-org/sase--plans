---
tier: tale
title: Dependency-aware task status setter and immediate recovery
goal: "Task statuses are authoritatively reconciled through the renamed bob
  task-status-setter command, and Ctrl+Enter immediately reopens Blocked dependents when
  closing their last open dependency.

  "
create_time: 2026-09-09 19:53:25
status: wip
---

# Plan: Dependency-aware task status setter and immediate recovery

## Context and product contract

The existing `bob mark-next-tasks` implementation already combines two sources of
derived state: the final Pomodoro link graph supplies Ready/Next/In-Progress rank, and
vault-wide Tasks `[id:: ...]`/`[dependsOn:: ...]` metadata supplies Blocked state.
Preserve that single-planner architecture and make its recovery rule explicit and
exhaustively covered:

1. Done, canceled, non-task, unknown, and otherwise terminal tasks remain untouched even
   if they retain dependency metadata.
2. A recognized active task with at least one dependency ID that resolves to an open
   task is or remains Blocked (`[?]`). Matching stays vault-wide and compatible with
   Obsidian Tasks: any open duplicate ID is sufficient, self-dependencies block, and
   missing/non-task/terminal targets do not block.
3. A Blocked task with no `dependsOn` values, or whose dependency IDs no longer resolve
   to any open task, is reopened. Its final Pomodoro-derived rank wins when available
   (`[/]` before `[*]`); otherwise it becomes Ready (`[ ]`). The command does not
   restore hidden historical state.
4. Other recognized active tasks retain the current monotonic promotion and
   unreachable-Next clearing rules.

Rename the canonical public command to `bob task-status-setter`, which now describes all
of these responsibilities. It becomes the only listed and documented name. Retain
`bob mark-next-tasks` as a hidden compatibility alias that routes to exactly the same
native implementation and emits the canonical command name in help and human output; do
not create a second behavior path or duplicate output schema. Preserve the existing
`--bob-dir`, `--dry-run`, and `--format` surface and all JSON field meanings.

Ctrl+Enter supplies an immediate, deliberately narrower version of recovery after it
successfully closes tasks. For every task actually changed to Done by the
keypress—including a directly selected task, a selected block-link target, and every
task closed by recursive transclusion or full Pomodoro completion—identify its real
Tasks `id` and inspect the post-close vault snapshot. A currently Blocked dependent
whose `dependsOn` list references one of those closed IDs becomes Ready only when none
of its dependency IDs still resolves to an open task. An open duplicate ID or any other
open dependency keeps it Blocked. Already-active, terminal, unknown, non-task,
unrelated, and no-longer-current snapshots are not rewritten.

The keymap does not have the CLI's final Pomodoro rank graph, so its immediate recovery
target is Ready rather than a guessed Next/In-Progress status. The next
`bob task-status-setter` run remains authoritative and can promote that task. The keymap
also does not opportunistically clean every stale Blocked task: a Blocked task with no
dependencies is recovered by the whole-vault command, while Ctrl+Enter limits edits to
dependents affected by the task(s) it just closed.

## Implementation

### 1. Canonicalize the CLI around `task-status-setter`

- In the Rust command registry, replace the listed `mark-next-tasks` entry with
  alphabetically positioned `task-status-setter`, rename the native command/module
  symbols and command-name constants where they communicate the old narrow purpose, and
  add one hidden dispatch alias for `mark-next-tasks`. Keep native-only behavior and
  script fallback behavior unchanged.
- Change usage, long help, examples, colored human headings, errors, and no-op summaries
  to say `bob task-status-setter`. Keep options alphabetized and preserve every public
  short alias as required by the CLI contract. Test that top-level help lists only the
  canonical name, the old alias still dispatches without materializing script assets,
  and help reached through either spelling presents the canonical usage.
- Rename the command documentation to `docs/task-status-setter.md` and update README
  links, installation smoke tests, `justfile` checks, examples, environment-variable
  prose, test names, and fixture labels that expose the old command name. Internal
  result/JSON field names such as `marked_next`, `marked_in_progress`, `marked_blocked`,
  and `unblocked` remain stable for consumers.
- Keep the current single transition planner, but add explicit integration coverage for
  a Blocked task with no dependency metadata in addition to closed, missing, duplicate,
  self, and mixed dependencies. Verify recovery to Ready, Next, and In Progress from the
  final Pomodoro graph, idempotent no-op reruns, dry-run non-mutation, terminal
  precedence, CRLF preservation, and atomic Blocked-registry guard failures.

### 2. Reopen affected dependents after Ctrl+Enter closes tasks

- In the linked `bob-plugins` source of truth, extend `task-status-cycler` with pure
  helpers that parse eligible `#task` lines outside fenced code, retain actual `id` and
  comma-separated `dependsOn` values in both supported Dataview field forms, and
  classify the installed Ready/Next/In-Progress/Blocked statuses as open. Reuse the
  plugin's existing line, editor, dependency-normalization, and Markdown-file helpers
  instead of introducing a separate identity convention.
- Build one post-close snapshot containing open IDs and Blocked dependents. Read every
  open Markdown editor from its live buffer and other Markdown files through the vault
  API so unsaved editor state is never overwritten or mistaken for disk state. Preserve
  Obsidian Tasks matching rules: duplicate IDs are aggregated with "any open wins,"
  missing IDs are ignored, Blocked itself is open, and only current `[?]` task lines are
  candidates for `[ ]` writes.
- Scope the plan to dependents whose `dependsOn` values intersect IDs belonging to tasks
  actually closed by this Ctrl+Enter action. Re-evaluate every candidate against the
  complete post-close open-ID index before changing it, then deduplicate same-file and
  repeated recursive targets so each checkbox is edited at most once.
- Funnel all Ctrl+Enter completion paths through one asynchronous post-close finalizer:
  direct active tasks, plain or embedded block-link targets, selected recursive trees,
  and whole Pomodoro completion batches. Run dependent recovery and existing
  closed-reference retirement in a defined serialized order so their vault-wide reads
  and writes cannot race each other. Preserve root-only reopening behavior when
  Ctrl+Enter is used on a Done target; that is a different direction from closing a
  dependency.
- Apply active-file changes through the editor while preserving cursor position, and
  cross-file changes with the existing snapshot/revalidation plus `vault.process`
  pattern. A stale line, unreadable note, concurrent edit, or failed write must be
  skipped and reported without clobbering content or rolling back tasks that were
  already closed; successful sibling reopenings remain valid. Re-running the finalizer
  is a no-op.
- Add focused helper and runtime tests for direct same-file and cross-file closes,
  bracket and parenthesized metadata, Ready recovery, multiple dependencies with one
  still open, duplicate IDs with one open target, missing IDs, self/cyclic metadata,
  absent IDs, already-active and terminal dependents, recursive/multi-target Pomodoro
  closure, open editor buffers, stale/concurrent snapshots, partial read/write failures,
  deduplication, serialization with link retirement, notices, and idempotency.
- Bump the `task-status-cycler` patch version and its repository README description.
  Deploy only that plugin from the linked repository via a selective `bob plugins sync`
  dry run followed by the guarded real sync; never edit the deployed vault copy
  directly. Verify its managed manifest and JavaScript match the source after
  deployment.

### 3. Keep the two recovery paths behaviorally aligned

- Factor or fixture the dependency truth table clearly enough that Rust and JavaScript
  tests assert the same open-ID, duplicate-ID, missing-ID, and remaining-blocker
  semantics even though only the CLI can compute Pomodoro rank.
- Document the timing distinction: Ctrl+Enter immediately changes an affected `[?]` task
  to Ready after its final open dependency closes, while `bob task-status-setter`
  performs the authoritative whole-vault pass and may choose Ready, Next, or In
  Progress. Document that closing a dependency never reopens terminal dependents and
  that the hidden old CLI spelling is compatibility-only.
- Preserve the installed Blocked Tasks status, dashboard queries, navigation `!`
  transaction, output JSON schema, directory/global-filter behavior, line endings, and
  existing reference retirement/restoration contracts.

## Validation and rollout

1. Run focused Rust unit and CLI integration tests for command dispatch/help and the
   full dependency recovery matrix, then run the repository-wide `just all` gate
   (`cargo fmt --check`, Clippy, and all Rust tests).
2. Exercise both CLI spellings against a controlled copied vault: use canonical
   `--dry-run --format json`, apply the canonical command, rerun for an already-in-sync
   result, then invoke the hidden alias to prove identical behavior and stable JSON.
   Explicitly cover no dependencies, all dependencies closed, another dependency still
   open, and Ready/Next/In-Progress recovery.
3. In `bob-plugins`, run the focused `task-status-cycler` test file, the complete
   `npm test` suite, and `npm run validate`. Confirm all manifest/version invariants
   pass.
4. Run the real-vault Tasks parity suite against a copied snapshot with a fixed date,
   then run `bob task-status-setter --dry-run --format json` against the live vault and
   review the proposed Blocked/unblocked rows without modifying task notes.
5. Preview and deploy only `task-status-cycler`, reload Obsidian, and smoke-test
   Ctrl+Enter once with a last closed dependency and once with another dependency still
   open. Confirm Ready recovery only in the first case, reference retirement still
   occurs, the cursor/editor remain stable, and a subsequent canonical CLI dry run
   proposes any Pomodoro-derived promotion correctly.
6. Finish with clean-diff audits and `git diff --check` in both source repositories;
   report any live-vault deployment files separately from source changes.

## Non-goals

- Do not reopen Done, Canceled, non-task, unknown, or unrelated tasks merely because
  they have no open dependencies.
- Do not persist a previous-status field or teach Ctrl+Enter to reconstruct the Pomodoro
  graph and guess Next/In Progress; authoritative ranking remains in
  `bob task-status-setter`.
- Do not change the `!` dependency-creation/unlink transaction, Tasks status
  registry/CSS, dashboard queries, or task dependency identity format.
- Do not remove the hidden `mark-next-tasks` compatibility alias in this change, add CLI
  flags, or rename stable JSON fields.
- Do not modify real vault task notes during validation; only the guarded plugin
  deployment is an intended live-vault write.
