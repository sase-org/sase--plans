---
tier: tale
title: Add a tables_to_lists input to
goal:
  The research-swarm linker keeps tables by default, and tables_to_lists=true restores
  today's table-to-list conversion byte-for-byte.
size: small
decisions:
  keep_tables_rule:
    ask:
      When `tables_to_lists=false`, should the linker get an explicit rule to keep the
      lead's tables as tables?
    choices:
      explicit:
        Add a short "Keep tables as tables" bullet so default runs reliably publish real
        tables
      silent:
        Restore the pre-conversion wording; the linker may still turn some tables into
        lists
    default: explicit
    why:
      The off default exists to show real tables in the PDF; a free linker may convert
      some
    answer: silent
proposed_by: bbugyi200.athena.17q
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 15:49:54
status: wip
---

# Add a `tables_to_lists` input to `#research_swarm`

## Goal

Add a bool input, `tables_to_lists` (default `false`), to the `#research_swarm` macro.
It controls whether the linker agent is told to convert every table in the lead's report
into a list. With the default, published reports keep real Markdown tables so the user
can judge how `bob ref create` renders them in the PDF. Passing `tables_to_lists=true`
must restore today's linker instructions **byte-for-byte**. The user expects to remove
the conversion entirely later, so each gate must be easy to delete in one step.

## Where the code lives

- All changes are in the linked repo **`sase-research-artifacts`**. Open it with
  `/sase_repo` (`sase repo open sase-research-artifacts -r "<reason>"`), work only in
  the path it prints, and read that repo's `AGENTS.md` first. Nothing changes in the
  `sase` repo itself. The opened repo becomes a commit obligation in the final
  declaration.
- Macro: `src/sase_research_artifacts/xprompts/research_swarm.md`
- Tests: `tests/test_macro_loading.py`
- Docs: `docs/macros.md`, `README.md`. Do **not** hand-edit `CHANGELOG.md`; it is
  managed by release-please. Suggested commit subject:
  `feat(research-swarm): add tables_to_lists input, off by default`.
- The global `sase` CLI loads this plugin from the user's primary checkout, not from the
  opened linked checkout. `sase macro show research_swarm` therefore will **not** show
  your edit. Verify with the repo's own venv (`just install-venv` if `.venv` is missing)
  and its tests.

## Background: where the table rule lives today

Commit `c787c88` ("have the linker convert tables into PDF-friendly lists") made the
rule unconditional. Later diagram work added two phrases that depend on it. Today the
linker segment touches tables in these five places, and each one needs a gate:

1. **Step 2 (inventory):** the parenthetical in
   `table (record its cells; step 3 converts each one into a list), and link in the lead's report.`
2. **Step 3 (restructure):** the whole `- **No tables.**` bullet and its nested
   sub-bullets. It runs from "The PDF renderer (pandoc → XeLaTeX) prints tables…"
   through "…or add totals, rankings, or conclusions the table did not state."
3. **Step 6 (re-check):** the sentence "When `pandoc` is available, confirm no table
   survives:", the `pandoc … | wc -l` command, and "prints `0`."
4. **Inside `{%- if run_diagrams %}`, "Place proposed diagrams":** "…even if that
   content was reordered, merged, or converted from a table into a list."
5. **Inside `{%- if run_diagrams %}`, "Anchor every placed figure":** the sentence "The
   table rule that forbids `:::` divs is about table substitutes and does not cover this
   figure wrapper."

## Implementation

### 1. Declare the input

In the frontmatter `input:` list, add this entry directly after `linker_model`, so it
sits with the other linker options:

```yaml
- name: tables_to_lists
  type: bool
  default: false
  description:
    Have the linker convert every table in the lead's report into a list (one item per
    row, every cell kept) and confirm with pandoc that no table survives. Off by
    default, so published reports keep real Markdown tables. Takes effect only when the
    linker runs; supplying it alone never launches the linker.
```

Do **not** change `run_linker`. Like `linker_model` and `audio_edition`, this input
modifies an agent but never adds one, so the agent count and execution matrix stay the
same.

### 2. Gate the five spots

Gate each spot with a plain `{% if tables_to_lists %}…{% endif %}` arm, or with
`{% if %}…{% else %}…{% endif %}` where the off state needs its own text. Removing the
feature later should then mean deleting the `true` arms and the input, and nothing else.
Use the same `{%-` whitespace-control idiom as the existing `{%- if audio %}` and
`{%- if image %}` bullets. Step 3 must never contain a blank or whitespace-only line in
any input combination; an existing test enforces this.

1. **Step 2:** the parenthetical renders only when `tables_to_lists` is true. The off
   state reads `…code block, table, and link in the lead's report.`, which is the
   pre-`c787c88` wording.
2. **Step 3:** the `- **No tables.**` bullet renders only when `tables_to_lists` is
   true.

   > [!decision] keep_tables_rule = explicit In the `{% else %}` arm, render this one
   > bullet at the same spot:
   > `- **Keep tables as tables.** Carry every table in the lead's report into the new file as a Markdown table with all of its rows and cells. You may move a table with its section, but do not convert it into a list, prose, a diagram, or an image.`
   > Wrap it at the file's usual width with the same 5-space continuation indent as the
   > neighboring bullets.

   > [!decision] keep_tables_rule = silent Add no `{% else %}` arm. With the flag off,
   > step 3 has no table bullet at all, exactly as before `c787c88`.

3. **Step 6:** keep "Confirm every cell of every table in the lead's report appears in
   the new file." in **both** states, because it is a general no-loss check. Only the
   pandoc no-table-survives sentence, its command, and "prints `0`." render when
   `tables_to_lists` is true. The next sentence, "Every Mermaid diagram from the lead's
   report is still present.", and the `{%- if run_diagrams %}` text after it stay
   unconditional, as they are today.
4. **Diagram placement:** on, keep today's "was reordered, merged, or converted from a
   table into a list." Off, render "was reordered or merged."
5. **Figure anchor:** the "The table rule that forbids `:::` divs…" sentence renders
   only when `tables_to_lists` is true. "This wrapper and the listen card are the only
   HTML blocks you write." stays in both states.

Leave the lead, researcher, image/diagrams, and audio segments alone. Leave the
`#research*` macros and the `mermaid_diagrams` skill alone too: their table guidance
("Never turn a table into a diagram", "Prefer a list or table…") does not depend on this
rule.

### 3. Tests (`tests/test_macro_loading.py`)

- `test_research_swarm_declares_typed_input`: insert `("tables_to_lists", "bool")` after
  `("linker_model", "model")` and assert `xp.inputs[20].default is False`. Shift the
  `audio`, `audio_model`, and `audio_edition` index assertions to 21, 22, and 23.
- Replace `test_research_swarm_linker_converts_tables_to_lists` with tests for both
  states. Pass the flag through `_swarm_segments({"tables_to_lists": "true"}, ...)`.
  Cover these cases:
  - **Default (`linker=True`, flag omitted):** none of these strings appear in the
    linker segment: `step 3 converts each one into a list`, `- **No tables.**`,
    `` `- *<Column header>:* <cell>` ``, `"t":"Table"`, and `confirm no table survives`.
    `Confirm every cell of every table` is still present in step 6. Under
    `keep_tables_rule=explicit`, `- **Keep tables as tables.**` is present in step 3.
  - **`tables_to_lists=false` given explicitly:** renders the same as the default.
  - **`tables_to_lists=true`:** every assertion from today's test still holds, and the
    keep-tables bullet is absent.
  - **Diagrams on (`diagrams=True`, and `image=True`), both states:**
    `converted from a table into a list` and `The table rule that forbids` appear
    exactly when the flag is true. `reordered or merged` appears exactly when it is
    false.
  - **Step-3 hygiene:** in both states, with and without `image`/`audio`, step 3 has no
    whitespace-only line, and the segment has no stray `{%` after
    `_without_wait_artifacts_loop`.
  - **Never implies the linker:** `_swarm_segments({"tables_to_lists": "true"})` with
    the default researchers still yields the same three segments as the plain default,
    and the lead still writes `<name>.md`.

### 4. Docs

- `docs/macros.md`: add an inputs-table row after `linker_model`:
  ``| `tables_to_lists` | bool | `false` | Linker converts every table into a list for the PDF (linker runs only) |``
  (match the column padding). In the linker agent's description (item 8,
  `<clan>.linker`), add one sentence: by default it keeps the lead's tables as tables,
  and with `tables_to_lists=true` it converts each table into a list with one item per
  row. Next to the existing linker example, add
  `#research_swarm(prompt="A research topic", image=true, tables_to_lists=true)`.
- `README.md`: in the `#research_swarm` bullet, after the linker sentence, add a short
  clause: optional `tables_to_lists` (default `false`) has the linker convert tables
  into PDF-friendly lists.
- Follow `docs/` conventions already used in that repo. No `mkdocs` or sase-repo docs
  change is needed.

## Verification

1. **On state is byte-identical to today.** Before editing, use the repo venv and the
   same expansion path the tests use
   (`expand_single_macro(..., preserve_segment_separators=True)` plus
   `split_segments_protecting_fences`). Render the linker segment for `linker=true`,
   `image=true`, `image=true audio=true`, and `diagrams=true`, and save each to a temp
   file **outside** the repo. After editing, render the same four combinations with
   `tables_to_lists=true` added and `diff` them against the saved files. They must be
   identical. Then diff the default renders against the saved files and confirm that
   only the five gated spots changed. Delete the temp files and do not commit any
   script.
2. `sase tool run check` in `sase-research-artifacts` (lint + tests). Do not run bare
   `just check`; it is guarded.

## Out of scope

- No change to `bob ref create` or the pandoc → XeLaTeX PDF renderer.
- No SASE feature flag. Flags are a `sase`-project concern, and this is a per-run macro
  input in a plugin.
- No change to the user's global `#research_swarm` aliases. The new default is the
  behavior the user wants.
- Eventual removal (delete the input, the `true` arms, and the on-state tests) is a
  separate, later change.
