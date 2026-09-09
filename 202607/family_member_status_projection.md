---
tier: tale
title: Correct ACE family-member status projection
goal: "The FAMILY MEMBERS roster in the ACE Agents-tab metadata panel reports each
  concrete family member's own status and runtime metadata while the top-level family
  row continues to show its aggregate workflow status.

  "
create_time: 2026-09-09 19:53:11
status: wip
---

# Plan: Correct ACE family-member status projection

## Context and diagnosis

The screenshot demonstrates that the Agents list already has two distinct and correct
concepts: the top-level `ep` family container is `WORKING TALE`, while its concrete
planner child is `TALE APPROVED` and its coder child is `WORKING TALE`. The
status-normalization path preserves that distinction: it assigns the planner child its
approved status, assigns the active coder its working status, then intentionally mirrors
the newest active child onto the family root so the root can summarize the whole chain.

The detail-panel roster loses that distinction. Its sequential-family resolver ignores
the concrete main workflow child already attached to the root's `runtime_children` and
instead treats the aggregate root object as the first real `--plan` member.
Consequently, the roster renders the root's aggregate `WORKING TALE` value under the
planner label. This is a presentation projection bug, not a status-computation or
Rust-core bug.

## Member projection and rendering

Correct the sequential-family member resolution used by
`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py` so a plan-family root
resolves its first roster/reply phase to the concrete main planner workflow child when
that row is present. Continue to use the aggregate root as the family-container row in
the Agents list, but do not reuse its aggregate status, duration, identity, or other
member annotations as the planner's metadata.

Preserve the established compatibility behavior for family shapes that do not have a
concrete main workflow child: rename-on-attach roots may still represent the first real
member themselves, legacy bare-root containers must not gain a duplicate member, and
synthetic planner projections and execution-neutral parallel-family rows must remain
excluded. Keep chain ordering and deduplication stable so planner/feedback/coder rows,
numbered jump targets, and reply phases all refer to the same concrete objects. Audit
the duplicate sequential-family row selection in
`src/sase/ace/tui/models/agent_tribe_summary.py`; reuse or align the corrected
presentation-neutral semantics if that path would otherwise keep counting the aggregate
root as a concrete member.

The correction must remain entirely in-memory. Do not introduce file reads,
subprocesses, refreshes, or other work into the render path; selecting from the
already-built `runtime_children` and `followup_agents` collections preserves the TUI's
cached, non-blocking detail-panel behavior.

## Regression coverage

Extend the focused family-display tests with the reported topology: an aggregate
`WORKING TALE` plan-family root, a concrete `TALE APPROVED` planner workflow child, and
an active `WORKING TALE` coder follow-up. Assert that the rendered FAMILY MEMBERS
entries use the planner and coder's exact statuses, identities, labels, models,
durations/annotations, ordering, and numbered jump targets rather than borrowing the
aggregate root state.

Retain explicit coverage for the no-concrete-planner fallback and for exclusion of
synthetic/parallel rows. Where practical, exercise the real normalization and
runtime-child attachment sequence so the regression test proves the same cross-layer
state transition seen in the screenshot: root aggregation remains `WORKING TALE`, the
planner remains `TALE APPROVED`, and the metadata roster projects both concrete children
correctly.

## Validation

Run `just install` before repository checks, then run the focused family-display and
tale-status normalization tests. Re-run any related tribe-summary or panel navigation
tests affected by sharing the member resolver. Finally run the mandatory `just check`,
which covers formatting, static analysis, SASE validation, the full test suite, and ACE
PNG snapshots. Inspect visual snapshot diff artifacts if the family roster fixture
changes, accepting a golden update only when it reflects the intended per-member status
correction.

## Risks and boundaries

The main risk is changing member identity or ordering while fixing only the status text,
which could make number-key navigation or reply-phase content point at a different row.
Resolving all roster fields and jump targets from one concrete member sequence avoids
that split-brain state. The aggregate family root must continue to drive grouping,
sorting, count chips, and the top-level Agents-list status; only surfaces claiming to
enumerate real family members should use the per-member projection.
