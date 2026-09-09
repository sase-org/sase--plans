---
tier: tale
title: Counted Obsidian task moves
goal:
  Ctrl+Shift+M safely moves the current task, or the current task plus the next N tasks
  when given an explicit Vim count, into a selected area or open project note.
create_time: 2026-09-09 19:53:09
status: wip
---

# Plan: Counted Obsidian task moves

## Context and intended behavior

`bob-navigation-hotkeys` already has the pieces needed for this workflow, but not the
move itself: counted Ctrl+Shift+P discovers the current real `#task` checkbox plus the
next N real tasks, project creation snapshots and removes task subtrees, task routing
understands `## Tasks` sections, and `ChildNotePickerModal` decorates area and project
notes with normalized type, project status, schedule, icon, color, search, and
accessibility metadata.

Add a `move-tasks-to-note` editor command with Ctrl+Shift+M as its default hotkey. A
bare invocation moves the current real Obsidian task. In Vim normal mode, an explicit
count continues to mean **additional task targets**: for example, `2<Ctrl+Shift+M>`
moves the current task plus the next two real tasks in document order. Discovery must
not wrap and must skip prose, ordinary bullets, dependency-only transclusions,
Pomodoros, YAML examples, and fenced examples, while retaining the existing support for
nested/quoted tasks and all checkbox statuses. If the note ends before N following tasks
are found, use the tasks that exist and expose the clamped actual count in the picker.

Moving a task means moving its complete contiguous list-item subtree, including
explanatory bullets, dependency navigation links, blank lines internal to the subtree,
and nested tasks. A selected nested task moves independently when its ancestor was not
selected. If counted targets overlap because a selected descendant is already inside an
earlier selected task's subtree, move that content exactly once and keep its original
nesting. Project lifecycle `#prj ... ^prj` tasks are structural controls rather than
portable work items: reject them as the starting task and skip them while finding later
movable targets.

The destination prompt must always be shown. It lists all area notes and open project
notes except the source file and known template files, sorted by path. Treat normalized
`wip` and `waiting` projects as open; exclude `done`, `canceled`/`cancelled`, missing,
or unknown project statuses. Each row must reuse the child-note presentation and search
logic so an area visibly reads as an area, a project visibly shows its normalized
status, and the existing icons, status colors, future-schedule badge, path, accessible
labels, and status search terms stay consistent. The modal subtitle should state the
selected task count, destination count, and any end-of-note clamping.

## Implementation design

### 1. Add the command and optional-count input path

- Register `move-tasks-to-note` in `plugins/bob-navigation-hotkeys/main.js` as an editor
  command with a default Ctrl+Shift+M hotkey so bare use works outside Vim normal mode
  without requiring a manual vault hotkey edit.
- Add a capture-phase KeyM handler beside the counted property/transclusion handlers. In
  a focused Vim-normal Markdown editor, consume the exact Ctrl+Shift+M chord, read and
  reset the pending Vim repeat, deduplicate window/document delivery, and open one move
  session. Use zero additional targets for a bare chord and the explicit repeat for a
  counted chord. Wrong modifiers, other modes, disabled Vim, and non-Markdown editors
  must fall through.
- Reuse the existing Markdown-context-aware counted task discovery rules, but layer on
  the movable-task restriction for `^prj`. Snapshot the source file path, live editor
  identity, cursor/viewport, requested versus actual target counts, and every captured
  task subtree before opening the modal. Invalid starting lines must produce a focused
  notice and must not open a destination picker.

### 2. Build a reusable typed-note destination picker

- Extract or share the row rendering used by `ChildNotePickerModal` instead of copying
  its area/project decoration. Continue to source metadata through
  `getFileChildNoteInfo`/`getChildNoteInfo`, `normalizeStatus`, the existing
  presentation constants, and `childNoteMatchesQuery`; keep the current child-note
  picker behavior unchanged.
- Add a pure destination collector/classifier over vault Markdown files. It should
  return only non-template areas and `wip`/`waiting` projects, exclude the active source
  path, attach the shared note info, and sort deterministically. Revalidate the selected
  file's live frontmatter and status at commit time rather than trusting a possibly
  stale metadata-cache menu entry.
- Implement a focused `TaskMoveDestinationPickerModal` on `FilteredPickerModal`. Always
  show it even for one result, render the shared badges and search highlights, prevent
  double submission, and keep the source context captured while the user filters the
  list.

### 3. Plan source removal and destination insertion without losing structure

- Add pure helpers that expand counted task-line snapshots into non-overlapping
  list-item subtree ranges. Use Markdown container/indent semantics rather than physical
  line distance, preserve each block's internal content and ordering, and remove
  disjoint source ranges from bottom to top with conservative blank-seam cleanup.
- Rebase each moved root out of its source indentation or blockquote container into a
  top-level destination task while preserving its list marker, checkbox, body, Dataview
  fields, trailing block ID, descendant relative indentation, and line endings. Do not
  carry unrelated lines between separately selected tasks.
- Insert moved blocks in source document order at the end of the destination's unfenced
  `## Tasks` direct body, before the next heading. Reuse the established
  section/blank-line rules and replace the standard project task placeholder when it is
  the only task. An open project with no valid `## Tasks` section is structurally
  invalid and must reject the move; for an area without that section, append a
  well-spaced `## Tasks` section so every eligible area remains a usable destination.
  Preserve CRLF/LF choice, terminal-newline state, surrounding headings, and unrelated
  content.
- Reject block-ID/identity collisions in the destination before writing. Preserve all
  ordinary task metadata. For each moved task with a block identity, recompute its
  canonical `[id:: ...]` value for the destination path and reject an ambiguous
  malformed identity rather than silently leaving it tied to the source note.

### 4. Keep task links and dependency identities valid across the move

- Build a preflight rewrite map for every moved trailing block ID: old source-qualified
  dependency ID to new destination-qualified ID, plus old source block link to the
  destination block link. Reuse the existing dependency-ID, `dependsOn`, backlink
  collection, and block-link rewriting primitives, extending them only where a
  path-preserving move needs richer source/destination paths than project creation
  currently supplies.
- Compose rewrites for the source remainder, destination content, and other affected
  Markdown files before mutation. Update `[dependsOn:: ...]`, managed dependency
  transclusions, embeds, and ordinary block links while preserving aliases and embeds.
  Within moved blocks, keep pathless links pathless when their target moves too; qualify
  a formerly same-file link back to the source when its target stays behind. References
  from content that stays in the source to a moved block must point to the destination.
- Detect duplicate destination block IDs and ambiguous or stale backlink originals
  during preflight. Abort with no writes rather than producing a move whose identity or
  navigation is known to be broken.

### 5. Commit the cross-file move with loss-safe ordering

- At selection time, reacquire live contents from open Markdown editors where present
  and cached vault contents otherwise. Verify the source editor/file is still active,
  every captured source block still matches exactly and is still a real movable task,
  the destination is still eligible, and every affected reference file still matches its
  planned preimage.
- A vault has no single transaction spanning multiple files, so use an explicit guarded
  commit protocol: write the destination and auxiliary reference rewrites first, and
  delete/rewrite the source last. Use one editor transaction per open editor and
  compare-and-transform `vault.process` callbacks for closed files. If any pre-source
  write fails, restore already-written files only when their exact postimage still
  matches; because source deletion has not occurred, the original task copy remains
  authoritative. If the final source transaction fails or becomes stale, perform the
  same guarded rollback. Report any rollback that cannot safely complete as a
  recoverable duplicate/repair condition, never as a successful move.
- On success, keep focus in the source note, place the caret on the next surviving
  logical line (or the nearest valid line), restore the viewport, and issue one concise
  notice naming the destination and moved selected-task count, including clamping when
  applicable. The source edit must be one undo group; destination edits made through an
  open editor should likewise be one transaction. No path should remove the source
  before a complete destination copy exists.

## Tests and verification

- Extend `scripts/test-navigation-hotkeys.cjs` with bare and explicit-count KeyM
  dispatch tests: exact additional-task semantics, Vim input reset, duplicate event
  suppression, bare/non-Vim command behavior, and fallthrough for insert or visual mode,
  wrong modifiers, unfocused editors, and invalid source lines.
- Cover task discovery and subtree capture for intervening non-tasks, every checkbox
  status, nested and quoted tasks, YAML/fenced pseudo-tasks, protected `^prj` tasks, no
  wrapping, end-of-note clamping, CRLF, child content, disjoint ranges, and overlapping
  parent/descendant targets without duplicated output.
- Test destination eligibility and menu metadata for areas, `wip` and `waiting`
  projects, terminal/unknown projects, source/template exclusion, deterministic sorting,
  status/type search, the existing icon/badge variants, and stale type/status rejection
  after the picker opens.
- Add pure insertion/removal regressions for a populated `## Tasks` section, placeholder
  replacement, project section absence rejection, area section creation, nested/quoted
  root rebasing, blank seams, headings, terminal newlines, CRLF, block-ID conflicts, and
  preservation of task statuses, properties, IDs, and children.
- Cover identity/reference migration across source, destination, and third-party notes:
  canonical `id` and `dependsOn` values, same-file and qualified block links,
  embeds/aliases, dependencies among two moved tasks, dependencies on a task left
  behind, and stale or ambiguous references.
- Exercise runtime success through both open-editor and closed-file destinations,
  verifying source cursor/viewport and per-editor transaction grouping. Inject
  destination, auxiliary-file, final-source, and rollback failures to prove the source
  copy is retained whenever the full move cannot complete and that a successful move
  leaves exactly one copy.
- Run the focused navigation test directly, then the complete `npm test` suite and
  `npm run validate` from the `bob-plugins` repository.

## Release and deployment

- Bump `plugins/bob-navigation-hotkeys/manifest.json` by one patch version and update
  the Bob Navigation Hotkeys row in `README.md` to document bare and `N<Ctrl+Shift+M>`
  task moves alongside counted property editing.
- After tests and manifest validation pass, deploy only `bob-navigation-hotkeys` with
  `bob plugins sync`, explicitly selecting the implementation checkout as the source and
  disabling its automatic pull. Verify deployed `main.js`, `manifest.json`, and
  `styles.css` are byte-for-byte identical to the source.

## Completion criteria

- Ctrl+Shift+M on a real task always opens a typed/status-aware destination picker;
  selecting an area or open project relocates the complete task subtree into its Tasks
  section without losing metadata or references.
- In Vim normal mode, `2<Ctrl+Shift+M>` moves the current task plus the next two movable
  real tasks, while a bare chord moves only the current task and all non-counted editing
  modes continue to work through the command hotkey.
- Terminal projects, malformed destinations, lifecycle tasks, collisions, stale
  source/target/reference snapshots, and write failures cannot silently produce data
  loss or be reported as successful moves.
- Focused and full automated tests, manifest validation, documentation/version updates,
  and source-to-vault plugin sync all pass.
