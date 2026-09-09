---
tier: tale
title: Preserve phase-local context across runner refreshes
goal: "Epic phase workers retain their authoritative plan and bead identity when a
  dependency wait refreshes the runner, and ACE never exposes the parent epic roadmap in
  a phase worker's SASE CONTEXT PLAN lane.

  "
create_time: 2026-09-09 19:53:19
status: wip
---

# Plan: Preserve phase-local context across runner refreshes

## Context and root cause

Epic launches give every phase worker durable `sdd_plan_path`, `epic_bead_id`,
`phase_bead_id`, and `plan_committed` metadata. ACE uses `phase_bead_id` as the
authoritative signal that the selected row is a phase: it derives the phase-specific
Bead description from the epic frontmatter, suppresses the full epic PLAN roadmap, and
retains the resolved plan path only to deduplicate the generic artifact list.

The reported `sase-6k.3` run started with an explicit parallel-family role of `phase`,
waited for `sase-6k.1`, and then refreshed its editable runner after the source HEAD
changed. The refreshed runner reused the same artifact directory, but its early
bootstrap write replaced `agent_meta.json` with a minimal PID/output record before
directive extraction tried to preserve the existing launch metadata. The host-only epic
environment variables had already been consumed on the original pass, so the refreshed
metadata permanently lost the plan and bead identity fields while reconstructing only
the family role from the prompt.

During ACE family reconciliation, missing plan metadata is backfilled from the epic root
so the damaged child receives the root's plan path and `epic_bead_id`, but it cannot
receive a child-specific `phase_bead_id` from that root. Associated-plan role resolution
does not currently treat the persisted `agent_family_role: phase` as authoritative. It
therefore interprets the dotted child name plus inherited epic ID as an epic author and
builds the full epic summary, producing the erroneous seven-phase PLAN lane.

## Preserve durable metadata through refreshed bootstrap

Adjust the runner bootstrap path so a code-refresh re-exec can still publish its early
PID/output seed for bootstrap-error visibility without destructively replacing the
existing `agent_meta.json`. Merge refreshed bootstrap fields into the existing object,
or equivalently defer destructive initialization only for the explicitly marked
refreshed pass, while retaining the current clean seed behavior for a first launch.
Directive extraction must then see and carry forward `sdd_plan_path`, `epic_bead_id`,
`phase_bead_id`, `plan_committed`, wait completion, and parallel-family identity before
it rewrites the full metadata.

Keep the preservation boundary narrow: one-shot epic environment variables must remain
child-local and must not leak into nested launches, normal first-run bootstrap failures
must still have a usable metadata record, and the refreshed pass must continue updating
process/output fields and the artifact index.

## Make ACE phase classification fail closed

Treat explicit `agent_family_role: phase` metadata as an authoritative phase
classification in associated-plan enrichment, alongside `phase_bead_id`. This defensive
path covers damaged in-flight and historical artifacts whose child-specific epic fields
were already lost, as well as any future partial metadata failure. A phase-classified
row must never receive an `AssociatedPlanSummary`, even when family reconciliation has
supplied the parent plan path and epic ID.

Where enough stable identity remains, use the explicit phase role, canonical agent name,
inherited epic ID, and validated epic frontmatter to preserve the phase-specific Bead
description without consulting mutable bead storage. Otherwise degrade to a bare phase
bead/name while still suppressing both the PLAN lane and duplicate generic plan
artifact. Preserve the existing behavior for epic authors and land agents, which should
continue to display the complete roadmap.

## Regression coverage

Add runner-level coverage that models a refreshed pass against a pre-existing phase
`agent_meta.json` and proves the early bootstrap stage does not erase the epic plan/bead
fields or parallel-family identity before full directive metadata is rebuilt. Retain
assertions for first-launch bootstrap-error recording so the durability fix cannot
regress that safety feature.

Add ACE model/header coverage for the exact damaged shape: an agent explicitly marked
with family role `phase`, missing `phase_bead_id`, and carrying a parent epic plan path
and epic ID after family backfill. Assert that enrichment remains phase-local, the full
associated plan is absent, the `SASE CONTEXT` PLAN lane is not rendered, the canonical
plan is not exposed as a generic artifact, and epic author/land rows still render the
complete roadmap. Include cache-key coverage if phase-role changes can alter enrichment
for an already selected row.

## Validation

Run the focused runner refresh/bootstrap and ACE associated-plan/header test modules
first, including the damaged-metadata regression. Then run `just install` followed by
the repository-required `just check`. If the PLAN lane change affects a visual fixture,
run the dedicated ACE visual suite, inspect generated diffs, and update snapshots only
when the intended phase-local presentation is the sole change.

Finally, exercise or construct a wait-refresh scenario in which a phase starts with epic
metadata, crosses a source-HEAD change, and resumes from the same artifact directory.
Verify the final `agent_meta.json` still contains all phase identity fields and that
selecting the phase in ACE shows its Bead context without the parent epic PLAN roadmap.
