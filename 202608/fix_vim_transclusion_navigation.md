---
tier: tale
title: Fix Vim navigation after task-link transclusion
goal:
  Bare and counted transclusion toggles preserve normal Vim navigation and never restore
  a stale cursor after asynchronous dependency synchronization.
size: medium
proposed_by: bbugyi200.athena.09x
create_time: 2026-09-09 20:00:06
status: wip
---

# Fix Vim navigation after task-link transclusion

## Objective

Make the Bob Navigation Hotkeys `!` gesture toggle the current line's Obsidian task-link
transclusion without leaving Vim in a pending command state or allowing a late
dependency-sync transaction to overwrite a subsequent `j`/`k` movement. Preserve the
existing counted `N!` behavior, dependency metadata synchronization, caret adjustment,
viewport stability, and non-Vim fallthrough behavior.

## Diagnosis

The vault is running the current `bob-navigation-hotkeys` 1.27.0 deployment, so the
symptom is not caused by plugin drift. The live Vim configuration maps bare `!` through
`obsidian-vimrc-support` as an Ex command:

```vim
exmap bob_toggle_transclusions obcommand bob-navigation-hotkeys:toggle-line-transclusions
nmap ! :bob_toggle_transclusions<CR>
```

`plugins/bob-navigation-hotkeys/main.js` also installs a capture-phase physical `!`
listener, but `handleCountedTransclusionTogglePhysicalKeydown` handles the event only
when `getPendingVimRepeat()` reports an explicit count. A bare `!` therefore bypasses
the guarded path that prevents propagation and calls `resetPendingVimInputState()`, and
instead traverses the VimRC Ex-command bridge. That bridge invokes the plugin's async
editor callback without awaiting it. This leaves the gesture split between Vim's
mapped-command processing and the later dependency-aware editor update.

The late update has a second race: `toggleCurrentLineTransclusions()` snapshots the
caret before awaiting cross-file reads/writes, and
`applyDependencyAwareTransclusionChanges()` always installs that snapshot as the final
transaction selection. If the user presses `j` or `k` while synchronization is pending,
the eventual transaction can replace the newer Vim selection with the stale one. In Live
Preview, the simultaneous embed decoration and cursor-visibility update can turn that
stale selection reset into a large viewport jump, commonly to the end of the note.
Counted `N!` already avoids the first half of this interaction by owning and resetting
the physical key event; bare `!` does not.

A previous viewport fix correctly replaced whole-document rewrites with line-bounded,
single-undo transactions. Keep that design: the remaining defect is command ownership
and stale selection restoration, not the dependency transformation itself.

## Implementation

1. In the linked `bob-plugins` repository, refactor the Bob Navigation Hotkeys
   capture-phase transclusion listener so it owns both bare and explicitly counted `!`
   in a focused Markdown editor while Vim is in normal mode and the current line has a
   toggleable link. Prevent the handled event from reaching the VimRC mapping, clear the
   pending Vim input state, dispatch bare `!` to the existing single-line toggle, and
   retain the current `N!` range semantics for explicit counts. Ignore physical
   key-repeat events and continue to fall through for insert/visual/replace mode,
   non-editor targets, modifier chords, unavailable Vim state, or lines without links.

2. Make dependency-aware transclusion commits selection-safe across asynchronous work.
   Carry both the invocation caret and its adjusted post-toggle caret into the commit
   boundary. Immediately before the line-bounded editor transaction, compare the live
   caret with the invocation caret: include the adjusted selection only if the user has
   not moved; otherwise omit the explicit selection and let the editor map the user's
   current selection through the same-line changes. Retain the existing document
   preimage checks so actual text edits still abort guarded writes, and do not restore a
   stale viewport or cursor after the user has navigated.

3. Extend `scripts/test-navigation-hotkeys.cjs` with focused event and async-race
   coverage. Prove that bare normal-mode `!` is prevented exactly once, resets Vim
   input, and invokes only the single-line toggle; explicit counts still invoke the
   counted path; held-key repeats and all existing fallthrough cases remain untouched.
   Add a deferred cross-file synchronization case in which the test moves the caret with
   a simulated `j`/`k` before resolving the vault operation, then assert that the final
   transaction changes the expected dependency/link lines without supplying a stale
   selection or changing the newer caret/viewport. Retain assertions for the
   unchanged-caret case so normal toggles still adjust the caret by the inserted or
   removed `!` and remain one undo group.

4. Increment the `bob-navigation-hotkeys` patch version in its manifest and matching
   README inventory entry, and briefly document that bare and counted `!` are handled
   directly in Vim normal mode so immediate navigation remains ordinary.

5. Run the focused navigation-hotkeys tests, the complete bob-plugins test suite, and
   manifest validation. Deploy the updated source-of-truth plugin to the Bob vault with
   `bob plugins sync -p bob-navigation-hotkeys`, verify `bob plugins list` reports it
   enabled and synced with no drift, and manually smoke-test in Obsidian Live Preview:
   toggle same-file and cross-file task links near the middle of a long note, press
   `j`/`k` immediately and after completion, exercise `N!`, and verify source-mode and
   insert-mode behavior still falls through correctly.

## Validation

Run from the linked `bob-plugins` repository unless noted otherwise:

```bash
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
bob plugins sync -p bob-navigation-hotkeys
bob plugins list
```

Manual acceptance in the Bob vault:

- Bare `!` toggles exactly the current line and the following `j` or `k` moves exactly
  one source line without a viewport jump.
- A quick `j` or `k` during a cross-file dependency update remains the final cursor
  location after that update completes.
- Explicit `N!` retains its existing counted range, caret, dependency metadata, and
  single-undo behavior.
- Insert, visual, and replace modes; modified `!` chords; auto-repeat; and lines with no
  toggleable links are not captured by the plugin.
- Same-file and cross-file dependency promotion/blocking and unlink semantics remain
  unchanged, and the deployed vault copy is byte-synced with the source plugin.
