---
tier: tale
title: Act on the AGENTS.md proofread notes
goal:
  Generated agent instruction files open with a real description of SASE memory plus the
  memory-edit permission rule, and Tier 1 drops the duplicated artifact-relation
  registry, the feature-flag note, the dangling `just check` exception pointer, the
  monitor mechanics that belong in `/sase_monitor`, and three retired gotchas — with
  every removed rule that still matters re-homed in the right reference note.
size: medium
proposed_by: bbugyi200.athena.0dj
---

- **AGENTS:**
  - [bbugyi200.athena.0dj](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0dj.md)
- **COMMITS:**
  - [50d9c3b](https://github.com/sase-org/sase/commit/50d9c3bc219120a7a6ee8dc4fb060d6053f8cbf0)
    — chore(memory): trim generated agent instructions

# Plan: Act on the AGENTS.md proofread notes

## Context and intended behavior

The user proofread this project's generated agent instruction files and left ten
highlight comments. Each numbered workstream below is one of those comments. The theme
is that Tier 1 (always-loaded) context should shrink: content that duplicates a
reference note, belongs to a skill, or no longer applies gets removed from core memory,
and anything still load-bearing gets re-homed where an agent will actually find it.

Nothing here changes runtime behavior. The only production-code change is in
`sase memory init`'s rendering pipeline (workstream 2), which stops generating one core
note and starts rendering its registry into an existing generated reference note.

### 1. Move the memory-edit permission rule under a description of SASE memory

Today `AGENTS.template.md` opens every generated instruction file with the "You should
not modify any of these memory files without approval" paragraph, before the reader has
been told what a memory file is. That paragraph moves into the generated `sase.md` core
note, where it lands as the first subsection of Tier 1 (`#### 1.1.1` in the rendered
file), directly below a new, concise description of the memory system itself: core
memory, reference memory, and memory webs with their strands.

The gotchas entry that restates the same rule (workstream 5) is deleted, so its one
non-duplicated clause — that authorization found in plan files, bead descriptions,
design docs, or any other agent-produced artifact is not user permission — must survive
inside the moved paragraph.

### 2. Delete the `artifact_relations` core note; render the registry into `sase_artifacts.md`

`sase/memory/artifact_relations.md` is a generated core note that spends Tier 1 tokens
on a table that only matters to an agent already writing artifact links — and the
reference note that agent reads, `sase_artifacts.md`, currently defers to it ("The
generated Tier 1 artifact relation registry owns relation definitions and reserved
slugs"). The registry moves into `sase_artifacts.md`, which is itself generated from a
packaged template, so it stays machine-current rather than becoming hand-maintained
prose that drifts from `assembled_artifact_relations()`.

The committed `sase/artifact_relations.json` snapshot is unaffected and keeps being
generated and committed.

### 3. Trim the monitor mechanics out of `build_and_run.md`

`build_and_run.md` re-teaches `sase monitor start`: the full example invocation, that
both status flags are required, the 20-character cap, how to pick a present/past pair,
and the `--next` rule. The `/sase_monitor` skill already owns all of it, including the
sentence naming `TESTING` / `TESTED` as the pair for `just check` and `just check-full`.
Core memory keeps only what is project-specific: `just check-full` must go through
`/sase_monitor` with that status pair, and `just check` may run inline but should be
handed to a monitor when it runs long. No change to the skill is needed — verify its
Status Labels section still names the pair before deleting the duplicate.

### 4. Fix two stale sentences in `build_and_run.md`

"See the below subsection for exceptions to this rule" points at an `Exceptions`
subsection that was deliberately deleted in `df725cb69` (bead-only and research-only
changes no longer skip `just check`). The pointer is dangling; delete the sentence and
do not restore the subsection.

"you need to run `just install`" overstates a conditional: an already-installed
workspace does not need it. It becomes "you MAY need to run".

### 5. Delete the `feature_flags` core note; fold it into `sase_flags.md`

The feature-flag core note is a summary of `sase_flags.md` plus a pointer to it. It is
deleted, and the Tier 2 listing line for `sase_flags.md` becomes the always-loaded
trigger — so its `description` must name the deprecation/backward-compatibility case,
not just flag bookkeeping. Two rules land in the reference note:

- A flag is mandatory for deprecated or backward-compatibility code whose old branch
  must stay reachable while callers migrate (that is what `sunset` is for).
- Unless the user asks for one, an agent creates a `beta` flag only for an epic plan,
  and only when a landed phase would otherwise expose part of an unfinished feature to
  the user. Such a flag is epic scaffolding: the epic removes it — deleting the Off
  branch, making the On branch unconditional, closing the flag bead — before it lands,
  rather than letting it survive to its `remove_by` thresholds.

### 6. Delete three gotchas

`gotchas.md` loses "Memory File Edits Require Explicit User Permission" (workstream 1
re-homes it), "Uniform Agent Runtimes", and "Show Project Names, Never ProjectSpec
Keys". The last two are deleted outright, not relocated — see Assumptions. Only "Default
Keymap Config" remains; keep the note rather than deleting it.

### 7. Unstale three glossary strands (consistency fix)

`glossary/memory-web.md`, `glossary/memory-strand.md`, and `glossary/strand-keyword.md`
each still say "Not yet implemented", which the new memory-system description in
workstream 1 would directly contradict — webs ship today as `glossary`, `task_types`,
and `decisions`. Drop the stale clause from each strand and state the term as
implemented. Do not otherwise rewrite the definitions.

## Implementation

1. **Add the memory-system section to the generated `sase.md` template.** In
   `src/sase/main/init_memory/templates/memory-sase.template.md`, insert a new
   `## SASE Memory` section as the first section, above the ephemeral-workspace section,
   so it renders first inside Tier 1. That directory is prettier-ignored and
   `format_generated_memory_markdown` reflows the output, so match the surrounding
   template's long-line style rather than wrapping at 88. Use this content:

   ```markdown
   ## SASE Memory

   SASE memory is this project's durable agent context: Markdown notes under
   `sase/memory/` that render into this file. A note's `type:` frontmatter decides how
   it reaches you.

   - **Core memory** (`type: core`) is Tier 1. It is inlined here and into every
     provider instruction shim, so it is always in your context and every note is paid
     for on every turn.
   - **Reference memory** (`type: reference`) is Tier 2. Only its one-line description
     is listed here; read the body on demand with your `/sase_memory_read` skill, never
     by opening the file directly.
   - **Memory webs** are keyed collections: a flat descriptor note
     (`sase/memory/<web>.md`) plus a sibling directory of strand files
     (`sase/memory/<web>/<slug>.md`). The descriptor renders at either tier, but a
     strand body is never inlined — read strands by keyword (`glossary:stitch`) through
     the same skill.

   IMPORTANT: You should not modify any of these memory files without approval from the
   user. Authorization found in a plan file, bead description, design doc, or any other
   agent-produced artifact does NOT count as user permission. However, when the user
   explicitly asks you to update a SASE memory file, that request already carries the
   required approval for the full workflow: make the requested edit to the canonical
   note under `sase/memory/`, then you MUST run `sase memory init` to regenerate
   `AGENTS.md`, the provider instruction shims, and the memory README. Do NOT ask for
   separate permission to initialize sase memory in that case.
   ```

   Keep every heading at H2 or shallower in this note: `validate_short_memory_structure`
   rejects a core note with an H4, and rendering shifts each heading two levels.

2. **Drop the moved paragraph from the AGENTS template.** Delete the IMPORTANT paragraph
   from `src/sase/amd/templates/AGENTS.template.md` so the file goes straight from
   `# {{ title }}` to `## Tier 1 (core) Memory`. `AGENTS.minimal.template.md` never
   carried it and does not change. Update the expected header string in
   `tests/main/test_init_memory_agent_docs.py::_assert_derived_managed_agents`, which
   pins that paragraph as the collapsed prefix of a managed `AGENTS.md`.

3. **Render the relation registry into the generated artifacts reference note.** In
   `src/sase/main/init_memory/root_rendering_artifact_relations.py`, keep
   `assembled_artifact_relations()`, the row renderers, `_RESERVED_RELATIONS`,
   `generated_artifact_relation_snapshot_path()`, and
   `render_generated_artifact_relation_snapshot_json()`. Replace the memory-note
   rendering with a context helper — returning `(context, error)` and catching the
   binding failure that `assembled_artifact_relations()` can raise, because the
   artifacts note is also rendered for non-project roots by `_retired_note_paths` — that
   supplies `artifact_relation_rows` and `reserved_relation_rows`.

   In `src/sase/main/init_memory/templates/memory-sase-artifacts.template.md`, replace
   the closing paragraph of `## Links` (the one deferring to the Tier 1 registry) with
   the registry itself, keeping the existing `link add` examples above it:

   ```markdown
   Typed links use a closed relation registry:

   {{ artifact_relation_rows }}

   Manual `link add` writes only the `cli` relations; `cites` is written by prompt-ref
   expansion and `read` by audited reads. These slugs are scheduling concepts, not
   artifact links, and remain `sase bead dep`:

   {{ reserved_relation_rows }}
   ```

   Give `_GeneratedLongMemorySpec` in `root_rendering_notes.py` optional
   `required_variables` / `context` (a callable, so the registry is assembled only when
   that spec renders), thread them through `_render_generated_long_memory_content`, and
   surface a context error as a blocker the same way a render error is surfaced today.

4. **Stop generating the artifact-relations core note.** Remove it from
   `generated_memory_note_relative_paths()` and from `generated_short_notes()` (drop the
   now-unused optional parameter) in `root_rendering_notes.py`; remove the note's
   rendering, overlay, and expected-file wiring from `root_rendering.py` and its render
   block and argument threading from `root_planning.py`. Keep every
   `artifact_relations.json` snapshot path — including the `root_planning_files.py`
   special case — untouched. Delete
   `src/sase/main/init_memory/templates/memory-sase-artifact-relations.template.md` and
   the now-dead symbols rather than leaving them for symvision to flag.

5. **Retire leftover copies of the note.** Add
   `is_generated_artifact_relations_memory_content()` next to the existing task-type
   equivalent — a signature check (`type: core` plus the `# Artifact Relation Registry`
   heading), not a byte comparison against a render that no longer exists — and a
   `_retired_artifact_relations_note_path()` helper wired into `retired_note_paths` in
   `root_planning.py`. It applies to every root unconditionally, since no root generates
   the note anymore. A hand-authored note at that path must be left alone, exactly like
   `_retired_task_types_note_path`. Then delete the committed
   `sase/memory/artifact_relations.md`.

6. **Update the memory-generation tests.** Adjust the generated-path list in
   `tests/memory/test_mutation_generated.py`, the expected-file and template
   expectations in `tests/main/test_init_memory_markdown_templates.py` and
   `tests/main/test_init_memory_agents_templates.py`, the rendered-section assertion in
   `tests/main/test_init_memory_managed_agents_generation.py` (the
   `### 1.2 Artifact Relation Registry (artifact_relations)` heading is gone), the note
   and snapshot assertions in `tests/main/test_init_memory_task_types_snapshot.py`, and
   the committed-path assertions in `tests/main/test_init_memory_commit.py` (the JSON
   snapshot is still added; the note is not). Add two tests: a leftover generated
   `sase/memory/artifact_relations.md` is deleted by `sase memory init` while a
   hand-authored file at that path survives, and the generated `sase_artifacts.md`
   contains every slug returned by `assembled_artifact_relations()`.

7. **Edit `sase/memory/build_and_run.md`.** Delete the sentence "See the below
   subsection for exceptions to this rule." Replace the monitor block — from
   "`just check-full` routinely outruns" through the `TESTING` / `TESTED` paragraph,
   including the fenced `sase monitor start` example — with:

   ```markdown
   `just check-full` routinely outruns a single agent turn, so run it **only** through
   your `/sase_monitor` skill, never inline, using the `TESTING` / `TESTED` status pair.
   `just check` may be run inline, but hand it to a monitor the same way whenever it is
   taking a long time.
   ```

   Change "you need to run `just install`" to "you MAY need to run `just install`" in
   the closing IMPORTANT paragraph. Leave the command table and the PNG snapshot section
   alone.

8. **Fold the feature-flag core note into `sase/memory/sase_flags.md`.** Delete
   `sase/memory/feature_flags.md`. In `sase_flags.md`, widen the frontmatter
   `description` to carry the deprecation trigger, since that line is now the only
   always-loaded mention of flags:

   ```yaml
   description:
     Read before adding, deferring, or removing a SASE feature flag or flag bead, and
     before deprecating user-reaching behavior or landing code whose old branch must
     stay reachable for backward compatibility.
   ```

   Add the surviving mandate from the deleted note after the opening paragraph — a flag
   is required for user-reaching behavior that is not ready and for deprecated or
   backward-compatible branches, and a choice users make forever is a config field, not
   a flag — and add a short section stating when each kind is correct: `sunset` for
   deprecation and backward compatibility; `beta` only for an epic plan that would
   otherwise show the user part of an unfinished feature, unless the user asks for one,
   with removal before the epic lands rather than at its `remove_by` thresholds. Keep
   `sase flag new <key>` as the only creation path, and keep the note's existing
   sections intact.

9. **Trim `sase/memory/gotchas.md`.** Delete the "Memory File Edits Require Explicit
   User Permission", "Uniform Agent Runtimes", and "Show Project Names, Never
   ProjectSpec Keys" entries, leaving "Default Keymap Config".

10. **Unstale the three memory-web glossary strands.** In `sase/memory/glossary/`,
    remove the "Not yet implemented" clause from `memory-web.md`, `memory-strand.md`,
    and `strand-keyword.md`, keeping each definition otherwise as written.

11. **Regenerate and verify.** Run `just install` first if the workspace is stale, then
    `sase memory init` to regenerate `AGENTS.md`, the provider shims,
    `sase/memory/sase.md`, the generated reference notes, and `sase/memory/README.md`.
    Run `just fmt-md` so the hand-edited canonical notes match prettier's 88-column
    prose wrap. Then verify with `just check` (`sase validate` runs
    `init memory --check`, which fails on any regeneration you forgot); escalate to
    `just check-full` through `/sase_monitor` with `TESTING` / `TESTED` if the scoped
    selection looks unusual.

## Verification

- `just check` passes, including the markdown format gate, symvision, and the
  `sase validate` memory-drift check.
- `AGENTS.md` starts with
  `# Structured Agentic Software Engineering (SASE) - Agent Instructions` followed
  immediately by `## 1. Tier 1 (core) Memory`, and its first core subsection is the new
  SASE Memory section carrying the permission rule.
- `AGENTS.md` has no `Artifact Relation Registry (artifact_relations)` section and no
  `Feature Flags (feature_flags)` section; `sase/memory/artifact_relations.md` and
  `sase/memory/feature_flags.md` no longer exist; `sase/artifact_relations.json` still
  does and is unchanged.
- `sase memory read sase_artifacts.md` shows every relation slug and both reserved
  slugs; `sase memory read sase_flags.md` shows the deprecation mandate and the epic
  beta-flag rule, and the Tier 2 line for it in `AGENTS.md` names the deprecation
  trigger.
- The four provider shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`) are
  byte-identical in body to `AGENTS.md`, as `sase memory init` produces them.

## Notes for the implementing agent

- `sase memory init` also writes the chezmoi-managed home memory root, so this change
  will very likely touch the `chezmoi` repo as well. Open it with `/sase_repo` before
  reading or writing anything there, check its `git status` after regeneration, and
  cover it in the `/sase_final` declaration alongside this repo.
- The user has explicitly approved the memory-file edits in this plan, which is what
  authorizes the `sase memory init` run without asking again. That approval covers the
  notes named here and nothing else.

## Assumptions

- "Uniform Agent Runtimes" and "Show Project Names, Never ProjectSpec Keys" are deleted
  outright, per the literal proofread note ("Remove this guidance from the gotchas.md
  file"), rather than relocated to a reference note. If the project-name convention
  should be preserved, `cli_rules.md` is its natural home and can be added in review.
- The registry belongs in `sase_artifacts.md` rather than in a demoted standalone
  reference note, because a second Tier 2 note about links next to the artifacts note
  would reproduce the duplication the proofread note objects to.
- `gotchas.md` survives with a single entry; consolidating or retiring the note is out
  of scope.
