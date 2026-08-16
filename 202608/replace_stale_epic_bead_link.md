---
tier: tale
title: Replace stale epic bead links during plan work
goal:
  Make sase bead work recover archived epic plans whose managed bead_id points to a
  missing bead by creating a fresh epic and replacing the stale link safely.
size: medium
proposed_by: bbugyi200.athena.031
create_time: 2026-08-15 19:53:04
status: done
---

- **PROMPT:**
  [prompts/202608/replace_stale_epic_bead_link.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/replace_stale_epic_bead_link.md)
- **AGENTS:**
  - [bbugyi200.athena.031](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.031.md)
- **COMMITS:**
  - [37fe22b](https://github.com/sase-org/sase/commit/37fe22b8115f77ae283e7fca7b663630ecdca511)
    — fix(bead): replace stale epic links from plan files

# Plan: Replace stale epic bead links during plan work

## Context

`sase bead work <epic-plan.md>` currently interprets every non-empty managed `bead_id`
as a resume request. When the archived plan survived but its linked bead did not, launch
stops after archiving with an instruction to remove the field manually. The archived
plan is the durable source of truth, so retrying the original scratch plan continues to
select that preserved stale link and fails in the same way.

Valid linked epic beads must continue to resume without duplication. A link to an
existing non-epic bead remains a consistency error rather than being silently replaced.
The recovery path must also retain the current launch lock, store-health, commit,
publication, and rollback boundaries.

## Implementation

1. Refine linked-epic resolution in the plan-file work helpers so callers can
   distinguish a genuinely missing linked bead from a valid resumable epic while still
   rejecting an existing bead of the wrong type. Use that result in both dry-run and
   mutating launch paths.
2. In dry-run mode, treat a missing managed `bead_id` as a replacement preview: report
   that a new epic would be created, do not claim the stale ID as resumable, and do not
   write either the plan or bead store.
3. In mutating mode, carry the exact stale ID into deterministic epic creation as the
   only link value authorized for replacement. Create the complete new bead DAG first,
   then use the existing locked plan-store commit path and frontmatter/header projection
   to replace `bead_id` directly with the new epic ID. Keep the ordinary duplicate guard
   for unexpected or changed link values, and make rollback restore the plan's original
   stale link if launch fails before state becomes durable.
4. Add concise rendered output identifying stale-link replacement so an interactive
   retry explains why a new epic is being created instead of resumed.

## Tests

1. Replace the existing missing-link failure expectation with end-to-end coverage that
   starts from an archived plan linked to a nonexistent bead, launches a newly created
   epic, and verifies the resulting DAG and managed `bead_id` point only to the new
   epic.
2. Cover dry-run behavior and confirm the stale plan content and bead store remain
   unchanged while the result previews creation rather than resume.
3. Cover rollback after an attempted stale-link replacement and verify the prior stale
   value is restored, while preserving existing tests for valid resume links and
   wrong-type/duplicate link rejection.
4. Add focused unit coverage at the deterministic creation boundary to prove only the
   explicitly expected stale value may be replaced and a mismatched existing value is
   rejected.

## Verification and delivery

Run `just install` before repository checks. Exercise the focused plan-file work,
resume, preview, and deterministic epic-creation test modules while iterating. Then run
`just check`; if its scoped selection escalates or reports unusual coverage, run
`just check-full` through `/sase_monitor`. Inspect the final diff and repository status,
then use `/sase_git_commit` to create the requested commit and push it through the
required SASE stitch workflow.
