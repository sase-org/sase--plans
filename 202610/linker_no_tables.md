---
tier: tale
title: Research-swarm linker replaces tables with PDF-friendly record lists
goal:
  The research-swarm linker converts every table in the lead's report into a list before
  publishing `<name>.md`, and checks that none survives, so Highlights PDFs no longer
  contain cramped equal-width pandoc tables.
size: small
proposed_by: bbugyi200.athena.10g
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.10g](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.10g.md)
- **COMMITS:**
  - [c787c88](https://github.com/sase-org/sase-research-artifacts/commit/c787c88cf0a27e8390a99d92d2ceaf350c91b7d5)
    — feat(research-swarm): have the linker convert tables into PDF-friendly lists

# Plan: Research-swarm linker replaces tables with PDF-friendly record lists

## Context

The research swarm's linker agent publishes `<name>.md`. That is the only file the
`research-highlights` hook renders into a Highlights PDF, through pandoc's `markdown`
reader and XeLaTeX. Pandoc prints every pipe table as a `longtable` with equal-width
columns, cramped wrapped cells, and rows that jump pages.

The research report
`research:202610/pdf_friendly_markdown_table_alternatives/pdf_friendly_markdown_table_alternatives.md`
recommends that the linker convert every table into a "record list". Its "Recommended
solution" section gives paste-ready prompt text for three places in the linker segment.
Read it with
`sase artifact read research:202610/pdf_friendly_markdown_table_alternatives/pdf_friendly_markdown_table_alternatives.md "<reason>"`
if you need the rationale. The text you need is reproduced below.

Where the edit goes:

- **Repo:** the linked repo `sase-research-artifacts`. Open it with
  `sase repo open sase-research-artifacts -r "<reason>"`, work only in the printed path,
  and read its `AGENTS.md` first.
- **Prompt file:** `src/sase_research_artifacts/xprompts/research_swarm.md`. Edit only
  the linker segment, which starts at
  `%if(should_run={{ run_linker }}) %id(linker, clan=research.{@1})`.
- **Test file:** `tests/test_macro_loading.py`.

## Scope

The user asked to update the linker's instructions and to change no more than needed.
This plan therefore makes only the report's three prompt edits and adds one small
regression test. It does **not** touch the lead segment, README, `docs/`, CHANGELOG, or
`bob-cli`. The docs do not describe the linker's formatting rules, so nothing there goes
stale.

## Changes

### 1. Step 2 (Inventory): record table cells

Replace the current step 2 paragraph with this text. The only change is the
parenthetical after "table":

```markdown
2. **Inventory what must survive.** Before writing, list every finding, recommendation,
   caveat, open question, confidence statement, number, date, version, code block, table
   (record its cells; step 3 converts each one into a list), and link in the lead's
   report.
```

### 2. Step 3 (Restructure): add a "No tables." bullet

Insert this bullet directly after the existing
`- **No table of contents and no block of jump links.** ...` bullet. The three
PDF-renderer rules (never number headings, no TOC, no tables) then sit together, and
`- Keep the lead's wording where it works. ...` stays the last step-3 bullet. Use the
text verbatim from the report:

```markdown
- **No tables.** The PDF renderer (pandoc → XeLaTeX) prints tables with equal-width
  columns, cramped wrapped cells, and rows that jump pages. Convert every table in the
  lead's report (pipe, grid, simple, multiline, or HTML) into a list with one item per
  row, keeping every cell's content:
  - Two columns: `- **<first cell>:** <second cell>`.
  - Three or more columns: a bullet whose first line is the row's identifying cell in
    bold (merge an ID column into it, e.g. `- **A1. Restart exactly once**`), then one
    nested bullet per remaining non-empty cell, `- *<Column header>:* <cell>`, in column
    order. Labels go in italics so bold inside a value stays distinct.
  - Ordered rows (phases, steps, ranks, tiers): the same shapes as a numbered list,
    keeping the original numbers.
  - Grouped rows (a blank first cell continues the group above): nest them under one
    bullet for the group.
  - Rows whose cells are all a few words: one line,
    `- **<row>:** <header> <value> · <header> <value> · …`.
  - Numbers: keep units, periods, denominators, and the difference between zero,
    unknown, and not applicable.
  - Long code in a cell: a fenced code block inside the list item.
  - Only when the rows are about six or fewer major named options the report discusses
    at length: one `###` section per option holding the same italic-labelled bullets.
    Never turn a row into a `####` heading. Turn a table caption into a one-line lead-in
    sentence. Put a blank line before every list (pandoc otherwise merges it into the
    paragraph above). Indent nested bullets 2 spaces under `- ` and 3 spaces under
    `1. `. Write URLs as `[text](url)` or `<url>`, never bare. Do not substitute a
    definition list (`Term` then `: definition`), an HTML table, a grid or simple table,
    a table in a code block, a callout or `:::` div, raw LaTeX, or an image. Never drop
    a cell, merge two rows, or add totals, rankings, or conclusions the table did not
    state.
```

Constraints for this insert:

- **Indentation:** keep it exactly as shown. The bullet takes 3 spaces, nested bullets
  5, and their continuation lines 7, matching the other step-3 bullets.
- **No whitespace-only lines.** The existing test
  `test_research_swarm_linker_opens_with_research_query_then_infographic` fails if step
  3 contains one.
- **No template syntax.** The file is Jinja plus SASE directives, so add no `{`, `%`, or
  `$(`. The snippet above contains none. The backticked `###` and `####` match the `##`
  and `###` already in this segment.

### 3. Step 6 (Re-check): confirm no table survives

Append these lines to the end of step 6, directly after the existing
`... infographic,{%- endif %} bottom-line section.` line and before the blank line that
precedes `7. **Write**`:

```markdown
Confirm every cell of every table in the lead's report appears in the new file. When
`pandoc` is available, confirm no table survives:
`pandoc <file> -f markdown -t json | grep -o -i -E '"t":"Table"|"t":"RawBlock","c":\["html","<table' | wc -l`
prints `0`.
```

This text differs from the report's step-6 text in one way: it adds the "When `pandoc`
is available" guard. Step 5 already guards its `pandoc <file> -t html` anchor check the
same way, and the linker may run on a machine without pandoc.

The pipeline was tested during planning on pandoc 3.1.11.1:

- It prints `1` for a pipe table and `1` for an HTML `<table>`.
- It prints `0` for record lists, numbered records with 3-space nested bullets,
  inline-code mentions of `<table>` and `"t":"Table"`, and a pipe table inside a fenced
  code block.

### 4. Regression test

In `tests/test_macro_loading.py`, add this test directly after
`test_research_swarm_linker_opens_with_research_query_then_infographic`. The rule is
unconditional, so one `linker=true` render covers it. Let `ruff format` reflow it if
needed.

```python
def test_research_swarm_linker_converts_tables_to_lists() -> None:
    linker = _swarm_segments({}, linker=True)[-1]
    assert "table (record its cells; step 3 converts each one into a list)" in linker

    step3 = linker.split("3. **Restructure**", 1)[1].split(
        "4. **Validate every link carried over.**", 1
    )[0]
    assert "- **No tables.**" in step3
    assert "`- *<Column header>:* <cell>`" in step3
    assert "3 spaces under `1. `" in step3
    assert "Never turn a row into a `####` heading." in step3

    step6 = linker.split("6. **Re-check against the step-2 inventory**", 1)[1].split(
        "7. **Write**", 1
    )[0]
    assert "Confirm every cell of every table" in step6
    assert '"t":"Table"' in step6
```

Each asserted phrase sits on one source line in the text above. If you rewrap any of
those lines, keep the phrase on one line.

## Verification

- In the `sase-research-artifacts` checkout, run `sase tool run check` (lint and tests).
  It must pass, including the new test and the existing step-3 whitespace-only-line
  assertion.
- Optional: render the linker segment and check that the new bullet sits between the "No
  table of contents" and "Keep the lead's wording" bullets. Use the same expansion the
  tests use, `_swarm_segments({}, linker=True)[-1]`.

Suggested commit subject for the `sase-research-artifacts` repo:
`feat(research-swarm): have the linker convert tables into PDF-friendly lists`.

## Out of scope

These are the report's optional follow-ups. Do not do them in this change:

- **No-linker swarms.** Without a linker, the lead publishes `<name>.md` directly, and
  its tables still reach the PDF. Fixing that means adding the same rule to the lead
  segment when `run_linker` is false.
- **Bob safety net.** A `bob-cli` Lua filter for table widths, or a table-to-list
  conversion, would catch tables in old reports.
- **End-to-end render.** Rendering a real report through `bob ref create` after this
  change.
