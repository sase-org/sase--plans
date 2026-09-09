---
tier: tale
title: Split blocked.md into reason-routed Tasks sections
goal:
  ~/bob/blocked.md renders seven H2 Tasks sections that partition every blocked task
  exactly once — dependency-blocked, recurring, and four future-schedule horizons, plus
  a self-auditing catch-all — with identical results in Obsidian Tasks and `bob query`.
create_time: 2026-09-09 19:53:02
status: wip
---

# Plan: Split `blocked.md` into reason-routed Tasks sections

## Goal

Replace the single catch-all query in `~/bob/blocked.md` with seven H2 sections, each
holding one `tasks` query. Dependency-blocked tasks separate from future-scheduled
tasks, and the future-scheduled tasks break down into useful, disjoint sets (repeating
rituals + three distance-based horizons + tomorrow).

The design invariant that makes this reliable: **every task the current query returns
lands in exactly one new section — no overlaps, no gaps.** This has already been
verified against the live vault (see [Verification](#verification)).

## Context

- `~/bob/blocked.md` currently holds one query:
  `(is blocked) OR (status.name includes Blocked)` under a single `## BLOCKED Tasks`
  heading. Today that returns 89 tasks.
- Two engines read this note and both must agree:
  1. **Obsidian Tasks plugin v8.2.2** (primary; `taskFormat: dataview`,
     `globalFilter: #task`).
  2. **`bob query --tasks-note blocked.md`** — bob-cli's native Tasks engine (QuickJS +
     vendored moment).
- `TQ_extra_instructions` frontmatter (Tasks "Query File Defaults") applies to _every_
  `tasks` block in the note, in both engines. It is the right home for note-wide
  invariants and the wrong home for per-section grouping.
- `bob task-status-hooks` derives the `[?]` Blocked status from two conditions: an open
  `dependsOn` target, or a task-level `[scheduled:: YYYY-MM-DD]` later than the daily
  anchor. Both reasons currently collapse into one undifferentiated list, which is
  exactly what this plan fixes.
- `dash.md` hides future-scheduled tasks
  (`!task.scheduled.moment || task.scheduled.moment.isSameOrBefore(moment(), "day")`),
  so a task with a future schedule is invisible everywhere except `blocked.md`. The new
  sections therefore route on the _schedule itself_, not on the `[?]` stamp, so the page
  stays correct even before `bob task-status-hooks` runs.

## Design

### Routing rule

Sections are matched by reason, and the reasons are mutually exclusive by construction:

| Section                  | Window / reason                                  | Today's count |
| ------------------------ | ------------------------------------------------ | ------------- |
| `DEPENDENT`              | `is blocked` — waiting on another open task      | 12            |
| `TOMORROW`               | one-off, scheduled tomorrow                      | 11            |
| `SOON (2–7 days out)`    | one-off, scheduled after tomorrow through +7d    | 38            |
| `LATER (8–30 days out)`  | one-off, scheduled +8d through +30d              | 4             |
| `HORIZON (31+ days out)` | one-off, scheduled beyond +30d                   | 10            |
| `RECURRING`              | repeating task awaiting its next run             | 14            |
| `STRANDED`               | `[?]` with no live reason left — should be empty | 0             |

Precedence, stated in the note's intro so a reader can predict where a task lands:
**dependency beats cadence, and cadence beats distance.** A dependency-blocked task
appears under `DEPENDENT` even if it also carries a future schedule; a repeating task
appears under `RECURRING` whatever its next date is. Every date section therefore
carries `is not blocked` and `is not recurring`, which is what makes the partition
provable rather than approximate.

### Design decisions and why

1. **`RECURRING` is pulled out ahead of the date buckets.** 14 of the 77
   future-scheduled tasks repeat, and 10 of them are the `gtd_daily.md` rituals
   scheduled for tomorrow. Left in place they would be half of `TOMORROW` forever.
   Pulled out, `TOMORROW` becomes a genuine preview of tomorrow's Dashboard, and the
   rituals get a cadence view (`group by recurrence`) that is more informative than
   their dates.
2. **Rolling day windows, not calendar weeks.** `this week` / `next week` resolve
   differently across the two engines (bob-cli anchors weeks on Monday; Obsidian's
   moment locale anchors on Sunday), which would silently create overlaps or gaps
   depending on which engine you read the note with. `tomorrow` / `+7d` / `+30d` are
   identical in both.
3. **Window bounds use `filter by function` with moment arithmetic, not relative-date
   keywords.** `today` and `tomorrow` are unambiguous in both engines; bare offsets like
   `before 8 days` are not portable (bob-cli's parser rejects `in 8 days`, and
   Obsidian's chrono parser is not guaranteed to accept `8 days`). The moment idiom is
   already proven in this vault — `dash.md` uses it — and it is exact in both engines.
4. **Null-safety via `?? false`, never `!x || …`.** A guard that returns _true_ for a
   missing date would put an undated task in two windows at once and break disjointness.
   `task.scheduled.moment?.…(…) ?? false` is false for missing dates and always returns
   a strict boolean, which both engines require (`A && B` errors with
   `filtering function must return true or false` when `A` is null).
5. **`group by path` moves out of `TQ_extra_instructions` into the sections that want
   it.** Query File Defaults are applied _before_ a block's own instructions, so leaving
   `group by path` in frontmatter would force path to be the outer grouping in every
   section and demote date grouping to a sub-level. The two `sort by` lines stay in
   frontmatter: date/recurrence groups are ordered by group name, so path+line remains
   exactly the right within-group tiebreak.
6. **`STRANDED` exists to make the partition total.** It should always be empty; if it
   fills up, the `[?]` stamps have drifted from reality and `bob task-status-hooks`
   needs a run. This is what turns the page from a filter into a self-auditing
   partition.
7. **Section counts come from Tasks' own "N tasks" header**
   (`searchResults.taskCountLocation: "top"`), so no `dataviewjs` count widget is
   needed. A hand-written widget would duplicate the routing logic in a second place and
   drift; the queries stay the single source of truth.

### Rejected alternatives

- **Tasks `preset` instructions** (defined in the plugin's `data.json`) to DRY up the
  repeated `+7d`/`+30d` bounds: rejected because it moves the routing logic out of the
  note and into plugin settings, where it is invisible to anyone reading `blocked.md` —
  including agents reading it through `bob`.
- **H3 sub-sections under two H2 reasons**: the request is explicit that each query gets
  its own H2.
- **A `group by recurring` sub-level instead of a `RECURRING` section**: keeps the noise
  inside the date buckets it was meant to drain.

## Implementation

Single step: rewrite `~/bob/blocked.md` with the content below. The vault is edited
directly on disk (this is a note, not a plugin — plugin sources live in the
`bob-plugins` repo, but notes are edited in the vault). Preserve the existing `parent`,
`created`, and `aliases` frontmatter values exactly as they already are in the file;
only `TQ_extra_instructions` changes (one line removed).

````markdown
---
parent: "[[dash]]"
created: 2026-07-13T07:22:18-04:00
aliases:
  - Blocked Tasks
TQ_extra_instructions: |-
  folder does not include _templates
  filter by function task.file.path !== query.file.path
  filter by function !task.tags.includes("#hide")
  sort by function task.file.path
  sort by function task.lineNumber
  short mode
  hide toolbar
---

# Blocked

Tasks that cannot move yet: they are waiting on another task (via `dependsOn`) or on a
future inline `scheduled` date. Everything actionable lives on the [[dash|Dashboard]].

Every blocked task appears in exactly one section below. Dependency beats cadence, and
cadence beats distance: waiting on another task lands in [[#DEPENDENT Tasks]], a
repeating task lands in [[#RECURRING Tasks]], and everything else lands in the section
matching how far out its `scheduled` date is.

## DEPENDENT Tasks

_Waiting on another open task to finish._

```tasks
is blocked
group by path
```

## TOMORROW Tasks

_One-off tasks that land on the [[dash|Dashboard]] tomorrow._

```tasks
is not blocked
is not recurring
scheduled on tomorrow
group by path
```

## SOON Tasks (2–7 days out)

_Arriving this coming week._

```tasks
is not blocked
is not recurring
scheduled after tomorrow
filter by function task.scheduled.moment?.isSameOrBefore(moment().add(7, "days"), "day") ?? false
group by scheduled
```

## LATER Tasks (8–30 days out)

_Committed, but not this week._

```tasks
is not blocked
is not recurring
filter by function task.scheduled.moment?.isAfter(moment().add(7, "days"), "day") ?? false
filter by function task.scheduled.moment?.isSameOrBefore(moment().add(30, "days"), "day") ?? false
group by scheduled
```

## HORIZON Tasks (31+ days out)

_Parked past the planning horizon — scan for anything that should be sooner, or
dropped._

```tasks
is not blocked
is not recurring
filter by function task.scheduled.moment?.isAfter(moment().add(30, "days"), "day") ?? false
group by function task.scheduled.moment?.format("YYYY-MM MMMM") ?? "Undated"
```

## RECURRING Tasks

_Repeating rituals waiting on their next run, grouped by cadence._

```tasks
is not blocked
is recurring
scheduled after today
group by recurrence
```

## STRANDED Tasks

_Marked Blocked with no live reason left. Should be empty — if it is not, run
`bob task-status-hooks`._

```tasks
is not blocked
status.name includes Blocked
NOT (scheduled after today)
group by path
```
````

Notes for the implementer:

- Keep the en dashes (`–`) in the `SOON`/`LATER` headings and the `?.` / `??` operators
  verbatim; both engines parse them, and the null-safety of `?? false` is load-bearing
  for disjointness.
- Do not reintroduce `group by path` into `TQ_extra_instructions`.
- Do not add a `dataviewjs` count widget; Tasks prints per-query counts already.

## Verification

Run all three checks after the edit. Counts will differ from the numbers below as the
vault changes — the invariants (no overlaps, no gaps) are what must hold.

1. **Every block parses and renders in bob-cli:**

   ```bash
   bob query --tasks-note blocked.md --format markdown | head -60
   ```

   Expect seven blocks, none reporting a query error.

2. **The partition is exact.** Save as `/tmp/check_blocked_partition.py` and run it; it
   compares the union of the seven sections against the query this plan replaces:

   ```python
   import json, subprocess, sys

   SECTIONS = {
    "DEPENDENT": ["is blocked"],
    "TOMORROW": ["is not blocked", "is not recurring", "scheduled on tomorrow"],
    "SOON": ["is not blocked", "is not recurring", "scheduled after tomorrow",
             'filter by function task.scheduled.moment?.isSameOrBefore(moment().add(7, "days"), "day") ?? false'],
    "LATER": ["is not blocked", "is not recurring",
              'filter by function task.scheduled.moment?.isAfter(moment().add(7, "days"), "day") ?? false',
              'filter by function task.scheduled.moment?.isSameOrBefore(moment().add(30, "days"), "day") ?? false'],
    "HORIZON": ["is not blocked", "is not recurring",
                'filter by function task.scheduled.moment?.isAfter(moment().add(30, "days"), "day") ?? false'],
    "RECURRING": ["is not blocked", "is recurring", "scheduled after today"],
    "STRANDED": ["is not blocked", "status.name includes Blocked", "NOT (scheduled after today)"],
   }
   OLD = ["(is blocked) OR (status.name includes Blocked)"]

   def run(lines):
       p = subprocess.run(["bob", "query", "--tasks-file", "-", "--origin", "blocked.md", "--format", "json"],
                          input="\n".join(lines) + "\n", capture_output=True, text=True)
       if p.returncode:
           sys.exit(f"query failed for {lines}:\n{p.stderr or p.stdout}")
       result = json.loads(p.stdout)["result"]
       ids = {(t["file"]["path"], t["lineNumber"])
              for g in result.get("groups") or [] for t in g.get("tasks") or []}
       return ids or {(t["file"]["path"], t["lineNumber"]) for t in result.get("tasks") or []}

   old = run(OLD)
   seen, union = {}, set()
   for name, lines in SECTIONS.items():
       ids = run(lines)
       print(f"{name:10} {len(ids):4d}")
       for i in ids:
           seen.setdefault(i, []).append(name)
       union |= ids

   overlaps = {k: v for k, v in seen.items() if len(v) > 1}
   print(f"\noverlaps: {len(overlaps)}  missing: {len(old - union)}  extra: {len(union - old)}")
   for k, v in list(overlaps.items())[:10]:
       print("  overlap", k, v)
   for k in list(old - union)[:10]:
       print("  missing", k)
   assert not overlaps and not (old - union), "partition is not exact"
   print("OK: union == old query, sections are disjoint")
   ```

   Expected output ends with `OK: union == old query, sections are disjoint`. (`extra`
   may be non-zero and is benign: it means a future-scheduled task has not been stamped
   `[?]` yet, which the new page correctly shows and the old query missed.)

3. **Obsidian renders it.** Open `blocked.md` in Obsidian and confirm: seven sections,
   no red query-error text, per-section counts matching step 2, `SOON`/`LATER` grouped
   by date headings (`2026-07-27 Monday`), `HORIZON` grouped by month
   (`2026-10 October`), `RECURRING` grouped by rule (`every day when done`), `STRANDED`
   empty.

## Out of scope

- **`dash.md`'s BLOCKED chip.** Its `dataviewjs` count uses the old predicate
  (`isBlocked || status.name includes Blocked`), so it will undercount by exactly the
  tasks in the `extra` bucket of check 2 — future-scheduled tasks that
  `bob task-status-hooks` has not stamped yet. Harmless and self-correcting on the next
  hook run; worth a separate change only if that lag becomes visible in practice.
- **bob-cli's boolean parser.** `(A) AND (B) AND (C)` on one line fails with
  `Could not interpret the following instruction as a Boolean combination`, while
  Obsidian Tasks accepts three-way chains. This plan sidesteps it entirely (one filter
  per line, implicitly ANDed, which is also more readable), but it is a real divergence
  in `src/native/dataview/tasks/parse.rs` worth its own fix.
