---
tier: tale
title: Restyle the PLAN lane to the SASE CONTEXT lane grammar
goal: 'The PLAN lane inside SASE CONTEXT reads as a native sibling of the MEMORY,
  SKILLS, and WORKSPACES lanes: one purple accent per lane with dim structural columns,
  field-label colons that align on a shared column, no underline or italic competing
  with the section heading, and a palette contract test that prevents any lane from
  wearing another lane''s colors again.

  '
create_time: 2026-07-16 17:53:21
status: wip
prompt: 202607/prompts/plan_lane_visual_polish.md
---

# Plan: Restyle the PLAN lane to the SASE CONTEXT lane grammar

## Product context

The PLAN lane recently moved inside `SASE CONTEXT` (commit `125f342cb`). The move is right, but the lane still wears the
styles it had as a standalone major section, and a user screenshot of the live TUI shows the result: clashing colors and
ragged alignment exactly where the panel should read as one calm, ranked list.

Sampling the screenshot pixel-by-pixel makes the complaints concrete:

| Element                      | Today                           | Problem                                                                                                                                  |
| ---------------------------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `Title:` / `Goal:` / `Path:` | `bold #87D7FF` cyan             | This is literally `COLOR_MEMORY_PRIMARY` — the PLAN lane's labels wear the MEMORY lane's file color.                                     |
| `tale` tier token            | `bold #FFD75F` gold             | Sits one line under the gold-underlined `SASE CONTEXT` heading and beside the purple `▸ PLAN` label — three saturated hues in two lines. |
| Title value                  | `#D7D7FF underline`             | Underline is the major-heading (and link) idiom; two underlined lines fight within four rows.                                            |
| Goal value                   | `italic #D7D7FF`                | Five lines of italic lavender; in this section italic means metadata (`↩ frontmatter`, role labels), never primary prose.                |
| Field-label colons           | left-aligned in a 9-cell column | `Title:`'s colon lands one cell right of `Goal:`'s and `Path:`'s — visibly ragged.                                                       |
| Epic phase id                | `bold #5FD7D7` cyan             | Another cyan inside the purple lane.                                                                                                     |
| Header phase count           | `#AFAFD7`                       | Every other lane's header details (`2 reads · 2 files`) are uniformly `dim`.                                                             |

The deeper issue: the three event lanes already speak a coherent visual grammar, and PLAN speaks a foreign dialect. The
grammar, latent in `_agent_context_common.py`:

- **Header:** `▸ LABEL` in `bold <accent>`; everything after the `·` is `dim` (`COLOR_SUMMARY`).
- **Primary values:** a lighter tint of the lane accent — MEMORY `#5FD7FF → #87D7FF`, SKILLS `#5FD75F → #87FF87`,
  WORKSPACES `#FF87D7 → #FFAFD7`.
- **Structure** (timestamps, separators, ordinals): `dim`.
- **Explanatory prose:** `COLOR_REASON` (`#D7D7AF`) everywhere.
- **File paths:** the shared artifact-path idiom (blue dirname, bold basename); **hints:** `bold #FFFF00`.
- **Glyphs:** bold accent.

This plan restyles the PLAN lane to that grammar. A user who has learned to read any lane has then learned to read all
four — that is the "intuitive" bar. A new palette contract test makes the grammar enforceable — that is the "reliable"
bar. And the section drops from six competing hues to one accent per lane plus shared idioms — that is the "beautiful"
bar.

## Design decisions

### 1. Tier and phase count become dim header details; `PLAN_TIER_STYLES` retires

The lane header becomes `▸ PLAN · tale` with `tale` in the same `dim` as MEMORY's `2 reads`. The per-tier rainbow is
actively harmful here, not just noisy: `plan` tier cyan `#5FD7FF` **is** the MEMORY accent, `epic` purple `#AF87FF`
**is** the PLAN accent itself (so the color carries zero information exactly when a phase roadmap follows), and `tale`
gold fights the gold section heading.

This deliberately supersedes the previous plan's "the color moves, it does not disappear" choice — the screenshot is the
evidence it doesn't work. No signal is truly lost: tier identity colors still live where tier is a _status_ (the
agents-tree `TALE DONE` badges); in a lane header the tier is a _detail_, and details are dim. `PLAN_TIER_STYLES` and
`PLAN_PHASE_COUNT_STYLE` are deleted outright (they have no other consumers — verified by grep), so Symvision sees no
orphaned symbols.

Rejected alternative: keeping per-tier colors but non-bold. Still lets `plan` impersonate MEMORY and `epic` vanish into
its own lane accent; dimming only `tale` would make tier colors inconsistent with each other.

### 2. PLAN gets a real primary tint: `#D7AFFF`

The other three lanes lighten their accent by the same channel step (`+0x28` on the non-saturated channels:
`#5FD7FF→#87D7FF`, `#5FD75F→#87FF87`, `#FF87D7→#FFAFD7`). Applying the identical step to PLAN's `#AF87FF` yields
`#D7AFFF`. The plan **title** (and each epic **phase title**) renders `bold #D7AFFF` — the lane's headline wears the
lane's tint, exactly like a skill name wears green.

Add `COLOR_PLAN_PRIMARY = "bold #D7AFFF"` to `_agent_context_common.py` beside its three siblings so the four-lane
palette reads as one table in one file. The title's underline is dropped (decision 4).

### 3. Field labels are structure: dim, with colons aligned on one column

`Title:`, `Goal:`, and `Path:` move from `bold #87D7FF` to `dim` — they play the same structural role the dim timestamp
column plays in event lanes, at the same 2-cell indent. They also become **right-justified** inside the existing 9-cell
label column (` Title:`, `  Goal:`, `  Path:`), so all three colons land on the same column while values stay at
column 9. This is the classic definition-list alignment; the value column, the 80-cell responsive fold, and
`PLAN_FIELD_LABEL_WIDTH` are all unchanged, so reflow behavior is untouched.

### 4. Underline and italic retire from values; prose joins the section's prose color

- **Title:** `bold #D7AFFF`, no underline. Underline stays reserved for the `SASE CONTEXT` heading (and link-like
  affordances elsewhere in the app).
- **Goal:** `COLOR_REASON` (`#D7D7AF`), regular weight, no italic. The goal is explanatory prose, and this section
  already has exactly one prose color — the `↳` reason lines. After this change every paragraph of prose in
  `SASE CONTEXT` is the same khaki, full stop.
- **Path:** unchanged — it already uses the shared artifact-path idiom and the `[N]` `bold #FFFF00` hint, both of which
  are section-wide idioms that must keep matching Deltas/Artifacts.

### 5. The epic phase roadmap joins the family

| Element      | Today               | New                                   | Why                                                                                      |
| ------------ | ------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------- |
| ordinal      | `dim #8787AF`       | `dim`                                 | structure is dim, no bespoke grays                                                       |
| `◆` glyph    | `#AF87FF`           | `bold #AF87FF`                        | glyphs are bold accent in every lane                                                     |
| phase title  | `bold #D7D7FF`      | `bold #D7AFFF` (`COLOR_PLAN_PRIMARY`) | headline wears the lane tint                                                             |
| phase id     | `bold #5FD7D7` cyan | `#AF87FF`                             | the last foreign cyan leaves the purple lane; non-bold keeps it subordinate to the title |
| dependencies | `dim #AFAFAF`       | `dim`                                 | structure is dim                                                                         |
| model        | `italic #AF87FF`    | unchanged                             | metadata annotation — italic accent is the idiom                                         |
| description  | `italic #AFAFAF`    | `COLOR_REASON` (`#D7D7AF`)            | prose is khaki, and italic means metadata                                                |

The `unavailable` placeholders (`tier unavailable`, `phases unavailable`, missing title/goal) unify on `COLOR_EMPTY`
(`dim italic`) instead of the bespoke `dim italic #878787`; the `(missing)` path suffix keeps its distinct
`dim italic #FF8787` because it is an error signal, not an empty state.

### 6. What deliberately does not change

- The four lane **accents**, the `▸` glyph, lane order, and blank-line rhythm.
- The responsive splice: `logical_text`, the `(start, end)` range plumbing, `Table.grid` folding at
  `PLAN_SECTION_MAX_WIDTH = 80`. Styles ride the existing render path — per the TUI-perf rules, no new I/O, refresh
  paths, or per-render stat work is introduced.
- `agent_associated_plan.py` (data resolution) and the Rust core boundary — this is presentation-only and stays in this
  repo.
- The `[N]` hint style, which must keep matching the other five hint sites.

## Implementation outline

1. **`_agent_context_common.py`** — add `COLOR_PLAN_PRIMARY = "bold #D7AFFF"` next to `COLOR_MEMORY_PRIMARY` /
   `COLOR_SKILL_NAME` / `COLOR_WORKSPACE_NAME`.
2. **`_agent_plan_section.py`** — retire `PLAN_TIER_STYLES`, `PLAN_PHASE_COUNT_STYLE`, and the bespoke
   gray/cyan/lavender constants per decisions 1–5; render tier and phase count through the header's `dim` details style;
   right-justify the three field labels inside the existing 9-cell column (both in `_rows()` and therefore identically
   in `logical_text` and the folded `Table.grid` path); point title/phase-title at `COLOR_PLAN_PRIMARY`,
   goal/phase-description at `COLOR_REASON`, placeholders at `COLOR_EMPTY`.
3. **Docs** — `docs/ace.md`'s PLAN-lane paragraphs (~357–363 and ~1544–1550) describe structure and tier semantics, not
   colors; verify no sentence claims a styling fact this change falsifies and leave them alone otherwise. `README.md`'s
   lane inventory is unaffected.

## Testing

Contract tests to invert in `tests/ace/tui/widgets/test_agent_display_plan_section.py`:

- The label/title/goal style assertions (`bold #87D7FF`, `#D7D7FF underline`, `italic #D7D7FF`, ~lines 181–185 and 499)
  flip to `dim`, `bold #D7AFFF`, and `#D7D7AF`.
- `test_tier_values_have_distinct_restrained_styles` (~line 419) inverts: its premise was per-tier colors. Replace with
  a parametrized assertion that every tier token renders in the header's uniform dim details style.
- Label-string assertions using `"  Title: "` / `"  Goal: "` / `"  Path: "` gain the right-justified spellings; the
  lossless `logical_text` reconstruction and the width-32/120 fold tests must keep passing unmodified in behavior — they
  are the guard that the splice survived the restyle.
- Phase-roadmap style assertions update per the decision-5 table.

New coverage (the "reliable" bar):

- **Palette contract:** the four lane accents and the four primary tints are pairwise distinct, and no style used by the
  PLAN lane equals another lane's accent or primary. This is the test that would have caught today's `bold #87D7FF`
  label bug at review time.
- **Header uniformity:** all four lane headers render their post-`·` details in `COLOR_SUMMARY`.
- **Colon alignment:** the three rendered labels have equal cell width with the colon in the same cell — the regression
  canary for the alignment fix.
- **No decoration:** the title span carries no underline and the goal span no italic, asserted directly so the exact
  user complaints cannot silently return.

Visual snapshots — three PNGs legitimately change: `agents_plan_goal_metadata_120x40`,
`agents_epic_phase_roadmap_120x40`, and `agents_context_zoom_modal_120x40`. Their wait anchors (`"SASE CONTEXT"`,
`"3 phases"`) and SVG substring assertions (`"Title:"`, `"tale"`, …) all survive a restyle since no asserted text
changes; confirm that, inspect the diff artifacts under `.pytest_cache/sase-visual/`, and only then accept with
`--sase-update-visual-snapshots`.

Close with `just install && just check`, plus `just test-visual`. Note: `just check` currently has two pre-existing
blockers unrelated to this work — stale `sase-6d` Symvision whitelist entries, and machine-level generated `sase_run`
provider-skill drift reported by a later validation stage. Do not absorb fixes for either into this change (the
Symvision cleanup is known to force unrelated public-API edits); if they still fire, run the full test suite and visual
suite directly and report the blockers as pre-existing.

## Risks

- **Dim labels on low-contrast terminals.** `dim` is already the timestamp column's style at the same indent, so the
  lane cannot become less readable than its siblings; the definition-list shape (aligned colons, bright values) carries
  the structure even if `dim` renders faint.
- **Tier scannability regresses for tier-focused users.** Mitigated: the tier stays in the header as text, and
  tier-as-status keeps its color identity in the agents tree.
- **Snapshot churn masking a real regression.** Three PNGs change for color/spacing reasons; inspect each diff artifact
  rather than blanket-accepting.
- **Stale style docs.** If any doc or help text turns out to name the old colors, update it in the same change rather
  than leaving a false claim (step 3 verifies).
