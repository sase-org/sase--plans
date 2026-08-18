---
tier: tale
title: Move the Tier 2 reference-material paragraph under the Long-Term Memory Files H3
goal:
  The generated `AGENTS.md` (and its provider shims) renders the "The below files
  contain detailed reference material..." instruction paragraph directly under the
  `Long-Term Memory Files` H3 instead of directly under the `Tier 2 (long-term) Memory`
  H2, and it disappears entirely when a root has no top-level long notes.
size: medium
proposed_by: bbugyi200.athena.05q
create_time: 2026-08-18 07:24:30
status: wip
---

# Plan: Move the Tier 2 reference-material paragraph under the `Long-Term Memory Files` H3

## Goal

`sase memory init` (aka `sase init memory`) currently emits this paragraph as the first
body content of the `## Tier 2 (long-term) Memory` section:

```
The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.
```

It must instead be the first body content of the `### Long-Term Memory Files` H3 that
lives inside Tier 2, i.e.:

```
## 2. Tier 2 (long-term) Memory

### 2.1 Long-Term Memory Files

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

#### 2.1.1 `sase/memory/cli_rules.md`
...

### 2.2 Glossary Terms
...
```

The paragraph's wording is unchanged. Only its position moves.

## Background

Commit `445afde7c` split Tier 2 into two H3 sections, `Long-Term Memory Files` and
`Glossary Terms`, both rendered from `src/sase/amd/_memory.py`. The glossary H3 already
owns its own intro paragraph (`_GLOSSARY_TERMS_INTRO`), but the long-memory intro was
left behind in the packaged Jinja template
(`src/sase/amd/templates/AGENTS.template.md`), where it sits above both H3s and applies
to the glossary section too. This change finishes that split: the long-memory intro
becomes a sibling of `_GLOSSARY_TERMS_INTRO`, owned by the section it describes.

Useful facts confirmed while planning:

- The packaged template is the only template in play. No template override
  (`memory.agents_template` / legacy `amd_agents_template`) is configured in this repo,
  in `~/.config/sase/`, or in the chezmoi repo; `src/sase/default_config.yml` sets both
  override keys to `null`.
- Rendered agent documents are re-wrapped by
  `sase.main.init_memory.formatting.format_generated_memory_markdown` at
  `markdown_print_width()` (88) before they are written, so a single-line constant
  reproduces today's exact three-line wrapping byte for byte. This was verified by
  running the formatter over the proposed constant; the resulting lines are identical to
  the current `AGENTS.md` lines.
- `sase.amd._agents_doc` parses Tier 2 entries by scanning for entry lines
  (`#### N.N.N \`sase/memory/x.md\``or the legacy`**\`...\`**`form) and stops a description at the next entry or heading. Prose that sits between the`###
  Long-Term Memory
  Files`heading and the first entry is skipped, so the move does not affect`_existing_agents_long_descriptions`, `parse_amd_agents_document`,
  or the "unexpected Tier 2 memory paths" guard.
- `just check` runs `just validate` → `sase init memory --check`, so the generated files
  in this repo (and the home root) must be regenerated in the same change or
  verification fails.

## Implementation

Work in the workspace checkout. Run `just install` first — workspace venvs are ephemeral
and may be stale.

### 1. Add the intro constant (`src/sase/amd/_memory.py`)

Next to the existing Tier 2 constants near the top of the module (after
`_GLOSSARY_TERMS_INTRO`), add:

```python
_LONG_MEMORY_FILES_INTRO = (
    "The below files contain detailed reference material. When working in "
    "their domain, you MUST use your `/sase_memory_read` skill to review "
    "their contents. Do not read canonical memory files directly."
)
```

Keep it a single unwrapped logical line, exactly like `_GLOSSARY_TERMS_INTRO`; the
generated-markdown formatter does the wrapping. There is no `keep-sorted` directive over
that constant block, so ordering is free — place it beside the constant it mirrors.

### 2. Render it inside the H3 (`src/sase/amd/_memory.py`)

Change `_render_long_memory_files_section` to mirror `_render_glossary_terms_section`:

```python
def _render_long_memory_files_section(entries: str) -> str:
    """Render the Tier 2 ``Long-Term Memory Files`` H3 section."""
    if not entries:
        return ""
    return f"{_LONG_MEMORY_FILES_HEADING}\n\n{_LONG_MEMORY_FILES_INTRO}\n\n{entries}"
```

The existing `if not entries: return ""` guard is what makes the paragraph disappear
when a root has no top-level long notes — that is intended new behavior, not a side
effect to work around.

### 3. Remove the paragraph from the packaged template

In `src/sase/amd/templates/AGENTS.template.md`, delete the three prose lines under
`## Tier 2 (long-term) Memory` so the section becomes:

```
## Tier 2 (long-term) Memory

{{ tier2_entries }}
```

Leave every other line of the template untouched; `## Tier 2 (long-term) Memory` is a
required structural anchor and `{{ tier2_entries }}` is a required Jinja variable
(`_MANAGED_TEMPLATE_VARS` in `src/sase/amd/_template.py`).

Do not touch `AGENTS.minimal.template.md` — the minimal template has no Tier 2 section.

### 4. Tests

No existing test asserts the paragraph inside a rendered `AGENTS.md`, so the test work
is additive. The three existing tests that assert this prose
(`tests/test_memory_notes.py`, `tests/main/test_memory_read_list.py`,
`tests/main/test_init_memory_markdown_templates.py:~205`) all cover
`render_children_section` from `src/sase/memory/notes.py`, which is a different surface
(`## Children` in `sase memory read` output) and MUST NOT change.

Watch the two text forms:

- `plan_amd_memory_sync(...).agents_content` is pre-formatter, so the intro appears as
  one long single line there.
- A written `AGENTS.md` (handler-level tests) is post-formatter, so the intro appears as
  the three wrapped lines quoted at the top of this plan.

Add coverage in `tests/main/test_init_memory_markdown_templates.py` (plan-level) and/or
`tests/main/test_init_memory_glossary.py` (handler-level, which already has
`_tier1_memory` /`_tier2_memory` helpers):

1. **Intro renders under the H3.** With at least one top-level long note, assert the
   rendered Tier 2 body has the intro after `### 2.1 Long-Term Memory Files` and before
   the first
   `#### 2.1.1 \`sase/memory/...\``entry, and that nothing but blank space sits between`## 2.
   Tier 2 (long-term) Memory`and`### 2.1 Long-Term Memory
   Files`. An index-ordering assertion in the style of the existing `test_long_memory_files_section_precedes_glossary_terms`
   is the cheapest way to say this.
2. **Intro is absent with no long notes.** Extend
   `test_glossary_terms_block_is_sole_tier2_content_without_other_notes` to assert
   `"The below files contain detailed reference material"` is not in the rendered
   content — that test already proves `Long-Term Memory Files` is absent, and the intro
   must now go with it.
3. **Intro does not leak into descriptions.** Assert
   `parse_amd_agents_document(rendered).long_memory_entries` still yields the expected
   `(path, description)` pairs with the intro in place (the existing test already checks
   `description == "Parent."`; keeping that assertion in the new/updated test is
   enough).

Also confirm `render_children_section`'s expected string in `tests/test_memory_notes.py`
and `tests/main/test_memory_read_list.py` still passes unchanged.

### 5. Docs

Two small, factual updates:

- `docs/init.md` (~line 219, the "For a SASE-managed project, `sase memory init` inlines
  each short-term note into Tier 1..." sentence): say that Tier 2's
  `Long-Term Memory Files` H3 carries the instruction paragraph pointing agents at
  `/sase_memory_read`, and that the H3 (with its paragraph) is omitted when a root has
  no top-level long notes.
- `docs/configuration.md` "generated templates" section (~lines 486-500, after the
  required-Jinja-variables table): note that `{{ tier2_entries }}` renders the entire
  Tier 2 body — both H3 sections and their instruction paragraphs — so a custom
  `agents_template` must not repeat that prose.

### 6. Regenerate the derived instruction files

The user's request authorizes this regeneration; do not hand-edit any generated file.

```bash
just install                 # if not already done
.venv/bin/sase memory init   # add --no-commit if the surrounding commit workflow prefers one commit
```

Expect `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md` at the repo
root to move the paragraph from under `## 2. Tier 2 (long-term) Memory` to under
`### 2.1 Long-Term Memory Files`, with no other diff.

**Expected side effect:** this machine has `use_chezmoi: true`, so `sase memory init`
also refreshes the home root's generated instruction files through the chezmoi source
tree (`home/AGENTS.md`, `home/CLAUDE.md`, `home/GEMINI.md`, `home/OPENCODE.md`,
`home/QWEN.md`) and can `chezmoi apply` them to `/home/bryan/*.md`. That is the intended
outcome — the home instructions get the same layout — and it is the tool's own write
path, not a manual cross-repo edit. If those chezmoi-side changes need a commit, open
the chezmoi repo with `/sase_repo` first.

## Verification

```bash
just install
just check
```

`just check` covers the gates this change can break: `fmt (markdown)` (prettier over the
regenerated `AGENTS.md` and shims), `SASE validation` (`sase init memory --check` drift
for both the project and home roots), and the scoped test lane. If the scoped selection
looks unusual or escalates, run `just check-full` through `/sase_monitor` — never
inline.

Targeted runs while iterating:

```bash
.venv/bin/python -m pytest tests/main/test_init_memory_markdown_templates.py \
  tests/main/test_init_memory_glossary.py tests/main/test_init_memory_managed_agents.py \
  tests/main/test_init_memory_agents_templates.py tests/test_memory_notes.py \
  tests/main/test_memory_read_list.py
```

Manual confirmation on the regenerated file:

```bash
sed -n '/^## 2\. Tier 2/,/^#### 2\.1\.1/p' AGENTS.md
```

The `## 2. Tier 2 (long-term) Memory` heading must be followed by a blank line and then
`### 2.1 Long-Term Memory Files`, with the paragraph below that heading.

## Design decisions

- **Keep the prose duplicated rather than sharing a constant with
  `sase.memory.notes.render_children_section`.** That function emits the same sentences
  for a note's `## Children` section, but it writes pre-wrapped text straight to stdout
  for `sase memory read` and its exact line breaks are asserted in two tests, while the
  AGENTS.md path is re-wrapped by the generated-markdown formatter. The two surfaces
  have different wrapping contracts, and mirroring `_GLOSSARY_TERMS_INTRO` keeps
  `_memory.py` internally consistent. The prose lives in exactly two places before and
  after this change, so nothing gets worse. Do not "helpfully" refactor them into one
  constant.
- **The Tier 2 H2 keeps no body of its own.** With no long notes and no glossary terms,
  Tier 2 renders as a bare heading. `## Tier 2 (long-term) Memory` is a required
  structural anchor, and every SASE-managed project generates
  `sase/memory/sase_beads.md` as a long note, so the empty case is effectively
  unreachable in practice.

## Out of scope

- `render_children_section` in `src/sase/memory/notes.py` and the `## Children` block in
  `sase memory read` output. Same words, different surface, unchanged.
- **Pre-existing docs drift discovered while planning (do not fix here):**
  `docs/init.md:207`, `docs/memory.md:119-120`, `docs/configuration.md:536`, and
  `docs/configuration.md:4118` still describe the glossary as a `**GLOSSARY TERMS:**`
  block/paragraph, which commit `445afde7c` replaced with the `Glossary Terms` H3. That
  is unrelated to this move and belongs in its own change.
