---
tier: tale
title: Route demoted project tasks into a created Requirements section
goal:
  Project notes with only a Tasks section gain a Requirements section when a task is
  demoted, without changing any other task-toggle routing.
size: small
proposed_by: bbugyi200.athena.08j
create_time: 2026-09-09 20:00:23
status: wip
---

# Create a Requirements section when demoting the only project task section

## Goal

Extend the `bob-plugins` Task Status Cycler so the `<Ctrl+Shift+]>` / `<Ctrl+}>` “Toggle
Obsidian task” action handles a project note whose only Markdown section is `## Tasks`:
when a top-level task in that section is converted to a normal bullet, append a new
`## Requirements` section to the bottom of the note and move the converted bullet block
into it. Preserve the existing routing behavior everywhere else.

## Implementation

1. Open the linked `bob-plugins` repository with `sase repo open bob-plugins` and use
   the returned checkout path for all work. Re-read its `AGENTS.md` before editing.
2. In `plugins/task-status-cycler/main.js`, add a focused document-frontmatter predicate
   for the project marker. It must inspect only a closed YAML frontmatter block at the
   start of the active editor document, recognize the top-level scalar
   `type: [[project]]` with the quoted form used by existing project notes as well, and
   avoid treating nested keys, body text, comments, or malformed/unclosed frontmatter as
   a project marker.
3. Extend `getObsidianTaskToggleDocumentPlan`'s demotion route after the source
   list-item block has been removed and no following Markdown section can be found. Use
   the existing fence/frontmatter-aware heading discovery and activate the new fallback
   only when all of the following hold:
   - the document is a project note;
   - the source is a top-level dash task in the direct body of an H2 named `Tasks`;
   - `## Tasks` is the document's only real Markdown section; and
   - there is no next section to receive the demoted bullet under the existing rule.
4. For that fallback, append `## Requirements` at EOF with exactly one blank line
   between the preceding document content and the heading and exactly one blank line
   between the heading and the moved bullet. Move the full converted list-item block,
   including indented children/continuation lines, and return a normal move plan whose
   cursor targets the converted first line and is centered through the existing move
   application path. Normalize pre-existing trailing blank lines rather than
   accumulating extra blank space, and preserve the current handling of notes with or
   without a final newline.
5. Leave all existing cases unchanged: promotion into an existing `Tasks` section;
   demotion into an existing next section (including an existing `Requirements`
   section); in-place demotion in non-project notes, project notes with another section,
   malformed/non-project frontmatter, tasks outside the direct `Tasks` body, and list
   shapes that are not eligible for section routing.
6. In `scripts/test-task-status-cycler.cjs`, add focused helper/planner and command-path
   regressions covering:
   - unquoted and quoted project type markers;
   - the minimal project note with only `## Tasks`, proving the new H2 and both blank
     separators are created and the converted bullet is moved;
   - preservation of nested child lines, cursor placement, and move-plan application;
   - trailing blank lines and final-newline variants without duplicate whitespace;
   - existing `Requirements`/other next sections continuing to receive the bullet
     without creating another section; and
   - frontmatter/body text or fenced headings not being mistaken for real metadata or
     sections; and
   - non-project, malformed-frontmatter, extra-section, wrong-section, and non-top-level
     inputs retaining their prior results.
7. Bump `plugins/task-status-cycler/manifest.json` from `1.9.0` to `1.10.0` for the new
   behavior and update the Task Status Cycler row in `README.md` so the documented
   plugin version and `<Ctrl+Shift+]>` behavior match the implementation.

## Validation and deployment

1. Run the focused suite: `node --test scripts/test-task-status-cycler.cjs`.
2. Run the repository-wide checks: `npm test` and `npm run validate`.
3. Review the final diff to confirm the fallback is limited to the requested project
   shape and no generated or unrelated files changed.
4. Because `bob-plugins/AGENTS.md` requires deployment after any source change, run
   `bob plugins sync` and report its result together with the test results.

## Acceptance criteria

- Demoting a top-level `#task` from the sole `## Tasks` section of a note with
  `type: [[project]]` produces a bottom-of-file `## Requirements` section and places the
  converted plain bullet block below it after a blank line.
- The source block is removed from `## Tasks`, descendants remain attached, the cursor
  follows the moved bullet, and spacing is stable across trailing/final-newline cases.
- Notes that do not match the exact project/section preconditions keep the current
  promotion, next-section demotion, or in-place conversion behavior.
- Focused tests, the full plugin test suite, manifest validation, and `bob plugins sync`
  all succeed.
