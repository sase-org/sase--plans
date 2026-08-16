---
tier: tale
title: Compile and render the declarative ref.pane contract
goal:
  A schema-v1 document provider can declare safe Artifacts presentation data in
  ref.pane, have that data validated and compiled into its pane contract and cache
  identity, and receive provider-specific rows, ordering, facets, grouping, and empty
  copy without executing provider code; the research provider demonstrates the full
  path.
size: medium
proposed_by: bbugyi200.athena.sase-m6.8
bead: sase-m6.8
create_time: 2026-08-16 13:45:25
status: done
---

- **PROMPT:**
  [prompts/202608/declarative_ref_pane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/declarative_ref_pane.md)
- **PARENT:** [202608/artifacts_pane_contract.md](artifacts_pane_contract.md)
- **BEAD:**
  [sase-m6.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.8.md)

# Plan: Compile and render the declarative `ref.pane` contract

## Outcome

Finish phase `sase-m6.8` by making `ref.pane` the Python-owned declaration layer over
the query, relation, grouping, and shell infrastructure already landed by the preceding
epic phases. A provider supplies only validated data: host code compiles it once during
discovery, records a deterministic presentation digest, and renders/sorts/groups rows
from the immutable contract. Existing schema-v1 providers with no pane block keep the
current host defaults.

The end-to-end proof is the linked `sase-research-artifacts` provider: it remains wire
schema version 1, declares a Research-specific pane, and changes `status` from a free
string to an enum with explicit values so completion, faceting, grouping, sorting, row
badges, and empty-state copy come from its data rather than SASE-side provider branches.

## Current-tree evidence and boundaries

- `src/sase/sidecar_ref_config.py` rejects inline unknown `ref` keys through
  `_KNOWN_REF_CONFIG_KEYS`; plugin `use:` specs take a different path. Add a single
  `REF_PANE_CONFIG_KEY` and normalize both paths to the same effective pane block.
- Rust's schema-v1 validator returns the existing provider-spec digest but does not own
  Python presentation. Keep both `DOCUMENT_REF_PROVIDER_SPEC_SCHEMA_VERSION` and the
  Rust wire schema at 1. A pane-only edit must still invalidate presentation caches.
- `ArtifactsPaneContract` already owns `presentation_digest`, query profile, relations,
  grouping, detail fields, and empty state. Extend this immutable model with normalized
  row/sort/facet/description presentation values rather than adding a second runtime
  configuration lookup.
- `ArtifactsDocumentsPane` already loads snapshots and compiles query/relation indexes
  off the Textual event loop. Continue that discipline: validation and row/sort/group
  preparation happen during discovery or worker snapshot construction, never during a
  keypress, render callback, or completion lookup.
- The provider document renderer is still plan-shaped: rows use
  `proposal_text`/`active_plan_text`/`archive_text`, ordering is hard-coded to plan
  recency and sections, grouping only recognizes built-in mode ids, and empty detail
  copy says Plans. Change only the generic document-provider path; the built-in Plan
  adapter retains its current specialized pipeline behavior.
- Providers may reference declared properties and closed host common fields only. They
  may not supply Rich markup, colors, keybindings, commands, Python entry points,
  callbacks, mutations, approvals, or capability assertions. Capabilities remain derived
  from facts; existing suppression-with-reason behavior remains separate.

## Declaration and validation contract

Introduce frozen Python presentation records for the normalized pane block, covering:

- `label`, `description`, and relative `order`;
- `row` roles (`title`, ordered badges/secondary fields, and prioritized list fields);
- ordered `default_sort` fields with `asc`/`desc` direction;
- typed facet fields;
- `group_by` as a safe shorthand over declared property keys, compiled into the existing
  `PaneGroupingDecl`/`GroupFoldRegistry` path;
- `empty_state.title` and `empty_state.body`;
- query/relation/grouping declarations already implemented by prior phases, preserving
  their existing schema-v1 forms and accepting pane-owned hints only where they map
  directly onto the existing host types.

Every presentation field is optional. Defaults preserve current behavior and are derived
from kind, declared properties, and the existing contract. Normalize authoring order and
aliases before hashing. Validate references against declared properties plus a small
documented host-common vocabulary (`filename`/title, path, project, and other
already-materialized row fields). Enum/list/scalar/date types determine which fields may
be faceted, sorted, badged, or grouped. Reject executable or host-owned concepts.
Malformed required constructs compile a visible degraded provider pane with a precise
`invalid_ref_pane` diagnostic; absent or optional hints fall back to host defaults and
one malformed document value is treated as missing for that row rather than removing the
tab.

## Implementation

1. In `src/sase/sidecar_ref_config.py`, add `REF_PANE_CONFIG_KEY` to the inline
   allowlist and preserve/deep-merge the pane mapping for both inline and `use:`
   provider specs. Add focused policy tests proving the two authoring paths normalize
   identically, schema version stays 1, lists replace rather than concatenate, and
   pane-only changes do not alter the Rust provider digest.

2. Add the immutable pane-presentation records and a pure compiler beside the existing
   provider contract compiler. Validate keys, shapes, referenced fields, property-type
   restrictions, sort directions, duplicate declarations, and bounded plain-text copy.
   Compile valid declarations into `ArtifactsPaneContract`; route errors through
   `ContractCompileResult` so the descriptor stays visible but degraded. Keep legacy
   top-level relation/grouping/suppression declarations compatible and make the pane
   compiler reuse the existing query, relation, grouping, and capability derivation
   functions rather than duplicating them.

3. Compute a canonical SHA-256 digest over the normalized Python pane declaration and
   include it in the contract's `presentation_digest` independently of the Rust
   provider-spec digest. Propagate the resulting presentation digest through the
   descriptor and the provider document snapshot/query/row cache identities so a
   pane-only edit cannot reuse stale rows, sort order, facets, or grouping. Keep digit
   assignment and host-derived accent in the final contract digest without letting
   providers allocate either.

4. Make provider descriptors honor compiled `label`, `description`, and `order` while
   retaining deterministic kind-based fallback labels and stable host-assigned digits.
   Sort provider descriptors by declared order with a deterministic label/kind tie
   break, leaving fixed panes in their established positions. Do not special-case the
   Research kind.

5. Parameterize the generic document-provider snapshot and rendering path by the
   compiled presentation data. Build a provider row view from already-loaded frontmatter
   and host-common fields, apply stable multi-key sort off-thread, render the declared
   title/badges/secondary fields with safe host styling, expose only the declared facet
   values to completion, and compile `group_by` into the existing fold registry. Drive
   no-content copy from `contract.empty_state`; retain the distinct host-owned no-match,
   loading, stale, and degraded states. Leave the Plan adapter's proposal/active/archive
   sections, approvals, and existing row renderers unchanged.

6. Extend focused contract, descriptor, query-profile, snapshot, row-rendering,
   grouping, empty-state, and cache-invalidation tests. Cover a provider with only
   `kind`/inventory defaults; a complete pane; invalid property references and unsafe
   fields; deterministic digest behavior; pane-only digest changes; enum completion;
   stable sort ties; missing/mistyped per-row values; and two providers whose declared
   order collides.

7. In the linked `sase-research-artifacts` repo, update `RESEARCH_REF_PROVIDER_SPEC` and
   its tests/docs to keep `schema_version: 1`, declare `status` as an enum with explicit
   values, and add a pane block for Research label, description, row fields,
   updated-time descending sort, status/tags facets, status grouping, and
   Research-specific empty copy. Verify the plugin through its own `just check` and
   through SASE's provider integration tests.

## Verification

- Run `just install` in the SASE workspace before tests, then run focused
  `sidecar_ref_config`, query-profile, Artifacts contract, provider discovery,
  document-row/sort/group/facet, and pane integration tests while iterating.
- Run `just install` and `just check` in `sase-research-artifacts` after its provider
  spec and docs/tests change.
- Exercise the research spec through SASE and assert the compiled contract carries its
  declared label, row/sort/facet/group/empty-state values; `status` completion is the
  declared enum; and changing only `ref.pane` changes the presentation/cache identity
  while leaving the Rust provider digest stable.
- Inspect any changed Artifacts PNG snapshots before accepting them. Update only
  intentional Research/provider presentation goldens, then rerun the affected visual
  nodes.
- Run `just check`. Because this phase touches broad Artifacts contract and rendering
  paths, then run `just check-full` through `/sase_monitor` with a follow-up action, as
  required by the parent epic's verification contract. Require both repositories to be
  clean except for the intended changes before closing only `sase-m6.8`.

## Non-goals

- No Rust provider-wire change, schema-version bump, typed `ArtifactValueWire`, or
  provider callback execution.
- No provider-declared capability assertion, mutation, approval, keymap, color, command,
  or layout/widget API.
- No redesign of the Plan adapter, shared query grammar, relation panel, fold registry,
  or keymap migration owned by sibling phases.
- No final epic conformance/docs/performance closeout owned by `sase-m6.10`; this phase
  adds focused coverage and records any broader follow-up on `sase-m6.8` for the epic's
  land agent.
