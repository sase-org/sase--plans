---
tier: tale
title: Land fixes for epic sase-1t3.13.1 Agents interaction latency
goal:
  Exact-delta share-back runs off the event loop with a consistent published graph,
  bead-warmup follow-ups never drop, and epic sase-1t3.13.1 plus its parent phase are
  closed out.
size: medium
decisions:
  parent_close:
    ask:
      Should the closeout close parent phase sase-1t3.13 once this tale's fixes land and
      the epic closes?
    choices:
      close: Close sase-1t3.13 now and leave targeted per-row copies to task sase-1tt
      hold:
        Keep sase-1t3.13 open, blocked on sase-1tt; soak phase sase-1t3.14 keeps waiting
    default: close
    why:
      The 2.3-2.9 s deep copy is gone; the leftover shallow copy predates the epic
      (sase-1tt)
    answer: close
proposed_by: bbugyi200.athena.sase-1t3.13.1.land
proposed_with: claude/opus
bead: sase-1t3.13.1
decided_by: auto
create_time: 2026-11-18 18:49:29
status: wip
---

- **PARENT:**
  [202611/interaction_latency.md](https://github.com/sase-org/sase--plans/blob/main/202611/interaction_latency.md)
- **BEAD:**
  [sase-1t3.13.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.13.1.md)

# Land fixes for epic sase-1t3.13.1 (Agents interaction latency)

## Context

Epic `sase-1t3.13.1` ("Finish measured Agents interaction latency recovery", plan
`plan:202611/interaction_latency.md`) has all five phases closed. Its commits are:

- `f81ff92e05`: viewport runtime patches plus surfaced-row bead warmup
  (`sase-1t3.13.1.3`)
- `10922458c1`: exact-delta copy-on-write fast path (`sase-1t3.13.1.4`)
- `194b1a1ef4`: bounded detail-header worker (`sase-1t3.13.1.2`)

The land agent reviewed the code and found three defects caused by the epic. This tale
fixes them and then finishes the epic's landing. Read `ace.md`, `tui.md`, `tui_perf.md`,
and `lint_and_test.md` through `/sase_memory_read` before you change code. Do **not**
edit any memory file. The outer epic's decision `tui_perf_memory = no` forbids it.

The land agent already finished follow-up triage and recorded it on `sase-1t3.13.1`:

- +1s on `sase-1gf`, `sase-lx`, and `sase-xo`
- new task `sase-1tt` for the remaining full-roster transient shallow copy
- two declined proposals

Do not re-triage them. Do not try to bound the transient pre-pipeline copy. That is
`sase-1tt`.

All paths below are relative to `src/sase/ace/tui/` unless stated otherwise.

## Defect 1: exact-delta share-back runs O(lists × roster × 317 fields) on the event loop

`actions/agents/_loading_disk_delta.py` calls `share_back_boundary_fast` (in
`actions/agents/_loading_graph.py`) on the UI thread after the last `await`.
`_share_back_exact_delta_lists` runs once per published list, eight to ten times in all.
Each run recomputes `agent_share_fingerprint` for every worked row. It also recomputes
it for every live row, because the memo is local to each list call. `Agent` has 317
dataclass fields, and `_fingerprint_value` builds tuples recursively.

The land agent's probe used normalized rosters (`make_agent` + `normalize_agent_graph`,
a clan every 10 rows) with 2- or 8-row status deltas. It measured about 250 ms at 165
rows and about 1.0 s at 660 rows **on the event loop for every exact artifact delta**.
cProfile of one 165-row share-back counted 1,980 fingerprint calls. This directly breaks
tui_perf rule 1 and the epic's goal.

Required fix:

1. Run the share-back off the event loop. Do it inside the **same** worker step that
   already runs `attach_finalize_plan_to_boundary`. For example, add a small worker
   function that attaches the finalize plan and then shares back, and call it through
   the existing `asyncio.to_thread`. This adds no new `await` before apply. Compute the
   trace statistics in the worker as well and return them with the boundary. Today they
   are `shared_live` and `retained_new`, built from full-roster `id()` sets on the UI
   thread. After the last `await`, the UI thread should only do O(1) bookkeeping and the
   existing apply checks. Keep the existing stale-token, `input_row_identities`, and
   `fleet_source_rows` protections. Reading live snapshot rows off-thread is already
   accepted by the copy step, so do not add locking.
2. Decide each distinct worked object **once per boundary**. Use one memo keyed by
   `id()` and shared across every published list (prep, fold, finalize query,
   `agents_query_result`, `scoped_agents`, capacity, and dismissed objects). Look up
   live candidates by identity once.
3. Replace the tuple-building fingerprint with a cheap comparison that exits early. The
   worked copy is a shallow copy, so most leaf values are the _same objects_ as the live
   row's. Compare non-relink fields `is`-first and fall back to `==` (iterating `vars()`
   is fine). Handle `AGENT_RELINK_FIELDS` with the closure rule in Defect 2, not by
   value. Hydrated projected-detail rows must still compare unequal to their compact
   worked copies. `prepare_projected_agent_copy` strips hydrated detail, and today such
   rows stay owned. Preserve that behavior.
4. If `agent_share_fingerprint` / `_fingerprint_value` lose every non-test caller,
   delete them or make them private, and update the tests. Symvision rejects public
   symbols that only tests use.

Target: the event-loop part of an exact-delta apply no longer grows with roster size.
Rerun the probe at 165 and 660 rows with 2- and 8-row deltas. Record the worker
share-back time and the event-loop time in the epic bead note. The worker share-back
should be a small fraction of the worker prep time, which is about 37 ms at 165 rows and
175 ms at 660 rows.

## Defect 2: share-back publishes an alias-inconsistent graph

Fingerprints compare `Agent` references by identity tuple only, and nothing relinks
after share-back. Two failures follow:

- **Orphan references.** Reproduced: with the clan fixture from
  `tests/_agents_tab_graph_isolation_helpers.clan_graph()`, change one member. The
  published clan container is a retained worked copy. Its `runtime_children` and
  `status_display_source` point at _worked copies_ of sibling members. Those siblings
  were shared back as live objects, so the referenced copies are in no published roster.
  The same happens with a 4-member clan (2 RUNNING, 2 DONE) for a status change or a
  scalar change on any member. The container's status overlays, runtime aggregation, and
  wait source then read orphan objects. They miss later in-place UI-thread updates to
  the published member rows.
- **Stale references (by construction).** A live row can be shared back while a row it
  references was replaced by a new copy. Its own fields are unchanged, and the
  reference's identity is the same. The shared live row then points at the superseded
  live object. That object is no longer in the roster, and the next delta keeps
  re-sharing it.

Required fix (closure-consistent share-back; share the decision memo from Defect 1):

1. A worked object W may resolve to its live counterpart L only if two things hold.
   First, W's non-relink fields equal L's. Second, for every `AGENT_RELINK_FIELDS`
   reference (single values and list elements, position by position), the decision for
   W's referenced object **is** exactly the object L references at that position. A
   referenced object that is not a worked copy resolves to itself.
2. Iterate to a fixed point. Un-sharing a row must un-share every row whose references
   now resolve differently. A clan or session component with one changed member is
   therefore published as owned copies. Unrelated components stay shared. Cost must stay
   O(rows + refs) per iteration and must never loop unbounded.
3. Every worked copy that stays owned gets its `AGENT_RELINK_FIELDS` relinked to the
   _decided_ published object: new list objects, never in-place edits to a live row's
   list. No published row may reference an orphan worked copy.
4. Audit the other boundary payloads that hold `Agent` objects or `id()`-keyed state. In
   particular, `finalize.tab_index` (`AgentTabIndex._key_by_id` is keyed by worked-copy
   ids; confirm the identity fallback resolves shared rows to the same tab, or remap
   it). Also check `prep.dismissed_agent_objects` and capacity rows, so that visible,
   capacity, and fleet aliases keep one object per identity.

## Defect 3: bead-warmup follow-up uses a zero-interval Textual timer

`actions/agents/_loading_bead_warmup.py` `_run_bead_confirmation_warmup` re-arms in its
`finally:` block with `self.set_timer(delay, ...)`, where
`delay = 1.0 if failed else (0.25 if retry_sweep else 0.0)`. In Textual 8.0.1,
`Timer._run` computes `int((now - start) / _interval + 1)` whenever `skip` is on and
`next_timer < now`. A zero interval therefore raises `ZeroDivisionError` in
`textual/timer.py:163`, and **the callback never runs**. The land agent reproduced this
with a minimal app: the `set_timer(0.0, …)` callback never fired and the traceback was
logged. The integration trace run logged the same error.

The result: a warmup request that arrived while a scan was running gets silently
dropped. The pre-epic code re-scheduled directly.

Required fix: never pass a zero (or negative) delay to `set_timer`/`set_interval` on
this path. Use a small positive floor, such as the existing 0.05 s activity-gate floor,
or schedule the thin synchronous `_schedule_bead_confirmation_warmup` with `call_later`.
Sweep the files this epic touched for any other `set_timer`/`set_interval` whose delay
can reach 0. Add a regression test showing that a pending request during a running scan
leads to a follow-up scan, and that no zero-delay timer is scheduled.

## Tests

Extend `tests/test_agents_tab_delta_copy_fast_path.py` (split into a sibling module if
it nears the toobig limit) and `tests/ace/tui/test_agents_bead_warmup.py`:

- A reusable invariant helper. After share-back, across every published list, each
  `AGENT_RELINK_FIELDS` reference whose identity is published `is` that published
  object. No reference points at a worked copy that is absent from publication. Run it
  for: a clan member scalar change, a status change, and a terminal transition (2- and
  4-member clans); an agent-session child change; child add/remove/reparent; deletion;
  overlapping capacity rows; fleet rows; and named-proc rows.
- A stale-reference case built by hand. A referrer's own fields are unchanged but a
  referenced row changed. Assert the referrer is not shared back pointing at the
  superseded object.
- Unrelated components keep live identity (existing assertions stay), and live rows stay
  unmutated until apply, including on exceptions.
- A thread/cost guard. A spy on the row-comparison function records each call's thread
  and call count. The comparison never runs on the event-loop thread. Calls stay at or
  under the number of distinct worked objects plus bounded closure re-checks; they must
  not be multiplied by the number of lists.
- The bead-warmup follow-up regression from Defect 3.

Run the focused suites (delta-copy, graph isolation/artifact-delta/merge, bead warmup,
runtime patching, header worker) and the integration phase's focused list. Then run
`just fmt` and `sase tool run check`. Do not run `just check-full`. Rendering is
unchanged, so no screenshot workflow is needed unless a snapshot test fails.

## Closeout (final step: finish epic sase-1t3.13.1's landing)

1. Append a note on `sase-1t3.13.1` covering:
   - the three fixes, with changed paths
   - before/after share-back measurements (worker and event loop, 165/660 rows, 2/8-row
     deltas)
   - verification results, including the `sase tool run check` id
2. Run `sase bead epic-symbols sase-1t3.13.1`. Resolve every listed entry per the
   Symvision epic-whitelist policy (wire it up, privatize, add a non-test pragma, or
   delete), or re-key it to a still-open bead that needs it. At planning time there were
   none.
3. Close the epic:
   `sase bead close sase-1t3.13.1 --note "<what you verified: phases .1-.5 reviewed; share-back moved off the event loop and made closure-consistent; zero-interval warmup timer fixed; measurements; check run id; follow-up triage outcomes are in the epic notes (sase-1gf/sase-lx/sase-xo +1, sase-1tt created, two declined)>"`.
   Never use `--force` merely to make the close succeed.
4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read sase-1t3.13.1`, i.e. `plan:202611/interaction_latency.md`.
6. Handle the parent. The epic's `parent_bead` is phase `sase-1t3.13` (interaction-pass
   of outer epic `sase-1t3`). Verify this child epic completed that phase's work:
   - detail-header lanes off the pump, with per-agent caches and stable content tokens
     (`.2`)
   - `,j` kept on highlight plus debounced detail (`.1`/`.5` evidence)
   - runtime aggregation bounded to surfaced rows (`.3`)
   - delta-load deep `copy_agent_graph` replaced on exact deltas (`.4` plus this tale)
   - bead warmup bounded to visible and WAITING rows, with lazy stranded sweeps (`.3`
     plus this tale)

   Run `sase bead epic-symbols sase-1t3.13` and resolve or re-key any entries.

> [!decision] parent_close = close Close only the parent phase:
> `sase bead close sase-1t3.13 --note "<verified behavior and measurements; remaining j/k p95 and selected-tribe misses are pre-existing (A/B equal at 5b08952c6d/af07b78ac2/HEAD) and tracked in sase-lx; targeted per-row transient copies tracked in sase-1tt>"`.
> Never close `sase-1t3` or any other ancestor. Its land agent and the soak phase
> `sase-1t3.14` own the rest.

> [!decision] parent_close = hold Do not close `sase-1t3.13`. Append a note on it naming
> `sase-1tt` as the remaining blocker, together with this tale's evidence. Then add the
> dependency `sase bead dep add sase-1t3.13 sase-1tt`.
