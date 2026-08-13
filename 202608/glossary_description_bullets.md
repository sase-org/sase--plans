---
tier: tale
title: List each glossary term as its own bullet in the Tier 2 description
goal:
  The generated Tier 2 entry for sase/memory/glossary.md in AGENTS.md and every provider
  instruction shim lists each glossary term, with its aliases, as its own Markdown
  bullet.
size: medium
proposed_by: bbugyi200.athena.z3
create_time: 2026-08-13 07:46:14
status: wip
---

# Render Glossary Terms As Bullets In The Tier 2 Description

## Goal

The `sase/memory/glossary.md` long note is generated from `memory.glossary` in the
project's `sase.yml`. Its frontmatter `description` is what `sase memory init` renders
into the `## Tier 2 (long-term) Memory` section of `AGENTS.md` and every provider
instruction shim (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`).

Today that description is one run-on paragraph with the terms semicolon-separated. Make
each glossary term (with its aliases) render as its own Markdown bullet.

### Current Output

```markdown
**`sase/memory/glossary.md`**  
Read this note before relying on any of these SASE glossary terms and aliases - Agent
Clan; Agent Family; Agent Hood (aka hood, agent neighborhood); Agent Lane; Agent
Instruction File (aka agents.md file); Agent Neighbor; Agent Tribe; Artifact Reference
(aka ref); Patch; Sase Project (aka project); Sase Repo (aka repo); Sase Workspace (aka
workspace); Stitch; Xprompt; Xprompt Memory (aka memory file); Xprompt Part; Xprompt
Swarm; Xprompt Workflow. Read it with `sase memory read glossary.md` whenever one of
those terms or aliases appears in a prompt, bead, plan, or code comment and you are not
certain what it means in SASE.
```

### Target Output

```markdown
**`sase/memory/glossary.md`**  
Read this note before relying on any of these SASE glossary terms and aliases:

- Agent Clan
- Agent Family
- Agent Hood (aka hood, agent neighborhood)
- Agent Lane
- Agent Instruction File (aka agents.md file)
- Agent Neighbor
- Agent Tribe
- Artifact Reference (aka ref)
- Patch
- Sase Project (aka project)
- Sase Repo (aka repo)
- Sase Workspace (aka workspace)
- Stitch
- Xprompt
- Xprompt Memory (aka memory file)
- Xprompt Part
- Xprompt Swarm
- Xprompt Workflow

Read it with `sase memory read glossary.md` whenever one of those terms or aliases
appears in a prompt, bead, plan, or code comment and you are not certain what it means
in SASE.
```

Bullet order stays the catalog order (which follows `memory.glossary` declaration order
in `sase.yml`), exactly as the semicolon list orders terms today. Term and alias text
keeps its current `md_escape` treatment and the current `(aka <a>, <b>)` alias form.

## Background: How A Long-Note Description Reaches `AGENTS.md`

1. `src/sase/main/init_memory/glossary.py::_glossary_memory_description` builds the
   description string from the Rust-backed glossary catalog.
2. `_render_glossary_memory` writes it into the note through
   `sase.memory.notes.apply_memory_frontmatter(...)`, which calls
   `_render_memory_frontmatter` (YAML dump) and then `_prettier_stable_frontmatter`.
3. `src/sase/main/init_memory/root_rendering.py::generated_long_notes` reads the
   description back out with `parse_memory_note_text`, producing a
   `GeneratedLongMemoryNote`.
4. `src/sase/amd/_memory.py::_render_managed_agents` feeds those descriptions to
   `sase.memory.notes.render_memory_note_references`, which emits ``**`<path>`**`` +
   two-space hard break + the description.
5. The result goes through the `AGENTS.template.md` `{{ tier2_entries }}` slot, then
   `format_generated_memory_markdown`, then is copied verbatim to each provider shim.

So the description round-trips through YAML frontmatter, and both the writer and the
reader must preserve line structure for bullets to survive.

## Design Decision

Make long-note `description` a **Markdown block** rather than a single line, and keep a
collapsed single-line view for the display surfaces that need one. This is the only
design that works, because the `AGENTS.md` Tier 2 body is rendered from the note's
frontmatter description; there is no second channel to smuggle a bullet list through.

Scope the change to the shared memory-note plumbing (so any long note can carry a block
description) rather than special-casing the glossary, since the parser, the writer, and
the `AGENTS.md` fallback reader all sit on that shared path.

## Verified Facts (Confirmed Empirically During Planning)

Do not re-derive these; they are already checked. Do re-confirm anything you change.

1. **`format_generated_memory_markdown` already renders the target shape correctly.**
   Given a Tier 2 entry whose description is
   `paragraph / blank / bullets / blank / paragraph`, it emits the label with a
   two-space hard break, wraps the paragraphs at the configured prose width, and wraps
   bullets with a `- ` / `  ` indent. **No change is needed in
   `src/sase/main/init_memory/formatting.py`.**
2. **Prettier agrees with that output.**
   `prettier --prose-wrap=always --print-width=88 --parser=markdown` is a fixpoint on
   it, so `just fmt-md-check` will pass.
3. **Prettier preserves YAML literal block scalars in frontmatter verbatim**, including
   the blank lines inside them.
4. **`_prettier_stable_frontmatter` already passes a `description: |-` block through
   unchanged**, because the block body lines are indented (so its `- ` re-indent branch
   does not fire) and the `description: ` wrap branch sees only the short `|-` header.
   This needs a regression test and a comment, not a rewrite.
5. **`yaml.safe_dump` will _not_ produce a literal block scalar on its own.** For a
   multi-line string it emits a single-quoted scalar in which each embedded newline is
   doubled, which does not round-trip. A custom `str` representer with `style="|"` is
   required.
6. **The two blockers today are exactly these:**
   - `notes.py::_render_memory_frontmatter` does `" ".join(description.split())`.
   - `notes.py::parse_memory_note_text` runs the description through
     `_normalized_scalar`, which is `" ".join(value.split())`.

## Implementation

### 1. `src/sase/memory/notes.py` — block-capable descriptions

- Add `_normalized_description(value: Any) -> str | None`:
  - non-`str` or blank → `None`;
  - collapse intra-line whitespace per line (`" ".join(line.split())`);
  - drop leading and trailing blank lines;
  - collapse runs of two or more blank lines to one;
  - join with `"\n"`.

  For a single-line input this is byte-identical to `_normalized_scalar` today, so
  existing notes are unaffected.

- Use `_normalized_description` for `description` in `parse_memory_note_text`. Leave
  `_normalized_scalar` in place for `type` and `_normalized_path_scalar` for `parent`.

- Add a public `collapse_description(description: str | None) -> str | None` helper
  (export it in `__all__`) that returns `" ".join(description.split())`, for the
  one-line display surfaces in step 3.

- Teach `_render_memory_frontmatter` to emit a literal block scalar when the description
  contains a newline:
  - Add a module-level `class _MemoryFrontmatterDumper(yaml.SafeDumper)` with a `str`
    representer that uses `style="|"` when the value contains a newline **and** is
    block-safe, and the default style otherwise. Keep the existing
    `allow_unicode`/`default_flow_style`/`sort_keys`/`width` arguments.
  - Block-safe means: no line has trailing whitespace after normalization (already true
    post-`_normalized_description`), and no line, once stripped, equals `---` or `...`.
    A stripped `---` line inside the block would terminate the frontmatter early for
    both `notes.py::_frontmatter_close_line_range` and
    `formatting.py::_split_frontmatter`, since both compare `line.strip() == "---"`.
  - When a multi-line description is _not_ block-safe, fall back to collapsing it to a
    single line rather than emitting frontmatter that cannot round-trip. Never emit
    unparseable output.
  - Keep the single-line path exactly as it is today (including
    `_can_wrap_plain_description` and the `textwrap` wrap in
    `_prettier_stable_frontmatter`).

- Add a short comment in `_prettier_stable_frontmatter` recording that literal block
  scalars must pass through untouched, and why the existing branches already do that
  (fact 4 above).

- `render_memory_note_references` needs no change: it appends the description as one
  element and joins with `"\n"`, so embedded newlines survive.

### 2. `src/sase/amd/_agents_doc.py` — parse block descriptions out of `AGENTS.md`

`_long_memory_entries` currently stops a description at the first blank line, and
`_description_text` collapses everything to one line. That path feeds
`_memory.py::_existing_agents_long_descriptions`, which is the fallback source of a
description for a long note whose frontmatter lacks one. With block descriptions in
`AGENTS.md`, that fallback would silently truncate at the first blank line.

- Change `_long_memory_entries` to consume description lines until the next ``**`…`**``
  entry line or the end of the Tier 2 section, keeping blank lines instead of breaking
  on them.
- Change `_description_text` to preserve line structure: normalize with the same rules
  as `_normalized_description` (per-line whitespace collapse, trimmed blank edges, no
  doubled blank lines), then apply the existing legacy `_Read when …_` strip to the
  final line only.
- Keep `_LONG_MEMORY_ENTRY_RE` anchored so bullet lines can never be mistaken for a new
  entry.

Apply the same structure-preserving normalization to the legacy `_AGENTS_LONG_MEMORY_RE`
branch in `_memory.py::_existing_agents_long_descriptions`, so both readers agree.

### 3. Collapse to one line at the single-line display surfaces

- `src/sase/main/init_memory/root_rendering.py::_note_description` — the memory README
  renders `- Description: {…}` as one bullet. Use `collapse_description(...)` so the
  README shape is unchanged.
- `src/sase/xprompt/loader_memory.py` (the `description=note.description` argument to
  `XPrompt`) — use `collapse_description(...)`. `XPrompt.description` flows into the
  xprompt catalog HTML, `sase xprompt show`, completion menus, and LSP hover, all of
  which assume a single line.
- `src/sase/memory/read_log.py` passes the description straight through into a rebuilt
  `MemoryNote`; leave it alone.

### 4. `src/sase/main/init_memory/glossary.py` — build the bullet block

Rewrite `_glossary_memory_description` to return the block from the **Target Output**
section:

- lead line:
  `Read this note before relying on any of these SASE glossary terms and aliases:`
- blank line
- one `- <Term>` or `- <Term> (aka <alias>, <alias>)` bullet per `catalog.entries`, in
  catalog order, using the existing `md_escape` calls and `entry.display_aliases`
- blank line
- trailer:
  ``Read it with `sase memory read glossary.md` whenever one of those terms or aliases appears in a prompt, bead, plan, or code comment and you are not certain what it means in SASE.``

Everything else in `glossary.py` stays as it is.

### 5. Documentation

- `docs/memory.md` — in the frontmatter section, note that a long note's `description`
  may be a Markdown block (authored as a YAML literal block scalar) that is rendered
  verbatim into the Tier 2 entry, and that single-line surfaces collapse it.
- `src/sase/main/init_memory/templates/memory-README.template.md` — extend the
  `description` bullet under **Frontmatter Schema** with the same point. This template
  is prettier-ignored, so keep its existing hand-wrapped style.

### 6. Regenerate the managed files

Do **not** hand-edit `sase/memory/glossary.md`, `AGENTS.md`, `sase/memory/README.md`, or
the provider shims. Regenerate them:

```bash
just install
sase memory init
sase memory init   # must be a no-op; proves idempotence
```

Approval of this plan is the user's authorization for that regeneration — it is the
deliverable, not an incidental memory edit. Regenerating updates
`sase/memory/glossary.md`, `sase/memory/README.md`, `AGENTS.md`, `CLAUDE.md`,
`GEMINI.md`, `OPENCODE.md`, and `QWEN.md`.

## Tests

- `tests/main/test_init_memory_glossary.py`
  - `test_memory_plan_glossary_description_indexes_terms_aliases_and_is_format_stable`
    asserts `": " not in description`, `"#" not in description`, and
    `"\t" not in description`. Those guarded the plain-scalar wrap path and are obsolete
    for a block description (the lead line now ends in a colon). Replace them with
    assertions on the new shape: the description starts with the lead line, contains
    `"\n- Agent Clan\n"` and `"\n- Artifact Reference (aka ref)\n"`, does not contain
    the suppressed alias `artifact references`, and ends with the trailer sentence.
  - Keep `format_generated_memory_markdown(glossary_text) == glossary_text`.
  - Keep the existing no-alias assertion (`"(aka" not in description` for an alias-free
    catalog) and the existing `parse_memory_note_text` round-trip.
  - Add: `apply_memory_frontmatter` on the generated note is a fixpoint (running it
    twice changes nothing), so `sase memory init` cannot churn.
- `tests/test_memory_notes.py`
  - A multi-line description round-trips through `apply_memory_frontmatter` →
    `parse_memory_note_text` unchanged, and the rendered frontmatter uses a
    `description: |-` literal block.
  - A single-line description renders byte-identically to today (regression guard for
    the wrap path).
  - A multi-line description containing a line that strips to `---` falls back to a
    single collapsed line and still parses.
  - `collapse_description` flattens a block to one line.
  - `_prettier_stable_frontmatter` leaves a block scalar untouched.
- `tests/main/test_init_memory_managed_agents.py` (or the closest `AGENTS.md` rendering
  test) — a long note with a block description renders in Tier 2 as label + hard break +
  paragraph + blank + bullets + blank + paragraph, and `parse_amd_agents_document`
  recovers the whole block rather than truncating at the first blank line.
- Confirm the rendered `AGENTS.md` still passes the existing structural-anchor and Tier
  2 path checks in `_render_managed_agents`.

## Verification

```bash
just install
just check
just fmt-md-check
```

Then run the exhaustive gate through `/sase_monitor` before landing — never inline:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Also eyeball the regenerated `CLAUDE.md` Tier 2 block to confirm the bullets read the
way the **Target Output** section specifies.

## Risks

- **Frontmatter round-trip.** The literal block scalar is the load-bearing piece. The
  `---`/`...` block-safety fallback and the fixpoint tests cover it.
- **Legacy `AGENTS.md` description recovery.** Step 2 changes how descriptions are read
  back out of an existing `AGENTS.md` for notes with no frontmatter description. Cover
  the blank-line-containing case and the legacy `_Read when …_` case with tests.
- **Wider Tier 2 section.** The glossary entry grows from 9 lines to about 24. That is
  the intent, and it is only in the read-on-demand index, not inlined memory.

## Out Of Scope

- Changing glossary term text, aliases, definitions, or ordering in `sase/sase.yml`.
- Any Rust change in `../sase-core`. The glossary catalog, validation, and alias
  resolution stay Rust-owned; only Python-side rendering changes here.
- Block descriptions for short notes (short notes carry no `description`).
- Reformatting the descriptions of the other long notes.
