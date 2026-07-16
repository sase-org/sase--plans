---
tier: tale
title: Fold Commits, Deltas, and Artifacts into a ranked ARTIFACTS lane in SASE CONTEXT
goal: 'The agent metadata panel presents the selected agent''s outputs as one ARTIFACTS
  lane — Commits, Deltas, and Artifacts — pinned to the bottom of SASE CONTEXT beneath
  PLAN, MEMORY, SKILLS, and WORKSPACES. Lane order is a declared contract with PLAN
  first and ARTIFACTS last, enforced across every combination of present lanes. The
  lane speaks the established lane grammar in the one accent the section already used
  for files, and the section''s cross-lane idioms are named and tested rather than
  left latent.

  '
create_time: 2026-07-16 18:27:01
status: done
prompt: 202607/prompts/artifacts_lane_in_sase_context.md
---

# Plan: Fold Commits, Deltas, and Artifacts into a ranked ARTIFACTS lane in SASE CONTEXT

## Product context

`SASE CONTEXT` now speaks a coherent four-lane grammar (`125f342cb`, `38760e2f2`): one accent per lane, dim structure,
khaki prose, shared idioms for paths and hints. But the agent's **outputs** never joined it. They sit _above_ the
section in three separate blocks, in the dialect the PLAN lane was just rescued from:

| Block        | Today                                                    | Problem                                                                                                              |
| ------------ | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `COMMITS:`   | major section + divider, `bold #87D7FF`                  | `#87D7FF` is literally `COLOR_MEMORY_PRIMARY` — the exact bug class the PLAN restyle just fixed, one section higher. |
| `Deltas:`    | inline block, `bold #87D7FF`                             | Same borrowed MEMORY tint; also title-case while its sibling shouts `COMMITS:`.                                      |
| `Artifacts:` | inline block, `bold #87D7FF`, per-view-mode icon rainbow | Same tint. Icon colors collide with `COLOR_REASON` and `COLOR_EXTERNAL_REPO_NAME` (below).                           |
| Placement    | all three render _before_ `SASE CONTEXT`                 | The panel's most verbose content leads; the ranked, scannable context list is pushed below it.                       |
| Grouping     | three peer blocks at three nesting levels                | Commits, deltas, and artifacts are one idea — "what this agent produced" — rendered as three unrelated things.       |

The deeper issue is ranking. The panel's job is to answer _what is this agent doing_ before _what did it emit_. Today it
answers in the opposite order, and the answer that matters most (PLAN) is buried under the answer that is longest
(commits + every changed file + every artifact). This plan inverts that: PLAN is pinned to the top of the section, and
the three output blocks fold into a single `ARTIFACTS` lane pinned to the bottom.

The lane order becomes a narrative arc, which is the "intuitive" bar:

> **PLAN** (intent) → **MEMORY**, **SKILLS**, **WORKSPACES** (what it consulted and used) → **ARTIFACTS** (what it
> produced).

A declared lane registry with a contract test over every presence combination is the "reliable" bar. And the lane
wearing the one color the section already used for files — rather than a sixth invented hue — is the "beautiful" bar.

## Design decisions

### 1. Lane order becomes a declared, enforced contract

`_agent_context.py` today renders lanes as four sequential `if` blocks, each re-implementing the
`if rendered_lane: text.append("\n")` separator dance. Order is incidental — nothing stops a future edit from appending
a lane in the wrong place, and the only guard is one test asserting PLAN < MEMORY < SKILLS < WORKSPACES.

Replace it with an explicit ordered registry:

```
CONTEXT_LANE_ORDER = ("PLAN", "MEMORY", "SKILLS", "WORKSPACES", "ARTIFACTS")
```

Rendering derives from that tuple: build `(label, renderer)` pairs, drop the empty ones, join survivors with the
blank-line rhythm. PLAN-first and ARTIFACTS-last stop being conventions maintained by reading order and become a data
structure with a test over the full presence powerset (2⁵ combinations): for every subset, the rendered lane labels
appear in registry order, PLAN is first whenever present, ARTIFACTS is last whenever present. This also deletes the
repeated separator boilerplate.

The user's two hard requirements — PLAN always top, ARTIFACTS always bottom — are then structural facts, not comments.

### 2. The ARTIFACTS accent was already in the section: `#5F87FF` → `#87AFFF`

The section's four accents occupy hues ~120° (SKILLS green), ~195° (MEMORY cyan), ~260° (PLAN violet), ~320° (WORKSPACES
pink). The entire warm band is spoken for by _semantic_ colors that cross lanes: error red (0°), the `SASE CONTEXT`
heading gold (~40°), external-repo orange `#FFAF00` (~41°), the delta-modified glyph `#FFD787` (~45°), hint yellow
`#FFFF00` (60°), and prose khaki `#D7D7AF`. A fifth accent cannot go there without impersonating a meaning.

The one free hue is blue (~225°) — and that is exactly where the section's **artifact-path idiom** already lives
(`#87AFFF`, ~220°). Better: the same `+0x28` step that maps PLAN's accent to its tint (`#AF87FF` → `#D7AFFF`, both
non-`FF` channels stepped, `FF` saturated) maps `#5F87FF` → `#87AFFF` — the existing path color, exactly.

So:

```
COLOR_ARTIFACTS_SUBHEADER = "bold #5F87FF"   # new accent
COLOR_ARTIFACTS_PRIMARY   = "bold #87AFFF"   # == today's basename style
```

This is a discovery, not an invention: the ARTIFACTS lane is the lane made entirely of paths, and its accent is the
parent shade of the color this section has always used for a file. The lane's primary value _is_ the path, exactly as a
skill name is the SKILLS primary. Choosing any other hue would give the lane a header color that contradicts its own
content.

Rejected: orange `#FFAF00` — an external-repo row (`◆ name`, bold orange) inside a bold-orange `▸ ARTIFACTS` lane would
read as a sub-lane header. Rejected: teal `#5FD7AF` / `#5FFFD7` — both land within ~30° of MEMORY's cyan while sharing
its bright-cool character, which is a worse collision than blue-vs-cyan (blue is markedly darker: G=`87` against
MEMORY's G=`D7`). Rejected: gold `#FFD75F` — one channel from the delta-modified glyph and the external-repo name
`#FFD787`.

### 3. Three fields, dim title-case labels

The lane holds three block-labelled fields, in causal order — the agent committed, which changed files, which produced
outputs:

```
▸ ARTIFACTS · 2 commits · 7 files · 3 artifacts
  Commits:
    ▣ sase
      [7] 38760e2f2 feat(ace): polish plan lane visuals
  Deltas:
    ~ [8] src/sase/ace/tui/widgets/prompt_panel/_agent_context.py  ~4
  Artifacts:
    ▤ [9] sase/repos/plans/202607/plan.md
```

- **Labels** are `dim` at a 2-cell indent — the same structural role, style, and indent as PLAN's `Title:`/`Goal:`/
  `Path:`. No colon-alignment column is needed: these introduce blocks below rather than values beside, so they stack
  with content between them and never form a ragged column.
- **`COMMITS:` retires** its uppercase spelling and its `append_major_section_divider` call. Uppercase and a divider are
  the major-section idiom; inside SASE CONTEXT it is a field, and fields are title-case. This also settles today's
  inconsistency where one of the three blocks shouts and two do not.
- **Header details** follow MEMORY's `2 reads · 2 files` shape via `count_phrase`, all `dim`: commit rows, delta files
  (primary + linked), artifact paths. Absent field ⇒ absent detail. No content in any of the three ⇒ no lane.
- Fields stay tight (no blank lines between them); the lane is one block, as MEMORY's rows are.

### 4. The artifact icon rainbow retires, exactly as the tier rainbow did

`_ICON_BY_VIEW_MODE` colors five view modes: image `▨ #5FD7AF`, video `▶ #FF875F`, pdf `▤ #D7D7AF`, markdown
`▤ #D7D7AF`, text `• #FFD787`. Two are outright collisions — `#D7D7AF` **is** `COLOR_REASON` (the section's prose color)
and `#FFD787` **is** `COLOR_EXTERNAL_REPO_NAME` — and `#FF875F` sits beside error red.

Crucially the color carries **zero** information: the glyph _shape_ already distinguishes exactly the same four groups
(`▨`, `▶`, `▤`, `•`), and pdf-vs-markdown is not distinguished by color today either — both are `▤ #D7D7AF`. So the
rainbow is pure noise plus two false signals. Icons unify on the lane accent (`bold #5F87FF`), with the existing `dim`
prefix for missing artifacts.

This is the same judgement, on the same grounds, that retired `PLAN_TIER_STYLES` in the approved predecessor: a per-kind
rainbow that duplicated a shape/text signal while impersonating other lanes' colors.

### 5. Shared idioms are named and preserved, not recolored

The palette contract must not be read as "every color in a lane is that lane's accent." The section has **cross-lane
idioms** that are correct precisely because they are identical everywhere:

| Idiom         | Style                                       | Also used by                     |
| ------------- | ------------------------------------------- | -------------------------------- |
| artifact path | `#87AFFF` dirname + `bold #87AFFF` basename | PLAN `Path:`, Deltas, Artifacts  |
| hint          | `[N]` `bold #FFFF00`                        | six sites                        |
| repo identity | `▣` + name pink / `◆` + name orange         | WORKSPACES lane, Commits, Deltas |
| delta glyphs  | green `+`, gold `~`, red `-`                | ChangeSpec DELTAS (documented)   |

Repo identity is the notable one: commit groups and linked delta groups already render repo names in the WORKSPACES
accent. That is **not** a collision to fix — the same repo must look the same everywhere it appears, and recoloring it
per-lane would be the actual bug. The delta glyphs are documented at `docs/ace.md:182` and shared with the ChangeSpec
surface; they stay.

So the ARTIFACTS lane is deliberately not monochrome. Its **chrome** (header, field labels, icons, paths) speaks the
lane grammar; its **content** keeps the section-wide idioms. The palette contract test encodes exactly this: lane
accents and primaries are pairwise distinct, no lane's chrome wears another lane's accent or primary, and the enumerated
shared idioms are exempt. This is the generalization of the test the predecessor plan introduced, and it is what makes
the grammar enforceable for a fifth lane rather than re-litigated per lane.

`_COLOR_PATH`/`_COLOR_BASENAME` are currently duplicated byte-for-byte in `_agent_artifacts.py` and `deltas_builder.py`;
both collapse onto shared constants in `_agent_context_common.py`. Values do not change, so the ChangeSpec surface is
untouched.

### 6. SASE CONTEXT renders atomically, on the full path only

This is the one genuinely debatable call, so it is stated plainly.

`append_agent_commits_section` is the only one of the three that renders on the `cheap` (immediate j/k) path — it reads
in-memory `step_output` and needs no disk. Deltas and artifacts require the async `summary`, as does all of SASE
CONTEXT.

The lane is gated with the rest of the section on `not cheap and summary is not None`. **Commits therefore stop
appearing on the cheap frame** and arrive with the debounced full render, exactly as deltas and artifacts do today.

The alternative — render a commits-only ARTIFACTS lane immediately — was rejected because it is _worse_ motion, not
better. Today COMMITS sits above SASE CONTEXT, so it never moves when enrichment lands. If a commits-only lane rendered
at the bottom of the section, enrichment would insert up to four lanes _above_ it and shove it down by many lines, and a
section heading would appear over a single partial lane. Gating it instead means SASE CONTEXT appears once, complete, in
one motion — and its hint numbers are allocated once from the complete lane set rather than being reshuffled when PLAN
arrives and claims `[1]`.

Per the TUI-perf rules this adds no I/O, refresh path, or per-render work; commits move from one already-rendered branch
to another. The cost is honest and bounded: commits are absent from a sub-second transient frame.

### 7. What deliberately does not change

- The four existing accents, the `▸` glyph, the blank-line rhythm, and MEMORY/SKILLS/WORKSPACES internals.
- The PLAN lane, its responsive `Table.grid` fold at 80 cells, and its `(start, end)` range plumbing.
- Hint semantics; numbers stay in visual reading order, which the registry preserves for free. Absolute numbers do
  change (PLAN/MEMORY now precede the outputs) — that is the point, and the tests asserting the old numbers encode the
  old order.
- Commit attribution, delta parsing/merging, artifact discovery, and `agent_associated_plan.py`. This is a presentation
  and placement change; the Rust core boundary is not crossed.
- ChangeSpec DELTAS rendering: the shared builder gains **defaulted** knobs only (`indent=""`, `header_style`), so its
  existing callers reproduce current output byte-for-byte.

## Implementation outline

1. **`_agent_context_common.py`** — add `COLOR_ARTIFACTS_SUBHEADER`, `COLOR_ARTIFACTS_PRIMARY`, and the shared
   `COLOR_ARTIFACT_PATH` / `COLOR_ARTIFACT_BASENAME` beside the existing four-lane palette, so all five lanes read as
   one table in one file.
2. **`deltas_builder.py`** — add `indent: str = ""` (applied to primary entries, group headers, and group entries,
   preserving today's relative 2/2/4 shape) and `header_style` defaulting to the current constant; consume the shared
   path constants. Existing callers pass neither and are unaffected.
3. **`_agent_commits.py`** — expose the commit groups (public accessor) and a row renderer; drop the divider and the
   `COMMITS:` header from the module. Attribution logic is untouched.
4. **New `_agent_artifacts_lane.py`** — owns the lane: presence test, `count_phrase` details, `▸ ARTIFACTS` header, and
   the three dim field labels delegating to the commit/delta/artifact renderers at lane indent.
5. **`_agent_artifacts.py`** — retire `_ICON_BY_VIEW_MODE` colors onto the lane accent; drop the local header and
   duplicated path constants.
6. **`_agent_context.py`** — the lane registry from decision 1; accept the agent plus delta/artifact inputs and render
   ARTIFACTS last.
7. **`_agent_display_header.py`** — delete the three call sites (lines ~464–482) and route their inputs into
   `append_agent_context_section`. Meta fields already precede the section, so the resulting order is Timestamps →
   OUTPUT VARIABLES → WORKFLOW VARIABLES → SASE CONTEXT → SLOW TOOL CALLS → ERROR.
8. **Docs** — the ordering claims are load-bearing and become false: `docs/ace.md:1555-1556` ("the full order is `PLAN`,
   `MEMORY`, `SKILLS`, then `WORKSPACES`"), `README.md:189` (`PLAN` → `MEMORY` → `SKILLS` → `WORKSPACES`), and
   `docs/ace.md:347-354` ("ahead of the audited ... lanes") all gain ARTIFACTS as the pinned last lane. The `COMMITS`
   bullet (`docs/ace.md:1566-1568`) moves under the SASE CONTEXT bullet and is reworded from major section to lane
   field; the DELTAS (`docs/ace.md:399-406`) and ARTIFACTS (`docs/ace.md:452-454`) passages, plus
   `docs/agent_images.md:163-165`, are re-anchored to the lane. `docs/ace.md:182` (green `+`, gold `~`, red `-`) stays
   true and is left alone.

## Testing

Contract tests to invert — each encodes the old placement, so each is evidence the move landed:

- `test_agent_display_artifact_metadata.py:24` — `Deltas: < Artifacts: < SASE CONTEXT < ▸ PLAN` inverts to
  `SASE CONTEXT < ▸ PLAN < ▸ ARTIFACTS < Commits: < Deltas: < Artifacts:`.
- `test_agent_display_plan_section.py:316` — the maximal-append-flow ordering test; its `COMMITS`/`DELTAS`/ `ARTIFACTS`
  stubs move from "before context" to inside it.
- `test_agent_display_output_variables.py:72` — `OUTPUT VARIABLES < Artifacts:` still holds but for a new reason;
  re-anchor on the section.
- `test_agent_context.py:195` (lane order) and `:226` (distinct accents) extend to five lanes.
- Hint-number tests encoding the old sequence: `test_agent_display_artifact_metadata.py:195` (deltas `[1]`, plan `[2]` →
  plan `[1]`, deltas `[2]`), `test_agent_deltas.py:475` (deltas `[1]`).
- `test_agent_display_step_metadata.py` — the ~15 `"COMMITS:\n"` assertions and the divider assertion (`:41`) become the
  lane's `Commits:` field. Direct-call hint tests (`:272`, `:135`) keep working.
- `test_agent_deltas.py:455`, `test_agent_display_artifact_metadata.py:134`, `test_prompt_panel_header.py:316` — the
  cheap-path omission tests gain a commits sibling assertion (decision 6).
- `test_agent_display_header_only.py:240` — pre/post-enrichment expectations gain commits.

New coverage (the "reliable" bar):

- **Lane order powerset:** for all 2⁵ presence combinations, rendered labels follow `CONTEXT_LANE_ORDER`, PLAN is first
  when present, ARTIFACTS is last when present. This is the test that makes the user's two requirements permanent.
- **Palette contract, generalized to five lanes:** accents and primaries pairwise distinct; no lane's chrome wears
  another lane's accent/primary; the enumerated shared idioms (decision 5) are exempt and asserted identical across
  their sites. This is what would have caught the `bold #87D7FF` headers.
- **Icon monochrome:** every view-mode icon renders in the lane accent and view mode is carried by shape alone.
- **Shared-builder non-regression:** `build_delta_entries_section` with default `indent`/`header_style` reproduces
  ChangeSpec output byte-for-byte — the guard that decision 7's "no ChangeSpec change" claim holds.

Visual snapshots expected to change: `agents_commit_messages_panel_120x40` (commits + deltas),
`agents_artifact_type_icons_120x40` (icons + placement), and `agents_context_zoom_modal_120x40` if its fixture carries
any of the three; `agents_metadata_zoom_modal_120x40` and `agents_linked_repo_diff_file_panel_120x40` may follow. Their
wait anchors and substring assertions (`"Deltas:"`, `"SASE CONTEXT"`, repo/file names) survive a move and restyle since
the asserted text persists — except `agents_commit_messages_panel`, whose anchors must be re-checked against the retired
uppercase `COMMITS:`. Inspect every diff artifact under `.pytest_cache/sase-visual/` before accepting; do not
blanket-update.

Close with `just install && just check`, plus `just test-visual`. Note the pre-existing, unrelated `just check` blocker:
stale Symvision launch/notification symbols. Do not absorb a fix for it (it forces unrelated public-API edits); if it
fires, run the full and visual suites directly and report it as pre-existing.

## Risks

- **Commits absent from the cheap frame** (decision 6). The deliberate trade for an atomic section and stable hint
  allocation. If it proves annoying in practice the fallback is to feed commits through the cached summary rather than
  to re-split the lane.
- **Blue vs. violet vs. cyan proximity.** ARTIFACTS `#5F87FF` is nearer PLAN `#AF87FF` in hue than any existing lane
  pair. Mitigated by lightness (MEMORY's cyan is far lighter), by maximal vertical separation (the registry pins the two
  at opposite ends of the section), and by the fact that blue already means "file" here. If it reads poorly on the real
  TUI, darkening to `#5F5FFF` preserves the tint relationship's spirit at some cost to the `#87AFFF` math.
- **Hint renumbering.** Absolute numbers shift for anyone with muscle memory. Unavoidable and correct: hints follow
  visual order, and visual order is what changed.
- **Snapshot churn masking a regression.** Several PNGs change for placement + color reasons at once; inspect each
  artifact rather than blanket-accepting.
- **Shared-builder blast radius.** `deltas_builder.py` feeds the ChangeSpec surface. Defaulted params plus the
  byte-for-byte non-regression test bound this; if a default must change, stop and reconsider.
- **Lane verbosity at depth.** The lane adds 2 cells of indent to already-verbose content. Accepted: the fields are
  pinned last precisely so their width cost lands where it interrupts least, and the 80-cell reason budget and delta
  rows are unaffected.
