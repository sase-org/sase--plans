---
tier: tale
title: Recover and land the completion architecture phase
goal:
  Reapply the lost sase-rm.2 implementation on current master, verify its shared
  Rust/Python completion contracts and fast-path budgets, and close only the phase.
size: medium
bead: sase-rm.2
proposed_by: bbugyi200.athena.sase-rm.2
create_time: 2026-08-21 05:07:53
status: done
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.2.md)

# Recover and land the completion architecture phase

## Context and outcome

Phase `sase-rm.2` owns four ready task contracts: `sase-m1`, `sase-ou`, `sase-ov`, and
`sase-re`. Its bead notes describe an implementation and passing checks from a prior
workspace, but the current primary and linked `sase-core` checkouts are clean and do not
contain those changes. Treat those notes as design evidence, not proof about the current
tree. Reapply the accepted phase design to current `master`, account for intervening
changes, and leave close-ready evidence for the epic land agent.

The completed phase must provide:

- one Rust-core `%model` filter used by the Rust xprompt LSP and Python/ACE;
- lazy, compatibility-preserving package facades for completion-relevant leaf imports;
- full primary, linked, sidecar, and external repository candidates with kind and
  project display-name descriptions; and
- live effective snippet-trigger candidates for only `sase snippet show/delete`, without
  importing the Python xprompt stack on the fast path.

Do not close the four task beads, the parent epic, or any ancestor. Do not create task
beads. Record distinct discoveries on `sase-rm.2` as `PROPOSED FOLLOW-UP:` notes. Do not
edit memory files or add a feature flag.

## Implementation

### 1. Move model filtering into `sase-core`

Open the linked repository with `sase repo open` and honor its instructions. Add a
public serde wire entry carrying every field emitted by Python's model-completion
catalog and a pure filter in `crates/sase_core`. Preserve catalog order and all current
semantics: leading-`@` alias-only gating; case-insensitive matching; first-slash
provider scoping only when a provider row exists; qualified scoped results; nested
slash-bearing model fallback; hidden-provider behavior; and additive compatibility with
old catalogs lacking provider rows.

Export the filter from `sase_core`, expose a strict list-of-dicts PyO3 binding from
`sase_core_py`, and replace the local twin in `sase_xprompt_lsp` while retaining its
schema checks, graceful malformed-payload fallback, and LSP-only detail, documentation,
and filter-text rendering. Add table-driven Rust and binding coverage for aliases,
provider scope, nested slashes, case folding, unknown providers, ordering, metadata
preservation, and old-shaped catalogs.

### 2. Route Python and ACE through the core contract

Keep Python's authoritative catalog construction and live overlays in
`src/sase/xprompt/model_completion.py`, but centralize entry-to-wire and wire-to-entry
conversion and make filtering a thin `sase_core_rs` adapter. Ensure the launch payload
and interactive path use one field list, returned rows are validated strictly, and ACE
continues receiving the same `_ModelCompletionEntry` metadata and display behavior.
Retain and extend the existing Python filtering, catalog, and directive-interaction
tests for parity with the Rust contract.

### 3. Make package facades lazy without shrinking their APIs

Introduce one deterministic PEP 562 lazy-export helper and convert the eager
`sase.core`, `sase.sdd`, and `sase.workspace_provider` facades to explicit
symbol-to-leaf maps. Preserve `__all__`, `from package import symbol`, `dir(package)`,
type-checker visibility, and cached resolved attributes. Adjust any completion-relevant
facade whose eager re-export still pulls ACE/Rich/Textual.

Add subprocess import-contract tests proving representative leaves avoid unrelated
siblings and heavy UI modules, legacy symbols resolve from their owning leaves, and all
public exports resolve when explicitly requested. Re-measure the completion candidates
contract and tighten its current 150 ms local allowance only to a stable measured
budget, retaining the CI multiplier and forbidden-import assertions.

### 4. Serve the complete repository inventory

Once leaf imports are safe, make the repo candidate provider call
`collect_repo_inventory()` and project its deterministic, deduplicated inventory.
Candidate insertion text is the repository display name; descriptions contain the
repository kind (`primary`, `linked`, `sidecar`, or `external`) and owning project
display name. Honor optional project scoping and never expose an internal project key
when a display name is known. Keep the path read-only: candidate lookup must not clone,
materialize, or resolve repositories.

Cover primary, linked, configured and materialized sidecars, external repositories,
deduplication, order, project scoping, display names, heavy-import exclusion, and the
`repo` latency lane.

### 5. Add live snippet-trigger candidates on the fast path

Expose the existing Rust `load_editor_snippet_catalog` through `sase_core_rs`, accepting
optional project and root-directory context and returning its stable JSON-shaped
response. Reuse Rust catalog precedence and generated-alias logic rather than importing
Python xprompt or snippet services.

Add `ValueKind.SNIPPET`, assign it only to the trigger positionals for `snippet show`
and `snippet delete`, register a lazy candidate provider, and project effective triggers
including generated aliases. Apply any temporary catalog environment only around the
native load call. Use the existing source-path/TTL cache boundary so normal catalog
changes refresh without a per-keystroke rescan; malformed rows or native errors degrade
to no candidates through the provider boundary.

Test parser/spec classification, projection and descriptions, configured and generated
triggers, override precedence, native CWD/project selection, prefix filtering, cache
refresh, forbidden imports, and the `snippet` latency lane. Update completion/CLI docs
and the generated CLI-spec snapshot to describe the new kind and expanded repository
inventory.

## Verification and closure

1. Run `just install` in the primary checkout so its editable environment builds and
   loads the modified linked Rust binding.
2. Iterate with focused Rust core/LSP/PyO3 tests and focused primary tests for model
   filtering, lazy exports, repository/snippet candidates, kind generation, caching, and
   fast-path imports/latency.
3. Run `just check` in the linked `sase-core` repository, then `just check` in the
   primary repository. If the primary lane escalates or requests a full suite, run
   `just check-full` only through `/sase_monitor` with the required statuses and a
   useful continuation prompt.
4. Manually inspect repo and snippet candidate output, prefix/project behavior, and
   generated shell specs so `snippet` attaches only to show/delete trigger positions.
5. Reconcile the old evidence notes against the current implementation and append
   corrected evidence for each of `sase-m1`, `sase-ou`, `sase-ov`, and `sase-re`, naming
   concrete paths and exact passing checks.
6. Run `sase bead epic-symbols sase-rm.2`. Resolve every returned symbol or re-key its
   Justfile ownership line to `sase-rm` or the still-open dependent phase `sase-rm.5`.
7. Close only `sase-rm.2` with `sase bead close sase-rm.2 --note "<what was verified>"`.
   Leave all task beads and ancestor plan beads open for the epic land agent.
