---
tier: tale
title: Provenance-aware artifact links in sase bead show
goal: "Make sase bead show present a bead's complete typed artifact-link neighborhood
  with correct relationship perspective and durable provenance, while providing a
  reliable --no-links escape hatch and preserving the command's single-pass bead-store
  read.

  "
size: medium
proposed_by: bbugyi200.athena.0af
create_time: 2026-08-22 11:18:36
status: wip
---

# Plan: Provenance-aware artifact links in `sase bead show`

## Outcome and product contract

`sase bead show <id>` should make typed artifact relationships immediately useful
without confusing them with scheduling dependencies. By default, full output will show
every link that touches the bead, including links stored on another bead, links owned by
a document sidecar, and aggregate-only links such as an agent citation of a bead. It
will use the relation registry to describe every edge from the displayed bead's
perspective: outgoing directed edges use their authored relation, incoming directed
edges use the registered inverse, and undirected `related` edges remain symmetric.
`DEPENDS ON` and `BLOCKS` remain separate sections backed only by `sase bead dep`.

The human-readable layout will keep relationship context together, after the existing
parent/children/dependency blocks and before long-form description and notes:

```text
LINKS (2)
  → implements · plan:202608/link-aware-bead-show.md
    Lands the approved CLI design.
    manual · added 2026-08-22T14:10:00Z by alice.athena.worker
  ↔ related · bead:sase-a1
    Shares the same rendering contract.
    migrated · added 2026-08-20T09:00:00Z by alice

REFERENCED BY (1)
  ← cited-by · agent:alice.athena.reviewer
    Prompt citation of bead:sase-b2.
    prompt citation · 3 uses · added 2026-08-22T15:00:00Z by alice.athena.reviewer
```

The arrows, inverse labels, canonical counterpart references, reason text, origin,
actor, timestamp, and use count are information, not decoration. Rich mode will add
semantic color through the existing additive `DetailPalette`; stripping SGR escapes must
still reproduce plain output byte-for-byte. Link reasons will use the configured prose
wrapping budget, while identity and provenance rows remain atomic. Rows will be
deterministically grouped and sorted by group, displayed relation, counterpart, and
stored timestamp so output is stable. `manual`, `migrated`, and any future nonautomatic
origins belong under `LINKS`; `prompt_ref` and `read` belong under `REFERENCED BY` with
friendly labels and their accumulated use counts. Unknown legacy values must remain
visible as raw labels instead of crashing or disappearing.

Add `--no-links` to `sase bead show`. The option must bypass all link-neighborhood
resolution and omit both human link sections. Compact output remains the established
single row and never performs link resolution, with or without the option. JSON output
will gain an additive top-level `artifact_links` array containing both stored endpoints,
the stored and perspective-aware relations, direction, counterpart ref, reason, origin,
actor, timestamp, and uses. The existing `issue.links` outbound storage projection
remains unchanged by default for compatibility. With `--no-links`, omit the top-level
array and omit `issue.links` so the option genuinely emits no artifact-link data.

Missing optional sidecars mean there are no external rows and are not an error. A
present but malformed or unsupported link index must never be silently treated as an
empty neighborhood: fail with the underlying diagnostic and a concise hint to rerun with
`--no-links` when the caller needs the rest of the bead detail. This preserves the
artifact-link contract's fail-loud behavior while leaving an explicit recovery path.

## Preserve exact bead-event provenance in `sase-core`

Extend the linked `sase-core` repository's bead detail wire contract so one detail read
can return the active bead-owned artifact-link rows that touch the selected bead in
either direction. Derive those rows from the same validated, deterministically merged
event snapshot used to build the issue graph. For each currently active link identity,
carry the winning `LinkAdded` event's actor and timestamp together with its source,
target, relation, description, and origin; a rewrite must replace provenance, a removal
must remove the row, and a later re-add must establish new provenance. Do not derive a
link timestamp from the containing bead's general `updated_at`, because unrelated bead
mutations would misattribute the edge. Legacy snapshot-only stores should retain their
current import-compatible fallback without inventing stronger provenance than the store
contains.

Expose the enriched detail snapshot through `sase_core_py` and decode it in
`src/sase/core/bead_read_facade.py` as a typed Python row model. Keep the existing
`Issue.links` wire field intact: it is the bead's outbound storage projection used by
list/search and compatibility consumers, whereas the new detail rows are the
provenance-bearing neighborhood used by `show`. Allow the detail query to skip this
extra projection when links were disabled so `--no-links` is a real fast path.

Add Rust and binding parity tests for outgoing, incoming, symmetric, rewritten, removed,
and re-added edges; exact actor/timestamp/origin preservation; shorthand bead
resolution; deterministic ordering; and the disabled path. The core implementation must
still reduce the bead event store only once per detail request.

## Assemble one complete neighborhood without a second bead read

Teach `ArtifactLinkStore` to accept the authoritative bead-owned rows already returned
by the core detail snapshot when loading a bead's neighborhood. Merge those with:

- schema-v2 sidecar rows whose source or target is `bead:<full-id>`; and
- matching aggregate-only rows where neither endpoint owns sidecar JSON, notably
  automatic `agent:` to `bead:` citations and audited reads.

Deduplicate with the artifact relation registry's directed/undirected identity rules,
never reread or reduce the bead store when authoritative rows were supplied, and keep
the existing standalone store API behavior for callers that do not have a detail
snapshot. This closes the current gap where a generic bead load can miss aggregate-only
incoming rows. Cover sidecar, aggregate-only, bead-to-bead inbound/outbound, duplicate,
missing-sidecar, malformed-index, and no-second-reduction cases in the artifact-link
store tests.

Build a small presentation view model in the `sase` repository that turns validated
graph rows into the current bead's perspective. It should call the Rust-owned relation
registry for inverse labels and directionality, retain the original endpoints and stored
relation for JSON, classify origins for the two human sections, provide friendly origin
labels, and own the deterministic ordering. Both the text and JSON renderers must
consume this one projection so direction, labels, provenance, and ordering cannot drift.

## Wire the CLI, renderers, and documentation

Add the parser flag and help text in `src/sase/main/parser_bead_queries.py`, including
an example and an explicit note that compact format never expands links. In
`src/sase/bead/cli_query.py`, request and merge the neighborhood only for full/JSON
output when links are enabled; surface graph errors with the `--no-links` recovery hint.
Extend `src/sase/bead/cli_detail.py` and its palette with the two counted sections,
direction glyphs, wrapped reasons, placeholders for genuinely absent legacy provenance,
and additive rich styling. Extend `src/sase/bead/cli_detail_json.py` with the documented
top-level shape and an `include_links` path that also suppresses the raw issue field.

Exercise the public command rather than only helper functions:

- parser defaults and `--no-links` behavior for full, JSON, and compact formats;
- a mixed neighborhood containing every registered directed relation in both directions,
  undirected `related`, all origin classes, multiple uses, and canonical refs of several
  kinds;
- exact provenance after unrelated bead updates and link rewrites;
- no-link omission and proof that the disabled path never opens the artifact-link store;
- fail-loud malformed graph behavior plus successful `--no-links` recovery;
- stable JSON keys and preservation of the existing default `issue.links` field;
- plain/rich SGR invariance, narrow wrapping, Unicode refs/reasons, deterministic
  snapshots, and no stray escape sequences; and
- the existing structural budget: one Rust bead-store reduction and no per-link repo or
  remote probes.

Update the CLI goldens and the `sase bead show` sections in `docs/beads.md`,
`docs/configuration.md`, and parser help. Document the visual vocabulary, the
`artifact_links` JSON schema, the distinction from dependencies and stored `refs`, the
failure/recovery behavior, and `--no-links`.

## Verification

In the linked `sase-core` repository, run its required `just check` so core and PyO3
binding tests are both exercised. In the primary `sase` repository, run `just install`
before verification, then run focused core-facade, artifact-link-store, parser, CLI
show, JSON, style/golden, and structural-budget tests. Finish with `just check`; use
`just check-full` through `/sase_monitor` if scoped selection escalates or reports an
unusual selection, as required by the repository instructions.
