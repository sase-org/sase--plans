---
tier: tale
title: Merge pomodoros with Ctrl+X in the entry move and rename picker
goal: Combine pomodoro names and append their bullets while keeping the current timed
  entry or selected future destination intact.
size: medium
proposed_by: bbugyi200.athena.0k5
status: done
---

# Merge pomodoros from the Ctrl+Shift+M entry picker

## Objective and scope

Add Ctrl+X to the existing pomodoro **entry** move/rename picker. After opening
Ctrl+Shift+M on a pomodoro header, the user can select another open pomodoro with
Ctrl+N/P or the arrow keys and press Ctrl+X to combine the two. The surviving pomodoro
keeps its existing bullets first, receives the absorbed pomodoro's bullets at the end,
and receives its name as a `+` suffix. When the source is the current timed pomodoro, it
survives and absorbs the selected future one.

This is a `tale`, size `medium`: one implementation agent can complete the pure planner,
picker integration, regression coverage, documentation, and deployment in one bounded
change. The work is in `bobs-org/bob-plugins`, not bob-cli's Rust capture or timer
interfaces. No separate phases or cross-repository contract changes are needed.

Preserve Enter/click's existing whole-entry move or rename behavior, counted invocation
behavior (counts ignored on entries), task moves, pomodoro sub-bullet moves, their
exact-duplicate suppression, and Ctrl+Shift+J/K reordering. Ctrl+X is an additional
action confined to the entry picker, not a global command or a shortcut on
task/sub-bullet pickers. Do not edit memory or live daily notes as part of
implementation.

## Repository access and existing implementation

Before reading or editing the plugin repository, use `/sase_repo` to open `bob-plugins`;
if the linked name is unavailable, open `gh:bobs-org/bob-plugins`. That fallback was
necessary during exploration. Use only the returned checkout path and read its
`AGENTS.md`. Its source is plain CommonJS `main.js`, with no compilation/bundling step.
Do not edit deployed plugin files directly. The repository instructions require
`bob plugins sync` after implementation changes.

Read the pomodoro glossary and Obsidian conventions using `/sase_memory_read` before
relying on their formats. Pomodoros are column-zero checkbox entries in `## Pomodoros`:
an open `()` placeholder is future/planned; an open time range is current; `[x]`, `[X]`,
and `[-]` are closed/cancelled. Reuse parsed state, not the note's date, wall-clock
comparisons, picker position, or name.

Relevant files, all relative to the opened plugin repository:

- `plugins/bob-navigation-hotkeys/main.js`: entry parsing and ownership in
  `parsePomodoroEntryLine`, `collectPomodoroEntries`, and `findPomodoroEntryContext`;
  child discovery in `discoverPomodoroEntryMoveTargets`; the shared pure
  `planPomodoroBulletMove`; suffix-only `planPomodoroEntryRename`;
  `isOpenPlaceholderPomodoroEntry` and `isOpenTimedPomodoroEntry`;
  `createPomodoroBulletMovePickerRows`; `FilteredPickerModal` and
  `PomodoroEntryMovePickerModal`; `openPomodoroEntryMovePicker` and
  `commitPomodoroEntryMoveSession`; transaction and focus helpers; exported `helpers`
  for tests.
- `scripts/test-navigation-hotkeys.cjs`: `TransactionEditor`,
  `createPomodoroMovePickerHarness`, `createPomodoroEntryMoveSession`, existing
  picker/session tests, and extensive entry-move/rename/reorder fixtures.
- `plugins/bob-navigation-hotkeys/manifest.json` and `README.md`: plugin version,
  feature description, and validation documentation. At exploration the plugin version
  was `1.34.0`; choose the next feature version from the implementation checkout's
  actual version rather than overwriting intervening changes.
- `package.json`: `npm test` runs all plugin regression suites; `npm run validate`
  checks manifests and JavaScript syntax.

Current behavior: choosing an existing row with Enter moves every discovered source
child subtree into that row and deletes the source header. It does not combine names or
preserve the current timed source. A typed novel name creates a rename row. The shared
move planner handles child indentation, append order, empty placeholders, shifted
destination lines, and protection against deleting unmoved content, but it suppresses
exact duplicate single-line bullets. The entry picker closes before committing so the
active Markdown editor can be validated; its parent uses an `opening` latch to reject
duplicate activation. Those mechanisms are useful, but the existing move action is not
the new merge contract.

Exploration baseline: the existing navigation suite filtered by
`Pomodoro|pomodoro|move picker closes` passed 116 tests, with 256 unrelated tests
skipped and no failures. No implementation changes were made while planning.

## Behavior contract

Use **invoked entry** for the header on which Ctrl+Shift+M was pressed, **selected
entry** for the highlighted existing picker row, **survivor** for the entry retained,
and **absorbed entry** for the entry removed.

| Invoked entry        | Selected entry       | Survivor | Absorbed entry |
| -------------------- | -------------------- | -------- | -------------- |
| Future placeholder A | Future placeholder B | B        | A              |
| Future placeholder A | Current timed B      | B        | A              |
| Current timed A      | Future placeholder B | A        | B              |

The survivor stays in its original ledger position relative to untouched entries.
Preserve its exact list/checkbox prefix and parenthetical text, including the time
range, emphasis, and `[t:: ...]` metadata. Never transfer or combine time ranges. Delete
only the absorbed entry and the child content transferred from it. Resolve both entries
by line identity in the captured document, not by name; duplicate names are valid
distinct choices.

The new action accepts two distinct open current/future entries in the same note's
`## Pomodoros` section. Reject self-selection, missing entries, closed/cancelled
participants, and two timed participants. The last case has no specified time-range
combination rule. Existing Enter-based operations on closed sources remain available.
Restrict the merge action rather than changing which rows the existing move/rename
picker exposes.

### Bullets and names

1. The survivor's existing child content remains first. Append all substantive child
   bullet subtrees from the absorbed entry in original order, preserving descendants,
   links, markers, continuation content, and relative indentation. Rebase the
   transferred subtree indentation using the existing helper. Do not merge by task-link
   identity or collapse duplicates for Ctrl+X: the requested operation appends the
   absorbed bullets, including exact duplicates. Existing Enter and sub-bullet moves
   retain their duplicate policy.
2. Use the existing whole-entry rules for empty placeholder bullets: a bare empty child
   with no descendants can be dropped; replace the survivor's lone empty child
   placeholder when actual bullets arrive. Empty absorbed entries still contribute their
   names and are deleted. Empty survivors are valid. Preserve an empty survivor's
   placeholder if no real bullets arrive.
3. Normalize nonempty names with the existing uppercase, whitespace-collapse,
   em-dash-removal convention, then join them in survivor-first order using literal `+`.
   Both named gives `SURVIVOR + ABSORBED`; only one named gives that name with no
   dangling plus; neither named leaves the survivor unnamed. Equal names intentionally
   produce `NAME + NAME`; an existing plus expression stays intact and is appended to
   without deduplication or rearrangement.
4. Retain the existing 48-character name limit. Validate each present name and the final
   combined name before applying anything. Reject invalid or over-length results with a
   specific notice suggesting shortening a name; never silently truncate, drop a name,
   or partially merge. A combined name of exactly 48 characters is valid. An unchanged
   name does not prevent an otherwise valid merge from transferring bullets and deleting
   the absorbed entry.
5. Preserve the current whole-entry no-content-loss guard: if the absorbed entry owns
   nonblank content outside movable subtrees and disposable empty bullets, refuse the
   entire merge. Likewise reject unsupported trailing header content that cannot be
   safely represented as a name on either participant; do not discard an unrecognized
   suffix while deleting its entry. Preserve LF/CRLF, terminal-newline state,
   surrounding headings, unrelated entries, and existing blank-seam handling.

Example: invoking on future `REVIEW` and selecting current `BUILD` yields the following,
with the original current time range intact. Invoking on current `BUILD` and selecting
future `REVIEW` yields the same result.

```markdown
## Pomodoros

- [ ] (**0920-0950** [t:: 30m]) — BUILD + REVIEW
  - existing build bullet
  - first review bullet
    - review detail
  - second review bullet
```

For two future entries, invoking on `BUILD` and selecting `REVIEW` instead produces
`REVIEW + BUILD`, with REVIEW's bullets first at REVIEW's ledger slot.

### Interaction and atomicity

- Keep Ctrl+N/P and arrows selecting the visible filtered rows. Ctrl+X acts on exactly
  `visibleItems[selectedIndex]`, including when a rename row precedes the intended
  existing row. It must never infer a destination from query text or silently pick the
  first matching name.
- In the entry picker, consume Ctrl+X using `preventDefault` and `stopPropagation` so it
  cannot cut the query or reach the editor. Follow the existing `isCtrlKey` modifier
  convention. Only an eligible existing row may commit. On a rename/invalid row or an
  empty list, keep the picker open, change neither query nor note, and provide a concise
  selection hint. An ineligible participant pair similarly produces a useful refusal
  rather than another action. Other picker classes retain their existing Ctrl+X
  behavior.
- Add entry-specific footer hints for Ctrl+X Merge and Enter Move/rename, alongside
  navigation and Escape. Update the title to include merge. Explain the direction in the
  subtitle or concise adjacent help: a timed invoked entry absorbs the selection; a
  future invoked entry is appended to the selection. Preserve the ignored-count
  indication.
- Use the same `opening` latch for merge and ordinary activation. Reject key
  repeat/reentrant submissions and prevent overlapping Ctrl+X/Enter commits. Validate
  the row/pair before closing, then close before the guarded commit and release the
  shared active-picker slot using the existing lifecycle.
- Revalidate active file path, active editor identity, and complete editor text against
  the frozen session before writing. Derive the entire result in pure code and apply it
  once through `applyEditorContentTransaction`. Merging and renaming must undo together;
  do not run separate editor move and rename writes. On stale sessions or invalid plans,
  write nothing.
- Focus the resulting survivor header, with cursor column clamped to its line length,
  and restore the source editor using the existing context helper. Report the actual
  direction and final name, such as `Merged REVIEW into BUILD + REVIEW`; for unnamed
  entries use existing pomodoro-position labels. An empty absorbed entry is still
  reported as a merge rather than a plain deletion. All failure notices refer to merging
  and make clear that nothing changed.

## Implementation steps

1. Add a pure `planPomodoroEntryMerge` helper near the entry move/rename planners.
   Accept invoked/selected entry line identities and captured raw headers, rediscover
   both from the supplied full document, validate the pair, and resolve
   survivor/absorbed roles using the table. Discover the absorbed entry's children after
   resolving direction: the invoked session's targets are wrong when the current entry
   absorbs the selected future one. Reuse the whole-entry move machinery with an
   explicit duplicate-preservation option whose default leaves existing behavior
   unchanged. Obtain the survivor's relocated line from that pure result, and compose
   the suffix rename in memory. Handle the both-unnamed and unchanged-name cases without
   passing an empty name to the rename validator or discarding the move result. If
   either stage fails, return the original input as `after`. Return clear metadata
   identifying both original roles, the survivor's final line/name, and transferred
   bullet count. Export the planner through `helpers`.
2. Add a merge-specific guarded commit method and entry-modal Ctrl+X handler. Reuse
   narrowly shared session validation/transaction helpers where useful, but keep Enter
   behavior explicit and avoid a general picker refactor. Ensure modal row eligibility,
   the pure planner, and commit validation agree. Add merge labels, direction help,
   notices, focus placement, and the shared activation guard described above. Keep real
   Ctrl+X keystroke handling in the entry subclass, not the global keymap or
   `FilteredPickerModal`.
3. Add meaningful pure-planner and modal/commit regressions in the existing test file.
   Update the README's navigation-plugin description with the Ctrl+X flow, direction,
   name order, append behavior, and name-length refusal; update its test-coverage
   paragraph and bump the plugin feature version in both the manifest and README table.
   Run the validation below, review the final diff, and deploy only this plugin from the
   actual opened checkout.

## Verification and acceptance

Use explicit before/after Markdown expectations for core cases, not just expectations
computed by the same helper under test.

- Exercise all three direction-table rows and both earlier/later survivor positions
  where applicable, with unequal subtree sizes. Assert exactly one participant remains;
  other entries retain their order; the survivor prefix and compact/colon-format time
  metadata are unchanged; bullets and names are survivor-first. Cover supported open
  checkbox variants through existing parser semantics.
- Cover nested descendants, continuation text, mixed source/destination indent styles,
  duplicate single-line bullets, duplicate-name entries, and the uncovered-content
  refusal. Test empty absorbed/survivor/both-empty entries, lone placeholders, one/both
  unnamed entries, equal names, and repeated merges.
- Cover normalized names, existing plus names, exact 48-character results, over-limit
  results, and unsupported header suffixes. Invalid cases return unchanged input.
  Exercise self, missing, closed/cancelled, two-timed, and stale raw-header cases
  without mutation.
- Include first/middle/last absorbed entries and LF/CRLF with and without a terminal
  newline; assert neighboring headings and unrelated note bytes are preserved apart from
  established deletion-seam cleanup.
- Drive the modal key handler with Ctrl+N/P followed by Ctrl+X, including a filtered
  list whose first row is Rename and whose highlighted existing row is not index zero.
  Assert the selected row is the one merged, event suppression, one guarded
  transaction/undo group, survivor cursor placement, correct notice, close-before-commit
  order, active-picker cleanup, and reentrancy protection. Test no-result/rename/invalid
  selections, Escape, content drift in either participant, active-file/editor changes,
  and a failed commit with no successful-merge notice.
- Keep explicit regressions that Enter still moves or renames, counted entry invocation
  ignores the count, and Ctrl+X does not gain merge behavior in the task or sub-bullet
  picker. Existing duplicate-suppression tests must continue to pass while new merge
  tests retain duplicates.

From the opened plugin repository, run:

```sh
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

After these pass, use bob-cli's documented deployment interface, replacing
`<opened-plugin-repo>` with the path returned by `sase repo open`:

```sh
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo <opened-plugin-repo> --format json
```

Inspect the dry-run and actual sync output. Use `--no-pull` and explicit `--repo` so
deployment uses the implementation checkout. Confirm the navigation plugin is reported
synced; sync can exit zero while skipping dirty deployed files, so an exit code alone is
insufficient. Do not add `--force` automatically if the dirty-file guard refuses. Report
that concrete limitation if it occurs.

If an interactive Obsidian session is available, reload the plugin and use a disposable
note to check Ctrl+Shift+M, Ctrl+N/P, Ctrl+X for a future-to-future merge and a
current/future merge in both invocation directions, then undo each once. Confirm query
text is not cut, time metadata survives, and Escape leaves the note unchanged. If
interactive access is unavailable, report that the automated suite covers the behavior
and that the Obsidian smoke test remains unverified; do not claim to have performed it.

Completion requires the new action and its regressions, passing test and manifest
checks, updated feature documentation/version, and attempted targeted deployment with
accurately reported status. Final reporting should identify the plugin repository as the
changed repository and distinguish verified deployment from any environment-dependent
limitation.
