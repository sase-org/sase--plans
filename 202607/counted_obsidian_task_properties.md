---
tier: tale
title: Counted Obsidian task property editing
goal: "Ctrl+Shift+P accepts an explicit Vim count and safely applies the chosen Dataview
  property operation to the current task plus the next N real tasks.

  "
create_time: 2026-09-09 19:53:09
status: wip
---

# Plan: Counted Obsidian task property editing

## Context and intended behavior

The `bob-navigation-hotkeys:set-bullet-property` command currently opens
`BulletPropertyPickerModal` for one cursor bullet and commits property changes back to
only that line. The Bob vault already binds the command to Ctrl+Shift+P, while the
plugin already has capture-phase Vim integrations that recover explicit numeric prefixes
before Obsidian or CodeMirror consumes them.

Extend that command so an explicit normal-mode count means **additional task targets**:
`2<Ctrl+Shift+P>` operates on the current task and the next two real `#task` checkboxes
(three tasks total). Search downward in document order, without wrapping, and skip
intervening prose, plain bullets, dependency transclusions, Pomodoro checkboxes,
frontmatter examples, and fenced examples. Nested or quoted real tasks and every
supported task status remain eligible. If fewer than N following tasks exist, use the
targets that do exist and make the actual target count visible in the picker. A bare
Ctrl+Shift+P retains the existing single-bullet behavior in Vim, insert mode, and
non-Vim editing.

The motivating `sase.md` example must therefore select `^read-sase-beads`,
`^fix-fix-just`, and `^fix-gh-act-and-pub` while skipping the transclusion child between
the first two tasks.

## Implementation design

### 1. Capture and normalize the optional Vim count

- In `plugins/bob-navigation-hotkeys/main.js`, add a narrowly scoped, capture-phase
  Ctrl+Shift+P listener alongside the existing counted transclusion and task-jump
  listeners. Reuse the focused Markdown view, normal-mode detection, pending-repeat
  parsing, input-state reset, event deduplication, and plugin cleanup patterns already
  present in this plugin.
- Intercept only the exact chord in a focused Vim-normal Markdown editor when the repeat
  is explicit. Prevent the corresponding Obsidian hotkey dispatch, reset the consumed
  Vim input state, and open the property picker once with N recorded as the number of
  additional tasks. Bare chords, other modifiers, non-normal Vim modes, disabled Vim,
  and unrelated editors must fall through unchanged.
- Require a real `#task` checkbox as the starting line for counted mode. Consume an
  invalid counted invocation with a focused notice instead of degrading it into a
  one-bullet edit that could surprise the user.

### 2. Represent a counted task session explicitly

- Add a pure target-discovery helper that snapshots the current task and up to N
  following real task lines using the plugin's existing Markdown context and
  `isObsidianTaskAtLine` rules. Return requested and actual counts plus stable
  `{line, rawLine}` snapshots; do not treat the count as a physical line range.
- Pass this target set into `BulletPropertyPickerModal`. Keep a one-target session
  compatible with the current uncounted command, but show clear batch context (for
  example, “3 tasks”) in the modal subtitle/footer whenever a count was explicit.
- Aggregate property metadata across the target set so rows and value stages truthfully
  distinguish absent, common, and mixed current values. In particular, deletion should
  be available when a property exists on any target, and a mixed batch must not
  misleadingly mark one value as current.

### 3. Plan and apply each property operation as one guarded batch

- Introduce pure batch planning for setting and deleting inline Dataview fields. Reuse
  `upsertBulletProperty` and `deleteBulletProperty` per target so spacing, trailing
  block IDs, unrelated metadata, and idempotent no-ops retain their established
  behavior.
- Before any write, verify that the same Markdown file/editor is active and every target
  line still exactly matches its snapshot and is still a real task. If any target is
  stale, abort the entire operation with no partial edits. Build the complete result
  first, then apply it as one editor transaction (with a safe fallback) so the batch is
  one undo step and retains the caret, selection/viewport, line endings, and line-count
  invariants.
- Use the same target set for Enter/set and Ctrl+D/delete. Notices should report the
  property/value and changed task count, distinguish no-op targets, and state clearly
  when an end-of-note count was clamped or a stale batch was rejected.

### 4. Preserve special property semantics across the batch

- Resolve the selected property's target independently for every source task. Ordinary
  `scheduled` fields remain inline, while a `scheduled` operation on a valid `^prj`
  lifecycle task continues to use project frontmatter and the existing visibility
  reconciliation. Compose that project-level plan with the other target edits before the
  single write; never emit a duplicate inline `scheduled` field on `^prj` merely because
  the invocation was counted.
- Generalize the `local_task_id` (`dependsOn`) flow to counted source tasks instead of
  silently applying it only to the cursor task. Exclude every source task from
  dependency candidates, represent all/none/mixed link state in the picker, and make a
  selected dependency converge across the sources (remove it when every source already
  has it; otherwise add it wherever missing). Reuse one block-ID prompt per selected
  dependency target, but plan the `dependsOn` edits and managed navigation bullets for
  every source parent. Validate all source and dependency snapshots before committing,
  and apply target-ID normalization, parent fields, and navigation bullets atomically so
  a failure cannot leave only part of the counted group updated.
- Preserve the current uncounted dependency workflow and notices when the session has
  one source task.

## Tests and verification

- Extend `scripts/test-navigation-hotkeys.cjs` with target-discovery cases for explicit
  additional-task semantics, intervening child transclusions and ordinary bullets,
  nested/quoted tasks, YAML/fenced pseudo-tasks, every task status, CRLF, no wrap, and
  end-of-note clamping.
- Add physical-key dispatch coverage proving an explicit count opens exactly one counted
  picker and resets Vim state, while a bare chord, insert/visual mode, wrong modifiers,
  unfocused editors, and duplicate window/document delivery fall through or are ignored
  appropriately.
- Add deterministic batch regressions for the motivating three-task `scheduled` update
  one week from a fixed date; mixed insert/update/no-op and Ctrl+D deletion;
  preservation of `created`, block IDs, and intervening content; stale-snapshot
  all-or-nothing failure; one transaction/undo group; cursor and viewport stability; and
  CRLF preservation.
- Cover counted `dependsOn` add/remove/mixed convergence, exclusion of all source tasks,
  prompted block IDs, per-parent navigation bullets, and atomic stale failure. Cover a
  batch that encounters a `^prj` task to ensure scheduled frontmatter/visibility
  semantics compose correctly with ordinary inline targets.
- Run the focused navigation test directly, then the complete `npm test` suite and
  `npm run validate` from the `bob-plugins` repository.

## Release and deployment

- Bump `plugins/bob-navigation-hotkeys/manifest.json` by one patch version and update
  the Bob Navigation Hotkeys row in `README.md` to document `N<Ctrl+Shift+P>` as current
  plus the next N tasks. The existing vault `hotkeys.json` binding is sufficient; no
  keymap configuration change should be needed.
- After all tests and manifest validation pass, run
  `bob plugins sync -p bob-navigation-hotkeys` as required by the repository and verify
  that the deployed vault plugin matches the source manifest and `main.js`.

## Completion criteria

- From a real task in Vim normal mode, `2<Ctrl+Shift+P>` followed by `scheduled` and a
  date updates the current task plus the next two real tasks, even when non-task lines
  intervene.
- Bare Ctrl+Shift+P remains behaviorally unchanged.
- Every picker operation either updates its entire counted target set according to the
  existing property-specific rules or makes no changes; it never silently edits only the
  first task or leaves a partial batch.
- Automated tests, repository validation, version documentation, and the source-to-vault
  plugin sync all pass.
