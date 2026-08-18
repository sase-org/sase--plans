---
tier: tale
title: Split Tier 2 into "Long-Term Memory Files" and "Glossary Terms" H3 sections
goal:
  Generated agent instruction files render Tier 2 as two numbered H3 sections — a
  long-term memory files section whose notes are H4 subsections, and a Glossary Terms
  section that lists every configured glossary term as a bullet — replacing the
  `**GLOSSARY TERMS:**` paragraph.
size: medium
proposed_by: bbugyi200.athena.05o
create_time: 2026-08-18 06:51:26
status: wip
---

# Plan: Tier 2 gets "Long-Term Memory Files" and "Glossary Terms" H3 sections

## Goal

`sase memory init` currently renders the Tier 2 section of `AGENTS.md` (and the provider
shims copied from it) as a bare `**GLOSSARY TERMS:**` paragraph followed by one H3
section per long-term memory note. Replace that with two H3 sections under the Tier 2
H2:

1. **Long-Term Memory Files** — one H4 subsection per top-level long note (the note path
   as the heading, its description as the body).
2. **Glossary Terms** — the `sase glossary read` instruction paragraph, followed by an
   unordered bullet list with one bullet per configured glossary term.

Hierarchical numbering must stay correct and fully automatic.

## Target Output

For a managed project with long notes and a configured glossary, the generated Tier 2
section must read exactly like this (numbering supplied by the existing numbering pass):

```markdown
## 2. Tier 2 (long-term) Memory

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

### 2.1 Long-Term Memory Files

#### 2.1.1 `sase/memory/cli_rules.md`

Read anytime new CLI subcommands or options are added.

#### 2.1.2 `sase/memory/sase_beads.md`

Read before creating, updating, closing, or querying sase beads.

### 2.2 Glossary Terms

Run `sase glossary read <term> -r "<why>"` before relying on any of these SASE terms; it
prints that term's definition plus every term the definition depends on. Aliases follow
in parentheses.

- Agent Clan
- Agent Hood (hood, agent neighborhood)
- Stitch
```

Notes on the target shape:

- The H2 intro paragraph ("The below files contain detailed reference material…") stays
  where it is, in the packaged `AGENTS.template.md`, above both H3 sections. It reads
  correctly because the memory-files section comes first.
- The `**GLOSSARY TERMS:**` bold label is removed; the H3 heading replaces it. The
  instruction sentence is preserved verbatim, but the trailing
  `Terms (aliases follow in parentheses): A; B; C` clause becomes the sentence "Aliases
  follow in parentheses." plus the bullet list.
- Each bullet keeps the existing entry rendering: `Term` when there are no displayed
  aliases, `Term (alias1, alias2)` when there are. Terms and aliases stay `md_escape`d.

**Assumption to confirm at approval:** the memory-files H3 is titled
`Long-Term Memory Files` and is ordered before `Glossary Terms`. If a different title or
ordering is wanted, it is a one-constant change in `src/sase/amd/_memory.py` (plus the
test/doc strings listed below).

## Ordering And Empty-Section Rules

- Long notes exist, glossary configured → `Long-Term Memory Files` then `Glossary Terms`
  (numbered `2.1` and `2.2`).
- Long notes exist, no glossary (the common case: every home root, and any project with
  no `memory.glossary`) → only `Long-Term Memory Files` (numbered `2.1`).
- No long notes, glossary configured → omit the empty `Long-Term Memory Files` H3
  entirely; `Glossary Terms` becomes `2.1`.
- Neither → `tier2_entries` renders empty, exactly as today.

## Files And Changes

### 1. `src/sase/memory/notes.py` — render note sections at H4

`render_long_memory_sections()` (around line 433) emits
`### \`{note.relative_path}\``. Change it to `#### \`{note.relative_path}\`` and update
the docstring to say the shape is a Tier 2 H4 subsection.

Verify before editing that this function still has exactly one caller —
`src/sase/amd/_memory.py`. In particular, do **not** touch `_render_memory_notes()` in
`src/sase/main/init_memory/root_rendering.py` (around line 451), which renders the
similar-looking
`### \`sase/memory/<note>.md\``sections in the generated`sase/memory/README.md`. The README keeps H3; only the AGENTS.md Tier 2 shape changes. `_render_memory_note_references()`(the`**\`path\`**`shape used by`render_children_section`)
is also unrelated and unchanged.

### 2. `src/sase/amd/_memory.py` — compose the two H3 sections

- Add module constants for the two headings, e.g.
  `_LONG_MEMORY_FILES_HEADING = "### Long-Term Memory Files"` and
  `_GLOSSARY_TERMS_HEADING = "### Glossary Terms"`.
- Rename `_render_glossary_terms_block()` (line ~287) to
  `_render_glossary_terms_section()` and change its body to return the H3 heading, then
  the instruction paragraph, then one `- ` bullet per term. Keep
  `_render_glossary_term_entry()` as the per-bullet renderer (it already handles the
  no-alias case and `md_escape`). Return `""` when there are no terms.
- Add `_render_long_memory_files_section(entries: str) -> str` that returns `""` for
  empty `entries` and `f"{_LONG_MEMORY_FILES_HEADING}\n\n{entries}"` otherwise.
- In `_render_managed_agents()` (the `tier2_entries = render_long_memory_sections(...)`
  block around line 362), build `tier2_entries` by joining the non-empty sections with a
  blank line, memory files first, glossary second. Delete the current "prepend glossary
  block" logic.
- Update the `plan_amd_memory_sync()` docstring (line ~456), which still describes a
  `**GLOSSARY TERMS:**` paragraph "at the top of Tier 2".
- Fix the legacy-document scan in `_existing_agents_long_descriptions()` (lines ~63-85).
  It currently accumulates description lines until the next entry or the next `##` line
  (`_H2_LINE_RE`). Make it stop at any heading (fence-aware, see item 3) so a trailing
  `### Glossary Terms` section can never be absorbed into the last note's description.

### 3. `src/sase/amd/_agents_doc.py` — parse H4 entries and bound descriptions

- `_LONG_MEMORY_SECTION_RE` (line ~32) matches `^###\s+…`. Widen it to `^#{3,4}\s+…` so
  the new H4 entries parse **and** documents already committed with H3 entries keep
  parsing. Backward compatibility here is what keeps the existing
  `long_memory_entry_path` / mixed-shape tests green and lets an already-generated
  `AGENTS.md` be re-read on the first init after this change.
- `_collect_long_memory_entries()` (line ~160) appends every non-entry line to the
  current entry's description until the section bound. With the glossary H3 now rendered
  **after** the note entries, the last note's description would swallow
  `### Glossary Terms`, its paragraph, and its bullets — and
  `_long_memory_description_updates()` would then write that polluted text into the
  note's `description:` frontmatter on disk. Add a heading boundary: stop collecting
  when a line is a heading that does not start a long-memory entry.
- That boundary must be **fence-aware**. Long-note descriptions may legally contain
  fenced code blocks (`_long_memory_description_blockers()` uses the fence-aware
  `iter_headings()`, so a `# comment` line inside a fence is allowed). Track fence state
  with `fence_marker()` / `heading_level()` from `src/sase/amd/_headings.py` while
  scanning, so a `#` line inside a fenced block does not truncate a description.

### 4. Structural self-check (optional but recommended)

`_render_managed_agents()` already re-parses its own rendered output and returns a
blocker when Tier 1/Tier 2 anchors or memory paths are wrong. Extend that check: when
long notes exist, the rendered document must contain the `Long-Term Memory Files`
heading; when glossary terms exist, it must contain the `Glossary Terms` heading. This
catches a project template override that drops `{{ tier2_entries }}`.

### 5. Do not change the template variable contract

Keep `_MANAGED_TEMPLATE_VARS` in `src/sase/amd/_template.py` as
`{title, tier1_sections, tier2_entries}` and keep
`src/sase/amd/templates/AGENTS.template.md` as-is. Both H3 sections are composed into
the existing `tier2_entries` value. Adding new required template variables would
invalidate every project's `amd_agents_template` override, which is why the composition
belongs in `_memory.py`.

Numbering needs no change: `number_agent_document_sections()` numbers by heading depth
relative to the shallowest heading, so the new H4 entries become `2.1.1`, `2.1.2`, … for
free. Add a regression test rather than touching `src/sase/amd/_section_numbers.py`.

## Tests

Update existing expectations (all assert the old H3 entry shape or the old glossary
paragraph):

- `tests/test_memory_notes.py` (lines ~236-296) — `render_long_memory_sections` now
  emits `#### \`sase/memory/<note>.md\``.
- `tests/main/test_init_memory_glossary.py` — every `**GLOSSARY TERMS:**` and
  `Terms (aliases follow in parentheses): …` assertion becomes an H3 + bullet assertion;
  keep the alias-suppression cases (`Patch` with only the derivable plural `patches`
  renders as a bare `- Patch` bullet), the idempotence/`--check` case, and the provider
  shim byte-equality case.
- `tests/main/test_init_memory_markdown_templates.py` — the
  `test_glossary_terms_block_precedes_discovered_long_notes` test (line ~245) now
  asserts the reverse order: the `Long-Term Memory Files` heading precedes the
  `Glossary Terms` heading, and both precede/contain what they should. Rename it
  accordingly.
- `tests/main/test_init_memory_managed_agents.py` (lines ~145, ~277, ~471, ~554),
  `tests/main/test_init_memory_plan.py` (line ~43),
  `tests/main/test_init_memory_validation.py` (line ~147),
  `tests/main/test_memory_agent_docs_list.py` (line ~71) — expected AGENTS.md strings
  become
  `#### 2.1.1 \`sase/memory/<note>.md\``. Leave the `sase/memory/README.md`assertions in`tests/main/test_init_memory_plan.py`(line ~58),`tests/main/test_init_memory_bead_note.py`, and `tests/main/test_init_memory_handler_outputs.py`
  at H3 — the README shape is unchanged.
- Parser tests in `tests/main/test_init_memory_agents_templates.py` (lines ~66-130) keep
  their legacy H3 cases and gain H4 cases.

Add:

- Full generated Tier 2 shape for a managed project with long notes and a glossary:
  `### 2.1 Long-Term Memory Files`, `#### 2.1.1 \`…\``, `### 2.2 Glossary
  Terms`, and one `- Term (aliases)` bullet per term, in catalog order.
- Glossary configured with **no** long notes → no `Long-Term Memory Files` heading and
  `### 2.1 Glossary Terms`.
- No glossary → no `Glossary Terms` heading and no leftover `GLOSSARY TERMS` text.
- Prettier fixpoint: `format_generated_memory_markdown(agents) == agents` for a glossary
  whose term + aliases bullet exceeds the configured print width (must wrap to a `- `
  item with two-space continuation indent).
- Idempotence: run init, run again, assert `plan_memory().actions == ()` and
  `run_memory(check=True) == 0` for a project that has both a glossary and long notes.
- Description-boundary regression: parse an AGENTS.md whose Tier 2 ends with a
  `### Glossary Terms` section and assert the final note's parsed description stops
  before it, plus a matching `_existing_agents_long_descriptions()` case. Then a
  round-trip case: a long note with no `description:` frontmatter, an existing generated
  AGENTS.md on disk, and a re-init — the note's written frontmatter must not contain
  glossary text.
- Fence regression: a long note whose description contains a fenced block with a `#`
  comment line survives parsing intact.
- Numbering regression covering `2.1` / `2.1.1` / `2.2` in one generated document.

## Docs

- `docs/init.md` (lines ~205-220) — "prepends a `**GLOSSARY TERMS:**` block to the Tier
  2 section" and "renders Tier 2 as one numbered section per long note" both need
  rewriting for the two-H3 shape.
- `docs/memory.md` (lines ~118-121) — the `## Glossary` section's description of the
  rendered block.
- `docs/configuration.md` (lines ~530-545 under `memory.glossary`, and line ~4118 under
  the `sase memory init` command) — same two descriptions.
- `src/sase/main/init_memory/templates/memory-README.template.md` (line ~26) says "Tier
  2 sections render those blocks verbatim" — still true; verify wording rather than
  assuming it needs an edit.

## Regeneration

The generated instruction files committed in this repo must be refreshed in the same
change, exactly as the precedent commits `538dec9fc` and `eaafcbe72` did:

```bash
sase memory init --check   # expect drift
sase memory init
```

That rewrites `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and
possibly `sase/memory/README.md`. Review the diff: the only changes should be the Tier 2
restructuring. No `sase/memory/*.md` note body should change — if a note's frontmatter
`description:` changes, that is the description-boundary bug from item 3 leaking, not an
expected edit.

Note the side effect: `sase memory init` also refreshes the managed **home** root's
instruction files (and their chezmoi source counterparts when `use_chezmoi: true`). That
is expected — the home Tier 2 gets the same new shape — but it lands outside this repo,
so call it out in the final report rather than silently leaving it.

This regeneration is authorized: the user requested this generator change directly, and
`CLAUDE.md`/`AGENTS.md` here are generated artifacts of the code being changed. No
canonical `sase/memory/*.md` note is hand-edited by this plan.

## Verification

```bash
just install
just check
```

Then, before landing, hand `just check-full` to `/sase_monitor` with a `--next` action —
it routinely outruns a single agent turn and must never be run inline. Prettier
(`just fmt-md-check`) covers every generated Markdown file, so the regenerated shims
must be a prettier fixpoint.

Targeted lane while iterating:

```bash
just test tests/main/test_init_memory_glossary.py \
  tests/main/test_init_memory_markdown_templates.py \
  tests/main/test_init_memory_managed_agents.py \
  tests/main/test_init_memory_agents_templates.py \
  tests/main/test_init_memory_plan.py \
  tests/main/test_memory_agent_docs_list.py \
  tests/test_memory_notes.py
```

## Acceptance Criteria

- Generated Tier 2 contains `### 2.1 Long-Term Memory Files` with `#### 2.1.N` note
  subsections, and `### 2.2 Glossary Terms` with one bullet per configured term.
- `**GLOSSARY TERMS:**` no longer appears in any generated file or in the repo.
- Empty-section rules above hold; a root with no glossary renders no glossary heading.
- `parse_amd_agents_document()` parses both the new H4 entries and legacy H3 entries,
  and never absorbs the glossary section into a note description.
- `sase memory init` is idempotent and `--check` clean on a second pass; generated files
  are a `format_generated_memory_markdown` and prettier fixpoint.
- Committed `AGENTS.md` + four provider shims regenerated; docs updated; `just check`
  green and `just check-full` green via monitor.

## Risks

- **Description pollution (highest).** Putting the glossary after the note entries makes
  the parser's unbounded description collection actively harmful, because parsed
  descriptions are written back into note frontmatter. Item 3's heading boundary plus
  the round-trip test is the mitigation; do not skip it.
- **Legacy parse regressions.** Any already-generated `AGENTS.md` in this or another
  repo still uses H3 entries and must keep parsing; that is why the regex accepts
  `#{3,4}` rather than moving to H4 only.
- **Template overrides.** Projects may override `AGENTS.template.md`; composing inside
  `tier2_entries` keeps those overrides working untouched.
- **Downstream repos.** Plugin repos (`sase-github`, `sase-telegram`, `sase-nvim`, …)
  pick up the new shape on their own next `sase memory init` run. Out of scope here; no
  action needed, since the parser accepts both shapes.
