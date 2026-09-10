---
tier: tale
title: Show Requirements only as the Tasks-only demotion default
goal:
  The task-demotion picker lists every actual destination once and offers an implicit
  Requirements destination only when Tasks is the note's sole selectable section.
size: small
proposed_by: bbugyi200.athena.08j.f0.f0
create_time: 2026-09-09 20:00:00
status: wip
---

# Show Requirements only as the Tasks-only demotion default

## Goal

Fix Task Status Cycler's destination picker so it never renders a synthetic
`Requirements` choice beside the same existing section, and so `Requirements` is the
implicit blank-input default only when the note has no selectable section other than
`Tasks`. Keep the picker open for every eligible top-level task demotion from the direct
body of `Tasks`, including notes with exactly one alternative section, and preserve all
existing move safety, formatting, keyboard, pointer, and accessibility behavior.

## Interaction contract

- Treat every real, nonempty, non-`Tasks` Markdown heading already collected by the
  fence/frontmatter-aware parser as an existing destination. “No other sections” means
  that this selectable-heading collection is empty; do not reintroduce a project-note or
  H2-only special case.
- With blank or whitespace-only input and no existing destinations, show one selected
  `Requirements` row marked as a new H2 at the bottom. Immediate Enter creates
  `## Requirements` and moves the converted bullet block there, retaining the current
  Tasks-only convenience.
- With blank or whitespace-only input and one or more existing destinations, omit the
  synthetic Requirements row. Show the real headings once each in document order and
  select the first one, so immediate Enter chooses that section. An existing
  Requirements heading remains visible as an ordinary real destination—once—not as an
  extra default action.
- Use a conditional input placeholder and truthful status copy: advertise `Requirements`
  only in the Tasks-only state; when real destinations exist, invite the user to filter
  them or type a new section name without implying that blank Enter will create
  Requirements.
- With nonblank input, retain fuzzy filtering and new-H2 creation, but enforce one row
  per actionable destination. If the normalized typed title exactly matches existing
  headings, show those real heading rows (all distinct occurrences, with their existing
  depth/line metadata) ahead of other fuzzy matches and do not prepend a duplicate
  “reuse existing” row. If there is no exact match, show one create action followed by
  fuzzy existing matches. Continue rejecting a title that normalizes to `Tasks`.
- Preserve duplicate-title headings as distinct choices by source identity. Deduplicate
  presentation actions, not actual headings: two `## Notes` headings at different lines
  must still appear twice and route to the selected occurrence.
- Continue resetting selection to the first valid row when the query changes, clamping
  or wrapping navigation over the rows actually shown, accepting the selected row by
  Enter or click, and leaving the note untouched on Escape, stale state, or invalid
  submission.

## Implementation

1. Open the linked repository with
   `sase repo open bob-plugins -r "Implement the demotion picker Requirements default fix"`,
   use only the printed checkout path, and re-read its `AGENTS.md` before editing.

2. Refactor the pure picker-model helpers in `plugins/task-status-cycler/main.js` so row
   composition depends on both trimmed query state and whether selectable headings
   exist.
   - Separate the Tasks-only blank default from typed-title actions instead of always
     passing a blank query through `getDemotionEffectiveTitle()` as Requirements.
   - Build blank-query rows directly from existing headings when available; otherwise
     build the single Requirements create row.
   - For nonblank queries, detect normalized exact matches before composing rows. Reuse
     the existing heading rows for exact matches, order exact matches before remaining
     fuzzy matches without collapsing distinct identities, and add a create/invalid row
     only when no exact match exists.
   - Derive `selectedRow`, `canSubmit`, empty-state text, and status text from the final
     row set so keyboard acceptance and accessibility state cannot refer to a hidden or
     duplicated primary action. Keep destination resolution and stale-snapshot checks
     authoritative even though the UI avoids invalid duplicates.

3. Update `DemotionSectionPickerModal` rendering in the same file to consume the revised
   model cleanly.
   - Set the input placeholder from model/state context (`Requirements` only when no
     selectable existing destination exists; otherwise neutral filter/create guidance).
   - Keep stable, unique option IDs and the current `listbox`, `option`,
     `aria-selected`, `aria-activedescendant`, live status, focus, mouse, and keyboard
     behavior with a row list that may or may not contain a primary action.
   - Reuse the existing visual row types and stylesheet; no CSS change is expected
     unless a minimal adjustment is required by the final model.

4. Update the focused tests in `scripts/test-task-status-cycler.cjs` to encode the new
   default contract and guard against both reported and generalized duplicates.
   - Tasks-only blank/whitespace input has one selected Requirements-create row and
     immediate Enter still creates the correctly formatted bottom H2.
   - Tasks plus one ordinary section has only that existing row, and immediate Enter
     moves the bullet there.
   - Tasks plus an existing Requirements section shows Requirements once and reuses it;
     Tasks plus Requirements and other sections shows every actual section once in
     document order with no synthetic Requirements row.
   - Exact case/whitespace-normalized typed matches do not duplicate their heading row;
     multiple same-title heading identities remain separate and selectable, while exact
     matches precede looser fuzzy results.
   - A typed unmatched title still exposes one create action, fuzzy existing choices
     remain reachable, `Tasks` stays invalid, and selection wrap/clamp and click/Enter
     submission use the revised row counts.
   - Modal seam assertions cover the conditional placeholder/status text and confirm
     that opening, cancelling, stale-note handling, cursor restoration, and centering
     remain unchanged.

5. Release this behavior correction as Task Status Cycler `1.11.1`: bump
   `plugins/task-status-cycler/manifest.json` and the README plugin table, and revise
   the README description to say that blank Enter chooses the first real destination
   when sections exist and creates Requirements only for a Tasks-only note. Do not
   change unrelated plugin documentation or commands.

## Validation and deployment

1. Run `node --test scripts/test-task-status-cycler.cjs`.
2. Run `npm test` and `npm run validate`.
3. Review `git diff` and `git status --short` to confirm the changes are limited to the
   picker model/modal, focused tests, Task Status Cycler patch version, and its README
   description (plus CSS only if the implementation proves a minimal style adjustment is
   necessary).
4. Smoke-test the modal in Obsidian for Tasks-only, one ordinary destination, existing
   Requirements alone, Requirements plus another destination, an unmatched typed new
   title, an exact typed match, keyboard navigation, pointer selection, and Escape.
   Confirm each displayed row is unique unless the document genuinely contains duplicate
   heading occurrences.
5. Run `bob plugins sync -p task-status-cycler` as required by `bob-plugins/AGENTS.md`
   and report the copied files together with automated and manual validation results.

## Acceptance criteria

- A note with only `Tasks` offers one Requirements-create default, and immediate Enter
  creates the bottom `## Requirements` section and moves the demoted bullet block.
- A note with any real non-`Tasks` section shows no synthetic Requirements default;
  immediate Enter selects the first existing destination in document order.
- An existing Requirements section is rendered exactly once per actual heading
  occurrence, both initially and after an exact normalized search, and is reused rather
  than duplicated in the document.
- Users can still type a valid new section name, select any fuzzy-matching existing
  heading, distinguish genuine duplicate headings, and cannot choose `Tasks`.
- The picker remains safe, accessible, keyboard-first, pointer-friendly, and visually
  native; all automated checks pass and the updated plugin is synced to the vault.

## Non-goals

- Do not stop prompting merely because a note has exactly one destination.
- Do not change which Markdown headings qualify as sections or change bullet insertion
  and new-section formatting.
- Do not persist a last-used destination, reorder the document, rename headings, add a
  plugin setting, or alter promotion/out-of-Tasks demotion behavior.
