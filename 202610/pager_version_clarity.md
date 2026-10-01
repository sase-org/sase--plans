---
tier: epic
title: 'Pager version clarity: always know which memory version you are reading'
goal: 'Whenever the SASE pager shows a memory or instruction file, one glance tells
  you exactly which version you are reading: now, uncommitted, a past version (with
  its number, its date, and where it sits in the file''s life), a deletion, or a comparison
  between two named versions. Every surface (subject line, time band, body frame,
  footer, timeline picker, trail) tells the same story from one model. History attaches
  to every memory section no matter how it was opened, and the design is legible in
  dark and light themes.

  '
phases:
- id: identity
  title: One version identity model, honest numbering, and reliable attachment
  depends_on: []
  size: medium
  description: 'identity: add the pure VersionMoment model and step function that
    every surface reads; number versions as absolute vK of N; treat a clean now as
    the newest version so the first `(` never shows a byte-identical copy; fix bare-selector
    and stale-entry-point attachment failures.'
- id: badge
  title: State pill, past frame, destination footer, and versioned trail
  depends_on:
  - identity
  size: medium
  description: 'badge: route every history colour through a theme-aware style set;
    replace the subject chip with a four-state pill; draw a past-accent gutter rail
    through pinned bodies; make the footer name where each time key goes; suffix trail
    crumbs with their version; add a pill legend to help.'
- id: band
  title: Time band with playhead scrubber, explicit diff endpoints, and tombstone
    chrome
  depends_on:
  - badge
  size: medium
  description: 'band: rebuild the time band around a playhead scrubber with labelled
    ends, absolute time, and the commit subject; show the compared range and both
    endpoints in the diff view; tint the band in the past; move the deletion notice
    out of the body and into chrome.'
- id: picker
  title: Timeline picker as an aligned table with open, now, and cursor markers
  depends_on:
  - badge
  size: medium
  description: 'picker: turn picker rows into structured, column-aligned, never-wrapping
    rows; always list now; mark the open version separately from the cursor; preview
    what Enter and = will do; make comparisons always read older to newer.'
- id: polish
  title: Documentation, live review, and end-to-end verification
  depends_on:
  - band
  - picker
  size: small
  description: 'polish: update the memory history and pager docs with the new anatomy,
    review every state live on real memory files and in the goldens, verify performance
    budgets, and record follow-ups.'
proposed_by: bbugyi200.athena.0v2
create_time: 2026-10-01 15:29:15
status: wip
bead_id: sase-1ee
---

- **PROMPT:** [prompts/202610/pager_version_clarity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/pager_version_clarity.md)
- **BEAD:** [sase-1ee](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ee/README.md)

# Plan: Pager version clarity

## 1. Problem

The memory-history epic (`sase-1dr`) gave the pager a time axis. On real data, though,
it is often unclear which version you are looking at. Evidence from a live run on
`sase/memory/gotchas.md` (25 versions, 4 of them hidden) and from the current PNG
goldens:

1. **The numbers are wrong.**
   - The subject chip divides the absolute ordinal by the _visible_ count, so it shows
     `⟲ PAST v25/21` and `⟲ PAST v24/21`.
   - At a clean now the chip reads `v21 versions`.
   - The timeline picker header says `25 versions · 4 hidden`.
   - So three surfaces show three different numbers for the same file.
2. **The first `(` goes nowhere.** A clean worktree is byte-identical to the newest
   commit (worktree OID = HEAD OID = v25's blob). Yet `(` from now lands on v25,
   labelled `PAST`, with the same body as now.
3. **The past is only a small label.** The violet `⟲ PAST v2/3` text is the only
   persistent cue.
   - `_history_chrome_state` always sets `age: ""`, so the date never reaches the chip.
   - The past colour is hard-coded to `#9d7cd8`, which is about 2.5:1 on the light
     theme. `history_palette_from_theme` is computed but its result is discarded.
   - Scroll past the band and nothing on screen says "past".
4. **You cannot tell where you are in time.**
   - The sparkline marks the current version with one reverse-video cell, often among 25
     or more.
   - `→ now` floats at the far right edge, detached from the sparkline.
   - Nothing says how many versions are newer.
5. **Diff endpoints are half-stated.** The chip says `diff vs v1` but never names the
   target, and nothing ties the endpoints to the red and green in the body. A picker
   compare can even run newer → older.
6. **The picker is hard to read.**
   - Rows are pre-joined strings, so long provenance wraps onto a second line and
     columns do not line up (`v9` vs `v25`).
   - The open version is distinguished by colour alone.
   - There is no "now" row when the worktree is clean.
7. **The footer and trail are vague.**
   - `( ) version` does not say where the keys go.
   - The footer shows `E edits now` and `E edit` at the same time.
   - Trail crumbs never say which version they will restore.
8. **Tombstones pollute the body.**
   - The provider injects `✖ deleted by <author> at <raw epoch> — last content shown` as
     body line 1.
   - This shifts every line number, leaks into search and copy, and shows a raw epoch.
9. **History sometimes does not attach at all.**
   - `sase memory history gotchas.md -f pager`, `… decisions -f pager`, and any other
     bare note name open with no time axis ("No history for this section").
     `_recognizes_section` sniffs for `sase/memory/` in the section's text and finds
     nothing.
   - A dev checkout whose editable-install metadata predates the `sase_pager_history`
     entry point gets no provider at all.

## 2. Design decisions

| #   | Decision                                                                                                                                                                                                                                                                                                                                                                                                                       | Why                                                                                                                                                                                                   |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | **One identity model.** A pure `VersionMoment` (§4) is the only place that computes the moment kind, version numbers, dates, newer counts, step destinations, and diff endpoints. The pill, band, rail, footer, picker, trail, and help all render from it.                                                                                                                                                                    | Today the numbers disagree because each surface counts on its own. If the footer reads its destination from the same function that runs the key, it can never be wrong.                               |
| D2  | **Ordinals are absolute, and "of N" means the newest committed ordinal**, counting hidden versions. Hidden versions are skipped while stepping but never renumbered.                                                                                                                                                                                                                                                           | `vK` is a stable identifier shared with `sase memory history -A vK`, the picker labels, and the feed. `v24 of 25` is true, while `v24/21` is impossible.                                              |
| D3  | **A clean now _is_ the newest version (`now ≡ v25`)** when the timeline's `worktree_oid`, `head_oid`, and the newest row's `blob_oid` are all present and equal. `(` from such a now goes straight to the version before it. A pin to the newest ordinal on such a subject (CLI `-A`, a feed link, or picker ⏎) is canonicalized to now. If any OID is missing or differs, fall back to treating now and vN as separate stops. | Stepping onto a byte-identical copy labelled PAST is the most confusing thing in the current UI. The conservative fallback means the canonicalization can never hide a real difference.               |
| D4  | **One state pill with four states, and loudness that follows risk.** `● NOW` is a quiet neutral pill. `◌ NOW · uncommitted` is amber. `⟲ PAST · vK of N` is a solid violet pill. `✖ DELETED` is red. The pill sits in the same place with the same shape in every state, and it is never cropped.                                                                                                                              | The user learns one component instead of a set of differently coloured words. Being in the past is the risky state, because you might mistake it for today, so it gets the strongest treatment.       |
| D5  | **The past is framed, not just labelled.** While a section is pinned in the past, its gutter rail (`│`) is drawn in the past accent the full height of the body, and the time band gets a faint past-accent tint. A deleted subject's rail uses the muted deleted tone.                                                                                                                                                        | A cue in peripheral vision survives scrolling, folded chrome, and short terminals. Changing only the rail keeps the line numbers and text untouched.                                                  |
| D6  | **The footer names destinations**: `( v21 · ) v25 · } now`. A verb appears only when its key would do something.                                                                                                                                                                                                                                                                                                               | It answers "where will this key take me?" before you press it. This also follows the pager's availability-driven footer convention.                                                                   |
| D7  | **Diffs always read forward in time**, from an older base to a newer target, and both endpoints are spelled out as `Δ v23 → v24`. The base is shown in the delete tone and the target in the insert tone, matching the body's struck-red and green words.                                                                                                                                                                      | Shown together, a named direction and colour-matched endpoints remove all doubt about what the red and green mean.                                                                                    |
| D8  | **Every history colour comes from the theme-aware palette**: no hard-coded `#9d7cd8`, `yellow`, `magenta`, `green`, or `red` in history chrome. The "changed line" gutter mark becomes blue (the familiar VCS "modified" colour), so it is distinct from the violet past rail and the red removal tick.                                                                                                                        | This fixes the light-theme contrast. Today's `magenta` "changed" mark renders as pink-red and reads as "removed".                                                                                     |
| D9  | **History attaches deterministically.** A section built by `build_history_document` carries the subject's canonical repo-relative path and is owned by the memory provider whatever the selector's spelling. sase's own memory provider is always discoverable, even when entry-point metadata is stale.                                                                                                                       | Short selectors are the documented, recommended form. A clarity redesign is worthless if the axis silently does not appear.                                                                           |
| D10 | **The deletion notice moves from the body into chrome** (the pill and a band tombstone row). The body shows exactly the last content, so line numbers match the file.                                                                                                                                                                                                                                                          | Line numbers, search, and copy must describe the file, not a banner.                                                                                                                                  |
| D11 | **No sase-core change.** Every input (ordinals, hidden flags, blob OIDs, the worktree and HEAD OIDs, provenance subject lines, committer times) is already on the timeline wire. `VersionMoment` is presentation and navigation glue in pager core and imports no memory modules.                                                                                                                                              | The Rust-boundary litmus test: no other frontend consumes this today. If a web or editor frontend later needs "now ≡ vN", promote it into the core timeline wire. That is a recorded follow-up (§10). |
| D12 | **No feature flag.** Each phase lands a complete, strictly clearer UI, and no phase exposes a half-built surface.                                                                                                                                                                                                                                                                                                              | Per the `sase_flags` memory, an epic gets a beta flag only when a landed phase would otherwise expose part of an unfinished feature.                                                                  |

## 3. Vocabulary

There are five moment kinds. Each is defined once and used everywhere.

| Kind        | Meaning                                                                 | Pill (full form)      | Context after the pill   |
| ----------- | ----------------------------------------------------------------------- | --------------------- | ------------------------ |
| `now`       | The live file, clean. When D3 holds it is identical to vN (`now ≡ vN`). | `● NOW · v25`         | `latest · 1mo ago` (dim) |
| `now_dirty` | The live file with uncommitted or staged edits on top of vN.            | `◌ NOW · uncommitted` | `on top of v25` (amber)  |
| `past`      | A pinned committed version.                                             | `⟲ PAST · v24 of 25`  | `1mo ago` (past accent)  |
| `deleted`   | The subject's newest version is a deletion (tombstone).                 | `✖ DELETED · v12`     | `2mo ago` (deleted tone) |
| `loading`   | The timeline is not indexed yet.                                        | (no pill)             | `indexing…` (dim)        |

More rules:

- **Diff view.** In the diff view the context gains `Δ v23 → v24`. The base is in the
  delete tone, `→` is dim, and the target is in the insert tone. For a dirty now it
  reads `Δ v25 → now`. For a clean now (which shows the latest change) it reads
  `Δ v24 → v25`.
- **Pill widths.** The pill shortens through fixed steps:
  - `⟲ PAST · v24 of 25` → `⟲ PAST · v24/25` → `⟲ v24/25` → `⟲ v24`
  - `● NOW · v25` → `● NOW`
  - `◌ NOW · uncommitted` → `◌ NOW`
  - `✖ DELETED · v12` → `✖ DELETED` → `✖ v12`
- **Context when the band is folded.** When the time band is folded away (12 rows or
  fewer), the past and deleted context gains the short date (`Aug 24 · 1mo ago`),
  because the band's absolute date is not on screen.
- **Honest states.** Untracked, ignored, no VCS, shallow, template, and unavailable keep
  their existing honest chips and band rows, now styled through the palette. They never
  show a pill.

## 4. The identity model (`identity` phase)

`src/sase/pager/history/moment.py` is pure, Textual-free, and imports nothing from
`sase.memory`.

```python
@dataclass(frozen=True, slots=True)
class VersionMoment:
    kind: Literal["loading", "now", "now_dirty", "past", "deleted"]
    ordinal: int               # displayed version; newest when kind == "now" and now ≡ vN; 0 when dirty
    newest: int                # N: newest committed ordinal, hidden versions included
    now_matches_newest: bool   # D3
    view: Literal["read", "diff"]
    diff: tuple[int, int] | None        # (base, target); 0 = now; base is always older
    committed_time: int | None          # the shown version's committer time
    commit: str | None
    commit_subject: str | None          # provenance.subject
    path_at_version: str | None         # set only when it differs from the current path
    newer_count: int                    # visible committed versions newer than the shown one
    older: int | None                   # step destinations, None means a boundary; 0 = now
    newer: int | None
    first: int | None
    to_now: int | None
```

- **Builders.**
  - `build_moment(*, rows, meta, visible_ordinals, pin, status)` takes the
    `SectionTimeState` data the screen already holds: timeline rows, the `timeline_meta`
    with `worktree_oid`/`head_oid`, `visible_ordinals`, the current pin, and the status.
    The memory provider already computes the hidden-class rules, so use its
    `visible_ordinals` rather than re-deriving them.
  - `step_target(moment, intent)` returns the destination for `older`, `newer`, `first`,
    and `now`, or `None` at a boundary.
  - `canonical_ordinal(ordinal, moment)` returns 0 when D3 makes vN equal to now.
- **Step rules.**
  - `older` from a clean `now ≡ vN` returns the newest visible ordinal below N.
  - `newer` from the last visible version below N returns 0 (now) when
    `now_matches_newest` holds, and otherwise returns N.
  - With a dirty now, `older` from now returns the newest visible version, which is
    HEAD.
  - For a tombstone, `to_now` is the deletion ordinal.
- **Caching.** Build the moment once per (history generation, pin, status, timeline
  identity) and keep it on `SectionTimeState`. The subject line repaints on scroll, so
  the moment must never be rebuilt per paint.
- **Fail open.** If building raises, the surfaces render as they would with no history.

The `identity` phase also wires the model in and fixes attachment:

1. **Stepping.**
   - `_screen_history._apply_step_intent` must use `step_target`. Delete the old ad-hoc
     candidate logic.
   - `_load_and_swap_version` and the arrival-pin path in `_index_one_section`
     canonicalize through `canonical_ordinal`. A pin to vN on a D3 subject swaps in the
     live section with the existing refresh-style swap, which pushes no trail entry, and
     it keeps the pin's `view` and `compare_base`.
   - The picker's open result goes through the same canonicalization.
2. **Boundary notices name the boundary**, for example "Already at v1, the oldest
   version", "Already at now (≡ v25)", or "now ≡ v1 is the only version".
3. **Interim chip fix.**
   - `_history_chrome_state` takes `ordinal`, `total` (now N), and `age` from the
     moment, so the existing chip reads `⟲ PAST v24/25 · 1mo` and `25 versions`. Remove
     the `v{total} versions` typo.
   - The `badge` phase replaces this chip entirely, but `identity` must land coherent on
     its own.
4. **Deterministic attachment (D9).**
   - `build_history_document` (`src/sase/memory/history/pager_provider.py`) stamps the
     resolved repo-relative path into `subject_ref` and `owner.source_reference`. Take
     the path from the version response's subject `paths` or its version `path`.
   - `MemoryHistoryProvider.recognizes` also accepts any section whose `version_pin`
     carries a memory-history subject id (`note:`, `web:`, `strand:`, `instructions:`).
   - Keep path sniffing for plain file sections.
5. **Provider discovery.** In `src/sase/pager/history/provider.py`,
   `_discover_entry_point_factories` falls back to the built-in factory
   `sase.memory.history.pager_provider:memory_history_provider_factory`, loaded lazily
   by string and deduplicated against entry points, when entry-point metadata does not
   list it. Pager core still never imports memory modules at import time.
6. **Tests.**
   - `tests/pager/test_history_models.py`, or a new
     `tests/pager/test_history_moment.py`:
     - every kind
     - D3 true, false, and missing-OID cases
     - hidden newest versions
     - a single-version subject
     - a tombstone
     - dirty with staged edits
     - step tables for every intent from every position
     - diff endpoints, including normalization of non-parent bases
   - `tests/pager/test_app_history.py`:
     - the first `(` from a clean now lands on vN−1
     - `)` from vN−1 returns to now
     - an `-A vN` arrival on a clean subject opens now
     - a dirty `(` lands on vN
   - `tests/memory/test_history_pager_provider.py`:
     - `gotchas.md`, `decisions`, and `glossary:stitch` selectors are all recognized
     - the built-in factory is found when entry points are empty

## 5. State pill, frame, footer, and trail (`badge` phase)

### 5.1 Theme-aware styles

- **Palette roles.** Extend `history_palette_from_theme`
  (`src/sase/pager/syntax_theme.py`) with these roles:
  - `modified`: blue, around `#58A6FF`
  - `now_pill_bg` and `now_pill_fg`: the neutral pill, a blend of about 18% foreground
    over the background
  - `past_pill_bg` and `past_pill_fg`
  - `uncommitted_pill_bg` and `uncommitted_pill_fg`
  - `deleted_pill_bg` and `deleted_pill_fg`
  - `rail_past` and `rail_deleted`
  - `band_past_tint`: about 14% past accent blended over the surface
- **Pill text colour.** Each pill's text is black or white, whichever passes 4.5:1
  against its background. If neither does, nudge the background until one does.
- **`HistoryStyles`.** Add a frozen dataclass in pager core,
  `src/sase/pager/history/styles.py`, resolved once per host theme and invalidated on
  theme change. It is the only source of history colours for the chip, band, gutter,
  diff body (`INSERT_STYLE`/`DELETE_STYLE` in `pager/history/diff.py`), picker, and
  trail suffix.
- **Fixes.**
  - Fix the dead `_history_past_style` in `_screen_chrome.py`.
  - Replace `PAST_STYLE`, `UNCOMMITTED_STYLE`, and the other hard-coded constants with
    style roles.
- **Tests**, covering every built-in theme, dark and light:
  - text roles ≥ 4.5:1 and marks ≥ 3.0:1
  - pill text against pill background ≥ 4.5:1
  - band text against `band_past_tint` ≥ 4.5:1
  - past hue at least 60° from the warning hue (existing rule)
  - `modified` hue at least 30° from the past hue

### 5.2 The pill

- **Rendering.** `_chrome.subject_line` renders `history_badge(moment, styles, budget)`
  in place of `_history_chip`, followed by the §3 context.
- **Layout.** The pill is a solid capsule: one padding cell on each side, bold text, the
  pill background, and the glyph plus a word in caps.
- **Width shedding.** As width shrinks, give things up in this order:
  1. the syntax hint
  2. the `⌘` character count
  3. the context, with the `Δ` segment kept longest in the diff view
  4. the pill's shorter forms (§3)
  5. the title, middle-truncated so the basename survives

  The pill itself is never cropped.

### 5.3 Past frame

- **Rail styles.** `apply_gutter` (`src/sase/pager/_gutter.py`) gains
  `rail_style: str | None`, applied to the plain `│` separator of every row of the
  section. `compose_body` (`src/sase/pager/_layout.py`) gains
  `rail_styles: Mapping[int, str]`, filled by `_compose_body_at_width` from each
  section's moment:
  - `past` gets `rail_past`
  - `deleted` gets `rail_deleted`
  - every other kind gets none
- **Precedence.** Goto emphasis wins over change marks, and change marks win over the
  rail.
- **Change-mark colours.** Added `▌` is `insert`, changed `▌` is `modified` (blue), and
  the removal `╴` is `delete`.

### 5.4 Footer

- **Time verbs.** `footer_legend` receives an ordered list of time verbs built from the
  moment, instead of the `history_available`/`history_pinned`/`history_diff_view`
  booleans:
  - `( v21` is shown when `older` exists.
  - `) v25` or `) now` is shown when `newer` exists.
  - `} now` is shown only when `to_now` differs from `newer`. A tombstone reads
    `} deleted`.
  - `= diff` or `= read`.
  - `@ timeline`.
- **Single `E` verb.** When pinned, it reads `E edit now`; otherwise it reads `E edit`.
  The footer must never show two `E` verbs.

### 5.5 Trail crumbs

- **Suffix.** `PagerTrailDisplayEntry` (`_trail_chrome_model.py`) gains
  `version_suffix`. It is built from `PagerTrailEntry.version_pins` (looked up by the
  entry's section identity) for back and forward entries, and from the live moment for
  the current entry:
  - `@v24` in the read view
  - `@v23→v24` in the diff view
  - `@✖` for a tombstone
  - nothing for now
- **Rendering.** The suffix is painted in the past accent after the crumb label, takes
  part in `signature`, and is shed before the label is truncated.

### 5.6 Help

- **Legend.** The "Time" group in `_trail_chrome_help.py` gains a four-line legend: the
  four pills with one-line meanings, plus `Δ vA → vB`.
- **Wording.** `( / )` reads "Older / newer version (the footer shows where)".

### 5.7 Mockups (120 columns, the real `gotchas.md` timeline)

Clean now. The band is still the old style until the `band` phase lands. `[● NOW · v25]`
is a neutral pill.

```text
 ▤ sase/memory/gotchas.md  [● NOW · v25]  latest · 1mo ago                                  100% · ⌘ 272c · md
```

Past read view. The pill is solid violet, and the `│` rail is violet for the whole body.
The changed line's `▌` is blue.

```text
 ▤ sase/memory/gotchas.md  [⟲ PAST · v24 of 25]  1mo ago                                   100% · ⌘ 1.7Kc · md
   1│ ---
   2▌ type: core
   3│ parent: AGENTS.md
 ( v21 · ) now · = diff · @ timeline · E edit now · 0-9a-z follow · y copy · / search · ? keys · q close
```

### 5.8 Tests and goldens

- `tests/pager/test_chrome.py`:
  - pill text and styles for every kind and view
  - each step of the shedding order at widths from 30 to 200
  - the pill is never cropped
- `tests/pager/test_gutter.py`: rail styles and precedence.
- Footer tests: destinations in every state, and a single `E` verb.
- `tests/pager/test_trail_chrome.py`: suffixes and signatures.
- PNG goldens (`tests/pager/visual/`):
  - Regenerate the history, time-band, and picker goldens. The pill shows in all of
    them.
  - Add `history_past-frame` at both sizes and in both themes. Its body must be long
    enough that the band scrolls out of mind and the rail is the cue.
- Run `just fix-tui-screenshots -- tests/pager/visual` and inspect every changed PNG in
  the report, as the `lint_and_test` memory requires.

## 6. The time band (`band` phase)

### 6.1 Anatomy

The band keeps its place: `#pager-time`, between the trail band and the chrome rule.

- **In the past, two rows.**
  - **Row 1, the timeline row**, answers where and when.
  - **Row 2, the meaning row**, answers what and who. It sits directly above the body it
    describes.
- **With one row** (from `chrome_row_budget`), only the meaning row is kept, because the
  pill already carries the identity.
- **At now, one row:** the scrubber plus `last changed …`.

```text
 ▤ sase/memory/gotchas.md  [⟲ PAST · v24 of 25]  1mo ago                                   100% · ⌘ 1.7Kc · md
 v1 ▇▂▅··▅▅▆▆▅▆▅▆█▅▃▅▆▂▅▅··▂▇ ● now   Mon Aug 24 2026 12:41 · feat(memory): rename memory tiers to core…  1 newer
 ▣ type: short → core · +1w −1w                       [0]sase-sq.1 · [1]bbugyi200.athena.sase-sq.1 · [2]c9ca0db
   1│ ---
```

### 6.2 The playhead scrubber

The new function is `render_scrubber(...)` in `src/sase/pager/_time_band_vocab.py`. Keep
`render_sparkline` unchanged, because the Memory panel History row uses it.

- **Endpoints.**
  - The left endpoint is dim `v1`, or the oldest ordinal.
  - The right endpoint is `● now`. It is `◌ now` in amber when dirty, and `✖ deleted`
    for a tombstone.
- **Slots.**
  - There is one slot per committed version, oldest to newest.
  - A slot is 2 cells (bar plus gap) when there are 12 versions or fewer, otherwise 1
    cell.
  - Versions are bucketed when there are more than the track width allows. The track is
    at most 60 cells.
  - Bar height is the existing log-scaled volume.
  - A hidden version is a dim `·`, and a deleted version is a `✖` in the deleted tone.
- **Progress colouring.** It reads like a video scrubber's watched portion:
  - **Past:**
    - slots before the shown version are in the past accent
    - the shown version is the **playhead**: reverse video, bold, past accent
    - slots after it are dim
    - the `now` endpoint is dim
  - **Now:**
    - every slot is in the normal foreground
    - when D3 holds, the newest slot and the `● now` endpoint together form the
      playhead, in the neutral pill style
    - when D3 cannot be confirmed, the `● now` endpoint alone is the playhead
  - **Dirty now:** the `◌ now` endpoint is the playhead, in amber.
  - **Diff view:**
    - slots outside the compared range are dim
    - the base slot is bold, in the delete tone
    - the slots between are in the past accent
    - the target is the playhead
- **Class tints.** Promotion and demotion tints give way to progress colouring, because
  the meaning row carries the class.

### 6.3 The text of the timeline row

- **Right of the scrubber**, depending on the moment:
  - Past: the full absolute date and time, then `·` and the commit subject.
  - Diff: `comparing v23 Aug 24 → v24 Aug 24 2026 12:41`.
  - Dirty now in diff: `comparing v25 Tue Aug 25 2026 → uncommitted edits`.
- **Right-aligned:** `N newer` (with `+ uncommitted` when dirty), then
  `⇡N on origin/<default>`.
- **Shedding order:**
  1. the commit subject
  2. `⇡N`
  3. `N newer`
  4. the weekday and time (the date stays)
  5. the scrubber shrinks to at least 8 cells
  6. the endpoint labels

### 6.4 Meaning row, now strip, and tombstone

- **Meaning row.** Its content and shedding stay as today. Add `· as <path>` when
  `path_at_version` differs from today's path, which shows renames. Style it through
  `HistoryStyles`.
- **Now strip.** It shows the scrubber, then
  `last changed Tue Aug 25 2026 · bbugyi200.athena.0dj`. When dirty it adds
  `◌ edits not durable until committed`.
- **Tombstone row.** It replaces the meaning row for `deleted`:
  `✖ deleted Mon Jul 13 2026 14:03 by <agent or author> · showing last content (v11)`.
- **Tombstone body.** `_derived_section` in `pager_provider.py` stops injecting the
  deletion line, so the body is exactly the last content.

### 6.5 Tint, model, and tests

- **Band tint.** In the past, `#pager-time` uses `band_past_tint` as its background, set
  from `HistoryStyles` in `_update_time_band`. In every other state it uses `$surface`.
- **Model.**
  - `build_time_band_data` takes the `VersionMoment` and stops recomputing current,
    newest, and dirty on its own.
  - The `_update_time_band` signature gains the view, the diff endpoints, the newer
    count, the tint, and the tombstone, so repaints are exact.
- **Tests.** `tests/pager/test_time_band.py`:
  - scrubber cells and styles for every kind, for 1, 2, 12, 13, 25, and 260 versions,
    with hidden and deleted slots, and for diff ranges (adjacent and non-adjacent)
  - row order and the one-row fallback
  - every shedding step at widths from 40 to 200
  - the tombstone row
  - `· as <path>`
  - the tint contrast
  - `tests/memory/test_history_pager_provider.py`: the tombstone body has no injected
    line
- **Goldens.** Regenerate `timeband_*` and `history_*`. Add:
  - `timeband_diff-range`: a non-adjacent picker compare
  - `timeband_many-versions`: 260 versions, bucketed, with the playhead mid-track
  - `history_tombstone`, updated to show chrome instead of the body banner

  Inspect them in both themes at both sizes.

## 7. The timeline picker (`picker` phase)

### 7.1 Rows

`build_picker_rows` (`src/sase/memory/history/timeline_picker.py`) returns structured
cells instead of a pre-joined `display` string. Keep `haystack` and `hidden`.

- **Cells:** `label`, `date`, `age`, `glyph`, `change`, `words`, `by` (the bead, else
  the agent), `sha`, plus the flags `pseudo`, `is_open`, `is_now_alias`, and `hidden`.
- **Dates** include the year when it differs from the current year.
- **The now row is always first.**
  - Clean (D3): `now  ≡ v25 · the live file`.
  - Dirty: `now  ◌ uncommitted · not durable until committed`.
  - Staged: a `stg` row, shown when core reports one, as today.
- **The vN row on a D3 subject** is tagged `≡ now`.

### 7.2 Layout

`TimelinePickerScreen` (`src/sase/pager/_timeline_picker.py`) lays rows out in fixed
columns fitted to the modal's width:

- **Columns:**
  - a 2-cell marker
  - the label, right-aligned to the widest label
  - the date
  - the age
  - the glyph
  - the change and word delta, which flexes and is truncated with `…`
  - by, at most 24 cells, truncated with `…`
  - the SHA, 7 cells
- **Shedding as width shrinks:** the SHA, then by, then the age.
- **No wrapping.** A row is always exactly one line.
- **Markers.**
  - The open version's row has a `●` marker, and its label is drawn in the moment
    colour. When now ≡ vN, the now row is the open one.
  - The cursor row has a `▸` marker and is drawn in reverse video.
  - When the cursor is on the open row, both markers show.
  - Hidden rows, once revealed with `.`, are dim.
- **Header:** `gotchas.md · 25 versions · 4 hidden`, with the open version's pill on the
  right, before `/ filter`. Both come from the moment, so the numbers match the subject
  line.
- **Live footer** that previews the cursor row's actions:
  - `⏎ open v21 · = compare v21 → v24 · . hidden · / filter · esc close`
  - On the open row: `● open v24 · . hidden · / filter · esc close`
- **Compare normalization (D7).**
  - The comparison is always older → newer.
  - If the cursor row is newer than the open version, the cursor's version becomes the
    shown target and the previously open version becomes the base. This pushes a trail
    entry, as a picker jump does today.
  - The footer preview always shows the exact `Δ` that will result.

```text
╭─ gotchas.md · 25 versions · 4 hidden ──────────────────────────── [⟲ PAST · v24] ─ / filter ─╮
│      now  ≡ v25 · the live file                                                            │
│      v25  Aug 25  1mo  ◆ § Code Conventions and Gotchas  −194w   bbugyi200.athena.0dj  50d9c3b│
│ ●    v24  Aug 24  1mo  ▣ type: short → core  +1w −1w             sase-sq.1             c9ca0db│
│   ▸  v21  Jul 28  2mo  ◆ § Code Conventions and Gotchas  +51w    bbugyi200.athena.mp…  4fb5980│
│      v20  Jul 18  2mo  ◆ § Code Conventions and Gotchas  +38w    e3                    ad45962│
│      ·· 4 hidden (↦ 2 moves, ≈ 2 reflows) · . show                                          │
│ ⏎ open v21 · = compare v21 → v24 · . hidden · / filter · esc close                          │
╰─────────────────────────────────────────────────────────────────────────────────────────────╯
```

### 7.3 Wiring and tests

- **Wiring.** `_screen_timeline.py` passes the moment to the picker and routes ⏎ through
  `canonical_ordinal`. It applies the compare normalization before
  `explicit_base`/`compare_base` reach `diff_endpoints`.
- **Tests:**
  - `tests/memory/test_timeline_picker.py`: the cells, the now row in all three states,
    the `≡ now` tag, and year formatting.
  - `tests/pager/test_history_timeline.py` and
    `tests/pager/test_app_history_timeline.py`:
    - column fitting at widths from 50 to 160, with no row ever wider than the list
    - the open and cursor markers
    - the footer preview text
    - compare normalization in both directions, including against now
    - the trail push
- **Goldens.** Regenerate `timeline_picker_*` and add `timeline_picker_many`: 260 rows
  with the cursor mid-list and a narrow 60×30 picker.

## 8. Polish (`polish` phase)

- **Docs.**
  - `docs/memory_history.md`: rewrite "Time band anatomy" around the pill, scrubber, and
    rows. Add a "Which version am I reading?" section with the §3 table, the footer
    destinations, the picker anatomy, and D3 in one plain sentence.
  - `docs/pager.md`: update the Keys section.
- **Live review.** Use the real repo through the installed or dev `sase`, taking
  `sase screenshot` captures via the ACE Memory panel `H`, or tmux captures of
  `sase memory history <selector> -f pager`. Review:
  - `sase/memory/gotchas.md` and the bare `gotchas.md`
  - `decisions`
  - `glossary:stitch`
  - `AGENTS.md` (hundreds of versions)
  - a deleted note found with `sase memory history -a -f json`
  - `-d` diff arrivals
  - the picker at both sizes

  Do not edit real memory files to make a dirty state; the goldens cover that state. Fix
  anything that reads unclearly before closing.

- **Performance.**
  - Run with `SASE_TUI_PERF=1` and `SASE_TUI_TRACE=1`.
  - First paint must be unchanged.
  - A warm `(`/`)` step on `AGENTS.md` must take at most 30 ms (p95).
  - `~/.sase/logs/tui_stalls.jsonl` must show no stalls during rapid stepping.
  - The subject line must not rebuild the moment on scroll; check with a counter in a
    unit test.
  - Record the numbers in this phase's bead notes.
- **Goldens.** Run the full `just fix-tui-screenshots`. Inspect every created or updated
  golden and confirm that no unrelated golden changed.
- **Follow-ups.** Record §10 as `PROPOSED FOLLOW-UP:` notes on this phase's bead.

## 9. Verification that applies to every phase

- **Instructions.** Read the `tui`, `tui_perf`, and `lint_and_test` memories before
  changing the pager, and follow their rules:
  - no IO on the render or keystroke path
  - workers use `spawn_pump_free_task` with generation guards
  - targeted `just fix-tui-screenshots -- <selectors>`, with the report inspected
- **Check.** Finish each phase with `sase tool run check`.
- **Unchanged behaviour.** Keep pager behaviour unchanged for sections with no history
  provider. Every new chrome path fails open to today's rendering.
- **Workspaces.** Write no plan, doc, or test that names a specific workspace directory.

## 10. Non-goals and follow-ups

- **Other surfaces.** `sase memory history` text output and the Memory panel History row
  keep their current look. Follow-up: give them the pill vocabulary and the `≡ now`
  marker, using the same `VersionMoment` helpers.
- **Core wire.** Follow-up: promote "now ≡ vN" into the sase-core timeline wire (for
  example `now_matches_ordinal`) if a non-Python frontend needs it (D11).
- **Word diff spacing.** Follow-up: a thin visual separation between adjacent deleted
  and inserted word runs (`short`+`core` currently abut). It must keep the search corpus
  character-aligned.
- **Out of scope.** Side-by-side diffs, an age lens, and restore stay out of scope, as
  in `sase-1dr`.
