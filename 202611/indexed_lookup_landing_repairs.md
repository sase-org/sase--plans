---
tier: tale
title: Finish named lookup correctness and land indexed agent lookups
goal:
  Preserve complete bounded identity resolution, prove lookup latency, and close
  sase-1t3.6.1 and parent phase sase-1t3.6.
size: medium
proposed_by: bbugyi200.athena.sase-1t3.6.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1t3.6.1
create_time: 2026-11-17 00:03:15
status: done
---

- **PARENT:**
  [202611/indexed_agent_lookups.md](https://github.com/sase-org/sase--plans/blob/main/202611/indexed_agent_lookups.md)
- **BEAD:**
  [sase-1t3.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.6.1.md)

# Finish named lookup correctness and land indexed agent lookups

Complete only the remaining work in epic `sase-1t3.6.1`, then finish its landing and
close its parent phase `sase-1t3.6`. This tale has no separate land agent. The coder
owns implementation, measurements, verification, and both closes in the same logical
turn. Do not wait for this work's own commit SHA, push, or CI before closing: host
finalization commits the completed work afterwards.

Read `sase bead read sase-1t3.6.1 -r "Need landing audit and follow-up outcomes"`, its
three closed children, and `sase bead read sase-1t3.6 -r "Need parent scope"`. Read the
approved `plan:202611/indexed_agent_lookups.md` through `sase artifact read`; it remains
the compatibility contract. Its parent is
`plan:202611/tui_storm_trust_heap_recovery.md`. No memory edits, global reinstall, live
TUI restart, archive cleanup, or change to trust behavior is authorized.

## Audit and bounded scope

The lander reviewed primary epic commits `dc09b5eba3`, `e1d81ee6c6`, `7c514e2f8e`, core
commits `a4b1c843`, `8593b148`, `b9f33b02`, current source, all child notes, and
intervening primary/base/core history. At primary `ca0732fc10`, core is pinned to
`b9f33b02`. The session/clan selector, current source revalidation, bounded artifact
traversal, and targeted hijack running-row observations are already implemented.
Preserve them; do not redo those phases. Read `file:explicit:342e54884b78a18ce53099ce`
with `sase artifact read` for the retained landing probes, source attribution, and
proposal dispositions.

The remaining defects are concentrated in `src/sase/agent/names/_lookup_named.py` and
its Rust boundaries:

1. `_find_named_agent_candidates` ignores `complete`/`truncated` and converts incomplete
   fallback results into a winner or `None`. A direct probe with
   `availability=missing, complete=False, truncated=True, hits=()` returned `None`.
2. `_rank_indexed_hits` assigns a hit to the first matching spelling before applying
   eligibility. A live child with exact name `alice.athena.foo`, workflow `foo`, and a
   parent timestamp disappears for candidates `("foo", "alice.athena.foo")`, although
   querying the exact second spelling alone finds it. The first spelling's non-root
   workflow exclusion must not discard the later spelling's eligible exact match.
3. `_find_named_dismissed_bundle_bounded` sorts whole directory listings before counting
   JSON files. With 500 non-JSON files and budget 1 it visited 1,000 entries. It does
   not expose incompleteness and does not cover arbitrary nested workflow-child bundles
   as the old recursive fixture oracle did.
4. `_bundle_winner_from_summaries` accepts deleted paths, and
   `_named_agent_from_bundle_file` fabricates a dismissed completion after open failure.
   A single stale summary produced `NamedAgent(is_done=True)` for a nonexistent bundle.
   It must continue to another valid candidate, or report a proven miss/unavailability
   as appropriate.
5. Ranking, bundle selection, and bundle recovery remain Python domain logic, contrary
   to the approved Rust boundary. The general archive reader sets `busy_timeout=30_000`,
   so it is also unsuitable as the bounded name lookup API.

This is one medium tale: a focused repair of named artifact/bundle selection and its
refusal boundary, using existing indexes and group machinery. No new service, database,
general loader redesign, or additional phased epic is needed.

## Implement the remaining contract

Open `sase-core` with `/sase_repo`, use only its printed path, and read its `AGENTS.md`.
Shared matching/ranking/recovery belongs in `sase_core`; Python supplies current owner
candidates, existing process-identity/liveness observations, and public dataclass
conversion. Keep `find_named_agent` arguments and successful return shape compatible.
Preserve the existing group exception API for callers.

1. Add a Rust named-selection helper and binding, alongside the existing artifact
   lookup/group selection contracts. Evaluate each hit for every applicable candidate
   spelling. Outer spelling order wins; within one spelling,
   live/dead-nonterminal/historical priority precedes exact/workflow priority and
   timestamp. Preserve `include_dead`, `only_done`, non-root workflow exclusion,
   dismissed prefixes, owner isolation, case behavior, and malformed marker handling.
   Python must retain the existing process identity check, not replace it with a
   PID-only observation. Remove Python ranking once the binding owns it.
2. Preserve explicit availability and completeness through named artifact and bundle
   selection. A truncated or errored scan cannot prove a miss, the latest winner, or
   historical completion. Rust `lookup.rs::recover` currently certifies exhausted group
   queries only; extend the names contract deliberately so a genuinely exhausted small
   source can be complete, while a large/truncated source remains incomplete. Reuse the
   existing artifact fallback rather than adding another scan. Return a typed
   unavailable error through the existing failure channel when a complete answer cannot
   be established.
3. Add a dedicated read-only named-bundle lookup in the existing `agent_archive` store,
   with typed availability/completeness and a short bounded SQLite lock timeout
   consistent with artifact identity reads. Do not change general archive consumers'
   query semantics just to serve this lookup. Reuse indexed `agent_name`/source
   `workflow_name` columns, all visibility states, and workflow children. Keep identity
   filtering in SQL before hydration. Validate matching paths/payloads and skip stale
   deleted or unusable candidates; hydrate only relevant candidates, not unrelated
   bundles. Preserve the original raw-suffix/filename-stem ordering and artifacts-dir
   versus bundle-parent fallback.
4. Put bundle recovery/selection in Rust. Bound directory entries, file probes, and
   decoded payloads before collection/sorting, including non-JSON files, malformed data,
   empty shards, and nested child bundles. Use the established 200-entry recovery budget
   and explicit exhaustion/truncation. Direct known pointers can be tried first. Never
   migrate/rebuild/reconcile from a lookup or silently conceal backend failure with
   `None`. Reuse maintenance/upserts so create, rename/promotion, dismissal, revive,
   deletion, and repair converge.
5. Integrate typed named unavailability with authoritative callers and changes landed
   since the epic started. In `_facts_resolve.py`, the `find_agent` call currently
   occurs before the group-unavailable `try`; encompass it and report
   `lookup_unavailable` with no stop target. Check CLI explain/persist-hijack, wait/fork
   completion, restart/drain, and new monitor recovery callers (`_rerun_monitor.py`,
   `_preview_recovery.py`, `recover_subject.py`, and recovery reconciliation) so an
   uncertain lookup cannot become `not_found`, completed, or permission to launch
   another successor. Use their existing refusal/retry channels. Keep StopDeps
   injection, row-local TUI resolution, pre-stop current observations, finalizer/gate
   latches, and monitor custody intact.

Review any further primary/base/core commits since the audit. In particular retain
`81dfe5710e`'s duck-typed overlay fix, `2e04694766`'s monitor lint/snapshot fixes,
`a2abeb1e`'s archive corpus schema compatibility, and the module splits incorporated by
`7c514e2f8e`. Do not switch unrelated most-recent-name, allocation, migration, or wipe
scans: they were explicitly out of scope.

## Verify and measure

Read current `lint_and_test.md` and, before any relevant edits, `ace.md`, `tui.md`, or
`symvision.md` through `/sase_memory_read`. Build the local extension with approved
workspace recipes; the lander's local extension was absent, so its pure Python probes do
not substitute for Rust-backed tests. No global install.

Add semantic regressions for all five audit findings. Cover exhausted-small versus
truncated recovery, every unavailable-index classification, a locked bundle index,
deleted newest bundle with an older valid match, child bundles, and the two-spelling
live-child collision. Lock tests should prove the configured bounded timeout using
generous ceilings or injected observations, not flaky micro-timing assertions. Compare
against the legacy scanner on controlled fixtures only. Assert healthy hits and misses
never call an unbounded scan or mutate/migrate an index. Increasing unrelated history
must not increase decoded matching records or bundle-file reads; count traversal/probes
too, including non-JSON directories. Test explicit maintenance restores the same request
to a complete result.

Run focused Rust lookup/archive/binding tests with `just` wrappers and focused Python
name/group/wait/fork/restart/hijack tests. Include the indexed lookup tests, relevant
dismissed bundle tests, row-local hijack parity, and the existing fakey hijack/drain
canaries where the changed refusal contract can affect them. Never bare cargo. Run
`just fix` and `sase tool run check` (the recorded `just check`) in each changed
repository, using `/sase_monitor` for long commands. Do not run `just check-full`. Keep
the core revision pin forward; host finalization commits core first and moves the
primary pin when both repositories change. Verify that already-landed bindings are
covered without inventing a future SHA.

Complete the original epic's missing performance acceptance: remeasure after the landed
storm fix; record environment/version, archive sizes, target names, and cold/warm state.
Use a synthetic archive at the inherited research scale (about 23,133 artifact rows and
65,875 bundles) with hit, miss, session, clan, and dismissed-only cases. For each,
report at least five warm samples as p50/p95/max. Separate internal resolution from
whole CLI startup; target warm resolution below one second. Use read-only real athena
`sase agent hijack NAME --explain` probes, never stop or hijack a live agent to
benchmark. A missing/stale live index must be reported honestly as unavailable, not
repaired during the read-only probe. Use an isolated prepared index for healthy-path
evidence if necessary and clearly distinguish it from installed live state. If lookup is
still slow, profile and fix its own cost; independent CLI startup costs get an evidenced
follow-up through `/sase_new_task`. Record results on `sase-1t3.6` and retain a durable
audit artifact.

The prior import-budget proposal is triaged by this lander on the active causal hijack
epic, not left for the coder. Current footprint is 12,335,426 normalized source bytes
versus pre-epic 12,006,094 (cap 12,162,484); the large new closure is hijack/monitor
recovery. This epic itself added 2,619 bytes to `_lookup_named` (other lookup modules
partly offset it). Moving domain logic to Rust should remove that own growth; keep new
adapters lazy and measure the own delta. Do not raise the import cap. A persistent
independently evidenced external failure may be recorded without reopening completed
scope. Full phase-proposal dispositions are on the epic landing note; copy those
outcomes into the final close note. No duplicate follow-up tasks are needed for failures
already fixed by later commits.

## Final step: finish this landing and its parent phase

After code, integration review, focused verification, and measurements are complete:

1. Reread `sase-1t3.6.1` and every descendant/note; verify the original plan and this
   tale are satisfied. Confirm later source drift did not break readiness. Record
   test/measurement evidence and every prior follow-up outcome, including why proposals
   were declined. Do not claim a failing check passed; distinguish independent baseline
   failures with retained witnesses.
2. Run `sase bead epic-symbols sase-1t3.6.1`. Resolve every entry under the Symvision
   epic-whitelist policy: wire it, privatize it, add an appropriate non-test pragma, or
   delete it. Re-key only to a still-open later bead that actually requires the
   exemption. No entries existed at the initial audit.
3. Close normally with
   `sase bead close sase-1t3.6.1 --note "<verified scope, integration, tests, measurements, and follow-up dispositions>"`.
   If stale symbols reject the close, finish cleanup and retry. If a named phase is
   incomplete, finish or reopen it. Never force merely to make the close pass, and never
   force a successful nested landing. Cancellation/supersession requires a genuine
   deliberate outcome with `--force --reason ... --resolution canceled|superseded`.
4. Run `just symvision` and verify the whitelist is clean. Open the plans sidecar with
   `/sase_repo`, read `plan:202611/indexed_agent_lookups.md` with `sase artifact read`,
   and set `status: done` in that plan's frontmatter at
   `202611/indexed_agent_lookups.md` within the printed plans checkout. Declare that
   sidecar modification for host commit too.
5. Read `sase bead read sase-1t3.6.1 -r "Need the parent link"` and confirm it still
   points to phase `sase-1t3.6`. Verify its indexed-lookup scope and recorded
   measurements, run `sase bead epic-symbols sase-1t3.6`, resolve or validly re-key any
   entries, then close only that phase normally with
   `sase bead close sase-1t3.6 --note "<child plan completion, parity, verification, and measured lookup latency>"`.
   Run `just symvision` if additional symbol cleanup changed its inputs. Leave
   containing epic `sase-1t3` and its plan status to its waiting land agent. If parent
   readiness is incomplete or ambiguous, record a concrete blocker on `sase-1t3.6` and
   report it rather than forcing closure.
6. Finish through `/sase_final` with commit decisions for every repository changed.
   Closing the verified epic and parent phase before this turn's host-owned commits is
   intentional; do not invent a post-push landing dependency or another lander.
