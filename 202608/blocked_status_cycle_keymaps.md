---
tier: tale
title:
  Cycle Blocked tasks with `<option+[>` / `<option+]>` and retire their future schedule
goal:
  The option-bracket task-status cycle reads `[?]` Blocked as a source status, leaving
  Blocked retires a strictly future task-level `scheduled` field, and a task that
  already keeps a Schedule Log gains one dated `🔓 unblocked by hand` entry.
size: medium
proposed_by: bbugyi200.athena.04w
create_time: 2026-09-09 19:59:44
status: wip
---

# Plan: option-bracket cycling out of Blocked (`[?]`)

## Goal

`<option+]>` and `<option+[>` currently do nothing on a `[?]` Blocked task: `[?]` is not
in the cycle ring, so the keypress is dead. Make Blocked a first-class **source** of the
cycle and clean up the state that made the task Blocked in the first place:

- `<option+]>` on `[?]` sets `[ ]` Ready; `<option+[>` on `[?]` sets `[-]` Cancelled.
  `[?]` is never written as a destination.
- Leaving `[?]` removes the task's own strictly future `[scheduled:: YYYY-MM-DD]` field,
  so a later `bob task-status-hooks` run does not immediately re-derive `[?]`.
- If (and only if) the task already owns a Schedule Log, prepend one newest-first entry
  recording the move from the removed date to today.

Every transition that does not start from `[?]` must stay byte-identical to today.

## Repository and ownership

The keymaps, the cycle ring, and the write paths all live in the linked `bob-plugins`
repository. Before any read or write, open it with:

```bash
sase repo open bob-plugins -r "Add Blocked-status support to the option-bracket task cycle"
```

Use only the path printed by that command. Never edit the deployed copies under the Bob
vault; `bob plugins sync` overwrites them.

Files in scope:

- `plugins/task-status-cycler/main.js` — cycle ring, pure helpers, write paths.
- `scripts/test-task-status-cycler.cjs` — pure-helper and plugin-flow regressions.
- `plugins/task-status-cycler/manifest.json` — feature release version.
- `README.md` (bob-plugins) — matching version and behavior summary.
- `docs/task-status-hooks.md` (this `bob-cli` workspace) — one short paragraph, beside
  the existing "Ctrl+Enter recovery in the Task Status Cycler plugin" paragraph, because
  that file already documents how the plugin's hand gestures interact with derived
  Blocked state.

No Rust change is needed. `bob task-status-hooks` keeps sole CLI ownership of derived
`[?]`; this feature only removes a blocking _reason_ so the CLI's own rules produce the
expected result on the next run.

## Current behavior

In `plugins/task-status-cycler/main.js`:

1. `FIXED_SYMBOLS = [" ", "/", "*", "x", "-"]` is both the ring and the set of writable
   destinations. `getStatusCycle()` returns it and
   `getAdjacentSymbol(symbol, direction)` indexes into it, returning `null` for any
   symbol outside it — so `[?]` yields `null`.
2. `isCyclableTaskStatus(taskStatus)` is `FIXED_SYMBOLS.includes(symbol)`. It gates the
   single-press path, the counted range loop, and transcluded-target resolution.
3. `handleCycleCommand` (registered as `cycle-task-status-forward` /
   `cycle-task-status-backward`) returns `false` when `getAdjacentSymbol` is `null`, so
   the single press is a no-op on `[?]`.
4. `getOptionBracketTaskCycleDirection` recognizes the physical Option+`[`/`]` keydown
   for the counted `N<option+]>` Vim path; `dispatchCountedTaskCycleEvent` refuses to
   consume the event unless `isCyclableTaskStatus` accepts the cursor line, and
   `cycleTaskStatusRange` walks a pre-write snapshot of line texts, editing each line in
   place.
5. Status writes go through `setActiveCheckboxStatus` (Tasks plugin command first, then
   `setActiveCheckboxStatusLocalWithTaskMetadata`) for the cursor line and
   `setCheckboxStatusLocalForLine` for other lines in a counted range. Transcluded
   targets are written by `replaceResolvedTranscludedTaskLine`, through the editor when
   the target is the active note and through `vault.process` otherwise.
6. The plugin has no `scheduled`-field or Schedule Log code at all, and every existing
   cycle edit is single-line, so nothing in the loop currently accounts for a changing
   line count.

`plugins/block-id-prompt/main.js` already implements the closest analogue — its `^^` /
`Ctrl+Shift+Enter` flows remove a uniquely future schedule and prepend a
`🍅 pulled into today's Pomodoro` entry to an existing log. Mirror its rules and its
helper shapes, but keep this plugin self-contained: plugins are deployed independently
and must not import each other's `main.js`.

## Behavioral contract

### Cycle shape

`[?]` joins the ring as a **source-only** slot immediately before `[ ]`:

```text
reading order:  ?  ->  " "  ->  /  ->  *  ->  x  ->  -  -> (wrap)
```

| Current | `<option+]>` | `<option+[>` |
| ------- | ------------ | ------------ |
| `[?]`   | `[ ]`        | `[-]`        |
| `[ ]`   | `[/]`        | `[-]`        |
| `[-]`   | `[ ]`        | `[x]`        |

`?` is never a destination: forward from `[-]` still wraps to `[ ]`, and backward from
`[ ]` still reaches `[-]`. Every other transition keeps today's exact result. Implement
this by keeping `FIXED_SYMBOLS` as the destination set and adding a separate
source-reading ring (`["?", " ", "/", "*", "x", "-"]`); step through the source ring in
the requested direction and skip a landing on `?`, then reject any destination that is
not in `FIXED_SYMBOLS`. Do not hand-write `[?]`: `bob task-status-hooks` clears a
hand-set `[?]` that has no open dependency and no future schedule, so a writable `?`
would be an unstable destination.

Widen `isCyclableTaskStatus` to the source ring so the single press, the counted range,
and transcluded targets all accept `[?]`. Leave `isOpenDoneTaskStatus`,
`CLOSABLE_TASK_SYMBOLS`, `isTranscludedCompletionTraversableStatus`,
`isTranscludedReopenableStatus`, and `canForceTranscludedTaskStatus` untouched:
`Ctrl+Enter` completion must keep treating `[?]` as a status it does not traverse or
close.

### Scheduled-field retirement

When, and only when, the **source** status of a cycled task line is `[?]`, retire that
task's own future schedule. The rule is destination-independent — today only `[ ]` and
`[-]` are reachable in one press, but the same cleanup must apply to whatever the ring
later makes reachable. Cancelling or completing a Blocked task retires the date too, so
reopening it later does not silently re-block it.

Recognize the same task-level forms as `bob task-status-hooks` and `block-id-prompt`:
`[scheduled:: YYYY-MM-DD]` and `(scheduled:: YYYY-MM-DD)`, anywhere on the line, in any
field order, with optional whitespace after `::`. Require the exact four-digit
year/two-digit month/two-digit day shape and validate the real calendar date (reject
`2026-02-30`, accept `2028-02-29`). Compare against the runtime's **local** calendar
components, never `toISOString()`, and make the clock injectable for tests (add a
`getScheduleLogDateString()` instance hook alongside the existing
`getCompletionDateString()`, both returning `formatLocalDate()`).

Retire the field only when the line carries **exactly one** recognized `scheduled` field
whose value is valid and strictly later than today. Any other shape — two or more
recognized fields, a malformed value, or a today/past date — changes the status only and
leaves the rest of the line byte-identical, because none of those is proof that a date
is what blocked the task. Removal is positional: delete that one span and collapse only
the whitespace it exposed, so surviving tokens stay separated by exactly one space with
no trailing whitespace, and other Dataview fields, tags, prose, list/quote prefixes, and
the trailing `^block-id` are preserved.

Only lines matching the Tasks global filter (`#task`, via the existing
`lineMatchesTasksGlobalFilterText`) are eligible for retirement; a bare `[?]` checkbox
that is not a Tasks task still cycles, with no field or log changes.

A task blocked only by `[dependsOn:: ...]` has no field to retire. It still cycles to
`[ ]`/`[-]`, gets no Schedule Log entry, and may be re-derived `[?]` by the next
`bob task-status-hooks` run. That is intended; this gesture does not reinterpret
dependency metadata.

### Existing-only Schedule Log entry

Only when a field was actually retired, and only when the task already owns a
direct-child Schedule Log marker, prepend one entry. Never create a marker: the marker
is the task's opt-in, exactly as in `bob-navigation-hotkeys` and `block-id-prompt`.

Recognize all three currently supported marker spellings as a direct child of the task
line (a marker owned by a nested grandchild does not count):

- `- 🗓️ **SCHEDULE LOG**`
- `- **SCHEDULE LOG**`
- legacy `- **Schedule log:**`

Insert the new entry as the first child of the marker (newest first), reusing the
marker's list marker and an existing entry's indentation when the log already has
entries, otherwise one tab deeper than the marker. The exact entry is:

```markdown
- _<removed-date> → <local-today>_ — 🔓 unblocked by hand
```

For example, cycling this task forward on 2026-08-17:

```markdown
- [?] #task Ship it [priority:: medium] [scheduled:: 2026-08-20] ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-06 → 2026-08-20_ — waiting on the API review
```

becomes:

```markdown
- [ ] #task Ship it [priority:: medium] ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-20 → 2026-08-17_ — 🔓 unblocked by hand
    - _2026-08-06 → 2026-08-20_ — waiting on the API review
```

The `from` half is the date just removed and the `to` half is today, so the entry reads
as a reschedule _to today_ rather than as an unschedule; the leading 🔓 (`U+1F513`)
marks it as machine-written, joining 🎲, 🤷, and 🍅 in the vault's reason vocabulary.
Use `→` (`U+2192`), `—` (`U+2014`), and `*` emphasis: `*` is what
`bob-navigation-hotkeys` (the log's canonical owner), its `SCHEDULE_LOG_ENTRY_RE`, and
`bob-cli`'s `capture_schedule_log.rs` all use. (`block-id-prompt` writes `_` instead;
that divergence is cosmetic, has no parser today, and is deliberately not changed here —
worth a separate cleanup decision.)

The removed date is always strictly future, so the entry never records a no-op move and
no "unchanged date" suppression rule is needed. Preserve LF vs CRLF and whether the file
ends in a newline.

### Ordering, guards, and failure modes

Apply the status write **first** and the cleanup **second**, using the pre-write status
`?` as the only carried-over fact. This keeps today's status semantics byte-identical
(including the Tasks plugin command on the cursor line) and gives the safest failure
mode: if the buffer changes underneath the gesture, the worst outcome is a status change
whose schedule survived, which `bob task-status-hooks` reconciles on its next run. Never
leave the reverse — a retired schedule on a task that is still `[?]`.

Re-derive the cleanup from the live buffer/file content after the status write rather
than from stale coordinates: re-read the task line, re-run the single-future-field
search, and only then remove the span. If the live line no longer parses as a task line,
or no longer carries the same single future field, skip the cleanup silently.

Field removal and the log insertion for one task must be applied together, in that
order, with the insertion strictly below the task line so the task's own line number
never moves.

### Counted ranges and shifting line numbers

`cycleTaskStatusRange` reads a pre-write snapshot of line texts and assumes every edit
is in place. Inserting a log entry breaks that assumption, so make the mapping explicit:

- Discover the Schedule Log marker in the **snapshot** (its position relative to the
  task is not changed by any edit this gesture makes), and record each insertion
  position in snapshot coordinates.
- Map every snapshot index the loop uses for an editor coordinate — status writes, the
  transcluded-target line argument, and the final cursor restore — through
  `editorLine(i) = i + <number of recorded insertions at positions <= i>`. A plain
  running offset is wrong: a task's log entry is inserted _inside_ its own child block,
  which can sit below a nested child task that the loop has not visited yet.
- The cursor-line status write is the one call that delegates to the Tasks plugin
  command, which can in principle add a line of its own. Compare the editor line count
  before and after that call and record any added line at `startLine + 1`. (Today's code
  has the same latent assumption; recording the measured delta closes it for the range
  loop without changing any status semantics.)

`endLine` stays in snapshot coordinates: the gesture keeps operating on exactly the
lines the user counted, and inserted log entries are never themselves cycled.

### Transcluded targets

Widening `isCyclableTaskStatus` also makes `[?]` reachable through the existing
transcluded path, where `<option+]>` on a bullet whose sole content is `![[Note#^id]]`
cycles the _target_ task. Support it fully rather than half-way: after
`cycleResolvedTranscludedTaskTarget` writes the status of a target whose pre-write
status was `?`, apply the same retirement and log insertion to that target's file —
through the editor when the target lives in the active note, and through `vault.process`
otherwise, reusing `splitTextByLineEndings` so each line keeps its own ending and a new
entry line inherits the marker line's ending. Two sequential writes to the same file are
acceptable; each is independently idempotent and guarded by re-reading the live content.

### Notices

Stay silent on success. Every existing cycle write is silent, and these keymaps are
meant for fast repeated presses; the Schedule Log entry is the durable record of what
happened. Do not add success notices, and do not add new failure notices beyond the
existing console/notice behavior.

## Implementation steps

1. Add the source ring constant plus the `scheduled`-field and calendar-date helpers
   near the existing task-line parsing helpers. Return spans and validated values so
   removal is positional.
2. Add the Schedule Log constants (marker regex covering the three spellings, `*`
   emphasis, `→`, `—`, and the `🔓 unblocked by hand` reason) and the direct-child
   marker discovery, entry-prefix, and entry-format helpers. Comment that this grammar
   mirrors `bob-navigation-hotkeys/main.js` and `block-id-prompt/main.js` and must stay
   compatible, and reuse the existing `CHILD_BULLET_INDENT_UNIT` tab for the fallback
   indent.
3. Rework `getAdjacentSymbol` to read from the source ring, skip `?` as a destination,
   and still reject anything outside `FIXED_SYMBOLS`; add a `getSourceStatusCycle()`
   beside `getStatusCycle()` and widen `isCyclableTaskStatus`.
4. Add the pure planner that composes retirement and insertion for one task line —
   input: line texts, task line index, today's date string; output: the rewritten task
   line, the removed date, and the optional insertion index plus entry text, or `null`
   when nothing applies. Add a text-level wrapper for the vault path built on
   `splitTextByLineEndings` / `replaceLineInSourceText`.
5. Wire the three call sites: the single-press branch of `handleCycleCommand`, the local
   task branch of `cycleTaskStatusRange` (with the snapshot-index mapping above), and
   `cycleResolvedTranscludedTaskTarget`. Keep each status write exactly as it is today.
6. Export the new pure helpers through the existing `helpers` object for tests; do not
   expose runtime internals unnecessarily.

## Tests

Add focused `node:test` cases to `scripts/test-task-status-cycler.cjs`. The harness
already provides `createInMemoryObsidianApp`, `createTextEditor`, and the
`commands: { commands: {}, executeCommandById: () => false }` stub that forces the local
status-write fallback; use an injected date everywhere.

1. Ring: `?` forward → `" "`, `?` backward → `"-"`, plus a full table asserting every
   pre-existing transition is unchanged — especially `-` forward → `" "` and `" "`
   backward → `"-"` never landing on `?` — and that no direction ever returns `?`.
2. `isCyclableTaskStatus` accepts `?` and still rejects unknown symbols; `Ctrl+Enter`
   traversal/closing predicates still reject `?` (regression for the existing
   `[?] #task Custom` fixtures).
3. Field parsing: bracket and parenthesized forms, arbitrary field order, whitespace
   after `::`, leap day, impossible date, wrong-width values, and the strictly-future
   boundary (yesterday, today, tomorrow).
4. Removal preserves other inline fields, tags, prose, numbered/quoted list prefixes,
   and the trailing block ID; collapses exposed whitespace to one space; introduces no
   trailing whitespace; preserves CRLF and a missing final newline.
5. Log insertion: canonical, emoji-less, and legacy markers; a nested grandchild marker
   ignored; marker with no entries falls back to marker indent plus one tab; existing
   entries reuse the first entry's indentation; the marker's list marker is reused; the
   entry lands directly beneath the marker; and no marker is ever created.
6. Exact entry bytes for `*2026-08-20 → 2026-08-17* — 🔓 unblocked by hand`, asserting
   `U+2192`, `U+2014`, and `U+1F513` explicitly.
7. Status-only outcomes: no scheduled field; today/past date; two recognized fields; a
   malformed value; a `[?]` checkbox line that is not a `#task`. Each asserts the rest
   of the document is byte-identical and no entry was written.
8. Backward cycling `[?]` → `[-]` retires the date and logs, matching forward.
9. Single press through `handleCycleCommand` on a `[?]` task with and without a log,
   both with the Tasks command stubbed out and with a stub that reports success,
   asserting the final document and the preserved cursor.
10. Counted `N<option+]>` over a range containing a `[?]` task with a log, followed by
    further tasks, including one where the `[?]` task's child block contains a nested
    `#task` line above its marker: assert every intended line changed and no unintended
    one did.
11. Counted range where the stubbed cursor-line Tasks command inserts a line, asserting
    later lines in the range are still edited correctly.
12. Transcluded `<option+]>` on `![[Tasks#^blocked]]` for a target in another note
    (vault path) and in the active note (editor path): status changed, date retired,
    entry logged.
13. Idempotence: pressing again on the now-`[ ]` task performs the ordinary `[ ] → [/]`
    transition with no further field or log changes.

Run the focused suite first, then everything, from the opened `bob-plugins` root:

```bash
node --test scripts/test-task-status-cycler.cjs
npm test
npm run validate
```

## Release, documentation, and deployment

1. Bump `plugins/task-status-cycler/manifest.json` from `1.8.0` to `1.9.0` and extend
   its one-line description to mention cycling out of Blocked.
2. Update the Task Status Cycler row in bob-plugins `README.md` to `1.9.0` and state
   that `<option+]>`/`<option+[>` cycle a `[?]` task to Ready/Cancelled, retire a
   uniquely future `scheduled` date, and prepend a `🔓 unblocked by hand` entry to an
   existing Schedule Log without ever creating one.
3. In this `bob-cli` workspace, add a short paragraph to `docs/task-status-hooks.md`
   beside the existing Ctrl+Enter recovery paragraph: hand-unblocking with the
   option-bracket cycle retires the future schedule that caused the Blocked marker,
   which is what keeps this command from re-deriving `[?]`; a remaining open dependency,
   or a project-level `scheduled` frontmatter date that `bob projects sync`
   re-propagates, still re-blocks the task on the next run.
4. After tests and validation pass, preview and deploy only this plugin from the opened
   linked-repository root, as that repository's `AGENTS.md` requires:

   ```bash
   bob plugins sync -p task-status-cycler -r "$PWD" --dry-run
   bob plugins sync -p task-status-cycler -r "$PWD"
   ```

5. Report that Obsidian must reload the Task Status Cycler plugin (or restart) before
   the new runtime code is active.

## Known limitations to document, not fix

- A schedule propagated from a project note's `scheduled:` frontmatter comes back on the
  next `bob projects sync`, which re-blocks the task. `Ctrl+D` on the `^prj` task's
  `scheduled` property remains the way to unschedule a whole project.
- A task blocked by an open `[dependsOn:: ...]` target is re-derived `[?]` by the next
  `bob task-status-hooks` run.
- No pre-Blocked status is stored anywhere, so `<option+]>` lands on Ready rather than
  restoring whatever the task was before it became Blocked.

## Out of scope

- Writing `[?]` as a cycle destination, or any other hand-set Blocked gesture.
- Changing `bob task-status-hooks`, `bob projects sync`, `bob capture`, the
  `Ctrl+Shift+P` property picker, or the Schedule Log schema itself.
- Creating a Schedule Log for a task that opted out by keeping none.
- Removing today/past schedules, or repairing malformed or duplicated schedule metadata.
- Pruning or restoring Pomodoro links; unblocking a task does not re-link it to today.
- Reconciling `block-id-prompt`'s `_` entry emphasis with the canonical `*`.
