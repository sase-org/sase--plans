---
tier: tale
title: Two-step Pomodoro daily-note navigation
goal: "Ctrl+9 keeps its direct Pomodoro jump within the active note, but when it falls
  back to today's daily note the first press only opens that note at its remembered
  in-session cursor and scroll location and the second press performs the existing
  Pomodoro jump.

  "
create_time: 2026-09-09 19:53:26
status: wip
---

# Plan: Two-step Pomodoro daily-note navigation

## Context and behavioral contract

The source of truth is the linked `bob-plugins` repository, specifically the
`bob-ledger-tools` Obsidian plugin. Today, `jumpToCurrentPomodoro()` immediately opens
today's daily note and calls the same cursor-and-centering jump during a single fallback
invocation. Change only that fallback sequencing; retain the existing Pomodoro
target-selection rules, notices, daily-note path resolution, file creation/opening
behavior, and direct jump behavior when the active note already has a qualifying target.

The resulting command behavior should be:

1. If the active Markdown note has a target, one `<Ctrl+9>` press jumps to and centers
   that target exactly as it does now.
2. If a non-daily active note has no target, the first press opens today's daily note
   and stops without moving or centering its cursor. If that daily note has been visited
   earlier in the current Obsidian session, restore its remembered cursor and page
   scroll; if it is still open in a leaf, simply reactivate that live leaf without
   disturbing its view. With no remembered session location, leave Obsidian's normal
   initial location alone.
3. Once today's daily note is active, the second press follows the ordinary same-file
   path and performs the pre-existing Pomodoro jump. If the daily note has no qualifying
   target, report the existing error on this second press, rather than during the first
   open-only press.
4. Do not carry a location across daily-note paths or Obsidian sessions, and do not let
   a pending restore from the first press overwrite a rapid second-press Pomodoro jump.

## Implementation design

- In `plugins/bob-ledger-tools/main.js`, separate “open the daily fallback” from “jump
  in the active editor.” The fallback invoked from a non-daily note should await the
  existing daily-note opening path, validate that a Markdown editor was opened, restore
  an applicable remembered location, and then return without scanning or jumping. The
  already-active-daily guard remains responsible for showing the existing no-target
  notice, while an actual target continues through `jumpToPomodoroTarget()`.
- Add plugin-owned, session-scoped daily-note location tracking inspired by the
  `<Ctrl+0>` dashboard implementation in `bob-navigation-hotkeys`, but keep it
  appropriate for an ordinary Markdown daily note: store a normalized cursor and raw
  CodeMirror scroll coordinates keyed by resolved daily-note path; capture
  selection/cursor updates and scroll events while the relevant daily editor is active;
  and never persist the map to disk. Clamp a remembered cursor if the note changed since
  capture.
- Preserve the distinction the dashboard hotkey makes between leaf states. When the
  daily file is already open in a leaf, activate it and trust its live cursor and
  viewport. When it must be freshly opened after an earlier in-session visit, snapshot
  the remembered location before opening and restore cursor without an implicit scroll,
  followed by the saved scroll coordinates. Use bounded deferred retries/assertion only
  where Obsidian may not yet have attached or settled the CodeMirror view.
- Coordinate asynchronous work explicitly: suppress capture while applying a remembered
  location, cancel stale deferred restore work on unload/navigation, and cancel any
  pending location restore before `jumpToPomodoroTarget()` moves and centers the cursor.
  This makes a quick second `<Ctrl+9>` authoritative. Clean up scroll listeners and all
  deferred handles when the plugin unloads or the active daily view changes.
- Keep the standalone “Open today's daily note” command and shared opening helpers
  compatible. Avoid coupling `bob-ledger-tools` to private state in
  `bob-navigation-hotkeys`; reuse the design principles locally so either plugin can
  load independently.

## Tests and release integration

- Add focused Node coverage for ledger navigation (either in a clearly named new
  ledger-navigation test file wired into `package.json`, or by broadening the existing
  ledger test harness). Stub Obsidian workspace leaves and CodeMirror views closely
  enough to assert cursor and raw scroll effects rather than only helper return values.
- Cover the command decision matrix: an active-note target still jumps immediately; the
  first fallback press opens without a cursor/center call; the second press jumps to the
  same target selected before this change; and a daily note with no target defers its
  existing notice until the second press.
- Cover location lifecycle and races: an already-open daily leaf keeps its live
  cursor/scroll, a closed-then-reopened daily note restores the remembered in-session
  location, a never-visited or different-date daily note receives no stale location,
  invalid/out-of-range saved data is handled safely, and a rapid second press cancels
  restoration so the Pomodoro jump remains final. Also cover listener/deferred cleanup
  to prevent a later leaf from being mutated.
- Increment the `bob-ledger-tools` patch version in its manifest and keep the README
  plugin table synchronized with that version. Run the focused ledger tests, the
  complete `npm test` suite, and `npm run validate` from the linked repository.
- After all source and validation checks pass, run `bob plugins sync` as required by the
  linked repository instructions so the tested `bob-ledger-tools` files are deployed to
  the Bob vault, and verify the sync completes successfully.

## Boundaries and risks

- Do not change which ledger entry qualifies as the jump target; this work only changes
  fallback timing and view restoration.
- Do not introduce durable settings for editor locations. The requested memory is
  deliberately limited to the current Obsidian/plugin session.
- Raw scroll restoration can race Obsidian/Vim visibility scrolling, so restoration must
  be bounded and cancelable, with the explicit Pomodoro jump always taking precedence
  over an older open-only action.
