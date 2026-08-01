---
tier: tale
title: Land reliable notification snoozing and resurfacing
goal:
  The released core dependency, cross-surface behavior, follow-ups, epic closure, Symvision cleanup, and durable plan
  status are fully integrated and verified.
proposed_by: bbugyi200.athena.sase-cy.land
bead: sase-cy
create_time: 2026-08-01 09:07:05
status: done
---

- **PROMPT:** [202608/prompts/land_reliable_notification_snoozing.md](prompts/land_reliable_notification_snoozing.md)
- **PARENT:**
  [202608/reliable_notification_snoozing.md](https://github.com/sase-org/sase--plans/blob/main/202608/reliable_notification_snoozing.md)
- **BEAD:** [sase-cy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-cy/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-cy.land](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-cy.land.md)
- **COMMITS:**
  - [9cf08e7](https://github.com/sase-org/sase/commit/9cf08e739663dcc62d91e5794bcaebfb6fe7d274) — build(deps): require
    core 0.17.5 for snoozing

# Plan: Land Reliable Notification Snoozing and Resurfacing

## Objective

Finish and land epic bead `sase-cy` after correcting the one integration defect found by its land-agent audit: the main
package still permits and locks `sase-core-rs` 0.17.4 even though the canonical snooze-expiry bindings first shipped in
0.17.5. Verify the minimum supported published core, re-run the cross-repository notification matrix, close the
already-filed dependency task, and perform the epic's close/symvision/plan-status sequence.

Do not broaden this tale into notification redesign. Preserve unrelated worktree changes if any appear. Do not invoke
the git commit workflow unless separately authorized or triggered by the normal post-completion finalizer.

## Audit evidence already established

- `sase bead show sase-cy` reports four closed phase children: `sase-cy.1` through `sase-cy.4`. Their source and tests
  were reviewed, and each phase's completion note maps to committed behavior.
- Main-repository epic commits are `09517a0f` (`sase-cy.1`), `459ef978` (`sase-cy.3`), `38c57e17` (`sase-cy.2`), and
  `7163200f` (`sase-cy.4`). Linked `sase-core` commits are `a856b665` and `64d4d4cf`; linked `sase-telegram` commits are
  `c9c9af6d` and `33ada2ab`. The full commit bodies carry the bead IDs.
- The reviewed implementation has the intended state contract: Rust validates/normalizes future aware deadlines,
  atomically expires due or malformed legacy snoozes to unread resurfaced activity, skips dismissed/permanent mutes,
  projects the next deadline, and preserves concurrent append/read integrity. Main CLI/mobile/ACE and Rust gateway use
  current-state reads and activity ordering/cursors. Telegram uses a versioned `(activity_at, id)` cursor, oldest-first
  sending, and stops before advancing past a failed delivery.
- Post-start commits were audited in all three repositories. Overlapping main commits for declared notification panels,
  notification docs/snapshots, and Admin Center state are already ancestors of the later epic commits, so the final
  source contains both behaviors. The intervening core 0.17.5/0.17.6 releases and Telegram 0.4.6 release introduce no
  conflicting source changes.
- The actual integration defect is already tracked by ready task `sase-d7`: `pyproject.toml` says
  `sase-core-rs>=0.17.4,<0.18.0`, `uv.lock` resolves 0.17.4, and clean resolution to that wheel deterministically breaks
  the new snooze facade/tests. Local development checks masked this because `just install` builds the linked core. Core
  changelog/release history confirms `a856b665` first shipped in `sase-core-rs` 0.17.5.

## Phase 1: Integrate the released core contract

1. Re-read the SASE bead memory through `/sase_memory_read` before mutating beads. Open the linked `sase-core`,
   `sase-telegram`, and `sase--plans` repositories through `/sase_repo`; use only the paths returned by that workflow.
   Fetch/read current base-branch history and recheck all relevant worktrees before editing.
2. In the main repository, raise the published `sase-core-rs` lower bound from 0.17.4 to 0.17.5 while retaining the
   `<0.18.0` upper bound. Regenerate `uv.lock` through the normal `uv` lock command so it no longer resolves an
   incompatible 0.17.4 wheel. Do not hand-edit lock records.
3. Prove the lower bound rather than only testing the newest linked core. Temporarily install the published
   `sase-core-rs==0.17.5` wheel into the main test environment, confirm the installed distribution version, and run the
   focused facade/store/catalog/mobile/snooze matrix with sync disabled so `uv` cannot silently replace it. At a minimum
   cover:
   - `tests/test_core_facade/test_notification_store.py`
   - `tests/notification_store/test_mute_snooze.py`
   - `tests/test_notification_catalog.py`
   - `tests/test_mobile_notifications_bridge.py`
   - `tests/notification_store/test_snooze_e2e_matrix.py` Restore the linked development core afterward with
     `just install`.
4. Re-run proportional cross-repository verification:
   - Main: notification/ACE deadline, mutation, CLI, mobile, sort, and end-to-end suites, then mandatory
     `just install && just check` because main files changed.
   - `sase-core`: `cargo fmt --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
     `cargo test --workspace`.
   - `sase-telegram`: `just install && just check`, plus run `tests/test_snooze_resurface_e2e.py` against the current
     main/core environment if the standalone compatibility environment skips it. The repository's CI already checks out
     current `sase` and `sase-core`, builds the core binding, and installs current `sase` before testing, so a local
     compatibility skip is not itself a new CI defect.
5. Close ready task `sase-d7` with a note naming the raised floor, regenerated lock, minimum-wheel proof, and final
   checks. If current base history already fixed it, verify that exact result and close the task as absorbed rather than
   duplicating edits.

## Phase 2 (final): Land the epic

1. Re-run `sase bead show sase-cy` and each child to ensure no child was reopened and every note remains accounted for.
   Recheck current base-branch commits since the epic's first commit and integrate any newly arrived notification
   overlap before proceeding.
2. Reconcile every collected `PROPOSED FOLLOW-UP:` without duplicates:
   - The strict SDD fixture proposals from `sase-cy.1`, `.2`, `.3`, and `.4` were filed/fixed as `sase-d0` by commit
     `58948eb9`; do not file again.
   - The suite-gate contention proposals from `sase-cy.1` and `.2` were filed/evaluated as `sase-cf` (closed canceled
     after repeated non-reproduction); do not file again unless the exact failure now reproduces with new evidence.
   - The OpenCode temp leak from `sase-cy.2` is already in-progress task `sase-d5`; do not file again.
   - The Config Center PNG drift from `sase-cy.4` was filed as `sase-d8` and handed to the Admin Center follow-up work;
     do not duplicate it. Re-run the exact visual node and record its present result in the close note.
   - The stale local Telegram editable-install proposal from `sase-cy.4` does not merit another task because current
     Telegram CI installs current main and core before its E2E run; verify the four E2E tests in a current environment
     and record that rationale.
   - The core-floor issue is task `sase-d7` and must have been completed in Phase 1. File a new ready task bead only for
     a genuinely new, worthwhile follow-up not covered above; its description must name the proposing child bead.
3. Close the epic normally with `sase bead close sase-cy --note "..."`. The note must summarize source/commit
   verification, post-start integration, dependency-floor correction and minimum-wheel proof, final test results, and
   why no duplicate follow-up beads were filed. If closure is rejected for an unfinished descendant, investigate and
   finish or reopen it. Never force a `done` close merely to succeed; use
   `--force --reason ... --resolution canceled|superseded` only if evidence establishes that deliberate outcome.
4. Only after the epic closes, run `just symvision`. Remove stale `sase-cy` epic-symbol whitelist entries and genuinely
   unused code it reports. If this changes main-repository files, run `just install && just check` again and append a
   bead note with the post-close verification result.
5. Through the audited `sase--plans` checkout, change only the epic plan `202608/reliable_notification_snoozing.md`
   frontmatter from `status: wip` to `status: done`. Verify the main, linked, and plans worktree diffs and report all
   changed files and checks.

## Completion criteria

- A clean dependency resolution cannot select a core older than 0.17.5, and the focused snooze contract passes against
  the published 0.17.5 minimum wheel.
- Main, Rust core, and Telegram checks pass with the real end-to-end resurface tests exercised in a current environment.
- `sase-d7` and `sase-cy` are closed with evidence-rich notes; no unfinished epic descendant was forced closed.
- Post-close Symvision is clean, with any expired epic whitelist entries removed.
- The durable epic plan has `status: done`, and every proposed follow-up is either linked to its existing bead/filed as
  a new ready task or explicitly explained in the epic close note.
