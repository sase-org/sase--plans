---
tier: tale
title: Projected edges from facts SASE already owns
goal: "A recomputed projection layer turns commit trailers, published agent metadata,
  and chop launch identity into ~12,500 typed edges in the machine-local read model,
  marked `origin: projected`, without adding one row to any sidecar index or bead event
  stream and without letting a projected row reach any path that treats an aggregate row
  as durable truth.

  "
size: medium
proposed_by: bbugyi200.athena.sase-ug.3
bead: sase-ug.3
create_time: 2026-08-26 15:39:03
status: wip
---

- **PARENT:** [202608/link_rail_every_tab.md](link_rail_every_tab.md)
- **BEAD:**
  [sase-ug.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ug/sase-ug.3.md)

# Plan: Projected edges from facts SASE already owns

This is phase `project` of epic `bead:sase-ug` (plan `202608/link_rail_every_tab.md`).
Phase `converge` (`bead:sase-ug.1`) has landed: `project_aggregate_rows()` in
`src/sase/sdd/_artifact_link_store_support.py:146` is now the single point every
aggregate-rebuilding path routes through, and the aggregate document carries a monotonic
`generation` with a CAS retry. This phase adds the projection layer that feeds that
point.

## What this phase owes

From the epic plan's `### Phase project` section:

1. A projection module with one pure rule per source fact, each emitting rows tagged
   `origin: projected` plus the rule id.
2. `produced-by` and `launched` registered in the closed relation registry in
   `../sase-core`, with inverses, plus the binding and Python-side declarations.
3. Projected rows materialized into the machine-local read model through `converge`'s
   single projection point — never into sidecar JSON or bead events.
4. Trailer extraction cached by commit sha and agent metadata by file mtime; the
   projection must not walk 13,412 commits or 10,011 agent directories on any
   interactive path.
5. Coverage tests per rule, an idempotence test, and a volume smoke test inside the
   chop's budget.

## Measured facts

Measured on this machine on 2026-08-26 against `sase` at `452ac54cf` (master, clean),
project key `gh_sase-org__sase`.

### The four sources, at primary-repo scope

| Rule               | Source fact                                                             |       Rows |
| ------------------ | ----------------------------------------------------------------------- | ---------: |
| `stitch-agent`     | `SASE_AGENT=` / legacy `AGENT=` footer in the primary repo              |      5,913 |
| `agent-bead`       | published `meta.json` `metadata.bead_id`/`epic_bead_id`/`phase_bead_id` |      5,137 |
| `stitch-bead`      | `SASE_BEAD=` footer in the primary repo                                 |      1,344 |
| `chop-agent`       | the `.chop.<base>.` segment of a published agent name                   |        110 |
| **Total, deduped** |                                                                         | **12,504** |

That total matches the epic plan's "roughly 12,600 more edges" estimate. Degree over the
projected subgraph alone: 14,680 nodes, median 1, p90 3, max 110 — the same shape the
rail's counted-chip design assumes.

### What it costs

| Measurement                                                 |  Before |          After |
| ----------------------------------------------------------- | ------: | -------------: |
| `artifact-links.json` on disk                               |  548 KB |        5.47 MB |
| Rows in the aggregate                                       |   1,278 |         13,552 |
| `json.loads` of the aggregate                               |   ~5 ms |         ~30 ms |
| `json.dumps(indent=2, sort_keys=True)` + atomic write       |   ~8 ms |         ~48 ms |
| `unique_rows()` over the full row set                       |   ~7 ms |         ~72 ms |
| `store.preview_aggregate()` end to end                      | ~500 ms | ~800 ms (est.) |
| Cold projection (full git walk + full agents-sidecar parse) |       — |      ~1,000 ms |
| Warm projection (nothing changed: HEAD read + stat walk)    |       — |         ~60 ms |

`rebuild_aggregate()` runs on every `sase artifact link add`, every recorded audited
read, and every derivation persist, so the warm number is the one that matters. The cold
number is paid once per new commit and once per agents-sidecar change.

### Two corrections to the epic plan's assumptions

Both were checked against the code and the live store; the plan's design intent survives
both, but its literal wording does not.

- **`metadata.chop_name` / `chop_lumberjack` do not exist in published agent metadata.**
  `V2_METADATA_FIELDS` (`src/sase/agents_sync/v2_validation.py:35`) is a closed set and
  contains neither. `agent_meta_from_chop_env()` writes `chop_lumberjack`/`chop_name`
  into the _local_ `agent_meta.json` under `~/.sase/projects/<key>/artifacts/`, which is
  never published. Verified: zero files in the agents sidecar mention `chop_lumberjack`.
  So the epic plan's stated fallback — the agent-name segment — is the only available
  source, and this plan uses it as the primary source.
- **That name segment carries the base _chop_ name, not the lumberjack.**
  `derive_chop_agent_name` (`sase-core`, `axe_chop/validation.rs:773`) composes
  `chop.<sanitized base chop>.<sanitized target key>.<run token>.<index+1>`. The
  lumberjack never appears. It is recovered from the AXE config instead (below).

## Design

### The invariant

> **A projected row enters the machine-local read model and nothing else.**

Every path that treats an aggregate row as _durable truth_ must exclude
`origin == "projected"`. This is not defensive coding; each of those paths is asking a
question a projected row cannot answer. The invariant is enforced by one shared
predicate and asserted by tests, and it is what lets phase `truthread` build a
store-versus-index drift report that means something.

### Rust core additions (`../sase-core`)

Two changes in `crates/sase_core/src/artifact_link/`, both required before any Python
here can run:

1. `wire.rs` — add `ArtifactLinkOriginWire::Projected` (`"projected"`), with
   `increments_uses()` false and `from_name("projected")` wired. Verified today that
   `artifact_link_validate_row` rejects `origin: "projected"` with
   `unknown variant \`projected\``, so this is a hard prerequisite.
2. `relation.rs` — two builtins appended to `builtin_artifact_relations()`:

   | Slug          | Inverse       | Directed | `written_by` | Source kinds | Target kinds |
   | ------------- | ------------- | -------- | ------------ | ------------ | ------------ |
   | `produced-by` | `produced`    | yes      | `projection` | `stitch`     | `agent`      |
   | `launched`    | `launched-by` | yes      | `projection` | `chop`       | `agent`      |

   Each needs a `direction_note`, a `positive_example` containing its own slug, and an
   inverted `negative_example`; `every_builtin_documents_direction_and_examples` already
   asserts all three. Extend `builtins_cover_v1_table`'s expected slug list.

`written_by: "projection"` is load-bearing and needs no new gate: `link add`
(`artifact_cli/link_ops.py:204`), the TUI link modal
(`ace/tui/actions/artifacts_link.py:167`), and the inlet
(`sdd/artifact_link_inlet.py:234`) all already admit only `written_by == "cli"`, so both
new slugs are unwritable by hand the moment they exist. `_agent_relation_values()` and
the completion catalog read the registry, so `relation:produced-by` filtering and
completion come for free.

No new ref kind. `chop:` stays virtual exactly as the epic plan requires — verified that
`artifact_link_canonicalize("chop:run_every/toobig_split")` passes through unchanged and
`artifact_link_validate_row` accepts a `chop:` endpoint, because neither gates on the
kind catalog.

### The four rules

All four are pure functions over already-loaded inputs, in a new
`src/sase/artifact_links/projection/` package that mirrors the existing
`src/sase/artifact_links/derive/` package. Every projected row is shaped:

```python
{
    "schema_version": 2,
    "source_ref": ...,
    "relation": ...,
    "target_ref": ...,
    "description": ...,          # names the fact, <= 240 chars
    "origin": "projected",
    "created_by": "projection:<rule-id>",
    "created_at": ...,           # the source fact's own timestamp when it has one
    "uses": 1,
}
```

`created_by` carries the rule id. That keeps the "which rule produced this" requirement
inside the existing closed row wire instead of adding a field, and the `$0` panel in
phase `panel` reads it directly.

| Rule           | Edge                                                   | `created_at`                  |
| -------------- | ------------------------------------------------------ | ----------------------------- |
| `stitch-bead`  | `stitch:<repo>@<sha40>` `implements` `bead:<id>`       | commit committer date         |
| `stitch-agent` | `stitch:<repo>@<sha40>` `produced-by` `agent:<name>`   | commit committer date         |
| `agent-bead`   | `agent:<global>` `implements` `bead:<id>`              | `meta.json` mtime, ISO-8601 Z |
| `chop-agent`   | `chop:<lumberjack>/<base>` `launched` `agent:<global>` | agent directory mtime         |

Details that are not obvious:

- **Repo scope for both stitch rules is the project's primary repository only.** This is
  a deliberate narrowing, and it is what the epic plan actually measured (its table says
  1,341 `stitch→bead` and 4,112 `stitch→agent`; the primary repo alone yields 1,344 and
  5,913 including the legacy `AGENT=` spelling). Widening to the sidecars adds 8,943
  rows from the beads sidecar and 4,868 from the plans sidecar — machine-generated
  bookkeeping commits nobody selects — and takes the aggregate past 10 MB. The epic
  plan's own risk row authorizes exactly this ("narrow `stitch-agent` first"). The
  primary repo is resolved from `collect_repo_inventory(project=self.project_key)`, the
  same call `_iter_reconciliation_stores` already uses; the ref uses `record.name`
  (`sase`), which is the spelling `builtin_entry_stitch.py:70` produces and the one
  `@stitch:<repo>@<sha>` resolves.
- **Both trailer spellings count.** `parse_commit_footer` already canonicalizes
  `SASE_AGENT=` and legacy `AGENT=` to one `AGENT` key, and `SASE_BEAD=` to `BEAD`. The
  agent ref goes through `commit_agent_association(tags.get("AGENT"), identity)`
  (`src/sase/association_agents.py:54`) so a bare shell name is globalized and projected
  to its sase agent the same way bead pages already do it — no new name handling.
- **`agent-bead` reads the published agents sidecar**,
  `<agents-root>/agents/<global>/meta.json`, taking `global_name` for the ref and
  emitting one row per non-empty `bead_id`, `epic_bead_id`, `phase_bead_id`. The
  `description` names which field produced it, so three rows to three different beads
  stay distinguishable. The agents root comes from
  `store.sdd_store.repo_root_for_kind("agents")`; no agents sidecar means the rule is a
  no-op, which is what keeps it inert in tests.
- **`chop-agent` recovers the lumberjack from AXE config.** Build
  `{sanitized-base-chop-name: (lumberjack, base)}` from `load_axe_config().lumberjacks`,
  including disabled chops, so a disabled chop keeps its edges. The sanitization must
  match Rust's `sanitized_component` exactly; reimplementing it in Python would violate
  the core boundary, so derive it through the existing binding:
  `derive_chop_agent_name(base, target_key=None, proposal_index=0, run_token=None)`
  returns `chop.<sanitized>.1`, and stripping the fixed prefix and suffix yields the
  sanitized base. An agent name containing `.chop.<segment>.` whose segment resolves in
  that map produces `chop:<lumberjack>/<base>`; one that does not resolve produces
  nothing. Verified against the live config: 27 configured chops map cleanly, and
  `chop.refresh_docs.sase.…` resolves to `chop:refresh_docs/refresh_docs`.
- **Why `chop:<lumberjack>/<base>` and not the expanded `refresh_docs[sase]` name.**
  `AxeEntrySelector` (`src/sase/axe/config_backend.py:26`) is AXE's exact identity for a
  configured entry and its `key_path` is `axe.lumberjacks.<lj>.chops.<base>`. That is
  the only chop identity that is stable, selectable, and editable, and it is what phase
  `subject` will resolve an AXE selection to. The `for_each` target key is preserved in
  the row's `description` rather than in the ref.

### Where the projection is injected

`ArtifactLinkStore` gains one method, `projected_rows()`, in a new
`ArtifactLinkStoreProjectedMixin` (`src/sase/sdd/_artifact_link_store_projected.py`).
`preview_aggregate()` and `preview_reconciled_aggregate()` each pass its result to
`project_aggregate_rows(...)` as a new keyword-only `projected_rows` argument, which:

- **drops every prior row whose `origin` is `projected`** — projected rows are
  recomputed, never carried forward, so a rule that stops matching stops emitting and
  the stale row disappears on the next rebuild; and
- **appends the fresh projected rows last**, so `unique_rows`'s first-wins dedup lets a
  store-backed row with the same identity beat a projected one. A hand-written
  `stitch:… implements bead:…` keeps its own `why`, `origin`, and `created_by`.

That keeps `converge`'s "one place decides which rows survive" property intact and adds
exactly one concept to it.

### Caching

One document, `~/.sase/projects/<key>/artifact-link-projection.json`, guarded by its own
`.lock` beside the existing aggregate lock, holding one entry per rule:
`{"signature": <...>, "rows": [...]}`.

- **Stitch rules.** Signature is the primary repo's HEAD sha, read from `.git/HEAD` (and
  the ref file or `packed-refs` it names) with no subprocess. Unchanged HEAD reuses the
  cached rows outright. A changed HEAD walks only `git log <cached>..HEAD` and merges;
  an unreachable cached sha (rebase, prune) falls back to a full walk.
- **`agent-bead`.** Per-agent entries keyed by `meta.json` `(st_mtime_ns, st_size)`;
  only entries whose stat changed are re-parsed. The stat walk over 10,036 directories
  measures 45 ms and the full parse it replaces measures 189 ms.
- **`chop-agent`.** Signature is the agents-sidecar directory listing digest plus the
  AXE config's own mtimes; the rule needs names only, and `os.listdir` of 10,037 entries
  measures 5 ms.

Every rule is independently best-effort: a rule that raises logs nothing to the user and
**returns its last cached rows rather than an empty tuple**, so a transient git failure
degrades to staleness rather than to a silent mass deletion of a whole relation class —
the precise failure mode `converge` was written to end. Cache writes are best-effort and
never fail a read.

### Enforcing the invariant

`is_projected_row(row)` and `store_backed_rows(rows)` go in
`_artifact_link_store_support.py` next to `project_aggregate_rows`. Applied at every
site that reads aggregate rows as durable truth:

| Site                                                                                                                                                     | Why it must not see projected rows                                                                                                                                                                                                                                                |
| -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `_artifact_link_store_rows.load_artifact_rows` aggregate fallback                                                                                        | This is the durable-truth per-ref path. It is also what `artifact_link_neighborhood.load_neighborhood_rows` reads, so leaving it unfiltered would push projected `implements` rows into the audited-read footer and the launch-prompt one-hop expansion, whose budget is 5 items. |
| `_artifact_link_store_bead_rows._aggregate_only_rows_touching`                                                                                           | `_is_aggregate_only` is true for every `agent:→bead:` and `stitch:→bead:` row, so without the filter every projected row would surface on its bead's neighborhood.                                                                                                                |
| `_artifact_link_store_reconcile.backfill_bead_endpoint_links`                                                                                            | It writes a bead endpoint **event** for every aggregate row with a bead target. Unfiltered, this phase would durably persist ~6,500 projected rows into bead event streams — a direct violation of "never into sidecar JSON or bead events".                                      |
| `artifact_cli/link_health.py` (`_dangling_refs`, `_missing_companions`, `_coverage_report`, `_stale_tables`, `dangling_and_orphaned_artifact_link_refs`) | It resolves every ref in the aggregate and feeds the rename-repair pass. Unfiltered it would try to resolve ~5,900 stitch shas through git and offer to "repair" refs that are recomputed, not stored.                                                                            |
| `artifact_cli/link_suggest.py`                                                                                                                           | Its contract is "existing _persisted_ links are exclusions"; projected rows would both suppress real suggestions and be mistaken for evidence.                                                                                                                                    |
| `sdd/artifact_link_backfill._stored_link_keys`                                                                                                           | Same reason: a derivation candidate must not be skipped because a projected row happens to match.                                                                                                                                                                                 |
| `sdd/_artifact_link_renames._rewrite_aggregate`                                                                                                          | Rewriting a row that the next rebuild recomputes is wasted work and can persist a rewrite the source fact does not support.                                                                                                                                                       |
| `doctor/checks_artifact_links.py`                                                                                                                        | Its staleness verdict is "aggregate versus sidecar `links/`". Compare store-backed rows only, and report the projected count separately in `data`, or the check turns red after every commit.                                                                                     |

`link list` is the one deliberate exception: with no reference it reads the aggregate
and **should** show projected rows, which is the point of materializing them. Its
`--origin` filter and Origin column already exist, so `--origin projected` isolates them
with no new option.

`remove_rows` grows one refusal: when every row matching the pair is projected, raise
with the rule id and the source fact instead of deleting from the aggregate a row the
next rebuild puts straight back.

### One performance fix this phase forces

`_row_identity` calls `artifact_relation_lookup` through the Rust binding once per row.
At 13,552 rows that is 13,552 boundary crossings per `unique_rows` call and it is called
twice per rebuild. Memoize the lookup by slug in a module-level dict; the registry is
compiled in and immutable within a process. Measured `unique_rows` cost at the new scale
is 72 ms without it.

## Implementation

Ordered; the Rust step genuinely gates everything after it.

1. **`../sase-core`, opened with `/sase_repo` first.** Add
   `ArtifactLinkOriginWire::Projected` and the two builtin relations with their
   inverses, direction notes, and both examples. Extend `builtins_cover_v1_table`, and
   add a Rust test that both new slugs round-trip through
   `relation_label_from_perspective` in both directions. Run the crate's tests. Then in
   this repo run `just rust-install` so `sase_core_rs` exposes them.
2. **`sase memory init`.** The relation registry renders into the committed snapshot
   `sase/artifact_relations.json` and into `sase/memory/sase_artifacts.md` through
   `root_rendering_artifact_relations.py`, and `sase validate` runs
   `sase init memory --check`, so `just check` fails until this is run. This is a
   mechanical regeneration required by step 1, not a hand edit of memory content;
   approving this plan authorizes it. It also refreshes `AGENTS.md`, `CLAUDE.md`, and
   the provider shims.
3. **`src/sase/artifact_links/projection/`** — `_model.py` (`ProjectedEdge`,
   `ProjectionInputs`), `_stitch_rules.py` (both trailer rules off one history walk),
   `_agent_bead.py`, `_chop_agent.py`, `_cache.py`, `_inputs.py`, `_entry.py`
   (`project_link_rows`), `__init__.py`. Keep every module well under the 700-line
   `toobig` warn threshold.
4. **`src/sase/sdd/_artifact_link_store_projected.py`** — the `projected_rows()` mixin,
   added to `ArtifactLinkStore`'s bases in `_artifact_link_store_impl.py`. It builds
   `ProjectionInputs` strictly from the store's own roots (`sdd_store`, `sidecar_roots`,
   `beads_dir`, `project_key`), so a test store over `tmp_path` with no repo inventory
   and no agents root projects nothing.
5. **`project_aggregate_rows`** — new keyword-only `projected_rows` argument with the
   drop-prior/append-last semantics above; both previews pass it.
6. **The invariant filters** — `is_projected_row` / `store_backed_rows` plus the eight
   sites in the table, the `remove_rows` refusal, and the `_row_identity` memoization.
7. **Tests.**
   - One coverage test per rule against a fixture git repo, a fixture agents sidecar,
     and a fixture AXE config: the exact expected refs, relations, `origin`,
     `created_by`, and `description`.
   - Idempotence: two projections over unchanged fixtures return byte-identical rows.
   - Incrementality: adding one commit re-walks only the new range and yields the union;
     an unreachable cached head falls back to a full walk.
   - Degradation: a rule whose input raises returns its cached rows, not `()`.
   - Convergence: extend
     `test_every_aggregate_writer_converges_regardless_of_publish_status` so every
     writer in every order agrees on the projected rows too.
   - Precedence: a stored row and a projected row with the same identity resolve to the
     stored row.
   - Non-persistence: after a rebuild carrying projected rows, no sidecar `links/*.json`
     and no bead event stream contains one — including after
     `backfill_bead_endpoint_links()`.
   - Invariant: `load_artifact_rows`, the bead neighborhood, `link_suggest`,
     `_stored_link_keys`, and `dangling_and_orphaned_artifact_link_refs` each see zero
     projected rows while `load_aggregate()` sees them.
   - Isolation: a store built over `tmp_path` with no repo inventory and no agents root
     projects zero rows and reads no AXE config.
   - Volume smoke: 12,500 synthetic projected rows through a full rebuild inside the
     `artifact_link_backfill` chop's 240-second budget, asserting a wall-clock bound
     with headroom rather than a tight number.
8. **Re-measure and record.** Row counts per rule, aggregate size, warm and cold
   projection time, and `preview_aggregate()` end to end, in the closing bead note. The
   epic plan asks for the degree distribution to be re-measured at the end of this phase
   to confirm the rail's grouping keys stay stable; report it.

## Verification

- `just check` inline, then `just check-full` through `/sase_monitor` before returning.
- The sase-core crate's own `cargo test` for the registry and origin changes.
- `sase artifact doctor` on the real project completes without attempting to resolve a
  projected ref, and its aggregate check does not report staleness merely because HEAD
  moved.
- `sase artifact link list --origin projected -l 5` shows projected rows with their rule
  in `created_by`; `sase artifact link list --origin manual` is unchanged from today.
- `sase artifact read bead:<id>` and a launch-prompt one-hop expansion show no projected
  neighbor.
- `sase agent search 'relation:produced-by'` and `'relation:launched'` resolve, since
  the agent query enum reads the registry.

## Risks and open decisions

| Risk                                                                          | Handling                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A 5.5 MB aggregate rewritten on every `link add` costs ~300 ms more per write | Measured, not estimated; warm projection is ~60 ms and the rest is JSON. If the owner judges it too expensive, the same rules can materialize into a sibling `artifact-links-projected.json` that only read-model readers union in — which would also make most of the invariant filters unnecessary. That is a real alternative and this plan does **not** take it, because the epic plan asks for one file and one projection point and `truthread`'s drift report is specified against that shape. Flagged here so approval can redirect it. |
| Projected rows dominate the default `sase artifact link list` output          | Rows are labelled `projected` in the existing Origin column and `--origin` already filters. No new option in this phase; `truthread` adds `--source store\|index`.                                                                                                                                                                                                                                                                                                                                                                              |
| The invariant is enforced by discipline at eight call sites                   | One shared predicate, one test per site, and the two most dangerous sites (bead-event backfill, rename repair) get explicit non-persistence tests.                                                                                                                                                                                                                                                                                                                                                                                              |
| `chop-agent` depends on live AXE config to resolve the lumberjack             | Disabled chops are included, so disabling keeps edges. A chop deleted from config stops projecting — correct for a recomputed layer, and it is 110 rows.                                                                                                                                                                                                                                                                                                                                                                                        |
| Sidecar-repo stitches are excluded, so a beads-sidecar commit shows no bead   | Deliberate and evidence-backed; widening is a one-line scope change if the rail ever surfaces sidecar commits.                                                                                                                                                                                                                                                                                                                                                                                                                                  |

## Out of scope

Everything the epic assigns elsewhere: the `--source store|index` read path and the
row-level drift report (`truthread`), the `LinkIndex` and selected-entity ref
(`subject`), the rail widget, the `$` grammar, the trail, the `$0` panel, and the
feature flag those phases introduce and `land` removes. No new CLI subcommand or option
is added here.
