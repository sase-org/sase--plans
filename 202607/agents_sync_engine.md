---
tier: tale
title: Agents sync engine and CLI
goal: Completed commit-associated agents synchronize safely through each project's
  hidden agents sidecar and are available through a robust CLI and cached status API.
bead: sase-8k.6
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 15:14:56
status: wip
---

- **PROMPT:** [202607/prompts/agents_sync_engine.md](prompts/agents_sync_engine.md)

# Plan: Agents sync engine and CLI

## Goal

Implement phase `sase-8k.6` as one cohesive Python-side sync subsystem: serialize completed, commit-associated local
agents into the hidden agents sidecar; integrate foreign bundles back into normal local artifacts, chats, and the name
registry; synchronize each enabled project's machine-level clone with bounded locking and one push retry; expose cheap
cached status; and add `sase agent sync` with human and JSON output. Keep TUI polling/indicator/update-flow work in the
dependent phase, leave the parent epic open, and do not introduce another plan/bead hierarchy.

The existing dependencies provide the required foundations: machine-qualified names and commit footers, an intrinsic
hidden `agents` sidecar in repository inventory, a stable machine-level clone path, repo-init consent/creation, and the
initial README/manifest scaffold. The Rust agent scanner is a projection without `deny_unknown_fields`, so reconstructed
`agent_meta.json` files can safely carry `imported_from_machine` and `imported_digest`; no `sase-core` change is needed.

## Implementation

### 1. Define strict bundle, manifest, status, and outcome contracts

- Add a focused `src/sase/agents_sync/` package with typed immutable records for portable metadata, commit records,
  manifest entries, project targets, integration/export counts, per-project `SyncOutcome`, and per-project/global sync
  status. Give every persisted format an explicit `schema_version` and stable JSON serialization.
- Treat sidecar contents as untrusted input. Validate manifest and bundle shapes, require each directory/name/machine
  relationship to agree, reject unsafe path components or unsupported schemas, and surface corruption as a
  project-scoped error instead of partially importing it.
- Define the portable `meta.json` projection as an allowlist: qualified name and machine; workflow, model/provider,
  changespec/CL and other display metadata; run timestamps; artifact timestamp; and artifact layout version. Exclude
  PIDs, workspace numbers/paths, chat paths, output paths, and other machine-local absolute state.
- Compute each bundle digest as SHA-256 over canonical UTF-8 JSON bytes for `meta.json` and `commits.json` plus the
  exact `chat.md` bytes, using an unambiguous framed order. Write bundle files, manifests, reconstructed marker files,
  and the status snapshot through same-directory temporary files plus `os.replace` so readers never observe partial
  JSON.
- Adjust agents-sidecar seeding in `src/sase/sdd/_init_files.py` so repo init creates the empty manifest/scaffold when
  absent but never replaces a populated sync manifest or agent tree on a later idempotent init. Update the existing
  sidecar-init tests to lock in that preservation contract.

### 2. Resolve sync targets from lifecycle and repository inventory

- Build enabled-project targets by pairing each project's primary `RepoRecord` with its hidden `agents` sidecar record
  from `collect_repo_inventory`; honor repeatable explicit project filters and aliases through the existing inventory
  lookup behavior. Carry the stable project key/display name, primary checkout, sidecar clone path, and configured
  remote into one target record.
- Represent disabled sidecars, missing workspace/remote metadata, an uncreated remote/clone, and inventory failures as
  truthful skip/error outcomes. Sync may clone an already-created configured remote into the machine-level location, but
  it must never create a remote or bypass repo-init consent.
- Require `require_machine_name()` for export/import/mutating sync. Status may still return a structured actionable
  configuration error, suitable for both table and JSON rendering.

### 3. Discover commit associations and export local agents

- Scan `commit_results.json`/`commit_result.json` under terminal `ace-run` artifacts for the cheap incremental source,
  accepting only records whose repository root is the target primary checkout. Merge those SHAs with a historical
  `git log` backfill of the primary repository, parsing commit messages through `parse_commit_footer` and collecting
  SHA, subject, and committed time for every `SASE_AGENT` attribution.
- Canonicalize legacy unqualified footer names as local names with the configured machine hood. Ignore foreign footer
  names for export, deduplicate commits by SHA, and skip missing/corrupt artifact/chat records with a diagnostic rather
  than failing the whole project.
- Match attributed names to terminal artifact directories using the canonical artifact-path iterators. Export only
  agents with `done.json`, readable `agent_meta.json`, a resolvable transcript, and at least one primary-repo commit.
  Build the allowlisted portable metadata and sorted commit list, compare the stable digest with the manifest, and
  atomically create or refresh only stale local-machine bundles before updating `manifest.json`.
- Keep the local machine authoritative for its own entries: a pulled same-machine bundle is never imported over local
  state, and a new export deterministically replaces the transport copy when local commits or metadata change.

### 4. Integrate foreign bundles into normal local storage

- For every valid manifest entry owned by another machine, verify all bundle files and the advertised digest before
  changing local state. Locate an existing imported artifact with the same qualified name/provenance; an identical
  digest is a no-op, while a newer digest refreshes that imported representation without touching a locally owned
  artifact.
- For a first import, materialize the canonical day-sharded `ace-run` path using the bundle's artifact timestamp and
  layout helper, probing forward by one second on collisions. Reconstruct `agent_meta.json` from portable fields plus
  `imported_from_machine`, `imported_digest`, and the newly materialized chat path; write a terminal `done.json`; and
  copy the transcript verbatim to a deterministic standard chat filename under the correct `YYYYMM` shard.
- Add an explicit registry claim path for an already-qualified imported foreign name. Share the registry's existing
  mutation lock/collision checks, but validate the source machine/name prefix and store the exact durable spelling so an
  unknown foreign hood cannot be rewritten as `<local>.<foreign>.<name>`. Refreshing the same imported owner is allowed;
  collisions with a local or differently sourced owner fail safely.
- Order writes so a failed validation/registry claim leaves no visible partial import, and clean up staging paths on
  failure. Update the artifact index after successful imported marker writes through the existing lifecycle helper so
  `sase agent list`, `sase chats`, lookup, and later TUI scans see the imported agent immediately.

### 5. Implement the locked pull/integrate/export/commit/push transaction

- Acquire a dedicated lock in each agents repo's git directory with a bounded wait. Lock contention returns a skipped
  outcome; unlike the generic fail-open store lock, two agent-sync writers must never proceed concurrently.
- Run all network git commands with `network_git_timeout`, `GIT_TERMINAL_PROMPT=0`, and SSH batch mode. Within the lock:
  ensure the configured clone exists, pull with rebase, integrate foreign bundles, export local bundles, and inspect the
  repo for sidecar payload changes.
- If dirty, stage only `manifest.json` and `agents/`, commit `chore(agents): sync from <machine>`, and push. On a
  non-fast-forward rejection, pull/rebase and rerun integration/export against the newly integrated manifest before
  retrying the push exactly once. Abort a conflicted rebase cleanly and report it rather than leaving an in-progress
  rebase or claiming success.
- Return a complete `SyncOutcome` for every project with pulled/integrated/exported/committed/pushed counts/flags, skip
  reason, and error detail. One project's failure must not prevent the all-project caller from processing the remaining
  targets.

### 6. Add cached, cheap sync status

- Persist `~/.sase/agents_sync/status_snapshot.json` as an atomic, versioned envelope containing per-project behind,
  ahead, and unexported-agent counts plus last fetch time and errors.
- Revalidation performs no network access: compare clone `HEAD` with its already-fetched upstream ref and recompute the
  unexported count only from terminal artifacts' incremental commit markers versus the local manifest. Missing clones,
  upstreams, manifests, and disabled sidecars remain explicit status states.
- Refresh/recompute fetches enabled project clones sequentially with the same noninteractive timeout policy, then
  revalidates and rewrites the snapshot. Normal callers reuse a fresh snapshot and revalidate local facts; explicit
  refresh forces all selected projects. Keep clocks/git runners injectable enough for deterministic cache, cadence, and
  no-network tests.
- Rewrite the status snapshot after a mutating sync so later CLI/TUI consumers immediately see the post-sync state.

### 7. Wire `sase agent sync` and documentation

- Add `sync` to the existing `sase agent` parser/handler and keep the group's displayed subcommands alphabetical. Make
  group help explicitly retain the bare-command delegation to `list`.
- Provide `-p/--project` as a repeatable selector, `-c/--check` for status-only operation, `-r/--refresh` to force the
  network status recompute (valid with check mode), and `-j/--json` for stable machine-readable results. Every long
  option has its short alias, help text distinguishes mutations from read-only checks, and invalid combinations fail at
  the CLI boundary.
- Render a colored Rich table with one row per project and truthful pulled/integrated/exported/pushed or status/skip
  details; emit only stable JSON on stdout in JSON mode and use a nonzero exit when any selected project has a real
  error while allowing benign disabled/not-created skips.
- Add `docs/agents_sidecar.md` covering privacy, bundle layout, machine hoods, local-authority semantics, config
  disable/private controls, sync/check/refresh behavior, retry/locking behavior, and recovery. Cross-link it from
  `docs/configuration.md`. Do not edit memory files or generated agent-instruction shims.

## Validation

- Add isolated unit tests for schema rejection and path safety, canonical digest stability, portable metadata
  allowlisting, atomic manifest/snapshot I/O, populated-manifest preservation, local/foreign partitioning, missing-file
  skips, and exact foreign-name registry claims.
- Add temporary-`SASE_HOME` bundle tests for export/import round trips, transcript and marker reconstruction, digest
  idempotence, foreign bundle refresh, timestamp collision probing, legacy unqualified footer backfill, and artifact
  index/name registry visibility.
- Add local bare-git integration tests for the full sync transaction, bounded lock contention, noninteractive git env,
  pull-before-export ordering, push-rejection rebase/recompute/retry, conflict cleanup, per-project failure isolation,
  and no remote creation without repo-init consent.
- Add status tests proving ordinary revalidation never fetches, ahead/behind and incremental unexported counts update
  from local state, corrupt/stale snapshots recover, refresh fetches selected projects, and post-sync snapshots are
  current.
- Extend parser/dispatch/CLI tests for alphabetical help, bare `sase agent` list delegation, repeatable project filters,
  option validation, colored table output, stable JSON, skip/error exit codes, and check versus refresh behavior.
- Run `just install`, focused pytest modules during development, `just fmt`, and the required full `just check`. Confirm
  the primary checkout has only intended source/test/doc changes and that the parent epic remains open before closing
  only bead `sase-8k.6`.
