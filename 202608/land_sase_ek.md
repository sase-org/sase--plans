---
tier: tale
title: Finish and land epic sase-ek
goal:
  Complete the remaining repository-kind acceptance details, require the published fixed core, and close the epic with
  its plan finalized.
size: medium
proposed_by: bbugyi200.athena.sase-ek.land
create_time: 2026-08-03 08:33:17
status: done
---

# Finish and land epic `sase-ek`

## Objective

Complete the remaining acceptance work for epic `sase-ek`, raise the installed `sase-core-rs` safety floor now that
`0.17.14` is published, and perform the epic's required close, Symvision cleanup, and durable-plan finalization.

## Verified starting point

- `sase-ek.1` is closed. Its Rust commit is `3aa9d2a` in the linked `sase-core` repository. Tag `v0.17.14` contains that
  commit, and `cargo test -p sase_core` passes on the current core head.
- `sase-ek.2` is closed. Its host commit is `70410a05b`; it carries `RepoRecord.kind` through
  `ArtifactRefRepository.to_wire()`, covers shared prompt-bar/LSP enumeration and explicit sidecar resolution, and
  documents the exclusion.
- `sase-ek.3` remains in progress. PyPI now serves `sase-core-rs 0.17.14`, but `pyproject.toml` and `uv.lock` still
  select `0.17.13`. A focused run of `tests/artifact_refs/test_context.py` and
  `tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py` therefore produced 8 passes and the expected one
  failure: the old installed core still offered the newest `plans` sidecar commit.
- The approved phase-1 plan required a named sidecar-kind constant/predicate and a shared inventory doc-comment update.
  The implementation currently uses an inline `repository.kind == "sidecar"` check and the doc comment does not describe
  the non-sidecar policy. Its tests also use a generic `human-code` kind rather than explicitly proving `primary`,
  `linked`, and `external` all remain eligible. These acceptance details must be completed as epic work.
- Every epic and child note was reviewed. No note contains a `PROPOSED FOLLOW-UP:` entry, so there is no follow-up task
  to create or decline.
- Commits after the first epic implementation commit were reviewed on both current heads. Later Rust work is bead-prefix
  migration; later host work is suite isolation, asynchronous sidecar publication, agent-CLI history, bead-prefix
  migration, ACE clan hints, and bead-show formatting. None duplicates, conflicts with, or needs conversion to the
  repository-kind completion policy. The host checkout equals `origin/master`.

## Phase 1 — Complete the Rust acceptance contract

1. Open the linked `sase-core` repository through `/sase_repo`; do not alter crate versions or release-plz-managed
   dependency pins.
2. In `crates/sase_core/src/editor/completion.rs`, replace the inline sidecar string comparison with the approved named
   policy constant and a small predicate, and use that predicate before checkout probing or `git log`.
3. Update the public inventory doc comment to state that commit enumeration covers non-sidecar repositories and that SDD
   sidecars are excluded because they are machine-written stores.
4. Strengthen the existing commit-inventory test so `""`, `primary`, `linked`, and `external` repositories are all
   explicitly retained while `sidecar` is excluded. Preserve the existing sidecar-only, row-cap, newest-sidecar, and
   explicit-resolution coverage.
5. Run `cargo fmt --all --check` and `cargo test -p sase_core`. Reopen `sase-ek.1` before making this acceptance fix,
   then close it normally with a note describing the constant, predicate, documentation, test matrix, and successful
   Rust checks.

## Phase 2 — Raise and verify the published core floor

1. In the primary `sase` repository, change the dependency to `sase-core-rs>=0.17.14,<0.18.0` and regenerate `uv.lock`;
   confirm both the project requirement and locked package version are `0.17.14`.
2. Run `just install` before any project check. Rerun the two focused test files above and confirm the sidecar row is
   absent and explicit sidecar resolution still succeeds.
3. Run the required full `just check` and address only regressions caused by this epic. If an unrelated failure exposes
   genuinely new work, use `/sase_new_task` before recording it; if it is caused by this epic, finish it here instead.
4. Close `sase-ek.3` normally with a note recording that PyPI serves `0.17.14`, the lock/floor now select it, the
   pre-bump focused failure was reproduced, and the post-bump focused/full checks pass.

## Phase 3 — Land the epic

1. Re-run `sase bead show sase-ek` and each child show. Confirm all descendants are closed, all notes remain addressed,
   and no new `PROPOSED FOLLOW-UP:` was added. Adjudicate any newly added proposal with `/sase_new_task` and record its
   exact outcome in the close note.
2. Close the epic with `sase bead close sase-ek --note "..."`. The note must identify the verified phase commits, the
   `v0.17.14`/PyPI provenance, the fixed Rust acceptance omissions, the `0.17.14` dependency/lock update, the focused
   and full verification, the later-commit integration scan, and that the audit found no proposed follow-ups. Do not use
   `--force` merely to make closure succeed; if closure names an unfinished phase, finish or deliberately reopen and
   resolve it under the documented lifecycle.
3. After the epic is closed, run `just symvision` if the recipe exists. Before fixing any reported Symvision issue, read
   `symvision.md` through `/sase_memory_read`; remove stale `sase-ek` whitelist entries and unused code it reports. If
   this changes files, rerun `just check`.
4. Finally, open the plans sidecar through `/sase_repo` and change only the linked plan
   `202608/commit_completion_excludes_sidecars.md` frontmatter from `status: wip` to `status: done`. Confirm
   `sase bead show sase-ek` reports a normal `done` closure and inspect all touched repository statuses for a clean,
   intentional handoff.
