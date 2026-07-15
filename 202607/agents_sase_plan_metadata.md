---
tier: tale
goal: Agents with an associated proposed plan see a polished SASE PLAN metadata section
  with the plan's goal, effective tier, and canonical path, without a top-level Goal
  field or any regression to Agents-tab responsiveness.
create_time: 2026-07-15 13:52:53
status: done
prompt: 202607/prompts/agents_sase_plan_metadata.md
---

# Plan: Add a SASE PLAN section to agent metadata

## Product contract

Replace the selected agent's top-level `Goal:` row with a dedicated `SASE PLAN` major section. Place it immediately
after the ordinary agent identity/runtime fields (ending with `Timestamps`) and before other optional major sections so
the panel reads as agent context first, plan context second, and execution outputs afterward. Omit the entire section
when the agent has no known plan association.

Render exactly three labeled rows in this order:

1. `Goal:` contains the complete normalized frontmatter goal. Preserve the existing responsive behavior: no truncation,
   hanging indentation, Unicode- aware folding, and an 80-cell maximum line width on wide panels.
2. `Tier:` is the user-facing enum `plan`, `epic`, or `none`. While a proposal is awaiting a decision, map authored
   `tier: tale` to `plan` and authored `tier: epic` to `epic`. After a decision, let the persisted approval outcome win:
   an explicitly uncommitted approval is `none`, an epic approval is `epic`, and committed tale/commit choices are
   `plan`. Never expose the internal word `tale` in this section.
3. `Path:` names the plan represented by the section. For a committed plan, select the canonical SDD copy and display it
   relative to the agent workspace (including sidecars such as `sase/repos/plans/...`). For a pending or explicitly
   uncommitted plan, select the durable machine-local archive and display its absolute home-shortened form
   (`~/.sase/plans/...`). Retain the resolved absolute path separately so hint mode can number and open the file,
   including paths containing spaces.

Use the established major-section visual language: a dim divider, gold underlined `SASE PLAN` heading, blue field
labels, the existing warm italic goal value, and distinct but restrained tier accents (`plan`, `epic`, and the
intentional `none` state). Make `Path:` use the existing path/basename styling and missing-file treatment. Keep all
values complete and responsive in narrow panels and the metadata zoom modal. If an old or damaged associated plan is
missing readable frontmatter, keep the known section/path visible and render a quiet unavailable fallback for the
missing value instead of making the whole section disappear.

## Data and enrichment design

Promote the existing plan-goal enrichment into one immutable associated-plan summary that carries normalized goal,
authored/effective tier, actual path, display path, commit state, and existence/readability state. Resolve the plan
association once, using the current precedence for direct planner/family metadata and epic/phase bead designs, then
parse frontmatter once per file signature. Continue to bound association caches and invalidate file-derived metadata by
mtime plus size.

Make the committed/uncommitted decision explicit in the agent snapshot rather than inferring it from path existence. Add
the optional `plan_committed` boolean to the Rust `AgentMetaWire` projection in the linked `sase-core` repository, parse
both `true` and `false` without truthiness coercion, and advance the artifact-index record-json schema so existing
indexed rows are rebuilt with the new field. Mirror the additive field in the Python wire model/conversion and in the
ACE `Agent` model, and populate it identically through filesystem-backed, snapshot-backed, running, completed, workflow,
and family/follow-up loading paths. Preserve `plan_action` as a compatibility signal for older records, but treat an
explicit `plan_committed` value as authoritative.

Centralize canonical plan selection so metadata and the existing artifact inventory agree: committed records prefer
`sdd_plan_path`; pending and uncommitted records prefer `plan_path.json`/the archived `plan_path`; bead associations
resolve their design path as today. Resolve relative references against the agent workspace, primary workspace, and
SDD/sidecar roots before formatting the display path. Avoid showing the same canonical plan again in the generic
`Artifacts:` metadata list once `SASE PLAN` owns its description and hint, while leaving the plan available in the file
panel and artifact APIs.

Build this summary only in the existing debounced detail-header enrichment worker. The cheap j/k render path must remain
memory-only and should either use the cached section or omit it until enrichment lands; it must not stat files, parse
YAML, query beads, read JSON, or invoke subprocesses. Ensure cache keys or refresh invalidation allow a visible pending
proposal to transition promptly from its authored tier/archive path to `none` or a committed plan/epic path after
approval metadata changes.

## Rendering and integration

Replace the goal-specific header wrapper with a focused responsive plan-section renderable that can be spliced into the
otherwise mutable Rich header while preserving the current `AgentHeader` append interface, logical `.plain` text,
styles/spans, error rendering, prompt/reply append paths, and hint rendering. Keep section construction in a small
dedicated renderer rather than continuing to grow the top-level header builder.

Update ACE documentation to describe `SASE PLAN`, its presence rule, the three tier values, committed versus local path
forms, responsive wrapping, and path hint behavior; remove documentation that describes `Goal` as a top-level agent
detail field.

## Verification

- Add Rust scan and artifact-index migration/parity coverage proving `plan_committed: true`, `false`, and absent survive
  live scans and indexed reads without changing older-record compatibility.
- Expand Python loader/model/bundle tests for filesystem and wire parity, canonical archived-versus-SDD path selection,
  family and bead associations, approval overrides, and pending authored-tier mapping.
- Replace the goal-only resolver tests with associated-plan summary tests for tale-to-`plan`, epic, explicit `none`,
  committed sidecar-relative paths, uncommitted `~/.sase` paths, missing files/frontmatter, mtime invalidation, and no
  hot-path I/O.
- Add renderer tests for section placement and omission, exact field order and styles, full wrapping/folding at narrow
  and wide widths, paths with spaces, missing-path presentation, hint mappings, zoom rendering, and removal of the old
  top-level `Goal:` row and duplicate artifact entry.
- Update the Agents visual snapshot to exercise a long goal plus all three fields in the normal panel, and add focused
  visual coverage for the narrow or zoomed layout if the existing snapshot does not demonstrate the responsive hierarchy
  clearly.
- Run the relevant Rust checks in `sase-core`, then install this workspace and run focused TUI/model tests,
  `just test-visual`, and the repository-required `just check`. If rendering changes affect navigation timing, exercise
  the existing j/k performance benchmark or trace and confirm the detail work remains outside the event loop.
