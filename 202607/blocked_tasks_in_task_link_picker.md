---
tier: tale
title: Show Blocked tasks in the ^^ task-link picker
goal:
  The ^^ task-link picker lists Blocked ([?]) tasks in a labeled Blocked group beneath
  the unblocked ones, each row marked with an amber [?] status pill and a
  reason-specific Blocked chip, without changing the order or default selection of the
  unblocked tasks.
create_time: 2026-09-09 19:53:02
status: wip
---

# Show Blocked (`[?]`) tasks in the `^^` task-link picker

## Problem

Typing `^^` after a wiki link (`[[Some Note]]^^` or `[[Some Note#^^]]`) opens the **Task
Link Picker** modal from the `block-id-prompt` plugin, which lists the target note's
open `#task` lines so one can be linked (and given a block ID if it lacks one).

That picker silently omits every task whose checkbox status is `[?]` (Obsidian Tasks
"Blocked"). The cause is a single constant that is narrower than its siblings in the
other Bob plugins:

| File                                       | Constant                       | Value                      |
| ------------------------------------------ | ------------------------------ | -------------------------- |
| `plugins/block-id-prompt/main.js:18`       | `OPEN_OBSIDIAN_TASK_STATUSES`  | `" "`, `"/"`, `"*"`        |
| `plugins/bob-navigation-hotkeys/main.js:9` | `OPEN_OBSIDIAN_TASK_STATUSES`  | `" "`, `"/"`, `"*"`, `"?"` |
| `plugins/task-status-cycler/main.js:23`    | `DEPENDENCY_OPEN_TASK_SYMBOLS` | `" "`, `"*"`, `"/"`, `"?"` |

So `block-id-prompt` is the outlier: everywhere else in Bob, Blocked is an _open_ state.
The user wants those tasks listed, and visually marked as blocked.

## Design

### Two independent notions of "blocked" already exist

The picker already renders a blocked treatment, but only for **dependency-blocked**
tasks — open tasks carrying `[dependsOn:: …]` whose targets are not done
(`resolveTaskDependencyState`, `main.js:852`). Those get an orange left border
(`.bid-tlp-row.is-dep-blocked`) and a "Blocked" chip in the row meta line.

Adding `[?]` introduces a second, orthogonal signal — **status-blocked** — and the two
do not coincide:

- A `[?]` task may have **no** unmet `dependsOn`. `bob-navigation-hotkeys` also stamps
  `[?]` for tasks with a _future_ `[scheduled:: YYYY-MM-DD]` propagated from a project
  (`reconcileBlockedScheduledTaskLine`, `bob-navigation-hotkeys/main.js:9808`), and a
  task can be marked Blocked by hand.
- An open `[ ]` / `[/]` / `[*]` task may have unmet `dependsOn` and not yet be stamped
  `[?]` (the `!` sync in `bob-navigation-hotkeys` has not run over it yet).

The design must therefore treat "blocked" as a single user-facing concept with two
contributing **reasons**, and must never claim "blocked by X" when no unmet dependency
is known.

### Decisions and rationale

**D1 — Widen `OPEN_OBSIDIAN_TASK_STATUSES` rather than special-case the picker.** In
`block-id-prompt`, that constant is read by exactly two functions: `taskItemFromLine`
(`main.js:705`, the picker's item builder) and `isOpenObsidianTaskLine` (`main.js:645`),
which is dead code in this file — not called anywhere and not exported. The blast radius
of adding `"?"` is therefore exactly the picker, and the change _removes_ an
inconsistency with the other two plugins instead of adding a local exception.

`DONE_OBSIDIAN_TASK_STATUSES` stays `x`/`X`/`-`, so `buildTaskDependencyIndex` already
treats a `[?]` dependency as an unmet blocker. No change needed there.

**D2 — Blocked tasks sink to the bottom of the list, in a labeled group; document order
is preserved inside each group.** This is the reliability decision. Today the first row
is the first open task in the note and Enter links it. If `[?]` tasks were interleaved
by line number, a note whose first task is Blocked would silently change what Enter
selects — a muscle-memory footgun. Sinking blocked tasks guarantees the top of the list
keeps exactly its current contents and order whenever any unblocked task exists, so the
common path is unchanged.

The partition is a stable two-bucket split (not a sort), so line order is preserved
within each group.

**D3 — One concept, one group: the "Blocked" group holds status-blocked _and_
dependency-blocked tasks.** Splitting them into separate buckets, or leaving
dependency-blocked tasks interleaved while sinking `[?]` tasks, would put two things
both labeled "Blocked" in two different places. The mental model is `Unblocked` above,
`Blocked` below. Accepted cost: an open `[ ]`/`[/]`/`[*]` task with unmet `dependsOn`
now moves to the bottom where it previously sat in line order. This is rare in a
`!`-synced vault (the sync stamps such tasks `[?]` anyway) and the group header makes
the new position self-explanatory.

**D4 — Group headers appear only when both groups are non-empty.** A note with no
blocked tasks renders pixel-identically to today; a note with nothing _but_ blocked
tasks needs no header either, because every row is individually marked and the subtitle
already reports the count. Structure appears exactly when it is needed to explain the
ordering — no chrome tax on the everyday case.

Labels are **"Unblocked"** and **"Blocked"**. "Unblocked" is chosen over "Ready"
deliberately: Bob's own vocabulary uses _Ready_ for `[ ]` specifically (alongside
_Next_, _In Progress_, _Blocked_ — see `scheduledRecoveryNoticeSuffix` in
`bob-navigation-hotkeys`), and the top group also contains `[/]` and `[*]`. "Unblocked"
is the exact complement of "Blocked" and collides with nothing.

**D5 — The status pill states _what_; the chip states _why_.** Three redundant "Blocked"
signals on one row (amber pill + chip + group header) is noise. So:

- The **pill** shows the literal checkbox — `[?]` — in a new amber `is-blocked` variant,
  joining the existing `is-active` (accent) / `is-next` (green) / `is-todo` (muted)
  language. Four statuses, four colors.
- The **chip** carries the _reason_ and is reason-specific, so it never asserts a
  dependency that does not exist. Its text label ("Blocked", "Blocked · N") means the
  marking never depends on color alone — accessible, and legible even when a homogeneous
  all-blocked list renders without headers.

**D6 — Do not infer a "scheduled" reason.** A `[?]` task blocked by a future
`[scheduled:: …]` (or `⏳ YYYY-MM-DD`) falls into the "no unmet dependency" tooltip
branch, which is truthful. Detecting it properly means parsing two date syntaxes plus
local-date comparison — real timezone risk for a tooltip. Out of scope; the honest
generic wording covers it.

**D7 — Blocked tasks stay selectable.** `^^` creates a link; linking to a blocked task
is legitimate (that is often exactly how a dependency gets recorded).
`shouldPromoteTaskToNext` (`main.js:1480`) already requires `status === " "`, so a `[?]`
task can never be auto-promoted to `[*]` when it receives a new block ID — verified, no
change needed, but pin it with a regression test.

### Resulting visual

```
┌────────────────────────────────────────────────────────────────┐
│ ▣  Link to task                                                │
│    7 open tasks in Projects · 3 blocked                        │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 🔍 Filter tasks                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                │
│ UNBLOCKED                                              4       │  ← sticky, uppercase, faint
│ ▎[/] Draft the migration note                    ↵ link ^draft │  ← accent left border = selected
│  [ ] Review inbox                                       + id   │
│  [*] Ship the picker change                     ↵ link ^ship   │
│  [ ] Write release notes                                + id   │
│                                                                │
│ BLOCKED                                                3       │  ← amber-tinted label
│ ▎[?] Deploy to the vault                        ↵ link ^deploy │  ← amber left border
│      Line 42  🔒 Blocked · 2                                   │
│ ▎[?] Announce in weekly note                            + id   │
│      Line 51  🔒 Blocked                                       │
│ ▎[ ] Archive old plugins                        ↵ link ^arch   │
│      Line 63  🔒 Blocked · 3                                   │
│                                                                │
│ ↑↓ Navigate   ^N ^P Move   ↵ Link   esc Dismiss                │
└────────────────────────────────────────────────────────────────┘
```

## Implementation

All paths below are relative to the `bob-plugins` checkout. Open it with:

```bash
sase repo open bob-plugins -r "Implement Blocked-task support in the ^^ task link picker"
```

Use the printed path for every read and write. **Do not re-run `sase repo open` after
you start editing** — it cleans the workspace and resets to `origin/master`, discarding
uncommitted work.

### 1. `plugins/block-id-prompt/main.js` — include Blocked tasks

Line 18:

```js
const OPEN_OBSIDIAN_TASK_STATUSES = new Set([" ", "/", "*", "?"]);
```

Add a named constant near it for the status symbol, e.g.
`const BLOCKED_OBSIDIAN_TASK_STATUS = "?";`, and use it in the helpers below rather than
a bare literal.

### 2. `plugins/block-id-prompt/main.js` — unified blocked state

Add a pure helper next to `countBlockedTasks` (~line 2272):

```js
function taskBlockedState(task) {
  const dependency = (task && task.dependency) || null;
  const statusBlocked = Boolean(task) && task.status === BLOCKED_OBSIDIAN_TASK_STATUS;
  const dependencyBlocked = Boolean(dependency && dependency.isBlocked);

  return {
    isBlocked: statusBlocked || dependencyBlocked,
    statusBlocked,
    dependencyBlocked,
    depCount: dependency ? dependency.depIds.length : 0,
    metCount: dependency ? dependency.metCount : 0,
    unmetCount: dependency ? dependency.unmetBlockers.length : 0,
    unresolvedCount: dependency ? dependency.unresolvedIds.length : 0,
    unmetBlockers: dependency ? dependency.unmetBlockers : [],
  };
}
```

**Do not change the meaning of `resolveTaskDependencyState(...).isBlocked`.** It must
keep meaning "has unmet `dependsOn`" — `scripts/test-block-id-prompt.cjs:90` and `:107`
assert exactly that, and `buildTaskDependencyIndex` depends on it. The broader notion
lives only in `taskBlockedState`.

Rewrite `countBlockedTasks` to count `taskBlockedState(task).isBlocked`.

### 3. `plugins/block-id-prompt/main.js` — stable partition

```js
function partitionTaskPickerItems(tasks) {
  const unblocked = [];
  const blocked = [];

  (tasks || []).forEach((task) => {
    (taskBlockedState(task).isBlocked ? blocked : unblocked).push(task);
  });

  return { unblocked, blocked };
}
```

`collectTaskPickerItems` must keep returning **document order** — existing tests index
`[0]` of its result (`test-block-id-prompt.cjs:89`, `:107`, `:361`, `:733`, `:771`).
Ordering is a presentation concern and belongs here.

### 4. `plugins/block-id-prompt/main.js` — reason-specific chip label and tooltip

```js
function blockedChipLabel(state) {
  return state.dependencyBlocked && state.depCount > 1
    ? `Blocked · ${state.unmetCount}`
    : "Blocked";
}
```

The `depCount > 1` guard preserves today's rule (a count is only useful when the task
lists more than one dependency).

Rework `buildBlockedTooltip` (currently `main.js:2282`) to take the unified state and
branch on the reason:

- `state.dependencyBlocked` → today's text, unchanged:
  `Blocked by: <up to 3 blocker titles>[, +N more][; N unresolved]`
- status-blocked only, `depCount === 0` →
  `Marked Blocked in the note; no dependencies listed`
- status-blocked only, `unresolvedCount > 0` →
  ``Marked Blocked in the note; N unresolved dependency `pluralize(N, "reference", "references")` ``
- status-blocked only, all dependencies met →
  ``Marked Blocked in the note; all N `pluralize(N, "dependency", "dependencies")` are done``

The last branch is a genuinely useful diagnostic: it flags a stale `[?]` whose blockers
have all been closed. Reuse the existing `pluralize` helper (`main.js:2131`).

### 5. `plugins/block-id-prompt/main.js` — status pill class

```js
function taskStatusClass(status) {
  if (status === "/") return "active";
  if (status === "*") return "next";
  if (status === BLOCKED_OBSIDIAN_TASK_STATUS) return "blocked";
  return "todo";
}
```

(Keep the file's existing brace style for these branches.)

### 6. `plugins/block-id-prompt/main.js` — grouped rendering

Leave `getFilteredTasks()` as a pure filter (its current contract). Apply ordering in
`renderResults`:

```js
renderResults() {
  if (!this.resultsEl) {
    return;
  }

  const groups = partitionTaskPickerItems(this.getFilteredTasks());
  this.visibleTasks = groups.unblocked.concat(groups.blocked);
  this.selectedIndex = this.clampSelectedIndex(
    this.selectedIndex,
    this.visibleTasks.length,
  );
  this.updateSubtitle();
  this.resultsEl.empty();

  if (this.visibleTasks.length === 0) {
    this.renderEmptyState();
    return;
  }

  const showGroupHeaders =
    groups.unblocked.length > 0 && groups.blocked.length > 0;
  const query = this.getQuery();

  this.visibleTasks.forEach((task, index) => {
    if (showGroupHeaders && index === 0) {
      this.renderGroupHeader("Unblocked", groups.unblocked.length, "unblocked");
    }
    if (showGroupHeaders && index === groups.unblocked.length) {
      this.renderGroupHeader("Blocked", groups.blocked.length, "blocked");
    }

    // ... existing row construction, with `is-dep-blocked` replaced by
    // `is-blocked` driven by taskBlockedState(task).isBlocked ...
  });
}
```

Because `visibleTasks` remains a flat array and rows are still indexed by their position
in it, `moveSelection`, `clampSelectedIndex`, `openTaskAtIndex`, and `scrollRowIntoView`
need **no changes**. Headers are extra DOM nodes, not list entries.

New method:

```js
renderGroupHeader(label, count, variant) {
  const headerEl = this.resultsEl.createDiv({
    cls: `bid-tlp-group is-${variant}`,
    attr: { role: "presentation", "aria-hidden": "true" },
  });
  headerEl.createSpan({ cls: "bid-tlp-group-label", text: label });
  headerEl.createSpan({ cls: "bid-tlp-group-count", text: String(count) });
}
```

`role="presentation"` + `aria-hidden` keeps the `role="listbox"` / `role="option"`
relationship intact — a bare `div` inside a listbox would otherwise break the options
model for screen readers.

In `renderTaskRow`, gate the chip on `taskBlockedState(task).isBlocked` (instead of
`task.dependency.isBlocked`) and source its text from `blockedChipLabel(state)` and its
`title`/`aria-label` from the reworked tooltip. Keep the `lock` icon — one icon for one
concept.

### 7. `plugins/block-id-prompt/styles.css`

- Rename `.bid-tlp-row.is-dep-blocked:not(.is-selected)` (line 173) to
  `.bid-tlp-row.is-blocked:not(.is-selected)`; the orange left border is unchanged.
- Add the pill variant, matching the shape of `.is-active` / `.is-next` (color + tinted
  border, no fill):

  ```css
  .bid-tlp-status-pill.is-blocked {
    color: var(--color-orange, #c1782a);
    border-color: color-mix(in srgb, var(--color-orange, #c1782a) 45%, transparent);
  }
  ```

- Add the sticky group header:

  ```css
  .bid-tlp-group {
    position: sticky;
    top: 0;
    z-index: 1;
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: var(--size-4-2);
    margin: var(--size-4-3) 0 var(--size-2-2);
    padding: var(--size-2-1) var(--size-4-3) var(--size-2-2);
    border-bottom: 1px solid var(--background-modifier-border);
    background-color: var(--background-primary);
    color: var(--text-faint);
    font-size: var(--font-ui-smaller);
    font-weight: var(--font-semibold);
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  .bid-tlp-group:first-child {
    margin-top: 0;
  }

  .bid-tlp-group.is-blocked {
    color: color-mix(in srgb, var(--color-orange, #c1782a) 70%, var(--text-faint));
  }

  .bid-tlp-group-count {
    color: var(--text-faint);
    font-weight: var(--font-normal);
    letter-spacing: 0;
  }
  ```

- Add `scroll-margin-block: 34px;` to `.bid-tlp-row` so
  `scrollIntoView({ block: "nearest" })` never parks the selected row underneath a
  sticky header.
- Give the four existing `var(--color-orange)` uses (lines 174, 232, 234, 235) the same
  `#c1782a` fallback, so the whole file degrades uniformly on themes that omit the
  variable (`.is-next` already sets this precedent with `var(--color-green, #3f8f5f)`).

Keep every value expressed in Obsidian theme variables, per the file's opening comment,
so the picker stays correct in light and dark themes.

### 8. `plugins/block-id-prompt/main.js` — exports

Extend `module.exports.helpers` (keep the list alphabetized) with the new pure helpers
so they are testable: `blockedChipLabel`, `buildBlockedTooltip`, `countBlockedTasks`,
`partitionTaskPickerItems`, `taskBlockedState`, `taskStatusClass`.

### 9. `scripts/test-block-id-prompt.cjs` — coverage

The test harness stubs `obsidian` with empty classes, so `TaskLinkPickerModal` cannot be
instantiated; follow the established pattern and test the **pure helpers** only. Add:

1. `collectTaskPickerItems` now yields `[?]` tasks, and still excludes `[x]`, `[X]`,
   `[-]`, lines without `#task`, and `[?]` lines carrying `#hide`.
2. `taskBlockedState` across all four quadrants: status-only, dependency-only, both,
   neither — asserting `isBlocked`, `statusBlocked`, `dependencyBlocked`, and the
   counts.
3. `partitionTaskPickerItems` sinks blocked tasks and preserves document line order
   inside both groups (use a fixture that interleaves blocked and unblocked lines).
4. `blockedChipLabel` for: single dependency (`"Blocked"`), multiple dependencies
   (`"Blocked · 2"`), status-only (`"Blocked"`).
5. `buildBlockedTooltip` for every branch — dependency list with `+N more`,
   `; N unresolved`, status-only with no dependencies, status-only with unresolved refs,
   status-only with all dependencies done.
6. `taskStatusClass("?") === "blocked"` and the three existing mappings.
7. Regression: `shouldPromoteTaskToNext` returns `false` for a `[?]` task that is the
   sole content of a Pomodoro sub-bullet (mirror the existing fixture at
   `test-block-id-prompt.cjs:487` but with `[?]`).
8. A `[?]` task still counts as an unmet blocker for its dependents via
   `resolveTaskDependencyState` — i.e. Blocked never became "done".
9. `countBlockedTasks` counts both reasons and does not double-count a task that is
   blocked for both.

### 10. `plugins/block-id-prompt/manifest.json` and `README.md`

- Bump `version` `1.4.0` → `1.5.0` (new user-visible behavior; versions are per-plugin,
  no lockstep release).
- Update the `block-id-prompt` row in the README plugin table: the version, and the
  description to note that the task picker lists Blocked `[?]` tasks in a separate
  Blocked group beneath the unblocked ones.

## Verification

From the `bob-plugins` checkout:

```bash
npm test          # includes scripts/test-block-id-prompt.cjs
npm run validate  # manifest fields, id/folder match, semver, node --check on main.js
```

Both must pass before deploying.

### Deploy to the vault

```bash
bob plugins sync -p block-id-prompt -r "<bob-plugins checkout path>" --no-pull --dry-run
bob plugins sync -p block-id-prompt -r "<bob-plugins checkout path>" --no-pull
```

`-r` is required because the command otherwise defaults to a path that does not exist in
a SASE workspace; `--no-pull` avoids a `git pull` against the checkout while it holds
uncommitted work. If sync reports a managed file as dirty in the vault, follow the
verification procedure before reaching for `-F` rather than force-copying blindly. After
the copy, the user must reload the plugin in Obsidian to pick up the new `main.js` /
`styles.css`.

### Manual checks in Obsidian

Against a note containing a mix of statuses:

1. `[[Note]]^^` lists `[?]` tasks; they appear under a **Blocked** header, below the
   unblocked ones.
2. With no blocked tasks present, the list looks exactly as it does today — no headers.
3. With only blocked tasks present, no headers appear, and every row still shows the
   amber `[?]` pill and a "Blocked" chip.
4. Enter on the default selection still links the first unblocked task.
5. Selecting a `[?]` task without a block ID opens the block-ID prompt, writes `^<id>`
   to the task line, and leaves the checkbox as `[?]` — no promotion to `[*]`.
6. Hovering the chip shows the correct reason for each of: unmet dependencies,
   marked-Blocked with no dependencies, and marked-Blocked with all dependencies done.
7. Filtering matches blocked tasks and keeps them grouped below.
8. Scroll a long list: the sticky header does not cover the selected row when arrowing
   through it.
9. Both light and dark themes.

## Non-goals

- **No toggle to hide blocked tasks.** They already sink and never displace anything; a
  keybinding plus persisted state is complexity the request does not need.
- **No changes to `bob-navigation-hotkeys` or `task-status-cycler`.** The `!` dependency
  sync, schedule propagation, and Blocked-dependent recovery are untouched.
- **No date parsing.** See D6.
- **No new filter semantics.** The query still fuzzy-matches the task title only, not
  blocker names.
- **No git commit.** Leave the checkout dirty unless the user explicitly asks for a
  commit.

## Risks

| Risk                                                                                                       | Mitigation                                                                                                                                                                                                        |
| ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Open tasks with unmet `dependsOn` change position (they now sink into the Blocked group).                  | Rare in a `!`-synced vault, where such tasks are already stamped `[?]`; the group header makes the position self-explanatory. Called out here as an intentional behavior change (D3).                             |
| The class rename `is-dep-blocked` → `is-blocked` would break a vault CSS snippet referencing the old name. | No such snippet is known; `main.js` and `styles.css` deploy together, so the plugin's own styling stays consistent. Grep the vault's `.obsidian/snippets/` for `bid-tlp-` before deploying if you want certainty. |
| Sticky headers could interact badly with an unusual theme (transparent modal backgrounds).                 | `scroll-margin-block` covers the selection case; if a theme misbehaves, dropping `position: sticky` from `.bid-tlp-group` degrades cleanly to inline headers with no other changes.                               |
| Widening `OPEN_OBSIDIAN_TASK_STATUSES` accidentally affects unrelated behavior.                            | Verified: in this file the constant is read only by `taskItemFromLine` (the picker) and by the unused, unexported `isOpenObsidianTaskLine`. `npm test` pins the rest.                                             |
