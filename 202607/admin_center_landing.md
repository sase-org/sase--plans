---
tier: tale
title: Fast SASE Admin Center landing page
goal: 'Opening the SASE Admin Center shows an immediate, polished landing page that
  teaches every numbered tab, while real tab panes and their data work are created
  only when the user explicitly enters them.

  '
create_time: 2026-07-20 11:49:33
status: done
prompt: 202607/prompts/admin_center_landing.md
---

# Plan: Fast SASE Admin Center landing page

## Outcome

Turn the generic SASE Admin Center entry point into a lightweight home view instead of reopening or eagerly composing a
working tab. The first paint should contain only static presentation widgets and should never wait for config inventory,
logs, projects, statistics, tasks, update discovery, xprompt discovery, disk-backed tab state, or concrete pane layout.

This is a home view, not an eighth tab. The seven existing tabs keep their alphabetical order and stable `1`-`7`
bindings, and direct actions such as **Open logs panel**, **Open tasks panel**, **Open statistics**, the Updates
indicator, and the comprehensive update shortcut continue to open their requested pane directly.

## Product and interaction design

The unqualified `#` binding and the generic **Open SASE Admin Center** command should open home every time, including
after another Admin Center tab was used and after an ACE restart. Remove the last-tab persistence behavior rather than
letting an old preference compete with the new onboarding contract. Do not delete an existing
`~/.sase/admin_center_tab.json` from the user's machine; it can remain as harmless ignored state.

Retain the existing aurora title, frame, and numbered clickable tab strip so the new screen feels like an intentional
front door to the same panel. While home is visible, no tab is highlighted. Use the current tab accent palette
throughout the landing menu and establish this hierarchy:

1. A centered lead: **Choose a section**.
2. A one-line orientation statement: **Configure, observe, and maintain SASE from one place.**
3. A prominent instruction: **Press 1-7 or click a tab to open it.**
4. Seven compact, evenly spaced menu rows. Each row starts with a colored number token and tab label, followed by its
   concise description.
5. A quiet footer hint: **Tab/Shift+Tab cycle tabs · q/Esc close**.

Use these descriptions as the canonical introduction to the tabs:

| Key | Tab        | Description                                                              |
| --- | ---------- | ------------------------------------------------------------------------ |
| `1` | Config     | Review and edit layered SASE settings with provenance and live previews. |
| `2` | Logs       | Inspect TUI activity, launch failures, and notification history.         |
| `3` | Projects   | Manage projects and inspect their repositories and workspaces.           |
| `4` | Statistics | Explore agent activity, runtime, outcomes, and trends over time.         |
| `5` | Tasks      | Follow background work, inspect live output, and manage running jobs.    |
| `6` | Updates    | Update SASE, plugins, and supported agent CLIs from one place.           |
| `7` | XPrompts   | Find, preview, and load reusable prompts and workflows.                  |

Keep the page visually calm: one readable column is preferable to a dense or uneven card grid, the accent colors should
carry the scan hierarchy, and the surrounding whitespace should make the seven choices feel deliberate. Put the body in
a bounded scrolling region so all entries remain reachable on short terminals; at the normal `120x40` size it should be
balanced without scrolling, and at `100x24` it must remain legible without horizontal clipping or overlapping the
header. The landing body should have no focusable controls: digits and modal-priority tab bindings must work
immediately, while mouse users use the already-clickable main tab strip.

From home, `Tab` enters Config and `Shift+Tab` enters XPrompts. Once a real tab is active, both bindings continue their
existing wrapping cycle across only the seven real tabs; home is shown again on the next generic reopen, not inserted
into that cycle. `1`-`7` and tab-strip clicks enter the matching tab. Out-of-range digits remain swallowed, and `q` /
`Esc` close from either home or a real pane.

## Lightweight view and lazy pane lifecycle

Refactor `ConfigCenterModal` around two distinct concepts: an internal home view and the existing `CenterTab` set. Keep
a single immutable tab-spec catalog as the source of truth for order, number, label, accent, description, and pane
identity; derive the tab strip and lookup structures from it so the onboarding copy and navigation cannot drift.

The generic modal composition should create only the frame/header, numbered strip, home caption/divider, landing widget,
and a lightweight content host. Do not instantiate any of the seven concrete panes on this path. Move concrete pane
imports behind an explicit per-tab factory and remove their unused eager re-exports from `tui/modals/__init__.py`, so
loading the lightweight modal itself does not pull in Config, Logs, Projects, Statistics, Tasks, Updates, or XPrompts
implementations as a side effect.

When navigation first targets a real tab, construct and mount only that pane, then make it visible, activate its
existing visibility lifecycle, focus its default browse surface, and update the strip/caption. Cache the mounted
instance for the lifetime of the modal so switching away and back preserves selection, filters, scroll position, and
already-loaded data. Subsequent navigation to that tab must reuse the same widget and must not duplicate workers,
timers, or widget IDs.

Make first-mount navigation idempotent and ordered so rapid digit/tab input cannot create two instances or leave the
header, content switcher, focus, and lifecycle hooks describing different tabs. Deactivate the old pane before hiding
it; activate and focus the new pane only after it is mounted and selected. If pane construction or mounting fails,
retain the previous stable view (or home), surface a useful error notification, and leave that tab eligible for a clean
retry.

An explicit `initial_tab` remains supported for direct-entry actions and tests. That path may compose or mount the one
requested pane immediately, but never its six siblings, and must preserve pane-specific inputs such as the project
scope, Statistics keymap registry, and Updates `auto_update` behavior. The generic action should stop supplying a
persisted tab and instead request home.

Remove the obsolete Admin Center tab-state read/write machinery: the startup disk read, in-memory/coalescing fields,
off-thread persistence runner, storage module, and persistence-only tests. This both enforces the new reopen behavior
and removes work from ACE startup. Keep this cleanup limited to Admin Center presentation state; do not change unrelated
persisted selections or pane-local caches.

The landing renderer itself must be pure and bounded: no filesystem access, config lookups, project discovery,
subprocesses, workers, timers, or async callbacks. Existing pane work should continue to use its established off-thread
and visibility-gated paths after first activation.

## Documentation and compatibility

Update the Admin Center overview and global keybinding documentation in `docs/configuration.md` and `docs/ace.md` to
describe home-first opening, numbered navigation, direct-entry commands, and lazy pane loading. Remove claims that `#`
reopens the last persisted tab. Keep tab-specific documentation and numeric assignments unchanged.

Review comments, docstrings, test helper descriptions, and help copy that assume every pane is mounted or a tab is
selected at modal open. Preserve the existing main help cue that `1`-`7` jump to tabs; the landing page adds the richer
first-use explanation rather than changing configured keymaps.

## Verification and performance acceptance

Add focused behavioral coverage around the modal seam:

- Generic `#` and **Open SASE Admin Center** start on home every time, with all seven labels, numbers, canonical
  descriptions, and navigation hints visible; using and closing a real tab does not change the next generic reopen.
- The home path has no concrete pane widgets, starts no pane loader/worker/timer, and performs no Admin Center
  state-file read or write. Tests should fail if an unrelated pane constructor or loader is touched.
- Each digit, tab direction, and tab-strip click mounts exactly the requested pane, updates the caption/selection/focus,
  and keeps the hidden ACE top-level tab unchanged. Re-entering an already visited tab reuses its widget and state.
- Rapid/repeated first navigation is idempotent, out-of-range digits remain contained by the modal, lifecycle callbacks
  see the correct active identity, and a simulated mount failure leaves a coherent retryable view.
- Direct Projects, Logs, Statistics, Tasks, Updates, and auto-update entry points still mount only their target pane and
  retain their existing semantics. Existing pane-specific suites continue to exercise navigation after lazy mounting;
  simplify broad “patch every sibling pane” fixtures where they are no longer necessary.

Add intentional PNG coverage for the landing page at `120x40` and a compact `100x24` viewport. Inspect the rendered
PNGs, not just snapshot equality, for hierarchy, spacing, accent contrast, full copy visibility, and short-terminal
behavior. Existing direct-tab snapshots should remain visually stable except where the canonical descriptions
intentionally change.

Before implementation, capture a repeatable baseline for opening the current eager modal through a Textual pilot. Add or
extend a slow benchmark that records p50/p95 from `#` dispatch until the landing view is mounted and painted, then
repeat after the change under both empty and populated project/config fixtures. Report the before/after numbers; do not
use a flaky wall-clock threshold in the normal unit suite. Acceptance requires a structurally constant home path (zero
concrete panes and zero data-scaled work) and a clear measured reduction from the eager baseline. Document the focused
benchmark command alongside the existing TUI performance recipes.

Run `just install` before repository checks, then run the focused Admin Center/pane tests, the new performance
benchmark, `just test-visual` (updating only intentional goldens), and finally the required `just check`.
