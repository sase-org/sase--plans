---
tier: tale
title: Repair the Hammerspoon startup regression and font-warning loop
goal:
  Hammerspoon loads and reloads safely without a nil Pomodoro runtime or repeated
  invalid-font warnings.
size: medium
proposed_by: bbugyi200.athena.00o.f0
create_time: 2026-09-09 20:00:24
status: wip
---

# Repair the Hammerspoon startup regression and font-warning loop

## Context and root cause

The Hammerspoon cutover commit `95369559` removed the legacy task-capture block from
`home/dot_hammerspoon/init.lua`. The deletion accidentally included the adjacent,
unrelated runtime bootstrap:

```lua
BobPomodoroCountdown = BobPomodoroCountdown or {}
```

The surviving line `local bobPomodoroRuntime = BobPomodoroCountdown` therefore assigns
`nil` on a fresh Hammerspoon Lua state. The first access to
`bobPomodoroRuntime.tickTimer` then raises the reported line-67 error and aborts the
rest of configuration loading. This is deletion-boundary damage from the cutover, not a
failure in `pomodoro_countdown.lua`, and the retired task-capture implementation must
remain removed.

The repeated `LuaSkin: invalid font specified: .SFNS-Bold` warnings are a second defect
in the surviving Pomodoro menu. Commit `ed77ad7b` began converting the dynamic menu-bar
default to a bold face. On the affected macOS/Hammerspoon combination,
`hs.styledtext.convertFont(...)` returns the private name `.SFNS-Bold`, but passing that
name back to `hs.styledtext.new(...)` is rejected. Missing or long-overdue Pomodoro
states are rendered once per second, so the invalid attribute produces one warning per
tick. Hammerspoon exposes `hs.styledtext.validFont(...)`; converted and fallback faces
must be checked before they are used.

The current test gap is material: `just test-hammerspoon` runs only four pure countdown
presentation cases. It never evaluates `init.lua`, so both syntax checks and the full
passing test suite missed the missing runtime bootstrap and cannot exercise
Hammerspoon-facing font attributes.

## Goals

- Make `home/dot_hammerspoon/init.lua` load successfully in a fresh Lua state and remain
  safe across configuration reloads.
- Preserve the existing Pomodoro menu, screenshot bindings, timer cleanup, and native
  Bob Mac Capture cutover.
- Keep the bold warning treatment when Hammerspoon exposes a valid bold face, while
  guaranteeing that no invalid/private converted font is passed to styled text.
- Add executable regression coverage that fails for the current cutover result and
  exercises the actual initialization boundary.

## Implementation

1. Repair Pomodoro runtime initialization in `home/dot_hammerspoon/init.lua`.
   - Re-establish a table-valued global runtime before creating the local alias so a
     fresh load cannot dereference `nil`.
   - Retain the global table deliberately across reloads so the existing timers,
     watcher, task, and menu can be stopped or reused before replacements are installed.
   - Defend against an unexpected non-table stale global rather than assuming only `nil`
     is possible.

2. Make bold menu-bar font selection validation-safe.
   - Resolve the dynamic menu-bar default once and ask `convertFont` for its bold
     variant.
   - Accept a converted font only when it has a usable name and `validFont(name)` is
     true. If the private system face is rejected, try a public bold fallback at the
     same menu-bar size and validate it as well; if no bold face validates, omit the
     custom font attribute and retain the state distinction through its existing text
     and color.
   - Build the missing and overdue-warning attribute tables from that validated result
     so the one-second render path can never submit `.SFNS-Bold` or another rejected
     name. Keep the ordinary and recently overdue presentations unchanged.

3. Add a focused Hammerspoon initialization spec under `tests/hammerspoon/`.
   - Provide a minimal, isolated `hs` mock for hotkeys, tasks, timers, menubar items,
     caffeinate/path watchers, notifications, and styled text, while stubbing the two
     required local modules.
   - Load the real `init.lua` with no pre-existing `BobPomodoroCountdown` and assert
     that it completes, creates a table runtime, and installs the expected runtime
     objects. Run this regression against the pre-fix source first to demonstrate that
     it catches the reported nil dereference.
   - Load it again with retained runtime objects and assert that old
     timers/watchers/tasks are stopped and the menu object is reused before new objects
     are installed.
   - Make the mock return `.SFNS-Bold` from `convertFont` and reject it from
     `validFont`; drive a missing/overdue render and assert that every font passed to
     `styledtext.new` is validated (or omitted), while the expected warning title/color
     remains intact.
   - Restore all globals and `package.loaded` entries after each example so the spec is
     order-independent and cannot contaminate the existing countdown tests.

4. Update only nearby test-runner wording if necessary so `just test-hammerspoon` no
   longer claims it runs request-model-only tests. Do not broaden the cutover, restore
   `task_capture.lua`, change Bob Mac Capture shortcuts, or modify unrelated Pomodoro
   parsing/presentation behavior.

## Validation and deployment

1. Establish the regression test's red-green behavior: confirm the new fresh-load case
   fails on the current source at the reported runtime dereference, then confirm it and
   the invalid-font case pass after the repair.
2. Run `just test-hammerspoon` and require the existing four presentation tests plus all
   new initialization/reload/font cases to pass together.
3. Run `luac -p` over every file in `home/dot_hammerspoon/` and `tests/hammerspoon/`,
   then run `stylua --check home/dot_hammerspoon tests/hammerspoon` after formatting the
   touched Lua files.
4. Review the final diff and verify that task-capture/WebView/hotkey code remains
   absent, the production capture shortcut is not reclaimed by Hammerspoon, and no
   unrelated dotfiles changed.
5. Follow the chezmoi repository deployment rule after committing by running
   `chezmoi update -a --force`. On the target Mac, reload Hammerspoon and verify:
   - configuration loading reaches the end without the line-67 exception;
   - the Pomodoro menu and remaining paste/screenshot hotkeys initialize;
   - a missing or long-overdue Pomodoro state renders for multiple one-second ticks
     without any `.SFNS-Bold`/`invalid font specified` warning;
   - repeated reloads do not duplicate live timers or watchers. If the implementation
     environment is not the target Mac, report this final physical smoke check
     explicitly rather than claiming it ran.

## Acceptance criteria

- A fresh evaluation of the real Hammerspoon init file cannot dereference a nil or
  non-table Pomodoro runtime.
- Reload preserves cleanup/reuse semantics for the prior Pomodoro runtime objects.
- No unvalidated converted font name reaches `hs.styledtext.new`; the reproduced
  `.SFNS-Bold` candidate follows a warning-free fallback path.
- All Hammerspoon tests, Lua syntax checks, and Lua formatting checks pass.
- The legacy Hammerspoon task-capture implementation and its production shortcut remain
  retired.
