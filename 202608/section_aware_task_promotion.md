---
tier: tale
title: Section-aware Obsidian task promotion
goal:
  Route plain bullets through section selection, default promotion to Tasks, and safely
  remove source sections that become empty in any eligible Obsidian note.
size: medium
proposed_by: bbugyi200.athena.08j.f0.f0.w0
create_time: 2026-09-09 20:00:29
status: wip
---

# Section-aware Obsidian task promotion

## Goal

Extend Task Status Cycler's `Ctrl+Shift+]` / `Ctrl+}` “Toggle Obsidian task” flow so a
top-level plain bullet in any Markdown note with an exact `Tasks` section can be routed
through a section picker, with `Tasks` as the default destination and with safe cleanup
of a source section that becomes empty.

The implementation lives in the linked `bob-plugins` repository. Open that repository
with `/sase_repo` before reading or changing it, and deploy the completed plugin with
the repository-required `bob plugins sync` workflow.

## Existing behavior to preserve

- A promotable top-level dash bullet outside `Tasks` is converted to
  `- [ ] #task ... [created::<today>]` and moved with its full list-item subtree into
  the existing exact `Tasks` section.
- A proper Obsidian task in the direct body of `Tasks` opens the existing destination
  picker, becomes a plain bullet, and moves to an existing or newly created non-`Tasks`
  section. The blank-selection default for this demotion remains the first visible real
  destination, or creation of `## Requirements` when `Tasks` is the only section.
- A proper task outside `Tasks` keeps its current next-section demotion behavior.
- Indented bullets, non-dash list markers, bullets already in the direct `Tasks` body,
  notes without `Tasks`, YAML/frontmatter headings, fenced headings, duplicate headings,
  nested list-item children/continuations, cursor restoration, CRLF content,
  final-newline state, stale-picker protection, and creation of a new `##` section
  retain their established semantics unless a requirement below explicitly changes them.

## Required behavior

### Promotion routing and conversion

- For a top-level plain dash bullet outside `Tasks`, first find an exact Markdown
  section titled `Tasks`; do not require project frontmatter or a project-note path.
- If the note has no real Markdown heading besides `Tasks`, keep the direct, no-prompt
  move to `Tasks` and convert the moved bullet to an Obsidian task.
- If the note has one or more real headings besides `Tasks`, open the section
  destination picker before writing. This condition includes the bullet's own source
  section, even when it would be excluded as a useful destination.
- Put the existing `Tasks` heading first and selected by default. Pressing Enter without
  typing or navigating therefore moves the bullet to `Tasks` and applies the normal
  checkbox, `#task`, and `[created::]` promotion rewrite.
- Allow choosing any other eligible existing section or typing a new nonempty section
  title. A non-`Tasks` destination moves the original bullet block without adding a
  checkbox, `#task`, `[created::]`, or any other task metadata. A typed title matching
  an existing heading case-insensitively reuses the existing heading; otherwise create a
  bottom-level `## <title>` section using the existing blank-line/final-newline
  conventions.
- Do not offer the source heading itself as an existing destination. Preserve duplicate
  destination headings as distinct rows and continue routing to the chosen occurrence by
  stable heading identity.
- Keep `Tasks` a valid destination only for a promotion prompt. It remains invalid for
  the existing `Tasks`-task demotion prompt, so demotion cannot recreate or target
  `Tasks`.

### Source-section cleanup

- When a plain bullet is moved to `Tasks`, another existing section, or a newly created
  section, determine the source section from the original parsed Markdown headings.
- Move the entire top-level list-item block, including nested children and continuation
  lines, before evaluating cleanup.
- Delete the source heading and its section-owned blank padding only when removal of
  that block leaves the source section with no substantive direct-body content and no
  child heading. This is the safe definition of “last bullet”: retain the section if it
  still owns another top-level bullet/checklist block, prose, a code fence, or any
  nested subsection.
- Never delete `Tasks`, never delete a heading for a bullet in the document preamble,
  and never delete the destination when source and destination identities could overlap.
- Normalize only the seam created by removal so surrounding sections retain their
  content, heading depth/order, blank-line style, CRLF behavior, and final-newline
  state.

## Implementation approach

1. In `plugins/task-status-cycler/main.js`, separate the current demotion-specific
   picker policy from the reusable picker UI/state. Carry an explicit prompt kind
   (`promotion` or `demotion`), default destination, allowed heading identities, source
   heading identity, original versus promoted moved blocks, and preview text in the
   document plan. Rename demotion-only helpers and instance fields where that improves
   clarity, but retain the existing modal styling hooks unless CSS changes are genuinely
   required.
2. Generalize heading collection, picker-row construction, status text, and destination
   resolution so promotion can include `Tasks` as its first/default existing row while
   demotion continues excluding/rejecting it. The no-query Enter path must resolve the
   selected visible row, and typed exact/fuzzy/create behavior must remain deterministic
   and accessible by keyboard or pointer.
3. Add source-section discovery and a reusable removal result that can remove the
   list-item block, optionally remove a proven-empty non-`Tasks` source section, and map
   original line/heading identities through every removal and blank-seam edit. Use it
   for all promotion destinations so targets before or after the source and duplicate
   headings continue to resolve correctly.
4. Build the promotion prompt only for an eligible top-level bullet in a note containing
   `Tasks` plus at least one other parsed heading. Resolve a `Tasks` selection with the
   promoted block and every non-`Tasks` selection with the untouched original block.
   Keep the current direct promotion, in-place fallbacks, and all task-demotion routes
   intact.
5. Update command/modal lifecycle names and stale/missing-destination notices as needed
   so one picker can serve both flows, still opens only once, performs no mutation until
   acceptance, rejects a changed editor/file snapshot, restores and centers the cursor
   on the moved bullet, and clears cleanly on accept, cancel, or failure.
6. Update the Task Status Cycler manifest description/version and the root `README.md`
   plugin table entry to document promotion prompting, Enter-default `Tasks` conversion,
   plain-bullet routing to non-`Tasks` sections, empty source-section deletion, and
   ordinary-note support. Change `plugins/task-status-cycler/styles.css` only if the
   generalized UI requires a visual adjustment.

## Tests

Extend `scripts/test-task-status-cycler.cjs` with focused helper and command/modal tests
covering:

- direct promotion with no non-`Tasks` headings;
- promotion prompts in both project and ordinary Markdown notes containing `Tasks`,
  including a note with no project frontmatter;
- untouched Enter selecting `Tasks`, converting exactly once, moving the full child
  subtree, and preserving cursor/centering behavior;
- selecting an existing non-`Tasks` section and creating a new section while leaving the
  bullet and descendants non-task-formatted;
- case-insensitive typed reuse, duplicate heading identity, fuzzy rows, keyboard
  navigation, pointer activation, cancel/double-submit, stale document/editor, and
  missing-heading handling for both prompt kinds;
- the source section being removed when its moved bullet block is its only substantive
  content, for destinations before and after it and for both `Tasks` and non-`Tasks`
  selections;
- the source section being retained when another bullet, prose, fenced content, or child
  heading remains, plus no cleanup for preamble bullets and no deletion of `Tasks`
  during demotion;
- CRLF documents, extra/trailing blank lines, final newline versus no final newline,
  nested children/continuations, headings hidden in YAML or fences, bullets already
  inside `Tasks`, notes without `Tasks`, indented bullets, and non-dash list markers.

Run the focused suite first:

```bash
node --test scripts/test-task-status-cycler.cjs
```

Then run repository-wide verification:

```bash
npm test
npm run validate
```

Finally, from the opened `bob-plugins` repository, deploy the verified plugin to the
vault as required by that repository's agent instructions:

```bash
bob plugins sync --no-pull -p task-status-cycler -r .
```

Confirm the sync reports the intended `main.js`, `manifest.json`, and (only if changed)
`styles.css` deployment without overwriting unrelated vault edits.

## Completion criteria

- The keymap prompts exactly when a promotable top-level bullet has `Tasks` plus another
  real section available in the note.
- Blank Enter always selects `Tasks` for promotion and produces the established Obsidian
  task syntax; any non-`Tasks` selection moves an unchanged plain bullet block.
- A now-empty non-`Tasks` source section is removed safely, while every nonempty,
  nested, preamble, destination, and `Tasks` section is preserved.
- The behavior is covered for ordinary notes as well as project notes, existing demotion
  behavior remains green, all focused and repository-wide checks pass,
  documentation/version metadata are current, and the plugin is synced to the vault.
