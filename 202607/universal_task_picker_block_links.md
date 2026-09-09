---
tier: tale
title:
  Make the `^^` task picker work on every block link, and restrict `[*]` promotion to
  sole-content Pomodoro sub-bullets
goal:
  The `^^` task picker opens and completes correctly for every wiki block-link form in
  every source context, while a linked task is promoted to `[*]` only when the block
  link is the entire content of a Pomodoro sub-bullet.
create_time: 2026-09-09 19:53:26
status: wip
---

# Plan: Universal `^^` task picker with sole-content Pomodoro promotion

## Summary

Two changes to the `block-id-prompt` plugin in the linked `bob-plugins` repository:

1. **Broaden the `^^` task picker to every block-link form.** The picker must open for
   any wiki link the caret marker is attached to, regardless of where that link lives in
   the note and regardless of whether the link already carries a block ID or a heading
   subpath. Three forms silently fail today, and one of them completes into a corrupt
   link.
2. **Narrow `[*]` (Next) promotion.** A selected `[ ]` task is promoted to `[*]` only
   when the source block link is the _entire_ content of a Pomodoro sub-bullet. Today
   any Pomodoro descendant line promotes, even when the link is embedded in prose on
   that line.

The source of truth is the linked `bob-plugins` repository (open it with the
`/sase_repo` skill). The deployed vault copy under `~/bob/.obsidian/plugins/` must never
be hand-edited; it is produced by `bob plugins sync`.

## Diagnosis / root cause

### Premise check: there is no source-context gate on the picker

Read this first — it changes what "make it work on any block link" means in code.

`inspectActiveEditor` (`plugins/block-id-prompt/main.js`) looks for the task-picker
marker **before** any context test, and the only conditions it applies are: single
cursor, active Markdown file, and "not inside a fenced code block". The Pomodoro
predicate (`sourceLineIsPomodoroSubBullet`) is consulted _only_ by
`shouldPromoteTaskToNext`, after a task has been chosen, purely to decide the `[*]`
status write. The repository's own
`.sase/sdd/plans/202607/fix_obsidian_task_block_link_carets.md` recorded the same
finding: "The problem is not an intentional source-context restriction."

This was verified empirically by driving the exported `findTaskPickerMarkerNearCursor`
helper over representative lines. The marker is discovered identically in prose,
headings, blockquotes, table cells, top-level bullets, nested bullets, and ordinary
`#task` lines. So the picker is **not** restricted to sub-bullets of tasks or Pomodoros,
and no gate needs to be removed.

What _is_ real is that marker discovery is keyed on the shape of the link's destination,
and three shapes lose:

| Typed line                                                                      | Today                                                        | Cause                                                                                                                                                                                              |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[[note#^abc^^]]` — `^^` typed inside the destination of an existing block link | no picker, nothing happens                                   | `parseTaskPickerPosition` rejects any `targetText` still containing `^`, and `parseTrailingTaskPickerMarker` never strips the pre-existing `#^abc`                                                 |
| `[[note#Heading]]^^` — heading link                                             | picker opens, then **completes into `[[note#Heading#^id]]`** | `parseRapidTaskPickerMarker` keeps the heading in `targetText`; `completeTaskSourceLink` appends `CANONICAL_BLOCK_LINK_PREFIX` (`#^`) to it, producing a two-subpath link that resolves to nothing |
| `[[note#Heading^^]]` — heading link, staged form                                | no picker                                                    | same `targetText.includes("^")`/subpath problem as row 1                                                                                                                                           |

The first row is the likely source of the reported "only works in task/Pomodoro
sub-bullets" impression: freshly typed links (`[[note]]` then `^^`) are the normal flow
inside Pomodoro and task sub-bullets and they work everywhere, whereas re-targeting an
_existing_ block link by typing `^^` at the end of its destination fails everywhere.

The root cause of all three rows is that destination normalization is duplicated and
inconsistent. `parseRapidTaskPickerMarker` routes through
`getCaretCompletionDestination`, which only strips a _trailing block reference_
(`parseTrailingBlockDestination`) and leaves heading subpaths intact.
`parseTrailingTaskPickerMarker` does not normalize at all — it just drops one caret
(`destination.slice(0, -1)`).

### `[*]` promotion is looser than intended

```js
function shouldPromoteTaskToNext(source, task) {
  return Boolean(task && task.status === " " && sourceLineIsPomodoroSubBullet(source));
}
```

`sourceLineIsPomodoroSubBullet` only asks "is this line an indented list item descending
from a Pomodoro entry inside `## Pomodoros`". So
`- working on [[note]]^^ (need to ask Bob first)` under a Pomodoro promotes the selected
task to `[*]`, which is not the intent: only a dedicated block-link sub-bullet
represents "this is the task for this Pomodoro".

The predicate already exists twice in the codebase in near-identical form:

- `isDedicatedLinkBullet(lines, reference, removalRange)` in `block-id-prompt` — used by
  `planFuturePomodoroLinkCleanup` to decide whether a matched reference owns its whole
  bullet.
- `getBareNonEmbeddedBlockLinkTargetFromListItem` /
  `getMoveOnlyPomodoroBlockLinkFromListItem` in `task-status-cycler` — same "link spans
  the entire trimmed list-item body" test.

The fix is to extract the shared bounds computation inside `block-id-prompt` and reuse
it for the promotion gate rather than adding a third copy.

## Implementation

All edits are in the linked `bob-plugins` repository, `plugins/block-id-prompt/`.

### 1. Unify destination normalization

Replace the body of `getCaretCompletionDestination(destination)` so it strips **any**
subpath (heading or block), not just a trailing block reference, reusing the existing
`stripLinkSubpath` helper:

- compute `path = stripLinkSubpath(destination)`;
- if `path === destination` (no subpath at all) return `path` unchanged;
- otherwise return `` `${path}#` ``.

`parseTrailingBlockDestination` stays where it is — it is still used by
`parseTrailingAtBlockRenameMarker` — but it is no longer part of caret completion.

This alone fixes the single-caret staging step, so `[[note#Heading]]^` now relocates to
`[[note#^]]` instead of `[[note#Heading^]]`, and `[[note#^abc]]^` keeps relocating to
`[[note#^]]` exactly as before.

### 2. Route both task-picker parsers through that normalizer

**`parseTrailingTaskPickerMarker`** — after determining `markerLength` (3 for `#^^`, 2
for `^^`):

```js
const rawBase = destination.slice(0, destination.length - markerLength);
if (rawBase.endsWith("^")) {
  return null; // three or more carets must not trigger the picker
}

// Re-attach the "#" that the 3-char marker consumed so the normalizer sees a real subpath.
const base = markerLength === 3 ? `${rawBase}#` : rawBase;
const position = `${getCaretCompletionDestination(base)}^`;
```

Hand `position` to the existing `taskPickerMarkerFromPosition`. The explicit
`rawBase.endsWith("^")` guard replaces the "three or more carets do not trigger the
picker" rule that the old `slice(0, -1)` shape enforced implicitly.

**`parseRapidTaskPickerMarker`** — drop the
`destination.endsWith("^") ? destination : ...` special case and always derive
`position` as `` `${getCaretCompletionDestination(destination)}^` ``. Keep the existing
`lineText[markerStartCh + 2] === "^"` guard that rejects three or more carets after
`]]`, and keep the `normalizeText(destination)` emptiness check.

`parseTaskPickerPosition` needs no change;
`startCh`/`endCh`/`raw`/`markerStartCh`/`markerEndCh` bookkeeping and alias handling are
unchanged, so completion and cancellation continue to consume the full marker with no
residue.

The resulting parse table — pin exactly these values in tests:

| Input                | `targetText`                              | `blockPrefix` |
| -------------------- | ----------------------------------------- | ------------- |
| `[[note]]^^`         | `note`                                    | `^`           |
| `[[note^^]]`         | `note`                                    | `^`           |
| `[[note#^^]]`        | `note`                                    | `#^`          |
| `[[note#^abc]]^^`    | `note`                                    | `#^`          |
| `[[note#^abc^^]]`    | `note`                                    | `#^`          |
| `[[note#Heading]]^^` | `note`                                    | `#^`          |
| `[[note#Heading^^]]` | `note`                                    | `#^`          |
| `[[^^]]`             | `` (same file)                            | `^`           |
| `[[#^^]]`            | `` (same file)                            | `#^`          |
| `[[#^abc]]^^`        | `` (same file)                            | `#^`          |
| `[[note\|alias]]^^`  | `note` (alias preserved in `aliasSuffix`) | `^`           |
| `![[note]]^^`        | `note`                                    | `^`           |
| `[[note^^^]]`        | rejected                                  | —             |
| `[[note#^^^]]`       | rejected                                  | —             |
| `[[note]]^^^`        | rejected                                  | —             |

Two intentional, cosmetic-only consequences to record in the tests so they are not
mistaken for regressions later:

- cancelling the picker on `[[note#Heading]]^^` reverts to `[[note#^]]` (canonical)
  rather than restoring the heading — consistent with how cancelling on
  `[[note#^abc]]^^` already discards `abc`;
- `[[note^abc]]^^` and `[[note^]]^^` (non-canonical bare-caret path forms) now yield
  `blockPrefix` `#^` instead of `^`, so their revert text becomes `[[note#^]]`.
  Completed output was already canonicalized to `#^`, so only the intermediate text
  changes.

### 3. Extract a list-item body-bounds helper

Add one helper that returns the trimmed body span of a list item, line-relative:

```js
function listItemBodyBounds(lineText) {
  // normalizeMarkdownLine first (strip a trailing \r)
  // match LIST_ITEM_PREFIX_RE; return null when the line is not a list item
  // return { start, end } after skipping leading and trailing [ \t] in the body
  // return null when start >= end (an empty bullet)
}
```

Rewrite `isDedicatedLinkBullet` to call it (converting to absolute offsets via
`lineContentBounds`) so the two callers cannot drift. Behavior of
`planFuturePomodoroLinkCleanup` must be unchanged — its existing tests are the guard.

### 4. Gate `[*]` promotion on sole-content bullets

Add:

```js
function isSoleContentLinkBullet(lineText, startCh, endCh) {
  // bounds = listItemBodyBounds(lineText); false when null
  // linkStart = startCh > 0 && lineText[startCh - 1] === "!" ? startCh - 1 : startCh
  // true iff bounds.start === linkStart && bounds.end === endCh
}

function sourceLinkIsSoleBulletContent(source) {
  // read source.editor.getLine(source.line) defensively (guard missing editor/getLine)
  // return isSoleContentLinkBullet(lineText, source.startCh, source.endCh)
}
```

and extend the promotion gate:

```js
function shouldPromoteTaskToNext(source, task) {
  return Boolean(
    task &&
    task.status === " " &&
    sourceLineIsPomodoroSubBullet(source) &&
    sourceLinkIsSoleBulletContent(source),
  );
}
```

Design decisions baked in here:

- **The check runs against the still-unfinished marker text.** Both call sites
  (`completeTaskLinkWithExistingId` and `submitLinkTaskBlockId`) call
  `shouldPromoteTaskToNext` after `sourceMarkerStillPresent`, so `[startCh, endCh)` is
  guaranteed to still cover the marker (`[[note]]^^`, `[[note#^abc^^]]`, …). Comparing
  that span against the bullet body is therefore equivalent to asking whether the
  _finished_ block link will be the bullet's sole content.
- **A leading `!` counts as part of the link.** `- ![[note]]^^` promotes: a transcluded
  task block link alone in a Pomodoro sub-bullet is the canonical "working on this now"
  form in this vault.
- **Nothing else is tolerated.** No leading `🍅` session marker, no trailing `#`
  move-only directive, no `~~…~~` strikethrough. Those are all written by
  `task-status-cycler` _after_ a Pomodoro closes or is marked, so they cannot be present
  at `^^`-completion time under an open Pomodoro.
- **Everything else about promotion is untouched**: still only `[ ]` (never `[/]`,
  `[*]`, or done statuses), still only Pomodoro descendants, still keyed off
  `findPomodoroSourceContext` (which does not require the ancestor Pomodoro to be open —
  see Scope boundaries).

### 5. Export the new helpers and add regression coverage

Extend `module.exports.helpers` with `isSoleContentLinkBullet`, `listItemBodyBounds`,
and `getCaretCompletionDestination` (whatever the tests need; keep the export list
alphabetized as it is today), then add cases to `scripts/test-block-id-prompt.cjs`:

**Context independence (pins the premise so a gate can never creep in).** For each of:
prose paragraph, ATX heading, blockquote, table cell, top-level bullet, deeply nested
bullet, an ordinary `[ ] #task` line, a `#task` sub-bullet, and a note with no
`## Pomodoros` section at all — assert `findTaskPickerMarkerNearCursor` finds the
marker, and assert `shouldPromoteTaskToNext` is `false` for a `[ ]` task from each of
those sources.

**Parse table.** One assertion per row of the table in step 2, exercising both
`findTaskPickerMarkerNearCursor` and the resulting `taskPickerRevertReplacement` /
`taskPickerRevertCursorCh`, so no external caret can be left behind and the revert text
is pinned. Include the embedded (`![[…]]`) and aliased (`[[…|alias]]`) variants, and
assert equivalence between the rapid `[[note]]^^` form and the staged `[[note^^]]` form.

**Fenced code.** Keep/extend the existing `lineIsInsideCodeFence` coverage for a `^^`
marker inside a fence.

**Sole-content promotion.** Under an open Pomodoro entry in a `## Pomodoros` section,
with a `[ ]` task:

| Source line                                        | Promotes? |
| -------------------------------------------------- | --------- |
| `    - [[note]]^^`                                 | yes       |
| `    - ![[note]]^^`                                | yes       |
| `    - [[note]]^^   ` (trailing whitespace)        | yes       |
| `    - [[note#^abc^^]]`                            | yes       |
| `    1. [[note]]^^` (ordered bullet)               | yes       |
| `        - [[note]]^^` (deeper descendant)         | yes       |
| `    - working on [[note]]^^`                      | no        |
| `    - [[note]]^^ — blocked on Bob`                | no        |
| `    - [[note]]^^ [[other]]`                       | no        |
| `    - [ ] #task [[note]]^^` (checkbox sub-bullet) | no        |
| `    [[note]]^^` (not a list item)                 | no        |
| `- [[note]]^^` (top-level, i.e. not a sub-bullet)  | no        |

Plus: a sole-content link under an _ordinary task_ sub-bullet does not promote, and
`[/]` / `[*]` selected tasks never promote regardless of source shape.

Prefer pure-helper assertions with the existing `createEditor` fake over timer-driven
plugin tests, matching the current file's style.

### 6. Bump the manifest and deploy

Bump `plugins/block-id-prompt/manifest.json` from `1.3.1` to `1.4.0` (user-visible
behavior changes in both directions). Then, from the linked `bob-plugins` checkout, sync
dry-run first and then for real:

```bash
bob plugins sync -p block-id-prompt -r "$PWD" --dry-run
bob plugins sync -p block-id-prompt -r "$PWD"
```

Confirm the deployed `~/bob/.obsidian/plugins/block-id-prompt/main.js` is byte-for-byte
identical to the repository copy. Obsidian (or the plugin) must be reloaded before the
new JavaScript is active.

## Verification

Run from the linked `bob-plugins` checkout:

- `node -c plugins/block-id-prompt/main.js`
- `npm test` (the full suite, not just the block-id-prompt file —
  `planFuturePomodoroLinkCleanup` and the `task-status-cycler` Pomodoro tests are the
  guard for step 3)
- `npm run validate`
- `git diff --check`
- the two `bob plugins sync` invocations above, then a byte comparison against the
  deployed copy

Then smoke-test in Obsidian:

1. In an ordinary paragraph in a non-daily note, type `[[some note]]` then `^^` quickly.
   The picker opens, the chosen task is linked, and its checkbox status does **not**
   change.
2. Repeat with a deliberate pause after the first caret (staged path). Same result.
3. On an **existing** `[[some note#^abc]]`, place the cursor after `abc` inside the
   brackets and type `^^`. The picker opens and the link is re-targeted to the newly
   chosen task.
4. On an existing heading link `[[some note#Some Heading]]`, type `^^` after `]]`. The
   completed link is `[[some note#^id]]` — not `[[some note#Some Heading#^id]]`.
5. In a daily note, under an open Pomodoro entry, add a sub-bullet whose only content is
   the link and complete it: the selected `[ ]` task becomes `[*]` and the notice
   reports "set Next".
6. Same Pomodoro, but put prose around the link on that sub-bullet: the link completes,
   the future-Pomodoro link cleanup still runs as before, and the task's status is left
   alone.
7. Confirm `^^`-looking text inside a fenced code block and in unrelated prose is
   untouched, and that Escape on the picker leaves no stray carets on the line.

## Scope boundaries

- **Markdown-style links are out of scope.** `[text](note#^id)^^` still does nothing.
  The whole caret workflow is wiki-link-only by construction: `WIKI_LINK_RE` drives
  marker discovery and `sourceReplacement` always emits `[[…]]`. Supporting Markdown
  links would mean a second replacement/revert form plus URI-encoding handling, and this
  vault's block links are wiki links. Worth a separate plan only if that changes.
- **A `^^` not adjacent to a wiki link stays inert** (e.g.
  `- some text [[note#^abc]] trailing^^`). Adjacency is what makes the marker
  unambiguous.
- **The open/done state of the ancestor Pomodoro is not being changed.**
  `shouldPromoteTaskToNext` uses `sourceLineIsPomodoroSubBullet`, which ignores
  `context.isOpen`, so linking under an already-done Pomodoro still promotes. That is
  pre-existing behavior and arguably wrong (`planFuturePomodoroLinkCleanup` _does_
  require `context.isOpen`), but it was not part of this request — flag it, do not
  change it.
- **No change to which tasks appear in the picker** (still open `#task` lines, still
  excluding `#hide`), to the dependency-blocked chip, or to task status definitions.
- **No changes to `bob-cli`**, the Tasks plugin configuration, task-status styling, or
  any other plugin in the monorepo.
- **No hand edits to vault notes or to the deployed vault plugin copy.**
