---
tier: tale
title: Stop one malformed projected link row from breaking every bead show
goal:
  sase bead show/read renders links for every bead even when the machine-local aggregate
  holds an unrelated malformed row, and projection never materializes rows whose refs
  fail validation.
size: small
proposed_by: bbugyi200.athena.0x6
create_time: 2026-10-06 06:08:11
status: wip
---

# One malformed projected link row breaks `sase bead show` for every bead

## Symptom

`sase bead show <any-id>` (and `sase bead read`) fails before printing anything:

```
Error: validation: bead id segment must contain only letters, digits, '-' and '_'
Hint: rerun with --no-links to show the rest of this bead without resolving artifact links.
```

This is not specific to the bead in the report (`sase-1gu.3`). Every bead fails. The ACE
TUI and pager use `enrich_with_artifact_link_neighborhood`, which degrades instead of
exiting, so there every bead silently loses its LINKS / REFERENCED BY section.

## Root cause (verified)

1. **A bad projected row.** Commit `8c8c47f72090` on master ends with a malformed
   trailer: `SASE_BEAD=[sase-1g4.2.1.5]`. It has square brackets but no `[n]` reference
   suffix. Every other commit uses either `SASE_BEAD=[id][n]` with a reference
   definition, or a bare `SASE_BEAD=id`. The Rust footer grammar does not treat a bare
   `[label]` as a linked value, so it parses as the plain string `[sase-1g4.2.1.5]`. The
   `stitch-bead` projection rule (`src/sase/artifact_links/projection/_stitch_rules.py`,
   `_parse_log_output` / `_tag_label`) then emits the row
   `stitch:sase@8c8c47f7… --implements--> bead:[sase-1g4.2.1.5]` without validating the
   ref. `project_link_rows` (`src/sase/artifact_links/projection/_entry.py`) puts it
   into the machine-local aggregate as an `origin: projected` row.

2. **A read path that checks every row.** `_aggregate_only_rows_touching` in
   `src/sase/sdd/_artifact_link_store_bead_rows.py` evaluates
   `self._is_aggregate_only(row)` first for **every** aggregate row (about 36k today).
   Only then does it check `row_touches(row, artifact_ref)` and
   `not is_projected_row(row)`. `_is_aggregate_only` → `sidecar_root_for` →
   `writes_sidecar_json` → `kind_of_ref` → `canonicalize_artifact_link_ref` calls the
   Rust `artifact_link_canonicalize` binding. That binding correctly rejects
   `bead:[sase-1g4.2.1.5]` with a `ValueError`, which `cli_query` turns into the error
   above. The row touches no bead being shown and is projected, so the cheap filters
   would have excluded it. Only the predicate order lets it crash the read.

   Verified by monkeypatching the reorder (`row_touches` → `not is_projected_row` →
   `_is_aggregate_only`): `sase bead read sase-1gu.3` then renders normally, including
   its REFERENCED BY section.

The Rust validator in sase-core (`validate_bead_id` in `artifact_ref/mod.rs`) is right
to reject the ref. No sase-core change is needed. Every fix below is local Python in
this repo, so the Rust-core boundary is not crossed.

No current writer was found that emits a bare `[label]` bead trailer, and this is the
only such commit in history. The trailer itself cannot be fixed (rewriting master
history is not an option), so the read and projection paths must tolerate it.

## Changes

### 1. Reorder the aggregate-only filter (the actual crash fix)

In `ArtifactLinkStore`'s bead-rows mixin
(`src/sase/sdd/_artifact_link_store_bead_rows.py`, `_aggregate_only_rows_touching`),
evaluate the predicates cheapest and most selective first:

```python
for row in self.load_aggregate().get("rows", [])
if row_touches(row, artifact_ref)
and not is_projected_row(row)
and self._is_aggregate_only(row)
```

`row_touches` is a pure string-set check, and `is_projected_row` is a dict lookup. This
also removes about 70k Rust canonicalize calls from every `bead show`, which is a free
perf win on a hot path. Behavior is unchanged for well-formed data, because the
predicates are pure and combined with `and`.

Keep the existing contract: a malformed row that genuinely **touches** the shown bead
and is store-backed (not projected) still raises. The `assemble_bead_link_neighborhood`
docstring promises that "a present but malformed … link index is raised". Only unrelated
or projected rows stop being able to poison the read.

### 2. Never materialize a projected row whose endpoints fail validation

In `project_link_rows` (`src/sase/artifact_links/projection/_entry.py`), drop any
`ProjectedEdge` whose `source_ref` or `target_ref` fails
`canonicalize_artifact_link_ref` (from `sase.sdd._artifact_link_store_support`). Catch
`ValueError` (and `TypeError`, to match how the binding can fail), skip the edge, and
log one `log.debug`/`log.warning` naming the rule id and the offending ref. Keep the
emitted row text otherwise byte-identical: do not canonicalize the refs being written,
so current row identities and dedup behavior do not change.

Do this at the generic entry point rather than only in `_stitch_rules.py`, for two
reasons:

- It covers every projection rule (`stitch-bead`, `stitch-agent`, `agent-bead`,
  `agent-wait-bead`, `chop-agent`) with one guard.
- The `stitch-bead` rule keeps an incremental per-HEAD row cache
  (`read_rule_cache`/`write_rule_cache`), and the bad row is already in that cache. A
  guard only in `_parse_log_output` would never see cached rows. A guard in
  `project_link_rows` filters them on every pass.

Because the aggregate rebuild drops every prior `origin: projected` row and appends only
freshly projected ones (`_artifact_link_store_support`'s survivor selection), the
existing bad row leaves the on-disk aggregate on the next rebuild. No manual data repair
or migration is needed.

Do **not** add grammar leniency that unwraps `[label]` into a bead id. It would recover
one edge from one historical commit, and footer grammar belongs to the shared Rust
commit-footer contract in sase-core. That is out of scope here.

## Tests

- `tests/sdd/test_artifact_link_store_projected.py` (or a sibling in `tests/sdd/`):
  build a store whose aggregate contains an unrelated projected row with a malformed
  endpoint (for example `target_ref: "bead:[sase-zz.1]"`, `origin: "projected"`) next to
  a well-formed aggregate-only row that touches `bead:sase-xx`. Assert that
  `load_artifact_rows("bead:sase-xx", bead_owned_rows=[...])` returns the well-formed
  row and does not raise. Add a second case where the malformed row is store-backed (not
  projected) but touches neither endpoint being read: it also must not raise.
- `tests/test_bead/test_bead_show_links.py`: one regression test showing that
  `artifact_link_neighborhood_detail` / `assemble_bead_link_neighborhood` succeeds when
  the aggregate holds an unrelated malformed projected row. Follow the existing fixtures
  in that file.
- `tests/artifact_links/test_projection_entry.py`: feed `project_link_rows` a rule
  output containing one edge with target `bead:[sase-xx]` and one valid edge.
  Monkeypatch a rule function if that is the simplest seam. Assert that only the valid
  row is materialized.
- `tests/artifact_links/test_stitch_rules_projection.py`: add a commit with trailer
  `SASE_BEAD=[sase-xx]` (bare brackets, no reference). Assert that
  `project_link_rows(...)` emits no `bead:[sase-xx]` row, and that a valid sibling
  commit's row is still emitted.

## Out of scope

- Any sase-core change. The Rust ref validator is correct.
- Rewriting the historical commit message.
- Footer-grammar leniency for bare `[label]` values.
- Changing the CLI's fail-loud behavior for malformed rows that genuinely touch the
  shown bead.

## Verification

- `just check` (per the lint-and-test memory). Run the targeted tests above first.
- Manual: `sase bead read sase-1gu.3 -r "<why>"` renders without `--no-links`, and the
  same holds for another bead (for example `sase-1g4.3`).
