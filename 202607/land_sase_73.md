---
tier: epic
title: Integrate and land smart summary folding
goal: 'Documentation and bead traceability match the completed clan, family, and tribe
  summary behavior on current master, the verified implementation remains green with
  later ACE changes, and epic sase-73 is closed with post-close Symvision hygiene
  and its canonical plan marked done.

  '
phases:
- id: integrate
  title: Reconcile documentation and epic traceability
  depends_on: []
  description: '''Reconcile documentation and epic traceability'' section: update
    user-facing fold guidance to the implemented smart-summary contract, preserve
    later navigation behavior, repair stale child-bead commit notes, and revalidate
    the integrated feature.'
- id: land
  title: Close and finalize sase-73
  depends_on:
  - integrate
  description: '''Close and finalize sase-73'' section: close the original epic, run
    post-close Symvision cleanup and repository checks, then mark the original epic
    plan done.'
create_time: 2026-07-19 11:50:36
status: wip
---

# Plan: Integrate and land smart summary folding

## Context and verified baseline

Epic `sase-73` implements one fold-aware presentation contract for clan, family, and tribe summaries in the Agents
metadata panel. Its original durable plan is `sase/repos/plans/202607/smart_summary_folding.md`; keep that plan at
`status: wip` until the final phase below.

All three child beads are closed. The canonical commits currently reachable from `master` are:

- `c433dc7590a64bfa186f311c89b4b75482d63683` — Phase 1, tribe documents, shared fold language, and always-visible member
  rosters (`sase-73.1`).
- `c85cdd7a369c9c79aa0be9e7a9044f7597ac41c3` — Phase 2, clan and family summary refinement (`sase-73.2`).
- `4665110c7e86f301d45c5288039afa150b39dd32` — Phase 3, cross-kind contracts and final tribe goldens (`sase-73.3`).

The child-bead notes do not all point at those canonical objects: `sase-73.1` names missing object `659c96b0f`, and
`sase-73.3` names pre-integration object `51af6074a`. The latter contains the same Phase 3 patch on an earlier parent,
but it is not the commit on current `master`. Repair those notes during integration so the archived epic remains
auditable.

The land audit read the implementation rather than trusting the bead notes:

- `_fold_language.py` owns the shared glyphs/styles, computed `Fold: N/M` line, scanning tail, and attention-count
  style. Clan, family, tribe, and the shared roster all consume it.
- `_member_roster.py` has no collapsed-heading suppression parameters; every non-empty roster renders numbered entries
  and publishes jump targets at every level.
- Tribe summaries omit empty sections, render reply/slow-call presence at all levels, show bounded previews at triage,
  group full bodies at inspect, keep tracebacks and unbounded bodies for forensics, and request runtime statistics only
  at level 4. The render path reads cached snapshots only.
- Clan summaries hide unknown/empty disk sections behind one document-level scanning tail, omit empty tribe/count
  filler, and retain immediate in-memory context. Family summaries omit absent xprompt/prompt sections while keeping
  pending reply states.
- Tribe/clan disk access remains in coalesced `run_worker(..., thread=True)` workers, with the existing 10-second disk
  and 60-second statistics TTLs; no filesystem access was added to the summary builders.

Commits after the first Phase 1 commit were reviewed through current `origin/master`. The relevant later changes are
already compatible: tribe panel isolation and focus descent use the populated roster; concrete family member projection
landed before Phase 2 and is used by family rendering and jumps; metadata extreme toggling is covered by the final fold
contracts; metadata search consumes the rendered document; and two-character apostrophe jump hints update only
entry-jump state and the overlapping collapsed-panel golden, not numbered member jumps. Preserve all of that behavior.

The remaining integration defect is documentation committed while the epic was incomplete. `docs/ace.md` and
`docs/agent_families.md` still describe the old Pulse/Roster/Members ladder, imply tribe rosters begin at level 2, say
reply/slow-call enrichment begins only at level 3, and describe per-section `loading…` rows. Those claims now contradict
the shipped implementation.

Fresh verification on the integrated tree produced these signals:

- 124 focused summary, member-jump, fold-mode, and section-navigation tests passed.
- 17 clan/family/tribe/interactions/retry PNG tests passed together.
- After fast-forwarding the later two-character-hints commit, all 8 interaction PNGs and 30 overlapping jump/panel tests
  passed.
- The full `just check` passed every formatting, lint (including Symvision), SASE, and committed-plan stage and ran
  19,212 tests. Six load/environment sensitive failures all passed in focused reruns. Use `TMPDIR=/dev/shm` for the
  final full check: `/tmp` currently has no free inodes, while `/var/tmp` made the directory-mtime cache test flaky;
  `/dev/shm` has ample space and inodes and that test passes there.

## Reconcile documentation and epic traceability

Update both `docs/ace.md` and `docs/agent_families.md` so they explain the actual shared contract without regressing
navigation guidance added by later commits:

- Name the tribe levels glance, triage, inspect, and forensics. State that the compact numbered top-level roster and its
  digit jump targets exist at all four levels.
- Describe level 1 as the header, compact roster, attention previews, and headings/counts only for non-empty sections;
  level 2 as bounded previews for every represented section; level 3 as nested roster detail and grouped full bodies
  with bounds; and level 4 as unbounded bodies, tracebacks, richest annotations, and runtime statistics.
- Explain that reply and slow-call _presence_ enrichment is requested off-thread at every tribe level so empty sections
  can stay absent. Runtime statistics remain level-4-only. Unknown required content produces one dim
  `⋯ scanning member data…` document tail; known-empty content produces no section or placeholder.
- Make the clan guidance use the same unknown-versus-empty rule and scanning tail, while retaining the three-level
  roster/triage/full-body ladder.
- State in the family guidance that absent xprompt and prompt sections are omitted, while meaningful pending reply text
  remains visible. Family rosters and digit jumps remain present at both effective levels.
- Preserve the already-current `zZ` extreme-toggle behavior, direct fold levels, two-character apostrophe entry hints,
  and the distinction between apostrophe entry hints and fixed numeric metadata-member jumps.

Use `sase bead update ... --notes` to replace the stale commit notes for `sase-73.1` and `sase-73.3` with the canonical
commit hashes listed above; leave `sase-73.2` unchanged except to confirm it already resolves correctly. Re-run
`sase bead show` for all three children and confirm the source commit subjects still carry their bead IDs.

Validation for this phase:

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run Markdown formatting and targeted summary/navigation tests, including the cross-kind contract module, member
   jumps, fold mode, section navigation, and the clan/family/tribe/interactions PNG suites.
3. Run `TMPDIR=/dev/shm just check`. If a high-load test flakes, rerun the exact failure with the same `TMPDIR`; do not
   accept a failure in any file touched by `sase-73` or by this integration.
4. Search both docs for the stale Pulse/Roster/Members progression, level-3-only tribe presence loading, and per-section
   `loading…` guidance.

## Close and finalize sase-73

This is the final phase and must run only after the integration phase is fully green.

1. Re-run `sase bead show sase-73` and each child. Confirm every child remains closed, the corrected notes resolve to
   the canonical `master` commits, and the original plan still points to
   `sase/repos/plans/202607/smart_summary_folding.md`.
2. Close the original epic with `sase bead close sase-73`.
3. Only after the close succeeds, run `TMPDIR=/dev/shm just symvision`. The current `Justfile` has no `sase-73`
   epic-symbol entries, but closing can expose stale symbols or whitelist state. Follow the Symvision cleanup hierarchy:
   remove stale epic entries, delete genuinely dead code, or make file-local symbols private; do not add replacement
   whitelists merely to silence the linter. Do not disturb the active `sase-77` epic-symbol entries.
4. Re-run the exact Symvision command after any cleanup, then run the relevant tests and `TMPDIR=/dev/shm just check`
   after all main-repository changes.
5. As the last state change, set `status: done` in the frontmatter of
   `sase/repos/plans/202607/smart_summary_folding.md`. Access that sidecar via `sase repo open plans` and preserve the
   rest of the validated epic frontmatter and body.
6. Finish with `sase bead show sase-73`, validate that the durable plan reports `status: done`, and ensure both the main
   checkout and plans sidecar contain only the intended landing changes.

Do not close `sase-73` early. If documentation, traceability, targeted tests, or current-master integration reveal
another behavioral mismatch, resolve and revalidate it before beginning this final phase.
