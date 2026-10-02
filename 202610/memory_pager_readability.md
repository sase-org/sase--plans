---
tier: tale
title: Make memory versions comfortable to read in the pager
goal: "Memory versions and other Markdown documents have a calm, legible reading
  hierarchy, with readable inline code, quiet metadata, consistent headings, and clear
  version context across dark and light pager hosts.

  "
size: medium
proposed_by: bbugyi200.athena.0va
create_time: 2026-10-02 07:08:21
status: wip
---

# Plan: Make memory versions comfortable to read in the pager

## Outcome and scope

Make the document the visual focus of the memory-version reader. Use neutral prose, soft
but clearly readable code accents, consistent heading weight, and compact history cues.
Keep source Markdown visible, including its punctuation, frontmatter, line numbers, and
exact source characters. The same Markdown styling should apply when a note is opened
directly, followed through a link, or viewed at any revision.

This is one bounded presentation change suitable for one coding agent. It belongs in the
Python/Textual pager: color resolution, token-to-display-role mapping, composed Rich
styles, and visual regression coverage. History retrieval, revision semantics, link
resolution, memory parsing, and Rust wire APIs stay outside this change. Do not edit any
canonical memory notes to create fixtures. There are no new CLI flags, settings,
keybindings, themes to select, or dependencies.

## Evidence and diagnosis

The user's screenshot, `~/tmp/screenshots/20261002_065709.png`, shows a dense
`tui_perf.md` version: amber frontmatter paragraphs and list markers, dark blue inline
identifiers, a blue first-level heading but an unstyled `## Rules`, and pale text on a
bright orange history strip. Preserve this description in the implementation context
even if the screenshot is unavailable later.

Inspection and small read-only runtime probes established:

- `src/sase/pager/syntax_theme.py` derives colors from raw theme attributes. For
  `textual-dark`, `Theme.background` is absent, so contrast is checked against
  `#000000`; the mounted pager actually paints `#121212`. Its heading color `#0178D4` is
  only about **4.14:1** against that surface, and inline code `#5983A1` is about
  **4.63:1**. Inline code technically clears the minimum but is much less readable than
  the surrounding prose in a dense technical note.
- `markdown_token_role()` in `src/sase/pager/syntax.py` handles `Generic.Heading` but
  misses `Generic.Subheading`, used by Pygments for `##` and deeper headings. Its
  generic source-token fallback makes list bullets and numbers bold keyword accents.
- `_markdown_syntax.py` sends frontmatter through the ordinary YAML code mapper. Keys,
  plain scalar prose, and folded prose can all become `CONSTANT`, coloring entire
  descriptions amber. This is a presentation mapping issue, not a reason to parse or
  rewrite memory YAML.
- The current history palette intends a faint violet tint; a default-theme probe
  produces `#15111E`. That does **not** explain the screenshot's orange strip. Treat the
  screenshot as a visual failure to reproduce, not proof that a particular tint
  calculation caused orange. Inspect composed widget and terminal styles when validating
  the new surface.
- A direct palette probe with `textual-ansi` raises a Rich `StyleSyntaxError` for
  `ansi_yellow`. Theme resolution must safely handle terminal/ANSI colors instead of
  assuming every Textual color serializes to a Rich-compatible RGB string.
- Existing syntax goldens show a short toy Markdown document; history goldens mostly use
  short garden prose. Neither adequately represents a dense memory note with many inline
  identifiers and a long frontmatter description.

## Visual design

### Reading hierarchy

Use a small Markdown-specific palette derived from the resolved host surface.
Illustrative anchors below specify the intended appearance on the default dark and light
surfaces; preserve host theme character through modest blending and contrast correction
rather than applying these values to every theme unchanged.

| Element                                  | Treatment                                   | Dark anchor on `#121212` | Light anchor on `#E0E0E0` |
| ---------------------------------------- | ------------------------------------------- | ------------------------ | ------------------------- |
| Prose and frontmatter values             | Normal-weight neutral text                  | `#E0E0E0`                | `#202020`                 |
| All Markdown headings                    | Bold, restrained cool accent                | `#B8D8E8`                | `#173C50`                 |
| Inline code                              | Clearly readable cool accent; normal weight | `#AACADD`                | `#234256`                 |
| Metadata keys and structural punctuation | Quiet neutral; keys may use weight          | `#A3ADB7`                | `#3C4248`                 |
| Strong/emphasis                          | Prose color with bold/italic respectively   | Inherit prose            | Inherit prose             |

All these anchors exceed 7:1 on their stated surfaces. Avoid colored backgrounds behind
individual code spans: repeated little boxes would interrupt the reading rhythm. Keep
backticks visible. Ordered/unordered list markers and fence delimiters use the quiet
structural role, not amber keywords. Frontmatter values, including multiline
descriptions, read as ordinary text; keys are identifiable without making the whole
block shout. Source-code YAML and YAML inside labeled fences retain the ordinary code
palette. Fenced languages retain useful language colors; unknown fences remain readable
literal text.

Use dedicated Markdown metadata/structural roles where needed. Handle
`Generic.Subheading` alongside `Generic.Heading`; do not replace Pygments with a new
Markdown parser. Preserve the existing region boundaries, offset validation, child-lexer
failure behavior, and span budgets. A small frontmatter-specific token-role mapper can
distinguish keys, values, comments, and delimiters without semantic YAML parsing. Do not
inherit generic code roles for ordinary prose structure merely because Pygments uses
`Keyword` or `Literal` tokens.

### History and interaction hierarchy

Use the host's neutral surface for the full history strip. Keep the recognizable violet
**PAST** pill and narrow gutter rail as the version cues; do not fill the whole strip
with an accent. Retain the existing NOW, uncommitted, and deleted wording, icons,
compact forms, and semantic colors. The version, timestamp, summary, and provenance
links remain in their current rows and keep their existing keyboard behavior. Give
metadata explicit readable secondary foregrounds instead of relying on terminal `dim` to
reduce contrast unpredictably.

Existing small link-key capsules remain discoverable. Preserve artifact-kind link colors
and label assignment, but correct their foreground against the surface where painted if
needed. Apply the same target foreground in the labeled body and the search base so
entering search does not change a link's appearance. Active search highlights retain
clear priority and a readable foreground/background pair. Keep existing missing-link and
inactive-label distinctions intentional.

Style ordering remains: base text, syntax, actionable link emphasis, then active search
treatment; history/change/goto rails remain in the gutter. Diff views and
producer-authored ANSI/Rich documents continue to own their styling and must not be
recolored as Markdown.

## Implementation

1. **Resolve colors from the actual host surfaces.** In `src/sase/pager/syntax_theme.py`
   and the pager host glue, introduce the smallest immutable presentation context needed
   to carry resolved body foreground, body background, and neutral chrome surface, plus
   the theme accents. Obtain these from Textual's resolved variables/computed styles
   after mount, including alpha/inherited backgrounds where relevant, instead of
   treating missing raw theme attributes as black. Keep pure palette construction
   independently testable. Support both standalone `SasePager` and a pager embedded in
   ACE. Normalize terminal/ANSI colors safely or use a documented conservative
   light/dark neutral fallback when RGB is unavailable; never emit an invalid Rich color
   or query the terminal synchronously.

2. **Apply the Markdown hierarchy.** Update `SyntaxRole`, `markdown_token_role`,
   `_markdown_syntax.py`, and `syntax_theme.py` together. Separate frontmatter display
   mapping from ordinary YAML mapping. Make all heading levels and list structures
   consistent, and implement the readable inline-code treatment. Keep color helpers
   scoped to pager presentation; avoid changes to shared xprompt/editor palettes just
   because a helper is currently imported from there.

3. **Quiet the surrounding chrome and preserve overlays.** Update
   `src/sase/pager/history/styles.py`, `_screen_chrome.py`, `_screen_time_band.py`,
   `_time_band_render.py`, and `_styles.py` as necessary to use the neutral history
   strip and correctly paired text and pill colors. Inspect `_labels.py` and its call
   sites for link colors that violate contrast on light surfaces; make any correction
   local to pager painting, shared by labeled and search rendering. Preserve the
   established shape and placement of pills, labels, gutters, and rows.

4. **Integrate theme changes without disturbing reading.** In `_screen_syntax.py`,
   include every resolved color input in the styled-cache signature. Retain token-result
   reuse and bounded background lexing. A theme change must invalidate old styled
   results, repaint history and both split panes, and refresh the active search base.
   Guard in-flight work with the existing generation/signature mechanism so a completion
   prepared for the old theme cannot overwrite the new one. Preserve first paint, scroll
   position, source offsets, link targets, and reading anchors. Avoid file I/O or theme
   probing in key handlers.

5. **Document the result.** Update the syntax-highlighting section of `docs/pager.md`
   and relevant visual description in `docs/memory_history.md` to describe the reading
   hierarchy and restrained history cues. Existing `--syntax none`,
   `pager.syntax: never`, `--color never`, and plain/redirected output retain their
   behavior. Do not add an appearance switch to compensate for an unreadable default.

## Contrast and reliability contracts

- On the built-in dark, light, and ACE Flexoki surfaces, target at least **7:1** for
  prose, headings, inline code, and frontmatter text. On unusual custom midtone
  backgrounds where 7:1 is physically impossible, use the strongest readable neutral
  fallback and guarantee at least **4.5:1** for generated text whose RGB background is
  known. Validate the neutral foreground too.
- Other generated syntax text, active links, metadata, and pill labels must meet
  **4.5:1** against their effective backgrounds. Rails and change marks need **3:1**.
  These are rendered-color contracts, not just palette-table assertions; include
  inherited colors, background fills, and intentional style modifiers.
- Unknown terminal-native colors degrade safely to readable host/default text; do not
  claim an RGB contrast guarantee when the terminal palette is unknown.
  Plain/color-disabled output still conveys version status in words and symbols.
- This change never changes source text, line numbering, wrapping policy, search
  offsets, copy output, follow/edit destinations, or which revision is displayed. The
  existing syntax size caps and malformed-input fallbacks continue to apply.

## Validation and visual acceptance

Extend existing tests with meaningful regressions, keeping fixtures synthetic and
deterministic:

- `tests/pager/test_syntax.py`: first/deeper headings, ordered/unordered lists, inline
  backticks, plain/quoted/folded frontmatter values, and separation from fenced YAML.
  Retain exact offsets and text for Unicode, tabs, CRLF, missing final newline,
  malformed frontmatter, and unknown/incomplete fences.
- `tests/pager/test_syntax_theme.py` and `test_history_styles.py`: resolved default
  backgrounds (including the missing raw dark background), dark/light/Flexoki, all
  built-in palette construction, invalid/custom and terminal-native colors, actual
  contrast bounds, and readable neutral fallbacks. Do not hard-code every final hex
  value; assert the hierarchy's behavior and contrast requirements.
- `tests/pager/test_screen_syntax.py` and focused app tests: asynchronous theme switch
  while lexing, no stale palette publication, active search restoration, now/past
  stepping, and split panes using the same current theme. Reuse existing tests for label
  stability, source copy, producer styles, syntax opt-out, and plain output; add
  coverage only for gaps touched by this change.
- Add a dense memory-note case to the pager visual suite containing multiline
  frontmatter, heading levels, numbered bold rules, many inline identifiers, followable
  paths, and a small fenced sample. Capture it in NOW and PAST states, dark and light,
  at 120x40 and 60x30. Add one Flexoki/embedded-host example and one active
  search-over-code/link example. Inject fixed history, time, and resolver data and wait
  for syntax completion using existing fixture patterns.

Read the applicable TUI performance/screenshot and lint/testing reference memories
through `/sase_memory_read` before implementation. Run focused tests while working, then
`just fix` and the required `sase tool run check` (the recorded `just check` recipe).
Run targeted `just fix-tui-screenshots -- tests/pager/visual/` for the affected pager
suite, narrowing selectors only when all affected goldens are covered. Use
`/sase_monitor` for long commands and inspect the resulting report and changed PNGs
before completion; a `partial` update is not complete evidence. Do not run
`just check-full` for this work.

Inspect rendered PNGs, not only test results. Compare a dense PAST note with the user's
screenshot: identifiers must read as effortlessly as prose, metadata must stop
dominating the page, every heading must be recognizable, and history must remain obvious
without a saturated full-width stripe. Check the narrow layout, light-theme link
readability, search contrast, and terminal color degradation explicitly. Review any
changed goldens outside these intended surfaces before accepting them. Report which
views were visually reviewed and any limits of terminal-native color verification.

## Completion criteria

The approved design is implemented on all Markdown pager entry paths; measured contrast
and offset/interaction regressions pass; focused visuals demonstrate the dense
memory-note reading experience in both themes and widths; the required verification
passes; and the user receives a concise explanation with before/after visual evidence.
Finalize through the normal host-owned SASE completion flow.
