---
tier: tale
title: Always-prompt Obsidian bullet routing
goal:
  Always show the section picker before promoting an eligible plain bullet in a note
  with Tasks, while preserving Tasks as the default and supporting typed section
  creation.
size: medium
proposed_by: bbugyi200.athena.08j.f0.f0.w0.f2
create_time: 2026-09-09 19:59:41
status: wip
---

# Always-prompt Obsidian bullet routing

## Goal

Extend Task Status Cycler's `Ctrl+Shift+]` / `Ctrl+}` “Toggle Obsidian task” flow so
every eligible top-level plain dash bullet in a Markdown note containing an exact
`Tasks` section opens the section picker before the note changes. Blank Enter continues
to select `Tasks`; typing a novel nonempty title and pressing Enter creates a new `##`
section and moves the still-plain bullet block there.

The implementation belongs in the linked `bob-plugins` repository. Open that repository
with `/sase_repo` before reading or changing it, and deploy the verified plugin through
the repository-required `bob plugins sync` workflow.

## Current baseline

- Promotion already prompts when a note has `Tasks` plus any other parsed heading. This
  includes the bullet's named source section, even though that source is not offered as
  a destination, so a note containing only `## Ideas` and `## Tasks` already prompts.
- The shared destination resolver already lets a promotion query reuse an existing
  heading case-insensitively or create a novel bottom-level `## <title>` section; a
  non-`Tasks` destination receives the original plain bullet block.
- A top-level preamble bullet in a `Tasks`-only note currently bypasses the picker and
  moves directly to `Tasks`. A plain bullet already in the direct body of `Tasks`
  currently converts in place without prompting. These are the always-prompt gaps.
- Promotion to `Tasks`, non-`Tasks` moves, full list-item subtree movement, safe removal
  of an emptied non-`Tasks` source section, duplicate heading identity, stale-picker
  protection, cursor restoration, CRLF handling, and final-newline preservation are
  already covered behavior to build on rather than reimplement.

## Required behavior

### Prompt eligibility and default

- For an eligible top-level plain dash bullet, find an exact Markdown section titled
  `Tasks`. If one exists, always return a promotion prompt before any editor mutation,
  whether the bullet is in the document preamble, in a non-`Tasks` section, or already
  in the direct body of `Tasks`, and regardless of how many other headings exist.
- Keep `Tasks` as the first selected promotion destination. Pressing Enter with an empty
  query must therefore select `Tasks` and apply the established checkbox, `#task`, and
  `[created:: <today>]` rewrite exactly once.
- Include the source `Tasks` heading as the default when the bullet is already in that
  section. Continue excluding a non-`Tasks` source heading as a pointless existing
  destination. Preserve document-order rows and stable identities for every other
  eligible heading, including duplicate titles.
- Preserve current fallbacks outside this scope: notes without an exact `Tasks` section,
  indented bullets, non-dash list markers, and other ineligible shapes keep their
  established in-place behavior. Proper task-to-bullet demotion behavior and its
  destination restrictions remain unchanged.

### Destination resolution

- Selecting the same `Tasks` section that already owns the bullet must promote it at its
  original location without removing/reinserting or reordering its list-item block.
  Preserve all child and continuation lines and restore the cursor relative to the
  promoted first line.
- Selecting a different exact `Tasks` destination moves the full block there and applies
  the promoted task syntax. Selecting an existing non-`Tasks` section moves the original
  block there without any checkbox or task metadata.
- When the user types a nonempty title, show the existing create action as the selected
  row when there is no normalized exact match. Enter must reuse a case-insensitive exact
  heading match; otherwise it must append `## <trimmed title>` at the bottom using the
  established blank-line and final-newline conventions, then move the original plain
  bullet and descendants into it.
- Apply the existing source cleanup to every actual move: remove a non-`Tasks` source
  section only when moving the block leaves it genuinely empty, and never delete
  `Tasks`, a preamble, a nonempty section, or the selected destination. Same-`Tasks`
  in-place promotion performs no section cleanup.
- Opening, filtering, navigating, cancelling, accepting, or failing a stale/missing
  destination must retain the current single-modal, no-write-before-accept, and
  double-submit guarantees.

## Implementation approach

1. In `plugins/task-status-cycler/main.js`, replace the promotion prompt's “has another
   heading” and “not already in Tasks” gates with one prompt construction path for every
   eligible plain bullet whenever exact `Tasks` exists. Keep the planner's early
   in-place fallbacks for notes without `Tasks` and unsupported list shapes.
2. Adjust promotion heading collection/allowance so the exact `Tasks` heading remains
   present and first even when it is also the source identity, while non-`Tasks` source
   headings stay excluded and demotion continues to reject `Tasks`.
3. Add an explicit same-source-`Tasks` resolution path that produces the promoted
   document/cursor result in place. Route all other existing and created destinations
   through the current removal, heading-remapping, insertion, and optional empty-source
   cleanup helpers.
4. Exercise the existing typed-create picker path from the newly prompted cases. Only
   change shared picker/model/modal code if needed to ensure empty-query Tasks
   selection, normalized exact reuse, novel-title creation, keyboard Enter, and
   status/placeholder text behave consistently for promotion as well as demotion.
5. Update `plugins/task-status-cycler/manifest.json` and the root `README.md` Task
   Status Cycler row to describe always-on promotion prompting and new-section creation,
   and bump the plugin's feature version consistently. Change `styles.css` only if an
   actual picker presentation change is required.

## Tests

Extend `scripts/test-task-status-cycler.cjs` with focused planner, resolver, and command
coverage for:

- the existing two-section case (`source` plus `Tasks`) continuing to prompt with
  `Tasks` selected and no write before acceptance;
- a preamble bullet in a `Tasks`-only note now prompting, with blank Enter moving and
  promoting it into `Tasks`;
- a plain bullet already inside `Tasks` now prompting, with blank Enter converting it in
  its original position without reordering siblings or descendants;
- choosing an existing non-`Tasks` section and typing a novel section name from both a
  `Tasks`-only/preamble case and a bullet already inside `Tasks`, keeping the moved
  block plain and creating exactly one bottom-level heading when appropriate;
- case-insensitive typed reuse, duplicate heading identities, source sections before and
  after the destination, safe empty-source deletion/retention, cursor centering, CRLF,
  and final-newline/no-final-newline documents;
- cancellation, repeated invocation, stale or missing destinations, proper demotions,
  notes without `Tasks`, indented bullets, and non-dash markers retaining their current
  behavior.

Run the focused suite first:

```bash
node --test scripts/test-task-status-cycler.cjs
```

Then run repository-wide verification:

```bash
npm test
npm run validate
git diff --check
```

Finally, from the opened `bob-plugins` checkout, deploy the verified plugin to the
vault:

```bash
bob plugins sync --no-pull -p task-status-cycler -r .
```

Confirm the sync reports the intended `main.js`, `manifest.json`, and only any genuinely
changed optional assets, without overwriting unrelated vault edits.

## Completion criteria

- Every eligible top-level plain bullet in a note with exact `Tasks` opens the picker,
  including preamble/`Tasks`-only and already-in-`Tasks` cases.
- Empty Enter selects `Tasks`; same-source `Tasks` promotion is in place, while other
  `Tasks` selections move and promote exactly once.
- Existing or newly created non-`Tasks` destinations receive an unchanged plain bullet
  subtree, and safe source-section cleanup remains correct.
- Existing demotion and ineligible-shape behavior remains green, focused and full
  validation passes, documentation/version metadata is current, and the plugin is synced
  to the vault.
