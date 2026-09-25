---
tier: tale
title: Attach clan-node ?N to the waiting count
goal: "Agent clan nodes render the existing orange `?<N>` unknown-wait marker
  immediately after the `W<W>` waiting count inside the status chip (`[R1 W1?1 D3]`),
  without changing `?<N>` color or member-row wait tokens.

  "
size: small
proposed_by: bbugyi200.apollo.1q
create_time: 2026-09-25 11:15:57
status: wip
---

# Attach clan-node `?N` to the waiting count

## Problem

Clan rows already aggregate distinct unknown wait targets from direct `WAITING` members
and draw an orange `?N` marker. Today that marker is a **separate chip after the count
chip**:

```
[R1 W1 D3] ?1
```

The desired placement is **inside the count chip, immediately after the waiting metric,
with no extra space**:

```
[R1 W1?1 D3]
```

`?N` must keep the current unknown-wait style (`WAIT_UNKNOWN_GLYPH` /
`WAIT_UNKNOWN_GLYPH_STYLE`, currently `"?"` and `"bold #FFAF5F"`). This is a
presentation change only.

## Current code

Clan list rows are assembled in `format_agent_option`
(`src/sase/ace/tui/widgets/_agent_list_render_agent.py`). After
`format_agent_count_chip(...)` it does:

```python
if clan_chip:
    text.append(" ")
    text.append_text(clan_chip)
if clan_unknown_wait_count > 0:
    text.append(" ")
    text.append(
        f"{WAIT_UNKNOWN_GLYPH}{clan_unknown_wait_count}",
        style=WAIT_UNKNOWN_GLYPH_STYLE,
    )
```

The chip itself (`src/sase/ace/tui/agent_count_chip.py`) zero-suppresses metrics and
emits `W` then the waiting digits as two spans: chrome letter, waiting-purple count
(`bold #AF87FF`). It has no unknown-wait input today.

The count `clan_unknown_wait_dependency_count` is already computed and threaded:

- `agent_row_context` / `format_agent_row` in `_agent_list_build_rows.py`
- incremental patching in `_agent_list_build_patching.py`
- `agent_render_key` in `_agent_list_render_cache.py`

Do **not** change how `N` is counted, cached, or patched. Only where the glyph is
painted.

Covering tests live in `tests/ace/tui/widgets/test_agent_list_clan_unknown_wait.py`. The
main placement test currently asserts `"[R1 W1 D3]"` appears **before** a separate
`"?1"`. Docs copy the old form in `docs/ace.md` (`QUEUED [Q1 W3] ?2`).

## Design

### Placement

When a clan row has both a waiting metric (`waiting > 0`) and
`clan_unknown_wait_count > 0`:

- Render `W<W>?<N>` as one waiting-slot token inside the brackets.
- No space between the waiting digits and `?N`.
- Keep the existing space **between metrics**, so later tokens still read `W1?1 D3`, not
  `W1?1D3`.
- If `W` is the last metric, close the chip immediately: `[R1 W1?1]`.

Style spans, left to right on that token:

| Piece          | Style                                                                                |
| -------------- | ------------------------------------------------------------------------------------ |
| `W`            | existing chip letter style (neutral chrome, or `chrome_style` when callers pass one) |
| waiting digits | `AGENT_COUNT_CHIP_METRIC_STYLES["waiting"]` (`bold #AF87FF`)                         |
| `?<N>`         | `WAIT_UNKNOWN_GLYPH_STYLE` (`bold #FFAF5F`) — **unchanged**                          |

Do not restyle `?N` with waiting purple, chip chrome, or unread/background styles.

### How to splice

Extend `format_agent_count_chip` with an optional `unknown_wait: int = 0`.

When the waiting metric is emitted (`waiting > 0`) and `unknown_wait > 0`, append
`f"{WAIT_UNKNOWN_GLYPH}{unknown_wait}"` with `WAIT_UNKNOWN_GLYPH_STYLE` immediately
after the waiting digits, still inside the same chip `Text`. Import the glyph constants
from `sase.ace.tui.wait_status_presentation` so the style cannot drift.

Default `unknown_wait=0` leaves every other caller unchanged: tribe headers, panel
titles, member rosters, compact identity chips. Only the clan **list row** passes a
non-zero value.

In `format_agent_option`, pass `unknown_wait=clan_unknown_wait_count` when the visible
waiting count is greater than zero. Drop the separate after-chip append on that path so
the row never shows both `W1?1` and a trailing `?1`.

### Fallback when `W` is absent

`clan_unknown_wait_dependency_count` only counts direct `WAITING` members, and
`clan_member_counts` increments `waiting` for the Waiting bucket, so production rows
with `N > 0` also have `W > 0`.

Keep the existing after-chip ` ?N` (same glyph and style) when
`clan_unknown_wait_count > 0` and the waiting metric is zero-suppressed. That preserves
`test_clan_unknown_renders_even_when_chip_is_empty` (`?2` still visible on an empty
chip) without inventing `W0` or a synthetic `W?2` token.

Do **not** force-emit a `W` metric solely to host `?N`.

### Unchanged

- `clan_unknown_wait_dependency_count` and wait-status maps
- render-cache key shape (`clan_unknown_wait_count` is already a key field)
- row-context / patch wiring
- member-row `WAITING ?N` / `WAITING ?1 ?2` tokens (`format_wait_dependency_summary` /
  `append_agent_row_status`)
- help-modal `?N` glossary (`Unknown agent or bead`)
- CLAN sticky-header `Status:` line in `prompt_panel/_agent_display_clan_identity.py` —
  it does **not** show `?N` today; do not add it here
- panel titles, tribe chips, and other `format_agent_count_chip` callers

This is presentation-only Python TUI work in this repo. No `sase-core` change. No
event-loop, refresh, or cache-key work; the paint stays O(1) per clan row.

## Implementation

### 1. Chip helper — `src/sase/ace/tui/agent_count_chip.py`

- Add `unknown_wait: int = 0` to `format_agent_count_chip`.
- Treat non-positive values as absent (`<= 0` is a no-op), matching the other
  zero-suppressing counts.
- After appending the waiting count digits, if `unknown_wait > 0`, append
  `f"{WAIT_UNKNOWN_GLYPH}{unknown_wait}"` with `WAIT_UNKNOWN_GLYPH_STYLE`.
- If `waiting == 0`, ignore `unknown_wait` inside the helper (zero-suppression of `W`
  stays as-is). The clan-row caller owns the after-chip fallback.
- Docstring: the chip may read `[S1 R2 W3?1 D4]` when `unknown_wait` is set.

### 2. Clan list row — `src/sase/ace/tui/widgets/_agent_list_render_agent.py`

Replace the current “chip, then a space, then `?N`” block with:

1. Compute `visible_clan_counts` as today.
2. Call
   `format_agent_count_chip(..., unknown_wait=clan_unknown_wait_count if visible_clan_counts.waiting else 0)`.
3. Append the chip when non-empty, as today.
4. If `clan_unknown_wait_count > 0` and `visible_clan_counts.waiting == 0`, append the
   existing standalone ` ?N` with `WAIT_UNKNOWN_GLYPH_STYLE`.

`cached_format_agent_option` already forwards `clan_unknown_wait_count`; leave it.

### 3. Tests

**Chip unit tests** (`tests/ace/tui/test_agent_count_chip.py`):

- `waiting=1, unknown_wait=1, done=3` → `"[W1?1 D3]"` (and with running:
  `"[R1 W1?1 D3]"`).
- `?1` uses `WAIT_UNKNOWN_GLYPH_STYLE`; `W` stays chrome; waiting digits stay
  waiting-purple.
- Multi-digit: `waiting=12, unknown_wait=3` → `"[W12?3]"`.
- `unknown_wait=2` with `waiting=0` does **not** inject `W` or `?2` into the chip (empty
  chip stays empty; `[R1 D3]` stays `[R1 D3]`).
- `unknown_wait=0` is a no-op.

**Clan row tests** (`tests/ace/tui/widgets/test_agent_list_clan_unknown_wait.py`):

- Change `test_clan_row_renders_unknown_after_chip_with_style` so the fixture
  (`R1 W1 D3` plus one unknown waiter) asserts `"[R1 W1?1 D3]"` in `left.plain`, `"?1"`
  is **not** a separate token after `]`, and `WAIT_UNKNOWN_GLYPH_STYLE` still covers
  `"?1"`.
- Keep `test_clan_row_without_unknowns_renders_no_marker`.
- Keep `test_clan_unknown_renders_even_when_chip_is_empty` (`"?2"` still present and
  styled when there is no `W`).
- `test_row_context_carries_clan_unknown_and_format_uses_it` and the bead-warmup render
  assertion can keep `"?1" in left.plain`; add that the chip form is `W1?1` rather than
  `W1 D…] ?1` when `W` is present.
- Reuse the existing `_styles_covering` helper; do not require the waiting-purple span
  to cover `?`.

### 4. Docs — `docs/ace.md`

Update the clan-row paragraph (currently around the Agents-tab clan chrome section)
from:

> adds an orange `?N` chip after its count chip, as in `QUEUED [Q1 W3] ?2`

to the in-chip form, for example:

> attaches an orange `?N` immediately after the waiting count inside the chip, as in
> `QUEUED [Q1 W3?2]`

Keep the meaning of `N` (distinct unknown agents, clan members, and beads across direct
`WAITING` members; shared targets count once). Do **not** rewrite the member-row
`WAITING ?1 ?2` examples later in the same file; those are a different token sequence.
`tests/ace/tui/test_agent_wait_dependency_status_counts.py` greps that later prose.

## Verification

- Run the focused tests above, then `sase tool run check` (or `just check` if the tool
  catalog is unavailable).
- Do **not** run `just check-full`.
- Current clan PNG fixtures
  (`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`) do not include an
  unknown-wait clan row. Do not add a new visual snapshot for this placement change. If
  a targeted `just fix-tui-screenshots` run reports a golden that still shows `] ?N`
  after a clan `W` count, update that golden and inspect the report; otherwise leave
  goldens alone.

## Out of scope

- Showing `?N` on the CLAN sticky-header `Status:` line
- Changing unknown-wait counting, dedup, or bead-warmup patching
- Restyling member `WAITING ?N` tokens
- sase-core / wire-format changes
