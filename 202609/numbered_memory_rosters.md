---
tier: tale
title: Render memory-web list rosters and Reference Memory as numbered lists
goal:
  Memory webs that declare a list roster render it as an ordered list, and the generated
  Reference Memory section renders one ordered-list entry per top-level reference note
  instead of one H3 subsection each.
size: medium
proposed_by: bbugyi200.apollo.4
---

- **AGENTS:**
  - [bbugyi200.apollo.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.apollo.4.md)
- **COMMITS:**
  - [8b0c654](https://github.com/sase-org/sase/commit/8b0c65476b9cb9326221122d87a221f2b495d4d4)
    — feat(memory): render memory rosters as ordered lists

# Numbered Memory Rosters And A Numbered Reference Memory Section

## Goal

Two rendering changes to generated agent instruction files, both moving toward the same
shape — a compact, numbered list:

1. A memory web whose descriptor declares `roster: list` renders its managed strand
   roster as an **ordered** list (`1.`, `2.`, `3.`, …) instead of a `-` bullet list.
   This affects `sase/memory/decisions.md` and `sase/memory/task_types.md` in this repo.
2. The `## Reference Memory` section of managed agent instruction files renders **one
   ordered-list entry per top-level reference note** instead of one numbered H3
   subsection per note. This makes Reference Memory look like a web roster and cuts
   roughly 20 always-loaded lines out of every generated agent instruction file in this
   repo (about 50 lines today, about 27 after).

### Before / after (roster)

Before:

```
- **Bug** (`bug`) - A defect an agent found while doing unrelated work, not an external
  tracker bug.
- **CI failure** (`ci`) - A confirmed true test or lint failure you did not cause, not a
  flake.
```

After:

```
1. **Bug** (`bug`) - A defect an agent found while doing unrelated work, not an external
   tracker bug.
2. **CI failure** (`ci`) - A confirmed true test or lint failure you did not cause, not
   a flake.
```

### Before / after (Reference Memory)

Before:

```
## 2. Reference Memory

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

### 2.1 `sase/memory/cli_rules.md`

Read anytime new CLI subcommands or options are added.

### 2.2 `sase/memory/generated_skills.md`

Read when working with sase agent skills (aka xprompt skills), which are generated from
source templates in the `src/sase/xprompts/skills/` and deployed to managed locations
(my chezmoi repo, for example).
```

After:

```
## 2. Reference Memory

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

1. **`sase/memory/cli_rules.md`** - Read anytime new CLI subcommands or options are
   added.
2. **`sase/memory/generated_skills.md`** - Read when working with sase agent skills (aka
   xprompt skills), which are generated from source templates in the
   `src/sase/xprompts/skills/` and deployed to managed locations (my chezmoi repo, for
   example).
```

## Decisions Already Made (do not re-litigate)

These were settled while planning; implement them as written.

- **Numbers increment (`1.`, `2.`, `3.`), they do not all read `1.`.** Verified against
  the repo's pinned Prettier 3.8.1 at `printWidth: 88`, `proseWrap: always`:
  incrementing numbers round-trip unchanged.
- **Continuation lines are indented to the marker width — three spaces for items 1–9,
  four spaces from item 10 on.** Prettier rewrites a two-space continuation indent to
  three, so two spaces would fight `just fmt-md-check`. Nothing new is needed to get
  this right: `sase.markdown_wrap.wrap_markdown` already derives the continuation prefix
  from the full list marker (`_LIST_ITEM_RE` matches `\d+[.)]`), so an ordered-list line
  passed through `wrap_markdown(line, width=markdown_print_width())` produces exactly
  what Prettier produces. A 10+ entry list was verified end to end.
- **Reference Memory entries use the same shape as a web roster entry:**
  ``N. **`<memory-relative-path>`** - <description>``, and
  ``N. **`<memory-relative-path>`**`` when the note has no description. Separator is `-`
  (hyphen), matching `render_strand_roster`.
- **Reference-note descriptions are collapsed to a single paragraph for this surface,
  and a multi-block description becomes a `sase memory init` blocker.** A list item that
  contains a blank line makes the whole ordered list _loose_, and Prettier then inserts
  blank lines between every item and eats the blank line before a nested list — a
  format-check fight that cannot be won by emitting the tight form. Rather than silently
  mangling a structured description into `Lead paragraph. - One - Two Trailer.`,
  `sase memory init` fails closed with an actionable message. This intentionally retires
  the documented "reference sections render literal block scalars verbatim" behavior
  _for the Reference Memory section only_; see the scope note below.
- **The `## Children` section is unchanged.** `render_children_section` /
  `_render_memory_note_references` in `src/sase/memory/notes.py` render the children of
  a reference note on `sase memory read` / `sase memory show`, not in agent instruction
  files. They keep their current ``**`path`**`` + verbatim-description shape, and they
  keep rendering block descriptions verbatim. Do not change them.
- **The inline roster style is unchanged.** `roster: inline` (the `glossary` web) still
  renders its single semicolon-separated `**GLOSSARY TERMS:** …` line.
- **No feature flag.** There is no user-selectable branch and no old branch that must
  stay reachable: `sase memory init` regenerates the whole document, and the parser
  simply keeps accepting the older on-disk shapes the way it already accepts the legacy
  `- @memory/<file>.md` bullets and ``**`memory/<file>.md`**`` entries. Consult
  `sase/memory/sase_flags.md` only if you conclude otherwise.
- **No `sase-core` (Rust) work.** Memory-note parsing, roster rendering, and agent
  document generation are Python-only in this repo; the Rust core is involved in memory
  only through `sase.core.glossary_facade` validation, which this change does not touch.
- **No new decision record.** This is a rendering-format change, not an architectural
  choice with rejected alternatives worth preserving.

## Implementation

### 1. Ordered-list strand rosters

`src/sase/memory/roster.py` does not exist; the file is `src/sase/memory/web/roster.py`.

In `render_strand_roster`, the `web.roster == "inline"` branch stays exactly as it is.
In the list branch, number the entries:

- Iterate `enumerate(strands, start=1)`.
- Build ``f"{number}. **{strand.keyword}** (`{strand.slug}`) - {summary}"`` (and the
  supersession-marker variant
  ``f"{number}. **{strand.keyword}** (`{strand.slug}`) - {marker} {summary}"``), keeping
  the existing `.rstrip()`.
- Keep passing each line through `wrap_markdown(bullet, width=width)` with `width`
  hoisted out of the loop as it already is.

Nothing reads the rendered roster back: `render_web_body_with_roster` replaces the whole
region between `<!-- sase:strands -->` and `<!-- /sase:strands -->` on every
`sase memory init`, so no parser needs teaching about this shape.

### 2. Ordered-list Reference Memory entries

In `src/sase/memory/notes.py`:

- Replace `render_long_memory_sections` with `render_long_memory_entries` (same
  signature: `Iterable[MemoryNote] -> str`). Update `__all__` and the docstring.
- Keep the existing filter and ordering: `type == "reference"`, sorted by
  `relative_path`.
- For each note, collapse the description with the existing `collapse_description`
  helper, then render ``f"{number}. **`{note.relative_path}`** - {description}"`` (or
  the bare ``f"{number}. **`{note.relative_path}`**"`` when the collapsed description is
  empty), and wrap it with `wrap_markdown(entry, width=width)`.
- Import `wrap_markdown` from `sase.markdown_wrap`; `sase.markdown_width` is already
  imported. Resolve `markdown_print_width()` **once** before the loop, per that module's
  docstring.
- Join entries with `"\n"` — a tight list, no blank lines between entries.

In `src/sase/amd/_memory.py`:

- Update the import and the single call site (currently
  `long_entries = render_long_memory_sections(rendered_long_notes)`).
- Extend `_long_memory_description_blockers` so a description that cannot render as one
  list-item paragraph is a blocker. Keep the existing heading check and its message
  (`"... description must not contain Markdown headings"`); add a second check that
  rejects a description containing a blank line or any line whose first non-blank
  character starts a Markdown block construct (`-`/`*`/`+` bullet, `N.`/`N)` ordered
  item, `>` quote, or a ` ` ```/`~~~`fence), with a message along the lines of`"<path>:
  reference memory note description must be a single paragraph"`. Keep the function
  returning a sorted, deterministic tuple of blockers.

Leave the `_LONG_MEMORY_INTRO` paragraph, its first-sentence presence assertion, and the
`expected_long_paths` structural assertion in `_render_managed_agents` as they are —
they keep working once the parser (step 3) understands the new shape.

### 3. Parse the new shape back (and keep parsing the old ones)

`src/sase/amd/_agents_doc.py` parses a document's Reference Memory entries for two
reasons: `_existing_agents_long_descriptions` recovers a description for a note whose
frontmatter has none, and `sase memory agent-docs list` counts reference entries. Both
must work against a document rendered in the new shape.

- Add a list-entry pattern beside `_LONG_MEMORY_ENTRY_RE` and `_LONG_MEMORY_SECTION_RE`,
  matching
  ``^\d+[.)]\s+\*\*`(?P<path>(?:sase/)?memory/[A-Za-z0-9_.-]+\.md)`\*\*(?:\s*-\s*(?P<description>.*))?$``.
- Teach `collect_long_memory_entries` the list shape. A list entry's description is the
  inline text after `-`, continued on the following **indented, non-blank** lines, and
  it ends at the first blank line, dedented line, heading, or next entry. Collapse those
  continuation lines back into one line (whitespace-collapsed, single spaces) so a
  recovered description matches what the note's frontmatter would hold.
- Do **not** change how the H3 (`### [N.N ]`path`) and legacy (` **`memory/x.md`** ``)
  shapes are parsed: they still collect a possibly multi-paragraph description verbatim,
  still stop at a sibling heading, and still keep fenced content. Existing tests cover
  this and must keep passing unchanged.
- `_long_memory_entry_path` should recognize all three shapes.

### 4. Documentation

Update the prose that describes the old shapes:

- `docs/init.md` — the paragraph describing what `sase memory init` renders ("renders
  reference memory as one numbered H3 subsection per reference note (headed by the note
  path, with the description as the body)" and "immediately followed by those per-note
  H3 subsections").
- `docs/memory.md` — the `type: reference` bullet near the top ("that block renders
  verbatim as the body of the note's numbered `## Reference Memory` section, while
  single-line surfaces collapse it"). It must now say the Reference Memory list
  collapses a description to one paragraph and that a multi-block description is
  rejected, while the `## Children` section on an audited read still renders a block
  verbatim.
- `docs/configuration.md` — the `{{ reference_entries }}` description ("the
  reference-memory instruction paragraph plus one H3 subsection per top-level reference
  note").
- `src/sase/main/init_memory/templates/memory-README.template.md` — the `description`
  bullet in the frontmatter schema ("reference sections render those blocks verbatim,
  while single-line surfaces collapse them").

### 5. Regenerate

Run `sase memory init` from the repo root and commit everything it rewrites:

- `AGENTS.md` and its byte-identical shims `CLAUDE.md`, `GEMINI.md`, `QWEN.md`,
  `OPENCODE.md`.
- The managed roster regions of `sase/memory/decisions.md` and
  `sase/memory/task_types.md`.
- `sase/memory/README.md` (its per-note `Lines:` / `Approx. tokens:` statistics move for
  the two web descriptors, and the frontmatter-schema bullet changes with the template).

Then run `sase memory init` a **second** time and confirm it reports no actions — the
generator must be idempotent, which is also what several tests assert via
`plan_memory().actions == ()`.

Do not hand-edit `AGENTS.md`, the provider shims, or the managed roster regions.

## Tests

Update the expectations that pin the old shapes, and add coverage for the new one.

Roster shape:

- `tests/memory/test_memory_web.py` (asserts
  ``"- **Alpha Term** (`alpha`) - This decision summary"``).
- `tests/memory/test_memory_web_supersession.py` (three assertions on `- **Alpha Term**`
  / `- **Beta Term**`, including the exact `_[superseded by ...]_` marker line).
- `tests/main/test_init_memory_task_types_note.py` (asserts ``"- **Bug** (`bug`)"``).

Reference Memory shape:

- `tests/test_memory_notes.py` — rename the three `test_render_long_memory_sections_*`
  tests and rewrite their expected output. The "preserves block descriptions" test
  becomes a _collapses_ test; the "omits empty description body" test becomes "renders a
  bare entry with no `-` suffix"; the ordering/filtering test keeps its intent with
  numbered entries. Leave the `_render_memory_note_references` /
  `render_children_section` tests alone.
- `tests/main/test_init_memory_managed_agents_descriptions.py` —
  `test_init_memory_managed_agents_renders_block_long_memory_descriptions` currently
  asserts a literal
  ``"### 2.1 `sase/memory/block.md`\n\nLead paragraph.\n\n- One\n- Two\n\nTrailer.\n"``
  block; convert it into a blocker test for the new single-paragraph rule. The two
  `_existing_agents_long_descriptions` legacy-recovery tests must keep passing
  **unchanged** — they are the back-compat guarantee. Add a new test that round-trips
  the new list shape through `parse_amd_agents_document` /
  `_existing_agents_long_descriptions`, including a wrapped multi-line entry and an
  entry numbered 10 or higher.
- Sweep the remaining generated-document tests for assertions that break, and fix them:
  `tests/main/test_init_memory_managed_agents_generation.py`,
  `tests/main/test_init_memory_memory_webs.py`,
  `tests/main/test_init_memory_agents_templates.py`,
  `tests/main/test_init_memory_glossary.py`,
  `tests/main/test_init_memory_markdown_templates.py`,
  `tests/main/test_init_memory_validation.py`,
  `tests/main/test_init_memory_agent_docs.py`,
  `tests/main/test_init_onboarding_memory.py`,
  `tests/main/test_memory_agent_docs_list.py`, `tests/main/test_init_memory_plan.py`,
  `tests/main/test_init_memory_handler_outputs.py`,
  `tests/main/test_init_memory_bead_note.py`, `tests/main/test_section_numbers.py`. Note
  that the `## N. Reference Memory` **H2** heading and the document-wide section numbers
  for `## Core Memory`, `## Reference Memory`, and `## Memory Webs` do **not** change —
  only the H3 children of Reference Memory disappear. Assertions on
  `"## 2. Reference Memory"` and on Memory Webs numbering should still hold.

Add or keep a test that runs Prettier over a generated `AGENTS.md`
(`test_init_memory_managed_agents_wraps_long_memory_descriptions` already does this with
`prettier --check`) so the ordered-list continuation indents stay format-stable.

## Verification

```bash
just install     # ephemeral workspace clones may have drifted deps
just check
```

`just check` covers ruff, mypy, symvision, toobig, keep-sorted, `fmt-md-check`
(Prettier), and the diff-scoped test lane. Two extras worth doing explicitly:

- `just fmt-md-check` must pass against the regenerated `AGENTS.md`, the four provider
  shims, `sase/memory/decisions.md`, `sase/memory/task_types.md`, and
  `sase/memory/README.md`. If it does not, the renderer is disagreeing with Prettier —
  fix the renderer, never the generated file by hand.
- Run `sase memory init` twice and confirm the second run is a no-op.

If `just check`'s scoped selection escalates or looks unusual, run `just check-full`
through `/sase_monitor` (never inline) with the `TESTING` / `TESTED` status pair, per
`sase/memory/lint_and_test.md`.

If renaming `render_long_memory_sections` trips symvision, follow
`sase/memory/symvision.md` — the rename should leave exactly one non-test consumer
(`src/sase/amd/_memory.py`), which is enough.

## Out Of Scope

- The `## Children` section rendering (`render_children_section`,
  `_render_memory_note_references`) and the `## Linked References` section — unchanged.
- The `roster: inline` style used by the `glossary` web — unchanged.
- Home and chezmoi-managed roots. `~/sase/memory/` has its own reference note and its
  own generated `~/CLAUDE.md`; those regenerate whenever the user next runs
  `sase memory init` for the home scope. Do not try to regenerate them from this repo's
  change, but do mention the pending home drift in the completion summary.
- Any change to how a reference note's `description` frontmatter is written or
  normalized on disk (`apply_memory_frontmatter`, `_normalized_description`) — only the
  rendered agent-instruction surface changes.
