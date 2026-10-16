---
tier: tale
title: Stop per-row alias resolution from SIGKILLing artifact_link_backfill
goal:
  The hourly artifact_link_backfill chop finishes the sase project's reconcile well
  inside its 240s budget because event-snapshot alias resolution costs one sase-core
  call per snapshot instead of one per row.
size: small
proposed_by: bbugyi200.apollo.87
create_time: 2026-10-16 15:44:23
status: wip
---

# Plan: Stop per-row alias resolution from SIGKILLing `artifact_link_backfill`

## Symptom

The hourly `housekeeping/artifact_link_backfill` chop is killed again and again by axe
(`timed out after 300s`, exit -9). On 2026-10-16 alone there were five error digests,
and the job has digests going back to 2026-09-03. Every killed run since at least
2026-10-12 ends the same way:

```
[artifact_link_backfill] gh_sase-org__sase: reconcile
[artifact_link_backfill] gh_sase-org__sase: reconcile/aggregate
dropping projected edge with invalid ref: rule=stitch-bead ref='bead:[sase-1g4.2.1.5]'
<SIGKILL>
```

The runs that survive show how close it is. `gh_sase-org__sase` reports
`reconcile_repair=199–216s` and logs
`reconcile deadline expired; deferred dangling-ref repair`, and the whole job finishes
in ~277s. Whether a run survives depends on whether `gh_bobs-org__bob-cli` takes ~40s or
~90s before sase starts. The chop's 240s budget cannot help: the time is spent inside
one `store.reconcile_aggregate()` call, which only checks its deadline between stores.
Because the deadline always expires there, sase's dangling-ref repair has been deferred
on every run.

## Root cause (measured on live data)

I profiled `preview_reconciled_aggregate()` for `gh_sase-org__sase` read-only. The
hidden machine store was resolved with `fresh=False`, so nothing was fetched or
integrated, and only the non-writing preview was called. The preview took **236s**, and
**171s** of that was spent inside one Rust binding:

```
ncalls  tottime  cumtime  function
 23516    2.9    180.4    _artifact_link_event_store.py:104(covers_row)
 13069    0.5    176.1    _artifact_link_event_store.py:117(_resolved_row_identity)
 13069  171.1    171.1    sase_core_rs.artifact_link_event_resolve_aliases
```

`ArtifactLinkEventSnapshot.covers_row()` in `src/sase/sdd/_artifact_link_event_store.py`
first checks the row's identity against the cached edge identities. That cache was added
by commit `1785ea5d16`, the fix for
`plan:202609/artifact_link_backfill_reconcile_timeout.md`. On a miss, `covers_row()`
falls back to `_resolved_row_identity(row)`. Whenever the snapshot has aliases, that
fallback calls `require_rust_binding("artifact_link_event_resolve_aliases")` once **per
row**, passing the whole `list(self.aliases)`. The sase event store now holds **532
rename aliases** and 6,551 edges. Every call does three things in sase-core's
`resolve_artifact_link_event_aliases` (`crates/sase_core/src/artifact_link/events.rs`):

1. It converts all 532 alias dicts into Rust.
2. It rebuilds `AliasResolver` from scratch: it canonicalizes each alias and walks each
   one's terminal chain.
3. It serializes the full alias list back to Python in the result
   (`aliases: resolver.aliases()`).

That costs about 13ms per call. Both hot callers reach it once per row:

- `_iter_reconciliation_compatibility_rows` / `_iter_reconciliation_legacy_sidecar_rows`
  in `src/sase/sdd/_artifact_link_store_reconcile.py`, which use ~128s;
- the `predicate` built by `_authoritative_source_was_consulted_for_pass` in
  `src/sase/sdd/_artifact_link_store_core.py`, which runs per prior aggregate row inside
  `project_aggregate_rows` and uses ~58s.

The work is O(rows × aliases), and it gets worse with every recorded rename.

The rest of the preview is ordinary and bounded:

- the event snapshots, ~15s;
- `projected_rows()`, ~14s;
- row validation, ~8s, of which roughly half comes from the `_resolved_row_identity`
  path itself.

### Prototype evidence

I swapped the per-row call for a single binding call per snapshot, which returns the
whole terminal map. The results matched, and it was much faster:

| measurement                                                    | today        | one call per snapshot   |
| -------------------------------------------------------------- | ------------ | ----------------------- |
| resolve the terminal map for all 532 aliases                   | —            | 0.018s (1 call)         |
| 1,500-row random sample of rows that miss the direct check     | 21.2s        | 0.28s, **0 mismatches** |
| all 31,721 aggregate + sidecar rows that miss the direct check | ~450s (est.) | 4.0s                    |

Removing ~171s from a ~236s pass should bring sase's reconcile/aggregate down to roughly
60s. The whole chop should then finish in about 150–200s, inside its own 240s budget.
That also leaves time for the deferred dangling-ref repair to run for sase again.

## Why one call per snapshot preserves semantics

In sase-core, `resolve_artifact_link_event_aliases(aliases, refs)` works in three steps:

1. It builds one `AliasResolver` from the alias list. Chain following, cycle rejection
   and conflict rejection all live there.
2. It resolves each requested ref **independently**, by an exact lookup of its canonical
   form in the resolver's precomputed terminal map. A ref that is absent from the map
   resolves to itself (`resolve_canonical_ref`).
3. It returns `resolved_refs` keyed by the canonical input ref.

So if one call passes every alias `old_ref`, the result is the complete terminal map.
For any row, the resolved source and target are `terminal.get(ref, ref)`. Every piece of
alias semantics stays in Rust, and Python only caches the result of a Rust call for the
lifetime of one immutable snapshot. That is consistent with the Rust core boundary
(`rust_core_backend_boundary`, `rust-core-required`). Python's
`canonicalize_artifact_link_ref` is the same Rust `artifact_link_canonicalize` binding,
so lookup keys line up.

## Change

All edits are in `src/sase/sdd/_artifact_link_event_store.py`, in the
`ArtifactLinkEventSnapshot` frozen, slotted dataclass. Callers do not change.

1. Add lazily-filled private cache state next to `_edge_identity_cache`. Declare each
   field with `field(default=None, init=False, repr=False, compare=False)`, so equality,
   `repr` and `dataclasses.replace()` behave as they do today. The cache needs to hold
   one of three outcomes of a single binding call:
   - **resolved:** a `dict[str, str]` terminal map built from `resolved_refs`, coercing
     keys and values with `str(...)`;
   - **malformed:** `resolved_refs` is not a `Mapping`;
   - **failed:** the binding raised `TypeError`, `ValueError` or `RuntimeError`. Keep
     the message.

   Fill it with `object.__setattr__`, the way `_cached_edge_identities()` does. Pick
   whatever small representation reads cleanly, for example a private helper that
   returns the map or `None` for malformed, plus a separate cached error string.

2. Add a private `_cached_alias_terminals()` helper. On first use it calls
   `require_rust_binding("artifact_link_event_resolve_aliases")(list(self.aliases), [str(alias["old_ref"]) for alias in self.aliases])`
   **once**, then records which of the three outcomes happened.
3. Rewrite `_resolved_row_identity(row)` to use the cache. Keep its observable behavior
   identical:
   - With no aliases, return `artifact_link_row_identity(row)` as today. The binding is
     never called.
   - Keep validating the row and canonicalizing `source_ref` and `target_ref` before the
     lookup, exactly as today. A bad row still raises from those calls, and
     `covers_row()` still turns that into `False`.
   - **failed:** raise a `ValueError` carrying the cached message, without calling the
     binding again. `covers_row()` already catches
     `(TypeError, ValueError, RuntimeError)` and returns `False`, as it does today on a
     per-row failure.
   - **malformed:** return `artifact_link_row_identity(validated)`, as today.
   - **resolved:** set `source_ref` and `target_ref` to
     `terminal.get(canonical, canonical)`, then return
     `artifact_link_row_identity(validate_artifact_link_row(resolved_row))`, as today.
4. Leave `covers_row()`'s structure alone:
   - the `has_inputs` / `blocking_problem_messages` early return;
   - the direct check against the cached edge identities;
   - then the alias-resolved check.

   Update the `_resolved_row_identity` docstring, or add a one-line comment, saying that
   it resolves through one cached sase-core call per snapshot.

Do not change `_edge_identity`, `_event_edge_identities`,
`reduce_artifact_link_event_inputs`, or any caller in
`_artifact_link_store_reconcile.py`, `_artifact_link_store_core.py` or
`_artifact_link_store_rows.py`. They all benefit automatically.

## Tests

Add these to `tests/sdd/test_artifact_link_event_store.py`, next to
`test_covers_row_caches_edge_identities`. Use the real `sase_core_rs` binding, and use
the same `plan:202608/<x>.md` ref style the existing test uses.

1. **`test_covers_row_resolves_aliases_with_one_binding_call`**:
   - Build a snapshot with `durable_event_count=1`, a 2-hop alias chain (`a.md → b.md`,
     `b.md → c.md`) and one directed `implements` edge from `c.md` to `d.md`.
   - Monkeypatch `event_store_module.require_rust_binding` with a wrapper that counts
     lookups of `"artifact_link_event_resolve_aliases"` and delegates everything to the
     real function.
   - Assert the following:
     - rows `a→d` and `b→d` are covered, through 2-hop and 1-hop resolution;
     - row `c→d` is covered directly;
     - row `x→d` is not covered;
     - repeated calls give the same answers;
     - the alias binding was invoked **exactly once**;
     - a fresh snapshot built from the same inputs is still `==` to the used one.
2. **`test_covers_row_caches_alias_resolution_failure`**:
   - Patch the binding lookup so `"artifact_link_event_resolve_aliases"` returns a stub
     that raises `ValueError` and counts its calls.
   - Assert that aliased rows report `False` on repeated calls, that a direct-identity
     row still reports `True`, and that the stub ran **once**.

The existing `test_covers_row_caches_edge_identities` and the rest of the artifact-link
suites (`tests/sdd/test_artifact_link_reconcile.py`,
`tests/sdd/test_artifact_link_event_store.py`,
`tests/test_axe_chop_artifact_link_backfill_*.py`) must keep passing unchanged.

## Verification

1. Run the repo verification the `lint_and_test` memory prescribes (`just check`,
   through `sase tool run` because the recipe is guarded). Fix anything it reports.
2. **Live read-only measurement.** This is the proof the fix addresses the SIGKILL, so
   run it with this checkout's code, not the globally installed `sase`, which is an
   editable install of the primary checkout. Write a throwaway script outside the repo.
   It should:
   - monkeypatch `sase.sdd._artifact_link_machine_store._ensure_hidden_document_root` to
     force `fresh=False`, so no hidden clone is fetched or integrated;
   - call
     `resolve_machine_artifact_link_store("gh_sase-org__sase", <primary checkout from `sase
     project show sase`>)`;
   - time `store.preview_reconciled_aggregate()`.

   That preview is non-writing apart from the best-effort projection rule cache. Also
   count `artifact_link_event_resolve_aliases` invocations during the pass. Expect about
   60s instead of ~236s, and about one call per event snapshot instead of ~13,000.
   Report the before and after numbers in the final summary.

3. Optionally time the follow-on steps the chop will now reach for sase again:
   `dangling_and_orphaned_artifact_link_refs(store, kinds=REPAIRABLE_RENAME_KINDS)`
   (read-only), so the summary can state the expected total sase reconcile/repair time.
   Do **not** run the real chop or `reconcile_and_repair_artifact_links` by hand. Those
   write the machine aggregate and commit to hidden clones. The scheduled job will
   exercise the fix once it lands.
4. After the fix lands, the user should check the next scheduled runs:
   - `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/*.log`
     should show `gh_sase-org__sase: done` with `reconcile_repair` well under 100s;
   - there should be no `reconcile deadline expired` warning;
   - there should be no new `artifact_link_backfill` error digests.

   Mention this in the final summary.

## Non-goals

- **sase-core changes.** This plan adds no resolver-handle binding and does not drop the
  echoed `aliases` list from the binding's result. One call per snapshot costs ~18ms, so
  a core change, wheel and pin bump buys nothing here.
- **Budgets and timeouts.** The chop budgets (`_CHOP_WORK_BUDGET_SECONDS`, the sweep
  budget) and the 5m axe timeout stay as they are, and no deadline checks are added
  inside a single store's row iteration. A partial projection there could drop rows.
- **Unrelated warnings in the same logs.** These are already tracked elsewhere or
  harmless:
  - `artifact-link bead event publication failed: 'Issue not found: bob-cli-5z'` and
    `artifact-link bead projection is not committed` (see `sase-1c5`, `sase-yy.8.6`);
  - `dropping projected edge with invalid ref: rule=stitch-bead ref='bead:[sase-1g4.2.1.5]'`.

  If one of them turns out to be untracked and worth fixing, record it with
  `/sase_new_task` rather than widening this change.
