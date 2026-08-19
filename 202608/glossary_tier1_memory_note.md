---
tier: tale
title:
  Generate sase/memory/glossary.md as a Tier 1 note and flatten the Tier 2 H3 wrapper
goal:
  "`sase memory init` generates a short-term `sase/memory/glossary.md` note that is
  inlined into Tier 1 of every agent instruction file, the Tier 2 `Glossary Terms` H3
  section is gone, and the `Long-Term Memory Files` H3 wrapper is removed so its
  instruction paragraph and per-note subsections sit directly under the Tier 2 H2."
size: medium
status: wip
proposed_by: bbugyi200.athena.07h
create_time: 2026-08-19 07:46:37
---

# Plan: Glossary becomes a Tier 1 memory note; Tier 2 loses its H3 wrapper

## Goal

Two structural changes to generated agent instruction files, landed together because
both rewrite the same `AGENTS.md` render path and both force a regeneration of this
repo's committed instruction files.

1. **Glossary moves to Tier 1 as a real note.** `sase memory init` renders a generated
   short-term memory note at `sase/memory/glossary.md` from the project's
   `memory.glossary` config. The existing short-note inlining machinery then renders it
   into the Tier 1 section of `AGENTS.md` and every provider shim. The Tier 2
   `Glossary Terms` H3 section is deleted.
2. **Tier 2 loses the `Long-Term Memory Files` H3 wrapper.** Its instruction paragraph
   moves to be direct body content of the `## Tier 2 (long-term) Memory` H2, and each
   long note's subsection moves up a level from H4 to H3.

## Background

Commit `eaafcbe72` ("retire generated glossary note for a Tier 2 instruction block")
deleted the previously generated `sase/memory/glossary.md` — which had rendered every
term's full definition as a _long_ note — and replaced it with the compact Tier 2
`Glossary Terms` H3 block. This plan reinstates a generated note at the same path, but
with the **compact** content (the instruction paragraph plus the one-line
`**GLOSSARY TERMS:**` roster), typed `short` instead of `long`. The full definitions
stay on demand behind `sase glossary read`; nothing about `sase glossary` changes.

Much of the plumbing this plan needs — the `sase_generated: glossary` marker, the
retirement sweep, and the "refuse to clobber a hand-authored note" blocker — existed
before `eaafcbe72` and can be read back out of `git show eaafcbe72^:<path>` rather than
reinvented.

## Target Output

For a managed project with long notes and a configured glossary (numbering supplied by
the existing `number_agent_document_sections()` pass):

```markdown
## 1. Tier 1 (short-term) Memory

The following memories contain core (always loaded) context:

### 1.1 Build & Run Commands (build_and_run)

...

### 1.3 Glossary Terms (glossary)

Run `sase glossary read <term> [<term> ...] -r "<why>"` before relying on any of these
SASE terms; it prints each term's definition plus every term those definitions depend
on. Pass every term you need in one command — one batched read costs far fewer tokens
than one read per term, because terms shared between definitions are printed once. Terms
are separated by semicolons; aliases follow in parentheses.

**GLOSSARY TERMS:** Agent Clan; Agent Hood (hood, agent neighborhood); Stitch

### 1.4 Code Conventions and Gotchas (gotchas)

...

## 2. Tier 2 (long-term) Memory

The below files contain detailed reference material. When working in their domain, you
MUST use your `/sase_memory_read` skill to review their contents. Do not read canonical
memory files directly.

### 2.1 `sase/memory/cli_rules.md`

Read anytime new CLI subcommands or options are added.

### 2.2 `sase/memory/sase_beads.md`

Read before creating, updating, closing, or querying sase beads.
```

And the note written to disk:

```markdown
---
type: short
parent: AGENTS.md
sase_generated: glossary
---

# Glossary Terms

Run `sase glossary read <term> [<term> ...] -r "<why>"` before relying on any of these
SASE terms; it prints each term's definition plus every term those definitions depend
on. Pass every term you need in one command — one batched read costs far fewer tokens
than one read per term, because terms shared between definitions are printed once. Terms
are separated by semicolons; aliases follow in parentheses.

**GLOSSARY TERMS:** Agent Clan; Agent Hood (hood, agent neighborhood); Stitch
```

## Decisions To Confirm At Approval

Each of these is a one-or-two-line change if the answer is different.

1. **Tier 1 position is alphabetical.** `_short_memory_bodies()` returns
   `dict(sorted(bodies.items()))`, so the note lands wherever `sase/memory/glossary.md`
   sorts — in this repo that is between `feature_flags.md` and `gotchas.md` (section
   `1.3`). This plan keeps that deterministic contract rather than special-casing the
   glossary to the front or back of Tier 1.
2. **Note title is `Glossary Terms`**, so the inlined header reads
   `### N.M Glossary Terms (glossary)`, matching the retired Tier 2 heading text.
3. **The `**GLOSSARY TERMS:**` bold label is kept.** It is now slightly redundant with
   the section heading, but it is the string agents and tests key on, and it separates
   the roster from the instruction paragraph. Dropping it is a one-line template edit.
4. **The instruction paragraph is copied verbatim** from `_GLOSSARY_TERMS_INTRO` in
   `src/sase/amd/_memory.py`. No prose rewrite is in scope.
5. **`memory.glossary` stays project-only.** The home root ignores home-level glossary
   config today and continues to; no home root ever gets a `glossary.md` note.

## Empty / Collision / Migration Rules

- **Terms configured** → write `sase/memory/glossary.md` with the marker frontmatter and
  inline it into Tier 1.
- **No `memory.glossary`, or an empty/absent glossary** → write nothing, and delete a
  _marked_ leftover note at that path (`sase_generated: glossary` in frontmatter). This
  is the existing `_retired_glossary_note_paths()` behavior, now conditional on whether
  terms are configured.
- **An existing note at that path with a `sase_generated: glossary` marker** (including
  a stale `type: long` copy from before `eaafcbe72`) → overwritten with the new
  short-note content. This is what makes the migration converge in one pass.
- **An existing note at that path _without_ the marker** (hand-authored) and terms are
  configured → **blocker**, do not overwrite. Restore the pre-`eaafcbe72`
  `_glossary_collision_blocker()` message: refusing to overwrite an unmarked glossary
  memory note; migrate its content into `memory.glossary` entries in `sase.yml` or
  remove it before initializing. Without configured terms the unmarked note keeps
  behaving as an ordinary hand-authored note, exactly as today.
- **Long notes exist / do not exist** in Tier 2 → unchanged from today except the H3
  wrapper: entries plus the intro paragraph when there is at least one top-level long
  note, and an entirely empty `tier2_entries` when there are none.

## Files And Changes

### 1. `src/sase/main/init_memory/templates/memory-sase-glossary.template.md` (new)

Packaged Jinja template for the note body, modeled on
`memory-sase-task-types.template.md` (single unwrapped paragraph per line; prettier
reflows on format). One required variable, `glossary_term_entries`, which receives the
already-joined `Term (alias, alias); Term; …` string:

```markdown
# Glossary Terms

Run `sase glossary read <term> [<term> ...] -r "<why>"` before relying on any of these
SASE terms; it prints each term's definition plus every term those definitions depend
on. Pass every term you need in one command — one batched read costs far fewer tokens
than one read per term, because terms shared between definitions are printed once. Terms
are separated by semicolons; aliases follow in parentheses.

**GLOSSARY TERMS:** {{ glossary_term_entries }}
```

Like `memory-sase-task-types.template.md` and `memory-sase-beads.template.md`, this gets
**no** `memory.*` override key in `sase.yml`. Only `sase_template` and `readme_template`
are overridable; do not add a third.

### 2. `src/sase/main/init_memory/root_rendering.py` — render the note

- Add `MEMORY_SASE_GLOSSARY_TEMPLATE_FILENAME` and
  `_MEMORY_SASE_GLOSSARY_TEMPLATE_VARS = frozenset({"glossary_term_entries"})` alongside
  the existing template constants.
- Rename `retired_glossary_memory_relative_path()` →
  `generated_glossary_memory_relative_path()` (same value,
  `CANONICAL_MEMORY_RELATIVE_ROOT / "glossary.md"`), and update its docstring: the path
  is generated again, not retired.
- Move `_render_glossary_term_entry()` here from `src/sase/amd/_memory.py` verbatim
  (with its `md_escape` import). It already handles the no-alias case; alias suppression
  for derivable plurals happens upstream in `build_glossary_catalog()`'s
  `display_aliases` and must not be reimplemented.
- Add
  `render_generated_glossary_memory_body(glossary_terms) -> tuple[str | None, str | None]`
  following `render_generated_task_types_memory_body()` exactly: join the entries with
  `"; "`, render the template, run `format_generated_memory_markdown()`, then
  `validate_short_memory_structure()` and return a
  `f"packaged {MEMORY_SASE_GLOSSARY_TEMPLATE_FILENAME}: {structure_error}"` blocker on
  failure.
- Add `_generated_glossary_memory_content(body)` =
  `apply_memory_frontmatter(body, note_type="short", parent=AGENTS_PARENT, extra={GENERATED_GLOSSARY_MARKER_KEY: GENERATED_GLOSSARY_MARKER_VALUE})`.
  `apply_memory_frontmatter()` already supports `extra`; the marker keys stay defined in
  `init_memory/glossary.py`.
- `generated_short_notes(...)` gains a `generated_glossary_body: str | None = None`
  parameter and includes the glossary path in the returned mapping only when it is not
  `None`.
- `render_expected_memory_files(...)` gains
  `generated_glossary_body: str | None = None`. When set, append a `MemoryExpectedFile`
  for the note (detail: `generated glossary memory note`, default
  `stale_operation="update"`) **and** add it to `note_overlay` so the regenerated
  `sase/memory/README.md` lists it with correct stats on the first pass.

### 3. `src/sase/main/init_memory/root_planning.py` — plan the note

- Change `_retired_glossary_note_paths(root)` to
  `_retired_glossary_note_paths(root, *, glossary_terms)` and return `()` when
  `glossary_terms` is not `None` and has terms — the path is generated in that case, not
  retired, and must therefore stay out of `excluded_note_paths` so discovery and the
  README see it.
- Restore `_glossary_collision_blocker(root, *, glossary_terms)` from
  `git show eaafcbe72^:src/sase/main/init_memory/root_planning.py` (lines ~226-244),
  adapted to take `glossary_terms`. Return its blocker early from
  `memory_root_context()`, before any rendering, exactly as the old code did.
- In `memory_root_context()`, render the glossary body (when terms are configured) next
  to the existing `render_generated_sase_memory_body()` /
  `render_generated_task_types_memory_body()` calls, returning a blocker context on
  error, and pass it into both `generated_short_notes(...)` and
  `render_expected_memory_files(...)`.
- **Stop passing `glossary_terms` to `_amd_sync_plan()`** and drop the parameter from
  `_amd_sync_plan()`. `glossary_terms` still arrives at `memory_root_context()` and
  `plan_memory_root()` from the handler; it is now consumed to build a short note rather
  than forwarded into the AMD render.

### 4. `src/sase/amd/_memory.py` — delete the Tier 2 glossary block, flatten the wrapper

Deletions (verify with a repo-wide grep that nothing else imports them; symvision will
fail the build on a leftover unused private symbol):

- `_GLOSSARY_TERMS_TITLE`, `_GLOSSARY_TERMS_HEADING`, `_GLOSSARY_TERMS_LABEL`,
  `_GLOSSARY_TERMS_INTRO` (its text moves into the new packaged template),
  `_render_glossary_term_entry()`, `_render_glossary_terms_section()`.
- `_LONG_MEMORY_FILES_TITLE`, `_LONG_MEMORY_FILES_HEADING`,
  `_render_long_memory_files_section()`.
- `_heading_has_title()` and `_rendered_has_heading()` — both exist only to serve the
  two heading self-checks being removed.
- The `from sase.agents_sync.rendering_markdown import md_escape` and
  `from sase.main.init_memory.glossary import ProjectGlossaryTerms` imports. Removing
  the latter also removes an `amd` → `main.init_memory` import edge, which is a small
  layering win worth noting in the commit message.

Keep `_LONG_MEMORY_FILES_INTRO` (rename it to something wrapper-free, e.g.
`_LONG_MEMORY_INTRO`) and build Tier 2 as: `""` when there are no rendered entries,
otherwise `f"{_LONG_MEMORY_INTRO}\n\n{entries}"`.

- Drop the `glossary_terms` parameter from `_render_managed_agents()` and from
  `plan_amd_memory_sync()`, and update `plan_amd_memory_sync()`'s docstring, which
  currently documents the `Glossary Terms` H3 behavior.
- Replace the two removed heading self-checks with one that still catches a project
  `agents_template` override that drops `{{ tier2_entries }}`: when
  `top_level_long_notes` is non-empty, the rendered document must contain
  `_LONG_MEMORY_INTRO`'s first sentence. Keep the existing Tier 1/Tier 2 anchor and
  memory-path parity checks untouched.

### 5. `src/sase/memory/notes.py` — long notes render at H3

`render_long_memory_sections()` (line ~433) emits
`#### \`{note.relative_path}\``. Change to `###
\`{note.relative_path}\``and update the docstring ("Tier 2 H4 subsections" → H3). Confirm before editing that`src/sase/amd/_memory.py`
is still the only production caller.

Do **not** touch `_render_memory_notes()` in
`src/sase/main/init_memory/root_rendering.py` (the
`### \`sase/memory/<note>.md\``sections in the generated`sase/memory/README.md`) or `render_children_section()`/`_render_memory_note_references()`(the`**\`path\`**`
shape). Both look similar and are unrelated.

### 6. `src/sase/amd/_agents_doc.py` — parsing stays backward compatible

`_LONG_MEMORY_SECTION_RE` already accepts `^#{3,4}\s+…`, so new H3 entries and
already-committed H4 entries both parse. No regex change needed — verify, do not assume.

Update `collect_long_memory_entries()`'s docstring, which cites "a trailing
`### Glossary Terms` section" as the motivation for its heading boundary. The boundary
itself must stay: it is what stops one note's description from swallowing the next
entry's heading, and those parsed descriptions get written back into note frontmatter by
`_long_memory_description_updates()`.

### 7. `src/sase/main/init_memory/glossary.py`

- Update the module docstring and `is_generated_glossary_memory_content()`'s docstring:
  the marker is live again, not "retired".
- `load_project_glossary_terms()` and `ProjectGlossaryTerms` are unchanged.

`root_application.py` and `init_memory_handler.py` need no change — they already thread
`glossary_terms` through to `plan_memory_root()`.

## Tests

Rewrite `tests/main/test_init_memory_glossary.py` around the new shape. Keep the helper
scaffolding (`_setup_project`, `_tier1_memory`, `_tier2_memory`, `_normalized`,
`_glossary_note_path`, `_marked_glossary_note`) and re-point the assertions:

- Terms configured → `sase/memory/glossary.md` is created with `type: short`,
  `parent: AGENTS.md`, and `sase_generated: glossary`; Tier 1 contains
  `Glossary Terms (glossary)` and the `**GLOSSARY TERMS:** …` roster; Tier 2 contains no
  `Glossary Terms` heading and no `GLOSSARY TERMS` text.
- Alias rendering: `Agent Clan (clan)`, semicolon separation, `md_escape`, and the
  derivable-plural suppression case (`Patch` with only alias `patches` renders bare) —
  these move from Tier 2 assertions to Tier 1 / note-body assertions unchanged in
  substance.
- Compactness: the roster stays one paragraph (no `- ` bullets) as the term count grows.
- README: the generated `sase/memory/README.md` lists `sase/memory/glossary.md` with
  `Type: short` on the **first** pass (this is what the `note_overlay` addition in item
  2 buys; assert it explicitly).
- No glossary configured → no note, no `GLOSSARY TERMS` anywhere, and a _marked_
  leftover note is deleted (keep the existing test).
- Marked leftover note **with** terms configured → overwritten in place, not deleted;
  second pass is clean. This replaces
  `test_memory_init_deletes_stale_generated_glossary_note_even_with_configured_terms`.
- Migration from the `eaafcbe72` era: a marked `type: long` note plus configured terms
  converges to the new `type: short` note in one pass, and `run_memory(check=True) == 0`
  afterwards.
- Unmarked hand-authored note **with** terms configured → `plan.actions == ()` and a
  blocker naming the path. This replaces
  `test_memory_plan_preserves_unmarked_glossary_note_as_ordinary_long_note`, whose
  no-terms half should be kept as a separate test.
- Idempotence and provider-shim byte equality (keep as-is).
- Home glossary config still ignored (keep as-is).
- Prettier/format fixpoint: `format_generated_memory_markdown(note) == note` and
  `format_generated_memory_markdown(agents) == agents` for a glossary whose roster line
  is long enough to force a wrap.

Update for the Tier 2 flattening (all currently assert the H3 wrapper or H4 entries):

- `tests/test_memory_notes.py` (~236-296) — `render_long_memory_sections` now emits
  `### \`sase/memory/<note>.md\``.
- `tests/main/test_init_memory_markdown_templates.py` (~195-280) — drop the two
  glossary-in-Tier-2 tests, and rewrite
  `test_long_memory_files_section_precedes_glossary_terms` as a Tier 2 shape test: intro
  paragraph directly under the H2, then `### 2.1 \`sase/memory/parent.md\``, and nothing
  between the H2 and the intro.
- `tests/main/test_init_memory_managed_agents.py` (~144-150, ~281, ~475, ~558) — H4 → H3
  and drop the `Long-Term Memory Files` assertions. Lines ~310-345
  (`_existing_agents_long_descriptions` legacy shapes) already use H3 and must stay
  green unchanged — that is the backward-compatibility check.
- `tests/main/test_memory_agent_docs_list.py` (~71),
  `tests/main/test_init_onboarding_memory.py` (~232),
  `tests/main/test_init_memory_validation.py` (~147),
  `tests/main/test_init_memory_plan.py` (~43) — `#### 2.1.1 \`…\``→`### 2.1
  \`…\``. Leave every `sase/memory/README.md` assertion at H3; the README shape is
  unchanged.
- `tests/main/test_init_memory_agents_templates.py` (~55-130) — already H3; keep, and
  add an H4 case to pin that legacy already-generated documents still parse.
- `tests/main/test_memory_read_list.py` (~196), `tests/main/test_memory_cli_show.py`
  (~23), `tests/test_memory_notes.py` (~224) also contain the same intro sentence, but
  they assert `render_children_section()`'s `## Children` block, not Tier 2. Leave them
  alone — the two renderers share prose and are easy to confuse.

Add:

- Numbering regression over one generated document covering
  `1.N Glossary Terms (glossary)` in Tier 1 and `2.1` / `2.2` note entries in Tier 2.
- Parse round-trip: an `AGENTS.md` whose Tier 2 has the intro paragraph as direct H2
  body content parses to exactly the expected long-memory entries, with the intro
  absorbed into no description.
- `tests/test_xprompt_memory_loader.py` already fixtures a `sase/memory/glossary.md`;
  confirm it still passes (it constructs its own note and does not depend on the
  generator).

## Docs

- `docs/init.md` (~205-225) — rewrite both paragraphs: `memory.glossary` now generates a
  Tier 1 note again (with the marker, the collision blocker, and the migration from a
  marked long note); and Tier 2 no longer has a `Long-Term Memory Files` H3, so the
  per-note subsections are H3 directly under the H2 and the instruction paragraph is
  omitted with them when a root has no top-level long notes.
- `docs/memory.md` (~44-47 and the `## Glossary` section at ~136-145) — the "Project
  glossary terms are not a memory note and have no `#memory/glossary` form" claim is now
  **false**. Glossary terms are a generated Tier 1 note, so `#memory/glossary` becomes a
  valid xprompt reference and the note shows up in `sase memory list`. Note that
  `sase memory read glossary.md` still fails by design — `read` rejects `short` notes as
  always-loaded context (`src/sase/memory/read_log.py` ~221) — and that full definitions
  still come from `sase glossary read`.
- `docs/xprompt.md` (~1197-1199) — same correction: `#memory/glossary` exists again.
- `docs/configuration.md` (~494-496) — `{{ tier2_entries }}` no longer renders two H3
  sections; it renders the long-memory instruction paragraph plus the per-note H3
  subsections. Also check the `memory.glossary` schema section and the
  `sase memory init` command description for the same stale claim.
- `src/sase/main/init_memory/templates/memory-README.template.md` (~12, ~26) — verify
  the Tier 1/Tier 2 wording still reads correctly; likely no edit needed.

## Regeneration

The generated instruction files committed in this repo must be refreshed in the same
change, as `eaafcbe72` and `538dec9fc` did:

```bash
sase memory init --check   # expect drift
sase memory init
```

Expected diff in this repo:

- **new** `sase/memory/glossary.md` (the 30 terms currently configured under
  `memory.glossary` in `sase/sase.yml`);
- `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md` — glossary section
  moves from Tier 2 to Tier 1, Tier 2 renumbers from `2.1 Long-Term Memory Files` /
  `2.1.N` to `2.N`, and Tier 1 renumbers after the inserted glossary section;
- `sase/memory/README.md` — one new row plus updated totals.

No hand-authored `sase/memory/*.md` note body may change. If a long note's frontmatter
`description:` changes, that is a description-boundary parsing bug leaking, not an
expected edit — stop and fix it.

`sase memory init` also refreshes the managed **home** root (and its chezmoi source
counterparts). The home root has no glossary config, so its only change is the Tier 2
flattening. That lands outside this repo — call it out in the final report rather than
leaving it silent.

This regeneration is authorized: the user requested this generator change directly, and
`AGENTS.md` plus the four provider shims are generated artifacts of the code being
changed. No canonical memory note is hand-edited by this plan.

## Verification

```bash
just install
just check
```

Targeted lane while iterating:

```bash
just test tests/main/test_init_memory_glossary.py \
  tests/main/test_init_memory_markdown_templates.py \
  tests/main/test_init_memory_managed_agents.py \
  tests/main/test_init_memory_agents_templates.py \
  tests/main/test_init_memory_plan.py \
  tests/main/test_init_memory_validation.py \
  tests/main/test_init_memory_committed_drift.py \
  tests/main/test_memory_agent_docs_list.py \
  tests/main/test_init_onboarding_memory.py \
  tests/test_memory_notes.py
```

`tests/main/test_init_memory_committed_drift.py` is the gate that proves the committed
tree and the generator agree; it must be green _after_ regeneration, not before.

Before landing, hand `just check-full` to `/sase_monitor` with a `--next` action — it
routinely outruns a single agent turn and must never be run inline. Prettier
(`just fmt-md-check`) covers every generated Markdown file, so the new note and the
regenerated shims must be prettier fixpoints.

## Acceptance Criteria

- `sase memory init` writes `sase/memory/glossary.md` with `type: short`,
  `parent: AGENTS.md`, `sase_generated: glossary`, and the compact roster body whenever
  a SASE-managed project configures `memory.glossary`.
- That note is inlined into Tier 1 as `### N.M Glossary Terms (glossary)` in `AGENTS.md`
  and every provider shim, and appears in the generated `sase/memory/README.md`.
- No `Glossary Terms` heading and no `GLOSSARY TERMS` text appears anywhere in Tier 2.
- `Long-Term Memory Files` appears nowhere in the repo; Tier 2 renders the instruction
  paragraph as direct H2 body content followed by `### N.M \`sase/memory/<note>.md\``
  subsections, and renders empty when a root has no top-level long notes.
- `parse_amd_agents_document()` parses both the new H3 entries and legacy H4 entries and
  never absorbs the Tier 2 intro paragraph into a note description.
- Migration converges in one pass from every prior state: no note, marked short note,
  marked long note, and no-terms-with-marked-note. An unmarked note plus configured
  terms produces a blocker and zero actions.
- `sase memory init` is idempotent and `--check` clean on a second pass; all generated
  files are `format_generated_memory_markdown` and prettier fixpoints.
- Committed `AGENTS.md`, four provider shims, `sase/memory/README.md`, and the new
  `sase/memory/glossary.md` are regenerated; docs updated; `just check` green and
  `just check-full` green via `/sase_monitor`.

## Risks

- **Silent clobber of a hand-authored note (highest).** `sase/memory/glossary.md` is a
  path a user may already own in some project. The unmarked-note blocker is the only
  thing standing between this change and destroying that content — implement it before
  the write path, and test it.
- **First-pass README staleness.** If the generated note is added to `expected_files`
  but not to `note_overlay`, the README misses it until a second `sase memory init`.
  That breaks `--check` idempotence in a way that only shows up on a fresh root; the
  explicit first-pass README assertion is the guard.
- **`excluded_note_paths` leakage.** The retirement sweep currently feeds the glossary
  path into `excluded_note_paths`, which suppresses it from discovery and the README.
  Making that conditional on "no terms configured" is easy to get backwards; a
  terms-configured run must produce an empty exclusion for this path.
- **Description pollution.** Parsed Tier 2 descriptions are written back into note
  frontmatter. The Tier 2 intro paragraph now sits between the H2 and the first entry
  rather than under an H3, so any regression in `collect_long_memory_entries()`'s
  boundary handling would land in a real note's `description:`. The regeneration diff
  review is the backstop.
- **Downstream repos.** Plugin repos (`sase-github`, `sase-telegram`, `sase-nvim`,
  `sase-research-artifacts`) pick up both changes on their own next `sase memory init`.
  Out of scope; the parser accepts both entry depths, so nothing breaks in the meantime.
- **Tier 1 token cost.** The roster moves from Tier 2 to Tier 1, but both live inside
  the always-loaded `AGENTS.md`, so the real token delta is roughly zero — the note file
  itself is the only new bytes on disk.
