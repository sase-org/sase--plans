---
tier: tale
title: Make task-type memory strands self-contained
goal:
  Agents can read complete, dynamically generated task-type details through the
  task_types memory web without invoking a separate catalog command.
size: medium
proposed_by: bbugyi200.athena.0dj.w0
---

- **AGENTS:**
  - [bbugyi200.athena.0dj.w0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0dj.w0.md)
- **COMMITS:**
  - [6b22f7e](https://github.com/sase-org/sase/commit/6b22f7e72d6d989c7350d3dec1ce41cc9ea0f5f5)
    — feat(task-types): render self-contained memory strands

# Make task-type memory strands self-contained

## Objective

Make `task_types` behave like every other memory web: agents read an individual task
type with `sase memory read task_types:<slug>`, and that strand contains all information
currently exposed by `sase bead task-type show <slug>`. Preserve automatic strand
membership from the project's effective, committed task-type catalog when bare
`sase init` or `sase memory init` runs.

## Context and constraints

The `sase-sq` epic introduced `task_types` as a generated core memory web with one
strand per committed, agent-creatable type. Today `root_rendering_task_types.py` reduces
each type to `when_to_use`, required/optional field names, and a pointer to
`sase bead task-type show`; meanwhile `cli_show.py` renders the complete field
definitions and validators, body template, triage policy, provenance, agent-creatable
state, presentation, and digest. The descriptor and the generated `sase_beads` reference
note repeat the CLI pointer. This leaves the memory web non-self-contained and gives the
CLI view and strand view separate projections that can drift.

Keep the existing catalog authority rules: builtins and project-configured types are
committed; plugin types participate only when their distribution is declared in
`plugins.required`; optional installed plugins must not perturb generated files; and
only agent-creatable catalog members receive strands. Do not change task creation,
task-type validation, the committed snapshot wire format, or the behavior of the
`task-type show` command.

## Implementation

1. Introduce a pure, shared task-type detail projection under `src/sase/task_types/` and
   make both human/JSON `sase bead task-type show` rendering and generated strand
   rendering consume it. The projection must cover the current show contract: resolved
   glyph and accent, slug, label, summary, `when_to_use`, optional `create_refusal`,
   every field's label/name/type/requiredness/roles/help and all supported validators
   (`pattern`, `max_length`, `values`, `minimum`, `maximum`), the body template or
   explicit absence, `triage.min_plus_ones`, provenance label/source/package/version,
   agent-creatable state, and digest. Preserve the show JSON schema and the current
   human CLI presentation while removing its private, duplicate data assembly.

2. Refactor `root_rendering_task_types.py` to select committed `TaskTypeRecord` objects
   once, filter the agent-creatable records for the web, and derive the snapshot entries
   and strand contents from those records. Render each strand as readable Markdown with
   frontmatter plus complete sections for the shared detail projection; use a fenced
   block for the body template so template Markdown is data rather than strand
   structure. Add an unambiguous generated-strand signature and teach retirement
   detection to recognize both the new signature and the legacy `task-type show`
   pointer, so removed types and retired home copies still converge while hand-authored
   files remain untouched.

3. Update the packaged `task_types` descriptor and `sase_beads` memory templates to
   direct agents to `sase memory read task_types:<slug>` for full details. Remove the
   generated strand pointer to `sase bead task-type show`; keep the CLI command itself
   available as an interactive catalog surface. Ensure the generated descriptor still
   provides the inline roster and discovered-work policy without inlining strand bodies
   into `AGENTS.md` or provider shims.

4. Strengthen tests around the shared contract and generated lifecycle:
   - Extend the task-type CLI tests so refactoring through the shared projection is
     proven not to change human or JSON show behavior, including conditional creation
     refusal, all validator forms, missing fields/templates, and provenance.
   - Replace the pointer-only init-memory assertion with a full-detail strand test that
     uses a representative record and proves every field in the shared show projection
     is represented in the Markdown strand. Also exercise
     `sase memory read task_types:<slug>` against generated files.
   - Preserve and expand catalog-membership coverage: project/builtin/required-plugin
     agent-creatable types generate deterministic strands; optional plugin and
     non-agent-creatable records do not; changing the effective catalog adds or retires
     the corresponding files on the next init.
   - Cover retirement recognition for both legacy pointer-based strands and the new
     generated signature, plus the existing hand-authored-file protection.

5. Regenerate the canonical project memory with `sase memory init --no-commit` after the
   code and template changes. Review the resulting `sase/memory/task_types.md`, all
   generated `sase/memory/task_types/*.md` strands, `sase/memory/sase_beads.md`,
   `sase/memory/README.md`, `AGENTS.md`, and provider instruction shims; accept only
   changes caused by the template/strand update. Do not hand-edit generated outputs.

## Acceptance criteria

- `sase memory read task_types:<slug> -r "..."` returns a self-contained task-type
  record with the same information as `sase bead task-type show <slug>` and requires no
  follow-up command.
- The shared detail projection is the single authority for both views, and tests fail if
  a show field is added without a strand representation.
- Generated task-type memory and its governing memory guidance no longer recommend
  `sase bead task-type show` as the way to read a strand.
- Bare `sase init` / `sase memory init` continues to create, refresh, and retire exactly
  the strands selected by the committed project catalog, including required-plugin
  types, without becoming machine-dependent on optional plugins.
- Strand bodies remain out of Tier 1 generated instructions; only the descriptor roster
  remains inline.
- Legacy generated strands retire safely, and hand-authored files at unexpected strand
  paths are preserved.

## Verification

Run the focused task-type, init-memory, and memory-read tests first, including
`tests/test_bead/test_cli_task_type.py`, `tests/test_bead/test_task_type_end_to_end.py`,
`tests/main/test_init_memory_task_types_note.py`, task-type snapshot tests, and the
memory selector/read tests that cover generated webs. Then run:

```bash
sase memory init --check
sase validate
just check
```

Finally inspect representative builtin, project, and required-plugin strands through
`sase memory read`, compare them field-for-field with `sase bead task-type show --json`,
and confirm a repository search finds no generated-memory recommendation to use
`sase bead task-type show` for strand details.
