---
tier: tale
title: Canonical parent plan references in agent-clan summaries
goal:
  Render managed parent-plan provenance with its canonical `plan:` identity so
  agent-clan summaries are accurate and the Agents-tab `v` flow resolves the parent in
  the plans store.
size: medium
proposed_by: bbugyi200.athena.0d2
---

- **AGENTS:**
  - [bbugyi200.athena.0d2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0d2.md)
- **COMMITS:**
  - [0f3f235](https://github.com/sase-org/sase/commit/0f3f2350d77270935b9bc6142f39df61b50c9ecc)
    — fix(sdd): render parent plan provenance canonically

# Plan

## Diagnosis

The `sase-sq.7.1` clan summary is faithfully rendering the archived child plan, but the
shared plan-display projection loses the parent artifact's kind:

1. `plan:202608/glossary_memory_web.md` contains a managed `PARENT` header entry whose
   stored label is `202608/memory_webs.md`. The header writer intentionally keeps that
   compact label while its target remains a relative or hosted hyperlink.
2. `plan_file_metadata_from_content` retains the header label and target in a typed
   `PlanProvenanceSection`.
3. `_provenance_value` in `src/sase/sdd/_plan_display_rendering.py` renders every
   provenance entry verbatim, despite knowing that this section is `PARENT`. The clan
   summary therefore emits `Parent: 202608/memory_webs.md` instead of the stable
   artifact identity `Parent: plan:202608/memory_webs.md`.
4. ACE's off-thread clan hint enrichment deliberately resolves only explicit logical
   plan tokens matching `plan:...`. The bare parent label bypasses that resolver; the
   generic file-hint fallback then maps it to `<agent-workspace>/202608/memory_webs.md`,
   which does not exist. The `v` action is not defective: it receives and opens the
   incorrect mapping produced upstream.

This was reproduced against the supplied screenshot and the archived plan: the child
`Path:` was the only logical plan reference indexed, while the parent became a missing
workspace-relative path. `sase artifact show plan:202608/memory_webs.md` resolves the
canonical reference exactly, confirming that the plans store and artifact resolver are
healthy.

## Implementation

1. Add a small, pure parent-provenance presentation normalization in the shared plan
   display layer. When rendering a `PARENT` section, project its managed month-sharded
   label as the canonical `plan:<path>` identity. Preserve already-canonical input
   without double-prefixing, leave every non-parent provenance kind unchanged, and keep
   stored header labels and link targets untouched.
2. Route both logical and width-aware plan rendering through that normalization. This
   makes the standalone plan section and the launch-time epic clan summary agree, and it
   repairs summaries generated from existing archived plans without requiring a sidecar
   migration.
3. Extend shared plan-display and epic-summary tests to assert
   `Parent: plan:202608/parent.md`, including compatibility for an existing bare managed
   header and idempotence for an already-canonical parent value. Retain target
   assertions so the display-only change cannot corrupt relative or hosted links.
4. Add an ACE clan regression that renders a plan summary containing parent provenance,
   runs the existing off-thread hint indexing and annotated clan-detail path, and
   asserts that the parent token resolves through the logical plan resolver to the
   plans-store file. Also assert that the resulting numbered hint mapping is the real
   parent path used by the `v` flow, not a workspace-relative fallback.

## Boundaries

- Do not edit `plan:202608/glossary_memory_web.md`, `plan:202608/memory_webs.md`, or any
  plans-sidecar content. Existing durable headers remain valid inputs; presentation
  normalization fixes both old and new summaries.
- Do not broaden generic file-path resolution to guess that every bare `YYYYMM/name.md`
  token is a plan. Explicit `plan:` identity avoids collisions with prompts and ordinary
  workspace files.
- Do not change the `v` dispatcher, viewer, plan-reference resolver, header wire, or
  Rust core. The fault is the Python shared presentation projection, and the existing
  downstream resolver behaves correctly once given the canonical token.

## Verification

Run the focused shared-display, clan-summary, and ACE hint suites covering the changed
paths. Reproduce the original archived-plan render and verify that it now emits
`Parent: plan:202608/memory_webs.md`, that clan hint enrichment indexes both child and
parent logical references, and that the parent hint target exists in the plans store.
Finally run `just check`, the repository's required whole-repo lint and diff-scoped test
gate.

## Acceptance criteria

- A clan summary generated from the existing `sase-sq.7.1` child plan displays
  `Parent: plan:202608/memory_webs.md`.
- Entering Agents-tab view-hint mode gives that parent a numbered hint whose mapping is
  the resolved plans-store file, so selecting it through `v` opens the parent plan.
- Other provenance rows, stored plan headers, and their hyperlink targets are byte-for-
  byte unchanged.
- Focused regression tests and `just check` pass.
