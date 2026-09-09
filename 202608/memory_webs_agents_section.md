---
tier: epic
status: done
title: Memory webs get their own agent-instruction section
goal: 'Generated agent instruction files render three tier-free sections — core memory,
  memory webs, reference memory — with every memory web inlined as its own numbered
  subsection of the memory-webs section, and no "Tier 1"/"Tier 2" memory vocabulary
  anywhere in the repo, its docs, its generated output, or its linked repos.

  '
phases:
  - id: webkind
    title: Web descriptors stop declaring a rendering tier
    depends_on: []
    size: medium
    description: "webkind: delete MemoryWeb.rendering_type, stop reading type:/parent:
      from web descriptors, strip both keys on init, always inline every web descriptor,
      and update every consumer surface that displayed a web's rendering tier.

      "
  - id: section
    title: Tier-free H2 sections and the new Memory Webs section
    depends_on:
      - webkind
    size: medium
    description: "section: rename the generated H2 anchors to Core Memory / Memory Webs
      / Reference Memory, rename the AGENTS template variables, render web descriptors
      into the new middle section, and teach the AGENTS.md parser and TUI rail the
      three-group shape.

      "
  - id: docs
    title: Documentation, memory notes, and regenerated artifacts
    depends_on:
      - section
    size: medium
    description:
      "docs: rewrite the tier vocabulary in docs/, the generated sase.md and README
      memory templates, the affected glossary strands, and a new decision record, then
      regenerate and commit every generated artifact here and in the chezmoi linked
      repo."
proposed_by: bbugyi200.athena.0g6.w0
bead_id: sase-vk
create_time: 2026-09-09 19:50:54
---

- **PROMPT:**
  [prompts/202608/memory_webs_agents_section.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/memory_webs_agents_section.md)
- **BEAD:**
  [sase-vk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-vk/README.md)

# Plan: Memory webs get their own agent-instruction section

## Goal

Today a generated agent instruction file has exactly two H2 sections,
`## 1. Tier 1 (core) Memory` and `## 2. Tier 2 (reference) Memory`, and a memory web
descriptor picks which one it lands in with `type: core` / `type: reference`
frontmatter.

After this epic:

1. Generated agent instruction files have up to three H2 sections, named without any
   tier vocabulary: `## Core Memory`, `## Memory Webs`, `## Reference Memory`, in that
   order.
2. Every memory web renders in the memory-webs section as its own numbered H3 subsection
   — for the `sase` project that means `### N.1 Decisions (decisions)`,
   `### N.2 Glossary Terms (glossary)`, `### N.3 Task Bead Types (task_types)`.
3. A web descriptor no longer declares `type:` (or `parent:`). Kind — note, web, or
   strand — decides where a memory renders; a web always renders in the memory-webs
   section.
4. The memory-webs section is omitted entirely when a root has no webs, so a root
   without webs (the chezmoi-managed home root is one) simply renders core memory then
   reference memory. Because the section names are no longer numbered concepts, a
   missing middle section is not a hole in a numbering scheme.
5. No "Tier 1" / "Tier 2" memory vocabulary survives in this repo, its generated output,
   or its linked repos.

Section numbers keep coming from the existing document-wide numbering pass in
`src/sase/amd/_section_numbers.py`; they are positional, not semantic, so the
memory-webs section is `2` when it is present and reference memory shifts to `2` when it
is not.

Unrelated uses of the word "tier" must be left alone. The TUI agent loader's Tier 1/Tier
2 lazy-loading vocabulary (`src/sase/ace/tui/actions/agents/*`,
`src/sase/ace/tui/models/agent_loader.py`, `docs/perf_runbook.md`), the VCS provider
resolution tiers (`docs/vcs.md`), the editor search tiers (`docs/editor.md`), and
plan/bead tiers (`tier: tale|epic`) are all different concepts. Historical SDD tale
files under `sdd/` and `CHANGELOG.md` entries are immutable history and must not be
rewritten.

## Current shape (read this before starting)

- `src/sase/amd/templates/AGENTS.template.md` is the packaged template. It takes
  `{{ title }}`, `{{ tier1_sections }}`, `{{ tier2_entries }}` and hard-codes the two H2
  headings. `src/sase/amd/templates/AGENTS.minimal.template.md` takes `{{ title }}` and
  `{{ tier1_sections }}`.
- `src/sase/amd/_template.py` declares the required variable sets and calls
  `number_agent_document_sections` after rendering.
  `sase.mdtemplates.render_markdown_template` is strict: a template missing a required
  variable, or naming an unknown one, is a blocker.
- `src/sase/amd/_memory.py` builds both section bodies. `_short_memory_bodies` collects
  every `type: core` note body (overlaid with generated bodies) sorted by
  `(priority, path)`; `_render_managed_agents` inlines each through
  `sase.amd.inline_memory.inline_memory_section` and renders reference notes through
  `sase.memory.notes.render_long_memory_sections`. It then re-parses its own output and
  refuses to return content whose structure does not match what it intended.
- `src/sase/amd/_agents_doc.py` owns the structural regexes (`_CORE_SECTION_RE`,
  `_REFERENCE_SECTION_RE`) used both for that self-check and for recovering existing
  reference-note descriptions out of an `AGENTS.md` already on disk.
- `src/sase/memory/web/frontmatter.py::parse_web_descriptor` requires
  `type: core|reference` on a descriptor, rejects `priority:` on a `reference` web, and
  stores the result as `MemoryWeb.rendering_type` (`src/sase/memory/web/models.py`).
- `src/sase/main/init_memory/root_planning.py::_memory_web_root_plan` renders each web's
  managed roster region, then — only for `rendering_type == "core"` — feeds the
  roster-marker-stripped descriptor body into the generated core-note bodies so it
  inlines like any other core note.
- A web descriptor is _also_ discovered as a flat note by
  `sase.memory.notes.discover_memory_notes`, because it is a flat `*.md` file under the
  memory root. That is how it reaches the core/reference pipelines today, and it is why
  dropping `type:` needs deliberate handling in several places.
- `src/sase/main/init_memory/root_rendering_task_types.py` renders the fully
  SASE-generated `task_types` web descriptor and stamps its frontmatter with
  `apply_memory_frontmatter(note_type="core", parent=AGENTS_PARENT, extra={"web": True, ...})`.

## Phase `webkind`: Web descriptors stop declaring a rendering tier

### Outcome

`MemoryWeb` no longer carries a rendering tier, a web descriptor's `type:` and `parent:`
frontmatter are ignored and then stripped by `sase memory init`, and every web
descriptor is inlined into agent instructions regardless of what it used to declare. The
generated document still has only the two existing H2 sections in this phase — web
descriptors keep landing in the core-memory section. This phase is deliberately a no-op
on document _structure_ so that the structural work in `section` starts from a clean
substrate.

### Model and parser

- Delete the `rendering_type` field from `MemoryWeb` in `src/sase/memory/web/models.py`.
- In `parse_web_descriptor` (`src/sase/memory/web/frontmatter.py`):
  - Stop reading `type:`. Do not error on it; ignore it. A descriptor that still
    declares `type: core` on disk must keep parsing so discovery works before the
    migration below has run.
  - Ignore `parent:` the same way.
  - Keep `priority:` and keep rejecting a non-negative-integer violation, but delete the
    `"priority is only meaningful on core memory webs"` branch. `priority` now means
    "ordering among webs"; it is valid on any web.
  - Keep `web:`, `description:`, `roster:`, `roster_label:`, `strand_noun:`, `closure:`,
    and `metadata:` exactly as they are.
- `normalize_memory_note_type` is still needed for flat notes; only the web descriptor's
  use of it goes away.

### Frontmatter migration on init

`sase memory init` already rewrites each descriptor to synchronize its managed roster
region (`render_web_descriptor_with_roster`, which preserves the raw frontmatter bytes
via `replace_web_body`). Extend that rewrite so the descriptor's frontmatter is also
normalized: `type:` and `parent:` are removed, every other key is preserved in its
existing order, and the body is untouched. Put the frontmatter rewrite next to the
roster rendering in `src/sase/memory/web/roster.py` (or a small sibling module) so
`render_web_descriptor_with_roster` returns fully canonical descriptor content, and both
`sase memory init` and `sase memory init --check` see the same expected bytes. `--check`
must report the drift rather than silently pass.

For the generated `task_types` descriptor, change
`src/sase/main/init_memory/root_rendering_task_types.py::_render_task_types_descriptor_content`
to stop calling `apply_memory_frontmatter(note_type="core", parent=AGENTS_PARENT, ...)`
and instead emit
`render_frontmatter_block({"web": True, "roster": "list", "strand_noun": "task type"})`
so the generated descriptor matches the same canonical shape. Keep
`validate_short_memory_structure` on the rendered body: a descriptor body is inlined, so
it must still be exactly one H1 with no headings deeper than H3.

Note that `is_generated_task_types_memory_content` and the retirement helpers in
`root_planning.py` recognize a generated descriptor partly by `web: true` frontmatter;
make sure they still recognize the new shape, and that a pre-migration descriptor
carrying `type: core` is still recognized so it converges instead of being orphaned.

### Always inline every web descriptor

In `src/sase/main/init_memory/root_planning.py::_memory_web_root_plan`, drop the
`if web.rendering_type == "core":` guard so every discovered web contributes its
roster-marker-stripped body to the inlined bodies map. Keep the
`strip_managed_roster_markers` call and keep passing `web.priority`.

Apply `sase.amd.inline_memory.validate_short_memory_structure` to every web descriptor
body and report a blocker naming the descriptor path when it fails, so a web whose body
cannot be inlined fails init with an actionable message instead of producing a malformed
document. `_short_memory_structure_blockers` in `src/sase/amd/_memory.py` already does
this for whatever lands in the inlined-bodies map; confirm web bodies still flow through
it in this phase and that the blocker text names the descriptor.

### Teach the flat-note pipeline about web descriptors

A parsed `MemoryNote` needs to be able to say "I am a web descriptor". Add that to
`sase.memory.notes`: either a `MemoryNote.is_web_descriptor` property reading
`self.frontmatter.get("web") is True`, or an explicit field populated in
`parse_memory_note_text`. Then:

- `src/sase/memory/inventory_reachability.py::_is_short_memory_note` currently requires
  `note.type == "core"`. It must also accept a web descriptor, otherwise a typeless
  descriptor stops counting as reachable through its inlined `### Title (stem)` header
  and `unreferenced_memory_files` turns it into a blocker. Rename the helper to
  something honest (for example `_is_inlined_memory_note`) while you are there.
- `src/sase/amd/_memory.py::_memory_frontmatter_updates` must skip web descriptors
  explicitly rather than relying on `note.type not in {"core", "reference"}` falling
  through, so init never re-stamps `type:`/`parent:` onto a descriptor it just stripped.
- `_short_memory_bodies` and `_long_memory_descriptions` / `top_level_long_notes` in the
  same module must exclude web descriptors explicitly. Web bodies arrive through the
  generated-bodies overlay from `root_planning`, not through note discovery; a
  descriptor must never be double-counted or listed as a reference note.
- `src/sase/memory/read_log.py` currently rejects a core note with "is always-loaded
  context and cannot be read with this command" and everything else with "memory file is
  not a reference memory note". A web descriptor now always falls into the second,
  misleading branch. Add an explicit web-descriptor branch with an actionable message
  that points at the strand selector, along the lines of "sase/memory/glossary.md is an
  always-loaded memory web descriptor; read its strands with
  `sase memory read glossary:<keyword>`". Reading a bare web name and a `web:keyword`
  selector must keep working unchanged.

### Consumer surfaces that displayed the rendering tier

Every one of these reads `MemoryWeb.rendering_type` and must stop:

- `src/sase/memory/web/cli.py`: drop the `Renders` column from the
  `sase memory web list` table and drop `"rendering_type"` from `_web_summary_json`.
  This is a JSON output change; record it in the changelog-worthy commit message and in
  `docs/memory.md` during the `docs` phase.
- `src/sase/memory/cli_list.py`: drop the `Renders` column from the memory-webs panel of
  `sase memory list`.
- `src/sase/memory/selector_render.py`: drop `"rendering_type"` from the rendered
  selector payload.
- `src/sase/ace/tui/memory_panel_catalog.py::memory_strand_note` builds a pseudo-note
  with `type=web.rendering_type`. A strand is not core and not reference; give the
  pseudo-note a typeless value and make sure the panel's "invalid type" warning path
  does not fire for it.
- `src/sase/ace/tui/modals/memory_panel_web_rendering.py`: remove the `("Renders", ...)`
  row from both the web property grid and the strand property grid.
- `src/sase/ace/tui/modals/memory_panel_rendering.py`: `_build_web_row_text` picks
  `_TIER1_MARK`/`_TIER2_MARK` from `rendering_type`. Give a web its own marker glyph
  instead (see the `section` phase for the rail grouping that goes with it). Also make
  sure `note.type_source == "missing"` no longer renders the `⚠` invalid-frontmatter
  marker for a web descriptor row, since a descriptor legitimately has no `type:` now.
- `src/sase/ace/tui/modals/memory_panel_actions.py::action_edit_note` opens
  `MemoryNoteFormModal` for the selected row. A web descriptor row must not open the
  flat note form (it would offer a core/reference choice that no longer exists). Refuse
  it with a notification in the same style as the existing strand refusal ("memory
  strands are edited from their source file"), for example "memory web descriptors are
  edited from their source file".
- `src/sase/ace/tui/modals/memory_panel_add.py`: the flat-note form's field is labeled
  `Tier` with options `core — Tier 1, always loaded` / `reference — Tier 2`. Relabel to
  `Type` with `core — always loaded` / `reference — read on demand`. The form itself
  still only creates flat notes; that is unchanged.

`symvision` will flag anything left unused after `rendering_type` goes away; read
`sase memory read symvision.md` before deleting a symbol it reports.

### Verification for `webkind`

- `just check`. If it escalates or the change reaches the broadening set, run
  `just check-full` through `/sase_monitor`.
- Run `sase memory init` in this repo, confirm it converges in a single pass (a second
  `sase memory init --check` is clean), and confirm `sase/memory/glossary.md`,
  `sase/memory/decisions.md`, and `sase/memory/task_types.md` lost their `type:` (and
  `parent:`) frontmatter while keeping their bodies and rosters byte-identical apart
  from that.
- Confirm the regenerated `AGENTS.md` in this repo still contains the three web
  subsections (still inside the Tier 1 section in this phase) and that the provider
  shims match it byte-for-byte.
- `sase memory read glossary.md -r "..."` must fail with the new actionable message;
  `sase memory read glossary:stitch -r "..."` must still succeed.
- Memory-panel PNG snapshots may move because of the new web marker. Run
  `just test-visual`, inspect `.pytest_cache/sase-visual/`, and accept intended changes
  with `--sase-update-visual-snapshots`.
- Memory note edits in this phase go through `/sase_memory_write` first.

## Phase `section`: Tier-free H2 sections and the new Memory Webs section

### Outcome

The generated document has `## Core Memory`, an optional `## Memory Webs`, and
`## Reference Memory`, with web descriptors rendered in the middle section.

### Template and template variables

Rewrite `src/sase/amd/templates/AGENTS.template.md` as:

```
# {{ title }}

## Core Memory

The following memories contain core (always loaded) context:

{{ core_sections }}

{{ web_sections }}

## Reference Memory

{{ reference_entries }}
```

Rename `tier1_sections` → `core_sections` and `tier2_entries` → `reference_entries`, and
add `web_sections`. Update `_MANAGED_TEMPLATE_VARS`, `_MINIMAL_TEMPLATE_VARS`, and the
`render_agents_template` signature in `src/sase/amd/_template.py`
(`AGENTS.minimal.template.md` takes `title` and `core_sections`).

`web_sections` carries the **entire** memory-webs section including its `## Memory Webs`
heading, and is the empty string when the root has no webs. That is what makes the
section disappear cleanly for a webless root: `format_generated_memory_markdown` in
`src/sase/main/init_memory/formatting.py` collapses runs of blank lines to one, so an
empty variable between two sections leaves no artifact. This mirrors how
`reference_entries` already carries the reference-memory instruction paragraph as body
content rather than having the template repeat it.

There are no configured `agents_template` / `agents_minimal_template` overrides anywhere
today (checked in this repo's `sase/sase.yml`, `~/.config/sase/sase.yml`, and the
chezmoi source tree), so renaming the required variables needs no compatibility alias.
The strict renderer already produces an actionable error naming the missing placeholders
if someone has an override.

### Section rendering

In `src/sase/amd/_memory.py`:

- Rename the local `tier1_sections` / `tier2_entries` plumbing to match the new variable
  names.
- Add a `web_sections` builder. Order webs by `(priority, slug)` so the ordering rule
  matches the core-note rule and `priority:` keeps a meaning. Render each web through
  the existing `inline_memory_section(relative_path, body)` so a web subsection has the
  same `### {H1 title} ({stem})` shape as an inlined core note — that shape is what
  `inlined_short_memory_files` matches for reachability, so reusing it keeps
  reachability working for free.
- Give the section a short generated intro paragraph, defined as a module constant next
  to `_LONG_MEMORY_INTRO`, along the lines of:

  > Each memory web below is a keyed collection. Its descriptor is always loaded, but a
  > strand's body is not: read strands on demand with your `/sase_memory_read` skill,
  > for example `sase memory read glossary:stitch -r "<why>"`.

  Keep it to one short paragraph — every word is paid for on every turn.

- Web descriptor bodies must no longer be merged into the core-note bodies map. Thread
  them from `root_planning` to `_memory.py` as their own collection (a
  `generated_web_notes` mapping of relative path → body + priority, parallel to
  `generated_short_notes`) rather than reusing the core-note overlay.
  `_short_memory_bodies` must not see them at all.
- Extend the post-render self-check in `_render_managed_agents` the way the two existing
  sections are checked: when webs exist, the rendered document must have a memory-webs
  section whose inlined paths equal the expected web paths, in order; when no webs
  exist, it must have no memory-webs section. Keep the existing core and reference
  assertions, updating their blocker strings to the new heading names.

### AGENTS.md parsing

In `src/sase/amd/_agents_doc.py`:

- Update `_CORE_SECTION_RE` to match `## [N. ]Core Memory` and `_REFERENCE_SECTION_RE`
  to match `## [N. ]Reference Memory`. **Keep the legacy `Tier 1 (core)` /
  `Tier 1 (short-term)` / `Tier 2 (reference)` / `Tier 2 (long-term)` spellings as
  additional alternatives.** These regexes are also used to recover reference-note
  descriptions from an `AGENTS.md` already on disk
  (`_existing_agents_long_descriptions`), so tolerating the old headings is what keeps
  the first upgrade run from losing descriptions. This is read-only tolerance for
  documents already written, not a behavior branch, so it needs no feature flag.
- Add a `_WEB_SECTION_RE` for `## [N. ]Memory Webs`, and add `has_web_section` plus
  `web_memory_paths` to `_AmdAgentsDocument`, parsed with the same `### Title (stem)`
  scan `_short_memory_paths` already implements (factor that scan into a shared helper
  rather than duplicating it).

### TUI rail grouping

`src/sase/ace/tui/memory_panel_catalog.py::_build_note_tree` currently emits core notes
first, then reference roots with children indented. Make it a three-group tree matching
the generated document: core notes (by `(priority, path)`), then webs (by
`(priority, slug)`), then reference roots alphabetically with their children. A web row
uses the marker introduced in `webkind` and keeps its existing expand/collapse and
strand nesting behavior. Update the docstring, which currently says "Order notes as Tier
1, then Tier 2 roots".

### Verification for `section`

- `just check`; `just check-full` through `/sase_monitor` before landing, since this
  phase touches the broadening set (template + renderer + parser + TUI).
- Run `sase memory init` and confirm this repo's `AGENTS.md` renders
  `## 1. Core Memory`, `## 2. Memory Webs` with `### 2.1 Decisions (decisions)`,
  `### 2.2 Glossary Terms (glossary)`, `### 2.3 Task Bead Types (task_types)`, and
  `## 3. Reference Memory`; and that `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, and
  `OPENCODE.md` are byte-identical copies.
- Confirm a webless root renders `## 1. Core Memory` then `## 2. Reference Memory` with
  no gap and no stray heading. The chezmoi-managed home root is exactly this case (its
  memory root holds only `obsidian.md`, `README.md`, and `sase.md`), so exercise it
  rather than only asserting it in a unit test.
- `sase memory init --check` is clean on a second pass.
- Re-run `just test-visual` and accept intended memory-panel snapshot changes.

### Tests for `webkind` and `section`

These files assert on the current heading text or template variables and will need
updating; treat the list as the starting point, not the whole job:

- `tests/main/test_init_memory_agents_templates.py`
- `tests/main/test_init_memory_managed_agents_generation.py`
- `tests/main/test_init_memory_agent_docs.py`
- `tests/main/test_init_memory_markdown_templates.py`
- `tests/main/test_init_memory_validation.py`
- `tests/main/test_init_memory_glossary.py`
- `tests/main/test_init_onboarding_memory.py`
- `tests/main/test_memory_agent_docs_list.py`
- `tests/main/test_section_numbers.py`
- `tests/test_memory_inventory.py`
- `tests/main/test_memory_web_cli.py` (asserts `rendering_type` in JSON)
- `tests/ace/tui/modals/test_memory_panel.py` and
  `tests/ace/tui/modals/memory_panel_test_helpers.py` (construct
  `MemoryWeb(rendering_type=...)`)

Add new coverage for, at minimum: a root with webs renders the three sections in order;
a root with no webs omits the memory-webs section entirely and reference memory numbers
as `2`; a descriptor carrying legacy `type: core` still parses and gets migrated in one
init pass; `priority:` orders webs within the section; a descriptor body that violates
the inline structure rules produces a blocker naming the descriptor; an `AGENTS.md` on
disk still using the legacy Tier headings has its reference-note descriptions recovered;
and `sase memory read <web>.md` fails with the new message while `<web>:<keyword>`
succeeds.

## Phase `docs`: Documentation, memory notes, and regenerated artifacts

### Prose in `docs/`

Rewrite the memory-tier vocabulary in these places, keeping the unrelated "tier" uses
named in the Goal section untouched:

- `docs/memory.md`: the opening list ("Each non-README note declares its tier in YAML
  frontmatter"), the `## 1. Tier 1 (core) Memory` reference, the numbered-Tier-2-section
  wording, the "Top-level reference notes are listed in Tier 2" sentence, and the entire
  `## Memory Webs` section — which currently says kind and rendering are independent
  axes and that a descriptor "renders exactly like an ordinary note". That is the claim
  this epic retires: a web always renders in the memory-webs section. Also document that
  `sase memory web list` no longer reports a rendering type, in the table and the JSON.
- `docs/init.md`: the `glossary.md` paragraph ("inlines its (core) descriptor body into
  Tier 1"), the "inlines each core note into Tier 1 ... renders Tier 2 as one numbered
  H3 subsection" paragraph and its `## Tier 2 (reference) Memory` heading reference, and
  the "Top-level project-only reference notes are listed in Tier 2" sentence.
- `docs/configuration.md`: the generated-templates table rows for
  `memory.agents_template` and `memory.agents_minimal_template` (new variable names,
  including `web_sections`), the `{{ tier2_entries }}` explanation paragraph beneath it,
  and the `sase memory init` description around "core notes are inlined into the Tier 1
  block" and "if a web's rendered roster or its Tier 1 inlining is stale".
- `docs/ace.md`: the Memory panel rail description ("Tier 1 (`short`) notes sort first,
  then Tier 2 (`long`) root notes ... A memory web (such as `glossary`) is a Tier 2 root
  row like any other note ... `●` for a Tier 1 note or `○` for Tier 2"). Rewrite for the
  three-group rail and the new web marker.
- `src/sase/memory/assets/memory-directory-map.prompt.md` is the prompt that generated
  `memory-directory-map.png`. Update its "Tier 1 short notes" / "Tier 2 long notes"
  vocabulary and its coordinate table labels. Regenerating the PNG itself is **not** in
  scope; leave the image and note in the phase's `PROPOSED FOLLOW-UP:` that the diagram
  is now stale.

### Generated memory templates and memory notes

All memory-file work in this phase goes through `/sase_memory_write` **first**.

- `src/sase/main/init_memory/templates/memory-sase.template.md`, `## SASE Memory`
  section. Rewrite the three bullets without tier vocabulary and in rendered order —
  core memory, memory webs, reference memory — something close to:
  - **Core memory** (`type: core`) is inlined here and into every provider instruction
    shim, so it is always in your context and is paid for on every turn.
  - **Memory webs** are keyed collections: a flat descriptor note
    (`sase/memory/<web>.md`) plus a sibling directory of strand files
    (`sase/memory/<web>/<slug>.md`). A web's descriptor is always inlined here; a strand
    body never is — read strands on demand with your `/sase_memory_read` skill
    (`sase memory read <web>:<keyword>`, for example `glossary:stitch`).
  - **Reference memory** (`type: reference`) is not inlined. Only its one-line
    description is listed here; read the body on demand with your `/sase_memory_read`
    skill, never by opening the file directly.

  Also fix the lead-in sentence, which currently says "A note's `type:` frontmatter
  decides how it reaches you" — that is no longer true for a web.

- `src/sase/main/init_memory/templates/memory-README.template.md`: the `type: core` /
  `type: reference` Tier bullets, the `priority` bullet ("render earlier in Tier 1"),
  the `description` bullet ("Tier 2 sections render those blocks verbatim"), the
  memory-web bullet, and the "What a memory _is_ and how it _renders_ are independent
  axes" paragraph — that paragraph's claim that `type:` on a web descriptor declares how
  it renders is now false and must be replaced with the kind-decides-placement rule. Add
  `web:`-frontmatter guidance stating that a descriptor must not declare `type:` or
  `parent:`.
- `sase/memory/glossary/core-memory.md`, `sase/memory/glossary/reference-memory.md`, and
  `sase/memory/glossary/memory-web.md` each define their term in tier vocabulary.
  Rewrite all three: core memory is inlined always-loaded context; reference memory is
  named with a description and fetched through an audited read; a memory web always
  renders its descriptor in the memory-webs section and never inlines a strand body.
  Watch the glossary validator — these strands mention each other and
  `closure: mentions` walks those mentions.
- Author a new decision strand under `sase/memory/decisions/` (suggested slug
  `webs-render-in-their-own-section`) recording the claim that a memory web's placement
  follows from its kind, not from a `type:` declaration; why it beat the alternatives
  (keeping `type:` on descriptors, or a numbered third tier); its cost (a web
  descriptor's body is now always paid for on every turn, and `type:`/`parent:` are
  silently stripped); and what would reopen it. State in that record's prose which part
  of the existing `memory-webs` record it supersedes — specifically the sentence "The
  descriptor's own body, not any strand body, participates in core or reference
  rendering." Per the decisions convention, do **not** edit
  `sase/memory/decisions/memory-webs.md` in place.

### Regeneration and linked repos

- Run `sase memory init` in this repo and commit the regenerated `AGENTS.md`,
  `CLAUDE.md`, `GEMINI.md`, `QWEN.md`, `OPENCODE.md`, `sase/memory/README.md`,
  `sase/memory/sase.md`, and the migrated web descriptors.
- The `chezmoi` linked repo holds the generated home agent docs (`home/AGENTS.md`,
  `home/CLAUDE.md`, `home/GEMINI.md`, `home/QWEN.md`, `home/OPENCODE.md`) and the home
  memory root (`home/sase/memory/`). Open it with `/sase_repo`, regenerate, and commit.
  The home root has no memory webs, so this is the webless case: its docs should come
  out with `## 1. Core Memory` and `## 2. Reference Memory`.
- `chezmoi/sase/memory/README.md` is a stale leftover from when the chezmoi repo was
  itself an enabled SASE project (it still uses the pre-legacy `type: short` /
  `type: long` vocabulary and was last touched 2026-08-07). `chezmoi` is not an enabled
  SASE project today, so `sase memory init` will not regenerate it. Route it through
  `/sase_memory_write` and hand-edit only its Tier lines to the new vocabulary.
- The other linked repos (`sase-core`, `sase-github`, `sase-nvim`,
  `sase-research-artifacts`, `sase-telegram`) were audited and carry no memory-tier
  references. `sase-core` has no `sase/memory/` tree at all. The only hits in
  `sase-telegram` and `chezmoi` outside the files above are historical SDD tale files
  describing the TUI's unrelated Tier-2 reconcile; leave them alone.

### Verification for `docs`

- `just check-full` through `/sase_monitor` before landing.
- `sase memory init --check` clean in this repo and for the home root.
- `rg -n 'Tier 1|Tier 2' -- ':!CHANGELOG.md' ':!sdd/'` in this repo returns only the
  unrelated hits enumerated in the Goal section (TUI agent-loader lazy loading, VCS
  provider resolution, editor search tiers, plan/bead tiers). Run the same sweep across
  the linked repo checkouts.
- `just fmt` / markdown formatting gates pass on the rewritten docs.

## Out of scope

- The other enabled SASE projects (`actstat`, `bob-cli`) have their own generated agent
  instruction files that will drift until `sase memory init` runs in each of them. They
  are not this repo and not linked repos, so refreshing them is out of scope. Record it
  as a `PROPOSED FOLLOW-UP:` note on the `docs` phase bead.
- Regenerating `src/sase/memory/assets/memory-directory-map.png` from its updated
  prompt.
- Any change to how flat notes declare `type: core` / `type: reference`. That axis is
  unchanged; only the word "tier" leaves the vocabulary.
- No feature flag is warranted. Nothing here keeps an old branch reachable while callers
  migrate: `sase memory init` migrates every descriptor in a single converging pass, the
  legacy heading regexes are read-only tolerance for files already on disk rather than a
  selectable behavior, and no `agents_template` override exists to break.
