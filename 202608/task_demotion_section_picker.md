---
tier: tale
title: Choose a destination section when demoting an Obsidian task
goal:
  Demoting a task from a Tasks section opens a polished, safe destination picker that
  can reuse any existing section or create a new H2, defaulting blank Enter to
  Requirements.
size: medium
proposed_by: bbugyi200.athena.08j.f0
create_time: 2026-09-09 20:00:32
status: wip
---

# Choose a destination section when demoting an Obsidian task

## Goal

Replace Task Status Cycler's automatic destination choice for eligible demotions from a
`Tasks` section with a polished section picker. When `<Ctrl+Shift+]>` / `<Ctrl+}>`
converts a top-level Obsidian task in the direct body of a real `Tasks` section into a
normal bullet, the command must leave the note untouched until the user chooses any
existing non-`Tasks` Markdown section or enters a new section name. Pressing Enter
immediately, without typing or navigating, must choose `Requirements`: reuse an existing
matching section or append a new `## Requirements` section at the bottom.

The interaction must be keyboard-first, pointer-friendly, accessible, visually native to
Obsidian, safe in the face of editor changes while the picker is open, and preserve the
current toggle behavior outside this exact demotion path.

## Product and interaction contract

### When the picker opens

- Open it for a demotion only when the source is an eligible top-level dash task in the
  direct body of a fence/frontmatter-aware Markdown heading named exactly `Tasks`. Apply
  this to project and non-project notes alike and regardless of whether the note has
  zero, one, or many other sections.
- Do not mutate the task or move its block when the picker opens. Obsidian's command
  availability check must remain side-effect-free; only the command execution path may
  open the picker.
- Keep all other routes unchanged: promotion into `Tasks`, demotion outside the direct
  `Tasks` body, indented/non-dash list shapes, and any case that currently converts or
  routes without this picker.

### Destination choices

- Offer every real, nonempty Markdown heading in document order except headings named
  `Tasks`. Reuse the existing parser so YAML/frontmatter text and headings inside fenced
  code blocks are never offered. Preserve heading depth as display metadata rather than
  limiting the picker to H2, because the existing router treats headings at every depth
  as sections.
- Represent each existing destination by stable document identity (at least its source
  heading line, depth, and title), not title alone. Do not coalesce duplicate titles;
  show enough context, such as heading level and one-based line number, to let the user
  choose the intended occurrence reliably.
- Put a primary destination action before the existing-section results. With a blank
  input it is `Requirements`; if a heading whose normalized title matches `Requirements`
  already exists, mark the action as an existing destination and reuse it, otherwise
  mark it as a new H2. Thus untouched Enter has one deterministic result and never
  creates a duplicate Requirements heading.
- As the user types, make that primary action create the trimmed title. If the entered
  title matches an existing heading case-insensitively after whitespace normalization,
  resolve to the existing heading instead of creating a duplicate. Reject a new title
  that normalizes to `Tasks`, with a concise inline explanation. A whitespace-only input
  is the same as blank and therefore resolves to `Requirements`.
- Keep existing destinations visible as fuzzy/filterable results beneath the primary
  action. Up/Down and `Ctrl+P`/`Ctrl+N` move the selection, Enter accepts it, Escape
  cancels, and clicking a row accepts that row. Navigation from the untouched initial
  state must make it easy to select even the sole existing non-`Tasks` section.

### Presentation

- Add a dedicated, namespaced modal headed with a clear action such as “Move bullet to
  section,” a short subtitle, and a compact preview of the bullet being demoted.
- Use a focused single-line search/name field with `Requirements` as its placeholder, a
  visually distinct “Create section” versus “Existing section” primary row, heading
  depth/location metadata on existing rows, an obvious selected state, a useful empty
  state, and a small keyboard-hint footer.
- Add `plugins/task-status-cycler/styles.css` using Obsidian theme variables, responsive
  width/height constraints, light/dark-theme-safe contrast, clear focus states, and
  reduced-motion handling. Keep every selector under a Task Status Cycler-specific
  namespace so the plugin cannot restyle unrelated Obsidian UI.
- Give the input and results appropriate labels, `listbox`/`option` semantics,
  `aria-selected`, live validation/status text, and mouse handlers that do not steal
  focus before selection. Focus the input after opening and return the editor cursor to
  the moved bullet after a successful choice.

## Implementation

1. Open the linked repository with
   `sase repo open bob-plugins -r "Implement the task-demotion section picker"`, use
   only the printed checkout path, and re-read its `AGENTS.md` before editing.

2. Refactor the pure Markdown routing layer in `plugins/task-status-cycler/main.js`
   around an explicit destination intent.
   - Add helpers that identify whether the active demotion originates in the direct body
     of a real `Tasks` section and collect selectable headings with stable identities.
   - Let the initial document planner return a prompt intent for that route rather than
     precomputing an automatic next-section move. Keep enough immutable source data to
     render the modal and later validate the submission.
   - Add a pure resolver that accepts either an existing-heading identity or a new H2
     title and returns the final move plan. It must support destinations before or after
     the source, headings at different depths, and duplicate titles without selecting
     the wrong occurrence.
   - For an existing destination, retain the current direct-body insertion contract:
     append after its last top-level bullet block, preserving that block's descendants,
     or insert the first bullet after exactly one heading separator when the section has
     none.
   - For a new destination, remove the source block first, collapse surplus trailing
     blank lines, append exactly one blank line, `## <title>`, one blank line, and the
     converted bullet block. Preserve the note's current final-newline convention.
   - In both cases move the complete source list-item block, preserve descendants and
     continuation lines verbatim, keep the converted first-line cursor column, and use
     the existing move application/centering path.
   - Remove the now-obsolete project-frontmatter-only Requirements fallback and its
     dedicated predicates. Project metadata is no longer a routing prerequisite; the new
     prompt contract subsumes that special case.

3. Add pure picker-state helpers so matching and keyboard behavior can be exhaustively
   tested without a browser DOM.
   - Centralize blank-to-`Requirements`, trimming/normalization, case-insensitive
     existing-title reuse, `Tasks` rejection, fuzzy filtering, selected-index clamping,
     and wraparound navigation.
   - Keep displayed spelling from the selected existing heading or trimmed user input;
     normalization is for matching and validation, not for silently renaming existing
     headings.

4. Implement a Task Status Cycler-namespaced `Modal` in
   `plugins/task-status-cycler/main.js` and connect it to the command lifecycle.
   - During command execution, open at most one picker for a prompt intent and return
     success without applying the demotion. Cancellation or dismissal must clear modal
     state and make no editor write.
   - Capture the editor/view/file identity, cursor, source lines, and source block when
     opening. Before submission, require the same note/editor and an unchanged document
     snapshot. If it is stale, close without writing and show a `Notice` explaining that
     the note changed and the command should be retried. Never apply a full-document
     replacement computed from stale text.
   - Resolve the selected destination against the validated snapshot, apply exactly one
     move, close the modal as completed, restore the cursor on the moved bullet, and
     center it. Guard Enter/click submission against double application.
   - Update the test Obsidian stubs for any newly imported modal/icon APIs and expose
     only the minimal pure helpers needed by the focused suite.

5. Add `plugins/task-status-cycler/styles.css` for the modal described above. Update the
   repository-layout and optional-CSS notes in `README.md` so Task Status Cycler's new
   stylesheet is documented and will remain part of the `bob plugins sync` contract.

6. Bump `plugins/task-status-cycler/manifest.json` from `1.10.0` to `1.11.0` and update
   the Task Status Cycler README row. Document that `<Ctrl+Shift+]>` / `<Ctrl+}>`
   prompts for an existing or new destination when demoting from `Tasks`, with blank
   Enter defaulting to `Requirements`.

## Tests

Extend `scripts/test-task-status-cycler.cjs` with focused regression coverage for:

- prompt eligibility in project and non-project notes with zero, one, and many
  non-`Tasks` sections, proving the command always prompts and performs no write before
  selection;
- exclusion of YAML/frontmatter, fenced, empty-title, and `Tasks` headings while real
  headings of different depths remain ordered and selectable;
- duplicate heading titles remaining distinct by identity and routing to the selected
  occurrence, including targets before and after the source block;
- untouched Enter resolving to a new `## Requirements` or reusing an existing
  case/whitespace-equivalent Requirements heading; whitespace-only input, typed new
  names, fuzzy existing-section selection, exact existing-title reuse, and rejection of
  `Tasks` as a new destination;
- keyboard selection/index behavior, Enter acceptance, Escape/cancel semantics, and
  double-submit protection through pure state and command/modal seams;
- existing-section insertion with and without bullets, and new-section formatting with
  extra trailing blanks, LF/CRLF-backed editors where supported by the existing harness,
  and final-newline variants;
- preservation of nested children/continuations, cursor placement, and centering for
  both existing and newly created destinations;
- stale document, changed active file/editor, or missing selected heading causing a
  notice and no write; and
- unchanged promotion and out-of-`Tasks` demotion behavior, plus existing ineligible
  indented/star/other list shapes.

Keep modal rendering thin over the tested state model. Add the lightest practical DOM
stub assertions for input focus, accessible roles/labels, row activation, and class
names; do not introduce a large UI framework solely for these tests.

## Validation and deployment

1. Run `node --test scripts/test-task-status-cycler.cjs`.
2. Run `npm test` and `npm run validate`.
3. Review the final diff and `git status --short` to confirm only Task Status Cycler,
   its focused tests, and the intended repository documentation changed.
4. Manually load the plugin in Obsidian and exercise keyboard and pointer selection in
   both light and dark themes and at a narrow modal width:
   - untouched Enter creates/reuses Requirements;
   - a note with exactly one other section still prompts and can select it;
   - typed names create a bottom H2 while exact matches reuse an existing section;
   - duplicate titles can be distinguished;
   - Escape leaves the note byte-for-byte unchanged; and
   - changing the note while the picker is open produces the stale-state notice and no
     destructive rewrite.
5. Because `bob-plugins/AGENTS.md` requires deployment after source changes, run
   `bob plugins sync` and report the copied `main.js`, `manifest.json`, and new
   `styles.css` together with the automated and manual results.

## Acceptance criteria

- Every eligible top-level task demotion from the direct body of `Tasks` opens the
  destination picker, even when there are zero or exactly one alternative sections, and
  the note is unchanged until a destination is accepted.
- Immediate Enter deterministically moves the converted bullet block to an existing
  Requirements section or creates a bottom `## Requirements` section with stable blank
  lines and no duplicate heading.
- Any real non-`Tasks` heading can be chosen, and a valid typed name creates a new
  bottom H2; duplicate titles, fenced text, and concurrent editor changes cannot route
  or overwrite content incorrectly.
- The modal is complete by keyboard, usable by pointer, accessible, responsive, and
  visually consistent with Obsidian in light and dark themes.
- Promotions and demotions outside the prompt's exact eligibility contract retain their
  current behavior, all automated checks pass, and the synced vault copy includes the
  new stylesheet.

## Non-goals

- Do not prompt when promoting a normal bullet into `Tasks`.
- Do not broaden section routing to indented tasks or list markers that are currently
  ineligible.
- Do not rename, reorder, or otherwise edit existing headings, and do not persist a
  last-used destination; untouched Enter must continue to default to Requirements.
- Do not add plugin settings or change unrelated Task Status Cycler commands.
