---
tier: tale
size: medium
title: Canonical ref to row resolution rules in sase-core
goal:
  Port the artifact-link ref to row-identity matching predicates into the sase-core Rust
  crate with wire types, sase_core_py bindings, and a thin Python facade, then rewrite
  _KnownTargetIndex / _known_target_for_ref to delegate every matching decision to that
  facade with no TUI behavior regression and the Class A identity gaps closed.
proposed_by: bbugyi200.apollo.sase-w3.1
bead: sase-w3.1
create_time: 2026-09-03 15:49:02
status: wip
---

- **PARENT:** [202609/link_follow_reliability.md](link_follow_reliability.md)
- **BEAD:**
  [sase-w3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w3/sase-w3.1.md)

# Canonical Ref -> Row Resolution Rules In sase-core

## Context

This is Phase 1 (`core-ref-resolution`) of epic `sase-w3` (Artifact link-follow
reliability), whose epic plan is `plan:202609/link_follow_reliability.md`.

The epic's "Class A" failure is identity mismatch: link chips carry a _synthesized_
`ArtifactEntryTarget` from `target_for_ref_kind()`, which guesses fields the panes do
not actually key rows by, so an exact tuple lookup misses even when the row is on
screen. The reconciliation logic that fixes this already exists as
`_known_target_for_ref` in `src/sase/ace/tui/relations/artifact_links.py`, but it lives
in the TUI, is duplicated nowhere else, and is not available to any other frontend.

Per the `rust_core_backend_boundary` core memory, canonical ref -> row matching is
shared backend behavior (a web frontend or a future `sase artifact link follow` CLI
needs the identical mapping), so the rules move to `sase-core` and the Python panes
become thin adapters.

**This phase changes no TUI behavior.** It delivers the shared rules, the bindings, the
facade, and a strictly-better (deterministic, project-disambiguated) implementation of
the existing in-pane relations resolution. Wiring the follow path to use it is Phase 2.

### Ground rules inherited from the epic plan

- Open the Rust core with the `/sase_repo` skill
  (`sase repo open sase-core -r "<reason>"`) and use only the path it prints. Do not
  guess or clone it another way.
- Dev installs build `sase_core_rs` from that checkout, so after editing Rust run
  `just install` in the sase repo before testing Python callers.
- Run sase-core's own gate (`just check` from the sase-core checkout root, which is
  `./scripts/check.sh all`) as well. Per sase-core's `AGENTS.md`, never verify with
  `cargo test -p sase_core` alone: that excludes the `sase_core_py` binding tests.
- release-plz owns sase-core versions: do **not** hand-edit
  `[workspace.package].version` or crate versions. Do **not** bump the `sase-core-rs`
  window in the sase repo's `pyproject.toml` either; `just install` prints an
  informational note when the checkout is ahead of the published window and the
  release-branch reconciler ratchets it.
- Both repositories become commit obligations in this turn's `/sase_final` declaration.
  The sase-core commit is the one sase's CI needs, so declare both.

### Required reading before finishing

- `sase memory read lint_and_test.md` (two-speed rule; `just install` first in an
  ephemeral workspace clone).
- `sase memory read tui_perf.md` before touching the relation projection, and
  `sase memory read symvision.md` before leaving any new public Python symbol without a
  non-test consumer.

## Verified Starting Point

Facts checked against the tree this plan was authored on:

- `src/sase/ace/tui/relations/artifact_links.py` holds `parse_link_ref` (line 212),
  `target_for_ref_kind` (line 223), `_target_for_ref` (line 262),
  `_known_target_for_ref` (line 276), and `_KnownTargetIndex` (line 39) with fields
  `targets`, `by_file_first_part`, `by_pane_last_part`, `by_stitch_repo_sha_prefix`,
  `agent_targets`.
- Pane row identities in production today:
  - beads `("beads", (project, kind, bead_id))` where kind is `task|flag|epic|phase`
    (`src/sase/ace/tui/widgets/artifacts/beads_list.py:45`)
  - stitches `("stitches", (repo, full_sha))`
    (`src/sase/ace/tui/widgets/artifacts/commits_timeline.py:36`)
  - files `("files", (logical_id,))` and `("files", (logical_id, version_id))`
    (`src/sase/ace/tui/widgets/artifacts/files_list.py:44`,
    `src/sase/ace/tui/relations/files.py:47`)
  - patches `("patches", (project_name, patch_name))`
    (`src/sase/ace/tui/widgets/artifacts/patch_entry.py:12`)
  - agents `("agents", (name,))`
    (`src/sase/ace/tui/widgets/artifacts/agents_list.py:44`)
  - documents `("ref:<provider_kind>", (project, doc_kind, identity))` where doc*kind is
    `proposal|active|archive` and a proposal's identity is a _notification id* while
    active/archive use the document path
    (`src/sase/ace/tui/widgets/artifacts/plans_list.py:55`,
    `src/sase/ace/tui/relations/documents.py:107`). One plan path can have proposal,
    active, and archive rows in the pane at the same time.
- Every `artifact_link_edges` caller passes `known_targets` as a `frozenset`
  (`RelationSource.known_targets()` in `src/sase/core/artifact_relations.py:137`), so
  today's index build order is hash order and its `setdefault` first-wins tie-breaks are
  **nondeterministic across processes**.
- Scale matters: the current project aggregate
  (`~/.sase/projects/gh_sase-org__sase/artifact-links.json`) has ~14,350 rows, so
  `artifact_link_edges` resolves ~28,700 refs per relation-index build. This runs on a
  pane worker (the module is deliberately I/O-free so it stays off the event loop), but
  it forbids a naive per-ref FFI call that marshals a full candidate list.
- sase-core already has: `artifact_link/` (`wire.rs`, `relation.rs`, `managed_table.rs`,
  `path.rs`, `inlet.rs`), `artifact_ref::canonical_artifact_ref_kind` (total; unknown
  labels pass through unchanged; the only registered aliases are `commit -> stitch` and
  `plans -> plan`), and precedent for opaque handle pyclasses plus plain dict-in /
  dict-out `#[pyfunction]` bindings in `crates/sase_core_py/src/lib.rs`.
- `tools/check_sase_core_rs_bindings` statically collects every name passed to
  `require_rust_binding` and asserts the installed extension exposes it, so a new
  binding name is gated automatically once `just install` rebuilds the extension.

## Design

### Why keys instead of "pass the candidates"

The epic plan sketches a predicate that takes `(kind, payload)` plus candidate row
identities and returns the matching identity. That shape is exactly right for Phase 2's
per-pane `entry_target_for_ref` (one call per link follow), and it is shipped here as
`resolve_artifact_row_identity`.

It is the wrong shape for the ~28,700-ref relation projection loop, which would marshal
the whole candidate list per ref. So the same rules are also exposed in a split form
that keeps the hot loop at one batched FFI call plus pure Python dict probes:

- `artifact_row_index_keys(identity)` -> the lookup keys one row answers to.
- `artifact_row_ref_lookup_keys(query)` -> the ordered keys a ref probes, first hit
  wins.

`resolve_artifact_row_identity` is implemented **in terms of** those two functions, so
the two entry points cannot drift: there is one rule set, two access patterns.

### Rust module

New file `crates/sase_core/src/artifact_link/row_resolution.rs`, re-exported from
`crates/sase_core/src/artifact_link/mod.rs`.

```rust
pub const ARTIFACT_ROW_RESOLUTION_WIRE_SCHEMA_VERSION: u64 = 1;

/// One selectable frontend row identity: owning pane plus ordered parts.
pub struct ArtifactRowIdentityWire { pub pane_id: String, pub parts: Vec<String> }

/// A link-graph ref split into its kind and payload.
pub struct ArtifactLinkRefPartsWire {
    pub schema_version: u64,
    pub kind: String,
    pub payload: String,
}

/// Everything a resolution needs besides the candidate rows.
pub struct ArtifactRowRefQueryWire {
    pub schema_version: u64,
    pub kind: String,
    pub payload: String,
    #[serde(default)] pub project_hint: Option<String>,
    /// Exact-first compatibility spellings for an `agent:` payload; empty
    /// means "match the payload verbatim".
    #[serde(default)] pub agent_name_candidates: Vec<String>,
}

pub fn parse_artifact_link_ref_parts(value: &str) -> Option<ArtifactLinkRefPartsWire>;
pub fn artifact_row_index_keys(identity: &ArtifactRowIdentityWire) -> Vec<Vec<String>>;
pub fn artifact_row_ref_lookup_keys(query: &ArtifactRowRefQueryWire) -> Vec<Vec<String>>;
pub fn resolve_artifact_row_identity(
    query: &ArtifactRowRefQueryWire,
    candidates: &[ArtifactRowIdentityWire],
) -> Option<ArtifactRowIdentityWire>;
```

`ArtifactRowIdentityWire` deliberately carries no `schema_version` (it is a leaf value
type marshalled once per row; `ManagedTableRowWire` is the same shape). The module
constant and the two request wires carry the version.

#### `parse_artifact_link_ref_parts`

Faithful port of `parse_link_ref`, plus the kind-alias registry:

1. Trim; strip one leading `@`; drop everything from the first `#`; trim again.
2. Partition on the first `:`. Missing separator, empty kind, or empty payload ->
   `None`.
3. Rewrite the kind through `canonical_artifact_ref_kind`, which is total: it maps
   `commit -> stitch` and `plans -> plan` and leaves every other label unchanged.

Using the registry rather than a hardcoded `commit -> stitch` is a strict improvement:
it is the same single source of truth every other ref consumer already uses, and it
makes a `plans:` spelling resolve against a `ref:plan` pane row instead of silently
routing to a `ref:plans` pane that does not exist. Payload is _not_ canonicalized (a
link payload is opaque to matching).

#### Key namespaces

Keys are `Vec<String>`; the first element is the namespace so key spaces can never
collide. Given `identity = (pane_id, parts)`, `artifact_row_index_keys` emits:

- always, when `parts` is non-empty: `["exact", pane_id, parts[0], ..., parts[n-1]]`
- `pane_id == "files"`: `["files.id", parts[0]]` (a file row's identity is its **first**
  part, because a versioned row is `(logical_id, version_id)`)
- `pane_id == "stitches"` and `parts.len() >= 2` and `parts[1]` non-empty:
  `["stitches.sha", parts[0], prefix]` for every **character** prefix of `parts[1]` of
  length `1..=len`. Character prefixes, not byte slices, so a non-hex payload cannot
  panic.
- `pane_id == "agents"`: `["agents.name", parts[last]]`
- `pane_id == "beads"`: `["beads.id", parts[last]]`, plus
  `["beads.project.id", parts[0], parts[last]]` when `parts.len() >= 2`
- `pane_id == "patches"`: `["patches.name", parts[last]]`, plus
  `["patches.project.name", parts[0], parts[last]]` when `parts.len() >= 2`
- `pane_id.starts_with("ref:")`: `[format!("{pane_id}.id"), parts[last]]`, plus
  `[format!("{pane_id}.project.id"), parts[0], parts[last]]` when `parts.len() >= 2`

The document rule uses `parts[last]`, which is what makes a `plan:` ref match a row
whatever its `proposal|active|archive` middle segment is, and what preserves the
existing (deliberate) semantics that a proposal row is addressed by its notification id.

Empty `parts` yields no keys.

#### Ref lookup key order

`artifact_row_ref_lookup_keys` returns an ordered probe list; the first key that hits
wins. An empty payload yields no keys.

| kind                  | ordered keys                                                                                                                                    |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `file`                | `["exact","files",payload]`, `["files.id",payload]`                                                                                             |
| `agent`               | `["exact","agents",payload]`, then `["agents.name", c]` for each `c` in `agent_name_candidates` (or just `payload` when the list is empty)      |
| `stitch`              | split payload once on `@`; when the separator is present and both repo and sha are non-empty: `["stitches.sha", repo, sha]`. Otherwise no keys. |
| `patch`               | `["patches.project.name", hint, payload]` when a project hint exists, then `["patches.name", payload]`                                          |
| `bead`                | `["beads.project.id", hint, payload]` when a project hint exists, then `["beads.id", payload]`                                                  |
| `bug`, `chat`, `chop` | no keys (virtual link-graph subject kinds with no owning pane, matching `target_for_ref_kind`)                                                  |
| anything else         | `["ref:<kind>.project.id", hint, payload]` when a project hint exists, then `["ref:<kind>.id", payload]`                                        |

#### Deliberate behavior deltas (all strict improvements)

1. **Deterministic tie-breaks.** Both the Rust resolver and the Python index order
   candidates by `(pane_id, parts)` ascending and keep the first row per key. Today's
   frozenset iteration makes ambiguous matches (same patch name in two projects, one
   plan path with both an active and an archive row, one sha prefix shared by two
   commits) resolve differently between processes.
2. **Project disambiguation.** `bead`, `patch`, and document lookups now probe a
   project-qualified key first when a hint is available, then fall back to the
   project-agnostic key. Today the project is ignored entirely.
3. **`stitch:` requires a non-empty sha.** Today `payload.partition("@")` on `"repo@"`
   produces an empty sha whose empty-string prefix key matches an arbitrary commit row
   in that repo. The empty prefix is no longer indexed and an empty sha yields no keys.
4. **Kind aliases.** `plans:` now canonicalizes to `plan:` (see above).

Everything else is a faithful port, including the file first-part fallback, the agent
exact-then-candidate order, and document tri-kind matching.

Out of scope on purpose: a bare `stitch:<sha>` ref with no repo still does not resolve.
The epic plan's rule is explicitly "match `full_sha.startswith(sha)` _within the repo_",
`target_for_ref_kind` already refuses to route a repo-less stitch ref to any pane, and
indexing a second repo-agnostic prefix key per stitch row would double the hottest index
in the build for no Phase 1 benefit.

### Bindings (`crates/sase_core_py/src/lib.rs`)

Follow the existing `py_to_json_value` / `json_value_to_py` dict-in / dict-out pattern.

- `artifact_row_resolution_wire_schema_version() -> int`
- `artifact_link_ref_parts(value: str) -> dict | None`
- `artifact_row_index_keys(identities: list[dict]) -> list[list[list[str]]]` — batched
  on purpose: one call per index build instead of one per row.
- `artifact_row_ref_lookup_keys(query: dict) -> list[list[str]]`
- `artifact_row_resolve(query: dict, candidates: list[dict]) -> dict | None`

Register each with `wrap_pyfunction!`, add each to the module-level doc comment list
that catalogs bindings, and add each name to the `artifact_link` binding-presence loop
in the in-crate `#[cfg(test)]` module (near the existing
`"artifact_link_row_schema_version"` list) with a small behavioral assertion.

### Python facade (`src/sase/core/artifact_row_resolution_facade.py`)

Modeled on `src/sase/core/agent_identity_facade.py`: `require_rust_binding` per
operation, dataclass/tuple conversion only, no matching logic.

```python
def parse_artifact_link_ref(value: str) -> tuple[str, str] | None
def artifact_row_index_keys(
    targets: Sequence[ArtifactEntryTarget],
) -> tuple[tuple[tuple[str, ...], ...], ...]
def artifact_row_ref_lookup_keys(
    kind: str,
    payload: str,
    *,
    project_hint: str | None = None,
    agent_name_candidates: Sequence[str] = (),
) -> tuple[tuple[str, ...], ...]
def resolve_artifact_row_target(
    kind: str,
    payload: str,
    candidates: Sequence[ArtifactEntryTarget],
    *,
    project_hint: str | None = None,
    agent_name_candidates: Sequence[str] = (),
) -> ArtifactEntryTarget | None
```

Every one of these gets a real non-test consumer in `artifact_links.py` in this same
phase (see below), so no `--epic-symbol` entry and no symvision pragma is needed. If any
symbol ends up without one, prefer deleting it over whitelisting.

### `artifact_links.py` rewrite

`_KnownTargetIndex` becomes a single key map plus the agent identity snapshot:

```python
@dataclass(frozen=True, slots=True)
class _KnownTargetIndex:
    targets: frozenset[ArtifactEntryTarget]
    by_key: Mapping[tuple[str, ...], ArtifactEntryTarget]
    agent_identity: AgentIdentitySnapshot | None
```

`build()`:

1. `ordered = sorted(set(known_targets), key=lambda t: (t.pane_id, t.parts))`.
2. One batched `artifact_row_index_keys(ordered)` call; `setdefault` each key so the
   first row in that deterministic order wins.
3. `agent_identity = AgentIdentitySnapshot.current()` **only when at least one row has
   `pane_id == "agents"`**, else `None`.

Capturing the snapshot once per build is itself a fix: today
`current_owner_agent_name_lookup_candidates(payload)` is called inside the per-ref loop
and re-reads the identity/config on every agent ref that misses an exact match.

`_known_target_for_ref(kind, payload, known_targets, *, project_hint=None)`:

- Given a `_KnownTargetIndex`: compute the probe keys with a module-level `@lru_cache`d
  helper keyed by `(kind, payload, project_hint, agent_identity_or_None)` — pure inputs,
  and `AgentIdentitySnapshot` is a frozen slots dataclass with tuple fields, so it is
  hashable and a config change produces a different key. The helper computes
  `current_owner_agent_name_lookup_candidates` for `agent` refs only, passes `None` as
  the identity for every other kind so the cache does not fragment, and calls
  `artifact_row_ref_lookup_keys`. Then probe `index.by_key` in order and return the
  first hit. This keeps the hot loop at pure dict lookups after the cache warms.
- Given a bare iterable (the compatibility path the existing tests use): delegate to
  `resolve_artifact_row_target`, computing agent candidates from
  `AgentIdentitySnapshot.current()` for `agent` refs. This is the honest no-index
  resolution and gives the direct facade entry point its production consumer.

`parse_link_ref` keeps its public name, signature, and `__all__` entry but delegates to
`parse_artifact_link_ref` behind a module-level `@lru_cache` (pure `str -> tuple`,
bounded size). It is called once per aggregate row in several loops, so memoizing keeps
the FFI out of the hot path; ref strings repeat heavily across rows and refreshes.

`_target_for_ref` passes its `project_hint` through to `_known_target_for_ref`.
`target_for_ref_kind` is unchanged in this phase (Phase 2 demotes it to a routing hint).

Nothing else in the module changes: `artifact_link_edges`, edge emission, dedup, the
snapshot cache, and the public exports all stay as they are.

## Steps

1. **Open the core checkout.** `sase repo open sase-core -r "..."`. Use only the printed
   path for every subsequent read and write.
2. **Write `crates/sase_core/src/artifact_link/row_resolution.rs`** with the wire types,
   the four functions, and the rules above. Re-export from `artifact_link/mod.rs`. Check
   whether `crates/sase_core/src/lib.rs` re-exports `artifact_link` symbols at crate
   root and follow whatever that file already does for this module.
3. **Rust unit tests** in the same file (`#[cfg(test)]`):
   - parse: `bead:sase-1`; `commit:sase@abc123` -> `stitch`; `plans:a.md` -> `plan`;
     `@plan:202608/a.md#why`; `not-a-ref` and `kind:` -> `None`.
   - one Class A regression per shape: epic/phase/flag bead ids resolving to their real
     row kind and project; a `plan:` payload matching an `active` row and a
     notification-id payload matching a `proposal` row; an abbreviated sha matching the
     full-sha row in its repo; the same patch name in two projects disambiguated by hint
     and not by hash order; an agent alias spelling matching a bare-name row; a `file:`
     logical id matching a `(logical_id, version_id)` row.
   - determinism: two candidates that both answer one key resolve to the
     `(pane_id, parts)`-smallest one, whatever order they are supplied in.
   - equivalence: `resolve_artifact_row_identity` agrees with probing
     `artifact_row_ref_lookup_keys` against a map built from `artifact_row_index_keys`.
   - guards: empty payload, empty parts, `stitch:repo@` with an empty sha, and
     `bug`/`chat`/`chop` all resolve to `None`.
4. **Add the five bindings** in `crates/sase_core_py/src/lib.rs`, register them, extend
   the module doc comment list, and extend the in-crate binding test.
5. **Verify sase-core**: `just check` from the sase-core checkout root. If it outruns
   the turn, run it through `/sase_monitor`.
6. **Back in the sase repo**: `just install` (this rebuilds `sase_core_rs` from the
   linked checkout — required before any Python test can see the new bindings).
7. **Add `src/sase/core/artifact_row_resolution_facade.py`** exactly as specified.
8. **Rewrite `artifact_links.py`** internals as specified.
9. **Update the two existing tests that assert the old index shape**:
   - `tests/ace/tui/test_artifacts_relation_sources.py::test_known_target_for_ref_uses_index_lookup_without_target_scan`
     constructs `_KnownTargetIndex` with the old field names. Rebuild it against
     `by_key` while keeping its intent (resolution never iterates `targets`).
   - `tests/ace/tui/test_artifacts_relation_sources.py::test_known_target_index_preserves_legacy_match_precedence`
     compares against a legacy frozenset scan, which is only stable because both sides
     shared one hash order. Replace it with assertions on the **new deterministic**
     precedence (project hint first, then `(pane_id, parts)`-smallest), and keep an
     unambiguous-case parity check against the legacy helper so the port is still
     pinned. Leave `test_artifact_link_edges_builds_the_known_target_index_once` and the
     `test_artifact_links_ref_kind.py` parse tests passing unchanged.
10. **New Python tests**:
    - `tests/core/test_artifact_row_resolution_facade.py`: facade-level coverage of each
      entry point, including a case per Class A shape asserting the facade resolves what
      bare `target_for_ref_kind` cannot (assert the synthesized target differs from the
      resolved one, so the test states the actual bug being fixed).
    - Extend `tests/ace/tui/test_artifact_links_ref_kind.py` with the `plans: -> plan`
      alias case.
    - A regression asserting `_known_target_for_ref` prefers the project-hinted row for
      an ambiguous bead id and patch name.
11. **Verify sase**: `just check` (inline; hand it to `/sase_monitor` if it drags). Fix
    every lint gate, including symvision on the new facade module.
12. **Close out**: `sase bead epic-symbols sase-w3.1` (expect no entries; if any appear,
    resolve them or re-key the Justfile line to `sase-w3`), then
    `sase bead close sase-w3.1 --note "<what was verified>"`. Do not close `sase-w3`.
    Declare both repositories in `/sase_final`.

## Risks And Mitigations

- **Hot-loop regression.** ~28,700 refs per relation-index build. Mitigated by the
  batched index-key call plus `@lru_cache` on both `parse_link_ref` and the probe-key
  helper; after warm-up the per-ref cost is a dict lookup, which is at or below today's
  cost. If a reviewer wants numbers, `pytest -s -m slow tests/perf/bench_tui_trace.py`
  is the existing harness.
- **Cache staleness.** Both caches key on pure inputs only; the agent identity snapshot
  is part of the probe-key cache key rather than being read inside the cached call, so a
  config change cannot serve a stale candidate set.
- **Cross-repo landing order.** sase's CI resolves `sase-core-rs` from a CI-built wheel,
  so the sase-core commit has to be part of the same declaration. Declare both repos; do
  not touch the `pyproject.toml` version window.
- **Ambiguity semantics change.** Deterministic tie-breaking can pick a _different_ row
  than a given past run did for genuinely ambiguous refs. That is the point (today's
  choice is arbitrary), and the project hint makes the common ambiguous cases correct
  rather than merely stable. Tests pin the new rule explicitly.

## Out Of Scope

Phase 2 and later own: `entry_target_for_ref` on `ArtifactEntryNavigator`, addressing
link follows by canonical ref, demoting `chip.neighbor_target`, dropping
`RelationRole.FAMILY`, the `_change_query_for_navigation` false-success fix, the
tri-state completion model, the reveal ladder, identity query fields, and hydration. Do
not start any of them here.
