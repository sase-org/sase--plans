---
tier: tale
size: medium
title: Finish and land the archive corpus epic (sase-1jm.2.1)
goal: 'The archive corpus stays compiled inside Rust behind an opaque handle and meets
  its latency budgets. A workflow child''s name looks up its owner. The cache module
  passes Symvision. Epic sase-1jm.2.1 is closed.

  '
proposed_by: bbugyi200.athena.sase-1jm.2.1.land
bead: sase-1jm.2.1
status: done
---

- **PARENT:**
  [202610/archive_corpus.md](https://github.com/sase-org/sase--plans/blob/main/202610/archive_corpus.md)
- **BEAD:**
  [sase-1jm.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jm/sase-1jm.2.1.md)

# Plan: Finish and land the archive corpus epic (sase-1jm.2.1)

This tale is the rest of the landing for epic `sase-1jm.2.1`, "Archive corpus,
in:archive scope, and CLI parity" (plan `202610/archive_corpus.md`). All six phase beads
are closed. The land agent's verification found three gaps that this epic caused. This
tale fixes them and then does the epic's closeout. Nothing resumes the landing after
this tale, so the final section is required.

The epic's DECISIONS are final. The view is `archive`, the token is `in:archive`, and
user-facing CLI text says Archive. Do not start the Archive view, the
`agents_archive_view` flag, restore, routing, or the shelf. Those belong to parent
phases `sase-1jm.3` through `sase-1jm.8`. Do not edit any memory note.

Follow-up triage is already done. The outcomes are recorded as a note on `sase-1jm.2.1`.
Do not re-file those follow-ups.

## Gaps found at landing

1. **Latency budgets were missed by about 6x and 90x.**
   `tests/perf/bench_agent_archive_corpus.py` uses a synthetic fixture of 11,000
   top-level rows plus 37,000 workflow children. It measured build p95 at about 1,700 ms
   against a 300 ms budget, and query plus first window p95 at about 2,700 ms against a
   30 ms budget. The lander profiled the cause:
   - The `compile_agent_archive_corpus` binding takes about 960 ms. Almost all of that
     time goes to serializing the whole compiled corpus into Python dicts: 11k rows, 96k
     name-map entries, and 11k containers.
   - `sase.core.agent_archive_facade.compile_agent_archive_corpus` then runs
     `json.loads(json.dumps(payload))`, which adds about 450 ms.
   - Each query binding receives the whole corpus dict back and deserializes it.
     `summarize` takes about 930 ms with an empty query and about 1.2 s with
     `outcome:failed`.
   - Every query rebuilds one `QueryRow` per row and a fresh `QueryCorpus`.
     `group_key(Day)` re-parses the RFC 3339 timestamp and the timezone string for each
     root.
2. **Workflow-child lookup is incomplete.** `compile.rs` maps every workflow child's
   names to the child's own archive key. In the real archive, about 11% of keyed child
   bundles (1,888 of 17,119 among 2026 bundles) have a `source_run_id` that matches no
   top-level row. For 1,573 of them, `parent_timestamp` equals a top-level row's
   `raw_suffix`. For those children, `lookup(<child name>)` returns no row, but the
   contract says a child resolves to its owner. The core fixture hides this because its
   child shares its parent's run id.
3. **Symvision is red on five epic symbols.** These public symbols in
   `src/sase/core/agent_archive_corpus_cache.py` have no non-test consumer:
   `AgentArchiveCorpusCacheKey`, `CachedAgentArchiveCorpus`, `archive_corpus_cache_key`,
   `invalidate_agent_archive_corpus_cache`, and `link_facet_signature`.

## 1. sase-core: a prepared corpus that stays in Rust

Open sase-core with `sase repo open sase-core` and read its `AGENTS.md` before you edit.
Its rules apply:

- Keep new files at or under 1,500 lines.
- `corpus/mod.rs` stays a facade that holds only `mod` and `pub use` lines.
- Import by module path. Do not add `lib.rs` root re-exports or `prelude.rs` `core_*`
  aliases.
- Do not use `macro_rules!`.
- Do not edit `CHANGELOG.md` or any version.
- Commit subjects are Conventional Commits.

Add a non-wire core type under `crates/sase_core/src/agent_archive/corpus/`, for example
`PreparedAgentArchiveCorpus` in a new `prepared.rs`. Model it on
`sase_core::prompt_prediction::CompiledPromptPredictionCorpus`. The type does the
query-independent work once, at compile time:

- one `QueryRow` per corpus row, from the existing `query_row_from_corpus_row` mapping
- each row's local day key in the compile timezone, so grouping never re-parses
  timestamps or the timezone
- an archive-key-to-row-index map, plus name-map entries that resolve to row indexes, so
  lookup does not scan every row
- a `QueryCorpus` cached per profile digest and built on the first query that uses that
  profile. The type must be `Send + Sync`, because one handle is shared across threads.

Port `count`, `summarize`, `rows`, and `lookup` onto the prepared corpus. Keep their
observable behavior exactly the same:

- newest activity first, then the archive key
- presentation roots and the container rules: worst outcome, matching-member count, and
  the label
- expanded containers consume the window
- empty groups are omitted
- unknown fields, unknown enum values, and an unknown `group_by` return the same typed
  errors

Keep a single implementation. The existing free functions over `&AgentArchiveCorpusWire`
can become thin wrappers that prepare the corpus and then query it, or the tests can
move to the prepared API. Measure compile cost before you trim it. For example, the
unused `containers` list can stay if it is cheap.

**Fix the workflow-child owner.** Read `s.raw_suffix` and `s.parent_timestamp` in the
compile `SELECT`. Both are v3 columns, and `raw_suffix` is indexed. Resolve a child's
owner after the full row pass, because rows stream in filename order:

1. If a top-level hidden row has the child's own archive key, that row is the owner.
2. Otherwise, if a top-level hidden row's `raw_suffix` equals the child's
   `parent_timestamp`, that row is the owner.
3. Otherwise, the child's names resolve to nothing, and lookup returns an empty result.

`AgentArchiveNameMatchWire.owner_key` carries the resolved owner. Add `raw_suffix` and
`parent_timestamp` to the test schema in `corpus/tests/mod.rs` and to `TestRow`. Add two
tests:

- a child with its own run id whose `parent_timestamp` points at a top-level row. Lookup
  by the child's short, canonical, and global names returns that owner.
- an orphan child. Lookup returns no row.

Keep every existing corpus test passing.

## 2. sase-core bindings: an opaque corpus handle

In `crates/sase_core_py/src/agent_custody/mod.rs`:

- Add `#[pyclass(name = "AgentArchiveCorpusHandle", module = "sase_core_rs", frozen)]`
  holding `Arc<PreparedAgentArchiveCorpus>`. Precedents are `PyPromptPredictionCorpus`
  and `PyQueryCorpusHandle`. Register it with `m.add_class` in `register_agent_custody`.
  Expose a `status()` method that returns the status wire dict through
  `serialize_to_py`, a `timezone` getter, and `__len__`, which returns the row count.
- `compile_agent_archive_corpus(request)` compiles with the GIL released and returns the
  handle. It no longer serializes the corpus.
- `summarize_`, `rows_`, `lookup_`, and `count_agent_archive_corpus(corpus, request)`
  take the handle plus the small request dict. Each one parses only the request, runs
  with the GIL released, and returns `serialize_to_py` of the small response. Keep all
  four as module-level functions, so `tools/check_sase_core_rs_bindings` can still see
  their names statically through `require_rust_binding`.
- No release has shipped these bindings yet: commit `bc406752` comes after `v0.37.2` and
  is in no tag. Changing their signatures in place is therefore not a breaking change
  for released sase, so do not mark the commit `!`.
- Rework the round-trip test in `agent_custody/tests.rs`. It should build a handle on a
  missing root and confirm that no index is created. It should call all four operations
  through the handle and the registered module attributes, and check `status()` and the
  `group_by` typed error. Add one non-empty round trip over a small v3 index.

Run `sase tool run check` in sase-core.

## 3. sase: facade, cache, and CLI

- In `src/sase/core/agent_archive_facade.py`, `AgentArchiveCorpus` wraps the handle
  instead of a `wire` dict. It carries the handle, `status` as a string,
  `status_details` (the mapping from `handle.status()`), and `timezone`. Remove the
  `json.loads(json.dumps(...))` round trip. Relative-date canonicalization stays in
  Python, and the query methods pass the handle. Leave the capability helpers alone. The
  facade still does not import Textual.
- In `src/sase/core/agent_archive_corpus_cache.py`, resolve the five Symvision entries
  without pragmas or `--epic-symbol` lines:
  - Rename `archive_corpus_cache_key`, `link_facet_signature`, and
    `AgentArchiveCorpusCacheKey` with a leading underscore, because they are used only
    in this file. Tests may import private names.
  - Give `CachedAgentArchiveCorpus` a real consumer by annotating
    `sase.agents.cli_search._get_archive_corpus` with it, which today returns `Any`.
  - Delete `invalidate_agent_archive_corpus_cache`. Its only callers are tests, and a
    private symbol that only tests use is still flagged as dead. Tests reset the
    module's `_CACHED` and `_GENERATION` through `monkeypatch` or a small fixture.
  - Update `__all__` to match.
- In `src/sase/agents/cli_search.py`, read the index notice details from
  `corpus.status_details` instead of `corpus.wire`.
- Update `tests/core/test_agent_archive_corpus_facade.py` and
  `tests/test_agent_search_cli.py` so nothing uses `.wire`. For example, assert
  `corpus.count("") == 0`, assert `len(corpus.handle) == 0`, and compare the key and
  status after a rebuild. The CLI parity oracle must keep comparing CLI `-j` identities
  and order against facade `rows()`.
- Extend `tests/_agent_archive_index_fixtures.py` so rows can set `raw_suffix` and
  `parent_timestamp`. Add a facade test showing that a workflow child with its own run
  id looks up its owner.
- Do not hand-edit `sase-core-revision.txt`. When this turn declares the sase-core
  checkout, the host writes the pin from that commit. See `docs/rust_backend.md`, "The
  CI source revision pin".

## 4. Verify

- Run `just install-venv` to rebuild `sase_core_rs` from the linked checkout.
- Run the bench: `.venv/bin/pytest -s -m slow tests/perf/bench_agent_archive_corpus.py`.
  It must pass both ceilings: build p95 under 300 ms, and query plus first window p95
  under 30 ms. Keep the printed numbers for the close note. The epic plan allows one
  exception: if build misses only because of the compile-time `restorable` stat, keep
  `restorable` correct, record the measurement, and file a follow-up through
  `/sase_new_task`. The query ceiling must still pass.
- Run the targeted tests: the core corpus tests, the `agent_custody` binding tests,
  `tests/core/test_agent_archive_corpus_facade.py`, and
  `tests/test_agent_search_cli.py`.
- Run `just _lint-symvision`. It must report nothing in `agent_archive_corpus_cache.py`.
  Two other entries are not this tale's work: the `update_failure_geometry.py` helpers
  are a `sase-1jc.9` follow-up, and `project_for_done` is task `sase-1jp`.
- Run `sase tool run check` in sase-core and in sase. These failures already happen on
  clean master and are tracked elsewhere:
  - `test_tui_app_import_stays_under_startup_budget`: `sase-1ic`
  - `test_bob_dry_run_canonical_report_has_no_digest_suffix`: `sase-1jj`
  - `test_tracked_marker_path_passing_sites_are_reviewed`: `sase-1by`
  - the `tests/ace/tui/test_node_finder_snapshot.py` collection error, plus
    `test_contract_manifest_matches_marker_selection`: `sase-1jc` note #6

  Treat any other failure as yours. Do not run `just check-full`.

## 5. Close out epic sase-1jm.2.1 (final step)

Do all of this in this same turn, after the code above is verified. Never use `--force`
to make a close succeed.

1. Run `sase bead epic-symbols sase-1jm.2.1` and `sase bead epic-symbols sase-1jm.2`.
   Both were empty at planning time. If an entry appears, resolve the symbol (wire it,
   privatize it, or delete it). Re-key it to `sase-1jm` only when a still-open later
   phase needs the exemption.
2. Close the epic: `sase bead close sase-1jm.2.1 --note "<verification>"`. The note
   states that the lander verified all six phases' work in code. It says that follow-up
   triage outcomes are in the epic's land-triage note. It names the Rust-held corpus
   handle, the measured bench p95 numbers, the workflow-child owner fix, the cache
   Symvision cleanup, and the `sase tool run check` results in both repos.
3. Run `just symvision`. Confirm there is no `agent_archive_corpus_cache.py` entry and
   no stale `--epic-symbol` entry for `sase-1jm.2.1`.
4. Open the plans sidecar with `sase repo open plans -r "<reason>"`. In the printed
   checkout, set `status: done` in the frontmatter of `202610/archive_corpus.md` (the
   PLAN path that `sase bead read sase-1jm.2.1` shows).
5. Handle the parent phase. `sase-1jm.2.1`'s `parent_bead` is phase `sase-1jm.2` of epic
   `sase-1jm`. Run `sase bead read sase-1jm.2 -r "<reason>"`.
   - If the host cascade already closed it, do nothing more.
   - If it is still open, verify that this child epic delivered that phase's work. That
     work is the core archive corpus, the `in:` scope token, and CLI parity, as
     described in the parent design `plan:202610/agents_archive_view.md`. Then close it
     with `sase bead close sase-1jm.2 --note "<what you verified>"`.
   - Do not close `sase-1jm`. Leave the two glossary follow-up notes on `sase-1jm.2` for
     the `sase-1jm` land agent.
