---
tier: tale
title: Migrate the generated glossary note from Tier 1 to Tier 2
goal:
  The generated sase/memory/glossary.md note is a long (Tier 2) memory note whose
  generated description indexes every glossary term and alias and tells agents to read
  the note for those definitions.
size: medium
proposed_by: bbugyi200.athena.ys
create_time: 2026-08-12 13:31:08
status: wip
---

# Migrate The Generated Glossary Note From Tier 1 To Tier 2

## Goal

Change `sase memory init` so the generated project glossary note
(`sase/memory/glossary.md`) is a **long** (Tier 2) memory note instead of a **short**
(Tier 1) note, and give it a generated `description` that enumerates every glossary term
and every displayed alias plus an instruction telling the agent to read the note when it
needs any of those definitions.

After this change, agent instruction files carry a compact term index in Tier 2 instead
of ~1600 tokens of inlined definitions in Tier 1, and agents fetch definitions on demand
with `sase memory read glossary.md`.

## Background (verified facts)

- `sase/memory/glossary.md` is fully generated from the `memory.glossary` section of a
  project-local `sase/sase.yml`. It is never hand-edited; the current frontmatter is
  `type: short`, `parent: AGENTS.md`,
  `description: Project-local glossary generated from sase.yml.`,
  `sase_generated: glossary`.
- `src/sase/main/init_memory/glossary.py` renders it. `_render_glossary_memory()` builds
  the body (`# Glossary of Terms`, then `## <Term>`, an optional
  `ALIASES: <comma-separated>` line, and the definition) and wraps it with
  `apply_memory_frontmatter(..., note_type="short", ...)`, returning
  `GeneratedGlossaryMemory(body=..., content=...)`.
- Tier 1 inlining happens because `root_rendering.generated_short_notes()` puts
  `generated_glossary.body` into the short-note mapping, which
  `amd/_memory.py::_short_memory_bodies()` merges over on-disk short notes and
  `_render_managed_agents()` inlines into `AGENTS.md`.
- Tier 2 entries come from `_render_managed_agents()`: it discovers notes on disk,
  overlays the `generated_long_notes` mapping (path -> `GeneratedLongMemoryNote`
  metadata), keeps `type == "long"` notes whose `parent` is `AGENTS.md`, and renders
  `render_memory_note_references()` output (a bold path line followed by the
  description).
- `root_planning.memory_root_context()` is the single call site that assembles both
  mappings and hands them to `_amd_sync_plan()`.
- `render_expected_memory_files()` writes the glossary file from its own dedicated
  `generated_glossary` parameter and overlays that content when rendering
  `sase/memory/README.md`, so the README picks up a new note type automatically.
- `sase/memory/sase_beads.md` and `sase/memory/sase_sizes.md` are the existing precedent
  for generated long notes (`_GENERATED_PROJECT_LONG_MEMORY_SPECS`,
  `generated_long_notes()`); `tests/main/test_init_memory_bead_note.py` is the precedent
  test module.
- Markdown formatting gate: prettier 3.8.1 with `proseWrap: always` and
  `printWidth: 88`, run by `just fmt-md` / `just fmt-md-check` over `**/*.md`.
  `memory/notes.py::_prettier_stable_frontmatter()` pre-wraps a long `description:`
  value into the indented block form prettier keeps, but only when
  `_can_wrap_plain_description()` holds: the value must contain no `": "`, no `"#"`, and
  no tab.

## Constraints And Decisions

1. **Keep the `sase_generated: glossary` marker.** The collision blocker
   (`_glossary_collision_blocker`) and retirement path (`_retired_glossary_note_paths`)
   both key off `is_generated_glossary_memory_content()`. Changing the note type must
   not change the marker, so an existing generated note is still recognized as
   SASE-owned and is overwritten (not blocked) on the migration pass.
2. **Description text must avoid `": "`, `"#"`, and tabs** so the frontmatter stays a
   prettier fixpoint. Use `(aka ...)` for aliases and an em dash (not a colon) in the
   lead-in sentence.
3. **Alias set = each entry's `display_aliases`.** That is the same set the body's
   `ALIASES:` line shows: configured aliases minus aliases that are derivable plurals of
   the term. Derivable plurals are matched automatically by the glossary matcher, so
   listing them adds noise without adding recall. Assumption worth flagging to the user:
   if they want literally every authored alias including redundant plurals, switch to
   `configured_aliases` — it is a one-word change in the same helper.
4. **Terms/aliases are user config**, so escape them for Markdown with `md_escape` the
   same way the body does. A term containing `": "` or `"#"` degrades only the
   frontmatter wrapping (single long `description:` line), never correctness; do not add
   sanitizing that would misrepresent a term.
5. **Do not merge the glossary into `generated_project_long_contents`.** That mapping
   drives packaged-template file writes and is gated on `include_project_memory`.
   Glossary generation is gated on project config instead. Only the AMD _metadata_
   mapping (`generated_long_notes`) gains the glossary entry, and it must be populated
   regardless of `include_project_memory`.

## Implementation Steps

### Step 1 — Render the glossary as a long note with a generated description

File: `src/sase/main/init_memory/glossary.py`

- Delete the fixed `GENERATED_GLOSSARY_DESCRIPTION` constant and add a helper, e.g.
  `_glossary_memory_description(catalog: GlossaryCatalog) -> str`, that builds the
  description from the catalog entries in catalog order:
  - Each item renders as `md_escape(entry.term)`, and when `entry.display_aliases` is
    non-empty, `md_escape(term) (aka <alias>, <alias>)` with each alias `md_escape`d.
  - Join items with `; ` (semicolons keep multiword terms with parenthesized aliases
    readable).
  - Wrap with a lead-in and a closing instruction that name the read command, for
    example:

    > Read this note before relying on any of these SASE glossary terms and aliases —
    > `<items>`. Read it with `sase memory read glossary.md` whenever one of those terms
    > or aliases appears in a prompt, bead, plan, or code comment and you are not
    > certain what it means in SASE.

    Exact wording is the implementer's call; it must (a) list every term and every
    displayed alias, (b) tell the agent to read the note for those definitions, and (c)
    satisfy Constraint 2.

- Change `_render_glossary_memory()` to call `apply_memory_frontmatter()` with
  `note_type="long"` and `description=_glossary_memory_description(catalog)`, keeping
  `parent=AGENTS_PARENT`, `extra=_generated_glossary_frontmatter_extra()`, and
  `preserve_existing_extra=False`.
- The body is unchanged.
- `GeneratedGlossaryMemory.body` loses its only production consumer in Step 2. Drop the
  field (leaving a `content`-only dataclass) rather than leaving dead state; symvision
  and `just lint` will flag it otherwise. Update the dataclass docstring and every
  construction/assertion site.

### Step 2 — Rewire the glossary from the short-note path to the long-note path

Files: `src/sase/main/init_memory/root_rendering.py`,
`src/sase/main/init_memory/root_planning.py`

- `root_rendering.generated_short_notes()`: remove the `generated_glossary` keyword so
  the function returns only the generated `sase/memory/sase.md` body. Update its
  docstring.
- Add a small helper in `root_rendering.py` for the glossary's long-note metadata entry,
  or build it at the `root_planning` call site — either is fine, but the
  `sase/memory/glossary.md` relative path must be spelled once, reusing
  `CANONICAL_MEMORY_RELATIVE_ROOT` (`root_planning._generated_glossary_relative_path()`
  already exists).
- `root_planning.memory_root_context()`: build the long-note content mapping as the
  packaged project long notes (only when `include_project_memory`) **plus** the glossary
  content when `generated_glossary is not None`, then pass
  `generated_long_notes(<that mapping>)` to `_amd_sync_plan()` unconditionally (today it
  passes `{}` unless `include_project_memory`). `generated_long_notes()` parses the
  frontmatter and raises `ValueError` when a description is missing; the glossary always
  has one after Step 1.
- Leave `render_expected_memory_files()`'s `generated_glossary` parameter and the file
  write / README overlay exactly as they are.

### Step 3 — Stop a stale on-disk short note from being inlined on the migration pass

File: `src/sase/amd/_memory.py`

On the first `sase memory init` after this change, the on-disk `sase/memory/glossary.md`
still says `type: short`. `_short_memory_bodies()` discovers it from disk, and the
generated overlay no longer replaces it, so without a fix that pass would inline the
glossary into Tier 1 **and** list it in Tier 2 — and `sase memory init --check` would
never converge.

- Give `_short_memory_bodies()` the set of generated long-note paths and drop those
  paths from the discovered short-note bodies. A path that a generated long note owns
  can never be a short body, so this is a general rule, not a glossary special case.
- Update the call in `plan_amd_memory_sync()` to pass `generated_long_notes or {}` (its
  keys) alongside `generated_short_notes`.
- Keep `_long_memory_descriptions()` and `_long_memory_description_updates()` as they
  are: the first already lets `generated_long_notes` override discovered descriptions,
  and the second already skips generated paths so it will not fight the file write.

### Step 4 — Tests

File: `tests/main/test_init_memory_glossary.py` (7 existing tests; update the ones that
assert Tier 1 inlining), plus a new or extended case for the AMD wiring.

Update/keep:

- `test_memory_plan_generates_project_glossary_before_agents` — assert the generated
  note is `type: long` and that the rendered `AGENTS.md` lists
  `**\`sase/memory/glossary.md\`\*\*`under`## Tier 2 (long-term)
  Memory`with the generated description, and that no`## Tier 1` section inlines the
  glossary body.
- `test_memory_apply_generates_glossary_idempotently_and_copies_provider_shims` — still
  idempotent; a second pass reports no changes and provider shims match `AGENTS.md`.
- `test_memory_plan_omits_alias_line_when_only_alias_is_term_plural` — the body rule is
  unchanged; extend it to assert the derivable plural is likewise absent from the
  generated description.
- `test_memory_plan_blocks_invalid_project_glossary`,
  `test_memory_plan_refuses_to_overwrite_unmarked_glossary_note`,
  `test_memory_init_retires_generated_glossary_when_config_is_removed`,
  `test_memory_init_ignores_home_glossary_config` — behavior must not change; adjust
  fixtures only if they hard-code `type: short`.

Add:

- Description content: every configured term and every displayed alias appears in the
  frontmatter `description`, aliases render inside `(aka ...)`, and the description
  contains the read instruction.
- Prettier stability: the generated description contains no `": "`, no `"#"`, and no
  tab, and the rendered note is unchanged by a second
  `format_generated_memory_markdown()` pass.
- Single-pass migration convergence: seed a repo whose `sase/memory/glossary.md` is a
  previously generated `type: short` note (marker present); one `sase memory init` pass
  rewrites it to `type: long`, `AGENTS.md` Tier 1 contains no glossary section, Tier 2
  lists it, and a following `sase memory init --check` reports no drift. This is the
  regression test for Step 3 — write it first and watch it fail before fixing
  `_short_memory_bodies()`.
- README statistics: `sase/memory/README.md` counts the glossary under long notes.

Check `tests/main/test_init_memory_managed_agents.py` and
`tests/main/test_init_memory_markdown_templates.py` for assertions about short/long note
sets that now shift, and update them if they break.

### Step 5 — Documentation

- `docs/init.md` (glossary paragraph, near the `memory.glossary` rendering description):
  replace the claim that initialization "uses the fresh body while composing `AGENTS.md`
  and the provider instruction files" with the Tier 2 behavior — the note is generated
  as a long note whose generated description indexes every term and alias, and agents
  read it on demand.
- `docs/configuration.md`, `### memory.glossary`: replace "adds that short note to
  `sase/memory/README.md`, and inlines it into `AGENTS.md` plus the provider instruction
  copies" with the long-note behavior, and document what the generated description
  contains. Also check the later `sase memory init` paragraph that mentions regenerating
  `sase/memory/glossary.md`.
- Do not hand-edit `CHANGELOG.md`; it is release-please-managed and prettier-ignored.

### Step 6 — Regenerate this repo's own memory artifacts

This repo has a non-empty `memory.glossary`, so its own generated files change.

- Run `sase memory init --no-commit` so the generated files land in the working tree
  without an out-of-band commit/push, then let the normal commit workflow stage them.
- Expected changes: `sase/memory/glossary.md` (now `type: long` with the enumerated
  description), `sase/memory/README.md` (note type + statistics), `AGENTS.md` (glossary
  moves from Tier 1 to Tier 2 and the remaining Tier 1 section numbers shift), and the
  provider shims `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md` (byte-identical
  copies of `AGENTS.md`).
- Confirm `sase memory init --check` then reports no drift.
- The user explicitly requested this migration, which authorizes the memory-file edit
  and the mandatory `sase memory init` follow-through.

## Verification

Run from the workspace checkout, in order:

1. `just install` (workspace virtualenvs may be stale).
2. `sase memory init --no-commit`, then `sase memory init --check` (expect no drift).
3. `just check-full` — every lint gate plus the full test suite. Use the full gate
   rather than `just check` because this change rewrites generated agent instruction
   files, touches the shared AMD memory-sync path used by every managed root, and edits
   docs; `just fmt-md-check` over the regenerated Markdown is part of that gate.

Manual sanity check after regeneration:

- `AGENTS.md` Tier 2 lists `sase/memory/glossary.md` with a description naming every
  term (Agent Clan through Xprompt Workflow) and every alias (hood, agent neighborhood,
  agents.md file, ref, project, repo, workspace, memory file).
- `AGENTS.md` Tier 1 no longer contains the glossary definitions.
- `sase memory read glossary.md --reason "<reason>"` prints the full glossary.

## Out Of Scope

- The `memory.glossary` config schema, validation, alias/plural matching, and the Rust
  glossary catalog are unchanged.
- ACE prompt highlighting, `K` preview, `Ctrl+]` go-to-definition, and the xprompt LSP
  read glossary entries from `sase/sase.yml`, not from the memory note, so they need no
  change.
- No change to how other short notes are inlined.
