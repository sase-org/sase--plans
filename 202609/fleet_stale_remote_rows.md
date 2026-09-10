---
tier: epic
title: Stop serving and rendering stale remote fleet rows
goal: 'A machine''s fleet API serves only the agents its own Agents list would present,
  with honest freshness and liveness, and viewers render honest statuses and authoritative
  counts - so a cleaned-up remote machine shows clean everywhere.

  '
parent_bead: sase-xe.16.11.7
phases:
- id: owner-scope
  title: Owner-side presentable snapshot scope and honest freshness
  size: large
  depends_on: []
  description: 'owner-scope: apply a core-owned presentable policy to the gateway
    fleet snapshot (demote dead active-tier records, bound recent completions, fold
    or drop orphaned members of dismissed families), replace hard-coded Fresh stamps
    with age-derived freshness plus an age-based snapshot rebuild, and make status
    bucketing liveness-aware.'
- id: dismissal-reconcile
  title: Close the dismissal identity leak and reconcile history
  size: medium
  depends_on: []
  description: 'dismissal-reconcile: extend the cleanup cascade to cover member records
    discovered from the index, add gc back-fill of dismissal identities for dead members
    of dismissed families, and surface silent index-sync failures.'
- id: core-ratchet
  title: Publish and adopt the new core surface
  size: small
  depends_on:
  - owner-scope
  - dismissal-reconcile
  description: 'core-ratchet: release the core changes, ratchet the revision pin and
    dependency floor, just install, and verify the published wheel contains the new
    surfaces.'
- id: viewer-honesty
  title: Liveness-aware rendering, authoritative banners, bounded requests
  size: medium
  depends_on:
  - core-ratchet
  description: 'viewer-honesty: make status projection consult liveness and connection
    health, source machine banners from authoritative counts, request the bounded
    terminal scope instead of include_terminal=True, make page merging generation-aware,
    and render observation age instead of stamped freshness.'
- id: live-proof
  title: Athena-to-Apollo verification of the repaired view
  size: small
  depends_on:
  - viewer-honesty
  description: 'live-proof: run gc reconciliation on both machines, verify athena''s
    apollo group matches apollo''s own presentation with honest counts, and prove
    dismissal and dead-agent transitions propagate on refresh and survive gateway
    restarts.'
proposed_by: bbugyi200.athena.0it
create_time: 2026-09-10 13:39:00
status: wip
bead_id: sase-xe.16.11.7.14
---

- **PROMPT:** [prompts/202609/fleet_stale_remote_rows.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fleet_stale_remote_rows.md)
- **BEAD:** [sase-xe.16.11.7.14](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.14.md)

# Plan: Stop serving and rendering stale remote fleet rows

## Problem

Athena's unified Agents list shows 58 "apollo" agents (banner: "58 agents · 29 running")
while apollo's own ACE shows none and `sase agent list` on apollo shows 1 running agent.
The user dismissed/killed everything on apollo days ago. Apollo is healthy:
`sase machine status apollo` returns `hello ok`, and the rows arrive marked
`offline · fresh`.

Live evidence gathered on 2026-09-10 (athena + apollo, both on sase
0.17.1+366.g4f6eb2b17 / core 0.33.0+4.gdc3d0a8b4):

- Apollo's `agent_artifact_index.sqlite` has 385 rows, 568 dismissed identities, and
  exactly 118 visible rows that escape `DISMISSED_NORMAL_VISIBILITY_FILTER`: 114 in the
  completed tier and 4 in the active tier. All are `ace-run` workflow member records
  (`*--gate`, `*--mon`, `*--code`, numbered members) whose `timestamp` never received a
  `dismissed_agents` identity when their families were dismissed.
- Athena's freshest federation worker cache entry (13:26 same day, live fetch) shows
  apollo serving 61 rows: lifecycle/liveness
  `terminal/dead 46 · failed/dead 12 · running/dead 2 · unknown/dead 1`, while the same
  payload's authoritative counts say `logical_agent_total=2, running=2`.
- The "29 running" banner is a client-side recount over projected rows that ignores
  `liveness: "dead"`; the authoritative counts are not used for machine group banners.
- Athena's own index has the same leak at scale (3,966 non-dismissed visible records,
  3,331 done + 635 "active", ~3,100 of them member records), so any peer viewing athena
  sees hundreds of stale rows (capped by the 512 serving limit).
- Apollo also served a 25-hour-old snapshot earlier the same day
  (`refreshed_at 2026-09-09 11:05` inside a payload fetched `2026-09-10 12:05`) with
  per-row and envelope freshness hard-coded `fresh`.

## Root cause chain

The binding contract is the one-list section of
`plan:202609/unified_agents_across_machines.md`: every origin contributes "active work,
pending decisions, unresolved operations, and the same bounded recent-completion
policy"; older history is explicit and paged; counts are honest and sourced from
authoritative summary counts, never loaded pages; stale data is never styled as a live
guarantee. Four distinct defects break it:

1. **Dismissal identities leak members (owner, core).** ACE cleanup plans targets from
   the rows it has loaded; the `agent_cleanup` planner cascade
   (`crates/sase_core/src/agent_cleanup/planner.rs`) only covers children present in
   that target list. Member/child `ace-run` workflow records (own timestamps) from
   earlier sessions or killed shells never get `dismissed_agents` identities
   (suffix-keyed, `crates/sase_core/src/agent_scan/index.rs`
   `DISMISSED_NORMAL_VISIBILITY_FILTER`), and nothing ever reconciles them. Locally they
   stay invisible (family folding + dead-PID filtering), so the leak is only observable
   through the fleet API.
2. **The owner-side fleet snapshot has no presentation policy (owner, gateway/core).**
   `build_snapshot_blocking` (`crates/sase_gateway/src/fleet_reads.rs` ~line 503) serves
   every visible index record (`include_active` unbounded + `include_recent_completed`
   limit 512) as a standalone agent row. No dead-PID demotion for the active tier (task
   bead sase-vb documents that tier keeping dead agents forever), no bounded
   recent-completion window, no family-member folding parity with the local list.
   Per-row and envelope freshness are hard-coded `Fresh` (`resolve_record`, ~line 650
   and ~line 564), and `current_snapshot` never rebuilds outside launch settlement or
   gateway restart, so a stale snapshot serves indefinitely while claiming freshness.
3. **Viewer status/count projection ignores liveness (viewer, Python).**
   `_status_from_summary` (`src/sase/ace/tui/models/_fleet_agents_rows.py` ~line 307)
   maps lifecycle/status without consulting `liveness` or `connection_health`, so a dead
   record renders RUNNING; `compute_banner_summary`
   (`src/sase/ace/tui/models/agent_groups/_tree.py` ~line 442) recounts those projected
   rows for the per-machine banner instead of using the authoritative counts (which
   correctly said running=2).
4. **Viewer requests and merges the wide window (viewer, Python).**
   `_fetch_fleet_catalog` hardcodes `include_terminal=True`
   (`src/sase/ace/tui/actions/agents/_fleet_refresh.py` ~line 288), and
   `merge_catalog_pages` (`src/sase/ace/tui/models/_fleet_agents_payload.py` ~line 148)
   unions pages across snapshot generations without dropping rows absent from a newer
   page, so vanished rows persist client-side.

Related beads: sase-vb (index active tier keeps dead agents forever), sase-kh (nothing
prunes hidden index rows). This epic fixes the fleet-visible consequences; it must not
silently close those beads.

## Constraints (binding for every phase)

- Shared policy belongs in `sase-core` (open it with `/sase_repo`); Python keeps
  presentation and thin adapters. Keep `sase_core` transport-free and independent of the
  gateway crate. Core verification runs the full `just check`/`scripts/check.sh`
  including PyO3.
- Never hide a potentially live agent. sase-vb's warning is binding: records carrying a
  live PID, a waiting marker, or genuinely undetermined liveness stay served and
  visible. Only records whose owner-side liveness is definitively Dead (or NotProcess)
  may be demoted, reconciled, or filtered.
- Dismissal stays a viewer-local operation on the owning machine; do not add a remote
  dismiss mutation and do not delete artifact records. Reconciliation adds dismissal
  identities or terminal demotion metadata; bundles and artifacts stay on disk.
- No new feature flags. This is a contract repair restoring already-shipped intended
  behavior; the old behavior does not need to stay reachable.
- Read `sase/memory/lint_and_test.md` before finishing any sase change, `tui_perf.md`
  before touching ACE refresh/render paths, and `symvision.md` for lint failures.
- Phase workers never create beads; record discoveries as `PROPOSED FOLLOW-UP:` notes on
  their own phase bead.
- Preserve laziness: zero enrolled machines still start no federation worker or timers;
  no new polling loops on keystroke paths.

## Phases

### owner-scope: Owner-side presentable snapshot scope and honest freshness (large)

In `sase-core` (gateway + core crates), make the served fleet snapshot match what the
owning machine's own list would show:

- Add a shared, core-owned "fleet presentable" policy applied when
  `build_snapshot_blocking` selects records: active-tier records whose owner liveness
  resolves Dead/NotProcess are not served as active work (demote to terminal
  presentation with their last observed status, or exclude when they fall outside the
  recent-completion window); terminal records are served under a bounded
  recent-completion policy (count- and age-bounded, matching the local windowed policy's
  intent) instead of a flat 512; member/child records carry their true family/kind
  metadata so viewers can fold them, and member records whose family root is dismissed
  are not served as standalone rows.
- Stop hard-coding `Fresh`: stamp real `observed_at`/build times, and derive envelope
  freshness from snapshot age (reuse `classify_cache_freshness`). Give
  `current_snapshot` an age-based rebuild policy so a gateway that has not rebuilt
  recently revalidates instead of serving a frozen snapshot forever; keep the existing
  refresh timeout and retain-previous-on-error behavior.
- Make status bucketing liveness-aware at the projection layer so a Dead record can
  never bucket as Running/Starting (the four-fact model presents it as WAS RUNNING with
  its observation age instead).
- Tests: snapshot excludes dead active-tier leftovers and orphaned members of dismissed
  families; bounded completion window; liveness-aware buckets; freshness reflects
  snapshot age; counts remain authoritative and page-independent.

### dismissal-reconcile: Close the dismissal identity leak and reconcile history (medium)

In `sase-core` (agent_cleanup + agent_scan) plus the thin Python callers:

- Extend the cleanup cascade so dismissing a family also records identities for its
  member/child records discovered from the index (by `parent_timestamp`/ `agent_family`
  lineage), not only children present in the loaded target list.
- Extend `sase agent index gc` reconciliation: back-fill dismissal identities (or
  terminal demotion) for visible member records whose family root is already dismissed
  and whose liveness is definitively dead, and for completed-tier records in the same
  state. Report counts; support a dry-run summary. This is the migration path for the
  existing leak (athena ~3.9k rows, apollo ~118 rows).
- The silent `try/except Exception: pass` wrappers around
  `sync_dismissed_agent_artifact_index` calls in the ACE dismissal path
  (`src/sase/ace/tui/actions/agents/_dismissing.py`, `_dismiss_memory.py`) must at
  minimum surface a diagnostic; a failed sync currently leaves rows visible to the fleet
  with no trace.
- Tests: cascade covers unloaded members; gc back-fill is idempotent, never touches
  records with live or unknown liveness, and leaves bundles/artifacts on disk.

### core-ratchet: Publish and adopt the new core surface (small)

Follow the release flow: land core changes, let release-plz publish, ratchet the sase
repo's core revision pin and dependency floor with the supported commands, run
`just install`, and verify the published wheel actually contains the new surfaces before
the viewer phase consumes them. Read `sase/memory/lint_and_test.md`.

### viewer-honesty: Liveness-aware rendering, authoritative banners, bounded requests (medium)

In the sase repo (ACE Python), consume the repaired contract:

- `_status_from_summary` consults `liveness`/`connection_health`: a dead record renders
  its terminal/WAS RUNNING presentation, never RUNNING.
- Per-machine group banners source their counts from the host's authoritative counts
  with honest scope wording (running/unknown), not from a recount of loaded page rows;
  keep local ("here") banner behavior unchanged.
- Request parity: stop hardcoding `include_terminal=True` for the default view; request
  the bounded recent-completion scope, keeping older history reachable through the
  existing explicit paging/continuation path.
- `merge_catalog_pages` becomes generation-aware: pages from an older
  `count_revision`/cursor generation never resurrect rows absent from the newer snapshot
  for the same host.
- Surface observation age instead of trusting stamped freshness: a cached or aged
  payload must not render the `fresh` chip (use the wire's `cached`/`age_seconds` and
  the new owner-side freshness).
- Tests: projection of a real serialized payload with dead RUNNING rows renders no
  RUNNING row and banner counts match the payload's authoritative counts; merge drops
  vanished rows; visual snapshot for the stale/WAS RUNNING presentation. Read
  `sase/memory/tui_perf.md` first; keep j/k budgets.

### live-proof: Athena-to-Apollo verification of the repaired view (small)

With both machines on the released builds (read `sase/memory/tailnet.md` first):

- Run the gc reconciliation on apollo and athena; record before/after visible-row counts
  from each index.
- Restart the managed gateways; from athena confirm the apollo group shows only apollo's
  real presentation set (its live agents plus bounded recent completions — on current
  state: the lone real running/recent dispatch agent, not 58 rows), the banner running
  count matches `sase agent list` on apollo, and rows survive a gateway restart without
  resurrecting.
- Dismiss a fresh terminal agent on apollo and confirm it leaves athena's view on the
  next refresh; confirm a genuinely running apollo agent still appears with honest
  liveness after its process is killed (WAS RUNNING/stale, then reconciled).
- Record pane evidence on the phase bead. This unblocks the stale-row portion of the
  open live-acceptance phase sase-xe.16.11.7.13; leave that phase's own record to its
  owner.

## Verification

- Full `just check` in both repos per phase; core phases run the complete check
  including PyO3 bindings.
- The live-proof phase is the acceptance gate: athena's apollo group must match apollo's
  own presentation, with honest counts and no dismissed/dead leftovers.
