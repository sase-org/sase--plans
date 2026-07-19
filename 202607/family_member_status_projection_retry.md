---
tier: tale
title: Correct ACE family-member status projection
goal: 'The FAMILY MEMBERS roster in the ACE Agents-tab metadata panel reports each
  concrete family member''s own status and runtime metadata while the top-level family
  row continues to show its aggregate workflow status.

  '
create_time: 2026-07-19 09:18:59
status: wip
prompt: 202607/prompts/family_member_status_projection_retry.md
---

# Plan: Correct ACE family-member status projection

## Context and root cause

The reported screenshot captures two valid but different projections of the `ep` tale family. The Agents list shows the
aggregate family container as `WORKING TALE`, its concrete main planner workflow step as `TALE APPROVED`, and its active
coder continuation as `WORKING TALE`. The persisted run metadata corroborates that topology: the family root is the
renamed `ep--plan` plan-chain root, and `ep--code` is its concrete coder child.

Status normalization is behaving correctly. It first derives `TALE APPROVED` for the concrete main planner step, then
derives `WORKING TALE` for the active coder, and finally mirrors the newest active child's status onto the family root
so the root summarizes the chain. The defect is downstream in family-member projection: the roster resolver enumerates
the root plus `followup_agents`, but the concrete main workflow step is attached through `runtime_children`. Because the
renamed root itself has the `--plan` identity, the resolver substitutes that aggregate root for the planner row. The
FAMILY MEMBERS entry therefore borrows the root's `WORKING TALE` status, model, runtime, identity, and content source
instead of using the concrete planner's `TALE APPROVED` state.

## Canonical concrete-member projection

Introduce or centralize a presentation-neutral resolver for the ordered concrete members represented by a sequential
family container. For a plan-chain root with a concrete main workflow agent child, resolve the planner phase to that
child from the already-built `runtime_children` graph, followed by the real feedback/question/coder continuations in
stable chain order. The aggregate root must remain the family-container row used for grouping, sorting, count chips,
runtime aggregation, and the top-level Agents-list status; it must not impersonate a concrete member when a concrete row
exists.

Keep compatibility behavior explicit. A rename-on-attach family root that is itself the first real member must still be
included. A plan-family root with no concrete main workflow child must retain the established root-backed fallback
rather than losing its planner phase. Synthetic planner projections and execution-neutral parallel-family rows must
remain excluded, and mixed `runtime_children`/`followup_agents` references must be deterministically deduplicated
without changing planner/feedback/coder ordering.

Use the same concrete-member semantics everywhere that claims to enumerate a family's real rows. In particular, align
the family detail renderer and the duplicate family-member selection in the tribe-summary model so status counts, folded
summaries, member labels, reply phases, and navigation targets cannot disagree about which object represents a member.
Keep this model-facing helper independent of widget code.

## Rendering and navigation integrity

Build every FAMILY MEMBERS field from the selected concrete object: identity, presented name, phase label, status,
provider/model, duration and other digest annotations. Use that same ordered sequence for family prompt/reply phases and
the published numbered member jump map. This prevents a cosmetic status fix from leaving number-key navigation or phase
content pointed at the aggregate container.

The correction must remain entirely in memory. Do not add filesystem reads, stats/globs, subprocesses, refreshes, async
work, or data-scaled traversal to the detail-panel render path. Selection from the normalized direct-child collections
preserves the TUI's cached, non-blocking behavior and keeps the existing detail-panel debounce boundary unchanged.

## Regression coverage

Extend focused family-display/model tests with the reported loaded topology: an aggregate `WORKING TALE` plan-family
root named for `--plan`, a concrete main workflow child normalized to `TALE APPROVED`, and an active coder continuation
normalized to `WORKING TALE`. Exercise the real status-normalization and runtime-child attachment path where practical,
then assert that:

- the root remains `WORKING TALE` in the aggregate Agents-list projection;
- the roster contains the concrete planner and coder exactly once, in chain order;
- the planner entry reports `TALE APPROVED` and its own identity, model, duration/annotations, and content source;
- the coder entry reports `WORKING TALE` and its own corresponding fields; and
- numbered jump targets resolve to those same concrete rows rather than back to the family container.

Retain explicit tests for rename-on-attach roots, missing-concrete-planner fallback, synthetic planner exclusion,
parallel-row exclusion, deterministic deduplication, and any tribe-summary counts affected by sharing the resolver. Add
or update a visual snapshot only if needed to prove the corrected visible status, and inspect the generated diff before
accepting it.

## Validation

Run `just install` before repository checks. Run the focused family-display, status-normalization, runtime-child
ordering, tribe-summary, and member-navigation tests touched by the change. Then run the mandatory `just check`, which
covers formatting, static analysis, SASE validation, the full test suite, and ACE PNG snapshots. If a PNG snapshot
fails, inspect `.pytest_cache/sase-visual/` and accept a new golden only when the only visual change is the intended
per-member projection.

## Risks and boundaries

The primary risk is changing identity or ordering while apparently fixing only the status string. That would make the
roster text, reply sections, status summaries, and number-key navigation disagree. A single shared resolver and
end-to-end assertions across those consumers address that risk. This is a Python/TUI model-projection defect; it does
not change shared backend/domain behavior and requires no `sase-core` change.
