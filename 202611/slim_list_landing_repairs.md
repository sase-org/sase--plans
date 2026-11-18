---
tier: tale
title: Finish compact list ownership and land sase-1t3.10.1
goal:
  Bound actual retained list/detail generations, preserve compact monitor identity, and
  close the verified epic and parent phase.
size: medium
proposed_by: bbugyi200.athena.sase-1t3.10.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1t3.10.1
create_time: 2026-11-18 12:58:56
status: wip
---

- **PARENT:**
  [202611/slim_agent_list_rows.md](https://github.com/sase-org/sase--plans/blob/main/202611/slim_agent_list_rows.md)
- **BEAD:**
  [sase-1t3.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.10.1.md)

# Finish compact list ownership and land sase-1t3.10.1

## Outcome and scope

Finish the remaining acceptance gaps in epic `sase-1t3.10.1`, then close that epic and
its delegated parent phase `sase-1t3.10` in this coding turn. This tale has no separate
land agent. Do not wait for this tale's commit SHA, push, or CI before closing: host
finalization commits the completed work after declaration.

Read `sase-1t3.10.1` and its latest notes, all four child phase notes, and the audited
artifacts `plan:202611/slim_agent_list_rows.md` and
`plan:202611/tui_storm_trust_heap_recovery.md`. The source baseline reviewed by the
lander is sase `fbc1e34fbc`; the compact Rust contract is `8cedd45d`, already contained
in sase's source pin `71c59dca`. Reuse newer fixes if they have landed.

The inherited decisions remain final: `hotfix_reinstall=yes`,
`trust_ack_compat=cutover`, and `tui_perf_memory=no`. Do not edit any memory note,
restart the user's TUI, or perform a global install. The outer epic's one-hour soak
belongs to `sase-1t3.14`; this tale needs focused ownership measurements and a
separate-instance smoke, not another integrated soak.

## Verified work and remaining evidence

The Rust compact projection reuses full selection/revalidation before returning compact
rows; its PyO3 bindings are registered. Python production indexed, source-fallback,
full-history, live-plan, and targeted-delta routes use compact constructors. The
artifact cache has one entry. Existing marker mutation/path audits cover the compact
running-marker release. These parts do not need reimplementation.

Lander verification: 38 focused compact-wire, row-adapter, workflow-loader, retention,
and marker-audit tests passed. Required `just check` ToolRun
`6b3ba7450a20b7f13a341f55e64a1f40` passed every lint/validation stage, including
Symvision, then expanded to the nonvisual suite because of the Justfile and rebuilt core
identity: 61,918 passed, 18 skipped, four failed. The four cancellation failures also
reproduced in a serial rerun and are a primary/core version mismatch, not slim-list
failures. The run paired primary `fbc1e34fbc` with linked core `98a22b79`; freshly
fetched upstream companion `4e5c1b9730` updates all four exact assertions and pins that
core revision. Use a primary tree containing that companion change when verifying this
tale; do not revert the newer cancellation contract or duplicate its test updates. The
failed run itself remains red; the lander did not rerun the whole suite.

Additional upstream drift through `43ef590bb6` was reviewed: chop recovery and its
companion contract, repository-integration module split, finalizer write visibility, TUI
code-swap wait notices, and command-substitution documentation. None introduces another
compact-list consumer or changes the repair boundaries below. Preserve the independent
Justfile exemption removal in `4e5c1b9730`.

The lander reproduced the following gaps:

Reproduction script: `file:explicit:390183b0f5d7c729aaa2f505`. Captured results:
`file:explicit:b20e7bc07e4622a2fb08abb5`.

1. `_projected_record.py::_note_projected_detail_retained` keys weak references by
   logical identity. Hydrating twenty distinct Agent objects with the same identity
   leaves twenty full payloads (1,900,000 raw-output bytes in the existing retention
   fixture), while the owner reports one entry.
2. `_agent_graph.py::copy_agent_graph` copies hydrated fields but omits the dynamically
   attached compact-value stash. `copy_agent_rows_relinked` shallow copies the stash and
   hydrated payload without registering the clone. After releasing the original, both
   copies still retain raw output and the owner reports zero. Production
   `_loading_graph.py` uses these copy paths.
3. `_display_detail_render.py::_hydrate_projected_record_detail` mutates and registers
   an Agent in a worker before checking current selection. Its post-await check compares
   logical identity only, so replacement rows and delayed/canceled workers are not
   protected by an object/version check.
4. A slow cache query for window 25 can complete after window 50 and overwrite the newer
   cache generation. Repeating window 50 then executes the query again: observed calls
   `[25, 50, 50]`, expected `[25, 50]`. Signature checks alone do not prevent
   out-of-order publication.
5. A real Rust scan of flat monitor metadata returns
   `agent_session_turn.monitor.supervisor_identity = "boot:9"`, while its compact result
   retains the top-level `process_identity = "boot:1"` and only `{exit_code: null}`
   under monitor. `_compact_monitor_supervisor_identity` compensates by retaining the
   entire metadata JSON in the 4096-entry JSON cache. This is epic-owned, not an
   optional later optimization.
6. `project_output_facts` clones every `meta_*` value without bounding its shape. A real
   compact binding call retained a 100,000-byte nested `_raw` under `meta_details` and
   another 100,000-byte diff under `meta_commits`. The phase-1 field inventory
   explicitly deferred `meta_commits`, but the projection does not. Compact done-row
   construction also calls `enrich_missing_commit_metadata`, which can backfill the full
   commit list.

The lander's integration cleanup removes all four exemptions keyed to this epic. It
deletes the unused Python single-directory and unbounded compact query wrappers,
preserves the Rust bindings and bounded production APIs, privatizes the in-file
detail-release helper, and deletes the test-only size accessor. Parity tests use the
production bounded query. Keep this cleanup; promote a helper again only when this
repair supplies a real production consumer.

## Implementation

### 1. Finish the compact Rust list-fact boundary

Open `sase-core` with `/sase_repo` and read its `AGENTS.md`. In
`crates/sase_core/src/agent_scan/list/project.rs`, preserve the full loader's
monitor-supervisor precedence as a compact scalar. Prefer resolving the existing compact
`meta.process_identity` from the full turn's supervisor identity for actual monitor
members, falling back to top-level identity. Preserve ordinary-agent behavior,
empty/missing distinctions, and canonical nested versus flat scanner input semantics.
Keep domain precedence in Rust.

Add projection and binding tests for conflicting top-level/supervisor stamps,
missing/empty supervisors, and non-monitor rows. Update Python compact enrichment to
consume the projected scalar; delete the filesystem workaround and its now-obsolete
path-audit exemption. Prove compact row construction does not read or retain
`agent_meta.json` to recover this fact.

Also make `project_output_facts` emit only inventoried, typed list/action facts instead
of cloning arbitrary `meta_*` JSON. Keep full commit collections and unknown nested
output payloads for selected detail. Inspect actual filter and row-action consumers
before choosing the retained keys; retain a minimal Rust summary or the established
indexed predicate when needed, never full-row hydration for filtering. Keep arbitrary
metadata rendering available after selection. Prevent compact done loaders from
reintroducing full commit lists through `enrich_missing_commit_metadata`; keep the full
loader's existing behavior. Test large nested `meta_*` values and commit arrays at the
binding and Agent boundary, alongside list/action parity and detail restoration.

Use local Rust build recipes, never bare cargo. Declare both changed repos through
`/sase_final`: the documented host finalizer commits the configured `revision_pin`
sibling first and writes its pushed SHA into `sase-core-revision.txt` before the primary
commit. Do not require your own new SHA before this tale's closeout, and do not change
release-owned package pins.

### 2. Bound actual hydrated objects across roster lifecycles

Fix `_projected_record.py`, `_agent_graph.py`, and the selected-detail scheduling glue
together. Track concrete hydrated generations, not just logical identity. Replacement or
eviction must release the displaced object's detail even when another owner still holds
it. Preserve compact list/filter/action facts and post-hydration optimistic changes.
Keep the owner weak so discarded rosters remain collectible.

Roster copies must not carry unmanaged selected detail. Prefer constructing compact
copies from the saved compact values, with coherent hydration flags, rather than copying
full payloads and trying to account for arbitrary clones. Handle both deep graph copies
and shallow relinked copies while preserving relationship aliases and avoiding mutation
of the source graph.

Keep I/O off the pump. Have hydration workers produce detached results; apply and
register them only after rechecking the current tab/view, selected identity, and the
exact current row generation/object. Stale or canceled completions must drop their
payloads, must not evict current detail, and must not schedule a refresh for a
replacement row. Keep duplicate-request coalescing, bounded current/previous selection
reuse, navigation gates, and immediate highlighting. Do not introduce another unbounded
worker pool or task registry.

### 3. Preserve the latest snapshot and integrate retention diagnostics

Give compact cache publication a request-generation/token check so a superseded
in-flight query cannot replace the latest generation. Preserve one entry, exact keys,
index/WAL mutation refusal, same-key-only lock-busy stale reads, and
full-history/revalidate bypass semantics. Do not share mutable Agents in the cache.

Resolve the obsolete full-shaped TUI snapshot chain as part of integration. The source
audit finds only re-exports and test callers for
`_agent_loader_scans.artifact_snapshot_for_tui_load`; production tiered loading uses its
compact twin. The full `load_agents_from_all_sources` path still has legitimate
consumers but scans directly. Confirm those boundaries and remove the unreachable old
TUI snapshot helpers and their `FULL_ARTIFACT_SNAPSHOT_CACHE` if no real consumer
exists. Preserve full-record domain APIs and legitimate full source/detail loaders. Do
not keep a duplicate full graph merely for old tests. Retarget useful cache/window/lock
regression tests to the compact path before deleting legacy-only test coverage.

Integrate retained owners with the newer `81e124eca8` bounded-cache census in
`src/sase/ace/tui/util/_gc_policy_census.py`. It currently sees only
`ARTIFACT_SNAPSHOT_CACHE`; the selected-detail owner and any justified remaining full
cache must be distinguishable so counts cannot conceal generations. Probe
already-imported owners only; keep census collection out of navigation paths. Reuse the
corrected defining-module watchlist from the epic.

### 4. Verify the repaired contracts

Extend meaningful ownership tests using the existing fixtures in
`tests/ace/tui/models/test_projected_detail_retention.py`. Exercise at least twenty
distinct identities, repeated same-identity replacements, both roster copy paths, and
payload collection after eviction. Assert actual retained payloads and compact row
facts, not just dictionary size. Add controlled out-of-order cache and hydration tests,
including tab switches, replacement objects with equal identities, cancellation, and
revisiting the current row.

Run compact wire/row/workflow parity, projected-detail, cache, marker-audit,
load-tiering oracle, and census tests appropriate to the changed paths. Preserve the
current import-budget cap: the post-epic `dd56fe2075` integration deferred unneeded
startup edges and reattributed the baseline; do not repeat the earlier unexplained
baseline increase. Capture comparable retained payload/count measurements and a
separate-instance smoke. Preserve list filtering, grouping, incremental row patching,
and existing full-record callers.

Run `just fix` and the required `sase tool run check` (`just check`) in every changed
code checkout. Route known-long builds/checks through `/sase_monitor` with explicit
continuation instructions that include this tale's closeout. Do not run
`just check-full`. A check-full-only failure following a passing `just check` is
test-infrastructure follow-up, not remaining epic work.

## Follow-up disposition already completed by the lander

- `sase-1t3.10.1.1` note #2 became medium bug `sase-1ti`: full Rust scans already
  omitted `held_by`, `hold_expires_at`, and `status_bucket` before this epic.
  Same-type/all-type/all-status searches, last-week sweep, and active-epic causal review
  found no duplicate. Leave this independent task to its owner; it also specifies
  carrying the fields through compact projection when fixed.
- `sase-1t3.10.1.3` note #1 was declined as already resolved by `cd0b9c5273`, which
  privatized `MemoryAuditRow` in its defining module.
- `sase-1t3.10.1.3` note #2 was declined as a separate task: supervisor identity loss
  and the full-JSON workaround are repaired in step 1 above. Closed `sase-1qr` fixed
  generic process identity earlier and is not a duplicate.
- Check-run cancellation failures corroborated existing tasks `sase-1tl` and `sase-1tm`
  with evidence `file:explicit:5a1946d926ef97e2b7ce4d60`. A causal note on active epic
  `sase-1tc.3.1.6` identifies its phase `.2`, Rust `98a22b79`, and the already-landed
  companion primary `4e5c1b9730`. No new task was created; this mismatch adds no
  slim-list repair work.

Retain these outcomes in the final close note; do not refile them.

## Final step: finish this landing in the same coding turn

1. Re-read `sase-1t3.10.1`, all descendants and notes, and its linked plan. Review drift
   since the lander's cleanup. Verify the repairs above, earlier compact-path parity,
   source-pin finalization configuration, and the inherited decisions. Record the
   passing checks and ownership measurements.
2. Run `sase bead epic-symbols sase-1t3.10.1`. Resolve every listed entry by wiring,
   privatizing, deleting, or a justified non-test pragma. Re-key only when an actually
   open later bead needs it. The lander's cleanup is committed before this tale starts,
   so no close step depends on this tale's own commit.
3. Close normally with
   `sase bead close sase-1t3.10.1 --note "<scope/notes/source/commit and drift verification, check results, measurements, and all follow-up outcomes above>"`.
   Run `just symvision` after the close. Open the plans repo via `/sase_repo` and set
   `status: done` in `202611/slim_agent_list_rows.md`, the epic's PLAN artifact. Include
   that edited repository in the final declaration.
4. Read the parent link again. It is phase `sase-1t3.10`; confirm this completed child
   implements its slim-list and bounded-retention scope. Run
   `sase bead epic-symbols sase-1t3.10` and resolve any leftovers, then close only that
   phase with
   `sase bead close sase-1t3.10 --note "<verified compact list path, bounded actual retention, completed child plan, and passing checks>"`.
   Confirm `just symvision`. Leave containing epic `sase-1t3` and its plan to their
   waiting land agent.
5. If a close rejects leftover symbols, finish cleanup and retry. If it rejects
   unfinished phases, finish or deliberately reopen the relevant work; never force a
   successful nested landing. `--force --reason ... --resolution canceled|superseded` is
   only for a deliberately canceled/superseded outcome, never to make a successful close
   pass. Finish through `/sase_final` with all changed repositories declared. No SHA,
   push, or CI wait follows this step.
