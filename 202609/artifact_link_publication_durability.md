---
tier: tale
title: Require durable owners and publish complete event files atomically
goal:
  Artifact-link event publication acknowledges an operation only when a durable owner
  receipt exists, installs event objects with an atomic no-replace write that survives
  process death, and routes manual `link add/rm` plus plan-links ingestion through the
  hidden machine store.
size: medium
bead: sase-yy.8.2
proposed_by: bbugyi200.athena.sase-yy.8.2
status: done
---

- **PARENT:**
  [202609/artifact_link_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_landing_repairs.md)
- **BEAD:**
  [sase-yy.8.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.8.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.8.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.8.2.md)
- **COMMITS:**
  - [da0a738](https://github.com/sase-org/sase-core/commit/da0a73895ff8d5aa3597df4abb3fe6004c443489)
    — feat(artifact-links): add publication ownership receipts

# Require durable owners and publish complete event files atomically

Implements phase `sase-yy.8.2` (`publication_durability`) of epic `sase-yy.8`, the
repair child of `sase-yy`. Phase `sase-yy.8.1` (frozen producer identity) is closed and
is already on `master` as `f5a3f5c9` plus `sase-core` `d5d5be4`; build on it.

## 1. Outcome

When this phase is done:

1. `publish_artifact_link_events` never reports `published >= 1` for an operation that
   has no durable receipt. Every acknowledgement is backed by one of: the event object
   committed in **every** required document sidecar root, a committed bead endpoint
   mutation, or a durable machine-local event object for an operation no document or
   bead owner can hold.
2. An operation whose required document owner cannot be resolved stays pending with a
   diagnostic naming the ref and kind, so the outbox drain retains it instead of
   retiring the only copy.
3. Event object installation is atomic and no-replace. `SIGKILL` at any point during
   installation leaves either nothing or the complete fsynced object — never a zero-byte
   final path that poisons every later retry.
4. `sase artifact link add`, `sase artifact link rm`, and plan `links:` frontmatter
   ingestion publish through `resolve_machine_artifact_link_store`, keep their
   synchronous published-on-success contract, and leave the numbered agent checkout with
   no event or `links/` mutation.

## 2. Load this context before editing

- `sase bead show sase-yy.8.2`.
- The epic plan, especially its `publication_durability` section:
  `sase artifact read plan:202609/artifact_link_landing_repairs.md "implementing sase-yy.8.2"`.
- The landing audit and its reproduction evidence:
  - `sase artifact read file:explicit:e7416d60ce1cf8d99a71fe3c "sase-yy.8.2 owner and install defects"`
  - `sase artifact read file:explicit:7f5e04013d3b9e6e1b5a54db "sase-yy.8.2 reproduction script"`
  - `sase artifact read file:explicit:a6e3025e88ceccabf89d82ae "sase-yy.8.2 reproduction output"`
- `sase memory read lint_and_test.md symvision.md -r "verify sase-yy.8.2 changes"`.
- Open the Rust core with `sase repo open sase-core -r "sase-yy.8.2 ownership contract"`
  and follow its `AGENTS.md`. Use only the path that command prints.

Blockers 3 and 4 of the audit are this phase's reproduced defects; blockers 1, 2, 5, 6,
and 7 belong to sibling phases and must not be repaired here.

## 3. Defects to repair

1. **False acknowledgement without an owner.** `_roots_by_operation` in
   `src/sase/sdd/_artifact_link_event_publish.py` builds the required root set from
   `store.sidecar_root_for(ref)`, which returns `None` both for a ref that genuinely has
   no document owner and for a document ref whose role root is unresolved. An empty root
   set then satisfies `set(roots) == durable.get(operation_id, set())`, so the operation
   is acknowledged with no event path anywhere, while `apply_events_to_aggregate`
   rebuilds with that same operation id excluded from the pending overlay — the row
   disappears and the drain retires the only copy. Missing one of two document owners
   passes the same comparison.
2. **Process death poisons event creation.** `_create_event_file` opens the final
   content-addressed path with `O_EXCL` and then writes into it. A kill between those
   two steps leaves a zero-byte final object, and every later replay raises
   `ArtifactLinkEventCorruptionError` forever.
3. **Bead no-op after a failed commit acknowledges anyway.** `apply_events_to_beads` in
   `src/sase/sdd/_artifact_link_event_project.py` returns `False` as soon as no bead row
   changed, without checking whether the _previous_ attempt's bead mutation ever got
   committed.
4. **Manual mutations write into the agent checkout.**
   `src/sase/artifact_cli/link_ops.py` and `src/sase/sdd/artifact_link_inlet.py` hand
   `resolve_artifact_link_store()`'s checkout store straight to the publisher instead of
   the hidden machine store.
5. **Outbox rewrite is not fsynced.** `_write_jsonl` in
   `src/sase/sdd/_artifact_link_outbox_io.py` replaces the queue file without fsyncing
   the temp file or the directory, so an acknowledgement rewrite can reach disk in a
   torn state relative to the receipts it claims.

## 4. Design

### 4.1 Ownership and receipt policy lives in Rust

Ownership and acknowledgement semantics are shared backend policy, so they belong in
`sase-core`. Add `crates/sase_core/src/artifact_link/ownership.rs`, register it in
`crates/sase_core/src/artifact_link/mod.rs`, and re-export from
`crates/sase_core/src/lib.rs` following the `publication_retry` module's shape.

```rust
pub const ARTIFACT_LINK_PUBLICATION_OWNERSHIP_WIRE_SCHEMA_VERSION: u64 = 1;

pub struct ArtifactLinkOwnerRefWire { pub reference: String, pub kind: String }

pub struct ArtifactLinkOwnerRequirementWire {
    pub schema_version: u64,
    pub operation_id: String,
    pub document_refs: Vec<ArtifactLinkOwnerRefWire>, // kind is a declared document kind
    pub bead_refs: Vec<String>,                       // `bead:` endpoints
    pub unowned_refs: Vec<String>,                    // no document/bead owner exists
}

pub struct ArtifactLinkPublicationEvidenceWire {
    pub schema_version: u64,
    pub operation_id: String,
    pub resolved_roots: BTreeMap<String, String>, // document kind -> resolved root path
    pub forced_roots: Vec<String>,                // caller-forced roots (baseline import)
    pub durable_roots: Vec<String>,               // roots proven to hold the object
    pub bead_owner: bool,                         // bead endpoint(s) plus a bead store
    pub bead_receipt: bool,                       // bead mutation is committed
    pub local_receipt: bool,                      // machine-local object is durable
}

pub struct ArtifactLinkPublicationReceiptWire {
    pub schema_version: u64,
    pub operation_id: String,
    pub acknowledged: bool,
    pub pending_reasons: Vec<String>,
}

pub fn artifact_link_event_owner_requirements(
    event: &ArtifactLinkEventWire,
    document_kinds: &[String],
) -> Result<ArtifactLinkOwnerRequirementWire, ArtifactLinkError>;

pub fn artifact_link_publication_receipt(
    requirements: &ArtifactLinkOwnerRequirementWire,
    evidence: &ArtifactLinkPublicationEvidenceWire,
) -> Result<ArtifactLinkPublicationReceiptWire, ArtifactLinkError>;
```

`artifact_link_event_owner_requirements` canonicalizes the event, extracts every
endpoint ref for `observation`, `edge-put`, `edge-remove`, `alias`, and every
`baseline-import` row, and partitions them: refs whose canonical kind is in
`document_kinds` become `document_refs`; `bead:` refs become `bead_refs`; everything
else (`agent:`, `stitch:`, `file:`, and any kind this project does not store in a
document sidecar) becomes `unowned_refs`. Use the existing
`parse_artifact_link_ref_parts` for canonical kinds. Refs are deduplicated and ordered
deterministically.

`artifact_link_publication_receipt` decides acknowledgement:

- every `document_refs` entry whose kind is absent from `evidence.resolved_roots` adds
  `pending_reasons` naming the ref, its kind, and that no sidecar root resolved;
- `required_roots = forced_roots ∪ resolved_roots.values()`; every required root absent
  from `durable_roots` adds a reason naming the root;
- `bead_owner && !bead_receipt` adds a bead-projection-not-committed reason;
- when `required_roots` is empty and `!bead_owner` and `!local_receipt`, add a "no
  durable owner receipt" reason;
- `acknowledged = pending_reasons.is_empty()`, and `operation_id` mismatch between
  requirements and evidence is a hard error.

Mismatched `schema_version` values are hard errors, as elsewhere in this module.

Expose three PyO3 bindings in `crates/sase_core_py/src/lib.rs` exactly as `d5d5be4`
exposed the producer helpers (import alias, `#[pyfunction]` wrapper,
`m.add_function(...)` registration, name in the module's exposure test, and a behavior
assertion in the same in-crate test):

- `artifact_link_publication_ownership_wire_schema_version() -> int`
- `artifact_link_event_owner_requirements(event: dict, document_kinds: list[str]) -> dict`
- `artifact_link_publication_receipt(requirements: dict, evidence: dict) -> dict`

Then add all three names to `tools/validate_sase_core_rs` (`REQUIRED_BINDINGS`, the
`_validate_artifact_link_schema` expected-version map, and a probe inside
`_validate_artifact_link_event_contract` that asserts one acknowledged and one pending
outcome) and to `tools/check_sase_core_rs_bindings`'s `REQUIRED_BINDINGS`, updating
`tests/test_validate_sase_core_rs_tool.py` and
`tests/test_check_sase_core_rs_bindings_tool.py` the same way `f5a3f5c9` did. Do not
hand-edit any release version or dependency pin; `release-plz` owns those, and the
published floor stays `sase-core-rs>=0.33.0,<0.34.0` until it publishes.

### 4.2 Declared document kinds, including unresolved ones

`ArtifactLinkStore.from_sdd_store` currently swallows a role whose root will not
resolve, so an unresolved sidecar is indistinguishable from a kind the project does not
store. Add a field to `ArtifactLinkStore` in
`src/sase/sdd/_artifact_link_store_impl.py`:

```python
unresolved_document_kinds: Mapping[str, str] = field(default_factory=dict)
```

`from_sdd_store` populates it with `kind -> diagnostic` for every document role whose
`repo_root_for_kind` raised (including `SddMaterializationError` from
`store.unresolved_sidecars`). The store's declared document kinds are then
`sorted({*sidecar_roots, *unresolved_document_kinds})`, which is what Python passes to
`artifact_link_event_owner_requirements`. Hand-built stores keep today's behavior
because the new mapping defaults to empty.

### 4.3 Atomic no-replace installation

New module `src/sase/sdd/_artifact_link_event_install.py` owning object installation
(move `_reject_existing_operation_collisions` here from the publisher so
`_artifact_link_event_publish.py` stays well under the `toobig` limits):

- `install_artifact_link_event_object(root, item) -> bool` returns whether this call
  created the object:
  1. If the final path already exists, compare bytes: identical means "already
     installed" (return `False`); a symlink raises `ArtifactLinkEventCorruptionError`;
     different bytes raise `ArtifactLinkEventCorruptionError` — **except** a zero-byte
     file that is not present in `HEAD` (or whose root is not a Git repo), which is a
     crash remnant of the old `O_EXCL` writer and is unlinked before installation
     proceeds. Never unlink or overwrite an object that `HEAD` contains.
  2. Otherwise write the payload into
     `<root>/link-events/v1/.staging/<digest>.<pid>.<uuid4>.tmp` with
     `O_WRONLY|O_CREAT|O_EXCL`, `flush()`, and `os.fsync()`.
  3. Install with `os.link(staging, final)` — the kernel guarantees atomic no-replace
     semantics; `FileExistsError` re-runs the step-1 comparison. Always unlink the
     staging file afterwards, then `os.fsync()` the final object's parent directory.
- `clean_artifact_link_event_staging(root) -> int` removes leftover `.tmp` files under
  `link-events/v1/.staging/`. The publisher calls it while it holds the root's
  `store_git_write_lock`, so a remnant can only come from a dead process. It must never
  touch anything outside that staging directory.

Add `ARTIFACT_LINK_EVENT_STAGING_GITIGNORE_PATTERN = "/link-events/**/.staging/"` to
`src/sase/sdd/_artifact_link_ignore.py` and have `ensure_artifact_link_lock_gitignore`
ensure both patterns, so a crash remnant can never make a hidden clone look dirty to
`_fresh_integration_blocker`. In `src/sase/sdd/_artifact_link_commit.py`, set
`group.needs_lock_ignore = True` for event groups as well as index groups so the rule
lands in the same scoped commit.

Keep the staging file name ending in `.tmp` and the directory dot-prefixed: the event
readers already skip dot-prefixed parts and only match `*.json`.

### 4.4 Machine-local durable receipts for owner-less operations

An `agent:` to `agent:` read, or any operation whose refs are all `unowned_refs` with no
bead endpoint, has no sidecar home. Today it is acknowledged for exactly that reason and
then lost. Preserve the established aggregate-only behavior by giving it a durable,
content-addressed, recoverable home on this machine.

New module `src/sase/sdd/_artifact_link_event_local_store.py`:

- `artifact_link_local_event_root(project_key) -> Path` validates the key with
  `validate_sase_project_name` and returns `sase_projects_dir() / project_key` (objects
  therefore land under `~/.sase/projects/<key>/link-events/v1/<shard>/<digest>.json`,
  the same layout the sidecars use, next to the existing outbox and aggregate files).
- `install_local_artifact_link_event(project_key, item) -> bool` and
  `local_artifact_link_event_is_durable(project_key, item) -> bool` reuse
  `install_artifact_link_event_object` and a byte comparison. There is no Git here, so
  durability is the fsynced object plus its fsynced parent directory.

Include that root in the two durable-event scans so the reduction, the aggregate, and
alias/observed-id lookups all see these operations:

- `ArtifactLinkEventStoreAdapter._durable_events` in
  `src/sase/sdd/_artifact_link_event_store.py`;
- `_iter_event_objects` in `src/sase/sdd/_artifact_link_event_project.py`.

Because the reduction is keyed by operation id, replaying the same operation cannot
inflate `uses`, while a genuinely distinct read of the same edge still counts. This is
not a second push-retry ledger: it holds only operations that no repository can ever
publish, and the landed per-root publication ledger remains the sole retry policy.

### 4.5 Bead receipt verification

Change `apply_events_to_beads` in `src/sase/sdd/_artifact_link_event_project.py` to
return a small frozen result (`changed: bool`, `receipt: bool`,
`diagnostic: str | None`) instead of a bare `bool`:

- apply the endpoint writes as today; when anything changed, commit through
  `commit_bead_link_events` as today;
- when nothing changed (a replayed operation whose bead write already landed), verify
  the previous mutation actually became committed: `git status --porcelain` scoped to
  `beads_dir` must be empty. If it is dirty, commit through the same path; if the bead
  store is not inside a Git repository, treat it as committed.
- any `ArtifactLinkPersistError`/exception yields `receipt=False` plus the diagnostic
  the publisher already surfaces.

`store.beads_dir is None` means there is no bead lane on this machine: `bead_owner` is
then `False` and acknowledgement falls back to the document or local receipt, which
keeps existing bead-less setups working without ever acknowledging on zero evidence.

### 4.6 Publisher rewrite

Rework `_publish_event_objects_locked` in `src/sase/sdd/_artifact_link_event_publish.py`
into explicit per-operation requirements and receipts. Add a thin helper module
`src/sase/sdd/_artifact_link_event_ownership.py` that wraps the two Rust bindings
through `require_rust_binding` and maps a store onto evidence inputs
(`document_kinds_for_store`, `event_owner_requirements`, `publication_receipt`).

Flow, keeping the existing lock order (project publisher lock, then per-root
`store_git_write_lock`, and the store lock released before pushing):

1. Canonicalize and dedupe objects (unchanged).
2. For each object, compute requirements from Rust, then resolve each required document
   kind against `store.sidecar_roots`; add `extra_roots` as forced roots. Group objects
   by resolved root exactly as today, but only for roots that resolved.
3. Publish per root as today (`clean_artifact_link_event_staging`, collision rejection,
   `install_artifact_link_event_object`, scoped commit, `_head_contains_event`),
   collecting durable roots per operation.
4. Install owner-less operations into the machine-local event root and record
   `local_receipt`. Owner-less means `requirements.document_refs` is empty (not merely
   unresolved), there are no forced roots, and `bead_owner` is false — an operation
   whose declared document kind failed to resolve stays pending instead of being
   diverted into machine-local storage.
5. Apply bead projections for operations whose required document roots are all durable,
   then compute `bead_receipt` (§4.5).
6. Ask Rust for each operation's receipt. Acknowledged ids become
   `published_operation_ids`; every `pending_reason` joins `skip_diagnostics` so the
   drain, the CLI, and the inlet surface it verbatim.
7. Rebuild the aggregate from acknowledged objects only, as today.

`_ArtifactLinkEventPublishReport`'s field set stays as it is; only its contents become
honest.

### 4.7 Outbox durability

In `src/sase/sdd/_artifact_link_outbox_io.py`, fsync the temp file before `os.replace`
in `_write_jsonl` and fsync the containing directory afterwards, and fsync the appended
queue file in `append_artifact_link_outbox_entry` and
`append_artifact_link_outbox_event`. Acknowledgement (the rewrite that drops drained
ids) must never become durable before the receipts it is based on.

### 4.8 Manual and inlet routing through the machine store

- `src/sase/artifact_cli/link_ops.py`: import `resolve_machine_artifact_link_store` at
  module level and add `_publication_store()` that resolves the checkout store's project
  key and returns `resolve_machine_artifact_link_store(project_key, Path.cwd())`,
  mirroring `src/sase/artifact_cli/link_import.py`. `add_artifact_link` and
  `remove_artifact_link` use that store for their row reads, observed operation ids, and
  publication. `handle_link_list` keeps using `_store()`. Keep `push_after_commit=None`
  and `mutation_origin="user"`, and keep raising on `publication_error` or an
  unacknowledged operation, so the command stays synchronously published-on-success.
- `src/sase/sdd/artifact_link_inlet.py`: `publish_plan_artifact_link_inlet` gains an
  optional `publication_store` parameter that defaults to
  `resolve_machine_artifact_link_store(link_store.project_key, Path.cwd())`, used for
  `active_operation_ids_for_row` and `publish_artifact_link_events`. The authored
  document keeps its current handling: the managed links block is still previewed and
  written through the resolved checkout store and `document_path`, and the block is
  written only after the events are acknowledged. Failure to resolve the machine store
  raises `ArtifactLinkFrontmatterInletError` — never a silent fall back to the agent
  checkout.
- Automatic writers stay enqueue-only from numbered agent checkouts; do not change
  `artifact_link_derivation.py`, `_artifact_link_renames.py`, or the drain's machine
  routing.

`resolve_machine_artifact_link_store` already returns the ordinary resolved store when
the project is not sidecar storage, so non-split projects keep working unchanged and the
ownership gate keeps failing closed there.

## 5. Tests

Split new tests by scenario rather than growing one oversized module; `tests` is under
the same `toobig` limits as `src`.

`tests/sdd/test_artifact_link_event_ownership.py`:

1. A declared-but-unresolved document owner (a store with `research` in
   `unresolved_document_kinds`) leaves the operation pending: `published == 0`, a
   diagnostic naming the ref and kind, no event object anywhere, and a following drain
   retains the outbox entry.
2. Two document endpoints where one root fails (make its commit fail, or its lock
   unavailable) acknowledge nothing; the recovered replay acknowledges exactly once and
   does not double count `uses` or duplicate rows.
3. A non-document observation (`agent:a read agent:b`, `sidecar_roots={}`) is
   acknowledged, leaves a durable object under the machine-local event root, projects
   one aggregate row, survives `rebuild_aggregate` after the queue entry is drained,
   stays at `uses=1` on replay of the same operation id, and reaches `uses=2` for a
   genuinely distinct second operation.
4. Forcing the machine-local install to fail keeps the owner-less operation pending with
   the "no durable owner receipt" diagnostic.
5. A bead-endpoint operation whose `commit_bead_link_events` fails is not acknowledged;
   the replay, whose bead write is now a no-op, is still not acknowledged while the bead
   store has uncommitted changes, and is acknowledged once it commits.

`tests/sdd/test_artifact_link_event_install.py`:

6. Real process death during installation: `fork`, have the child kill itself between
   the staged write and `os.link`, and assert the parent sees no final object, exactly
   one staging remnant, and a clean successful retry that also removes the remnant.
7. A zero-byte unpublished remnant at a final path is repaired; an object whose bytes
   differ and which `HEAD` contains still raises `ArtifactLinkEventCorruptionError`; a
   symlink still raises.
8. Crash seams after installation, after commit, after push, and before acknowledgement
   each replay cleanly: no duplicated objects, no lost or double-counted operation ids,
   and the push-failure case recovers solely through
   `sweep_artifact_link_publication_retries` (the existing per-root ledger).
9. Staging remnants are covered by the sidecar `.gitignore` rule, so
   `git status --porcelain --untracked-files=all` is empty after a crashed install.

`tests/main/test_artifact_cli_link_machine_store.py` and
`tests/sdd/test_artifact_link_inlet_machine_store.py`:

10. `handle_link_add` / `handle_link_rm` and `publish_plan_artifact_link_inlet` run
    through their real public entry points with a numbered agent checkout and a separate
    hidden machine clone: events and commits appear only in the hidden clone, the agent
    checkout has no `link-events/` and no `links/` mutation and stays clean, the inlet's
    managed links block is still written into the authored document, and an unresolved
    machine store surfaces as a command error rather than a checkout write.
    `tests/sdd/test_artifact_link_hidden_clone_e2e.py` already builds exactly this shape
    (project file, seeded role remotes, `hidden_sidecar_clone_dir`, primary
    fast-forward); reuse its fixture style rather than inventing another one, and see
    `tests/sdd/test_artifact_link_machine_store.py` for the resolver-level helpers.

Existing publisher, acceptance, CLI, and inlet tests that assert the old
zero-owner-acknowledges behavior must be updated to the new contract rather than worked
around; helpers such as `_patch_store` in `tests/main/test_artifact_cli_link.py` need to
patch the machine-store resolver too.

## 6. Verification

1. `just install` (this workspace's virtualenv may have drifted, and the Rust change
   must be rebuilt into it from the linked `sase-core` checkout).
2. In the `sase-core` checkout opened with `/sase_repo`: `just check` (or
   `./scripts/check.sh`), which runs `fmt-check`, `clippy`, and the full workspace test
   suite including the `sase_core_py` binding tests. Never verify with
   `cargo test -p sase_core` alone.
3. `.venv/bin/python tools/validate_sase_core_rs` must exit 0.
4. Focused `pytest` over the artifact-link suites: `tests/sdd/test_artifact_link_*`,
   `tests/main/test_artifact_cli_link*`, `tests/main/test_artifact_link_outbox.py`,
   `tests/test_validate_sase_core_rs_tool.py`,
   `tests/test_check_sase_core_rs_bindings_tool.py`.
5. `just check` for the repo-wide gates. One failure is known and pre-existing on
   `master`: `check_feature_flags` rule 8 reports that live flag bead `sase-z0` has no
   `link_events` registry definition (the registry entry was removed in `a8d99d295`).
   Confirm that is the only failure, do not fix it here — closing `sase-z0` is the
   epic's `acceptance` phase (`sase-yy.8.5`) — and record it with
   `sase bead note sase-yy.8.2 'PROPOSED FOLLOW-UP: ...'` if it is still present.
6. Run `just check-full` only through `/sase_monitor` if `just check`'s scoped selection
   escalates or reports an unusual selection; the epic's combined-tree `check-full` is a
   landing action, not this phase's.

## 7. Constraints

- Phases `sase-yy.8.3`, `sase-yy.8.4`, and `sase-yy.8.5` are in progress in parallel.
  Keep the diff inside the files this plan names. Do not touch
  `_artifact_link_store_reconcile.py`, `_artifact_link_cutover_state.py`,
  `artifact_link_import_indexes.py`, or
  `tests/sdd/test_artifact_link_event_acceptance.py` beyond what the new contract
  strictly forces.
- Preserve the landed module splits and public facades, the sidecar eviction protection,
  the clone fallback, and the per-root publication ledger (its deadline, fairness,
  aging, unpublished-head discovery, and release-evidence behavior). Never add a second
  push-retry ledger.
- Shared identity, ownership, and receipt policy belongs in Rust `sase-core`; Python
  keeps filesystem, locking, process glue, and thin binding adapters.
- Follow this package's symbol convention so `symvision` stays green: the new
  `sase.sdd._artifact_link_event_*` modules are private _modules_ that define _public_
  symbols with `__all__`; importers alias them locally (`... as _name`) the way
  `_artifact_link_event_publish.py` already imports from
  `_artifact_link_event_canonical.py`. Never import a `_`-prefixed symbol across files.
- Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on `sase-yy.8.2`
  with `sase bead note`; do not create beads, and do not close any ancestor bead.
- Close only `sase-yy.8.2`, after running `sase bead epic-symbols sase-yy.8.2` and
  resolving any leftover `--epic-symbol` entries for this phase.
