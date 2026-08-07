---
tier: tale
title: Compact, non-wrapping source metadata in the Admin Center Logs pane
goal:
  Every row in the Logs tab source list renders on exactly two lines — a title line and
  a compact `size · age` subtitle — and wrapping is structurally impossible regardless
  of pane width, title length, or jump-hint prefixes.
proposed_by: bbugyi200.athena.uu
create_time: 2026-08-07 13:25:26
status: done
---

# Plan: Compact, non-wrapping source metadata in the Admin Center Logs pane

## Problem

In the "Logs" tab of the SASE Admin Center, every row in the left "Sources" pane wraps
onto a third line. The subtitle renders as:

```
● Launch & Fan-out Failures
   17.3 KB · 2026-08-07 13:14
EDT
```

The orphaned `EDT` appears on its own unindented line for all ten sources, which turns a
20-line list into a 30-line list and makes the pane look broken.

### Root cause

`source_label()` in `src/sase/ace/tui/modals/logs_pane_render.py:172-191` builds a
two-line `rich.text.Text`:

```python
text.append("\n   ")
meta_parts = [p for p in (format_size(size) if size else None, mtime) if p]
text.append(" · ".join(meta_parts) or source.description, style="dim")
```

with `format_size()` producing `17.3 KB` (up to 8 cells) and `format_mtime()` producing
`2026-08-07 13:14 EDT` (20 cells). The subtitle is therefore `3 + 8 + 3 + 20 = 34` cells
at its widest.

The available width is **30 cells**:

| Element                                                                   | Rule                             | Remaining |
| ------------------------------------------------------------------------- | -------------------------------- | --------- |
| `LogsPane #logs-source-panel`                                             | `width: 38` (`styles.tcss:5890`) | 38        |
| ...its `border: solid $secondary`                                         | −2                               | 36        |
| ...its `padding: 0 1`                                                     | −2                               | 34        |
| `LogsPane #log-source-list`                                               | `width: 100%`, `border: none`    | 34        |
| ...Textual `OptionList` default `padding: 0 1` (not overridden)           | −2                               | 32        |
| ...`scrollbar-gutter: stable` at the default `scrollbar-size-vertical: 2` | −2                               | **30**    |

34 > 30, so Rich word-wraps at the last space that fits — right before `EDT`.

The returned `Text` also leaves `no_wrap` unset, so wrapping is the _default_ behavior
rather than a bug that happens to trigger at one width. That is the part worth fixing
structurally.

## Design

Three ideas, in priority order.

### 1. The list is for glancing; the detail pane is for precision

The detail pane header already renders the full, timezone-qualified timestamp
(`logs_pane_render.py:145-148`):

```
~/.sase/logs/tui.log  ·  2026-08-07 13:14 EDT  ·  500 lines
```

So the sidebar does not need to repeat it. Repeating a 20-cell absolute timestamp on
every row is also nearly information-free: the year, month, and day are identical for
almost every source, so the only discriminating characters are the four in `13:14`. The
sidebar's actual job during triage is answering _"which log has recent activity, and how
much is in it?"_ — a relative age answers that with no mental arithmetic, and an
absolute timestamp does not.

Precision therefore lives one keystroke away in the detail pane; the list gets a
glanceable summary. Nothing is lost.

### 2. Two compact fields, each with a bounded width

**Size** — at most 4 cells, replacing `format_size()`:

| Bytes     | Today      | New    |
| --------- | ---------- | ------ |
| 842       | `842 B`    | `842B` |
| 13,107    | `12.8 KB`  | `13K`  |
| 619,315   | `604.8 KB` | `605K` |
| 1,782,579 | `1.7 MB`   | `1.7M` |
| 2,097,152 | `2.0 MB`   | `2.0M` |

**Age** — at most 8 cells, relative while relative is meaningful, absolute once it is
not. This mirrors the established idiom in
`src/sase/ace/tui/modals/plugins_browser_agent_clis_history.py:252-267`:

| Elapsed                                   | Rendered   |
| ----------------------------------------- | ---------- |
| < 60s (or a future mtime from clock skew) | `now`      |
| < 60m                                     | `2m ago`   |
| < 24h                                     | `3h ago`   |
| < 7d                                      | `2d ago`   |
| < 365d                                    | `Jun 17`   |
| otherwise                                 | `Jun 2025` |

The absolute tail matters for more than brevity: a dormant log reading `Jun 17` is more
useful than one reading `412d ago`, and — see _Determinism_ below — it is what keeps the
PNG snapshot fixture stable.

### 3. Wrapping becomes structurally impossible, not merely unlikely

Shorter strings alone are not a guarantee — a long title or a jump-hint prefix can still
overflow line 1. So `source_label()` will return
`Text(no_wrap=True, overflow="ellipsis")`.

This is already the codebase idiom for `OptionList` rows: see `format_saved_group_row()`
in `src/sase/ace/tui/modals/saved_agent_group_revival_rendering.py:96`. It also composes
correctly with jump mode — `apply_jump_hint_prefix()` (same file, lines 73-86)
explicitly rebuilds the decorated label with `no_wrap=label.no_wrap` and
`overflow=label.overflow`, so a hinted row inherits the guarantee.

Verified against the installed Rich/Textual (Textual 8.0.1): a 31-cell line at width 30
renders as `[a] ● Launch & Fan-out Failur…` on one line with `no_wrap` set, versus two
wrapped lines without it.

The result: content is short enough that ellipsis never triggers in normal use, and if
it ever does, the row degrades to a truncation instead of a wrap.

### Resulting layout

```
● Launch & Fan-out Failures
  17K  · 2m ago
● TUI Diagnostics
  1.7M · 2m ago
● TUI External Tools
  13K  · Jun 17
○ TUI Stalls
  empty
```

Two beauty details:

- The size field is **left-aligned and padded to 4 cells** (`f"{size:<4}"`), which puts
  every `·` in the same column. Left alignment (not right) keeps each row's ink starting
  at a consistent column, since the subtitle sits under a title.
- The subtitle indent drops from **3 spaces to 2** so it aligns exactly under the title
  text rather than one cell off from it. The `empty` subtitle moves too.

Widest possible subtitle: `2 + 4 + 1 + 1 + 1 + 8 = 17` cells against 30 available —
roughly 40% headroom, so the layout survives a narrower pane or a future field.

### Determinism

`tests/ace/tui/visual/conftest.py:72-91` freezes the clock for PNG snapshots by patching
`local_now` at an explicit per-module list of targets, because each module binds the
name at import time. The Logs fixture
(`tests/ace/tui/visual/_ace_config_center_logs_helpers.py:19`) pins every log mtime to a
fixed `2026-06-17 14:30 UTC`.

Against the frozen `_FIXED_VISUAL_NOW = 2026-07-06 12:00` that is ~19 days, which lands
in the `%b %d` band and renders `Jun 17` — deterministic. But that is determinism by
luck: it would silently become date-dependent the day someone moves the fixture mtime
inside the 7-day window. So the plan adds the new module to the patch list, making
determinism structural.

### Rejected alternatives

| Alternative                                                                         | Why not                                                                                                                                                                                                                                                  |
| ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Widen `#logs-source-panel` past 38                                                  | Steals width from the detail pane, which shows long tracebacks and file paths. Treats the symptom.                                                                                                                                                       |
| Collapse each row to a single line                                                  | `● Launch & Fan-out Failures` is already 27 of 30 cells; a same-line size and age would force truncating titles, which are the primary scanning key.                                                                                                     |
| Keep an absolute time, just shorter (`08-07 13:14`, or `ls`-style `13:14`/`Jun 17`) | Never goes stale, but makes the reader compute freshness against the current time — the exact question the pane exists to answer. The detail pane already covers the absolute case.                                                                      |
| Add a repaint timer so relative ages stay live                                      | Real but small problem (see _Known tradeoff_), and the cost is high: a recurring timer in a modal pane risks disturbing focus, selection, and jump-hint state, and TUI timers are governed by the `tui_perf` memory. Not worth it for a transient modal. |
| Color the age green when very fresh                                                 | The `●`/`○` glyph already carries state, and a second color axis in a 17-cell subtitle is noise. Keeping the subtitle uniformly `dim` is the restrained choice.                                                                                          |

### Known tradeoff

Relative ages are computed when the pane loads, and `LogsPane` has no refresh timer. A
load happens on mount, on `r`, and on every selection change (`j`/`k`), so an age only
drifts while the user sits idle on the tab — and drift is invisible at `h`/`d`
granularity. This is accepted deliberately; `r` is the documented refresh.

## Non-goals

- No change to the detail pane. `format_mtime()` keeps its `%Y-%m-%d %H:%M %Z` output
  and stays pinned by `tests/test_timezone_display_tui.py:87`.
- No change to `#logs-source-panel`'s width or to any other `styles.tcss` rule.
- No renaming of the log source titles in `src/sase/ace/tui/logs/sources.py`.
- No refresh timer.
- No consolidation of the three separate `stat()` calls per source (`exists()`,
  `_source_size()`, `format_mtime()`). Real but unrelated; leave it.

## Implementation

All source changes are in `src/sase/ace/tui/modals/logs_pane_render.py` unless noted.

### Step 1 — replace `format_size` with `format_size_compact`

Replace `format_size()` (lines 36-45). The loop must promote to the next unit whenever
the rounded value would need four digits, so the field is never wider than 4 cells:

```python
_SIZE_UNITS = ("B", "K", "M", "G", "T")


def format_size_compact(num_bytes: int) -> str:
    """Human-readable byte size in at most four cells (``17K``, ``1.7M``)."""
    size = float(max(0, num_bytes))
    index = 0
    while index + 1 < len(_SIZE_UNITS) and round(size) >= 1000:
        size /= 1024
        index += 1
    unit = _SIZE_UNITS[index]
    if index == 0:
        return f"{int(size)}{unit}"
    if size < 10:
        return f"{size:.1f}{unit}"
    return f"{round(size)}{unit}"
```

The `round(size) >= 1000` guard (rather than `size >= 1024`) is what makes 1023 bytes
render `1.0K` instead of `1023B`, and 1,048,000 bytes render `1.0M` instead of `1023K`.
Both are 4 cells.

Mirror the rename through `src/sase/ace/tui/modals/logs_pane.py`: the aliased import at
line 25 and the `__all__` entry at line 405 become `_format_size_compact`. Also update
`__all__` in `logs_pane_render.py` (line 198).

### Step 2 — add `format_relative_age`

New helper next to `format_mtime()`. Take the epoch directly (callers already have
`stat().st_mtime`) and accept an injectable `now` for tests, matching
`tasks_pane_render._relative_time`:

```python
def format_relative_age(epoch: float, *, now: datetime | None = None) -> str | None:
    """Compact freshness label for a log mtime (``2m ago``, ``Jun 17``)."""
```

Implementation notes:

- Return `str | None`, mirroring the existing `format_mtime()` contract: `None` when
  `parse_local(epoch)` cannot parse the value, so the caller drops the field through the
  same code path it already uses for a failed `stat()`.
- Normalize both sides to naive configured-timezone datetimes before subtracting,
  exactly as `plans_rendering._compact_relative_age` does: `parse_local(epoch)` returns
  an _aware_ datetime while `local_now()` returns a _naive_ one, so subtracting them
  directly raises `TypeError`. Use `local_now() - to_local(parse_local(epoch))`.
- Clamp a negative delta (a future mtime from clock skew) to zero so it renders `now`
  rather than a negative count.
- Render the two absolute bands through `format_local(epoch, "%b %d")` and
  `format_local(epoch, "%b %Y")` so the configured timezone is honored, consistent with
  `format_mtime()`.

Add both new names to `__all__`.

### Step 3 — rewrite `source_label`

Replace lines 172-191:

```python
def source_label(source: LogSource, *, now: datetime | None = None) -> Text:
    """Two-line row: ``● Title`` then a compact, non-wrapping metadata subtitle."""
    text = Text(no_wrap=True, overflow="ellipsis")
    ...
```

- Keep the existing `●`/`○` glyph and title styling unchanged.
- Change the subtitle indent from `"\n   "` to `"\n  "` on both the populated and the
  `empty` branch.
- Build the subtitle as `f"{size_text:<4} · {age_text}"`, keeping the current
  degradation behavior: when `_source_size()` or the mtime `stat()` returns `None`, drop
  that field and join what remains; when both are missing, fall back to
  `source.description` as today.
- Thread `now` through so it reaches `format_relative_age`.

### Step 4 — thread `now` from the loader

In `src/sase/ace/tui/modals/logs_pane.py:49`, resolve `now = local_now()` once before
the list comprehension and pass it to every `_source_label(source, now=now)` call, so
all ten rows in a single load share one reference instant rather than drifting across
the loop.

### Step 5 — keep PNG snapshots deterministic

Add `"sase.ace.tui.modals.logs_pane_render.local_now"` to the target tuple in
`_pin_agent_list_clock_for_visual_snapshots` (`tests/ace/tui/visual/conftest.py:80-90`).

## Testing

### New unit tests (`tests/ace/tui/test_logs_pane.py`)

1. **`format_size_compact` width bound** — parametrized over
   `0, 999, 1000, 1023, 13107, 619315, 1048000, 1782579, 2097152`, asserting both the
   exact string and `len(result) <= 4` for every case.
2. **`format_relative_age` bands** — parametrized with an explicit `now`, covering one
   value inside each of the six bands plus a future-dated mtime (clock skew → `now`) and
   each band boundary (59s/60s, 59m/60m, 23h/24h, 6d/7d, 364d/365d).
3. **`source_label` shape** — seed a source of known size and mtime, pass an explicit
   `now`, and assert the exact two-line `plain` output, that `label.no_wrap is True`,
   and that `label.overflow == "ellipsis"`.
4. **Empty source** — asserts the `empty` branch also carries the `no_wrap` and
   `overflow` flags and the 2-space indent.

### New regression tests (pilot harness, same file)

These are the tests that actually pin the reported bug. Use the existing
`_open_logs_pane` helper.

5. **Rows fit the real pane width.** Read the CSS-derived width from the live widget
   rather than hardcoding 30, so the test fails if either the content or the stylesheet
   drifts:

   ```python
   option_list = pane._option_list()
   width = option_list.scrollable_content_region.width
   for index in range(option_list.option_count):
       label = option_list.get_option_at_index(index).prompt
       for line in label.plain.splitlines():
           assert cell_len(line) <= width
   ```

6. **Rows render on exactly two lines.** Measure through Rich, which honors
   `no_wrap`/`overflow`, so the assertion covers the _rendered_ result rather than the
   source string:

   ```python
   assert len(label.wrap(Console(width=width), width)) == 2
   ```

   Run this assertion both in normal mode and after pressing `'` to enter jump mode. The
   jump-mode pass is the important one: a hinted `[a] ● Launch & Fan-out Failures` is 31
   cells against 30 and would wrap to three lines today, but ellipsizes to two with
   `no_wrap` set. Note that assertion 5 does _not_ hold in jump mode by design — hints
   may legitimately ellipsize — so keep the two assertions in separate tests.

### Existing tests

- `tests/test_timezone_display_tui.py` must pass untouched — it pins `format_mtime`,
  which this plan does not change. Treat any failure there as a sign that Step 1/2
  modified the wrong function.
- `tests/ace/tui/test_logs_pane.py:539`
  (`test_logs_tab_jump_mode_reuses_cached_source_labels`) monkeypatches
  `lp._source_label`; confirm the new keyword-only `now` parameter does not break its
  stub signature, and widen the stub to `(*_a, **_kw)` if it does.

### Visual snapshots

`config_center_logs_tab_120x40.png` and `config_center_logs_tab_toasts_120x40.png` both
change. Regenerate and **visually inspect** the result — this is a beauty-driven change,
so the diff is the deliverable, not a formality:

```bash
just test-visual -- --sase-update-visual-snapshots
just test-visual
```

Confirm in the regenerated PNG that every source row occupies exactly two lines, that
the `·` separators form a straight column, and that the ages read `Jun 17` (the fixed
fixture mtime), not a drifting day count.

### Gates

```bash
just install
just check
```

Run `just check-full` before landing. If Symvision flags the renamed `format_size` /
`_format_size` symbols or the re-export surface in `logs_pane.py:__all__`, consult the
`symvision` long-term memory through `/sase_memory_read` rather than adding a pragma by
reflex.

## Acceptance criteria

1. Every row in the Logs tab source list renders on exactly two lines at the default
   pane width, in both normal and jump mode.
2. The subtitle reads `<size ≤4 cells> · <age ≤8 cells>` with the `·` column aligned
   across rows, indented two spaces to sit under the title text.
3. `source_label()` returns a `Text` with `no_wrap=True` and `overflow="ellipsis"`, so
   wrapping cannot occur at any width.
4. The detail pane still shows the full `2026-08-07 13:14 EDT` timestamp.
5. Relative ages honor the configured timezone and are injectable via `now`, and the
   Logs PNG snapshots are stable across days.
6. `just check-full` and `just test-visual` pass, and the regenerated PNGs have been
   visually reviewed.
