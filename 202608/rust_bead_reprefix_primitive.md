---
tier: tale
title: Build the Rust bead identity, alias, and store re-prefix primitive
goal:
  SASE can preview and atomically apply a deterministic, collision-safe re-prefix of one canonical bead event store,
  preserve old bead IDs as validated aliases, resolve those aliases on every bead operation, and expose the mapping and
  exact-token rewrite contracts to Python for the later cross-store migration phases.
proposed_by: bbugyi200.athena.sase-ei.1
bead: sase-ei.1
create_time: 2026-08-03 04:56:48
status: done
---

- **PROMPT:**
  [prompts/202608/rust_bead_reprefix_primitive.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/rust_bead_reprefix_primitive.md)
- **PARENT:**
  [202608/historical_bead_reprefix.md](https://github.com/sase-org/sase--plans/blob/main/202608/historical_bead_reprefix.md)
- **BEAD:** [sase-ei.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ei/sase-ei.1.md)

# Plan: Build the Rust bead identity, alias, and store re-prefix primitive

## Context and boundaries

The current forward-only prefix repair changes `config.json.issue_prefix` before minting a new top-level bead, but it
does not rename existing event streams or references. A historical store can therefore contain canonical IDs such as
`gh_bobs-org__bob-cli-1` through `gh_bobs-org__bob-cli-5` beside already-correct IDs such as `bob-cli-6`.

This tale implements phase `sase-ei.1`, the core foundation for the larger historical migration. The semantic owner is
the linked `sase-core` repository; the Python repository receives only facade/model adapters and parity tests. The later
epic phases remain responsible for plan and ChangeSpec codecs, bead compatibility pages, historical agent/chat storage,
the public `sase bead migrate-prefix` command, multi-repository locks/receipts/commits/publication, and end-to-end docs.
No CLI command or cross-repository transaction is added here.

## Current architecture

- `crates/sase_core/src/bead/config.rs` owns the JSON config wire but currently contains only `issue_prefix`,
  `next_counter`, and `owner`.
- `crates/sase_core/src/bead/events.rs` owns the append-only event schema, content-derived event IDs, and reducer.
  `crates/sase_core/src/bead/jsonl.rs` owns event-store and `issues.jsonl` I/O.
- `crates/sase_core/src/bead/read.rs`, `history.rs`, `mutation.rs`, `work.rs`, and the Rust fast-path `cli.rs` resolve
  or consume bead IDs through several paths. Exact aliases must be applied consistently before suffix shorthand.
- `crates/sase_core_py/src/lib.rs` serializes Rust wires into plain Python values. The Python side routes public bead
  operations through `src/sase/core/bead_read_facade.py`, `bead_mutation_facade.py`, and `BeadProject`.

## Implementation

### 1. Define one validated identity and alias contract in `sase-core`

Add a focused bead identity/re-prefix module and export its stable wire types through `bead/mod.rs` and
`sase_core::lib.rs`.

- Centralize the safe-prefix policy used by minting: non-empty, no whitespace, dot, slash, backslash, `--`, or trailing
  hyphen. Parse only source IDs shaped as `OLD-<base36>(.<decimal>)*`, preserving the complete counter/child suffix.
- Extend `BeadConfigWire` with a serde-defaulted, ordered `id_aliases: BTreeMap<String, String>`. Legacy configs decode
  to an empty map; saved configs render aliases deterministically.
- Validate aliases against the canonical store: an alias cannot be empty, equal its target, shadow a canonical ID, or
  target another alias; every target must be canonical. Reject chains, cycles, duplicate ownership, and destinations
  that collide with canonical IDs.
- Provide one exact-ID resolver which checks a full canonical ID, then an exact alias, and only then unique suffix
  shorthand. Load and validate aliases on store-backed reads. Make `MutableStore` use the same resolver so aliases work
  for direct Rust/PyO3 mutation calls as well as Python's pre-resolution.
- Route show, history/lost-notes selection, dependency and reference operations, open/close/update/remove, ready flags,
  claim/release/preclaim, epic work planning, and the Rust bead CLI fast path through the shared resolver. Outcomes and
  machine data report canonical IDs even when the caller supplied an alias.

### 2. Add the boundary-aware exact bead-ID token rewriter

Implement a pure, deterministic longest-match rewriter that accepts an ordered old-to-new map and returns rewritten text
plus per-ID and total replacement counts.

- Match only complete known IDs. Do not replace an ID embedded in a larger bead/identifier token, and do not rewrite an
  unmapped descendant merely because its parent prefix matches. Longest IDs win so `old-1.12` cannot be partially
  rewritten as `old-1`.
- Keep punctuation, Markdown/URL surroundings, and unrelated ProjectSpec-key text byte-identical.
- Expose the rewriter as a serializable Rust outcome and a PyO3 binding for the later plan, ChangeSpec, agent, and chat
  planners; add a thin typed Python facade rather than duplicating boundary logic.

### 3. Build a closed, deterministic prefix-migration preview

Introduce serde request/outcome wires for preview and apply. The request carries `from_prefix`, `to_prefix`, and an
optional exact old-to-new agent-name map for actor/assignee fields. The preview outcome carries a schema version,
preimage digest, changed/no-op state, bead-ID map, event-ID map, token counts, stream/event/issue counts, and the
resulting alias additions so later transaction code can render and revalidate the exact plan.

- Read and validate config, manifest, every event stream, and the reduced projection before planning. Collect old IDs
  from stream/root IDs, event issue IDs, issue snapshots and parents, dependency/removal lists, and `bead:` artifact
  references. Include historical removed issues represented only in events. Rewrite prose only for IDs already in this
  closed map; a free-form mention does not invent a new canonical identity.
- Preserve every suffix verbatim and leave already-correct streams byte-identical. Preserve `next_counter`, all
  timestamps, operation order/count, statuses, resolutions, notes, and nonidentity data.
- Reject unsafe or equal prefixes; malformed/unsupported stores; missing parents; active (`claimed` or `in_progress`)
  affected beads; destination canonical IDs outside the map; duplicate destinations; alias shadowing/chains/cycles; and
  inconsistent agent mappings. A mixed store is valid unless old and new forms collide.
- Digest all managed preimage bytes in deterministic path order. Preview performs no writes and returns the same outcome
  for the same bytes.

### 4. Rewrite every canonical event identity surface and prove isomorphism

Plan rewritten streams in memory, preserving stream and event order while changing only identity-bearing data.

- Rename stream/root IDs, record `issue_id`, creation snapshots, parents, dependency endpoints, forced/cascade removal
  IDs, exact bead references, and mapped tokens in stored title/description/notes/design/reference prose.
- Rewrite actors, assignees, creators/reporters, and preclaim agent names only through the explicit agent-name map; do
  not infer agent renames from raw prefix replacement.
- Recompute every content-derived event ID with the existing minting function and return the complete old-to-new event
  ID map. Detect any duplicate or inconsistent regenerated ID.
- Reduce both preimage and rewritten streams. Apply the same typed identity/text transformation to the preimage
  projection and require exact equality with the postimage reduction. Regenerate canonical `issues.jsonl` from the
  rewritten reduction only after that isomorphism check passes.
- Extend config aliases with every old canonical ID mapped directly to its new canonical ID and set `issue_prefix` to
  the destination without moving `next_counter` backward.

### 5. Apply the audited store plan without leaving a partial core store

The apply request includes the preview's expected preimage digest. Under the existing bead mutation lock, recompute the
entire plan, reject stale bytes, stage the new config, projection, manifest, and complete stream directory, validate the
staged store again, and then install the managed paths.

- Replace the stream directory as a unit so obsolete old-prefix filenames cannot survive. Use sibling backups and atomic
  renames for the managed paths; if staging or installation fails, restore the exact preimage and remove only this
  operation's temporary files.
- Keep `beads.db` and unrelated files untouched; the Python host can refresh its derived DB through the established
  `BeadProject` refresh path.
- Re-read the installed store, confirm the expected postimage digest and reduction, then return the same audit fields as
  preview. A rejected request and a failed or stale apply leave managed bytes unchanged. A repeated preview/apply after
  success is a deterministic no-op with aliases still resolving.
- Structure the installer behind an internal file-operation abstraction or failpoint-capable helper so tests can inject
  failure after staging and during each replacement step and assert rollback, without adding a production CLI flag.

### 6. Expose PyO3 and Python facade/model adapters

- Add documented PyO3 functions for prefix validation, exact-token rewriting, preview, and digest-guarded apply. Reuse
  the existing Rust-error conversion and serde-to-Python helpers.
- Add Python dataclasses/converters for the versioned request/outcome shape and focused facade calls. Apply must use the
  bead-store write sandbox guard; preview and token rewriting remain read-only.
- Update Python config defaults/types to include an empty `id_aliases` map while preserving unknown config keys. Ensure
  `BeadProject.resolve_id`, direct read facades, and all mutation facades observe aliases through Rust and return
  canonical `Issue` IDs.
- Update binding inventory/documentation and validation smoke coverage so a stale installed extension fails clearly
  instead of silently hiding the new contract.

## Tests

Add focused Rust unit/integration fixtures and Python binding/facade parity coverage for:

- legacy config decode, deterministic alias serialization, exact old-ID resolution before shorthand, canonical output,
  unknown IDs, ambiguous shorthand, alias chains/cycles, aliases shadowing canonical IDs, and missing targets;
- all public read/history/work and mutation families using an old ID, including dependency endpoints, close/open,
  update/remove, references, ready state, claim/release/preclaim, and Rust fast-path CLI dispatch;
- token boundaries, adjacent punctuation and URLs, embedded identifiers, longest-match parent/child IDs, unmapped
  descendants, Unicode surroundings, deterministic replacement counts, and exact no-op bytes;
- a mixed-prefix event store with parent/child lineage, cross-stream dependencies, artifact refs, prose, old and mapped
  agent actors, update/close/remove history, plus-one evidence, and an already-correct stream. Assert mapping closure,
  suffix preservation, stable counters/timestamps/order/count, deterministic event IDs, projection isomorphism, and old
  alias compatibility after apply;
- blockers and immutability: unsafe/equal prefixes, active affected beads, old/new counter collision, missing parent,
  malformed streams, alias conflicts, stale apply digest, and injected stage/install failures all leave config,
  `issues.jsonl`, manifest, and stream bytes identical;
- deterministic repeated previews, successful apply followed by a no-op rerun, and Python request/outcome round trips.

## Verification

1. In the linked Rust checkout, run `cargo fmt --all`, then `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
2. In the SASE checkout, run `just install` so the editable environment rebuilds `sase_core_rs` from the linked source.
3. Run targeted Python bead facade, binding-inventory, config, read/mutation, history, and work/CLI tests while
   iterating.
4. Finish with the mandatory `just check` in the SASE checkout and re-run the Rust workspace check if any linked-core
   code changed after its first full pass.

## Completion criteria

The phase is complete when the Rust and Python suites prove a dry-run-pure, digest-guarded, rollback-safe canonical
store rename; old IDs resolve directly to canonical IDs on every bead surface; later phases can consume stable ID,
event, alias, and token-rewrite outcomes; both repositories are clean except for the intentional implementation; and
bead `sase-ei.1` is closed with a note naming the Rust and Python verification that passed. The parent epic remains
open.
