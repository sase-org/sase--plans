---
tier: tale
title: Unify completion filtering and expose complete live catalogs
goal:
  Make model filtering a single Rust-core contract shared by ACE and the xprompt LSP,
  keep leaf imports completion-safe, and offer complete repository and snippet-trigger
  candidates within the measured shell-completion budget.
size: medium
proposed_by: bbugyi200.athena.sase-rm.2
bead: sase-rm.2
create_time: 2026-08-20 15:01:36
status: wip
---

- **PROMPT:**
  [prompts/202608/completion_architecture.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/completion_architecture.md)
- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.2.md)

# Unify completion filtering and expose complete live catalogs

## Outcome

Complete phase `sase-rm.2` and provide close-ready evidence for its four assigned task
beads without closing those tasks or the parent epic:

- `sase-m1`: one `sase-core` implementation filters `%model` catalog entries for both
  the Python/ACE binding and the Rust xprompt LSP.
- `sase-ou`: importing a `sase.core.*` leaf no longer eagerly imports unrelated facade
  modules, and compatibility exports remain available lazily.
- `sase-ov`: repository completion offers primary, linked, sidecar, and external names
  with kind descriptions while preserving the fast-path import and latency contracts.
- `sase-re`: `sase snippet show` and `sase snippet delete` complete live effective
  snippet triggers through a completion-safe catalog projection.

The implementation spans the primary `sase` checkout and the linked `sase-core` checkout
opened through `sase repo open`. Shared filtering and native catalog loading belong in
Rust core; Python remains catalog assembly, wire validation, parser-kind classification,
caching, and presentation glue.

## Constraints and invariants

- Preserve the existing `%model` behavior exactly: leading-`@` alias gating, first-slash
  provider scoping, nested slash-bearing model names, case-insensitive matching,
  provider/model/alias catalog order, hidden-provider behavior, and additive old/new
  catalog compatibility.
- Keep the model catalog schema additive. Old catalogs without provider rows continue to
  use fallback prefix matching, and malformed or unsupported LSP catalog payloads still
  degrade to empty results rather than crashing the server.
- Preserve public imports such as `from sase.core import shorten_path`, equivalent SDD
  exports, and workspace-provider exports. Package facades become lazy; their public API
  does not disappear.
- Completion imports and calls must not load `sase.main.parser`, `sase.ace`, Rich,
  Textual, or other unrelated facade trees. Candidate lookup is read-only,
  non-interactive, bounded by the existing disk cache, and does not clone or resolve a
  repository as a side effect.
- Repository candidates use project display names, never internal project keys when a
  display name is known, and include each repository's `primary`, `linked`, `sidecar`,
  or `external` kind in its description.
- Snippet candidates include the effective trigger and generated aliases returned by the
  Rust editor catalog, use the current working directory/project context, and do not
  import the Python `sase.xprompt` or full snippet service on the fast path.
- Do not add a feature flag: this work completes already-approved production behavior
  and removes duplicate implementation; it is not an early or disabled branch.
- Do not edit SASE memory files, create follow-up beads, close assigned task beads, or
  close the parent epic. Distinct discoveries become `PROPOSED FOLLOW-UP:` notes on
  `sase-rm.2`.

## Implementation

### 1. Put model completion filtering in `sase-core`

Add a public, serde-compatible model-completion entry wire and a pure filter function in
`crates/sase_core`. Carry every field currently emitted by the Python catalog so the
binding can round-trip rows without losing ACE display metadata. Implement the current
filter precedence once in core:

1. a leading `@` admits only implicit/user aliases and matches normalized aliases;
2. a known provider row before the first `/` scopes to that provider's model rows and
   returns qualified copies in original catalog order; and
3. otherwise value/alias prefix matching preserves legacy and nested-slash behavior.

Export the wire and filter from `sase_core`, add table-driven unit coverage for empty,
alias-only, scoped, nested-slash, case-insensitive, unknown-provider, ordering, and
old-shaped catalogs, and expose the function through `sase_core_rs` with strict input
validation and a plain list-of-dicts result.

Update `crates/sase_xprompt_lsp/src/server.rs` to parse catalog rows into the core wire
and call the shared filter before converting them into LSP candidates. Delete the local
filtering twin while retaining LSP-specific detail/documentation rendering, catalog
schema checks, and graceful malformed-row handling. Re-run the existing provider-scope
and version-skew tests against the shared function.

### 2. Route Python and ACE through the core filter

In `src/sase/xprompt/model_completion.py`, keep Python's authoritative catalog builder
and overlay logic, but replace its local provider/alias filter with a strict thin
adapter to the new `sase_core_rs` binding. Centralize entry-to-wire and wire-to-entry
conversion so `model_completion_catalog_payload()` and interactive filtering use the
same field list and validate the returned shape. Remove the Python filtering helpers
that are now owned by core.

Retain the existing Python catalog and ACE interaction tests, and add binding parity
coverage that exercises provider scoping, aliases, nested slash values, ordering, and
all metadata fields. ACE continues to consume `_ModelCompletionEntry` objects, so its
rendering and provider drill-down behavior remains unchanged while its filtering comes
from Rust.

### 3. Make facade imports lazy and completion-safe

Replace eager re-export imports in `sase.core.__init__` with a deterministic lazy export
map using module `__getattr__`/`__dir__`, caching resolved attributes in the package
namespace. Preserve `__all__` and every existing public symbol. Apply the same
compatibility-preserving pattern to the SDD and workspace-provider package facades whose
`__init__` execution currently pulls ACE when repository inventory imports their leaf
modules.

Add subprocess import-contract tests proving:

- importing representative `sase.core`, `sase.sdd`, and `sase.workspace_provider` leaves
  does not load unrelated siblings or ACE;
- importing a legacy symbol from each package lazily loads only its owning module; and
- the complete public `__all__` surfaces still resolve when explicitly requested.

Measure the candidates fast path after the split and tighten
`tests/main/test_completion_candidates_contract.py` from its currently documented
eager-facade allowance to a budget supported by repeated local measurements, retaining
the CI multiplier and import-set assertion.

### 4. Serve the complete repository inventory

Change the repo candidate provider to call `collect_repo_inventory()` only after the
facade import paths are safe. Project the full inventory into deduplicated candidates:
use the repository display name as the insertion, its kind plus owning project display
name as the description, honor optional project scoping, and retain deterministic
inventory order. Ensure the inventory path remains read-only and does not invoke
workspace-provider ref resolution or repository cloning.

Extend focused repository-candidate fixtures to include primary, linked, configured and
materialized sidecars, and opened external repositories. Assert kind descriptions,
display-name projection, deduplication, project scoping, no-heavy-import behavior, and
the `repo` latency lane.

### 5. Add completion-safe snippet-trigger candidates

Expose the existing Rust `load_editor_snippet_catalog` through `sase_core_rs` with a
small binding accepting an optional project reference and root directory, and returning
its stable JSON-shaped response. This reuses Rust's native xprompt/config discovery,
snippet composition, override precedence, and generated-alias projection without loading
Python xprompt, snippet, Rich, or ACE modules.

Add a `snippet` completion value kind, map only the `trigger` positionals of
`sase snippet show` and `sase snippet delete` to it, and register a lazy catalog
provider. Convert successful Rust catalog entries into candidates with trigger as the
value and concise source/description text; malformed rows or catalog errors degrade to
no candidates through the existing completion-provider boundary. Include the kind in the
shipped-kind contract and cache it with a source path/TTL strategy that observes normal
catalog changes without per-keystroke rescans.

Test parser/spec classification, candidate projection for xprompt-derived, user-config,
overridden, and generated-alias triggers, project/CWD selection, prefix filtering, cache
refresh, no-heavy-import behavior, and the `snippet` latency lane. Update
`docs/completion.md` and the CLI command table to list and explain the new live trigger
kind and the expanded repository inventory, keeping measured figures aligned with the
new budget.

## Verification and phase handoff

1. Run `just install` before primary-repository tests so the local Python environment
   contains the updated linked Rust binding.
2. While iterating, run focused Rust unit/LSP/PyO3 tests and focused primary tests for
   model filtering, lazy imports, repo/snippet catalogs, completion generation, cache,
   and fast-path contracts.
3. Run `just check` in `sase-core`, as required by that repository's instructions.
4. Run the measured candidates contract repeatedly with the cache disabled where
   appropriate, then run `just check` in the primary repository. If the primary check
   escalates or reports unusual selection, use the prescribed monitored full check.
5. Manually inspect `sase completion candidates repo` and
   `sase completion candidates snippet`, including project-scoped and prefix-filtered
   calls, and confirm representative generated zsh/bash/fish specs attach `snippet` only
   to `snippet show/delete` trigger positions.
6. Append one evidence-rich note block to `sase-rm.2` for each of `sase-m1`, `sase-ou`,
   `sase-ov`, and `sase-re`, naming the implementation paths and exact passing checks.
7. Run `sase bead epic-symbols sase-rm.2`. Resolve every returned symbol or re-key its
   Justfile ownership line to the parent epic or still-open later phase before closing.
8. Close only `sase-rm.2` with a concise `sase bead close ... --note` that names the
   cross-repository and primary verification. Leave the four task beads and every
   ancestor for the epic land agent.
