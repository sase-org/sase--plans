---
tier: tale
goal:
  Ensure the sase_fix_just chop stops launching new repair workflows whenever a
  same-family nonterminal ChangeSpec, including a reservation stub, already exists.
create_time: 2026-09-09 19:53:12
status: wip
---

# Plan: Restore the sase_fix_just ChangeSpec guard

## Context and diagnosis

The `sase_fix_just` script chop in the linked chezmoi repository reads the lumberjack's
`all_changespecs_file` snapshot and guards launches by matching the `sase_fix_just_`
ChangeSpec prefix. Its current status allowlist considers only `WIP`, `Draft`, `Ready`,
and `Mailed` blocking. SASE's branch-name allocator also creates `STATUS: Reserved`
ChangeSpec stubs while a PR name is claimed. The live `run_every` snapshot contains many
matching `sase_fix_just_tests_*` entries in that state, so the guard ignores them and
can launch another workflow; it only appears healthy while a separate matching `Draft`
entry is present.

This is a latent status-model mismatch rather than a snapshot or prefix failure:
reservations predate the chop, but the original implementation and test fixture never
included them. The safe contract is to permit another launch only when every matching
ChangeSpec is in a known terminal lifecycle state, instead of maintaining an incomplete
list of blocking states.

## Implementation

1. Update `home/bin/executable_sase_chop_sase_fix_just` in the linked chezmoi repository
   so a matching ChangeSpec blocks unless its normalized status is one of SASE's
   terminal states (`Submitted`, `Archived`, or `Reverted`). Preserve the existing
   prefix match, status-decoration normalization, fail-safe snapshot handling, compact
   diagnostics, and successful terminal-only launch behavior. This makes `Reserved`,
   missing, and future nonterminal statuses fail closed.
2. Extend `tests/bash/sase_fix_just_chop_test.sh` with regression coverage for
   `Reserved` entries from the real snapshot shape and for an unrecognized/nonterminal
   status. Keep the existing coverage proving terminal entries allow a launch and
   malformed or unavailable snapshots skip safely.
3. Run the focused bashunit test for the chop, then the chezmoi repository's bash test
   target (and broader test target if practical) to catch integration regressions.
   Re-run the live snapshot guard in a no-launching scenario to confirm its diagnostics
   include the reserved matches rather than relying only on the remaining `Draft` entry.

## Risks and boundaries

Treating unknown statuses as blocking is intentionally conservative for a scheduled
autonomous launcher: a new lifecycle state may delay a run, but cannot create duplicate
repair agents or more orphaned reservations. Terminal history remains nonblocking, so
already-submitted, archived, and reverted work will not permanently disable the chop.
This change does not delete existing reservation stubs or alter SASE's reservation
cleanup machinery; it restores the chop's launch guard only.
