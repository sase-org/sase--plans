---
tier: epic
title: Priority bullet property that rolls a scheduled date
goal: "The Ctrl+Shift+P bullet-property picker offers a `priority` property whose
  P2/P3/P4 levels each write a random `scheduled` date drawn from a per-level day range
  configured in bob/config.yml, and the `scheduled` date picker offers a visually
  distinct re-rollable date suggestion whenever the current task already has a priority.

  "
phases:
  - id: config
    title: "Config schema for `values: priority`"
    depends_on: []
    size: medium
    description: "config: teach the bullet-property config loader a `values: priority`
      kind with a validated `levels` list (label, value, day range) and a `schedules`
      target, and add the `priority` entry to the chezmoi-managed bob/config.yml.

      "
  - id: picker
    title: Priority value stage and single-task write
    depends_on:
      - config
    size: medium
    description: "picker: render the P2/P3/P4 value stage, roll a random date from the
      chosen level's day range, and write the priority field plus the derived scheduled
      value in one guarded edit that reuses the existing Blocked/recovery and
      project-frontmatter behavior.

      "
  - id: counted
    title: Counted-session priority writes
    depends_on:
      - picker
    size: medium
    description: "counted: extend the counted batch planner so N<Ctrl+Shift+P> applies
      one priority to every counted task while rolling an independent scheduled date per
      task.

      "
  - id: suggest
    title: Priority-derived suggestion in the date picker
    depends_on:
      - picker
    size: medium
    description: "suggest: pin a visually distinct, re-rollable suggested date at the
      top of the `scheduled` value stage whenever the current task (or every counted
      task) already has a priority.

      "
  - id: release
    title: Docs, version bump, and vault deploy
    depends_on:
      - counted
      - suggest
    size: small
    description:
      "release: document the new property kind, bump the plugin manifest and README, and
      deploy the plugin to the vault."
proposed_by: bbugyi200.athena.s8
create_time: 2026-09-09 20:00:21
status: wip
---

- **PROMPT:**
  [prompts/202608/priority_property.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/priority_property.md)

# Plan: Priority bullet property that rolls a scheduled date

## Repositories

This epic spans two repositories plus documentation in the primary repo. Phase agents
MUST open the non-primary repositories through the `/sase_repo` skill and use the
printed path for every read and write:

- `sase repo open chezmoi -r "<reason>"` — owns `home/dot_config/bob/config.yml`, the
  source of the deployed `~/.config/bob/config.yml` that the plugin reads. After
  committing there, run `chezmoi update -a --force` so the change reaches `~/.config/`.
- `sase repo open bob-plugins -r "<reason>"` — owns
  `plugins/bob-navigation-hotkeys/main.js`, `styles.css`, `manifest.json`,
  `scripts/test-navigation-hotkeys.cjs`, and `README.md`.
- The primary `bob-cli` repo owns `docs/projects.md`.

Run the plugin test suite from the `bob-plugins` checkout with `npm test` (it shells out
to `node --test scripts/*.cjs`); `npm run validate` checks manifests.

## Background

`Ctrl+Shift+P` ("Set bullet property") opens `BulletPropertyPickerModal` in
`plugins/bob-navigation-hotkeys/main.js`. It is a two-stage picker:

1. **Property stage** — one row per entry in the `properties` list of
   `~/.config/bob/config.yml`.
2. **Value stage** — depends on the entry's `values`:
   - `values: date` → `createBulletPropertyDateItems()` renders ten relative-date
     presets, and typing `2026-08-06`, `6/24`, or `+3d` adds a live "Use …" row via
     `createBulletPropertyTypedDateItem()`.
   - `values: local_task_id` → a task picker for `[dependsOn:: …]`.
   - a scalar list → one row per literal value.

Selecting a value calls `BulletPropertyPickerModal.applySelectedValue()`, which routes
to one of three writers depending on the resolved target:

- `setCountedBulletPropertyValue()` for an explicit-count session (`N<Ctrl+Shift+P>`),
- `setProjectNoteScheduledValue()` when `resolveBulletPropertyTarget()` says
  `project-frontmatter` (the `scheduled` property on a `^prj` project lifecycle task
  writes YAML, not an inline field),
- `setBulletPropertyValue()` for the ordinary inline case.

`setBulletPropertyValue()` already carries the `scheduled`-specific side effects that
this feature must not bypass: a future date marks the task Blocked via
`blockObsidianTaskCheckboxStatus()`, and a due date runs
`buildTargetScheduledRecoveryByLine()` / `reconcileBlockedScheduledTaskLine()` behind a
staleness guard.

## Design constraint: what may be written to `[priority:: …]`

**This constraint drives the whole design; do not silently drop it.**

The vault's Obsidian Tasks plugin is configured with `taskFormat: dataview` and the
global filter `#task` (`~/bob/.obsidian/plugins/obsidian-tasks-plugin/data.json`). In
that format `priority` is a **reserved Tasks metadata key** whose only legal values are
`highest`, `high`, `medium`, `low`, and `lowest`.

Tasks-format parsers consume trailing inline fields **right to left and stop at the
first unrecognized field**. Both parsers this vault depends on behave that way:

- `bob-cli` `src/native/dataview/tasks/task.rs` — `parse_details()` loops over
  `try_take_dataview_field()`, whose `"priority" => Priority::from_name(value)` arm
  returns `false` for anything outside the five legal names; the loop then `break`s. An
  existing test asserts that after an unrecognized field the whole remaining line stays
  in the task _description_.
- `bob-plugins` `parseTrailingRecoveryTaskMetadata()` in `main.js` has the same
  allow-list and the same `break`.
- The real Obsidian Tasks plugin, which renders every query in the vault, parses the
  same way.

So writing a literal `[priority:: P2]` would not merely be "unrecognized" — it would
**halt parsing of every inline field to its left on that line**. A task carrying
`[scheduled:: …]`, `[id:: …]`, or `[dependsOn:: …]` would lose all of them from
`bob query`, the dash, dependency handling, and scheduled-based Blocked/recovery, and
would render the raw `[priority:: P2]` text inside its description.
`upsertBulletProperty()` appends new fields at the end of the line, which is exactly the
position that causes this.

**Therefore: the picker displays `P2`/`P3`/`P4`, and the file stores the Tasks-native
equivalent.** The label→value mapping is config data, not code, so it is visible and
adjustable in one place:

| Picker label | Written value         | Random `scheduled` window |
| ------------ | --------------------- | ------------------------- |
| (unset)      | _no field_            | — (implicitly `P1`)       |
| `P2`         | `[priority:: high]`   | 2–7 days from today       |
| `P3`         | `[priority:: medium]` | 8–30 days from today      |
| `P4`         | `[priority:: low]`    | 31–90 days from today     |

A confirming data point: `bob query` finds **zero** tasks in the vault that currently
use any Tasks priority (neither `[priority:: …]` nor `⏫`/`🔼`/`🔺`), so there is
nothing to migrate and no existing query whose ordering this could disturb.

Two consequences worth stating plainly, so they can be overruled at review time rather
than discovered later:

1. Obsidian Tasks ranks an unset priority as `normal`, which sits _between_ `medium` and
   `low`. No assignment of three Tasks priorities can put "unset" first, so
   `P1 < P2 < P3 < P4` is **not** expressible as a Tasks _priority_ sort. It does not
   need to be: this feature encodes urgency in the `scheduled` date, and a P1 task — no
   priority, scheduled today or overdue — already sorts first in any scheduled-ordered
   view.
2. Anyone reading the raw Markdown sees `[priority:: high]`, not `P2`. Every surface the
   picker controls (property row, value stage, notices, the date-picker suggestion)
   shows `P2`.

If the user prefers literal `P2` in the file despite the parsing damage, only
`home/dot_config/bob/config.yml` changes — set `value: P2`. No plugin code depends on
the stored spelling.

## Config schema for `values: priority`

### chezmoi: `home/dot_config/bob/config.yml`

Replace the file with:

```yaml
# Bullet properties offered by the Obsidian "Set bullet property" picker (Ctrl+Shift+P).
# `values` is the literal string `date` (YYYY-MM-DD), the literal string `local_task_id`
# (the identifier of an open #task chosen from the current file), the literal string
# `priority` (a P-level that also rolls a random date into another date property), or a
# list of allowed values.
properties:
  - name: scheduled
    values: date
  - name: dependsOn
    values: local_task_id
  # Choosing a level writes `[priority:: <value>]` and rolls a random date into the
  # property named by `schedules`, drawn inclusively from that level's day window.
  # A task with no priority field is implicitly P1: do it now, no roll.
  #
  # `value` is what lands in the note and MUST stay one of Obsidian Tasks' priority
  # names (highest, high, medium, low, lowest). Tasks-format parsers read trailing
  # inline fields right to left and stop at the first one they do not recognize, so an
  # invented value such as `P2` would also hide every `[scheduled:: ]`, `[id:: ]`, and
  # `[dependsOn:: ]` field to its left from Obsidian Tasks, `bob query`, and the dash.
  # `label` is what the picker shows.
  - name: priority
    values: priority
    schedules: scheduled
    levels:
      - label: P2
        value: high
        min_days: 2
        max_days: 7
      - label: P3
        value: medium
        min_days: 8
        max_days: 30
      - label: P4
        value: low
        min_days: 31
        max_days: 90
```

### Plugin: parsing and validation

In `main.js`, `normalizeBulletPropertyValues(name, values, options)` currently returns
`"date"`, `"local_task_id"`, or a frozen array. It cannot express the extra keys a
priority entry needs, so promote per-entry normalization into
`validateBulletPropertyConfig()`:

- Add `"priority"` as a third literal `values` kind.
- When `values === "priority"`, read the sibling `levels` and `schedules` keys off the
  same entry and produce a frozen property object of the shape:

  ```js
  Object.freeze({
    name: "priority",
    values: "priority",
    schedules: "scheduled",
    levels: Object.freeze([
      Object.freeze({ label: "P2", value: "high", minDays: 2, maxDays: 7 }),
      // …
    ]),
    levelsByValue: /* Map from written value to level */,
  })
  ```

- Keep every other `values` kind byte-identical in shape so no existing call site
  changes.

Validation rules, each with its own `showBulletPropertyNotice()` message that names the
offending property and level (match the existing message voice, e.g.
`Bullet property "priority" level #2 …`):

- `levels` must be a non-empty list; a `levels` key on a non-`priority` entry is an
  error.
- Each level needs a non-empty string `label`, unique within the property.
- Each level needs a non-empty scalar `value`, unique within the property, containing
  none of `[`, `]`, `::`, or a newline (it is interpolated into `[priority:: <value>]`).
- `min_days` and `max_days` must be integers with `0 <= min_days <= max_days`. Reject
  non-integers, negatives, and inverted ranges.
- `schedules` is optional and defaults to `"scheduled"`. It must name **another**
  property in the same config whose `values` is `date`; otherwise the config is
  rejected. This is what makes "priority sets scheduled" data rather than a hardcoded
  string, and it fails loudly if the two entries ever drift apart. Because it is a
  cross-entry check, run it after all entries are normalized.
- A `priority` entry whose `name` duplicates another property is already caught by the
  existing duplicate-name check; keep that behavior.

Existing invalid-config behavior is unchanged: one `Notice`, `null` returned, picker
does not open.

### Tests for this phase

Add to `scripts/test-navigation-hotkeys.cjs`, driving
`helpers.validateBulletPropertyConfig()` directly with plain JS objects (the suite's
`parseTestYaml()` obsidian stub is flat-only and cannot parse the nested `levels` list;
do not route these tests through `loadBulletPropertyConfig()` unless you pass a richer
`parseYaml` through its options):

- A valid priority config normalizes to the frozen shape above, including
  `minDays`/`maxDays` camelCase and the default `schedules: "scheduled"`.
- Each rejection rule above produces `null` and exactly one notice naming the property.
- The existing `date`, `local_task_id`, and scalar-list entries still normalize
  unchanged.

## Priority value stage and single-task write

### Rolling a date

Add a pure helper next to the other date helpers, and export it from
`module.exports.helpers`:

```js
function rollPriorityScheduledDate(level, baseDate, random = Math.random) {
  const span = level.maxDays - level.minDays + 1;
  const offset = level.minDays + Math.floor(random() * span);
  return addLocalDateDays(getLocalDateStart(baseDate), offset);
}
```

Both ends of the configured window are inclusive. `random` is injected so tests are
deterministic; production callers pass nothing. Clamp the computed offset into
`[minDays, maxDays]` so a `random()` that returns exactly `1` cannot overshoot.

### Value stage rows

Extend `createBulletPropertyValueItems()` with a `values === "priority"` branch that
returns one row per configured level, in config order:

- `kind: "value"`, `value: level.value`, `label: level.label`.
- `detail`: `` `${level.value} · in ${level.minDays}–${level.maxDays} days` `` — the row
  states both what lands in the file and what window the roll uses, which is where the
  label/value split stops being surprising.
- `current: level.value === currentValue`, so an already-prioritized task shows its
  level with the existing `check-circle-2` icon and `current` pill.
- `priorityLevel: level`, carried through to the writer.
- `searchText`: label, value, and the day range, so `P3`, `medium`, and `30` all match.

Show exactly the configured levels — **no synthetic `P1` row**, per the request that the
property offer only P2/P3/P4. `Ctrl+D` on the property row already clears
`[priority:: …]`, which is the way back to implicit P1; make sure the stage-one footer
hint still reads `^D Delete` for this property. Deleting a priority deliberately leaves
`scheduled` alone — the date is a real commitment once made.

In `showValueStage()`, give the priority stage its own chrome:
`headerIcon: "signal-high"`, `placeholder: "Filter priorities"`,
`resultsLabel: "priority levels"`, and a subtitle that reads
`Choose a level · rolls a scheduled date` (append `· current: P2` when set, mapping the
stored value back through `levelsByValue`).

In `renderValueItem()`, add a `bob-cnp-priority-value-row` class and a trailing pill
showing the window (e.g. `2–7d`) via the existing `bob-cnp-pill` styling. Add the class
to `styles.css` next to the other value-row rules, using Obsidian variables in the same
`color-mix(in srgb, var(--…))` idiom already used there — no literal colors.

### Property-stage display

A task with `[priority:: high]` should read `priority · P2` in the property stage, not
`priority · high`. In `createBulletPropertyItems()` (and the counted equivalent) add a
`currentLabel` field for priority properties resolved through `levelsByValue`, falling
back to the raw value when a hand-written value matches no level. Render `currentLabel`
in `renderPropertyItem()` and include it in the stage-one `filterItem()` search text so
typing `P2` finds the row.

### Writing

Selecting a level must produce **one** guarded edit and **one** notice covering both
fields.

`setBulletPropertyValue()` already contains the scheduled status logic that must be
reused. Rather than duplicating it, extract its body into a shared private method that
takes an ordered list of `{ name, value }` inline-field edits plus the scheduled value
used for the status decision, then:

- `setBulletPropertyValue()` calls it with a single edit (behavior byte-identical to
  today — verify against the existing tests, which must keep passing unchanged).
- A new
  `setBulletPriorityValue(cm, cursor, filePath, lineText, property, level, context)`
  calls it with
  `[{ name: property.name, value: level.value }, { name: property.schedules, value: rolled }]`.
  Apply the priority edit first so the scheduled field stays rightmost, which is where
  the right-to-left Tasks parsers expect date metadata.

`applySelectedValue()` routes to `setBulletPriorityValue()` when
`this.selectedPropertyItem.property.values === "priority"`.

**Project lifecycle tasks.** When the cursor is on a `^prj` task,
`resolveBulletPropertyTarget("scheduled", context)` returns `project-frontmatter`,
meaning the date belongs in the project note's YAML, not in an inline field. The
priority writer MUST honour that: resolve the target for `property.schedules` (not for
`priority`, which is always inline) and, when it is `project-frontmatter`, write
`[priority:: …]` inline on the `^prj` line and send the rolled date through
`setProjectNoteScheduledValue()` so schedule propagation, `#hide` removal, and the
Blocked/recovery decision all still run. Give `setProjectNoteScheduledValue()` an
optional notice-prefix (or notice-suppression) option so the combined operation still
emits a single notice rather than two.

### Notices

Single task, ordinary inline case:

```
priority → P2 (high); scheduled → 2026-08-06 · Thu; marked task Blocked
```

Reuse the existing `blocked` and `scheduledRecoveryNoticeSuffix()` fragments verbatim so
the Blocked/recovery vocabulary stays identical to a manual `scheduled` edit. Because
every shipped window starts at `min_days >= 2`, the future/Blocked branch is the normal
outcome; keep the due-date recovery branch wired anyway, since a config may legitimately
set `min_days: 0`.

### Tests for this phase

- `rollPriorityScheduledDate()` with a stubbed `random` returning `0`, `0.5`, and
  `0.999…` yields the window's first, middle, and last day; a `random` returning exactly
  `1` still yields `max_days`.
- `createBulletPropertyValueItems()` on a priority property returns the levels in config
  order with the right `label`, `value`, `detail`, `searchText`, and `current` flag.
- End-to-end through the existing picker harness (`createBulletPropertyPickerHarness()`,
  extended with a priority-bearing config and an injected deterministic roll): choosing
  `P2` on `- [ ] #task One ^one` produces a line carrying both `[priority:: high]` and
  the rolled `[scheduled:: …]`, marks the task Blocked, and emits exactly one notice.
- Choosing a level on a task that already has `[priority:: medium] [scheduled:: <old>]`
  replaces both values in place and does not duplicate either field.
- A `^prj` lifecycle task writes `[priority:: …]` inline and the rolled date into
  project frontmatter, leaving no inline `[scheduled:: …]` on the `^prj` line.
- The pre-existing `setBulletPropertyValue()` tests still pass after the refactor.

## Counted-session priority writes

`N<Ctrl+Shift+P>` snapshots the current `#task` plus the next N real tasks and routes
writes through `setCountedBulletPropertyValue()` → `planCountedBulletPropertyBatch()`.
That planner is built around a single `(name, value)` pair applied to every target, and
it decides `shouldBlockInlineTasks` / `shouldRecoverInlineTasks` **once** for the whole
batch.

Priority needs a shared priority value but an **independent roll per task** — spreading
a batch of newly-prioritized tasks across the window is the point, and a shared date
would defeat it.

Extend the planner with a `"set-priority"` operation:

- `options.priorityValue` — the level's written value, identical for every target.
- `options.scheduledValueByLine` — a `Map` keyed by each target's **original** line
  number, holding that target's rolled `YYYY-MM-DD`. The caller rolls once per target so
  tests can inject a deterministic sequence.
- Inside the per-target loop, apply both `upsertBulletProperty()` calls (priority first,
  scheduled second) and evaluate the future-Blocked / due-recovery decision **per
  target** against that target's own date instead of the batch-level flags. Keep the
  existing batch-level flags for the `set` and `delete` operations untouched.
- If a counted session includes a `^prj` lifecycle task, that target's own rolled date
  drives the existing `project-frontmatter` branch (`planProjectScheduledUpdate()`), and
  the `^prj` line keeps no inline `scheduled` field — exactly as the current `scheduled`
  batch behaves.

`setCountedBulletPropertyValue()` gains a sibling (or an options argument) that rolls
the map, runs the existing staleness guard, applies the transaction, and emits one
notice:

```
priority → P2 (high) on 3 tasks; scheduled rolled per task; marked 3 tasks Blocked
```

Reuse `formatCountLabel()`, `getCountedTaskNoticeSuffix()`, and
`scheduledRecoveryNoticeSuffix()`.

The counted property stage aggregates values across targets via
`createCountedBulletPropertyItems()` — confirm the priority row reports `common` /
`mixed` correctly and that the `mixed` subtitle shows labels (`P2, P4`) rather than raw
values.

### Tests for this phase

- Three counted tasks, injected rolls of the window's first/middle/last day: each task
  gets `[priority:: high]` plus its own distinct `[scheduled:: …]`, all three become
  Blocked, and the notice reports three tasks.
- A counted session mixing a `^prj` lifecycle task with ordinary tasks writes the
  project date to frontmatter and inline dates to the others.
- A stale session (content changed under the picker) aborts with the existing "no tasks
  were updated" notice and leaves the buffer untouched.
- Existing counted `scheduled` and `dependsOn` tests still pass.

## Priority-derived suggestion in the date picker

When the user opens `scheduled` on a task that already has a priority, the date stage
should offer a fresh roll from that priority's window, pinned first and unmistakably
different from the ten fixed presets.

- In `showValueStage()` for a `date` property, look up the config's priority property
  (the one whose `schedules` names this date property), read the current task's stored
  priority value, and resolve it through `levelsByValue`. No priority, or a stored value
  matching no level → no suggestion row, and the stage renders exactly as it does today.
- When it resolves, prepend one item built by a new
  `createPriorityRollDateItem(level, baseDate, currentValue, random)`:
  - `label`: `` `${level.label} roll` `` (e.g. `P2 roll`).
  - `detail`:
    `` `${value} · ${weekday} · random in ${level.minDays}–${level.maxDays} days` ``.
  - `priorityRoll: true`, plus `level`, so rendering and re-rolling can find it.
  - `searchText` includes the label, the date, the weekday, and `random priority`.
- Rendering (`renderValueItem()`): the row gets `is-priority-roll` in addition to the
  existing classes, the `dices` Lucide icon instead of `circle`/`calendar-plus`, and a
  trailing `bob-cnp-pill` reading the level label. Style it in `styles.css` with a left
  accent border and a tinted background derived from `var(--text-accent)` via
  `color-mix()`, distinct from both `.is-current` (which uses `--interactive-accent`)
  and `.is-dynamic`. Confirm it stays legible when the row is also `.is-selected`.
- **Re-roll**: `Ctrl+R` while the date stage is showing a suggestion re-rolls it,
  re-renders, and keeps the selection on that row. Handle it in
  `BulletPropertyPickerModal.handleKeydown()` in the same shape as the existing `Ctrl+D`
  branch, and add `{ keys: ["^R"], label: "Re-roll" }` to the stage-two footer hints
  **only** while a suggestion row exists, so the footer never advertises a key that does
  nothing.
- Choosing the row writes through the ordinary `scheduled` path (`applySelectedValue()`
  needs no special case — the item carries a plain `value`), so Blocked/recovery,
  project-frontmatter routing, and counted batching all behave exactly as for a preset
  date. In a counted session the suggestion appears only when **every** counted task
  shares one priority, and selecting it applies that single date to all of them;
  per-task rolls remain the priority stage's job.
- The suggestion is generated once per stage entry (and once per `Ctrl+R`), never on
  each keystroke, so typing in the filter box does not make the date jump.

### Tests for this phase

- A task with `[priority:: high]` produces a first item flagged `priorityRoll` whose
  date lies inside the 2–7 day window; a task with no priority, or with
  `[priority:: highest]` (no matching level), produces the unchanged ten-item list.
- Re-rolling with an injected sequence replaces the item's value and leaves the rest of
  the list and the selected index untouched.
- Selecting the suggestion writes the same line shape as selecting a preset date.
- A counted session with mixed priorities shows no suggestion.

## Docs, version bump, and vault deploy

- `bob-plugins/README.md`: bump the Bob Navigation Hotkeys row's version and extend its
  description with the priority property and the date-picker roll.
- `plugins/bob-navigation-hotkeys/manifest.json`: bump `version` (a feature addition —
  `1.14.0`) and refresh `description`. Run `npm run validate`.
- `bob-cli` `docs/projects.md`: after the "Scheduling from the `^prj` task" section,
  document the `priority` property — the label/value split and **why** it exists (the
  Tasks trailing-field constraint above), the three windows, `Ctrl+D` clearing a
  priority without touching `scheduled`, the per-task rolls in counted sessions, and the
  `Ctrl+R` re-roll in the date picker.
- Full test suite green: `npm test` in `bob-plugins`.
- Deploy: from the `bob-plugins` checkout printed by `sase repo open`, preview then
  apply — `bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run`, then the
  same without `--dry-run`. `-r "$PWD"` is required because the default source path does
  not exist in a SASE workspace. If sync reports the file dirty in the vault, verify the
  on-disk vault file matches the committed baseline before considering `--force`.
- Apply the config: commit in `chezmoi`, then run `chezmoi update -a --force` and
  confirm `~/.config/bob/config.yml` contains the new entry.
- Tell the user to reload the plugin in Obsidian (the vault copy of `main.js` is only
  picked up on reload).

## Out of scope

- Any change to `bob-cli`'s Rust Tasks parser. The design deliberately writes values
  that parser already accepts.
- Teaching Obsidian Tasks or `bob query` a `P1`-first priority ordering — not
  expressible; ordering comes from `scheduled`.
- Re-rolling or migrating the schedules of tasks that already carry a priority.
