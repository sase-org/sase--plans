---
tier: tale
title: Restore Agents status bucket priority
goal: 'The Agents tab''s by-status view once again puts actionable stopped and failed
  work first, running work next, waiting work immediately below running, and completed
  work last, while preserving launch-recency sorting and atomic clan/family rendering
  within every bucket.

  '
create_time: 2026-07-18 15:41:15
status: wip
prompt: 202607/prompts/restore_agents_status_priority.md
---

# Plan: Restore Agents status bucket priority

## Root cause and scope

The regression is in the Agents tab's presentation-layer bucket priority, not in status classification or clan
projection. Commit `b9d0e5371` added the desired newest-launch-first ordering within `BY_STATUS` buckets, but it also
changed `_STATUS_BUCKETS` in `src/sase/ace/tui/models/agent_groups/_buckets.py` from
`Stopped, Failed, Running, Waiting, Done, Starting` to `Running, Done, Waiting, Stopped, Failed, Starting`. The same
commit rewrote the model/widget expectations and `docs/ace.md` to describe that new order, so the current tests pass
while reproducing the screenshot.

`walk_order()` applies the fixed bucket sort key before its launch-recency key. That means the newer recency and
atomic-cluster code only orders display units inside a bucket; it did not move `Stopped` or `Waiting` across buckets.
Clan, family, and workflow projection should therefore remain unchanged. This is a Python/Textual presentation fix and
does not require a Rust core API change or a new refresh path.

## Restore the status priority without undoing recency ordering

Restore the Agents-tab `BY_STATUS` priority to `Stopped, Failed, Running, Waiting, Done, Starting` in the local
bucket-order definition and its explanatory comments. Keep `Starting` as the final internal bucket so the existing
hidden-starting-row behavior remains stable, and keep unknown future bucket names sorted after every known bucket.

Do not change the shared status-to-bucket mapping, integration/API ordering, filter semantics, or any
`STANDARD`/`BY_DATE` behavior. Preserve the July 18 launch-recency implementation in `_keys.py`, including root-anchored
recency, atomic clan/family/workflow clusters, deterministic missing/equal timestamp tie-breakers, shared
render/enumeration order, and the row-patch safety signature that rebuilds when a launch anchor changes.

Update the Agents grouping-mode documentation in `docs/ace.md` so both the bucket table and prose state the restored
priority while retaining the existing description of newest-first ordering within each bucket. Make the distinction
explicit: status priority chooses the bucket's position, and launch time only chooses positions among display units in
that same bucket.

## Regression coverage and validation

Update the focused model and widget regressions to assert the complete visible order
`Stopped, Failed, Running, Waiting, Done` from scrambled input, with `Starting` still omitted from rendered rows. Ensure
the model fixture includes all status buckets so the test proves that timestamps or input order cannot override bucket
priority. Keep or strengthen the existing within-bucket tests for descending launch recency, equal/missing timestamps,
rendered/enumerated banner agreement, and contiguous root-anchored families and clans. Adjust any atomic-cluster
ordering fixture whose cross-bucket sentinels were changed by the regressing commit, while continuing to assert that the
entire clan subtree renders as one unit.

Run `just install`, then the focused agent-group model and AgentList bucket tests, including the display-patch coverage
that protects recency-driven rebuilds. Finally run the mandatory repository-wide `just check`. The fix may legitimately
reorder a visual fixture that contains multiple status buckets; inspect any PNG diff and update a golden only when it
reflects exactly the restored bucket priority, with no unrelated layout or styling change.

## Risks and safeguards

The main risk is over-reverting `b9d0e5371` and losing its valid within-bucket recency or clan/family continuity
behavior. Limit production changes to the fixed bucket-priority definition unless a focused regression exposes a real
dependency, and use the existing recency, enumeration, patch-safety, and atomic-subtree tests as guards. A second risk
is documentation or widget tests continuing to bless the accidental order; sweep all `BY_STATUS` order strings and exact
banner-label assertions so code, tests, and user-facing docs agree.
