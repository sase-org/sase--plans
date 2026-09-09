---
tier: tale
title: Resolve artifact-link index conflicts semantically
goal:
  Distinct concurrent artifact-link index updates merge automatically in both rebase
  paths while ambiguous histories continue to fail closed.
size: medium
proposed_by: bbugyi200.athena.sase-yy.1
bead: sase-yy.1
status: done
---

- **PARENT:** [202609/artifact_link_events_v2.md](artifact_link_events_v2.md)
- **BEAD:**
  [sase-yy.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.1.md)
- **COMMITS:**
  - [55770cb](https://github.com/sase-org/sase-core/commit/55770cb46f4ab99d61289fb6efee7c6fb96877db)
    — feat(artifact-links): merge link indexes

# Semantic resolver for artifact-link index conflicts

## Objective

Complete phase `sase-yy.1` by adding a conservative, deterministic three-way merge for
legacy schema-v2 `links/**/*.json` indexes in `sase-core`, exposing it through
`sase_core_rs`, and using one Python resolver chain from both managed SDD integration
and commit-first rebase repair. Distinct concurrent link rows must integrate without an
agent pause; ambiguous same-edge histories, modify/delete cases, malformed indexes, and
unrelated paths must continue to fail closed.

This phase is rollout protection only. It must not change artifact-link write behavior,
introduce immutable link events, or attempt to resolve managed Markdown blocks.

## Implementation

1. Add the Rust merge primitive in the linked `sase-core` repository.
   - Add `merge_artifact_link_indexes(base, ours, theirs)` beside the existing
     artifact-link wire logic and export it from the artifact-link module and crate.
   - Strictly validate schema version 2, canonicalize and require matching
     `artifact_ref` values, validate every row, and reject duplicate relation-aware
     dedup keys within an input index.
   - Merge by `artifact_link_dedup_key`: preserve a one-sided change from base, accept
     identical two-sided changes, union distinct additions, and reject competing
     same-key changes (including counter/description ambiguity and modify/delete) with a
     structured error that names the ambiguous keys. Model a missing Git base for a
     both-added file as an empty validated index in the Python adapter.
   - Preserve surviving base-key positions in base order, then append every key absent
     from base sorted by `(created_at, dedup key)`. This makes swapped branch inputs and
     repeated merges byte/logically identical without guessing from timestamps or
     counters.
   - Add Rust unit/property-style coverage for validation, distinct additions, one-sided
     and identical edits/deletes, same-key refusals, symmetry, idempotence, and
     deterministic ordering.

2. Expose the primitive through the Rust/Python boundary.
   - Add an `artifact_link_merge_indexes` PyO3 binding that accepts/returns plain index
     dictionaries and maps structured Rust failures to an informative Python error.
   - Cover the binding in `sase-core`'s binding tests and export registration.
   - Add the binding to SASE's `tools/validate_sase_core_rs` contract and its contract
     tests, and call it through a thin Python facade. Keep binding names statically
     visible to `tools/check_sase_core_rs_bindings`.
   - Follow current release ownership: do not hand-edit `sase-core` crate versions or
     invent an unpublished Python dependency floor. Ensure the normal linked-core
     landing/pin ratchet records the first core revision containing the binding; only
     use the supported core-window ratchet if that release is actually published.

3. Add the Python artifact-link conflict resolver and shared dispatch chain.
   - Add a resolver in the artifact-link/SDD domain that reuses
     `sase.bead.conflict_resolver_git` for repository discovery, conflict-stage probes,
     rebase-aware upstream/local stage ordering, and staging.
   - Claim only canonical-location `links/**/*.json` paths. Resolve stage sets `{1,2,3}`
     (both modified) and `{2,3}` (both added); reject `{1,2}`/`{1,3}` modify-delete
     conflicts and unexpected stage shapes without touching them.
   - Decode stage JSON, pass validated index dictionaries to the Rust facade, write with
     the same `indent=2`, sorted-key, UTF-8, trailing-newline bytes as
     `atomic_write_json`, and stage only successfully resolved paths.
   - Introduce a shared semantic-resolver chain that assigns every current unmerged path
     to the bead resolver or artifact-link resolver before mutating anything, rejects
     unclaimed paths with the existing conservative behavior, dispatches each resolver
     over its owned paths, and aggregates resolved files plus bead relocation metadata.
     Preserve the standalone bead resolver command's current strict behavior.

4. Wire the shared chain into both rebase-repair paths.
   - Replace the direct bead-only call in `src/sase/sdd/_repository_integration.py` so
     managed sidecar integration can repair link-only and mixed bead/link rounds,
     continue rebase, and still abort back to the exact starting state on any unclaimed
     or failed conflict.
   - Add a clearly named repaired-semantic-conflicts integration status while retaining
     the legacy bead-specific enum value as a recognized success for compatibility;
     return the semantic status for resolver-chain repairs and include it in
     `SddIntegrationOutcome.succeeded` and machine-owned recovery handling.
   - Generalize the commit-first path in
     `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` from its bead-prefix gate
     to the same shared claim-and-dispatch chain. Keep its bounded rebase loop,
     checkpoint recovery, and the existing user-facing paused-conflict/resume message
     when any path is unclaimed or resolution fails.

5. Prove resolver and integration behavior in SASE.
   - Add focused Python resolver tests for both-added and both-modified indexes,
     canonical serialization, rebase stage orientation, malformed/mismatched inputs,
     ambiguous same-key edits, modify/delete refusal, and unrelated-path non-claims.
   - Extend transactional integration tests with independent clones that commit distinct
     reader rows to the same index and verify managed integration completes cleanly,
     reports the semantic repair status, and preserves every row. Add mixed bead/link
     coverage and prove an ambiguous link conflict aborts and restores the exact
     starting state with an informative error.
   - Extend commit-first VCS integration coverage with the same divergent-index setup,
     proving automatic completion for resolvable links and the existing resumable pause
     for ambiguous or unclaimed conflicts.

6. Verify and close only the assigned phase.
   - In the linked `sase-core` repository, run its required `just check` gate.
   - In SASE, install/rebuild the local Rust binding as needed, run focused
     Rust-binding, resolver, repository-transaction, and commit-first tests while
     iterating, then run the mandatory SASE verification from the project's lint/test
     memory (`just check`).
   - Inspect `git diff` and both repository statuses to confirm no unrelated changes, no
     paused rebase, and no generated residue.
   - Run `sase bead epic-symbols sase-yy.1`; resolve every remaining symbol or re-key
     its Justfile line to the parent/later open phase as appropriate. Then close only
     `sase-yy.1` with
     `sase bead close sase-yy.1 --note "<verified Rust, Python, and integration evidence>"`.
     Do not close `sase-yy` or any ancestor, and record any genuinely out-of-scope
     discovery only as a `PROPOSED FOLLOW-UP:` note on this phase.

## Acceptance criteria

- Two branches adding distinct rows to the same valid schema-v2 artifact-link index
  integrate automatically and produce deterministic normal-writer bytes.
- One-sided/identical changes merge, while ambiguous same-key histories, modify/delete,
  malformed/mismatched indexes, and arbitrary files remain unresolved with actionable
  diagnostics and no guessed data.
- Both managed SDD integration and commit-first rebase repair use the same resolver
  claim/dispatch rules; mixed supported conflicts complete, while any unsupported path
  preserves the existing abort-or-resumable-pause contract.
- Rust-core and SASE verification pass, the new binding is contract-checked, epic
  symbols are cleared or safely re-keyed, and only phase `sase-yy.1` is closed.
