---
tier: epic
title: Complete fleet snapshot correctness and released live acceptance
goal: "Remote fleet reads accept ordinary prompts, preserve safe dismissal and bounded
  presentation with explicit history, merge only coherent snapshots, and render truthful
  family/status/count evidence on verified released builds with live Athena-to-Apollo
  acceptance.

  "
parent_bead: sase-xe.16.11.7.14
phases:
  - id: payload-safety
    title: Make owner-generated fleet labels valid and repair contract fixtures
    size: medium
    depends_on: []
    description:
      "payload-safety: normalize owner-produced display intent and repair correlated
      wire fixtures with projection and gateway regressions."
  - id: dismissal-parity
    title: Complete unloaded-family dismissal and protected-liveness guarantees
    size: medium
    depends_on:
      - payload-safety
    description:
      "dismissal-parity: prove unloaded-member cleanup, preserve live and protected
      records, and surface index-sync failures."
  - id: catalog-snapshots
    title: Separate bounded presentation from history and identify real snapshots
    size: large
    depends_on:
      - dismissal-parity
    description:
      "catalog-snapshots: make older history explicitly pageable and bind catalog
      continuation and shared merging to genuine snapshot identities."
  - id: released-builds
    title: Repair actual release-plz packaging and prove the published core surface
    size: medium
    depends_on:
      - catalog-snapshots
    description:
      "released-builds: repair release-plz packaging, publish the repaired core, ratchet
      the combined-tree pin and floor, and verify real wheels."
  - id: viewer-integration
    title: Integrate snapshot, family, and count evidence into the current Agents UI
    size: medium
    depends_on:
      - released-builds
    description:
      "viewer-integration: consume snapshot policy, family lineage, observation age, and
      authoritative counts in current grouping, queries, and rendering."
  - id: live-acceptance
    title: Prove the repaired Apollo view, dismissal propagation, and restart behavior
    size: medium
    depends_on:
      - viewer-integration
    description:
      "live-acceptance: verify released Athena-to-Apollo presentation, controlled
      dismissal and death transitions, history paging, restart resilience, and
      combined-tree checks."
proposed_by: bbugyi200.athena.sase-xe.16.11.7.14.land
create_time: 2026-09-10 19:57:57
status: wip
---

- **PROMPT:**
  [prompts/202609/fleet_remaining_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fleet_remaining_acceptance.md)
- **PARENT:**
  [202609/fleet_stale_remote_rows.md](https://github.com/sase-org/sase--plans/blob/main/202609/fleet_stale_remote_rows.md)

# Complete fleet snapshot correctness and released live acceptance

## Outcome and scope

Finish the unmet requirements of `sase-xe.16.11.7.14`, whose five phase beads are closed
but whose actual live acceptance and published adoption remain incomplete. The default
remote Agents view must match the owner's presentable set, keep older history explicitly
reachable, never resurrect removed rows through paging, and show honest liveness, family
structure, observation age, and authoritative counts.

This child implements only the remaining work. Its `parent_bead` is the handoff to the
interrupted fleet landing. Do not create a phase for closing that parent, running its
post-close Symvision pass, or marking its original plan done. Do not close any ancestor
from a phase worker.

Read the original acceptance contract through
`sase artifact read plan:202609/fleet_stale_remote_rows.md`, and the detailed land audit
through `sase artifact read file:explicit:e2c64521530457f9913aa9aa`. The latter artifact
was minted successfully, but its explicit bead attachment failed because the plans
publisher had unpublished commits; this literal reference preserves access to it. The
parent bead's landing note records the same blockers.

## Audited starting point

SASE master `2dcd6a136`, core master `e0f105d`, audited 2026-09-10. Epic commits: SASE
`f22339c18` and `cef3ea98d`; core `270e501`, `70df126`, and `93281c3`. Preserve the
delivered presentation policy, age-based rebuild, gc reconciliation, liveness buckets,
and core/Python adapters.

Confirmed probes on current SASE source and the installed development core:

- A normal two-line prompt fails projection with
  `intent must not contain control characters`. This also blocked Apollo hello/summary
  during phase .5. The helper predates .1 (`ee7163e` introduced it), correcting the
  phase's attribution; it still blocks this epic's acceptance.
- Merging `(generation=g,sequence=1,rows=[removed])` with `(g,2,[current])` returns both
  rows. A late older generation replaces current rows. The gateway generation identifies
  the event store lifetime, not each rebuild.
- A known WAITING-only host with authoritative total=1/running=0 renders
  `1 agent · 1 unknown` because the banner invents unknown as total minus running.
- A test fixture overrides a projected root summary to `row_kind=monitor` without
  updating `family_role`; core rejects it with the new consistency invariant.

Source review additionally confirms that catalog continuation only pages the already
bounded snapshot vector, so it cannot reach older excluded history; new family_role and
parent_timestamp are not consumed by the Python Agent projection; offline host health is
not an input to status projection; ordinary False sync results remain silent at ACE
dismissal callers.

At audit time PyPI's latest core is 0.33.0 (uploaded before this epic). Core
[Release-plz run 34543186306](https://github.com/sase-org/sase-core/actions/runs/34543186306)
fails packaging because its generated worktree lacks a sase_gateway version requirement,
even though core Cargo.toml already contains the earlier fix. SASE still declares
`sase-core-rs>=0.33.0,<0.34.0` and pins `da0a738`.

## Constraints and verification discipline

- Open core and any other repository through `/sase_repo`. Use the returned path; never
  assume another worker's directory. Read all applicable AGENTS instructions.
- Shared presentation, liveness, snapshot/paging, and count policy belongs in Rust core,
  with gateway transport and thin Python bindings/adapters. Python owns Textual
  presentation and orchestration, not a second backend implementation.
- Preserve potentially live, Unknown, waiting, and pending-question records. Never infer
  death merely from disconnection or viewer cache age. Preserve artifact data; dismissal
  remains owner-local and no remote dismiss mutation is introduced.
- This is the original contract repair: no new feature flags. Consult flag memory before
  touching a flag if current release integration proves one is unavoidable.
- Read `lint_and_test.md`, `symvision.md`, and `tui_perf.md` with `/sase_memory_read`;
  read `cli_rules.md` before adding a CLI option. Keep zero-host startup lazy,
  network/index work off the Textual pump, existing refresh coalescing, bounded paging,
  and j/k responsiveness.
- Restore the checkout's editable core build with the supported install workflow before
  tests. Its extension was missing at audit time. A runtime dev import whose package
  metadata says 0.33.0 is not proof of the published wheel.
- Core changes require full `just check`/`scripts/check.sh`, including PyO3. Use
  `/sase_monitor` for long commands. The known loader omission is task `sase-xv`: until
  fixed, derive the selected interpreter's library directory and preserve any existing
  LD_LIBRARY_PATH for verification; do not skip binding tests.
- SASE changes require `just check`; combined-tree landing requires `just check-full`
  through `/sase_monitor` using TESTING/TESTED. Run relevant fleet PNG acceptance and
  inspect intentional golden changes. Do not characterize a blocked run as a pass.
- Phase workers record discoveries as PROPOSED FOLLOW-UP notes, never new task beads. Do
  not close a phase with unmet acceptance or a promise that a later worker will finish
  that same phase. Use monitor continuations for publication waits.
- Host finalizers own commits. Release-plz owns versions; obey core's restriction on
  manual release-version changes. A local cargo package pass alone is insufficient
  because the current failure occurs inside release-plz's generated worktree.

## Phases

### payload-safety

Repair owner-generated summary labels in core `fleet_contract.rs` and the gateway
snapshot path so ordinary multiline prompts cannot make a host's hello, summary,
catalog, and detail APIs fail validation. Normalize unsafe control characters in
owner-produced display intent before UTF-8-safe byte bounding; omit a resulting empty
optional label. Cover both raw_prompt_snippet and plan_action. Preserve strict external
validation and existing secret/local-path safeguards. One malformed display value must
not silently erase the host's whole presentable set.

Add projection and gateway regressions with newline, CR/tab, Unicode and byte limits,
plus invalid external-wire coverage. Correct the actual correlated-field fixture
overrides in `tests/test_fleet_contract_counts_sase_core_rs.py` and sibling contract
tests. The helper `_summary_for_agent` already projects via core; do not add a dummy
field only at the helper and overlook row_kind overrides. Re-run the ten ACE fleet nodes
named by the parent's own discovered-issue note against the current core. Record their
disposition and full binding verification evidence.

### dismissal-parity

Complete the parent's normal-dismissal acceptance using index-discovered lineage, not
just the loaded UI target list. Start with a production-path regression where a
dismissed root has unloaded terminal and dead-active members; ensure their identities
are reconciled immediately and stay reconciled after projection replacement, gc, and
restart. Preserve the existing bounded lineage lookup and conservative handling of
unresolved ancestry.

`record_is_definitively_dead_for_dismissal_backfill` currently returns True for done
markers before checking protection, and skips any PID/running marker without resolving
it. Use shared trustworthy liveness/protection facts to reconcile truly dead active
leftovers while preserving live/Unknown, waiting, and pending-question records,
including contradictory/stale marker combinations. Keep artifact files intact.

Test dry-run/apply parity and idempotence, including a root dismissal not yet projected
into the index. Surface failed False returns as well as exceptions from index sync at
`_dismissing.py` and `_dismiss_memory.py`; absorbed errors must not leave the user with
a silent success. Validate unloaded-member behavior end to end through the actual
cleanup/sync path rather than only mocking the reconciliation call.

### catalog-snapshots

Finish the default-scope/older-history contract in core and gateway. The default view
includes active/protected work plus the bounded 200-row/seven-day recent tier. Provide
an explicit paged history scope that can reach records outside that tier without making
default refresh load the archive. Extend and validate the existing typed catalog
query/response and bindings as needed; preserve filters, limits, authoritative summary
counts and lazy per-host continuation. Update the gateway contract manifest. Include
tests with over 200 completions and records older than seven days proving both default
exclusion and explicit-history reachability.

Define snapshot identity that changes when the served set changes, including an
age-driven rebuild with no event/count-revision change. Event-store generation is not a
snapshot identifier. Continuation must be bound to one snapshot: either serve that
snapshot consistently or report a typed reset requiring the viewer to restart that
host's page accumulation. Do not compare opaque generations lexicographically.

Put shared page-merge/acceptance rules in core and expose a thin binding for viewers.
Reject stale arrivals using trustworthy request/snapshot evidence, never an assumption
that the second argument is newer. Same-snapshot pages union without duplicates and
allow current revisions to replace stale copies; a replacement snapshot drops absent
rows and obsolete cursors/count inputs. Cover same-generation newer snapshots,
out-of-order responses, gateway restarts, equal-count membership changes, empty
snapshots, and a rebuild during paging. Preserve retain-previous-on-error freshness.

### released-builds

Repair and verify the actual core release-plz path. Inspect current release state and
the active `sase-z4.6.5.4.5` published-floor work before changing shared release
infrastructure; adopt any already-landed repair instead of duplicating it. The known
failure is reproducible in Release-plz run 34543186306 after `93281c3`; investigate the
gateway's excluded release configuration and generated manifest handling. Reproduce the
release-plz packaging path, preserve package ownership, and let the supported release
workflow publish a version containing all preceding core phases.

Use `/sase_monitor` to wait and resume through release/CI completion, fixing real
release blockers within the authorized workflow. Then ratchet the SASE revision pin and
supported dependency window with the existing tools, preserving newer lineage schema 27
and artifact-link binding requirements from the combined tree. Run `just install` and
the binding/environment validators.

Install the actual newly published wheel in a clean environment with no editable core or
development override. Prove the summary fields/invariants, reconciliation,
snapshot/history contract, gateway/federation binaries and fleet behavioral smoke. Add
durable binding-floor checks as needed. Record exact core commit, release version, wheel
provenance and validation results. Do not close on an unpublished version or substitute
local development installs for this gate.

### viewer-integration

Consume the repaired released contract in `_fleet_refresh.py` and the fleet payload and
row adapters. Replace the Python-only generation comparison with the shared policy,
preserving request-generation guards and per-host cursor reset. Use the bounded default
scope and existing explicit user-driven history affordance; complete that affordance if
the current code has no path to request the new history scope.

Carry family_role/parent lineage into the common Agent representation and existing
family folding with origin-qualified identity. Preserve root/member/monitor/gate/proc
and historical-shell semantics without hiding potentially live members. Integrate with
the newer `agent_live_query.py`/agents-live profile so status, role and machine queries
see the same facts as grouping and rendering; add remote/local parity tests.

Render Dead/NotProcess rows as their observed stopped/terminal state. For offline or
stale observations, show qualified last-known status and observation age without
claiming current liveness. Preserve specific pending attention. Prefer actual row
observed_at, conservatively combining owner and viewer age/health evidence.

Use authoritative count/coverage evidence for machine L0 banners. Do not infer unknown
as total minus running: queued/waiting/failed/done are known states. Missing counts or
empty loaded pages must not fabricate zero, completeness, or current running counts.
Preserve the newer BY_MACHINE status subgroups and their correctly scoped counts, plus
render-cache invalidation when only host count/health evidence changes.

Use the post-`8d9f24833` split fixtures (`_fleet_summary_fixture.py`,
`_fleet_response_fixture.py`, etc.), the post-`16001bb36` split build helpers, and the
current capacity/usage chrome. Add serialized-payload regressions and inspect a PNG
snapshot for the stale/WAS RUNNING and authoritative-banner presentation. Test partial
pages, known non-running agents, empty hosts, offline last-known Alive rows, family role
queries, removal after refresh and same-snapshot continuation. Prove zero-host laziness
and keep existing navigation budgets.

### live-acceptance

Read `tailnet.md` through `/sase_memory_read` and the original .5 notes. With both
machines on the released repaired cohort, run gc dry-run/apply and capture visible rows,
dismissal additions, package/binary versions, hello/summary payloads, and the actual
local/remote Agents presentation. Earlier .5 gc counts (Apollo 126 -> 17; Athena 784
-> 1233) are historical evidence, not this phase's acceptance result.

From Athena verify Apollo's presentation and authoritative running count agree with
Apollo's own current local presentation, including bounded recent completions. Capture
actual pane evidence. Dismiss a controlled fresh terminal agent on Apollo and prove it
disappears from Athena on refresh. Use a controlled test agent for the running-to-dead
transition; prove honest last-observed status, eventual reconciliation, and preservation
of unrelated live/Unknown/protected work.

Restart gateways through the supported managed workflow and prove removed rows do not
resurrect. Also exercise explicit older-history paging and a snapshot change during
continuation. Keep operations confined to the test workload and respect the deployment
workflow's real operational gates.

Run the complete combined-tree verification (SASE `just check-full` via monitor, full
core including PyO3, relevant visual acceptance). Record concrete before/after evidence
and exact results on this phase. If release or a fleet endpoint is still broken,
continue repair or record an unresolved blocker without closing this phase. This phase
supplies the missing stale-row acceptance evidence to the interrupted parent landing;
the separate unified-list live phase `sase-xe.16.11.7.13` retains its own owner and
record.

## Follow-up dispositions retained for resumed landing

All five original proposals were collected. The loader omission from .1 is the existing
small bug `sase-xv` and received landing corroboration. The live `sase-z0/link_events`
lint issue from .2 is owned by `sase-yy.8`/.8.5 and received the proposing bead's
evidence. No new task is warranted for either.

The include_terminal proposal, family-role fixture proposal and multiline-intent
proposal remain required work in this child. The first two are direct omissions or
integration defects of the fleet epic; the last blocks its explicit acceptance even
though the helper predates .1. Record these dispositions in the eventual parent close
note. The initial `epic-symbols` scan was clean; the resumed land agent must check it
again after the child lands and follow the original ancestor-readiness instructions.
