---
tier: tale
goal: 'Agents associated with epic plans see every declared epic phase as a polished,
  responsive roadmap inside the leading SASE PLAN section, with authoritative frontmatter
  semantics and no regression to metadata reliability or navigation responsiveness.

  '
create_time: 2026-07-15 14:42:53
status: wip
prompt: 202607/prompts/epic_phase_metadata.md
---

# Plan: Render epic phases in SASE PLAN metadata

## Product contract

Extend the existing Agents-tab `SASE PLAN` section for associated epic plan files. Keep the section immediately after
the ordinary top-level agent metadata fields, whose final row is `Timestamps`, and before every optional or
content-oriented section: output variables, commits, deltas, artifacts, workflow variables, SASE context, slow tool
calls, errors, prompts, replies, chats, and later sections appended by the detail renderer. Treat xprompt usage and its
indented entries as ordinary metadata fields, so they remain above `SASE PLAN`. Preserve the current omission behavior
when no associated plan exists.

For tale plans, retain the current compact `Goal`, `Tier`, and `Path` presentation unchanged. For epic plans, keep those
three rows in the same order and add a visually distinct phase roadmap beneath them:

```text
Phases: 3
  1 ◆ Planner and safety checks
    core · no dependencies
    Establish the canonical normalized data model.
  2 ◆ Responsive phase renderer
    render · after core · model codex/gpt-5.6-sol
    Render every phase without truncation.
```

The exact styling should follow ACE's existing visual vocabulary rather than resemble raw YAML. Use a restrained
`Phases: N` subheading, stable one-based ordinals, a small non-status glyph, prominent phase titles, quiet canonical
IDs, and human-readable dependency text (`no dependencies` or `after <id>, ...`). Show an authored phase model on the
same compact metadata line only when one is present; do not synthesize or expose an internal default-model value. Render
a phase description on its own softly styled, hanging-indented line when present. Ordinals describe the authored
frontmatter order only: do not color or glyph them as execution state, and do not infer live bead progress from this
static roadmap.

Render every normalized phase and every authored value completely. Long titles, descriptions, dependency lists, models,
and unbroken or wide-Unicode tokens must fold by terminal cell width with stable hanging indentation and no ellipsis.
Retain the section's 80-cell maximum content width on wide panels and reflow to the available width in the normal
metadata panel and metadata zoom view. Let the panel scroll naturally for large epics rather than silently collapsing or
capping the list.

Use the authored file tier to decide whether phase data belongs to the plan, so an explicitly uncommitted epic can still
describe its phases even though its effective displayed tier is `none`. When the association is known to be epic but the
file is missing, unreadable, malformed, or fails authoritative epic validation, keep the existing Goal/Tier/Path
fallbacks and show one quiet `Phases: unavailable` state instead of displaying partial, misleading entries or dropping
the whole section. Never show a phase block for a readable tale plan.

## Data and reliability design

Add an immutable phase-summary record carrying the normalized `id`, `title`, ordered `depends_on` IDs, optional
`description`, and optional `model`, and include a tuple of those records plus explicit availability state in the
existing associated-plan summary. Populate it only in the current deferred detail-header enrichment worker and retain it
inside the existing file-signature cache. The immediate j/k path must remain memory-only: no YAML reads, validation,
stats, bead queries, or subprocesses may move into header construction or Rich rendering.

Do not create a second permissive interpretation of epic frontmatter. Read the plan content once per cache miss and use
the existing Rust-backed `sase.sdd.plan_validate` contract to obtain normalized epic phases only after the complete epic
schema validates. Reuse or modestly expose the adapter's validated-plan/phase types as needed rather than reaching
through private symbols. Continue the existing best-effort frontmatter path for Goal/Tier and missing-file diagnostics,
so damaged historical plans degrade as they do today while their phase list is all-or-nothing. Preserve mtime-plus-size
cache invalidation, bounded entries, negative TTL behavior, family propagation, canonical path selection, artifact
deduplication, and approval-state invalidation.

Keep this as presentation and Python enrichment work in the current repository. The authoritative schema and dependency
rules already live in `sase-core`; no wire, index-schema, bead, or Rust-domain change is needed unless implementation
uncovers a genuine gap in the existing normalized validation payload. In that event, update the shared core API rather
than duplicating validation rules in ACE, and document the boundary change in the implementation handoff.

## Rendering and integration

Evolve the dedicated responsive plan renderable so Goal/Tier/Path and the epic roadmap remain one logical major section
for Rich rendering, `.plain` inspection, styles/spans, hint mode, zoom mode, and the mutable `AgentHeader.append`
contract. Compose phase rows with width-aware Rich primitives and explicit indentation columns; avoid pre-wrapping by
Python character count, since terminal cell width and wide Unicode must remain correct. Keep the path as the section's
only file hint and preserve its current missing-file treatment.

Make the section boundary structural and testable: construct all top-level metadata first, splice `SASE PLAN` exactly
once, then append every major section. Strengthen the existing placement contract rather than relying on the current
partial assertions against only output variables and deltas. Ensure the final divider and all prompt/reply append flows
remain below the plan section, with no duplicate separators or blank-space regressions when phases are absent or
unavailable.

Update ACE documentation to describe the phase roadmap, the meaning of order and dependencies, optional description and
model display, complete responsive wrapping, malformed-plan fallback, and the guarantee that `SASE PLAN` is the first
major section after ordinary metadata fields.

## Verification

- Extend associated-plan model/cache tests with a fully valid epic containing multiple roots, chained and fan-in
  dependencies, descriptions, and a phase-specific model; assert exact immutable normalized values, authored-epic
  behavior when effective tier is `none`, cache hits, signature invalidation, and no phase data for tales.
- Cover missing, unreadable, invalid-YAML, structurally invalid, and schema-invalid epic plans. Confirm they cannot leak
  partial phases, do not crash enrichment, retain the known plan section/path, and render the quiet unavailable state
  only when epic context is known.
- Expand renderer tests for exact field/subheading order, count grammar, ID/dependency/model/description styling,
  multiple dependency roots, omitted optional fields, and complete narrow/wide folding of ASCII, spaces, unbroken
  tokens, and wide Unicode. Preserve `.plain`, spans, path hints, zoom rendering, and post-header append behavior.
- Build one maximal header fixture that enables every optional major section and assert
  `Timestamps < SASE PLAN < each other section`; separately prove Name through Timestamps and xprompt metadata remain
  above it and tale/no-association layouts retain their established spacing.
- Add or update a focused Agents PNG golden that visibly exercises a compact three-phase epic, including a dependency, a
  description, and an optional model at normal panel width. Keep the existing tale visual coverage so both compact and
  expanded forms are reviewed, and exercise metadata zoom or a narrow rendering case if the normal golden cannot show
  the wrapping hierarchy clearly.
- Run focused model/header tests and the targeted visual snapshot first, then `just install`, the dedicated visual
  suite, and the repository-required `just check`. Re-run the existing Agents j/k performance benchmark or trace to
  confirm plan parsing remains deferred and cached and that rendering the roadmap does not compromise navigation
  responsiveness.
