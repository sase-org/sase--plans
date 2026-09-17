---
tier: tale
title: Stop Agents-tab visible-set oscillation across load tiers
goal: Consecutive broad and bounded agent loads converge on a stable query-visible
  roster and tribe-panel set without recycling Tier 2 reconciliation.
size: medium
proposed_by: bbugyi200.athena.sase-127.1
bead: sase-127.1
status: done
---

- **PARENT:**
  [202609/agents_tab_flicker.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_flicker.md)
- **BEAD:**
  [sase-127.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-127/sase-127.1.md)

# Stop Agents-tab visible-set oscillation across load tiers

## Objective

Complete phase `sase-127.1` by making the Agents-tab roster converge after a broad,
query-specific load. A later bounded Tier 1 prefix must patch that established roster
instead of shrinking it, the full-history/query latch must settle rather than re-arm,
and the rendered tribe-panel keys must remain stable while an agent search query is
active.

## Current diagnosis and invariants

- `merge_incomplete_load_after_complete_history()` already recognizes
  `bounded_prefix && has_more` as a partial snapshot, but its cached-row loop can still
  discard members while applying dismissal/deletion filtering. A bounded read did not
  observe omitted rows and therefore cannot use omission as proof that they vanished.
- `_apply_loaded_agents_prepared_inner()` only records
  `_agents_complete_history_query_key` when `load_state.complete_history` is true. A
  requested Tier 2/index-revalidation result can currently be accepted even when the
  index completeness envelope says the history is incomplete, leaving the latch unset
  and allowing `_agents_history_reconcile_pending` to cycle.
- Query keys are isolation boundaries. A complete result for one committed query must
  not authorize merging or suppress reconciliation for another query.
- Deliberate removal remains authoritative: exact artifact-delta tombstones and explicit
  dismiss actions must continue to remove rows. The fix must not resurrect them merely
  to satisfy roster stability.
- Do not call a bounded or otherwise incomplete Tier 1 snapshot complete. If a
  full-history request receives an incomplete index result, obtain an actually complete
  source-backed snapshot (or an equivalently authoritative result) before latching it.

## Implementation

1. Reproduce the phase symptom first in focused tests around the prepared-apply
   boundary. Build a `FakeAgentApp` with an active search query and a broad/revalidated
   result containing matching agents in several tribes, apply a bounded Tier 1
   `bounded_prefix=True, has_more=True` subset, and demonstrate that current behavior
   can change the visible identities, query-key latch/reconcile state, or
   `panel_keys_for()` output. Keep assertions at the externally meaningful level:
   ordered visible identities, ordered panel keys (including an `epic` tribe), and the
   reconcile-pending flag.

2. Make the full-history loading contract truthful in
   `src/sase/ace/tui/models/_agent_loader_artifacts.py` and its public loader facade. A
   full-history/Tier 2 request may use the revalidating index fast path only when the
   returned completeness envelope proves `complete_history`. If that result is
   incomplete, fall back to the existing unbounded source scan and return a
   `tier="tier2", complete_history=True` load state. Preserve the bounded Tier 1 path,
   its caps, and its `complete_history=False` semantics.

3. Tighten `merge_incomplete_load_after_complete_history()` in
   `src/sase/ace/tui/actions/agents/_loading_compute_merge.py` so a bounded partial load
   is a monotonic patch over the cached visible universe. Incoming rows should still
   replace matching cached rows and add newly discovered rows, while cached rows omitted
   from the prefix remain. Apply only authoritative removal evidence appropriate to the
   load (not broad suffix-based dismissal/deletion inference caused by an incomplete
   prefix), and retain the existing artifact-delta tombstone behavior. Recompute tree
   relationships, hideable partitions, and capacity rows from the merged universe as
   today.

4. Keep the apply-state latch in `src/sase/ace/tui/actions/agents/_loading_apply.py`
   keyed to the committed search query. Once the truthful complete load lands, set
   `_agents_complete_history_query_key`, clear `_agents_history_reconcile_pending`, and
   ensure the immediately following same-query bounded load neither clears that latch
   nor re-arms Tier 2. Preserve the changed-query behavior: a partial result for a
   different key must invalidate/re-arm rather than reuse stale completeness.

5. Extend regression coverage in the existing loader, incomplete-merge, apply-boundary,
   and lazy Tier 2 reconcile test modules as appropriate:
   - an incomplete full-history index result falls back to a complete source snapshot;
   - a complete same-query apply followed by a bounded prefix has identical visible
     identities and identical `panel_keys_for()` output with an active query;
   - the complete query key remains latched and history reconcile stays unarmed;
   - changed-query partial loads still re-arm;
   - explicit dismissals and artifact-delta deletions still remove rows, while omission
     from a bounded prefix alone never does.

## Verification and completion

Run the focused regression modules first, including at least:

```bash
.venv/bin/pytest -q \
  tests/test_agents_tab_apply_boundary.py \
  tests/test_agents_tab_incomplete_merge.py \
  tests/test_agents_tab_artifact_delta_merge.py \
  tests/ace/tui/test_lazy_tier2_reconcile_apply.py \
  tests/ace/tui/actions/test_agent_loader_phase5_fallback_wiring.py
```

Then follow the required `lint_and_test.md` procedure for tracked SASE changes and run
`just check`. Before closing the phase, run `sase bead epic-symbols sase-127.1` and
resolve every listed symbol or re-key its Justfile entry to a still-open bead. Close
only this phase with:

```bash
sase bead close sase-127.1 --note "<focused tests and just check verified; visible identities, panel keys, and reconcile latch remain stable across the bounded apply>"
```

Do not close `sase-127` or any ancestor. If unrelated follow-up work is discovered,
record it on this phase as a `PROPOSED FOLLOW-UP:` note rather than creating a bead.
