---
tier: tale
title: Render Tier 2 long-memory entries as numbered sections
goal:
  Managed agent instruction files render each Tier 2 long-memory note as its own
  numbered `###` section (heading = note path, body = note description) instead of a
  bold description list, while legacy-format documents still parse and every long note
  stays reachable.
size: medium
proposed_by: bbugyi200.athena.04u
create_time: 2026-08-17 11:01:37
status: wip
---

# Plan: Render Tier 2 long-memory entries as numbered sections

## Goal

In generated agent instruction files (`AGENTS.md` and its provider shims), replace the
Tier 2 description-list shape:

```markdown
**`sase/memory/cli_rules.md`** Read anytime new CLI subcommands or options are added.
```

with a numbered section per long note, mirroring how Tier 1 short notes already render:

```markdown
### 2.1 `sase/memory/cli_rules.md`

Read anytime new CLI subcommands or options are added.
```

The `2.1` prefix is not authored by the renderer. The renderer emits
`### `sase/memory/cli_rules.md``; the existing document-wide numbering pass (`sase.amd._section_numbers.number_agent_document_sections`, applied in `sase.amd._template.render_agents_template`) numbers every heading at or below the base H2 level, so Tier 2 entries pick up `2.1`, `2.2`, ... exactly the way Tier 1 sections pick up `1.1`, `1.2`,
....

## Design Decisions (Read These Before Implementing)

### 1. The heading text is the note's relative path, not the note's H1 title

Tier 1 renders `### 1.1 Build & Run Commands (build_and_run)` — an H1 title plus a
basename. Tier 2 deliberately does **not** copy that: the heading is the note's
root-relative path in a code span, `### `sase/memory/cli_rules.md``. Three reasons, in
order of importance:

1. **Reachability.** `sase memory init` validation walks plain memory-path mentions out
   of `AGENTS.md` (`sase.memory.inventory_reachability._reachable_memory_files_for_init`
   via `sase.memory.inventory_references._MEMORY_PATH_RE`) and reports every unreachable
   note as a blocker (`{root}: unreferenced memory file {path}`, raised from
   `sase.main.init_memory_handler._memory_plan_blockers`). Short notes have an explicit
   carve-out because they are inlined; top-level long notes do **not** — today they are
   reachable precisely because the Tier 2 entry prints `sase/memory/<note>.md` as
   literal text. Dropping the path from the rendering would make every top-level long
   note unreferenced and block `sase memory init` in every managed root. Keeping the
   path in the heading preserves reachability with zero changes to validation.
   `_MEMORY_PATH_RE` matches inside backticks (its lookbehind is `(?<![\w./-])` and the
   path body excludes backticks), so a code-span heading is matched exactly like the
   current bold code span.
2. **Convergence.** A title-based heading would have to come from the note body.
   `sase.amd._memory._render_managed_agents` builds `MemoryNote`s for freshly generated
   long notes with `body=""` when the file does not exist yet, so a first
   `sase memory init` would render a fallback title and the very next `--check` would
   render the real H1 title — permanent, self-inflicted drift. Path headings have no
   such dependency on note bodies.
3. **Precedent.** `sase/memory/README.md` already renders one section per note as
   `### `sase/memory/<note>.md``(see`sase.main.init_memory.root_rendering._render_memory_notes`). Tier 2 now matches that surface, and the path stays copy-pasteable into `sase
   memory read <note>.md`/`/sase_memory_read`.

### 2. `tier2_entries` keeps its template-variable name

`AGENTS.template.md` is user-overridable (`memory.agents_template` /
`amd_agents_template`, documented in `docs/configuration.md`), and
`sase.amd._template._MANAGED_TEMPLATE_VARS` requires the exact variable names. Renaming
`tier2_entries` to something like `tier2_sections` would break every existing user
override for no functional gain. **Do not rename it.** The template file itself
(`src/sase/amd/templates/AGENTS.template.md`) needs no change at all: only the string
substituted into `{{ tier2_entries }}` changes shape.

### 3. Legacy parsing stays permanently

The bold `**`sase/memory/<note>.md`**` shape must keep parsing, exactly as the legacy
`- @memory/<note>.md` Tier 1 bullet shape still parses today. It is read from
already-written documents in two places:

- `sase.amd._memory._existing_agents_long_descriptions`, which recovers a description
  from the current `AGENTS.md` for a long note whose frontmatter has none. On the first
  run after this change, every existing `AGENTS.md` on disk is still in the old shape,
  so losing legacy parsing would silently drop recovered descriptions.
- `sase.amd.inventory` (`sase memory agent-docs list`), which reports a per-document
  long reference count from the parsed entries.

Both shapes are recognized side by side; only the renderer switches.

### 4. No feature flag

Checked against `sase/memory/sase_flags.md`. This is not a disabled beta, a partially
landed path, or a deprecation whose old branch must stay user-reachable: the rendering
flips unconditionally in one change, generated output is regenerated in the same change,
and old-shape _parsing_ is permanent backward compatibility for already-written
documents, not a routed alternative branch. A flag would have nothing to gate.

### 5. Stays in Python

Per `sase/memory/rust_core_backend_boundary.md`, shared backend/domain behavior belongs
in `../sase-core`. The entire agent-document generator already lives in this repo
(`src/sase/amd/`, `src/sase/memory/notes.py`) and this change is an incremental format
change to that existing Python code. Do **not** move or duplicate the generator into the
Rust core as part of this work.

## Non-Goals

- **`## Children` sections stay as-is.**
  `sase.memory.notes.render_memory_note_references` also renders the `## Children` block
  inside canonical long notes (`render_children_section`). Those are canonical memory
  notes, not agent instruction files, and are out of scope.
  `render_memory_note_references` keeps its current shape, its current callers, and its
  current tests.
- No change to Tier 1 rendering, to `inline_memory_section`, or to the numbering pass.
- No change to the note frontmatter schema, to `sase memory read`, or to the
  audited-read workflow.
- No renaming of `tier2_entries` (see Design Decision 2).

## Implementation

### Step 1 — New renderer: `src/sase/memory/notes.py`

Add, next to `render_memory_note_references`:

```python
def render_long_memory_sections(notes: Iterable[MemoryNote]) -> str:
    """Render notes as the AGENTS.md Tier 2 numbered-section shape."""
```

- Same input contract as `render_memory_note_references`: filter to
  `note.type == "long"` and sort by `note.relative_path` so output stays deterministic.
- Each note renders as
  `### `{note.relative_path}``followed by a blank line and`note.description`; a note
  with an empty description renders the heading alone.
- Blocks are separated by exactly one blank line, matching the joining style
  `render_memory_note_references` already uses.
- Export it in `__all__` (Symvision requires public symbols to be exported and used).
- Keep `render_memory_note_references` untouched.

### Step 2 — Render Tier 2 with it: `src/sase/amd/_memory.py`

In `_render_managed_agents`, replace

```python
tier2_entries = render_memory_note_references(rendered_long_notes)
```

with the new `render_long_memory_sections(rendered_long_notes)` and update the import.
Nothing else in that function changes: the existing post-render structural check
(`parsed_long_paths != expected_long_paths`) now validates the new shape through the
updated parser and is the safety net that catches a corrupted Tier 2 block.

### Step 3 — Parse both shapes: `src/sase/amd/_agents_doc.py`

- Add a section-heading pattern alongside `_LONG_MEMORY_ENTRY_RE`, tolerating the
  numbering prefix the way `_SHORT_SECTION_RE` / `_LONG_SECTION_RE` already do:

  ```python
  _LONG_MEMORY_SECTION_RE = re.compile(
      r"^###\s+(?:\d+(?:\.\d+)*\.?\s+)?`(?P<path>(?:sase/)?memory/[A-Za-z0-9_.-]+\.md)`$"
  )
  ```

  Both the unnumbered (freshly rendered, pre-numbering) and numbered (written to disk)
  forms must parse — `sase memory init --check` compares against on-disk numbered
  documents.

- Factor the "does this line start a long-memory entry?" test into one helper, e.g.
  `long_memory_entry_path(line: str) -> str | None`, that tries the section pattern then
  the legacy bold pattern and returns the canonicalized path
  (`canonical_memory_reference(...).as_posix()`), or `None`. Export it so Step 4 can
  reuse it.
- Rework `_long_memory_entries` to use that helper for entry starts. Description
  accumulation is otherwise unchanged: consume following lines until the next entry
  start or the section end, skipping legacy AMD comments, then normalize through
  `_description_text` (which keeps stripping the legacy `_Read when ..._` suffix).
- For the section shape there is no inline description on the heading line, so the
  heading line contributes no description text; for the legacy shape the trailing text
  on the `**`...`**` line still does.
- `_short_memory_paths` needs no change: `_SHORT_MEMORY_HEADER_RE` requires a trailing
  `(name)` parenthetical that a Tier 2 path heading never has, and section bounds
  already stop at the next H2 anyway.

### Step 4 — Anchorless fallback: `src/sase/amd/_memory.py`

`_existing_agents_long_descriptions` falls back to `_AGENTS_LONG_MEMORY_RE` when a
document has no `## Tier 2 (long-term) Memory` anchor (custom templates). Replace that
single-shape regex with a line scan built on the Step 3 helper so both shapes are
recovered from anchorless documents. Preserve today's termination semantics exactly: a
description ends at the next entry start or at any `^##\s+` H2 heading, and is
normalized with `normalize_long_memory_description_lines`.

### Step 5 — Block descriptions that would corrupt the section structure

A long-note description containing its own Markdown heading would break Tier 2's section
structure (and could be misread as another entry). Add a blocker analogous to
`_short_memory_structure_blockers`:

- In `src/sase/amd/_memory.py`, add `_long_memory_description_blockers(descriptions)`
  returning one actionable message per offending note, e.g.
  `"sase/memory/foo.md: long memory note description must not contain Markdown headings"`.
- Detect headings with the existing **fence-aware** helper
  `sase.amd._headings.iter_headings`, the same primitive
  `inline_memory.validate_short_memory_structure` uses, so a `#` inside a fenced code
  block in a description stays content.
- Call it in `plan_amd_memory_sync` right after `_long_memory_descriptions(...)` and
  return an `AmdMemorySyncPlan` with those blockers, mirroring the existing
  `structure_blockers` early return.
- No current note trips this; it is a guard against silently malformed future output.

### Step 6 — Tests

Update the assertions that pin the generated shape, keep the ones that pin legacy
parsing, and add coverage for what is new. The generated-shape assertions live in these
files (grep for ``**`sase/memory/`` and ``**`memory/``):

`tests/test_memory_notes.py`, `tests/main/test_init_memory_agents_templates.py`,
`tests/main/test_memory_read_list.py`, `tests/main/test_init_onboarding_memory.py`,
`tests/main/test_init_memory_glossary.py`, `tests/main/test_init_memory_bead_note.py`,
`tests/main/test_init_memory_markdown_templates.py`,
`tests/main/test_init_memory_validation.py`, `tests/main/test_init_memory_plan.py`,
`tests/main/test_init_memory_managed_agents.py`,
`tests/main/test_init_memory_formatting.py`, plus the `managed_agents()` fixture in
`tests/main/test_memory_agent_docs_list.py`.

For each occurrence decide which of the two it is:

- **Input written to a fake `AGENTS.md`** (e.g.
  `test_init_memory_managed_agents.py:119-124`, `:293-305`, and the parser cases in
  `test_init_memory_agents_templates.py`): keep at least one of each in the legacy shape
  so legacy parsing and description recovery stay covered, and add a new-shape twin.
- **Assertion about generated output** (e.g.
  `test_init_memory_managed_agents.py:141-145`, `:275-277`): rewrite to the section
  shape.

New tests to add:

1. `render_long_memory_sections` unit tests in `tests/test_memory_notes.py`: ordering,
   multi-paragraph/bulleted description bodies, empty description, and non-`long` notes
   filtered out. Leave the existing `render_memory_note_references` children tests
   intact as the regression guard for the Non-Goal.
2. Parser tests in `tests/main/test_init_memory_agents_templates.py`: numbered and
   unnumbered section headings parse to the same paths and descriptions; a document
   mixing both shapes parses every entry; legacy-only documents are unchanged.
3. End-to-end numbering assertion (in `test_init_memory_managed_agents.py`): a generated
   managed `AGENTS.md` contains
   ``### 2.1 `sase/memory/<first>.md```— i.e. the numbering pass really does number Tier 2 entries under`##
   2.`.
4. Round-trip assertion: a generated document parses back to exactly the expected long
   paths **and** descriptions, including a multi-line block description (extend the
   existing `block.md` case rather than writing a new fixture).
5. **Reachability regression guard** (the highest-value new test): a managed root whose
   only long note is top-level produces no `unreferenced memory file` blocker, and
   `sase.memory.inventory_reachability.unreferenced_memory_files_for_init` returns empty
   for it. This is what protects Design Decision 1 from a future "let's use the H1
   title" refactor.
6. Blocker test for Step 5: a long note whose description contains `## Heading` yields
   the new blocker, and one whose description contains a fenced block with a `#` comment
   line does not.

### Step 7 — Documentation

Update the prose that describes the old shape:

- `docs/memory.md` (~line 17): "generated Tier 2 entries render that block verbatim" →
  say the block renders verbatim as the body of the note's numbered Tier 2 section.
- `docs/init.md` (~line 214): "renders Tier 2 from long-note descriptions" → say Tier 2
  renders one numbered section per long note, headed by the note path, with the
  description as the body.
- `docs/configuration.md` (~line 3908): "long-term notes are rendered as a
  description-driven reference list" → describe the numbered-section shape. Leave the
  `tier2_entries` row of the template-variable table (~line 485) named as-is; if it
  describes the value's shape, adjust the wording only.
- `src/sase/main/init_memory/templates/memory-README.template.md` (~line 26): "Tier 2
  entries render those blocks verbatim" → "Tier 2 sections render those blocks
  verbatim". This template regenerates `sase/memory/README.md`, which is covered by
  Step 8.

Do **not** hand-edit `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, or
`sase/memory/README.md`; they are generated in Step 8. No canonical note under
`sase/memory/` needs a content edit for this change.

### Step 8 — Regenerate managed documents

Run the workspace-pinned build (a bare `sase` on `PATH` can resolve to a different
checkout with different generator templates — see `sase.main.init_memory.staleness`):

```bash
.venv/bin/sase memory init
```

This regenerates the project `AGENTS.md`, the four provider shims, and
`sase/memory/README.md`. Expect it to also refresh the **home / chezmoi-home root**:
`sase.main.init_memory_handler._memory_root_plans` always plans the home root, and with
`use_chezmoi: true` the home files are written into the chezmoi source tree and deployed
(`--no-commit` skips only the project-side git path, not home deployment). That is
expected and required — otherwise `just check` reports drift for the home root — but
call it out explicitly in the completion report, since it changes files in the chezmoi
repo.

Sanity-check the result before moving on: the project `AGENTS.md` Tier 2 block should
read `### 2.1 `sase/memory/cli_rules.md`` … `### 2.8 `sase/memory/xprompts.md`` (eight
top-level long notes, alphabetical by path), the glossary entry's bulleted description
should survive unchanged as that section's body, and the four provider shims should be
byte-identical copies of `AGENTS.md`.

## Verification

1. `just install` first — this workspace is ephemeral and its virtualenv may be stale.
2. `.venv/bin/sase memory init --check` must report no drift after Step 8.
3. `just check` inline (whole-repo lint gates + diff-scoped tests). If it runs long,
   hand it to `/sase_monitor` with a `--next` action instead of blocking.
4. `just check-full` through `/sase_monitor` before landing —
   `sase monitor start --command 'just check-full' --next '<follow-up action>'`. It is
   required here rather than optional: this change touches the memory generator,
   packaged templates, and docs (the broadening set), and it must never run inline.
5. Confirm `sase memory agent-docs list` still reports the project root as `managed`
   with a nonzero long-memory count (that number now comes from the new parser path).

## Risks And Mitigations

- **Every managed root drifts until regenerated.** Any `AGENTS.md` generated by an older
  build stays in the old shape and will be rewritten on its next `sase memory init`.
  That is the intended migration and needs no compatibility window, because old-shape
  parsing is retained (Design Decision 3). Other SASE projects regenerate on their own
  next init.
- **Reachability regression.** Mitigated by keeping the path in the heading plus the
  explicit test in Step 6.5. If a reviewer proposes a title-based heading, point them at
  Design Decision 1.
- **Description content colliding with the section grammar.** Mitigated by the Step 5
  blocker and by the existing post-render path round-trip check in
  `_render_managed_agents`.
- **A user template override that reformats `{{ tier2_entries }}`** (for example, one
  that indents it or wraps it in a list) would now be injecting headings. This is
  detected, not silently broken: the post-render structural check fails with an
  actionable "unexpected Tier 2 memory paths" blocker. No code change needed; just do
  not rename the variable.
