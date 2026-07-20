---
tier: tale
title: Converge hooks suffix-transform writes
goal: Stop terminal ChangeSpec suffix cleanup from restoring markers removed earlier
  in the same scheduler cycle, while making repeated cleanup a true no-op.
create_time: 2026-07-20 16:36:50
status: wip
prompt: 202607/prompts/converge_hooks_suffix_writes.md
---

# Plan: Converge hooks suffix-transform writes

## Context and outcome

The suffix-transform scheduler currently mixes two persistence models. Individual old-entry hook cleanup re-reads the
ChangeSpec under its file lock, but terminal-status cleanup derives a replacement HOOKS field from the scheduler's older
parsed snapshot and writes that list wholesale. When both transforms run in one cycle, the later terminal cleanup can
restore an error marker that the earlier fresh-read update removed, creating an endless strip/restore loop and repeated
success logs. The terminal comment cleanup has the same stale-list shape; commit suffix cleanup already performs its own
locked read-modify-write.

Keep this work in the Python ChangeSpec persistence and scheduler layers: the wire representation and cross-frontend
domain model do not change, so no Rust-core API change is needed.

## Implementation

1. Add a hook-list transformation primitive in the hook persistence module. It will acquire the ChangeSpec lock, parse
   the target ChangeSpec from disk inside that lock, invoke a pure transformation against the current hook list, and
   write only when the returned list is materially different. Missing targets, unchanged transforms, lock timeouts, and
   failures must return a non-success result without producing a write.
2. Route hook suffix-type mutation through that primitive so targeted and bulk suffix transforms share the same
   freshness and idempotence contract. Preserve the public boolean behavior: `True` means an actual persisted mutation,
   while already-clean or no-longer-matching state is `False`.
3. Rewrite terminal-status hook cleanup to transform the freshly read hooks rather than the `ChangeSpec` snapshot passed
   into the scheduler. Generate update messages only for lines actually changed in that locked transform, so a second
   pass reports zero updates and the scheduler stops emitting false success lines.
4. Apply the same fresh-read discipline to terminal comment suffix cleanup, which currently rewrites the complete
   COMMENTS field from a caller-supplied snapshot. Preserve unrelated comments and concurrent changes while clearing
   only the currently matching reviewer's marker. Confirm that COMMITS already use a fresh locked read and leave that
   path unchanged.
5. Keep running-agent semantics intact: terminal hook `running_agent` markers become `killed_agent`, hook errors become
   plain suffixes, and terminal comment error/running-agent markers are cleared exactly as before.

## Regression coverage

- Add an integration-style stale-snapshot regression using a temporary ProjectSpec: parse and retain a stale terminal
  ChangeSpec, perform the old-entry fresh hook strip, then run terminal cleanup with the stale object. Assert the
  stripped marker remains plain, unrelated/current hook data is preserved, and a second terminal cleanup returns no
  updates and does not rewrite the file.
- Extend focused persistence tests for changed versus unchanged hook transforms and terminal running-agent conversion.
- Cover comment cleanup with a stale caller snapshot so an unrelated on-disk comment mutation is preserved and repeated
  cleanup is idempotent.
- Run the focused suffix-transform and hook/comment persistence tests during development, then run `just install`
  followed by the repository-required `just check` before completion.

## Completion

After all checks pass, update only bead `sase-8g.1` to `closed`. Do not close its parent epic and do not create
additional beads.
