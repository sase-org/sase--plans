---
tier: tale
title: Land and close sase-s5
goal:
  Close sase-s5 only after its coordinated implementation and final canonical behavior
  are fully reverified.
size: medium
proposed_by: bbugyi200.athena.0bc
create_time: 2026-08-22 19:40:33
status: wip
---

# Land and close `sase-s5`

## Goal

Finish the land-agent work for epic bead `sase-s5` only after proving that its three
closed phases remain complete on the current canonical trees. Preserve the intended
contract: the `research-highlights` hook ignores artifact-copy events, runs once from
the canonical checkout path for a committed consolidated report, and is idempotent under
finalizer reconciliation. Close the epic normally, leave the Symvision whitelist clean,
and mark the epic's durable plan complete.

## Current evidence to revalidate

- `sase-s5.1`, `.2`, and `.3` are closed with `done` resolutions and no recorded
  `PROPOSED FOLLOW-UP:` notes.
- SASE commits `740df4518` (producer-aware filtering) and `176247aa0` (coordinated
  regression coverage) are ancestors of `origin/master`.
- The linked `sase-research-artifacts` commit `a045047` is at `origin/master` and adds
  the `commit`, `sdd`, and `finalizer` provider policy.
- `sase bead epic-symbols sase-s5` currently reports no entries.
- `sase validate` currently passes, with only a deferred generated-provider-skill drift
  warning that must be reconciled from a clean, canonical SASE tree during landing.
- Post-epic commits touched finalizer internals and the config schema, so the lander
  must confirm that their current forms still preserve finalizer reconciliation and the
  `filters.producers` schema rather than relying only on the original phase snapshots.

Treat these as observations, not assumptions: repeat the checks because the canonical
branches and bead store can change between plan approval and execution.

## Implementation

1. Re-audit the epic and all three phase beads with `sase bead show`, their notes and
   histories, the dependency tree, and the audited read of
   `plan:202608/file_hook_producer_filter.md`. Re-scan for proposed follow-ups and
   unresolved descendant work. Run `sase bead epic-symbols sase-s5` before making a
   close decision. If epic-caused work remains, keep it in this epic and finish it;
   route only genuinely distinct, out-of-scope follow-ups through `/sase_new_task`, and
   retain every triage outcome for the final close note.

2. Confirm both repositories are clean and canonical. In SASE, verify that commits
   `740df4518` and `176247aa0` remain ancestors of the current canonical branch. Open
   `sase-research-artifacts` through `/sase_repo`, verify `a045047` is likewise landed,
   and do not cherry-pick already-landed work. Review the actual current producer
   vocabulary, config/schema/listing contract, preflight and batch matching, provider
   template, and end-to-end regression. Review commits since `740df4518`, excluding the
   epic's own commits, for overlap or semantic drift—especially the later finalizer
   refactors and schema update—and integrate any conflict needed to preserve the epic's
   acceptance criteria.

3. From the coordinated current trees, run `just install` before repository checks. Run
   the focused SASE file-hook/config/CLI/regression tests and the focused plugin
   provider/filter tests, then `just check` in each repository. Re-run `sase validate`
   and inspect `sase file-hook list --json`; require schema version 4 and an effective
   `research-highlights` entry whose command remains
   `bob highlights create --include-id` and whose producers are exactly `commit`, `sdd`,
   and `finalizer` while all existing sidecar/path/agent/operation filters remain
   intact.

4. Re-run the isolated coordinated regression without writing to the personal Bob or
   Obsidian vault. Require artifact registration to retain its digest-suffixed durable
   copy but produce a normal `no_match` audit with no batch or spawn; require commit
   dispatch to spawn once using the canonical Markdown basename; require finalizer
   reconciliation to return `batch_already_present` without a second spawn; and require
   Bob's canonical-input dry run to plan an unsuffixed PDF and marker id. Do not delete
   or rewrite historical digest-suffixed PDFs, notes, annotations, or artifact copies.

5. Before closing the epic, run the required whole-repository SASE verification only
   through `/sase_monitor`: monitor `just check-full` with `TESTING`/`TESTED` statuses
   and a continuation that inspects the retained result, fixes any epic-caused failure,
   and resumes this landing. Do not close on a failed, timed-out, or unexplained flaky
   result; classify unrelated failures under the project's bead rules.

6. Once the SASE source is clean, merged, pushed, and fully verified, preview the
   deferred generated-skill drift and deploy it from that canonical commit with the
   generated-skill workflow (`sase skill init --force`, allowing its normal chezmoi
   commit/push/apply sequence). Do not hand-edit generated provider skill files and do
   not use a dirty-source override. Re-run `sase validate` and the generated-skill check
   so the earlier drift warning is gone.

7. Recheck that every child is closed, there are no active dependency blockers, no
   unresolved proposed follow-ups, and `sase bead epic-symbols sase-s5` is empty. Close
   normally with
   `sase bead close sase-s5 --note "<verification, integration, test, generated-skill, and follow-up-triage outcomes>"`;
   never use `--force` merely to bypass readiness. Run `just symvision` after closure,
   then set `status: done` in the frontmatter of the linked durable epic plan. Confirm
   `sase bead show sase-s5` reports a `done` closure and that the plan link remains
   valid. The epic is top-level, so no ancestor bead needs propagation.

## Acceptance criteria

- Both epic repositories are clean and their phase commits are present on their
  canonical branches with no unintegrated post-epic conflict.
- Focused SASE and plugin checks, the isolated end-to-end behavior, `sase validate`, and
  the monitored `just check-full` all pass on the final tree.
- Effective configuration exposes only `commit`, `sdd`, and `finalizer` for
  `research-highlights`; artifact dispatch makes no batch or process, commit dispatch
  runs once on the canonical basename, and finalizer reconciliation is idempotent.
- Generated provider skills match the clean canonical SASE source, with no deferred
  drift warning.
- No historical vault or artifact content is removed or rewritten.
- `sase-s5` is closed normally with a detailed evidence note, Symvision has no stale
  epic exemption, and `plan:202608/file_hook_producer_filter.md` is marked `done`.
