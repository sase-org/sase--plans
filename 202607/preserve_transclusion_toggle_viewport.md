---
tier: tale
goal:
  Preserve the active Obsidian editor viewport and caret context whenever the `!`
  transclusion keymap toggles links, without regressing dependency synchronization,
  counted toggles, or undo behavior.
create_time: 2026-09-09 19:53:19
status: wip
---

# Plan: Preserve the viewport during `!` transclusion toggles

## Diagnosis and intended outcome

The `!` keymap invokes Bob Navigation Hotkeys' `toggle-line-transclusions` command. The
visible jump is explained by a regression in that command's source-editor update path;
the linked configuration sources contain no second `!` implementation, although desktop
Obsidian was unavailable during planning so its live hotkey table could not be
inspected.

Before dependency-aware toggling, the command replaced only the current line. Commit
`982ac8b` changed both the single-line and counted paths to build a complete
next-document snapshot and pass it to `replaceEditorContent()`. That helper calls
`Editor.replaceRange()` from `{line: 0, ch: 0}` through the end of the note. Because the
active selection lies inside that document-wide replacement, the editor temporarily maps
it to the replacement boundary at the end of the note. The later `setCursor()` restores
the logical line, but Obsidian/CodeMirror's cursor-visibility behavior then scrolls just
far enough to expose it, leaving that line at the bottom of the viewport. Existing tests
verify resulting Markdown and dependency metadata, but do not constrain the editor
transaction's range or viewport effect.

After the fix, a bare `!`, a Vim-counted `N!`, and both add/remove directions must leave
the user's selected line at the same visual position whenever it was already visible.
The command must still adjust the caret column around inserted or removed `!` markers,
synchronize `dependsOn`/`id` fields, promote eligible dependency targets, reject stale
asynchronous snapshots, and behave as one undoable edit.

## Implementation

1. Open the linked `bob-plugins` source through `sase repo open bob-plugins`; keep the
   implementation and tests in that source-of-truth repository, never in the deployed
   vault copy.

2. Replace the document-wide source rewrite used by
   `applyDependencyAwareTransclusionChanges()` with a focused editor transaction:
   - Continue computing and validating `originalLines` and `nextLines` exactly as today,
     including cross-file reads/writes, stale-source checks, legacy dependency-ID
     propagation, hidden-target rules, and monotonic status promotion.
   - Derive `EditorChange` entries only for source lines whose final text differs, with
     each change bounded to that original line. The dependency transformations preserve
     line count, so treat a line-count mismatch as an invariant failure rather than
     silently falling back to a whole-document replacement.
   - Apply all line changes through the public
     `Editor.transaction({changes, selection})` API, which is supported well below the
     plugin's minimum Obsidian version. Include the final adjusted caret in the same
     transaction so CodeMirror never observes an intermediate selection at the end of
     the document and the whole command remains one undo step.
   - Keep a narrow, line-local fallback for editor test doubles or unexpected compatible
     editor shapes that lack `transaction()`. Apply replacements from the bottom upward
     and restore the caret only after all replacements; never use the document-wide
     replacement helper in the interactive `!` path.
   - Have the single and counted callers calculate their existing adjusted caret and
     pass it into the shared apply operation. Do not issue an explicit cursor-visibility
     scroll; the caret was already visible and the targeted transaction should preserve
     the viewport naturally.

3. Add regression coverage in `scripts/test-navigation-hotkeys.cjs` at the runtime
   command boundary, not only at the pure text-transform layer:
   - Extend or add an editor double that records transactions, changes, selection,
     scroll state, and undo grouping.
   - Reproduce a mid-document caret with content both above and at least ten lines below
     it. Assert that a single `!` uses one transaction, changes only the required
     line(s), keeps the viewport sentinel unchanged, and places the caret at the
     marker-adjusted column.
   - Cover a same-file dependency toggle where the child link, parent task, and target
     task are on non-adjacent lines; assert that all metadata/status edits are included
     as separate line-bounded changes in one transaction and unrelated lines are not
     replaced.
   - Cover removal, counted `N!`, and the asynchronous cross-file path so both callers
     and both toggle directions share the viewport-safe source update. Retain assertions
     for stale source/target snapshots and external-write failure behavior.
   - Exercise the line-local fallback separately and verify bottom-up replacement order,
     caret restoration, and no multi-line replacement range. Keep CRLF/source-content
     assertions where relevant so changing edit granularity does not alter line endings.

4. Update release metadata for Bob Navigation Hotkeys with a patch version bump in
   `plugins/bob-navigation-hotkeys/manifest.json` and the matching plugin catalog entry
   in `README.md`.

## Validation and deployment

1. Run the focused navigation-hotkeys tests during development, then run the complete
   `npm test` suite and `npm run validate` in `bob-plugins`. Re-run both after the
   version/catalog edits. The current pre-change baseline is 147 passing tests and 6
   valid plugin manifests.

2. Review the final diff to confirm that dependency semantics and unrelated
   navigation/scroll helpers are unchanged, no whole-document replacement remains
   reachable from the single or counted `!` flow, and the working trees contain only
   planned changes.

3. Deploy the source-of-truth plugin with `bob plugins sync` as required by the project
   workflow. If desktop Obsidian is available, reload the plugin and smoke-test with the
   caret in the middle of a long note while at least ten following lines are visible:
   - toggle an ordinary wikilink on and off with bare `!`;
   - toggle a valid same-file and cross-file dependency in both directions;
   - exercise a counted toggle;
   - confirm the selected line retains its visual viewport position, the adjusted caret
     is correct, dependency metadata/status changes remain correct, and one undo
     reverses the complete source edit.

If desktop Obsidian remains unavailable, record that limitation explicitly after
automated validation and provide the exact smoke sequence above for the next live
session; deployment and automated regression coverage should still be completed.
