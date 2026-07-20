---
tier: tale
title: Extract the shared PLAN-lane display module
goal: 'PLAN-file metadata and logical Rich rendering are reusable outside the ACE
  widget while ResponsivePlanSection retains identical behavior, responsive output,
  cache performance, and PNG snapshots.

  '
create_time: 2026-07-20 14:43:43
status: wip
prompt: 202607/prompts/shared_plan_display.md
---

# Plan: Extract the shared PLAN-lane display module

## Context and scope

The ACE Agents-tab PLAN lane currently owns both presentation-independent plan metadata construction and all styled
logical rendering. Later clan-summary work needs exactly the same plan-file view, but duplicating those rules would let
the two displays drift. This change is limited to the `sase-8d.1` shared-renderer phase: it will create the reusable
display boundary and migrate the existing PLAN lane onto it. Summary-script argument handling, new console scripts, epic
summary wiring, documentation for those scripts, and Rust-core changes remain outside this bead.

## Shared plan display model and loading

Add a focused module under `sase.sdd` that owns the immutable phase value type and the display-shaped plan metadata
needed by both a TUI lane and a serialized summary: normalized title and goal, authored/display tier information,
display path, phase availability, and validated phase summaries. Keep filesystem and ACE-only association state—resolved
paths, commit/readability flags, role resolution, and cache entries—in the existing TUI model layer. Re-export moved
types through the current associated-plan modules so callers and tests keep a stable import surface.

Extract the existing frontmatter normalization, tier detection, and epic phase conversion into shared loading helpers
backed by the canonical plan validator in launch mode. The loader must report unavailable or invalid metadata without
raising, preserve tale-versus-epic semantics, and accept the path/display-path context needed by non-TUI consumers. Keep
the Agents-tab mtime/size cache and negative TTL behavior intact; its miss path delegates metadata construction to the
shared helper, so no file reads, stats, parsing, or validation move into `ResponsivePlanSection` or another render-time
path.

## Shared logical rendering and ACE delegation

Move the PLAN palette and plan-specific styles to the shared presentation boundary, retaining compatibility imports from
the current ACE helper modules where necessary. Provide pure Rich-text builders for the lane details,
`Title:`/`Goal:`/`Path:` rows, phase title/metadata/description blocks, and the complete unwrapped logical document.
Builders take explicit display data and a target width or caller-supplied path context; they perform no filesystem or
cache access and preserve the current labels, indentation, glyphs, pluralized phase counts, missing/unavailable states,
dependency/model wording, and styles.

Refactor `ResponsivePlanSection` into a responsive adapter over those shared values and logical builders. Its
table-based wrapping remains responsible for hanging field indentation and lossless terminal-cell folding at narrow
widths, and path jump-hint registration remains in the ACE layer. Both `logical_text` and `__rich_console__` must
consume the same shared rows and phase blocks so the fixed logical representation and responsive widget cannot diverge.
The result must be byte-for-byte equivalent in plain text and style-equivalent for existing PLAN-lane fixtures, with no
new render-path I/O.

## Compatibility and verification

Add unit coverage at the shared-module boundary for valid tale and epic files, whitespace normalization, phase
conversion, invalid/unavailable inputs, tier and phase-availability reporting, logical field/phase ordering, optional
description/model handling, dependency wording, styles, and width-aware output. Add explicit parity tests that feed one
display value to the shared builders and `ResponsivePlanSection` and compare their logical styled lines. Retain and run
the existing associated-plan cache tests to pin mtime invalidation, negative TTL behavior, and validator call counts,
plus the existing responsive widget tests for long ASCII, wide Unicode, narrow terminals, missing files, and jump hints.

Before broader verification, run the focused shared-display, associated-plan, and PLAN-section test modules. Because
this checkout may be stale, run `just install` before repository gates, then run `just check` as required for SASE
source changes. Run the dedicated `just test-visual` suite and require the existing agent PLAN-lane PNG snapshots to
pass exactly without accepting golden updates. If lint exposes symbol-move issues, use the mandated Symvision memory
workflow before correcting them and rerun all affected checks. Once every gate passes, update `sase-8d.1` to `closed`;
leave parent epic `sase-8d` and all other beads unchanged, and create no beads.
