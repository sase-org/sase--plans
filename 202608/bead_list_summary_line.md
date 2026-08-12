---
tier: tale
title: Summary line for sase bead list
goal:
  Every non-empty `sase bead list` run ends with a blank line and one compact, colored
  summary line that describes exactly the rows printed above it, discloses how many
  matching beads were hidden by a limit, and shares its numbers with the `--format json`
  envelope.
size: medium
proposed_by: bbugyi200.athena.yj
create_time: 2026-08-12 10:48:56
status: wip
---

# Plan: summary line for `sase bead list`

## Problem

`sase bead list` prints rows and stops. The reader has to count glyphs by hand to answer
"how many are in progress?", and — worse — a closed listing silently prints only the
newest 20 with nothing on screen saying that 43 more matched. The command's most
misleading behavior is exactly the one it never mentions.

Add one line at the end: after a blank line, a single summary that describes the
listing.

## Design

### The invariant that drives everything

**Every number in the summary is verifiable by counting the rows printed above it,
except one clause that explicitly counts the beads that are _not_ on screen.**

That rule decides all the ambiguous cases below. The summary never characterizes beads
the reader cannot see, so it can never contradict the screen.

### The line

```
<head>[ · <type group>][ · <status group>][ · <hidden clause>]
```

Separators are `·` — the same separator the rows already use between the ID and the
title. Entries _within_ a group are separated by two spaces. That two-level hierarchy
(space inside a group, `·` between groups) is what makes the line scannable without
labels.

**Head** — `{shown}[ {status-adjective}]{ }{type-noun}`

- `type-noun` is the shared type's noun when every printed row has the same type
  (`plan`/`plans`, `phase`/`phases`, `task`/`tasks`), otherwise the neutral
  `bead`/`beads`. Singular when `shown == 1`.
- `status-adjective` is present only when every printed row has the same status: `open`,
  `claimed`, `ready`, `snoozed`, `in-progress`, `closed`.
- Both are derived from the printed rows only.

**Type group** — `▸ 3  ↳ 3`, in canonical `BEAD_TYPE_VALUES` order (plan, phase, task),
zero buckets omitted.

**Status group** — `○ 5  ◐ 1`, in canonical `bead_status_display_order()` order (open,
claimed, ready, snoozed, in_progress, closed), zero buckets omitted.

**Group omission rule** — a group is omitted exactly when the head already carries its
word. One distinct type ⇒ the head folded the type noun ⇒ no type group. One distinct
status ⇒ the head folded the status adjective ⇒ no status group. The summary therefore
never says the same thing twice, and a group is never silently dropped without the head
picking up the information.

**Hidden clause** — present iff `matched > shown`:

- `5 hidden` when the user passed `--limit` explicitly.
- `5 hidden (--limit 0 shows all)` when the truncation came from SASE's own
  `DEFAULT_CLOSED_LIST_LIMIT`, i.e. the user never passed `--limit`.

`hidden` is deliberately its own noun phrase rather than `20 of 63 …`: it inherits no
adjective from the head, so it counts beads without claiming anything about them. And
teaching the escape hatch only when the truncation was _our_ idea, never when the user
asked for it, keeps the hint from becoming noise.

### Worked examples

| Situation                                         | Line                                               |
| ------------------------------------------------- | -------------------------------------------------- |
| 6 rows, plans+phases, open+in_progress, all shown | `6 beads · ▸ 3  ↳ 3 · ○ 5  ◐ 1`                    |
| `--limit 2`, 6 matched, both open, plan+phase     | `2 open beads · ▸ 1  ↳ 1 · 4 hidden`               |
| 3 rows, all closed, plans+phases, all shown       | `3 closed beads · ▸ 2  ↳ 1`                        |
| `-n 1`, 3 matched, one closed plan                | `1 closed plan · 2 hidden`                         |
| `--status closed`, 20 of 25, implicit limit       | `20 closed plans · 5 hidden (--limit 0 shows all)` |
| `--status closed --limit 0`, 25 closed plans      | `25 closed plans`                                  |

The line's length is bounded by construction — at most one head, three type buckets, six
status buckets, and one hidden clause — so it stays one line for a store of any size. No
wrapping or truncation logic is needed, and none should be added.

### Color

Reuse the row vocabulary so the summary reads as a legend for what the reader just
scanned, honoring the existing `-c, --color`:

- Group glyphs: the same `cli_style` the rows use (`bead_status_presentation(...)`,
  `bead_type_presentation(...)`). Counts stay uncolored, exactly like the rows color the
  glyph and not the title.
- A folded status adjective takes that status's `cli_style`; a folded type noun takes
  that type's `cli_style`; the neutral `bead`/`beads` and every number stay plain.
- The hidden clause renders dim (`ansi_sgr(dim=True)`), never grey `\x1b[90m` — grey
  already means _snoozed_ in this vocabulary and must not be borrowed for meta-text.
- Separators stay plain.

### Format coverage

- `compact`: blank line, then the summary.
- `full`: blank line, then the summary, using the same resolved `use_color`. The detail
  blocks themselves render plain (`render_issue_detail` defaults to `DetailStyle.PLAIN`
  here), so a colored footer is the only styled text on that surface — which is the
  point: it anchors the bottom of a long plain dump. `--color never` still silences it.
- `json`: **no prose line.** The envelope instead gains `by_type` and `by_status` count
  maps, built from the same summary object that renders the prose, so the two encodings
  can never disagree. Both maps always carry every bucket including zeros, because a
  stable key set is what machine consumers want; `count`/`total` already carry
  shown/matched.
- Empty listing: unchanged. `No issues found.` returns early and prints no summary —
  `0 beads` under "No issues found." is noise, and this keeps `list_empty.stdout` and
  the empty-JSON golden byte-identical.

### Why this lives in Python, not `sase-core`

`sase bead list` is deliberately excluded from the Rust fast path
(`src/sase/main/bead_fast_path.py:37` returns `None` for `list` and `show`), so
`handle_bead_list` is the only implementation the CLI reaches. The aggregation is a
`Counter` over a list Python already holds in memory; routing it through a binding would
add a round trip to compute `len()`. The presentation half — glyphs, accents, wording —
already lives in the top-level `src/sase/bead_*_presentation.py` family, which is the
established home for cross-surface bead vocabulary.

`crates/sase_core/src/bead/cli.rs::handle_list` exists but is unreachable from the CLI
and already omits the size column, badges, and creation cell that Python renders. Do
**not** touch it in this plan; re-enabling that path is a separate change that would
need a `sase-core` release and a dependency-floor bump.

### Non-goals

- `sase bead ready`, `bead blocked`, and `bead search` keep their current footers (or
  lack of one). The new module is shaped so they can adopt it later, but unifying them
  is a separate change.
- No new CLI flag. `--format json` is the machine-readable surface, and `--color never`
  already covers the styling axis; a `--no-summary` flag would be surface with no
  demonstrated need.

## Implementation

### 1. New module `src/sase/bead_summary_presentation.py`

Public surface:

```python
class BeadSummaryRow(Protocol):
    @property
    def issue_type(self) -> object: ...
    @property
    def status(self) -> object: ...


@dataclass(frozen=True)
class BeadListSummary:
    shown: int
    matched: int
    by_type: Mapping[BeadTypeValue, int]      # every bucket, zeros included
    by_status: Mapping[BeadStatusValue, int]  # every bucket, zeros included

    @property
    def hidden(self) -> int: ...              # max(0, matched - shown)


def summarize_bead_rows(
    rows: Iterable[BeadSummaryRow], *, matched: int
) -> BeadListSummary: ...


def bead_list_summary_line(
    summary: BeadListSummary, *, use_color: bool, implicit_limit: bool
) -> str: ...
```

Notes that matter:

- The `Protocol` uses **read-only properties**, not bare annotations. Read-only property
  protocols are covariant, so `Issue` (whose `issue_type` is `IssueType`, not `object`)
  satisfies it; a bare `issue_type: object` annotation would be invariant and mypy would
  reject `Issue`. This has been verified against the repo's mypy settings.
- The module must **not** import from `sase.bead`. `src/sase/bead/__init__.py` pulls in
  `BeadProject`, and the sibling presentation modules stay import-light on purpose. The
  Protocol is what buys that decoupling while keeping the call site plain.
- `summarize_bead_rows` normalizes each value through the existing
  `bead_type_presentation` / `bead_status_presentation` normalizers and raises on an
  unknown value, matching `bead_type_cli_cell`'s stance: a normalization failure is a
  bug, not missing data, and must fail loudly rather than print a wrong count.
- Wording lives here as two explicit module-level maps — singular/plural type nouns
  keyed by `BeadTypeValue`, status adjectives keyed by `BeadStatusValue` — not as new
  fields on `_BeadTypePresentation`/`_BeadStatusPresentation`. Only summary surfaces
  need these words today, and this module is the shared home for them; a test asserts
  each map covers exactly `BEAD_TYPE_VALUES` / `BEAD_STATUS_VALUES` so a new type or
  status cannot be added without supplying its word. Leaving the two shared dataclasses
  untouched also keeps their exhaustive field-value tests green.
- Renderer order: head, then type group (if not folded), then status group (if not
  folded), then hidden clause (if `hidden`), joined with `·`.
- `shown == 0` is not reachable from the CLI, but the pure functions must still behave
  (empty maps, `0 beads`); cover it so a future caller cannot trip over it.

### 2. Wire it into `src/sase/bead/cli_query.py`

In `handle_bead_list`:

- Capture `explicit_limit = getattr(args, "limit", None) is not None` **before** the
  block that substitutes `DEFAULT_CLOSED_LIST_LIMIT`, and pass
  `implicit_limit=not explicit_limit` to the renderer. That branch is the only way
  `limit` becomes non-`None` without the user, so this is exact.
- Build the summary once, after the `issues[-limit:]` slice, with the already-computed
  `total` as `matched`: `summary = summarize_bead_rows(issues, matched=total)`.
- `compact` and `full`: after the existing `print(..., end="")`, emit
  `print(f"\n{bead_list_summary_line(summary, use_color=use_color, implicit_limit=...)}")`.
- `json`: pass `summary` into `_render_list_json` and insert `"by_type"` and
  `"by_status"` after `"implied_status_closed"`, before `"results"`.
- Leave the `if not issues:` early returns exactly as they are.

### 3. Help and docs

- `src/sase/main/parser_bead_queries.py` — extend the `list` parser `description` to say
  that compact and full listings end with a summary line, that it describes only the
  printed rows, and that it names how many matching beads a limit hid. Add a line to the
  `epilog` next to the existing "Size column:" legend showing the shape of the summary.
  Keep options sorted and keep the existing short aliases.
- `docs/beads.md` — under `### sase bead list`, add a **Summary line** subsection after
  the compact-row anatomy: the grammar, the "describes only the printed rows" invariant,
  the fold rules, the two hidden-clause forms, the color mapping, and the note that
  `--format json` carries the same numbers as `by_type`/`by_status` instead of prose.
  Also fix the flag table's `--tier` row to `-r, --tier`, which already has that short
  alias in the parser but is missing it in the table.

### 4. Tests

New `tests/test_bead_summary_presentation.py`:

- counting, canonical ordering, zero-bucket omission in the rendered line while
  `by_type`/`by_status` keep every bucket;
- each fold branch — neither, type only, status only, both — and singular vs. plural;
- hidden clause present/absent, with and without the `(--limit 0 shows all)` hint;
- `use_color=False` produces no escapes; `use_color=True` produces exactly the
  `cli_style` of each glyph, the folded words' accents, and `ansi_sgr(dim=True)` for the
  hidden clause;
- the wording maps cover exactly `BEAD_TYPE_VALUES` and `BEAD_STATUS_VALUES`;
- an unknown type or status raises;
- `shown == 0` renders without crashing.

`tests/test_bead/test_cli_list.py`:

- compact output ends with a blank line then the summary, and the summary's counts match
  the rows actually printed under `--limit`;
- `--format full` gets the same line;
- `--format json` contains no prose line and gains `by_type`/`by_status` with all
  buckets, whose non-zero values equal the compact line's counts;
- the implicit-closed fallback path renders the `(--limit 0 shows all)` hint, and an
  explicit `--limit` does not;
- an empty listing still prints only `No issues found.`

`tests/test_bead/golden/cli/` — the golden corpus already exercises every branch of the
grammar, so update each file with the derived line (the fixtures pin the clock via the
`pinned_bead_clock` fixture, so these stay byte-stable):

| Golden                                | Appended summary                                   |
| ------------------------------------- | -------------------------------------------------- |
| `list.stdout`                         | `6 beads · ▸ 3  ↳ 3 · ○ 5  ◐ 1`                    |
| `list_full.stdout`                    | `6 beads · ▸ 3  ↳ 3 · ○ 5  ◐ 1`                    |
| `list_limit.stdout`                   | `2 open beads · ▸ 1  ↳ 1 · 4 hidden`               |
| `list_implicit_closed.stdout`         | `3 closed beads · ▸ 2  ↳ 1`                        |
| `list_implicit_closed_full.stdout`    | `3 closed beads · ▸ 2  ↳ 1`                        |
| `list_implicit_closed_limit.stdout`   | `1 closed plan · 2 hidden`                         |
| `list_implicit_closed_filters.stdout` | `1 closed plan`                                    |
| `list_implicit_closed_default.stdout` | `20 closed plans · 5 hidden (--limit 0 shows all)` |
| `list_closed_default.stdout`          | `20 closed plans · 5 hidden (--limit 0 shows all)` |
| `list_closed_unlimited.stdout`        | `25 closed plans`                                  |

Each is preceded by one blank line. `list_json.stdout`, `list_json_limit.stdout`,
`list_implicit_closed_json.stdout`, and `list_empty_json.stdout` gain the two new
envelope keys. `list_empty.stdout` is unchanged.

Treat the table as the expected result, not as bytes to paste blindly: if the
implementation disagrees with a cell, work out which one is wrong before editing either.

## Validation

1. `just install` first — this is an ephemeral workspace and its virtualenv may be
   stale.
2. `just check`.
3. `just check-full` before landing. This change touches a shared top-level presentation
   module and the bead golden corpus, which is exactly the case the two-speed rule
   reserves the full suite for.
4. Eyeball the real thing in the workspace, with and without color, and confirm the
   line's alignment and hierarchy read well against actual rows:
   ```bash
   sase bead list
   sase bead list --color never
   sase bead list --status closed
   sase bead list --status closed --limit 0
   sase bead list --type task --limit 3
   sase bead list --format full --limit 2
   sase bead list --format json --limit 2
   ```
   Confirm the `(--limit 0 shows all)` hint appears for `--status closed` and is absent
   for `--limit 3`.

## Definition of done

- Non-empty compact and full listings end with a blank line and one summary line
  matching the grammar above; empty listings are byte-identical to today.
- Every count in the line equals a count of the rows printed above it; the only number
  about unprinted beads is the `hidden` clause.
- `--format json` carries `by_type`/`by_status` derived from the same summary object as
  the prose, and prints no prose line.
- `--color never` and `NO_COLOR` produce a plain summary line.
- `just check-full` passes.
