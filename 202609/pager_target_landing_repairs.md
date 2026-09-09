---
tier: epic
status: done
title: Complete pager target identity before landing sase-xy.5
goal:
  "Pager links use the complete semantic target and owning-repository contract on clean
  installations: stale provenance falls through safely, repository/revision/filter
  evidence cannot be bypassed, configured document kinds and fragments survive into
  actions, and the published Rust binding floor contains every required API."
parent_bead: sase-xy.5
phases:
  - id: repository-target-contract
    title: Finish repository-owned target selection in Rust core
    size: medium
    depends_on: []
    description:
      "repository-target-contract: make source-directory, checkout identity, revision,
      filtering, directory, fallback, ambiguity, and bounded-search outcomes truthful in
      the shared Rust resolver."
  - id: semantic-pager-contract
    title: Preserve the complete scanner target through pager actions
    size: medium
    depends_on:
      - repository-target-contract
    description:
      "semantic-pager-contract: carry configured kinds, Markdown destinations and
      fragments, hosted provenance, and owner-scoped resolver outcomes through every
      pager entry point without Python fallback guesses."
  - id: clean-install-contract
    title: Ratchet the binding floor and prove the combined clean-install contract
    size: medium
    depends_on:
      - repository-target-contract
      - semantic-pager-contract
    description:
      "clean-install-contract: publish or consume the completed core contract, ratchet
      packaging and capability validation, and exercise the full rendered-link corpus
      from a clean environment."
proposed_by: bbugyi200.athena.sase-xy.5.land
bead_id: sase-xy.5.5
create_time: 2026-09-09 19:52:35
---

- **PROMPT:**
  [prompts/202609/pager_target_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/pager_target_landing_repairs.md)
- **PARENT:**
  [202609/pager_target_integrity.md](https://github.com/sase-org/sase--plans/blob/main/202609/pager_target_integrity.md)
- **BEAD:**
  [sase-xy.5.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xy/sase-xy.5.5.md)

# Complete the pager target landing

## Why this child plan exists

The land audit for `sase-xy.5` reviewed its four closed phase beads and every note, the
parent plan `plan:202609/pager_target_integrity.md`, current source in `sase` and the
opened `sase-core` checkout, and all commits landed after the epic began. The screenshot
paths and basic scanner-to-action flow now work, but several claims in the parent plan
are not true on the combined tree. These are caused by this epic and must be completed
before its land agent can close it.

Phase 1's scanner and Python adapter were co-landed in the otherwise unrelated shared
pending-action commits `f7852f54` (`sase-core`) and `e0c575503` (`sase`), rather than in
commits carrying `sase-xy.5.1`. Phase 2 landed as `0ec30508` in `sase-core` and
`d8a299c2c` in `sase`; phases 3 and 4 landed as `2b08ca301` and `2ba228da3` in `sase`.
The current code, not the phase notes, is the acceptance baseline.

## Reproduced gaps

1. A clean editable install resolves `sase-core-rs==0.32.34` from `uv.lock` and the
   declared `>=0.32.34,<0.33.0` range. That wheel has neither
   `artifact_ref_scan_document` nor
   `artifact_ref_target_resolution_wire_schema_version`; calling the new Python adapter
   fails before resolution. The phase checks passed only after locally rebuilding the
   linked core. `tools/validate_sase_core_rs` also does not require the new
   document-scan and source-target capabilities, so setup cannot diagnose this contract
   directly.
2. `resolve_against_attached_checkouts()` returns `missing_checkout` immediately when
   every recorded checkout is stale. It never reaches a live alternate checkout from the
   same repository inventory, contradicting the deleted-producer-workspace requirement.
   The existing test leaves `owner.checkout_candidates` empty and does not exercise that
   production shape.
3. `ArtifactRefDocumentOwnerWire.source_directory` and `project_key` are stored but not
   used by `resolve_document_source_target()`. A one-component path that exists beside
   its source document is therefore reported missing because suffix search is disabled
   for one component.
4. An owner's `revision` is copied into a successful response without comparing it to
   repository or checkout evidence. `ArtifactRefRepository.shas` is not populated by
   normal `artifact_ref_context()` assembly. A different local revision can therefore be
   reported as an exact target, while `UnavailableRevision` is unreachable.
5. The Rust resolver accepts files only. Python then probes every owner and inventory
   checkout with `_existing_owner_scoped_path()` and returns the first existing file or
   directory. That fallback ignores explicit repository scope, distinct-repository
   ambiguity, revision evidence, and filters. It recreates the guessing behavior the
   epic was meant to remove.
6. `DeniedFiltered` is present in the shared target-resolution wire but has no path
   policy input or producing branch. The parent contract requires configured access
   denials not to fall through to unrestricted lookup. The implementation must either
   carry an applicable existing policy into repository candidates and enforce it, or
   remove an unsupported outcome and prove that every configured filter remains owned by
   the typed-document resolver; it must not retain a public outcome that tests can only
   fake.
7. The Rust scanner accepts caller-supplied `known_kinds`, but pager `scan_links()`
   always calls it with an empty set. Custom configured document kinds therefore work in
   the thin wrapper test but are not rendered through a real pager entry point.
8. The scanner produces rich target metadata (`artifact_reference`, Markdown and hosted
   destinations), but `LinkSpan` collapses it to kind/text/target. Markdown file
   fragments such as `docs/guide.md#section` are treated as filename bytes, so the
   fragment neither lands nor receives an explicit unavailable result. Hosted provenance
   cannot participate in the parent plan's repository/revision agreement rule after the
   Python conversion.
9. `document_owner_from_artifact()` adds every repository checkout in the artifact
   context to the owner's unqualified `checkout_candidates`. The Rust attached-candidate
   fast path can consequently choose the first of two unrelated repositories before the
   inventory has a chance to report ambiguity.

## Integration baseline and constraints

- Preserve the single background resolution and pure context-merge work from child epic
  `sase-xy.4` (`51445642`, `4b90cc9e`) and the source-language/line-number pager work
  that landed while `sase-xy.5` was active. Do not revive synchronous UI-thread
  filesystem or Git work.
- The machine-writable artifact-link changes (`87f4cf14`) and later test-cost/flake
  bookkeeping do not add pager entry points. Keep them intact.
- The pending-action Symvision follow-up proposed by `sase-xy.5.1` is already resolved:
  the Telegram adapter consumes the shared API, the host symbols remain live, and the
  pager lint cleanup is present in `c92cee70`. Do not duplicate that work.
- The additive proposal from `sase-xy.5.4` to recognize bare extensionless relative
  directories is intentionally outside this child plan and is tracked as ready feature
  task `sase-yb`. Existing `./directory` links remain in scope here, including truthful
  repository-aware resolution.
- Another active epic may land Rust editor/dispatch changes after this audit. Re-open
  the current `sase-core` checkout through the normal repository workflow, integrate its
  latest default branch before editing, and choose a released dependency floor that
  contains both those changes and this plan's completed contract. Do not pin to a local
  unpublished revision or manually bump Rust crate versions.

## Behavioral contract

1. An attached checkout is evidence, not a terminal namespace. Stale or missing attached
   paths fall through to other checkouts of the same identified repository. Attached
   candidates from unrelated repositories never become one ordered first-hit list.
   Direct and suffix hits are deduplicated by logical repository/path identity; distinct
   repositories remain ambiguous with followable evidence.
2. Resolve a relative target beside `owner.source_directory` before repository-root
   candidates when that directory is valid provenance. Enforce traversal safety and
   repository containment. Files and explicitly rendered directories use the same core
   decision path and failure model.
3. When a repository or revision is explicit, a successful target must prove both. A
   checkout at a different revision is not exact. Use existing bounded VCS/artifact
   materialization facilities when they can prove and expose the requested content;
   otherwise return `unavailable_revision` with retryability and evidence. Do not run
   interactive or unbounded Git commands.
4. Preserve all applicable configured path policies before probing or suffix-walking. A
   denied path produces the declared filtered outcome and cannot be recovered by a
   Python direct probe. If the host has no repository-source policy to carry, make that
   absence explicit in the wire/design and remove misleading dead branches rather than
   inventing security semantics.
5. Freeze configured artifact-ref kinds and aliases into the immutable pager document or
   section before UI rendering. Scanning remains pure and I/O-free. Every production
   adapter with an artifact context passes its `known_kinds`; an unowned stdin/file path
   retains the compiled kind catalog without global config reads during paint.
6. Preserve one semantic target record from scan through follow, copy, edit, cache
   identity, navigation, and diagnostics. Markdown destinations control the action;
   line/column and supported `#L...` or heading fragments land, or explicitly report an
   unavailable fragment without silently opening the wrong location. Managed-table
   hosted destinations remain separately actionable and qualify local repository or
   revision identity only after agreement is proven.
7. Resolution performs one bounded repository/context assembly and one target search per
   action. Worker-computed diagnostics and candidates are reused by the UI. No Python
   fallback may stat arbitrary owner repositories after core returned a scoped,
   ambiguous, filtered, or revision failure.
8. A clean install from the declared dependency range exposes every scanner/resolver
   binding and schema version the Python checkout requires. Capability validation names
   these APIs, and the lock selects a released wheel containing the final core work.

## Phase 1: Finish repository-owned target selection in Rust core

Work primarily in `sase-core`'s `artifact_ref/repository_resolution.rs`, wire types,
PyO3 binding, and focused tests, with only the thin Python model/context changes needed
to construct honest wire data.

Replace bare attached checkout strings with sufficient repository identity, or validate
them against the inventory before selection. Use `source_directory` first, fall through
stale attached paths to live same-repository checkouts, and handle files and directories
without a Python first-hit escape hatch. Define one decision function for exact,
source-relative, alternate-checkout, suffix, ambiguous, missing-checkout, filtered,
revision-unavailable, proven-missing, and budget/error outcomes. Preserve ordered
checkouts within one repository and deduplicate its copies before ambiguity.

Make revision behavior real. Populate or derive bounded checkout revision evidence off
the event loop, compare it before success, and use existing artifact/VCS materialization
only where it can prove the requested repository/path/revision. Do not label the current
worktree exact for a historical revision merely because the path exists. Decide the
filter contract from actual configuration: transport and enforce an applicable source
path policy, or remove the dead source-resolution category while retaining typed
document filtering. Document and test the decision.

Add Rust unit and PyO3 round-trip tests that reproduce stale attached checkout fallback,
source-relative one-component paths, same path in distinct attached repositories,
directory ambiguity, explicit repository exclusion, matching and mismatching revision,
applicable filter denial, traversal, budget exhaustion, and unavailable checkouts. Run
the complete `sase-core` check surface required by that repository.

## Phase 2: Preserve the complete scanner target through pager actions

Update the Python scanner/document model and every construction path in `pager/`,
`artifact_cli/read.py`, `main/pager_handler.py`, ACE's link-index adapter, bead-show
documents, and generic pager entry points. Carry immutable known-kind data and the full
semantic target metadata rather than discarding everything except a string. Keep custom
attached target object identity unchanged.

Correct `document_owner_from_artifact()` so only checkouts actually evidenced as the
document's owner are attached; leave the repository inventory in the artifact context.
Remove `_existing_owner_scoped_path()` or reduce it to a core-approved exact candidate
that cannot override scope, ambiguity, revision, or policy. Reuse the phase-1 core
outcome for file and directory landings, copy, edit, and diagnostics.

Implement fragment interpretation for Markdown file targets and retain typed artifact
fragments. Line and column suffixes continue to drive editor placement. Supported
headings land on their real line; unsupported fragments stay visible and report why
instead of silently dropping bytes. Preserve both the canonical typed target and the
separate hosted URL occurrence in generated link tables, validating repository/revision
agreement before using hosted provenance locally.

Extend focused Python and Textual Pilot tests with a custom configured document kind and
alias through production construction, stale owner workspace plus live alternate
checkout, two unrelated attached repositories, same-named directories, revision
mismatch, filtered source policy (if supported by phase 1), Markdown line/heading
fragments, URL provenance mismatch, copy/edit parity, retry/reload, and one-search/no-UI
I/O instrumentation. Keep the original screenshot corpus and all existing pager labels.

## Phase 3: Ratchet and prove the clean-install contract

After the Rust changes are on the default branch and released, raise the minimum
`sase-core-rs` version and refresh `uv.lock` to the first release containing the entire
contract. Extend `tools/validate_sase_core_rs` and its tests to require the document
scanner, target resolver, and both schema-version getters. A clean environment using the
declared dependency—not a locally rebuilt untracked wheel—must import and execute the
adapters.

Re-run the original screenshot fixture plus every rendered-link contract/navigation/
failure suite, artifact-ref resolution and parsing suites, artifact-read/ACE/bead-show
entry-point suites, and pager visual snapshots. Verify source syntax and line-number
changes that landed during the epic remain intact. Run `just install` as required,
ordinary repository checks after changes, the Rust repository's full check, and the
monitored full landing gate prescribed by `lint_and_test.md`. Distinguish any current
known full-lane flake or unrelated cross-repository failure with existing bead evidence;
do not weaken tests, filters, link scanning, or dependency validation to obtain green.
