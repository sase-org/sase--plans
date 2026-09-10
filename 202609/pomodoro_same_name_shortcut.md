---
tier: tale
title: Create a same-name Pomodoro destination with +
goal:
  Typing + in the Pomodoro sub-bullet move picker creates a fresh destination with the
  source name and moves the selected bullets into it.
size: small
proposed_by: bbugyi200.athena.0gp
create_time: 2026-09-09 20:00:40
status: wip
---

# Create a same-name Pomodoro destination with `+`

## Objective and scope

When `Ctrl+Shift+M` opens the Pomodoro sub-bullet move picker, entering `+` and
confirming should create a fresh Pomodoro with the source Pomodoro's name and move the
selected bullets into it. This includes counted moves. In the user's terminology these
named Pomodoro entries are the sections; this feature does not create Markdown headings.

This is a small tale for one implementation agent: the existing move engine already
creates duplicate names correctly. The change belongs to one picker in the linked
`bob-plugins` repository, with regression coverage and help text. No changes to
bob-cli's Rust code, capture syntax, or bob-mac-capture are needed.

## Repository access and implementation context

Before reading or editing plugin files, use `/sase_repo` and run:

```sh
sase repo open bob-plugins -r "Implement the approved same-name Pomodoro move shortcut"
```

Use the returned repository path for all plugin work, and read its `AGENTS.md`. All
plugin paths below are relative to that repository. Its plugins are plain CommonJS;
`main.js` is source, with no bundle/build step. Edit the source repo and deploy through
`bob plugins sync`, as its instructions require.

Relevant code in `plugins/bob-navigation-hotkeys/main.js`:

- `openTaskMoveOrPomodoroBulletPicker` dispatches sub-bullets to
  `PomodoroBulletMovePickerModal`, Pomodoro entry lines to the separate
  `PomodoroEntryMovePickerModal`, and other tasks to the task move picker.
- `createPomodoroBulletMovePickerRows(entries, sourceEntryLine, rawQuery, options)`
  serves both Pomodoro modes. Its default mode is `bullets`; `options.mode === "entry"`
  selects move/rename behavior. It suppresses creation when a normalized name matches an
  open entry, including the source in bullet mode. A literal `+` currently produces a
  new row named `+`.
- Picker sessions already contain every entry, including closed entries, and the source
  entry's exact line and parsed name. Find the source by its line, not by matching names
  or assuming it is the current timed/open entry.
- `normalizePomodoroName` strips em dashes, collapses whitespace, uppercases, and
  rejects empty or over-length names. Use this existing name convention.
- `commitPomodoroBulletMoveSession` accepts a `{ kind: "new", name }` row, checks that
  the editor and content still match the session, then uses `planPomodoroBulletMove` and
  one editor transaction.
- `planPomodoroBulletMove` already accepts `{ kind: "new", name: "FOCUS" }` even when
  other entries are named FOCUS. It inserts below the source's remaining subtree, or at
  the source's former position if the source empties and is deleted. Existing logic
  preserves descendants, order, indentation, line endings, cursor placement, and
  counted-move semantics.

Planning verification reproduced the bug without editing source: typing FOCUS selected
the existing FOCUS entry, typing `+` proposed `New Pomodoro +`, and passing a new FOCUS
destination directly to the planner correctly split a source into a fresh FOCUS entry
while leaving another FOCUS entry alone. The existing Pomodoro-focused navigation tests
passed on the inspected tree.

## Behavior contract

1. Only a raw query whose trimmed value is exactly `+` invokes the shortcut in the
   Pomodoro sub-bullet picker. Surrounding whitespace is accepted. Names such as `C++`,
   `A+B`, `++`, and `+FOCUS` keep their ordinary name/filter meaning; do not add special
   handling to the shared name normalizer.
2. Resolve and normalize the source entry's name. Return one selectable `kind: "new"`
   row displaying the resolved name, using the existing row shape and creation
   presentation. This row must appear even when the source or any other open entry has
   that name. Do not also offer entries whose names or preview text merely contain `+`
   for this special query.
3. An unnamed source, missing source, or source name that fails existing validation
   yields one non-actionable `kind: "invalid"` row with an explanatory message. For an
   unnamed source, explain that `+` needs a named source and that the user can type a
   new name. Do not invent a name from its time range, ordinal, preview, or the shortcut
   token.
4. On confirmation, move the selected bullet and its descendants, plus the requested
   next siblings, through the existing creation transaction. Copy the name only: the
   destination is a fresh open `- [ ] () — NAME` placeholder, with no copied status or
   time range. Named closed/cancelled sources that are already eligible for sub-bullet
   moves support the shortcut too.
5. Preserve source cleanup: a partial move leaves its source; moving its final owned
   content deletes it. Creation then uses the former source position. A new same-name
   destination must never be merged into another entry simply because their names match.
6. This alias applies to the sub-bullet creation workflow. Preserve the separate
   entry-line move/rename mode, ordinary task movement, blank-query destinations, normal
   typed-name collision behavior, cancellation, and stale session guards. Do not broaden
   whole-entry movement to create destinations.

For example, with the cursor on `move`, enter `+`:

```markdown
## Pomodoros

- [ ] () — FOCUS
  - keep
  - move
- [ ] () — FOCUS
  - existing
```

The result is:

```markdown
## Pomodoros

- [ ] () — FOCUS
  - keep
- [ ] () — FOCUS
  - move
- [ ] () — FOCUS
  - existing
```

## Implementation

1. Add a focused branch for bullet mode and trimmed `+` in
   `createPomodoroBulletMovePickerRows`, before ordinary query normalization,
   name-collision suppression, and existing-destination filtering. Resolve the source
   from the complete input entries, validate its name, and return the single frozen
   new/invalid row using existing conventions. Keep the planner and transaction
   interface unchanged unless a demonstrated wiring need requires a minimal adjustment.
2. Update `PomodoroBulletMovePickerModal` help/placeholder text to advertise `+` for a
   new Pomodoro with the same name. The row must show the actual destination name, and
   the existing success notice should say, for example,
   `Moved 1 bullet to new Pomodoro FOCUS`.
3. Extend `scripts/test-navigation-hotkeys.cjs`, reusing its helper exports,
   `createPomodoroMovePickerHarness`, and transaction editor. Test observable results
   with explicit Markdown expectations rather than calculating the expected document by
   calling the same planner under test:
   - `+` and whitespace-padded `+` select a fresh source-name destination, including
     when another open same-name entry exists. A preview containing `+` must not compete
     with the creation row.
   - Source lookup uses the selected entry, including a named closed source;
     normalization of the copied name follows existing conventions.
   - Unnamed, missing, and invalid-name sources produce non-actionable rows; selecting
     an invalid row does not write to the editor.
   - Partial and counted moves into the generated row preserve bullet order and nested
     descendants, leave other same-name entries untouched, and correctly delete a fully
     emptied source. Include CRLF preservation in a representative same-name move
     fixture.
   - A modal/session test obtains the `+` row from the actual picker and confirms it
     through the picker callback. Assert the expected document, one transaction/undo
     group, first moved bullet cursor, and resolved-name notice. Cover cancellation and
     stale-content rejection without writes.
   - Non-special plus-containing names, normal existing-name lookup, and entry-mode
     move/rename behavior retain their existing semantics.
4. Document the shortcut and named-source requirement in `README.md` beside the existing
   Pomodoro move behavior. Bump the plugin's minor version from the current `1.32.0` to
   `1.33.0` in its manifest and README table; if the approved worker opens a newer
   version, choose the next minor version.

## Validation and deployment

Run the focused navigation tests during implementation, then the repository checks once
the change is complete:

```sh
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

Inspect the final diff for scope and ensure changes are limited to the plugin, its
tests, manifest, and README. The required behavior is complete when `+` creates the
correct new destination through the real picker path, the expected edge cases are
covered, and the checks pass.

After validation, deploy this plugin from the repository path returned by
`sase repo open`. Bind that path to a task-specific variable such as
`pomodoro_plugins_repo` and run:

```sh
bob plugins sync --repo "$pomodoro_plugins_repo" --no-pull --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --repo "$pomodoro_plugins_repo" --no-pull --plugin bob-navigation-hotkeys
```

Confirm the sync actually copied the changed files or found them up to date; report any
dirty-vault skip accurately and preserve the default dirty-file guard. The explicit
repository path ensures deployment uses this implementation. If an Obsidian GUI is
available, reload the plugin and smoke-test `Ctrl+Shift+M`, `+`, Enter in a disposable
example with two same-name entries; check the new entry, source cleanup, and one-step
Undo. Otherwise report GUI verification as unperformed while giving the automated and
sync results.
