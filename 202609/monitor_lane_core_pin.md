---
tier: tale
title: Finish the monitor lane index cutover
goal: CI uses the lane-scoped sase-core query and its regression test always runs.
size: small
proposed_by: bbugyi200.athena.sase-18e.land
bead: sase-18e
create_time: 2026-09-24 19:29:12
status: wip
---

- **PARENT:**
  [202609/codex_monitor_handoff_cutoff.md](https://github.com/sase-org/sase--plans/blob/main/202609/codex_monitor_handoff_cutoff.md)
- **BEAD:**
  [sase-18e](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18e/README.md)

# Finish the monitor lane index cutover

The `sase-18e` landing audit found one unfinished part of phase `sase-18e.2`.
`sase-core-revision.txt` still points to `6d0d0e6d5c0e783e68650f908e8c9eb01ceea7dc`,
which predates sase-core commit `20ac645a754167048b95f2edc9b0c6578ee5f290`. That commit
adds the `AgentSession` candidate filter used by `store.lane_monitor_records`. The old
CI pin makes the query degrade to a full project scan, and
`tests/monitor/test_monitor_store_lane_index.py` skips its critical assertion when the
field is absent.

## Work

1. Open the linked `sase-core` checkout with `sase repo open sase-core` and verify that
   the target revision is committed and available in the remote history. Move
   `sase-core-revision.txt` to that revision or a later committed revision that contains
   it, following `docs/rust_backend.md`. Do not change the published `sase-core-rs`
   version window.
2. Remove `_skip_unless_core_has_agent_session_filter` and its call from
   `tests/monitor/test_monitor_store_lane_index.py`. Remove imports used only by the
   skip helper. The lane-index test must execute against the pinned core instead of
   silently skipping.
3. Run the focused lane-index and monitor-start tests against the resulting core, then
   run `just check`. Repair any failures caused by this pin update.

The parent link to `sase-18e` returns its land agent to the interrupted verification and
follow-up triage after this tale lands.
